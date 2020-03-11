# aws-cloudwatchlogs

Herewith the final architecture of the lab:
![Image of CloudWatchLogs Architecture](https://d2908q01vomqb2.cloudfront.net/fc074d501302eb2b93e2554793fcaf50b3bf7291/2018/04/25/SPO_Data-ingestion_final.png)

## Prologue: Understanding about the architecture
This lab is going to covers the following:
1. AWS CloudWatch Logs Agent
  * Install CloudWatch Logs Agent on EC2 Amazon Linux 2
  * Look into CloudWatch Logs Agent config files - (a) awslogs.conf, (b) awscli.conf
  * Publish 2 types of log files - (a) delimiter, (b) json
2. Creating CloudWatch Metrics & Alarms
  * Create Filter/Metrics for both delimiter & JSON
  * Configure SNS to send notification as soon as the Alarms triggered
3. Using CloudWatch Logs Insight
  * Type of query
4. Publish CloudWatch Metrics to AWS
  * Prebuilt CloudWatch Metrics - Memory, Disk Usage etc
  * Custom Built CloudWatch Metrics
5. Best Practices (Notes)


## Step 1: AWS CloudWatch Logs Agent
