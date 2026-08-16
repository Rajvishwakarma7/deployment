# AWS CloudWatch Production Monitoring Setup

> Reusable guide for setting up EC2 monitoring, CloudWatch Agent
> metrics, Docker logs, and a CloudWatch dashboard.
>
> **This document intentionally uses generic project names so it can be
> reused for any AWS project.**

------------------------------------------------------------------------

## 1. Project-Specific Values

Replace these examples for each project:

``` text
Project Name: ProductionApp
Log Group: /production-app/docker
Dashboard: ProductionApp-Production
AWS Region: ap-south-1
```

For another project, for example:

``` text
Project Name: EcommerceAPI
Log Group: /ecommerce-api/docker
Dashboard: EcommerceAPI-Production
```

------------------------------------------------------------------------

## 2. Architecture

``` text
EC2 Server
   |
   +-- CPU ----------------------> EC2 CloudWatch Metrics
   |
   +-- NetworkIn / NetworkOut ---> EC2 CloudWatch Metrics
   |
   +-- Memory -------------------> CloudWatch Agent -> CWAgent
   |
   +-- Disk ---------------------> CloudWatch Agent -> CWAgent
   |
   +-- Docker Logs --------------> CloudWatch Agent
                                      |
                                      v
                              /production-app/docker
                                      |
                                      v
                                Logs Insights
                                      |
                                      v
                             Production Dashboard
```

------------------------------------------------------------------------

## 3. AWS Region

Use the same AWS region as the EC2 instance.

Example:

``` text
ap-south-1
Asia Pacific (Mumbai)
```

Always select the correct region in the AWS Console before checking
metrics or logs.

------------------------------------------------------------------------

## 4. EC2 IAM Permission

The EC2 instance needs permission to send metrics and logs to
CloudWatch.

Attach:

``` text
CloudWatchAgentServerPolicy
```

### Console

``` text
AWS Console
  ↓
EC2
  ↓
Instances
  ↓
Select EC2
  ↓
Security
  ↓
IAM Role
```

Add:

``` text
CloudWatchAgentServerPolicy
```

------------------------------------------------------------------------

## 5. SSH Into EC2

``` bash
ssh -i your-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

------------------------------------------------------------------------

## 6. Update Ubuntu

``` bash
sudo apt update
```

Optional:

``` bash
sudo apt upgrade -y
```

------------------------------------------------------------------------

## 7. Install CloudWatch Agent

Download:

``` bash
wget https://amazoncloudwatch-agent.s3.amazonaws.com/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
```

Install:

``` bash
sudo dpkg -i -E ./amazon-cloudwatch-agent.deb
```

Verify:

``` bash
ls /opt/aws/amazon-cloudwatch-agent/bin/
```

------------------------------------------------------------------------

## 8. Create CloudWatch Agent Configuration

Create the directory:

``` bash
sudo mkdir -p /opt/aws/amazon-cloudwatch-agent/etc
```

Open:

``` bash
sudo nano /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

Use:

``` json
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "root"
  },
  "metrics": {
    "namespace": "CWAgent",
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}"
    },
    "metrics_collected": {
      "mem": {
        "measurement": [
          "mem_used_percent"
        ],
        "metrics_collection_interval": 60
      },
      "disk": {
        "measurement": [
          "used_percent"
        ],
        "resources": [
          "/"
        ],
        "ignore_file_system_types": [
          "sysfs",
          "devtmpfs",
          "tmpfs",
          "devpts",
          "proc",
          "squashfs",
          "overlay"
        ],
        "metrics_collection_interval": 60
      }
    }
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/lib/docker/containers/*/*.log",
            "log_group_name": "/production-app/docker",
            "log_stream_name": "{instance_id}",
            "timezone": "UTC"
          }
        ]
      }
    }
  }
}
```

For another project, change:

``` json
"log_group_name": "/production-app/docker"
```

to:

``` text
/your-project/docker
```

Save:

``` text
Ctrl + O
Enter
Ctrl + X
```

------------------------------------------------------------------------

## 9. Docker Log Location

Docker normally stores JSON logs under:

``` text
/var/lib/docker/containers/
```

The CloudWatch Agent collects:

``` text
/var/lib/docker/containers/*/*.log
```

Example CloudWatch log group:

``` text
/production-app/docker
```

Example stream:

``` text
i-0123456789abcdef0
```

------------------------------------------------------------------------

