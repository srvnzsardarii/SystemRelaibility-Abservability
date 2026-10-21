# Percona Monitoring and Management(PMM) on Ubuntu
A:step by step guide to installing **Percona Monitoring and Management(PMM)** on Ubuntu Linux and using it to monitor opensource databases (MySQL, PostgreSQL, MongoDB, and others).
**Note:** Commands target PMM 3.x. Always verify against the official docs before production use: https://docs.percona.com/percona-monitoring-and-management/
## Table of Contents
1.[Overview and Architecture](#1-overview and architecture)
2.[Prerequisites](#2-prerequisites)
3.[Install PMM Server (Docker)](#3-install pmm server docker)
4.[First Login and Security Hardening](#4-first login and security hardening)
5.[Install PMM Client on Database Hosts](#5-install pmm client on database hosts)
6.[Connect the Client to the Server](#6-connect the client to the server)
7.[Add Databases to Monitoring](#7-add databases to monitoring)
8.[Verification](#8-verification)
9.[Backup,Upgrade and Maintenance](#9-backup upgrade and maintenance)
10.[Troubleshooting](#10-troubleshooting)
11.[Uninstall](#11-uninstall)
12.[References](#12-references)

# 1.Overview and Architecture
PMM has two components:
| Component | Where it runs | Role |
|-----------|---------------|------|
| PMM Server|One dedicated host(or container)|Stores metrics,serves the web UI(Grafana dashboards,Query Analytics,alerting)|
| PMM Client(`pmm-admin`,`pmm-agent`)|Every host running a database|Collects metrics and query data and sends them to the server|

```
 +-----------------+        +-----------------+
 | DB host 1       |        | DB host 2       |
 | MySQL           |        | PostgreSQL      |
 | pmm-client      |        | pmm-client      |
 +--------+--------+        +--------+--------+
          |   HTTPS (443)            |
          +------------+-------------+
                       v
              +------------------+
              | PMM Server       |
              | (Docker)         |
              | Web UI :443      |
              +------------------+
```

## 2.Prerequisites
**PMM Server host**
- Ubuntu 22.04 LTS or 24.04 LTS(64bit)
- Minimum 2 CPU cores,4 GB RAM,20 GB disk(scale disk with number of monitored services and retention)
- Docker Engine installed
- Inbound TCP **443** open from your admin network and from all DB hosts
**PMM Client hosts(database servers)**
- Ubuntu with a running database(MySQL,Percona Server,MariaDB,PostgreSQL,MongoDB, ...)
- Outbound access to the PMM Server on TCP **443**
- `sudo` access
**Install Docker(PMM Server host):**
```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
curl -fsSL https://get.docker.com | sudo sh
sudo systemctl enable --now docker
docker --version

```

## 3.Install PMM Server(Docker)
```bash
#Persistent volume for all PMM data
docker volume create pmm-data
#Run PMM Server
docker run --detach \
  --restart always \
  --publish 443:8443 \
  --volume pmm-data:/srv \
  --name pmm-server \
  percona/pmm-server:3
```
Check that it is healthy:
```bash
docker ps --filter name=pmm-server
docker logs pmm-server --tail 50
```
Open the firewall if `ufw` is active:
```bash
sudo ufw allow 443/tcp
```

## 4.First Login and Security Hardening
1.Browse to `https://<PMM_SERVER_IP>`(accept the self signed certificate warning for now).
2.Default credentials:`admin` / `admin`.You will be prompted to **change the password immediately.
3.Recommended hardening:
   - Replace the self signed certificate with a trusted one(reverse proxy such as Nginx or mount certs into the container).
   - Restrict port 443 to trusted networks only.
   - Create a Service Account+token for client registration(Administration > Users and access > Service Accounts,role *Admin*).Use this token instead of the admin password when registering clients.

## 5.Install PMM Client on Database Hosts
Run on each database host:
```bash
wget https://repo.percona.com/apt/percona-release_latest.generic_all.deb
sudo dpkg -i percona-release_latest.generic_all.deb
sudo apt update
sudo percona-release enable pmm3-client release
sudo apt install -y pmm-client
pmm-admin --version
```

## 6.Connect the Client to the Server
Using a service account token (recommended):
```bash
sudo pmm-admin config \
  --server-insecure-tls \
  --server-url=https://service_token:<YOUR_TOKEN>@<PMM_SERVER_IP>:443
```
Remove `--server-insecure-tls` once the server uses a trusted certificate.
Verify:
```bash
pmm-admin status
```

## 7.Add Databases to Monitoring
## 7.1.MySQL/Percona Server/MariaDB
Enable Performance Schema(`/etc/mysql/mysql.conf.d/mysqld.cnf`):
```ini
[mysqld]
performance_schema = ON
```
Restart MySQL,then create a dedicated monitoring user:
```sql
CREATE USER 'pmm'@'localhost' IDENTIFIED BY 'StrongPassword#1'
  WITH MAX_USER_CONNECTIONS 10;
GRANT SELECT, PROCESS, REPLICATION CLIENT, RELOAD, BACKUP_ADMIN
  ON *.* TO 'pmm'@'localhost';
FLUSH PRIVILEGES;
```
> On MariaDB or MySQL 5.7,`BACKUP_ADMIN` may not exist;
Register the service:
```bash
sudo pmm-admin add mysql \
  --username=pmm \
  --password='StrongPassword#1' \
  --query-source=perfschema \
  --host=127.0.0.1 --port=3306 \
  mysql-prod-01
```
##7.2.PostgreSQL
Enable `pg_stat_statements` in `postgresql.conf`:
```ini
shared_preload_libraries = 'pg_stat_statements'
track_io_timing = on
```
Restart PostgreSQL,then:
```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
CREATE USER pmm WITH SUPERUSER ENCRYPTED PASSWORD 'StrongPassword#1';
```
Ensure `pg_hba.conf` allows the `pmm` user locally (e.g.`local all pmm scram-sha-256` or `host all pmm 127.0.0.1/32 scram-sha-256`),then reload PostgreSQL.
Register the service:
```bash
sudo pmm-admin add postgresql \
  --username=pmm \
  --password='StrongPassword#1' \
  --query-source=pgstatements \
  --host=127.0.0.1 --port=5432 \
  pgsql-prod-01
```
##7.3 MongoDB (optional)
```bash
sudo pmm-admin add mongodb \
  --username=pmm --password='StrongPassword#1' \
  --host=127.0.0.1 --port=27017 \
  mongo-prod-01
```
(The MongoDB `pmm` user needs the `clusterMonitor` role and read access to `local`;(see the official docs for the exact role set.))

## 8.Verification
On the client host:
```bash
pmm-admin list     # services registered
pmm-admin status   # agent connected to server
```
In the web UI:
-Home Dashboard=shows the node and services.
-Dashboards>MySQL/PostgreSQL=show live metrics.
-Query Analytics(QAN)=shows top queries(may take a few minutes).

## 9.Backup,Upgrade and Maintenance
Backup the PMM Server volume:
```bash
docker stop pmm-server
docker run --rm -v pmm-data:/srv -v $(pwd):/backup ubuntu \
  tar czf /backup/pmm-data-$(date +%F).tar.gz /srv
docker start pmm-server
```
Upgrade PMM Server(back up first):use the in UI upgrade where available or pull the new image and recreate the container with the same `pmm-data` volume:
```bash
docker pull percona/pmm-server:3
docker stop pmm-server && docker rename pmm-server pmm-server-old
docker run --detach --restart always --publish 443:8443 \
  --volume pmm-data:/srv --name pmm-server percona/pmm-server:3
```
Upgrade PMM Client:
```bash
sudo apt update && sudo apt install --only-upgrade pmm-client
```
Keep client and server on compatible versions(server should be the same or newer than the client).

## 10. Troubleshooting
| Symptom | Check |
|---------|-------|
|`pmm-admin status` cannot connect|Firewall/security group on 443;correct server IP;`--server-insecure-tls` for self-signed certs|
|No MySQL query data|`performance_schema=ON`;user privileges;`--query-source=perfschema`|
|No PostgreSQL query data|`pg_stat_statements` in `shared_preload_libraries` and extension created|
|Service shows as down|`pmm-admin list`;`journalctl -u pmm-agent -f`;DB user credentials|
|Server UI unreachable|`docker ps`,`docker logs pmm-server`|
|Disk filling up|Reduce retention in Settings>Advanced Settings;expand volume|
Client logs:`sudo journalctl -u pmm-agent -n 100`

## 11.Uninstall
```bash
#Remove a monitored service
sudo pmm-admin remove mysql mysql-prod-01
#Remove the client
sudo pmm-admin unregister --force
sudo apt remove --purge -y pmm-client
#Remove the server (DELETES ALL DATA)
docker stop pmm-server && docker rm pmm-server
docker volume rm pmm-data
```

## 12. References
-PMM documentation:https://docs.percona.com/percona-monitoring-and-management/
-PMM Docker install:https://docs.percona.com/percona-monitoring-and-management/3/install-pmm/install-pmm-server/deployment-options/docker/index.html
-Percona repositories:https://docs.percona.com/percona-software-repositories/

Maintainer: Sarvenaz Sardari
