# aws-cloudwatchlogs

Herewith the complete architecture of the labs if you complete this lab, and the other 2 advanced labs:
![Image of CloudWatchLogs Architecture](https://github.com/kuettai/aws-cloudwatchlogs/blob/master/img/cw-final.png?raw=true)

In this lab, we focus on the `grey line`, which covers:
- CloudWatch Log groups
- IAM roles
- CloudWatch Agents
- EC2 installations
- CloudWatch Metrics & Filters
- Amazon Simple Notification Services (SNS)
- CloudWatch Alarms
- CloudWatch Log Insights
- Publish Memory & Diskusage to CloudWatch
- Publish Custom Metrics to CloudWatch

## Introduction: Understanding about the architecture :smile:

## Agenda
This lab is going to covers the following:
- [Prologue: Preparing your environment](#Prologue)
- [Chapter 1: AWS CloudWatch Logs Agent](#Chapter-1)
  - Install CloudWatch Logs Agent on EC2 Amazon Linux 2
  - Look into CloudWatch Logs Agent config files - (a) awslogs.conf, (b) awscli.conf
  - Prepare simulation logs
  - Publish 2 types of log files - (a) delimiter, (b) json
- [Chapter 2: Creating CloudWatch Metrics & Alarms](#Chapter-2)
  - Create Filter/Metrics for both delimiter & JSON
  - Configure SNS to send notification as soon as the Alarms triggered
- [Chapter 3: Using CloudWatch Logs Insight](#Chapter-3)
  - Type of query
- [Chapter 4: Publish CloudWatch Metrics to AWS](#Chapter-4)
  - Prebuilt CloudWatch Metrics - Memory, Disk Usage etc
  - Custom Built CloudWatch Metrics
- [Chapter 5: Best Practices (Notes)](#Chapter-5)
- [Appendix and References](#References-and-References)


## Prologue
### Create IAM roles
By completing this chapter, we will achieve the following:

![Image of Prologue Diagram](https://github.com/kuettai/aws-cloudwatchlogs/blob/master/img/cw-prologue.png?raw=true)

1. Login into AWS Console
1. After login to AWS console, go to `IAM` service
1. On the left hand side, click `Roles`
1. After that, click `Create role` button at the next screen
1. At this setup wizard, follows through the steps below:
      -  Under __Select type of trusted entity__, select `AWS Service`
      -  Under __Choose a use case__, select `EC2`
      -  Leave everything else as default, scroll to the bottom, click `Next: Permissions` button
1. In the __Filter policies__ text box, type this `CloudWatchAgentServerPolicy`, tick on the checkbox appears
      -  Scroll to the bottom, click `Next: Tags` button
      -  Click on `Next: Review` button
1. In the next screen:
      -  __Role name*__: `MyEC2CloudWatchAgentRole`
      -  Scroll to the bottom, click `Create role` button
1. You should see the following success role creation message:

*The role __MyEC2CloudWatchAgentRole__ has been created*

### Create CloudWatch Log Groups
1. Next, navigate to `CloudWatch` service
1. On the left hand side, look for `Log groups` and click on it
1. Click on `Actions` -> `Create log group`
      -  __Log Group Name__: /labs/crm/access_log
      -  Click `Create log group` button
1. After successfully create the first log group, click on `Actions` -> `Create log group` to create second log group
      -  __Log Group Name__: /labs/crm/app_json_log
      -  Click `Create log group` button
1. (Optional) Besides the newly create Log Groups, click on `Never Expire`, set the __Retention:__ from `Never Expire` to `1 week (7 days)`. Click on `Ok` button. Repeat this for another log group.

### Create EC2
1. Next, navigate to `EC2` service
1. On the left hand side, look for `Instances` and click on it
1. After that, click `Launch Instance` button
1. At this setup wizard, follows through the steps below:
      - Click on `Select` button next to __Amazon Linux 2 AMI (HVM)__
      - Click on `Next: Configure Instance Details`
1. Step 3: Configure Instance Details
      - Auto-assign Public IP: `Use subnet setting (Enable)`
      - IAM role: `MyEC2CloudWatchAgentRole`
      - User data:
```bash
#!/bin/bash
yum update -y

```

1. Leave everything as default
      - Scroll to the bottom, click `Next: Add Storage` button
1. Click `Next: Add Tags` button
1. Step 5: Add Tags
      -  Click `Add Tag` button
      -  Key: `Name`
      -  Value: `myWebApp`
      -  Click `Next: Configure Security Group`
1. Step 6: Configure Security Group
      -  Security group name: `myWebAppSG`
      -  Click `Add Rule` button, select __Type:__ `HTTP`
      -  Click `Add Rule` button, select __Type:__ `SSH` (Skip this step if SSH is already available in the rule)
      -  Click `Review and Launch` button
1. Step 7: Review Instance Launch
      -  Take a final look on the configuration
      -  Click `Launch` button
1. At the popup dialog, you will see two (2) dropdown input.
      -  `Create a new key pair`
      -  __Key pair name:__ `lab-cwl-agent`
      -  Click `Download Key Pair`
      -  Click `Launch Instances` button
1. Wait until your newly create instance has `running` __Instance State__.
1. Congrats! You have completed the __Prologue__

## Chapter 1
Back to [Agenda](#Agenda)

!!!!! __Make sure__ you have completed the setup in [Prologue](#Prologue) before proceeding.

### AWS CloudWatch Logs Agent
By completing this chapter, we will achieve the following:

![Image of Chapter 1 Architecture Diagram](https://github.com/kuettai/aws-cloudwatchlogs/blob/master/img/cw-chap1-v2.png?raw=true)
### SSH into your newly created EC2
1. You should be viewing __EC2__ listing page
1. Click on the checkbox appears beside `myWebApp`
1. Click on `Connect` button on top of EC2 listing
1. Follow the steps to SSH into `myWebApp` __EC2__

### Setting up your environments
Prepare software: required for CloudWatch Logs agent to work:
```bash
# Run the following commands as ec2-user
## Ensure all softwares are up to date
sudo yum update -y

## Install AWS Cloudwatch Logs Agent
sudo yum install awslogs -y

## Important Folders/Files
# - /etc/awslogs/awslogs.conf <-- Main configure files!
# - /etc/awslogs/awscli.conf  <-- cli config, usually need to change it 1 time only
# - /var/log/awslogs.log      <-- storing awslogs agent, useful for troubleshooting

## Let's looks into the contents of the logs file:
sudo su -       # switch to root user
cat /etc/awslogs/awslogs.conf
cat /etc/awslogs/awscli.conf

# To further understand agent-configuration parameters:
https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AgentReference.html
```

Prepare software: required for this lab to perform simulation:
```bash
## Install PHP
## This is to simulate
##  - page not found error
##  - page redirection
##  - healthy page
amazon-linux-extras install -y php7.2

## Install Apache Web Server
## This is important for log simulation
##  - in delimiter format
##  - in JSON format
yum install httpd -y
systemctl enable httpd
systemctl start httpd

## To Test if installation succeed
curl http://169.254.169.254/latest/meta-data/public-hostname

## Copy & Paste the output above to your web-browsers
## If you managed to see a apache 'Test Page', congrats!
```

Create few web pages to simulate different logs events:
```bash
cd /var/www/html
cat <<EoF > redirect.php
<?php header("Location: ads.php");
EoF

cat <<EoF > ads.php
<?php echo "<pre>"; print_r(\$_SERVER);
EoF

cat <<EoF > index.php
<?php echo "Welcome to index page!";
EoF

cat <<EoF > timeout.php
<?php while(1){
  sleep(5);
}
EoF
```

Access the webpage to simulate entries into access_log
```bash
## To test 'em out
WEBURL=$(curl http://169.254.169.254/latest/meta-data/public-hostname)
curl $WEBURL/index.php
curl $WEBURL/index.php
curl $WEBURL/index.php
curl $WEBURL/redirect.php
curl $WEBURL/redirect.php
curl $WEBURL/nopage.php
curl $WEBURL/timeout.php  ##this will take 30seconds before hitting timeout.

## Test it in Web Browsers
echo $WEBURL/index.php ## Copy paste the output to web-browser
echo $WEBURL/redirect.php ## Copy paste the output to web-browser
echo $WEBURL/nopage.php ## Copy paste the output to web-browser
echo $WEBURL/timeout.php ## Copy paste the output to web-browser

## Looks into the access_log
tail -n100 /var/log/httpd/access_log
```

Next, simulate __JSON__ log into a file (assuming it is your daily application logs)
```bash
cd /var/www/html
wget https://raw.githubusercontent.com/kuettai/aws-cloudwatchlogs/master/src/genJsonLog.sh genJsonLog.sh
chmod +x genJsonLog.sh
./genJsonLog.sh app.log >> app.log

## Let it run for few seconds
## (Windows) CTRL + C
## (Mac) control + C
```

Next up, publish custom metrics to CloudWatch Log Groups
```bash
## Run the following comands as __root__ user
## Edit awscli.conf, you can use any text editor
vi /etc/awslogs/awscli.conf

# Change region=us-east-1
# to region=<Your_Region>

# For example, in Singapore Region: region=ap-southeast-1
# Visit this for all aws regions:
## https://docs.aws.amazon.com/general/latest/gr/rande.partial.html

#--------
#Setup Log Files to be publish:
vi /etc/awslogs/awslogs.conf

## You may remove the entry of [/var/log/messages] entirely
## Add the following lines to the bottom of the files to support both apache access_log and application json log
[/var/log/httpd/access_log]
datetime_format = %b %d %H:%M:%S
file = /var/log/httpd/access_log
buffer_duration = 5000
log_stream_name = {instance_id}
initial_position = start_of_file
log_group_name = /labs/crm/access_log

[/var/www/html/app.log]
file = /var/www/html/app.log
buffer_duration = 5000
log_stream_name = {instance_id}
initial_position = start_of_file
log_group_name = /labs/crm/app_json_log

## Save and close the file
```

Next up, start the services and monitor the CloudWatch Agents
```bash
systemctl restart awslogsd
tail -f /var/log/awslogs.log

## Look into the cloudwatch agents activities
## If spotted any error, fix it; here are few common scenarios:
##  - typo in cloudwatch log_group_name
##  - datetime_format does not match with your logs
##  - log file does not exists
```
### Verify ApplicationLogs on CloudWatch LogGroups
1. From the console, navigate to `CloudWatch` service
1. On the left hand side, click on Log groups
1. Click on `/labs/crm/app_json_log`, after that click on the Log streams, should be starting with __i-[random_alpha_numberic]__
1. Sample of application log, *in __JSON__ format*, on your EC2 instance is now available in CloudWatch Log Group.

### Verify AccessLogs on CloudWatch LogGroups
1. On the left hand side, click on Log groups
1. This time, click on `/labs/crm/access_log`, after that click on the Log stream that start with __i-__[random_alpha_numberic].
1. Sample of access log, *in __SPACE SEPARATOR__ format*, on your EC2 instance is now available in CloudWatch Log Group.

Notes:
There is an entry on AWS Blog that provides in-depth guidance on changing Apache Logs into JSON format, then publish it to cloudwatch log groups.
[AWS Blog on Simplifying Apache Logs](https://aws.amazon.com/blogs/mt/simplifying-apache-server-logs-with-amazon-cloudwatch-logs-insights/)

## Chapter 2
Back to [Agenda](#Agenda)

### Creating CloudWatch Metrics & Alarms
By completing this chapter, we will achieve the following:

![Image of Chapter 2 Architecture Diagram](https://github.com/kuettai/aws-cloudwatchlogs/blob/master/img/cw-chap2.png?raw=true)

Before diving into CloudWatch metrics & alarms, i need you to understand the following diagram
![Image of Chapter 2 Architecture Diagram](https://github.com/kuettai/aws-cloudwatchlogs/blob/master/img/cw-metric-query-loggroup-v2.png?raw=true)

- You can have multiple Log Groups
- 1 Log Group can have 1 or multiple log stream(s)
- 1 Log Streams have multiple log events
- __CloudWatch metrics/alarms__ is working on collection of log streams under one log group.
- However, we can apply __Query Log__ either on log group level, or each stream.

#### Create Metrics on JSON Log Group
1. From the console, navigate to `CloudWatch` service
1. On the left hand side, click on Log groups
1. Click on `/labs/crm/app_json_log`
1. Navigate to the subtab `Metric filters`
1. Click on `Create metric filter` button. You can find this button at the bottom of the page, or the top right corner
1. Step 1 - Define Pattern
    1. Instead of defining the pattern, skip the first input, under `Test pattern -> Select log data to test`, pick the data from log steam `i-[random_alpha_numberic]`
    1. Visualize sample of the log pattern under `Log event messages`
    1. Back to the top, `Filter pattern`, input this: `{$.status="FATAL"}`
    1. [Optional] You can lookup more [Advance JSON CloudWatch Metric Filtering here](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/FilterAndPatternSyntax.html#w2aac15c13b9c28b6b6b6)
    1. Click on `Test Pattern` button below
    1. Expand the `Show test results`, validate your filtered output against your filter. You should only see all event messages that stated "FATAL" under __status__
    1. Click on the `Next Button`
1. Step 2 - Assign Metrics
    1. __Filter name__: CRM-AppLogsFatalMetric
    1. __Metric namespace__: CRM
    1. __Metric name__: AppLogsFatalCount
    1. __Metric value__: 1
    1. __Default value - optional__: 0
1. Step 3 - Review and create
    1. Review the metrics
    1. Click on `Create metric filter`
1. After successfully create the metric filter, click on `AppLogsFatalCount`. (If you do not see it. Navigate to `Log Groups -> /labs/crm/app_json_log -> Metrics Filter`)
1. Under Metrics
    1. Click on `CRM -> Metrics without any dimension`
    1. Check on the checkbox beside `AppLogsFatalCount`
    1. Navigate to next tab, `Graphed metrics (1)`
    1. Under `Statistic`, change the value `Average` to `Sum`
    1. Under `Period`, change the value `5 Minutes` to `1 Minute`
    1. At the top of the graph, changes from `3h` to `1h`

#### Create SNS Topics, for Alarms notification
1. Navigate to `Simple Notification Services` a.k.a `SNS`
1. On the left hand side, click `Topics`
1. Click `Create topic`
1. Details
    1. __Name__: `CRMAppFatalSNS`
    1. Leave everything as defaults, click `Create Topic` button at the bottom of the page
1. Under __Subscriptions__ tab
    1. Click on `Create subscription` button
1. Under __Details__
    1. __Protocol__: Email
    1. __EndPoint__: <Key In Your Email Address>
    1. Click on `Create subscription` button on the bottom right
1. Now, go to your mailbox (check your __spam__ if not in __inbox__) for __AWS Notification - Subscription Confirmation__
1. Click `Confirm subscription` in the email body.
1. A new tab will open, showing 'Subscription Confirmed!'

#### Create Alarms using the metrics above
1. Navigate to `CloudWatch` service
1. On the left hand side, click on `Alarms`
1. Click `Create Alarm` button on the top right
1. Step 1 - Specify metric and conditions
    1. Graph - Click `Select metric`
    1. Select `CRM -> Metrics with no dimension -> AppLogsFatalCount`. Tick on the checkbox beside `AppLogsFatalCount`
    1. Click `Select metrics` at the bottom right
    1. Change __Statistic__ from `Average` to `Sum`
    1. Change __Period__ from `5 minutes` to `1 minute`
    1. __Threshold type__: `Static`
    1. __Define alarm conditions__: `Greater`
    1. __than...___: 1
    1. Expand `Additional configuration`
    1. __Datapoints of alarm__: `1` of out `1`
    1. __Missing data treatment__: `Treat missing data as good (not breaching threshold)`
    1. Click `Next`
1. Step 2 - Configure actions
    1. __Send a notification to...__: `CRMAppFatalSNS`
    1. Click `Next`
1. Step 3 - Add name and description
    1. __Alarm name__: `CRMAppFatalAlarm`
    1. Click `Next`
1. Step 4 - Preview and create
    1. Review your setting one last time
    1. Click `Create alarm` at the bottom right of the page

#### To simulate alarms
```bash
## ssh to your ec2 instance
## Run the following only if you are not a root user
sudo su -

## Generate some application logs
cd /var/www/html
./genJsonLog.sh >> app.log

## Leave it running
```

#### Wait for alarms
1. Look into your mailbox, you should receive __CloudWatch Notification__
1. After that, navigate to `CloudWatch` -> `Alarms`
1. Click on `CRMAppFatalAlarm` and visualize the metrics
1. Navigate back to your __EC2 instance terminal__ and stop the `genJsonLog.sh` job. (command + c, or ctrl + c)
1. After a minute, refresh your browser. Scroll down to __History__, the latest entry should stated `Alarm updated from In alarm to OK`


=================================
#### Challenge!
<details>
  <summary>Click here to expand</summary>

  ### Scenario
  Dev team want to get near real-time notification when there are more than 5 status_code=404 happens
  * You can repeat the step above on apache log.
  * Example of __Filter pattern__: `[ip, id, user, timestamp, request, status_code=4* || status_code=5*, size]`
  * The above is to extract apacheLog with status code start with __4 or 5__, eg: 400, 404, 502
</details>

=================================


## Chapter 3
Back to [Agenda](#Agenda)

### Using CloudWatch Logs Insight
1. Navigate to `CloudWatch` service
1. On the left hand side, click on `Log Groups`

#### Perform query on JSON log
1. Click `/labs/crm/app_json_log`
1. At the top right corner, click on `Query log group` button
1. Without changing anything, click on `Run query` button
1. Look at the result shows below: (a) histogram, (b) log events
1. (a) You can hover over the `bar` on the histogram to look into the __count__ hits per minutes
1. (b) You can expand the log events by click on the (+) magnifying glass beside each log event.
    1. There are 2 timing available in the log events details: @ingestionTime, @timestamp.
    1. @ingestionTime is the time that CloudWatch Agent publishes the log event to CloudWatch
    1. @timestamp is the time that actual log happens
    1. You can perform filter in this detail view too


```sql
fields @timestamp, code, msg
| filter status = "FATAL" or status = "WARN"
| sort @timestamp desc
| limit 20
```
1. Next, use the filter above to query (copy the text, and replace the text in the CloudWatch Query textbox):
1. Try to understand the query above
    - __fields__ - identify which columns to show
    - __filter__ - whereClause, applying filter logic to find out data you want
    - __sort__ - sort the result
    - __limit 20__ - shows only 20 results
1. Execute the query by click on `Run query` button

```sql
fields @timestamp, code, msg
| stats count(status) as cntStatus by status
| filter status = "FATAL" or status = "WARN"
```
1. Next, use the filter above to query (copy the text, and replace the text in the CloudWatch Query textbox):
1. Try to understand the query above
    - __stats__ - perform aggregation function, such as `count`, `sum`, `average`, `max`, `min`
    - __as__ - Give a name to the aggreate columns
    - __by__ - Defining the logic of group by for your aggregation function
1. Execute the query by click on `Run query` button
1. Click on `Visualization` tab, at the left hand side, change `Line` to `Bar`. You can have better presentation of data here.


#### Perform query on Apache Log
This scenario is used to showcase log files store in delimiter format, or fix regular expression pattern format.
1. Click on `Clear` button at the top of textarea
1. Click on `Select log group(s)` dropdown
1. Click `/labs/crm/access_log`, make sure the checkbox is __ticked__, click on any whitespace on the screen to hide the dropdown

Run the queries below and examine the output
```sql
fields @timestamp, @message
| sort @timestamp desc
| limit 50
```

```sql
parse @message '* - - [*] "* * *" * * "*" "*"' client, dateTimeString, httpVerb, url, protocol, statusCode, bytes, referer, userAgent
| fields @message
| sort @timestamp desc
| limit 50
```

```sql
parse @message '* - - [*] "* * *" * * "*" "*"' client, dateTimeString, httpVerb, url, protocol, statusCode, bytes, referer, userAgent
| filter statusCode in [404, 504]
```

```sql
parse @message '* - - [*] "* * *" * * "*" "*"' client, dateTimeString, httpVerb, url, protocol, statusCode, bytes, referer, userAgent
| filter statusCode in [404, 504]
| stats count(*) as statCnt by statusCode, httpVerb
```

To further customize your query, refers to [AWS Documention on CloudWatch Log Insights Query Syntax](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html)

## Chapter 4
Back to [Agenda](#Agenda)

### Publish CloudWatch Metrics to AWS
By completing this chapter, we will achieve the following:
1. Publish Memory & Disk Metrics to CloudWatch
1. Publish Custom Metrics to CloudWatch

#### Publish Memory & Disk Metrics to CloudWatch
In this lab, we will cover installation guide on AWS EC2 with Amazon Linux 2. Installation guide for other OS can be found in [AWS Offical Documentation here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/mon-scripts.html)

```bash
## Login to your EC2 (If you got logout)
## Make sure you run the following commands as root,
## Step 1. Install dependencies
yum install -y perl-Switch perl-DateTime perl-Sys-Syslog perl-LWP-Protocol-https perl-Digest-SHA.x86_64

## Step 2. Download monitoring scripts
cd /var/www/html
curl https://aws-cloudwatch.s3.amazonaws.com/downloads/CloudWatchMonitoringScripts-1.2.2.zip -O
unzip CloudWatchMonitoringScripts-1.2.2.zip
rm -f CloudWatchMonitoringScripts-1.2.2.zip
cd aws-scripts-mon


## Run the following command to verify output:
## With --verify parameter, it will not publish to cloudwatch
./mon-put-instance-data.pl --mem-used-incl-cache-buff --mem-util --mem-used --mem-avail --swap-util --disk-path=/ --disk-space-util --disk-space-used --disk-space-avail --verify --verbose

## Remove --verify parameter
./mon-put-instance-data.pl --mem-used-incl-cache-buff --mem-util --mem-used --mem-avail --swap-util --disk-path=/ --disk-space-util --disk-space-used --disk-space-avail --verbose

## When executing this script in schedule, add this parameter: --from-cron
## Add this to scheduler to run every minutes
* * * * * /var/www/html/aws-scripts-mon/mon-put-instance-data.pl --mem-used-incl-cache-buff --mem-util --mem-used --mem-avail --swap-util --disk-path=/ --disk-space-util --disk-space-used --disk-space-avail --from-cron

## If you want the scheduler to run every 5 minutes, use this:
*/5 * * * * /var/www/html/aws-scripts-mon/mon-put-instance-data.pl --mem-used-incl-cache-buff --mem-util --mem-used --mem-avail --swap-util --disk-path=/ --disk-space-util --disk-space-used --disk-space-avail --from-cron


## To view custom metrics using the script:
./mon-get-instance-stats.pl --recent-hours=12

# You might run into error due to no permission issues:
# Go to IAM -> Roles -> [Select created role]
# Attach inline policy with the following permissions,
#   Service: CloudWatch
#   Actions: List, Read
#   Resources: All resources
# Save. It takes about 5-10seconds for new permission to kick in.
```
After published out data to CloudWatch, you can setup your dashboard using these new metrics
1. Navigate to `CloudWatch`
1. On the left hand side, click `Metrics`
1. Under `Custom Namespaces`, click `System/Linux`

#### Publish Custom Metrics to CloudWatch
```bash
export serverId=$(curl http://169.254.169.254/latest/meta-data/instance-id)
export serverType=$(curl http://169.254.169.254/latest/meta-data/instance-type)
while true
do
  r=$(((RANDOM%100)))
  aws cloudwatch put-metric-data --metric-name ActiveUsersCount --namespace CRMAppNS --unit Count --value $r --dimensions InstanceId=$serverId,InstanceType=$serverType
  sleep 5
done
```
1. Let the script above to run for about 1-2 minutes
1. Navigate to `CloudWatch`
1. On the left hand side, click `Metrics`
1. Under `Custom Namespaces`, click `CRMAppNS`
1. Click `InstanceId,InstanceType`
1. Check on the checkbox
1. Switch to next tab, `Graphed metrics`
1. Change period value from `5 minutes` to `5 seconds`
1. Visualize the graph


__Congratulations!__ You completed the labs.

## Chapter 5
Back to [Agenda](#Agenda)

### Best Practices
#### Operational Excellence
- Define important metrics in your application log
- Output your application log in consistency manners (use separator, or JSON)
- Plan your metrics, alarms and actions (Notification is good, but not the best ending. Notification still requires human intervention to read, understand the logs, and make plans against the alarm manually)
- Create your metrics and alarms, build auto-recovery workflow if possible. (For example, if detected Diskspace full, go to EC2 install to clean up temp files automatically)

#### Cost Optimisation
- Enable `Stored bytes` column at `CloudWatch -> Log Groups` table view by clicking the 'gear' icon at the top right of the view.
- Set Log Groups Expiry, (e.g: 7 days)
- CloudWatch Log storage cost $0.03 per GB. If logs required to be store for Long Term Archival, store it in S3 IA ($0.0125 per GB) or S3 Glacier ($0.004 per GB), or even S3 Glacier Deep Archive ($0.00099 per GB). The prices are based on us-east-1, N. Virginia region, as of 23-Mar-2020. Look at the table below for comparison.

| Size | CloudWatch | S3 IA |S3 Glacier | S3 Glacier Deep Archive
|---|---:|---:|---:|---:
|1GB|$0.03|$0.0125|$0.004|$0.00099
|50GB|$1.5|$0.625|$0.2|$0.0495
|200GB|$6|$2.5|$0.8|$0.198
|1,000GB|$30|$12.5|$4|$0.99
|80,000GB|$2400|$1000|$320|$79.2

#### Security
- Use IAM Roles to grant EC2 permissions for CloudWatch Log Agent to publish log events to CloudWatch
- Turn on encryption at CloudWatch Log Groups (As of 26-Apr-2020, it can only be enable via CLI)
  - refers to: https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/encrypt-log-data-kms.html
  - KMS create keys
  - Ensure EC2Role has permission to access the key
  - Ensure IAM Role has permission to perform actions on "Cloudwatch Logs -> Associate Key"
  - Ensure KMS key policy allows "Principal -> Service -> logs.[region].amazonaws.com"
- CloudWatch Logs Agent is encrypting data in transit by default


#### Reliability
- Store /etc/awslogs/awslogs.conf either in github or S3 bucket.
- Leverage on EC2 User Data or System Manager to install and apply configuration files to all servers

#### Performance Efficiency
N/A

## Next Steps
1. [Todo] Store CloudWatch configurations on S3 Bucket, and automate installation in new instance
1. [Todo] Stream daily logs to S3

## Appendix and References

https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/SubscriptionFilters.html
https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/S3ExportTasks.html
