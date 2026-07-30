# AWS CloudWatch Agent Setup on Windows EC2

## Overview

This document covers the setup and troubleshooting of the **Amazon
CloudWatch Agent on a Windows Server running on AWS EC2**.

The CloudWatch Agent can collect additional operating-system-level
metrics that are not included in standard EC2 monitoring, such as:

-   Memory utilization
-   Disk utilization/free space
-   CPU metrics
-   Network metrics
-   Windows Event Logs
-   Application log files

------------------------------------------------------------------------

# 1. Recommended Architecture

``` text
                 AWS
                  |
             Amazon CloudWatch
              /            \
          Metrics           Logs
             |                |
        CloudWatch Agent   Log Groups
             |
        Windows EC2
             |
     +-------+--------+
     |       |        |
    CPU    Memory    Disk
                      |
                     C:
                     D:
```

For multiple Windows EC2 instances, using **AWS Systems Manager (SSM) +
Parameter Store** is recommended instead of manually configuring every
server.

------------------------------------------------------------------------

# 2. IAM Role Requirements

The Windows EC2 instance should have an IAM role attached.

Recommended policies:

``` text
CloudWatchAgentServerPolicy
AmazonSSMManagedInstanceCore
```

`CloudWatchAgentServerPolicy` allows the CloudWatch Agent to publish
metrics and logs.

`AmazonSSMManagedInstanceCore` is recommended when managing the agent
through AWS Systems Manager.

### Check the IAM Role

AWS Console:

``` text
EC2
 → Instances
 → Select Instance
 → Security
 → IAM Role
```

------------------------------------------------------------------------

# 3. Download CloudWatch Agent

RDP into the Windows EC2 instance and open **PowerShell as
Administrator**.

Run:

``` powershell
cd C:\
```

Download the CloudWatch Agent MSI:

``` powershell
Invoke-WebRequest `
  -Uri "https://amazoncloudwatch-agent.s3.amazonaws.com/windows/amd64/latest/amazon-cloudwatch-agent.msi" `
  -OutFile "amazon-cloudwatch-agent.msi"
```

------------------------------------------------------------------------

# 4. Install CloudWatch Agent

Run:

``` powershell
msiexec /i C:\amazon-cloudwatch-agent.msi
```

The CloudWatch Agent is normally installed under:

``` text
C:\Program Files\Amazon\AmazonCloudWatchAgent\
```

Check the Windows service:

``` powershell
Get-Service AmazonCloudWatchAgent
```

------------------------------------------------------------------------

# 5. Create CloudWatch Agent Configuration

There are two common approaches.

## Option A --- Use the Configuration Wizard

This is recommended for initial setup.

Run:

``` powershell
cd "C:\Program Files\Amazon\AmazonCloudWatchAgent"
```

Then:

``` powershell
.\amazon-cloudwatch-agent-config-wizard.exe
```

The wizard asks which metrics and logs should be collected.

Typical Windows metrics include:

``` text
CPU
Memory
Disk
Network
Processes
```

Typical Windows Event Logs include:

``` text
System
Application
Security
```

Application logs can also be configured, for example:

``` text
C:\App\logs\application.log
C:\App\logs\error.log
```

The wizard commonly creates:

``` text
C:\Program Files\Amazon\AmazonCloudWatchAgent\config.json
```

Verify:

``` powershell
Test-Path "C:\Program Files\Amazon\AmazonCloudWatchAgent\config.json"
```

Expected:

``` text
True
```

------------------------------------------------------------------------

## Option B --- Create JSON Configuration Manually

For a basic CPU, memory, disk, and network setup, create:

``` text
C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json
```

Example:

``` json
{
  "agent": {
    "metrics_collection_interval": 60
  },
  "metrics": {
    "namespace": "CWAgent",
    "metrics_collected": {
      "Processor": {
        "measurement": [
          "% Processor Time"
        ],
        "resources": [
          "*"
        ]
      },
      "Memory": {
        "measurement": [
          "% Committed Bytes In Use"
        ]
      },
      "LogicalDisk": {
        "measurement": [
          "% Free Space"
        ],
        "resources": [
          "*"
        ]
      },
      "Network Interface": {
        "measurement": [
          "Bytes Total/sec"
        ],
        "resources": [
          "*"
        ]
      }
    }
  }
}
```

Save the file exactly as:

``` text
C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json
```

### Important

Make sure Windows Notepad does not save the file as:

``` text
amazon-cloudwatch-agent.json.txt
```

Check the file:

``` powershell
Get-Item "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json"
```

------------------------------------------------------------------------

# 6. Validate the Configuration

If using the configuration file under `ProgramData`, run:

