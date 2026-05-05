Got it 👍 — here is the **.md file with FULL detailed content of first 4 questions** (no trimming, proper structure, ready to save):

---

```md
# MySQL Master-Slave Replication (Full Setup + POC + SOURCE_HOST + Auth Fix)

---

## 1️⃣ Setup MySQL Master-Slave from Scratch (Docker)

Below is a from-scratch MySQL master–slave replication setup using Docker containers, with steps to promote slave as new master during failure.

### Assumption

```

Server 1 = Master DB (10.0.0.10)
Server 2 = Slave DB  (10.0.0.20)
MySQL version = 8.0

````

---

## Step 1: Create MySQL Master Container

```bash
mkdir -p /opt/mysql-master/data
cd /opt/mysql-master
````

### docker-compose.yml

```yaml
version: "3.8"

services:
  mysql-master:
    image: mysql:8.0
    container_name: mysql-master
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: augmento
    ports:
      - "3306:3306"
    volumes:
      - ./data:/var/lib/mysql
      - ./my.cnf:/etc/mysql/conf.d/my.cnf
```

### my.cnf

```ini
[mysqld]
server-id=1
log-bin=mysql-bin
binlog_format=ROW
bind-address=0.0.0.0
```

Start container:

```bash
docker compose up -d
```

---

## Step 2: Create Replication User

```sql
CREATE USER 'replica_user'@'%' IDENTIFIED BY 'ReplicaPass@123';
GRANT REPLICATION SLAVE ON *.* TO 'replica_user'@'%';
FLUSH PRIVILEGES;
```

---

## Step 3: Get Master Status

```sql
SHOW MASTER STATUS;
```

Example:

```
File: mysql-bin.000003
Position: 157
```

---

## Step 4: Create Slave Container

```bash
mkdir -p /opt/mysql-slave/data
cd /opt/mysql-slave
```

### docker-compose.yml

```yaml
version: "3.8"

services:
  mysql-slave:
    image: mysql:8.0
    container_name: mysql-slave
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: augmento
    ports:
      - "3306:3306"
    volumes:
      - ./data:/var/lib/mysql
      - ./my.cnf:/etc/mysql/conf.d/my.cnf
```

### my.cnf

```ini
[mysqld]
server-id=2
relay-log=mysql-relay-bin
read_only=ON
super_read_only=ON
bind-address=0.0.0.0
```

Start:

```bash
docker compose up -d
```

---

## Step 5: Configure Replication

```sql
CHANGE REPLICATION SOURCE TO
SOURCE_HOST='10.0.0.10',
SOURCE_PORT=3306,
SOURCE_USER='replica_user',
SOURCE_PASSWORD='ReplicaPass@123',
SOURCE_LOG_FILE='mysql-bin.000003',
SOURCE_LOG_POS=157;

START REPLICA;
```

---

## Step 6: Failover (Slave → Master)

```sql
STOP REPLICA;
RESET REPLICA ALL;

SET GLOBAL read_only = OFF;
SET GLOBAL super_read_only = OFF;
```

Update config:

```ini
log-bin=mysql-bin
```

Restart:

```bash
docker restart mysql-slave
```

Now slave becomes master.

---

## Step 7: Reconfigure Old Master as Slave

```sql
CHANGE REPLICATION SOURCE TO
SOURCE_HOST='10.0.0.20',
SOURCE_USER='replica_user',
SOURCE_PASSWORD='ReplicaPass@123',
SOURCE_AUTO_POSITION=1;

START REPLICA;
```

---

## 2️⃣ Can we do POC on Same Server?

Yes, you can do the POC on the same server using two MySQL containers.

Example:

```
mysql-master → port 3307
mysql-slave  → port 3308
```

### Connect

```bash
mysql -h 127.0.0.1 -P 3307 -uroot -p
mysql -h 127.0.0.1 -P 3308 -uroot -p
```

### Important Note

* This is only for testing
* No high availability
* If server fails → both containers down

---

## 3️⃣ How to Set SOURCE_HOST

### Option 1 (Best): Docker Network

```bash
docker network create mysql-net
```

### Master compose

```yaml
services:
  mysql-master:
    image: mysql:8.0
    container_name: mysql-master
    networks:
      - mysql-net
    ports:
      - "3307:3306"
```

### Slave compose

```yaml
services:
  mysql-slave:
    image: mysql:8.0
    container_name: mysql-slave
    networks:
      - mysql-net
    ports:
      - "3308:3306"

networks:
  mysql-net:
    external: true
```

### Use container name

```sql
SOURCE_HOST='mysql-master'
```

---

### Option 2: Host IP

```sql
SOURCE_HOST='192.168.1.10'
SOURCE_PORT=3307
```

---

## ❌ Common Mistakes

```
SOURCE_HOST='localhost' ❌
SOURCE_HOST='127.0.0.1' ❌
```

---

## ✅ Debug

```bash
docker exec -it mysql-slave ping mysql-master
docker exec -it mysql-slave mysql -h mysql-master -uroot -p
```

---

## 4️⃣ Authentication Error Fix

### Error

```
Authentication plugin 'caching_sha2_password' reported error: Authentication requires secure connection
```

---

### Root Cause

* MySQL 8 uses `caching_sha2_password`
* Replication requires secure connection OR different auth plugin

---

### Fix

```sql
DROP USER IF EXISTS 'replica_user'@'%';

CREATE USER 'replica_user'@'%'
IDENTIFIED WITH mysql_native_password BY 'ReplicaPass@123';

GRANT REPLICATION SLAVE ON *.* TO 'replica_user'@'%';

FLUSH PRIVILEGES;
```

---

### Reconfigure Slave

```sql
STOP REPLICA;
RESET REPLICA ALL;

CHANGE REPLICATION SOURCE TO
SOURCE_HOST='mysql-master',
SOURCE_PORT=3306,
SOURCE_USER='replica_user',
SOURCE_PASSWORD='ReplicaPass@123',
SOURCE_LOG_FILE='mysql-bin.000003',
SOURCE_LOG_POS=157;

START REPLICA;
```

---

### Verify

```sql
SHOW REPLICA STATUS\G
```

Expected:

```
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
```

---

## ✅ Final Summary

* Master → writes data
* Slave → replicates data
* Use container name for SOURCE_HOST
* Use mysql_native_password for replication user
* Same server setup only for POC
* Promote slave manually during failure

---

```

---

### 👍 Done

You now have:
- Full content of first 4 questions  
- No missing steps  
- Clean `.md` file  

---

If you want next:

👉 I can generate **failover testing steps (real simulation)**  
👉 or **auto failover setup (Orchestrator / ProxySQL)**
```