## 10. Start CloudWatch Agent

``` bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl   -a fetch-config   -m ec2   -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json   -s
```

------------------------------------------------------------------------

## 11. Check Agent Status

``` bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl   -a status
```

Also:

``` bash
sudo systemctl status amazon-cloudwatch-agent
```

------------------------------------------------------------------------

## 12. Enable Agent After Reboot

``` bash
sudo systemctl enable amazon-cloudwatch-agent
```

------------------------------------------------------------------------

## 13. CloudWatch Agent Logs

Follow the agent log:

``` bash
sudo tail -f /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```

Check errors:

``` bash
sudo grep -i error /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```

Restart if necessary:

``` bash
sudo systemctl restart amazon-cloudwatch-agent
```

------------------------------------------------------------------------

# 14. Create CloudWatch Log Group

Go to:

``` text
CloudWatch
  ↓
Logs
  ↓
Log groups
  ↓
Create log group
```

Create:

``` text
/production-app/docker
```

Recommended retention:

``` text
90 days
```

For another project:

``` text
/your-project/docker
```

------------------------------------------------------------------------

# 15. Verify Docker Logs

Go to:

``` text
CloudWatch
  ↓
Logs
  ↓
Log groups
  ↓
/production-app/docker
```

Open the EC2 instance log stream.

You should see application/Docker messages such as:

``` text
Application started
Database connected
Received request
```

------------------------------------------------------------------------

# 16. Test Docker Logs From EC2

Check containers:

``` bash
docker ps
```

Check logs:

``` bash
docker logs YOUR_CONTAINER_NAME
```

Follow logs:

``` bash
docker logs -f YOUR_CONTAINER_NAME
```

------------------------------------------------------------------------

# 17. CloudWatch Metrics

## EC2 Built-in Metrics

Available automatically:

``` text
CPUUtilization
NetworkIn
NetworkOut
```

## CloudWatch Agent Metrics

Collected by the agent:

``` text
mem_used_percent
disk_used_percent
```

Namespace:

``` text
CWAgent
```

------------------------------------------------------------------------

# 18. Create CloudWatch Dashboard

Go to:

``` text
CloudWatch
  ↓
Dashboards
  ↓
Create dashboard
```

Example:

``` text
ProductionApp-Production
```

The dashboard will contain:

``` text
NetworkIn / NetworkOut
CPUUtilization
mem_used_percent
disk_used_percent
Application / Docker Logs
```

------------------------------------------------------------------------

# 19. Add NetworkIn / NetworkOut

Click:

``` text
+ Add widget
```

Select:

``` text
CloudWatch
  ↓
Metrics
  ↓
Line
```

Then:

``` text
EC2
  ↓
Per-Instance Metrics
```

Select your EC2 instance and:

``` text
NetworkIn
NetworkOut
```

Create the widget.

------------------------------------------------------------------------

# 20. Add CPUUtilization

``` text
+ Add widget
  ↓
CloudWatch
  ↓
Metrics
  ↓
Line
  ↓
EC2
  ↓
Per-Instance Metrics
```

Select:

``` text
CPUUtilization
```

Create the widget.

------------------------------------------------------------------------

# 21. Add Memory

``` text
+ Add widget
  ↓
CloudWatch
  ↓
Metrics
  ↓
Line
  ↓
CWAgent
```

Select:

``` text
mem_used_percent
```

for your EC2 instance.

Create the widget.

------------------------------------------------------------------------

# 22. Add Disk

``` text
+ Add widget
  ↓
CloudWatch
  ↓
Metrics
  ↓
Line
  ↓
CWAgent
```

Select:

``` text
disk_used_percent
```

Make sure the filesystem is:

``` text
/
```

Create the widget.

------------------------------------------------------------------------

# 23. Add Application / Docker Logs

Use:

``` text
+ Add widget
```

Select:

``` text
Logs table
```

Then open:

``` text
CloudWatch Logs Insights
```

Select:

``` text
/production-app/docker
```

Use:

``` sql
fields @timestamp, @message
| sort @timestamp desc
| limit 50
```

Run the query.

Then add it to the dashboard.

### Important

Use:

``` text
Logs → Table
```

for raw application logs.

Do not use a Line chart for raw logs because logs are individual events,
not numerical time-series data.

Use Line charts for:

``` text
CPU
Memory
Disk
Network
```

------------------------------------------------------------------------

