# Zabbix Monitoring

## 1. What is Zabbix?

Zabbix is an **open-source IT infrastructure monitoring platform** used to monitor the health, performance, and availability of servers, networks, applications, databases, virtual machines, cloud services, and websites.

It helps system administrators detect problems early and receive alerts before they impact users.

### What Zabbix Can Monitor

Zabbix can monitor:

- CPU utilization
- Memory utilization
- Disk utilization and disk I/O
- Network traffic and availability
- Server uptime and availability
- Application performance
- Databases such as MySQL and PostgreSQL
- Network devices such as routers, switches, and firewalls
- Virtual machines
- Cloud infrastructure
- Website availability and response time
- Custom metrics through scripts and APIs

### Main Components of Zabbix

A typical Zabbix deployment consists of:

#### 1. Zabbix Server

The Zabbix Server is the central component. It:

- Collects monitoring data
- Evaluates triggers
- Processes alerts
- Communicates with agents and proxies

#### 2. Database

Zabbix stores configuration, monitoring data, events, and historical information in a database.

Common database options include:

- MySQL
- PostgreSQL

#### 3. Zabbix Web Interface

The web interface provides:

- Dashboards
- Graphs
- Monitoring views
- Alerts
- Configuration
- Reports

#### 4. Zabbix Agent

The Zabbix Agent runs on monitored servers and collects detailed system-level metrics.

For example:

```text
CPU usage
Memory usage
Disk usage
Network usage
Processes
Services
```

#### 5. Zabbix Proxy

A Zabbix Proxy can be used when monitoring remote environments.

The proxy collects monitoring information and forwards it to the Zabbix Server.

### Key Features

- Real-time monitoring
- Custom dashboards
- Graphs and historical data
- Email and webhook notifications
- Automatic network discovery
- Template-based monitoring
- Historical data and trend analysis
- REST API
- Script-based custom monitoring
- Monitoring of on-premises and cloud infrastructure

### Example

Suppose you have a Linux web server.

Zabbix can monitor:

```text
CPU          → 75%
Memory       → 65%
Disk         → 80%
Network      → Normal
Web Service  → Running
```

You can configure a trigger such as:

```text
Disk usage > 90%
```

When the condition is met, Zabbix can generate an alert:

```text
WARNING: Disk usage on server01 is above 90%
```

### Zabbix vs Prometheus

| Zabbix | Prometheus |
|---|---|
| Full-stack infrastructure monitoring | Metrics-focused monitoring |
| Built-in dashboards and alerting | Commonly paired with Grafana |
| Agent-based and agentless monitoring | Primarily pull-based metrics collection |
| Strong for traditional IT infrastructure | Very popular for Kubernetes/cloud-native environments |
| Template-based monitoring | Exporter-based monitoring |

---

# 2. Can Zabbix Monitor MySQL Replication Running in Docker Containers?

**Yes. Zabbix can monitor MySQL replication running inside Docker containers.**

Zabbix can monitor MySQL replication status, replication lag, replication threads, and replication errors.

## What Can Be Monitored?

For a MySQL Primary → Replica setup, Zabbix can monitor:

- MySQL availability
- Replication status
- IO/receiver thread status
- SQL/applier thread status
- Replication lag
- `Seconds_Behind_Source`
- Binary log position
- Relay log position
- Replication errors
- GTID status
- Connections
- Queries
- CPU and memory usage

## Checking MySQL Replication Status

For modern MySQL versions, use:

```sql
SHOW REPLICA STATUS\G
```

For older MySQL versions:

```sql
SHOW SLAVE STATUS\G
```

Example output:

```text
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
Seconds_Behind_Source: 0
Last_IO_Error:
Last_SQL_Error:
```

The exact field names can vary depending on the MySQL version and terminology being used.

---

# 3. Zabbix + MySQL Replication + Docker Architecture

A typical architecture can look like this:

```text
                 Docker Host
              ┌─────────────────┐
              │                 │
              │ MySQL Primary   │
              │                 │
              │ MySQL Replica   │
              │                 │
              └────────┬────────┘
                       │
                 Zabbix Agent
                       │
                       ↓
                  Zabbix Server
                       │
                       ↓
                 Zabbix Dashboard
```

The Zabbix Agent can execute scripts or commands that connect to the MySQL container and retrieve replication information.

---

# 4. Recommended Approach for Docker

There are two possible approaches.

## Option 1 — Install Zabbix Agent Inside the MySQL Container

It is technically possible to install the Zabbix Agent inside the MySQL container.

However, this is generally **not recommended** because containers should ideally run a single primary application/process.

Installing monitoring agents inside application containers can also make container images and lifecycle management more complicated.

