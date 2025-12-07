# FileBot Docker (Slim)

This directory contains the Docker setup for an optimized "Slim" build of FileBot 4.8.5.

## Overview

This build uses Alpine Linux, JRE 11 (stripped), and software rendering to achieve a small image size (~600MB) while maintaining functionality.

## Usage

### Prerequisites

- Docker
- Docker Compose

### Running with Docker Compose

1.  Navigate to this directory:
    ```bash
    cd Assets/filebot-docker
    ```

2.  Build and run the container:
    ```bash
    docker-compose up -d --build
    ```

3.  Access the application:
    Open your web browser and go to `http://localhost:5800`.

### Configuration

The `docker-compose.yml` file allows you to configure the container environment, including:
-   **Display Resolution:** `DISPLAY_WIDTH` and `DISPLAY_HEIGHT`
-   **User ID/Group ID:** `USER_ID` and `GROUP_ID` to match your host permissions
-   **Timezone:** `TZ`

## Details

-   **Base Image:** `jlesage/baseimage-gui:alpine-3.18-v4`
-   **Java Version:** Liberica JRE 11 (Stripped)
-   **FileBot Version:** 4.8.5 (Portable)
-   **Optimization:** Binaries are stripped of debugging symbols to reduce size.