``` powershell
& "C:\Program Files\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent-ctl.ps1" `
-a validate-config `
-m ec2 `
-c file:"C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json"
```

If successful, you should see something similar to:

``` text
Valid Json input schema.
```

------------------------------------------------------------------------

# 7. Start the CloudWatch Agent

If your configuration is located at:

``` text
C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json
```

run:

``` powershell
& "C:\Program Files\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent-ctl.ps1" `
-a fetch-config `
-m ec2 `
-c file:"C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" `
-s
```

If using the wizard-generated file:

``` text
C:\Program Files\Amazon\AmazonCloudWatchAgent\config.json
```

run:

``` powershell
& "C:\Program Files\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent-ctl.ps1" `
-a fetch-config `
-m ec2 `
-c file:"C:\Program Files\Amazon\AmazonCloudWatchAgent\config.json" `
-s
```

------------------------------------------------------------------------

# 8. Check Agent Status

Run:

``` powershell
& "C:\Program Files\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent-ctl.ps1" -a status -m ec2
```

Expected output:

``` json
{
  "status": "running",
  "starttime": "...",
  "configstatus": "configured"
}
```

You can also check the Windows service:

``` powershell
Get-Service AmazonCloudWatchAgent
```

Expected:

``` text
Status   Name
------   ----
Running  AmazonCloudWatchAgent
```

------------------------------------------------------------------------

# 9. Verify Metrics in CloudWatch

Go to:

``` text
AWS Console
 → CloudWatch
 → Metrics
 → All metrics
```

Look for:

``` text
CWAgent
```

Depending on your configuration, metrics may include:

``` text
Processor
Memory
LogicalDisk
Network Interface
```

Example structure:

``` text
CWAgent
 └── InstanceId
      └── i-xxxxxxxxxxxx
           ├── Memory
           ├── Processor
           └── LogicalDisk
```

------------------------------------------------------------------------

# 10. Monitor Windows Disk Utilization

Default EC2 CloudWatch metrics do not provide the same level of Windows
OS disk and memory information that the CloudWatch Agent can provide.

The agent can collect disk metrics such as:

``` text
C: Drive
 └── % Free Space

D: Drive
 └── % Free Space
```

A CloudWatch alarm can then be created, for example:

``` text
C: Drive % Free Space < 15%
           |
           v
    CloudWatch Alarm
           |
           v
          SNS
           |
           v
      Email / Slack
```

------------------------------------------------------------------------

# 11. CloudWatch Agent Logs

If the agent fails to start or metrics are not appearing, check:

``` text
C:\ProgramData\Amazon\AmazonCloudWatchAgent\Logs\
```

The main agent log can be inspected using:

``` powershell
Get-Content "C:\ProgramData\Amazon\AmazonCloudWatchAgent\Logs\amazon-cloudwatch-agent.log" -Tail 100
```

Also check the Windows service:

``` powershell
Get-Service AmazonCloudWatchAgent
```

And the agent status:

``` powershell
& "C:\Program Files\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent-ctl.ps1" -a status -m ec2
```

------------------------------------------------------------------------

# 12. Troubleshooting Encountered

## Error

The following error was encountered:

``` text
Starting config-downloader, this will map back to a call to amazon-cloudwatch-agent
Executing C:\Program Files\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.exe with arguments: [config-downloader -config C:\ProgramData\Amazon\AmazonCloudWatchAgent\common-config.toml -multi-config default -mode ec2 -download-source file:C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json -output-dir C:\ProgramData\Amazon\AmazonCloudWatchAgent\Configs]

I! Trying to detect region from ec2
D! [EC2] Found active network interface
I! imds retry client will retry 1 times

2026/07/30 09:25:05 E! fail to fetch/remove json config:
open C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json:
The system cannot find the file specified.

E! config-downloader process exited with non-zero status: 1
```

## Root Cause

The CloudWatch Agent was trying to load:

``` text
C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json
```

but that file did not exist.

The key error is:

``` text
The system cannot find the file specified.
```

This means the immediate problem is the missing configuration file, not
necessarily an EC2 networking problem.

------------------------------------------------------------------------

# 13. Troubleshooting Steps

## Step 1 --- Check whether the file exists

Run:

``` powershell
Test-Path "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json"
```

If the result is:

``` text
False
```

the configuration file is missing.

------------------------------------------------------------------------

## Step 2 --- Create Configuration

Either manually create:

``` text
C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json
```

or use the configuration wizard:

``` powershell
cd "C:\Program Files\Amazon\AmazonCloudWatchAgent"

