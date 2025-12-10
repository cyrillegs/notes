# MongoDB 7.0 Docker Setup on Ubuntu 24.04

[![Docker](https://img.shields.io/badge/Docker-Yes-blue)](https://www.docker.com/) 
[![Ubuntu](https://img.shields.io/badge/OS-Ubuntu%2024.04-orange)](https://ubuntu.com/) 
[![MongoDB](https://img.shields.io/badge/MongoDB-7.0-green)](https://www.mongodb.com/)

> This guide explains how to run **MongoDB 7.0** in Docker on Ubuntu 24.04 with:
> - Persistent storage  
> - Multi-project databases/users  
> - Automatic restart on VPS reboot  
> - Optional Mongo Express web GUI

---

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Folder Structure](#folder-structure)
3. [Run MongoDB Container](#run-mongodb-container)
4. [Verify MongoDB](#verify-mongodb)
5. [Connect Using Mongosh](#connect-using-mongosh)
6. [Create Databases/Users for Multiple Projects](#create-databasesusers-for-multiple-projects)
7. [Web GUI: Mongo Express (Optional)](#web-gui-mongo-express-optional)
8. [Persistence & Backup](#persistence--backup)
9. [Managing Docker Container](#managing-docker-container)
10. [Notes & Best Practices](#notes--best-practices)

---

## Prerequisites

- Ubuntu 24.04 VPS  
- Docker installed (`docker --version`)  
- Optional: Docker Compose  
- Basic Linux terminal knowledge

---

## Folder Structure

Create folders for MongoDB persistence:

```bash
mkdir -p ~/mongo/mongodb/{data,logs}
chmod 777 ~/mongo/mongodb/data
chmod 777 ~/mongo/mongodb/logs
````

* `data/` → database files
* `logs/` → MongoDB logs

> **Tip:** Adjust permissions later for security.

---

## Run MongoDB Container

```bash
docker run -d \
  --name mongodb \
  --restart unless-stopped \
  -p 27017:27017 \
  -v ~/mongo/mongodb/data:/data/db \
  -v ~/mongo/mongodb/logs:/logs \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=StrongPassword123 \
  mongo:7.0
```

* `--restart unless-stopped` → auto-start on VPS reboot
* `-v` → persistent storage
* `-p 27017:27017` → expose MongoDB port

---

## Verify MongoDB

```bash
docker ps
docker logs mongodb
```

Expected:

* STATUS: Up
* No crashing/restarting

---

## Connect Using Mongosh

```bash
docker exec -it mongodb mongosh -u admin -p StrongPassword123
```

---

## Create Databases/Users for Multiple Projects

```js
// Project A
use projectA_db
db.createUser({
  user: "projectA_user",
  pwd: "passwordA",
  roles: [{ role: "readWrite", db: "projectA_db" }]
})

// Project B
use projectB_db
db.createUser({
  user: "projectB_user",
  pwd: "passwordB",
  roles: [{ role: "readWrite", db: "projectB_db" }]
})
```

**Connection strings:**

```
mongodb://projectA_user:passwordA@127.0.0.1:27017/projectA_db
mongodb://projectB_user:passwordB@127.0.0.1:27017/projectB_db
```

---

## Web GUI: Mongo Express (Optional)

Run Mongo Express:

```bash
docker run -d \
  --name mongo-express \
  --restart unless-stopped \
  -p 8081:8081 \
  -e ME_CONFIG_MONGODB_ADMINUSERNAME=admin \
  -e ME_CONFIG_MONGODB_ADMINPASSWORD=StrongPassword123 \
  -e ME_CONFIG_MONGODB_SERVER=host.docker.internal \
  mongo-express
```

* Access in browser: `http://your_vps_ip:8081`
* Optional: allow port through UFW:

```bash
sudo ufw allow 8081/tcp
sudo ufw reload
```

---

## Persistence & Backup

* MongoDB data is in `~/mongo/mongodb/data`.
* Backup:

```bash
tar -czvf mongo-backup.tar.gz ~/mongo/mongodb/data
```

* Restore:

```bash
tar -xzvf mongo-backup.tar.gz -C ~/mongo/mongodb/data
```

---

## Managing Docker Container

```bash
# Restart
docker restart mongodb

# Stop
docker stop mongodb

# Remove (data persists)
docker rm mongodb
```

---

## Notes & Best Practices

<details>
<summary>Click to expand</summary>

* Warnings like `Soft rlimits for open file descriptors too low` are safe for small projects.
* For production: use XFS filesystem for WiredTiger storage engine.
* Increase `ulimit -n` for high-load servers.
* Keep `--restart unless-stopped` to survive reboots.
* Use strong passwords and firewall rules if exposing ports publicly.
* Mongo Express is convenient but not production-secure — use VPN or reverse proxy with authentication if needed.

</details>

---

✅ **Setup complete!**
Your MongoDB Docker container is **persistent, multi-project ready, auto-restarting**, and optionally accessible via a web GUI.

```
