# Uptime Kuma Deep Dive & Setup Guide


<div align="center" width="100%">
    <img src="./public/icon.svg" width="128" alt="Uptime Kuma Logo" />


**Uptime Kuma** is a free and open-source monitoring tool for your websites and applications. It provides real-time notifications when your services go down and helps you keep track of their availability.

**GitHub Repository:** [louislam/uptime-kuma](https://github.com/louislam/uptime-kuma)
**Official Website:** [https://uptime.kuma.pet/](https://uptime.kuma.pet/)

</div>

In this guide, we will cover a deep dive into Uptime Kuma, including:
- UI overview
- SQLite-based setup
- MariaDB/MySQL-based setup
- Migration from SQLite to MySQL/MariaDB

## 📑 Table of Contents
- [1️⃣ SQLite Setup](#1️⃣-sqlite-setup)
- [2️⃣ MariaDB/MySQL Setup](#2️⃣-mariadbmysql-setup)
- [3️⃣ Migration (SQLite to MariaDB)](#3️⃣-migration-sqlite-to-mariadb)
- [📚 Official Documentation & References](#📚-official-documentation--references)


## 🧩 Types of Setup

There are two types of database setups:
1. SQLite (default, lightweight)
2. MariaDB/MySQL (recommended for production)



## 1️⃣ SQLite Setup

> https://uptimekuma.org/install-uptime-kuma-docker/

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:2
    container_name: uptime-kuma
    restart: always
    ports:
      - "3001:3001"
    volumes:
      - ./data:/app/data
    environment:
      - TZ=UTC
      - UMASK=0022
    networks:
      - kuma_network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3001"]
      interval: 30s
      retries: 3
      start_period: 10s
      timeout: 5s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

networks:
  kuma_network:
    driver: bridge
````

**Access URL:**
[http://192.168.1.222:3001/setup-database](http://192.168.1.222:3001/setup-database)



## 2️⃣ MariaDB/MySQL Setup

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:2
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - "3001:3001"
    depends_on:
      mariadb:
        condition: service_healthy
    environment:
      - UPTIME_KUMA_DB_TYPE=mariadb
      - UPTIME_KUMA_DB_HOSTNAME=mariadb
      - UPTIME_KUMA_DB_PORT=3306
      - UPTIME_KUMA_DB_NAME=kuma
      - UPTIME_KUMA_DB_USERNAME=kuma
      - UPTIME_KUMA_DB_PASSWORD=password
      - UPTIME_KUMA_HOST=0.0.0.0
      - UPTIME_KUMA_PORT=3001
    volumes:
      - ./kuma-data:/app/data

  mariadb:
    image: mariadb:11.4
    container_name: mariadb
    restart: unless-stopped
    environment:
      - MARIADB_ROOT_PASSWORD=password
      - MARIADB_DATABASE=kuma
      - MARIADB_USER=kuma
      - MARIADB_PASSWORD=password
    volumes:
      - ./mariadb-data:/var/lib/mysql
    ports:
      - "3306:3306"
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 10s
      timeout: 5s
      retries: 5
```

**Access URL:**
[http://192.168.1.222:3001/setup](http://192.168.1.222:3001/setup)

> Note: The database type is preconfigured, so the setup page will not prompt for it.



## 3️⃣ Migration (SQLite to MariaDB)

### 🪜 Steps

1. Start Uptime Kuma:

```bash
docker compose up -d
```

2. Sign up and access dashboard:
   [http://192.168.1.221:3001/dashboard](http://192.168.1.221:3001/dashboard)

3. Stop the container before migration:

```bash
docker stop uptime-kuma
```



### 📦 Copy SQLite Database

```bash
mkdir -p ./mariadb-data/tmp
cp -a ./data/kuma.db ./mariadb-data/tmp
ls ./mariadb-data/tmp
du -sh ./mariadb-data/tmp
```



### 🧹 Optional: Cleanup Old Heartbeat Data

```bash
#!/bin/bash
DB_PATH="./kuma.db"

sqlite3 "$DB_PATH" <<EOF
DELETE FROM heartbeat WHERE time < datetime('now', '-30 days');
VACUUM;
EOF
```



### ⚠️ Important Note

Do **not** copy the database while Uptime Kuma is running.
Otherwise, you may encounter:

```
Error: database disk image is malformed
```

This indicates SQLite database corruption, not a MySQL or migration tool issue.



### 🔄 Migration Process

```bash
docker exec -it mariadb /bin/bash

cd /var/lib/mysql/tmp

apt update && apt install python3-pip sqlite3 -y
pip install sqlite3-to-mysql --break-system-packages
```

#### Verify SQLite Database

```bash
sqlite3 kuma.db "SELECT name FROM sqlite_master WHERE type='table';"
sqlite3 kuma.db ".mode csv" ".output table.csv" "SELECT * FROM monitor;"
sqlite3 kuma.db "PRAGMA integrity_check;"
```

#### Run Migration

```bash
sqlite3mysql \
  --sqlite-file /var/lib/mysql/tmp/kuma.db \
  --mysql-database kuma \
  --mysql-user kuma \
  --mysql-password password
```



### 🔁 Restart Services

```bash
docker compose up -d --force-recreate
```



## 📚 Official Documentation & References

### 📖 Official Documentation

* (Add official documentation link here if needed)

### 🚀 Demo

* [https://uptime.kuma.pet/](https://uptime.kuma.pet/)
* [https://status.easyselfhost.com/](https://status.easyselfhost.com/)

### ⚙️ Setup Guide

* [https://github.com/louislam/uptime-kuma/wiki](https://github.com/louislam/uptime-kuma/wiki)
* [https://github.com/louislam/uptime-kuma/wiki/Environment-Variables](https://github.com/louislam/uptime-kuma/wiki/Environment-Variables)
* [https://uptimekuma.org/install-uptime-kuma-docker/](https://uptimekuma.org/install-uptime-kuma-docker/)

### 🔐 Reverse Proxy

* [https://github.com/louislam/uptime-kuma/wiki/Reverse-Proxy](https://github.com/louislam/uptime-kuma/wiki/Reverse-Proxy)
* [https://github.com/louislam/uptime-kuma/wiki/Reverse-Proxy-with-Cloudflare-Tunnel](https://github.com/louislam/uptime-kuma/wiki/Reverse-Proxy-with-Cloudflare-Tunnel)

### 🔄 Migration Guide

* [https://github.com/louislam/uptime-kuma/wiki/Migration-From-v1-To-v2](https://github.com/louislam/uptime-kuma/wiki/Migration-From-v1-To-v2)


## 📝 License

This guide is provided as-is for educational and professional use.

## 🤝 Contributing
Feel free to suggest improvements or report issues via pull requests or the issues tab.

## 💼 Connect with Me 👇😊

*   🔥 [**YouTube**](https://www.youtube.com/@DevOpsinAction?sub_confirmation=1)
*   ✍️ [**Blog**](https://ibraransari.blogspot.com/)
*   💼 [**LinkedIn**](https://www.linkedin.com/in/ansariibrar/)
*   👨‍💻 [**GitHub**](https://github.com/meibraransari?tab=repositories)
*   💬 [**Telegram**](https://t.me/DevOpsinActionTelegram)
*   🐳 [**Docker Hub**](https://hub.docker.com/u/ibraransaridocker)

### ⭐ If You Found This Helpful...

***Please star the repo and share it! Thanks a lot!*** 🌟

