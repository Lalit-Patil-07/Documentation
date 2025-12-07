# =========================================================
# STAGE 1: DOWNLOADER & OPTIMIZER
# =========================================================
FROM alpine:3.18 AS downloader

# Install tools + binutils (for the 'strip' command)
RUN apk add --no-cache wget tar xz binutils

WORKDIR /tmp

# 1. Download FileBot
RUN echo "=== Downloading FileBot ===" && \
    wget -q -O filebot.tar.xz "https://github.com/barry-allen07/FB-Mod/releases/download/4.8.5/FileBot_4.8.5-Linux-Portable.tar.xz" && \
    mkdir -p /filebot-extract && \
    tar -xJf filebot.tar.xz -C /filebot-extract

# 2. Download Logo
RUN echo "=== Downloading Logo ===" && \
    wget -q -O icon.png "https://raw.githubusercontent.com/jlesage/docker-templates/master/jlesage/images/filebot-icon.png"

# 3. Download Liberica JRE 11 Full (Standard Linux/Glibc)
# NOTE: Using JRE (not JDK) saves ~300MB
RUN echo "=== Downloading Liberica JRE 11 Full ===" && \
    wget -q -O java.tar.gz "https://download.bell-sw.com/java/11.0.29+10/bellsoft-jre11.0.29+10-linux-amd64-full.tar.gz" && \
    mkdir -p /jre && \
    tar -xzf java.tar.gz -C /jre --strip-components=1 \
        --exclude='demo' \
        --exclude='man' \
        --exclude='src.zip' \
        --exclude='legal'

# 4. OPTIMIZATION: STRIP BINARIES HERE
# We strip debugging symbols NOW, before copying to the final image.
# This prevents layer duplication in the final stage.
RUN echo "=== Stripping Binaries ===" && \
    find /jre -name "*.so" -exec strip --strip-unneeded {} \; 2>/dev/null || true && \
    find /filebot-extract -name "*.so" -exec strip --strip-unneeded {} \; 2>/dev/null || true

# =========================================================
# STAGE 2: RUNTIME IMAGE (Final)
# =========================================================
FROM jlesage/baseimage-gui:alpine-3.18-v4

ENV APP_NAME="FileBot" \
    DISPLAY_WIDTH=1280 \
    DISPLAY_HEIGHT=720 \
    KEEP_APP_RUNNING=1

# 1. Install Runtime Dependencies
# gcompat: Allows standard Linux Java to run on Alpine
RUN apk add --no-cache \
    gcompat \
    libc6-compat \
    libstdc++ \
    libx11 \
    libxrender \
    libxtst \
    libxi \
    libxxf86vm \
    mesa-gl \
    gtk+3.0 \
    bash \
    mediainfo \
    chromaprint \
    p7zip \
    fontconfig \
    ttf-dejavu \
    && rm -rf /var/cache/apk/* \
    && fc-cache -f

# 2. Create directories
RUN mkdir -p /opt/filebot /opt/java /config /data

# 3. Copy ONLY the stripped files from the downloader
COPY --from=downloader /filebot-extract/ /opt/filebot/
COPY --from=downloader /jre/ /opt/java/
COPY --from=downloader /tmp/icon.png /opt/filebot/icon.png

# 4. Install Icon
RUN install_app_icon.sh /opt/filebot/icon.png

# 5. Permissions
RUN find /opt/filebot -name "*.sh" -exec chmod +x {} \; 2>/dev/null || true && \
    mkdir -p /opt/filebot/data && \
    chmod -R 777 /opt/filebot/data && \
    chmod +x /opt/java/bin/java

# ---------------------------------------------------------
# LAUNCHER SCRIPT
# ---------------------------------------------------------
RUN cat > /opt/filebot/filebot-launcher.sh << 'EOF'
#!/bin/bash
set -e

# JAVA SETUP
JAVA_HOME="/opt/java"
JAVA_BIN="$JAVA_HOME/bin/java"
export LD_LIBRARY_PATH="$JAVA_HOME/lib:$JAVA_HOME/lib/server:$LD_LIBRARY_PATH"

# CONFIG
mkdir -p /config/.java/.userPrefs /config/tmp
chmod -R 777 /config
export APP_DATA=/config

# CLASSPATH
CLASSPATH=""
for jar in $(find /opt/filebot -name "*.jar" -type f); do
    if [ -n "$CLASSPATH" ]; then
        CLASSPATH="$CLASSPATH:$jar"
    else
        CLASSPATH="$jar"
    fi
done

FILEBOT_JAR=$(find /opt/filebot -name "FileBot.jar" -o -name "filebot.jar" | head -1)

echo "Starting FileBot (Slim Mode)..."

# LAUNCH COMMAND
exec $JAVA_BIN \
    -Dapplication.deployment=docker \
    -Dnet.filebot.Archive.extractor=SevenZipExecutable \
    -Dnet.filebot.AcoustID.fpcalc=/usr/bin/fpcalc \
    -Djava.awt.headless=false \
    -Dprism.order=sw \
    -Dapplication.dir=/config \
    -Djava.io.tmpdir=/config/tmp \
    -Duser.home=/config \
    -Djava.util.prefs.userRoot=/config/.java \
    --add-modules javafx.controls,javafx.fxml,javafx.swing,javafx.media,javafx.web \
    -cp "$CLASSPATH" \
    -jar "$FILEBOT_JAR" "$@"
EOF

RUN chmod +x /opt/filebot/filebot-launcher.sh

# Startapp
RUN echo '#!/bin/bash' > /startapp.sh && \
    echo 'exec /opt/filebot/filebot-launcher.sh' >> /startapp.sh && \
    chmod +x /startapp.sh

WORKDIR /config
EXPOSE 5800 5900
VOLUME ["/config", "/data"]
LABEL maintainer="Lalit Patil"