# 24. Logs Insights Query Explained

``` sql
fields @timestamp, @message
```

Shows timestamp and log message.

``` sql
| sort @timestamp desc
```

Shows newest logs first.

``` sql
| limit 50
```

Shows the latest 50 events.

------------------------------------------------------------------------

# 25. Final Dashboard

``` text
+----------------------+----------------------+----------------------+----------------------+
| NetworkIn/NetworkOut | CPUUtilization       | mem_used_percent     | disk_used_percent    |
+----------------------+----------------------+----------------------+----------------------+
|                                                                                |
|                       Application / Docker Logs                              |
|                                                                                |
| Timestamp                         Message                                    |
| 2026-08-16 ...                    Application started                         |
| 2026-08-16 ...                    Received request                            |
| 2026-08-16 ...                    Database connected                          |
+--------------------------------------------------------------------------------+
```

Save the dashboard after adding the widgets.

------------------------------------------------------------------------

# 26. Monitoring Summary

  Component          Metric / Log               Source
  ------------------ -------------------------- ------------------
  CPU                `CPUUtilization`           EC2
  Network Download   `NetworkIn`                EC2
  Network Upload     `NetworkOut`               EC2
  Memory             `mem_used_percent`         CloudWatch Agent
  Disk               `disk_used_percent`        CloudWatch Agent
  Docker/API Logs    `/production-app/docker`   CloudWatch Agent

------------------------------------------------------------------------

# 27. Troubleshooting

## Memory is missing

Check:

``` text
1. CloudWatch Agent status
2. IAM role
3. CWAgent namespace
4. AWS region
5. Wait a few minutes
```

## Disk is missing

Check:

``` text
1. Disk configuration
2. "/" is configured
3. CWAgent namespace
4. Agent logs
```

## Docker logs are missing

Check:

``` text
1. /var/lib/docker/containers/
2. docker logs YOUR_CONTAINER_NAME
3. CloudWatch Agent status
4. IAM permissions
5. Log group name
6. EC2 instance log stream
7. CloudWatch Agent logs
```

## Logs Insights error

If you see:

``` text
A log group, a data source, a facet, or a tag filter must be selected to run a query
```

select the log group first:

``` text
/production-app/docker
```

Then run:

``` sql
fields @timestamp, @message
| sort @timestamp desc
| limit 50
```

------------------------------------------------------------------------

# 28. Useful Commands

### Docker

``` bash
docker ps
docker logs YOUR_CONTAINER_NAME
docker logs -f YOUR_CONTAINER_NAME
```

### CloudWatch Agent

``` bash
sudo systemctl status amazon-cloudwatch-agent
```

``` bash
sudo systemctl restart amazon-cloudwatch-agent
```

``` bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl   -a status
```

### Agent Logs

``` bash
sudo tail -f /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```

------------------------------------------------------------------------

# 29. Recommended Log Retention

Example:

``` text
Log Group:
 /production-app/docker

Retention:
 90 days
```

This automatically removes log events older than 90 days.

------------------------------------------------------------------------

# 30. Reuse This Guide For Another Project

For every new project, change only the project-specific values.

Example:

``` text
Project:
MyBackend

Log Group:
/my-backend/docker

Dashboard:
MyBackend-Production
```

The CloudWatch Agent installation, metrics, Logs Insights query, and
dashboard process remain the same.

------------------------------------------------------------------------

# Quick Reference

## CloudWatch Agent Config

``` text
/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

## Log Group

``` text
/production-app/docker
```

## Metrics

``` text
CPUUtilization
NetworkIn
NetworkOut
mem_used_percent
disk_used_percent
```

## Logs Query

``` sql
fields @timestamp, @message
| sort @timestamp desc
| limit 50
```

## Agent Status

``` bash
sudo systemctl status amazon-cloudwatch-agent
```

## Agent Restart

``` bash
sudo systemctl restart amazon-cloudwatch-agent
```

## Agent Log

``` bash
sudo tail -f /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```

------------------------------------------------------------------------

# Final Result

You now have a reusable production monitoring setup:

``` text
EC2
+
CloudWatch Agent
+
CPU
+
Memory
+
Disk
+
Network
+
Docker/Application Logs
+
CloudWatch Dashboard
```

When a production issue happens:

``` text
CloudWatch
  ↓
Production Dashboard
  ↓
CPU / Memory / Disk / Network
  ↓
Application / Docker Logs
```