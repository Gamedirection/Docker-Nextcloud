# Context

```
OS: Debian GNU/Linux 12 (bookworm) x86_64
CPU: AMD Ryzen 9 5950X (32) @ 3.399GHz 
GPU: 00:02.0 Red Hat, Inc. QXL paravirtual graphic card 
Memory: 22GB / 82GB
```

> [!NOTE]
> ## When idle:
> *I run about 100+ other containers on this server. This snapshot is at a pretty inactive time.*
> - `CPU: 3-5%` 
> - `Memory: 450MB ram`
> - `Network: 700RX and 100TX`
> - `I/O: 75MB Read & 275MB Write`

> *I run an external proxy to manage my domain name*

# Compose.yaml Notes for clarity

```yaml
# ======================================
# Nextcloud - Insecure (HTTP) Setup
# MariaDB backend + Apache base
# For local/private testing only.
# ======================================
name: nextcloud

services:
  # -----------------------
  # MariaDB - Database Layer
  # -----------------------
  mariadb:
    image: mariadb:10.11 # Stable LTS release of MariaDB
    container_name: mariadb
    restart: always
    # Use recommended transaction isolation and binary logging.
    # Binary logging is needed for replication and consistent backups.
    command: --transaction-isolation=READ-COMMITTED --log-bin=binlog --binlog-format=ROW
    volumes:
      # Mount host path for database persistence
      - ${DB_DATA_LOCATION}:/var/lib/mysql
    environment:
      # Auto-upgrade settings improve startup if the DB version changes.
      - MARIADB_AUTO_UPGRADE=1
      - MARIADB_DISABLE_UPGRADE_BACKUP=1
      - INNODB_BUFFER_POOL_INSTANCES=4
      # Sensitive credentials are passed through environment variables.
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD:?Set before deployment}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD:?Set before deployment}
      - MYSQL_DATABASE=${MYSQL_DATABASE:?Set before deployment}
      - MYSQL_USER=${MYSQL_USER:?Set before deployment}
    env_file:
      # Optional additional environment file for overrides or external app configs
      - .env
    networks:
      - nextcloud

  # -----------------------
  # Nextcloud - Application Layer
  # -----------------------
  nextcloud:
    image: nextcloud:production
    container_name: nextcloud
    restart: always
    # HTTP only (insecure testing version). For HTTPS add reverse proxy later.
    ports:
      - ${PORT}:80
    # Legacy linking (still works fine in local bridge networks)
    links:
      - mariadb
      - redis
      - collabora
    volumes:
      # Application data, external storage mounts, and configuration
      - ${DATA_LOCATION}:/var/www/html
      - ${CLI_PHP}:/usr/local/etc/php/conf.d/cli-php.ini
      - nas-storage:/srv/storage:rw
    environment:
      # File and process ownership
      - PUID=1000
      - PGID=1000
      - TZ=${TZ}               # System timezone
      # PHP tuning for heavy usage
      - PHP_MEMORY_LIMIT=${PHP_MEMORY_LIMIT}
      - PHP_MAX_EXECUTION_TIME=600
      - PHP_MAX_INPUT_TIME=360
      - PHP_POST_MAX_SIZE=16G
      - PHP_UPLOAD_LIMIT=${PHP_UPLOAD_LIMIT}
      - APPCONFIG_APPSTORETIMEOUT=300
      # PHP Opcache tuning for large Nextcloud installs
      - PHP_OPCACHE_MEMORY_CONSUMPTION=512
      - PHP_OPCACHE_INTERNED_STRINGS_BUFFER=32
      - PHP_OPCACHE_MAX_ACCELERATED_FILES=20000
      - PHP_OPCACHE_REVALIDATE_FREQ=60
      - PHP_OPCACHE_ENABLE_CLI=1
      # Redis caching configuration
      - REDIS_HOST=${REDIS_HOST}
      - REDIS_HOST_PORT=${REDIS_PORT}
      # Initial setup credentials (only used on first run)
      - NEXTCLOUD_ADMIN_USER=${NEXTCLOUD_ADMIN_USER:?Set before deployment}
      - NEXTCLOUD_ADMIN_PASSWORD=${NEXTCLOUD_ADMIN_PASSWORD:?Set before deployment}
      # Network access configuration
      - NEXTCLOUD_TRUSTED_DOMAINS=${NEXTCLOUD_TRUSTED_DOMAINS}
      - OVERWRITEPROTOCOL=${OVERWRITEPROTOCOL}
      - OVERWRITECLIURL=${OVERWRITECLIURL}

    # -----------------------------------------------------------------
    # Entry command - installs cron, generates PHP configs at runtime.
    # Instead of using a secondary cron container, this setup runs cron
    # inside the Nextcloud container using Linux's built-in cron service.
    # -----------------------------------------------------------------
    command: >
      bash -c "
      apt-get update -qq &&
      apt-get install -y -qq cron &&
      mkdir -p /var/log/nextcloud &&
      chown www-data:www-data /var/log/nextcloud &&
      # Runtime PHP tuning
      echo 'memory_limit = 4096M' >> /usr/local/etc/php/conf.d/memory.ini &&
      echo 'max_execution_time = 600' >> /usr/local/etc/php/conf.d/execution.ini &&
      echo 'opcache.memory_consumption=512' >> /usr/local/etc/php/conf.d/opcache.ini &&
      echo 'opcache.interned_strings_buffer=32' >> /usr/local/etc/php/conf.d/opcache.ini &&
      echo 'opcache.max_accelerated_files=20000' >> /usr/local/etc/php/conf.d/opcache.ini &&
      # Cron job setup to handle background tasks
      echo '*/5 * * * * www-data /usr/local/bin/php -f /var/www/html/cron.php >> /var/log/nextcloud/cron.log 2>&1' > /etc/cron.d/nextcloud &&
      chmod 0644 /etc/cron.d/nextcloud &&
      cron -L 15 &&
      exec apache2-foreground
      "
    env_file:
      - .env
    networks:
      - nextcloud

  # -----------------------
  # Redis - Caching Layer
  # -----------------------
  redis:
    container_name: redis-nc
    image: redis:latest
    # Use tmpfs to avoid persistent writes (keeps Redis RAM-only)
    volumes:
      - type: tmpfs
        target: /data
        tmpfs:
          size: 4294967296 # 4GB tmpfs cache
    environment:
      - REDIS_MEMORY=2GB
    expose:
      - 6379 # Internal port, not published externally
    command: redis-server --save 60 1 --loglevel warning
    networks:
      - nextcloud
    restart: always

  # -----------------------
  # Collabora - Office Integration
  # -----------------------
  collabora:
    container_name: collabora
    hostname: collabora
    privileged: true
    tty: true
    ports:
      - 9980:9980 # Collabora online editing service
    cap_add:
      - MKNOD
    image: collabora/code:latest
    env_file:
      - .env
    networks:
      - nextcloud
    restart: always

  # -----------------------
  # Whiteboard
  # -----------------------
  whiteboard:
    container_name: whiteboard
    image: ghcr.io/nextcloud-releases/whiteboard:release
    ports:
      - 8081:8081
    environment:
      - NEXTCLOUD_URL=${NEXTCLOUD_URL}
      - JWT_SECRET_KEY=${JWT_SECRET_KEY}
    env_file:
      - .env
    networks:
      - nextcloud

  # -----------------------
  # Watchtower - Auto Updater
  # -----------------------
  watchtower:
    container_name: watchtower
    image: containrrr/watchtower
    volumes:
      # Gives Watchtower access to Docker API to monitor/update containers
      - /var/run/docker.sock:/var/run/docker.sock

# -----------------------
# Network and Volume Definitions
# -----------------------
networks:
  nextcloud:
    driver: bridge

volumes:
  # External NAS volumes mapped to Nextcloud folders
  nas-storage:
    external: true
```

# .env file notes
```.env
DATA_LOCATION=/path/to/NextCloud/nextcloud
DB_DATA_LOCATION=/path/to/NextCloud/mariadb
CLI_PHP=/path/to/NextCloud/nextcloud/php-config/cli-php.ini
PORT=8080

NEXTCLOUD_TRUSTED_DOMAINS=nextcloud.example.com
OVERWRITEPROTOCOL=https
OVERWRITECLIURL=https://nextcloud.example.com

REDIS_HOST=redis-nc
REDIS_PORT=6379
PHP_MEMORY_LIMIT=2048M
PHP_UPLOAD_LIMIT=100M

MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD:?Set before deployment}
MYSQL_PASSWORD=${MYSQL_PASSWORD:?Set before deployment}
MYSQL_DATABASE=nextcloud
MYSQL_USER=nextcloud

NEXTCLOUD_ADMIN_USER=${NEXTCLOUD_ADMIN_USER:?Set before deployment}
NEXTCLOUD_ADMIN_PASSWORD=${NEXTCLOUD_ADMIN_PASSWORD:?Set before deployment}
JWT_SECRET_KEY=${JWT_SECRET_KEY:?Set before deployment}
TZ=UTC
```
