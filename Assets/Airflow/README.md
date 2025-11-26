# Airflow 3.0.6 Deployment Guide on Ubuntu

**Architecture Overview:**

  * **Version:** Airflow 3.0.6
  * **Executor:** CeleryExecutor (Distributed scaling)
  * **Database:** PostgreSQL (Metadata)
  * **Broker:** Redis (Task Queue)
  * **Auth Manager:** FAB (Flask AppBuilder) for User Management & UI
  * **OS:** Ubuntu 22.04 / 24.04 LTS

-----

## Phase 1: System Prerequisites

Update the system and install the necessary backend services.

```bash
# 1. Update Packages
sudo apt update && sudo apt upgrade -y

# 2. Install Python Dependencies
sudo apt install -y python3-pip python3-venv software-properties-common build-essential libssl-dev libffi-dev python3-dev

# 3. Install & Start Redis (Message Broker)
sudo apt install -y redis-server
sudo systemctl enable --now redis-server

# 4. Install & Start PostgreSQL (Metadata Database)
sudo apt install -y postgresql postgresql-contrib
sudo systemctl enable --now postgresql
```

-----

## Phase 2: Database Setup

Configure PostgreSQL. **Note:** For Postgres 15+, explicit schema permissions are required.

```bash
# Log in to Postgres
sudo -u postgres psql
```

**Run the following SQL commands:**

```sql
-- Create Database and User
CREATE DATABASE airflow_db;
CREATE USER airflow_user WITH PASSWORD 'airflow_pass';

-- Grant Access
GRANT ALL PRIVILEGES ON DATABASE airflow_db TO airflow_user;

-- CRITICAL FIX for Postgres 15+: Grant Schema Permissions
\c airflow_db
GRANT ALL ON SCHEMA public TO airflow_user;

-- Exit
\q
```

-----

## Phase 3: Installation & Environment

We will install Airflow in `/opt/airflow` and set permissions for the default `ubuntu` user.

### 1\. Create Directories

```bash
sudo mkdir -p /opt/airflow/{dags,logs,plugins,config}
sudo chown -R ubuntu:ubuntu /opt/airflow
```

### 2\. Create Virtual Environment

```bash
cd /opt/airflow
python3 -m venv venv
source venv/bin/activate
```

### 3\. Install Airflow 3.0.6 & FAB Provider

We install Airflow with `celery`, `postgres`, and `redis` extras. We also install the **FAB Provider** to enable the classic UI and user management command.

```bash
# Set version variables
AIRFLOW_VERSION=3.0.6
PYTHON_VERSION="$(python --version | cut -d " " -f 2 | cut -d "." -f 1-2)"
CONSTRAINT_URL="https://raw.githubusercontent.com/apache/airflow/constraints-${AIRFLOW_VERSION}/constraints-${PYTHON_VERSION}.txt"

# Install Airflow
pip install "apache-airflow[celery,postgres,redis]==${AIRFLOW_VERSION}" --constraint "${CONSTRAINT_URL}"

# Install Flask AppBuilder (Required for Admin UI and 'airflow users' command)
pip install apache-airflow-providers-fab
```

-----

## Phase 4: Configuration

We must configure Airflow to use Celery and the FAB Auth manager.

### 1\. Generate Default Config

```bash
export AIRFLOW_HOME=/opt/airflow
airflow config list > /dev/null
```

### 2\. Edit `airflow.cfg`

Open the configuration file:

```bash
nano /opt/airflow/airflow.cfg
```

**Find and change the following settings:**

```ini
[core]
# 1. Use Celery Executor for MWAA-like scaling
executor = CeleryExecutor

# 2. Use FAB Auth Manager (Restores the 'users' command and Login UI)
auth_manager = airflow.providers.fab.auth_manager.fab_auth_manager.FabAuthManager

# 3. Load Examples (Set to False for Production)
load_examples = False

[database]
# 4. Connect to Postgres
sql_alchemy_conn = postgresql+psycopg2://airflow_user:airflow_pass@localhost/airflow_db

[celery]
# 5. Configure Redis Broker
broker_url = redis://localhost:6379/0
result_backend = db+postgresql://airflow_user:airflow_pass@localhost/airflow_db
```