## Option 2 — Install Zabbix Agent on the Docker Host

This is generally the cleaner approach.

Architecture:

```text
                  Docker Host
        ┌────────────────────────────┐
        │                            │
        │   MySQL Primary Container  │
        │                            │
        │   MySQL Replica Container  │
        │                            │
        │   Zabbix Agent             │
        │                            │
        └──────────────┬─────────────┘
                       │
                       ↓
                  Zabbix Server
```

The Zabbix Agent can execute a script that runs a MySQL command inside the container.

For example:

```bash
docker exec mysql-replica mysql -uroot -p'PASSWORD' -e "SHOW REPLICA STATUS\G"
```

The output can then be processed by a script and returned to Zabbix.

---

# 5. Important MySQL Replication Metrics

The following metrics are especially important.

## Replica IO/Receiver Thread

Example:

```text
Replica_IO_Running: Yes
```

If it becomes `No`, the replica may no longer be receiving binary log events from the primary.

Possible Zabbix trigger:

```text
Replica_IO_Running != Yes
```

Alert:

```text
CRITICAL: MySQL replication IO thread stopped
```

## Replica SQL/Applier Thread

Example:

```text
Replica_SQL_Running: Yes
```

If it becomes `No`, SQL events may no longer be applied on the replica.

Possible trigger:

```text
Replica_SQL_Running != Yes
```

Alert:

```text
CRITICAL: MySQL replication SQL thread stopped
```

## Replication Lag

Example:

```text
Seconds_Behind_Source: 0
```

This indicates that the replica is currently caught up with the source, subject to the limitations of this metric.

Example trigger:

```text
Seconds_Behind_Source > 300
```

Alert:

```text
WARNING: MySQL replication lag is greater than 5 minutes
```

## Replication Errors

Important fields include:

```text
Last_IO_Error
Last_SQL_Error
```

If these contain an error, Zabbix can generate an alert.

Example:

```text
Last_SQL_Error != ""
```

Alert:

```text
CRITICAL: MySQL replication SQL error detected
```

---

# 6. Example Monitoring Flow

A typical monitoring flow can be:

```text
MySQL Replica Container
        │
        │ SHOW REPLICA STATUS
        ↓
Monitoring Script
        │
        ↓
Zabbix Agent
        │
        ↓
Zabbix Server
        │
        ├── Replication Status
        ├── Replication Lag
        ├── IO Thread
        ├── SQL Thread
        └── Replication Errors
        │
        ↓
Zabbix Trigger
        │
        ↓
Email / Webhook / Notification
```

---

# 7. Example Alerts

### Replication IO Thread Failure

```text
Replica_IO_Running = No
```

Alert:

```text
CRITICAL: MySQL Replica IO thread is stopped
```

### Replication SQL Thread Failure

```text
Replica_SQL_Running = No
```

Alert:

```text
CRITICAL: MySQL Replica SQL thread is stopped
```

### High Replication Lag

```text
Seconds_Behind_Source > 300
```

Alert:

```text
WARNING: MySQL replication lag is greater than 5 minutes
```

### Replication SQL Error

```text
Last_SQL_Error != ""
```

Alert:

```text
CRITICAL: MySQL replication SQL error detected
```

---

# 8. Recommended Setup

For a Docker-based MySQL replication environment, a practical setup would be:

```text
                    Zabbix Server
                          │
                          │
                    Zabbix Agent
                          │
                    Docker Host
                          │
             ┌────────────┴────────────┐
             │                         │
      MySQL Primary              MySQL Replica
             │                         │
             └────── Replication ──────┘
```

Recommended approach:

1. Install Zabbix Agent on the Docker host.
2. Keep MySQL containers focused on running MySQL.
3. Create a monitoring script for MySQL replication.
4. Use `SHOW REPLICA STATUS\G` to retrieve replication status.
5. Configure Zabbix items for important metrics.
6. Configure triggers for replication failures and lag.
7. Configure email, Slack, Teams, or webhook notifications as required.
8. Create a Zabbix dashboard showing MySQL replication health.

---

# 9. Conclusion

**Zabbix can effectively monitor MySQL replication running inside Docker containers.**

The most important metrics to monitor are:

```text
Replication availability
        ↓
IO/Receiver Thread
        ↓
SQL/Applier Thread
        ↓
Replication Lag
        ↓
Replication Errors
```

For Docker environments, installing the **Zabbix Agent on the Docker host** and using scripts to query the MySQL container is generally a clean and maintainable approach.

For larger environments, Zabbix's MySQL monitoring templates can also be used as a starting point and customized for replication-specific monitoring.
