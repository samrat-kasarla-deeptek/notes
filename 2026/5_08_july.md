To install and configure the **Amazon CloudWatch Agent on an Ubuntu EC2 instance**, follow these steps.

## 1. Connect to your EC2 Ubuntu instance

```bash
ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>
```

---

## 2. Update packages

```bash
sudo apt update
sudo apt upgrade -y
```

---

## 3. Download CloudWatch Agent package

For Ubuntu (64-bit):

```bash
wget https://amazoncloudwatch-agent.s3.amazonaws.com/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
```

---

## 4. Install CloudWatch Agent

```bash
sudo dpkg -i amazon-cloudwatch-agent.deb
```

Verify installation:

```bash
/opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
```

---

# 5. Attach IAM Role to EC2 Instance

The EC2 instance needs permissions to send metrics and logs.

Create an IAM Role with:

CloudWatchAgentServerPolicy

Attach this role to your EC2 instance:

```
EC2 Console
   |
   EC2 Instance
   |
Actions
   |
Security
   |
Modify IAM Role
   |
Select CloudWatch Role
```

---

# 6. Create CloudWatch Agent Configuration

You can create it manually.

Create file:

```bash
sudo vi /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

Example configuration:

```json
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "root"
  },
  "metrics": {
    "namespace": "CWAgent",
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}",
      "ImageId": "${aws:ImageId}",
      "InstanceType": "${aws:InstanceType}"
    },
    "metrics_collected": {
      "mem": {
        "measurement": [
          "mem_used_percent"
        ]
      },
      "disk": {
        "measurement": [
          "used_percent"
        ],
        "resources": [
          "*"
        ]
      }
    }
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/syslog",
            "log_group_name": "/ec2/ubuntu/syslog",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  }
}
```

This collects:

* Memory utilization
* Disk utilization
* Ubuntu system logs

---

# 7. Start CloudWatch Agent

Run:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
-a fetch-config \
-m ec2 \
-c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json \
-s
```

---

# 8. Check Agent Status

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
```

Expected:

```json
{
  "status": "running",
  "starttime": "2026-07-08T..."
}
```

---

# 9. Check Agent Logs

If metrics are not appearing:

```bash
sudo tail -f /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```

---

# 10. Verify Metrics in AWS Console

Go to:

```
AWS Console
   |
CloudWatch
   |
Metrics
   |
All metrics
   |
CWAgent
```

You should see:

```
CWAgent
 └── InstanceId
       ├── mem_used_percent
       └── disk_used_percent
```

---

# 11. Enable Agent After Reboot

The agent service is enabled automatically when started with `-s`, but you can verify:

```bash
sudo systemctl status amazon-cloudwatch-agent
```

Enable manually:

```bash
sudo systemctl enable amazon-cloudwatch-agent
```

---

## Common Interview Follow-up

**Q: Why CPU metrics are visible without installing CloudWatch Agent but memory is not?**

**Answer:**

EC2 sends basic metrics (CPU, Network, Disk IO) automatically through the hypervisor. Memory utilization is inside the operating system, so CloudWatch needs the CloudWatch Agent to collect it.

---

For a production EC2 environment, I would normally configure:

* Memory
* Disk usage
* Swap
* Process monitoring
* Application logs
* Nginx/Apache logs
* Custom application metrics

using a centralized CloudWatch Agent configuration managed through **AWS Systems Manager Parameter Store**.