-----

## Phase 5: Initialization

Initialize the database and create the admin user.

```bash
# 1. Run Database Migrations
airflow db migrate

# 2. Create Admin User
airflow users create \
    --username admin \
    --password admin \
    --firstname Admin \
    --lastname User \
    --role Admin \
    --email admin@example.com
```

-----

## Phase 6: Service Management (Systemd)

In Airflow 3.0, the "Webserver" is replaced by the "API Server". We create 5 separate services.

### 1\. Create Service Files

Run the following commands to create the files in `/etc/systemd/system/`.

**A. API Server (The UI)** - `/etc/systemd/system/airflow-apiserver.service`

```ini
[Unit]
Description=Airflow API Server (UI)
After=network.target postgresql.service redis-server.service

[Service]
Environment="AIRFLOW_HOME=/opt/airflow"
User=ubuntu
Group=ubuntu
Type=simple
# Note: --port 8080 makes it serve the UI on the standard port
ExecStart=/opt/airflow/venv/bin/airflow api-server --port 8080
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

**B. Scheduler** - `/etc/systemd/system/airflow-scheduler.service`

```ini
[Unit]
Description=Airflow Scheduler
After=network.target postgresql.service redis-server.service

[Service]
Environment="AIRFLOW_HOME=/opt/airflow"
User=ubuntu
Group=ubuntu
Type=simple
ExecStart=/opt/airflow/venv/bin/airflow scheduler
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

**C. Worker** - `/etc/systemd/system/airflow-worker.service`

```ini
[Unit]
Description=Airflow Celery Worker
After=network.target postgresql.service redis-server.service

[Service]
Environment="AIRFLOW_HOME=/opt/airflow"
User=ubuntu
Group=ubuntu
Type=simple
ExecStart=/opt/airflow/venv/bin/airflow celery worker
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

**D. Triggerer** - `/etc/systemd/system/airflow-triggerer.service`

```ini
[Unit]
Description=Airflow Triggerer
After=network.target postgresql.service redis-server.service

[Service]
Environment="AIRFLOW_HOME=/opt/airflow"
User=ubuntu
Group=ubuntu
Type=simple
ExecStart=/opt/airflow/venv/bin/airflow triggerer
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

**E. DAG Processor** - `/etc/systemd/system/airflow-dag-processor.service`
*Required in v3 to read DAG files and update the DB.*

```ini
[Unit]
Description=Airflow DAG Processor
After=network.target postgresql.service

[Service]
Environment="AIRFLOW_HOME=/opt/airflow"
User=ubuntu
Group=ubuntu
Type=simple
ExecStart=/opt/airflow/venv/bin/airflow dag-processor
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

### 2\. Start All Services

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now airflow-apiserver airflow-scheduler airflow-worker airflow-triggerer airflow-dag-processor
```

-----

## Phase 7: Verification

1.  **Check Status:**
    ```bash
    sudo systemctl status airflow-apiserver airflow-scheduler airflow-dag-processor
    ```
2.  **Access UI:**
    Open browser to `http://<YOUR_VM_IP>:8080`.
      * **User:** `admin`
      * **Password:** `admin`
3.  **Deploy a DAG:**
    Create a python file in `/opt/airflow/dags/`.
    ```bash
    nano /opt/airflow/dags/test_dag.py
    ```
    *Wait \~30 seconds for the **DAG Processor** to pick it up.*

-----

## Troubleshooting Cheat Sheet

| Error | Cause | Fix |
| :--- | :--- | :--- |
| `permission denied for schema public` | Postgres 15+ security change. | Run `GRANT ALL ON SCHEMA public TO airflow_user;` in SQL. |
| `invalid choice: 'users'` | Missing FAB provider. | Run `pip install apache-airflow-providers-fab`. |
| `airflow webserver command not found` | Airflow 3 removed Webserver. | Use `airflow api-server` instead. |
| **DAGs not showing in UI** | Missing DAG Processor. | Ensure `airflow-dag-processor` service is running. |
| **Tasks stuck in "Deferrable"** | Missing Triggerer. | Ensure `airflow-triggerer` service is running. |