.\amazon-cloudwatch-agent-config-wizard.exe
```

If the wizard creates:

``` text
C:\Program Files\Amazon\AmazonCloudWatchAgent\config.json
```

use that file when starting the agent.

------------------------------------------------------------------------

## Step 3 --- Start Using the Correct Configuration

For a manually created configuration:

``` powershell
& "C:\Program Files\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent-ctl.ps1" `
-a fetch-config `
-m ec2 `
-c file:"C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" `
-s
```

For the wizard-generated configuration:

``` powershell
& "C:\Program Files\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent-ctl.ps1" `
-a fetch-config `
-m ec2 `
-c file:"C:\Program Files\Amazon\AmazonCloudWatchAgent\config.json" `
-s
```

------------------------------------------------------------------------

# 14. Important Difference Between Configuration Locations

The troubleshooting issue occurred because the agent was instructed to
use:

``` text
C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json
```

but the wizard may instead create:

``` text
C:\Program Files\Amazon\AmazonCloudWatchAgent\config.json
```

These are different files.

If the configuration exists at:

``` text
C:\Program Files\Amazon\AmazonCloudWatchAgent\config.json
```

do not continue using:

``` text
C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json
```

unless you create that file there.

Instead, explicitly point the agent to the actual configuration:

``` powershell
-c file:"C:\Program Files\Amazon\AmazonCloudWatchAgent\config.json"
```

------------------------------------------------------------------------

# 15. Recommended Monitoring Configuration

For a production Windows EC2 server, a useful initial monitoring
configuration is:

## System Metrics

``` text
CPU
Memory
C: Disk
D: Disk
Network
```

## Windows Logs

``` text
System
Application
Security
```

## Application Logs

If applicable:

``` text
C:\Application\logs\application.log
C:\Application\logs\error.log
```

## Suggested CloudWatch Alarms

``` text
CPU > 80%
Memory > 80%
C: Disk Free Space < 15%
D: Disk Free Space < 15%
Status Check Failed
```

These alarms can send notifications through SNS.

------------------------------------------------------------------------

# 16. Production Recommendation

For a single Windows EC2 server, local JSON configuration is acceptable.

For multiple Windows EC2 servers, consider:

``` text
AWS Systems Manager
        |
        v
Parameter Store
        |
        v
CloudWatch Agent Configuration
        |
        v
Multiple Windows EC2 Instances
```

This avoids manually maintaining separate configuration files on every
server.

------------------------------------------------------------------------

# 17. Quick Command Reference

### Check agent service

``` powershell
Get-Service AmazonCloudWatchAgent
```

### Check configuration file

``` powershell
Test-Path "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json"
```

### Run configuration wizard

``` powershell
cd "C:\Program Files\Amazon\AmazonCloudWatchAgent"

.\amazon-cloudwatch-agent-config-wizard.exe
```

### Validate configuration

``` powershell
& "C:\Program Files\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent-ctl.ps1" `
-a validate-config `
-m ec2 `
-c file:"C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json"
```

### Start agent

``` powershell
& "C:\Program Files\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent-ctl.ps1" `
-a fetch-config `
-m ec2 `
-c file:"C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" `
-s
```

### Check status

``` powershell
& "C:\Program Files\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent-ctl.ps1" -a status -m ec2
```

### Check logs

``` powershell
Get-Content "C:\ProgramData\Amazon\AmazonCloudWatchAgent\Logs\amazon-cloudwatch-agent.log" -Tail 100
```

------------------------------------------------------------------------

# 18. Final Checklist

Before considering the setup complete, verify:

-   [ ] CloudWatch Agent installed
-   [ ] EC2 IAM role attached
-   [ ] `CloudWatchAgentServerPolicy` attached
-   [ ] Configuration JSON created
-   [ ] Configuration path is correct
-   [ ] Configuration validated successfully
-   [ ] CloudWatch Agent service is running
-   [ ] Agent status shows `running`
-   [ ] Agent status shows `configured`
-   [ ] `CWAgent` metrics are visible in CloudWatch
-   [ ] Memory metrics are visible
-   [ ] Disk metrics are visible
-   [ ] Required Windows logs are visible
-   [ ] CloudWatch alarms are configured if required

------------------------------------------------------------------------

## Summary

The specific error encountered was caused by a missing configuration
file:

``` text
C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json
```

The CloudWatch Agent installation itself was not necessarily broken.

The correct approach is:

``` text
Install Agent
      |
      v
Create Configuration
      |
      v
Validate Configuration
      |
      v
Start Agent Using Correct Config Path
      |
      v
Check Agent Status
      |
      v
Verify CWAgent Metrics
```

For the encountered error, the most important action is to ensure the
configuration file exists and that the `-c file:` parameter points to
the actual file.
