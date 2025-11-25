# Automated SNS Alerts for EC2 Disk Space Utilization
### Assignment 16 — AWS Lambda | CloudWatch | SNS | EventBridge

## Objective
Implement an automated alerting system that monitors **EC2 disk space utilization** using AWS Lambda.  
If disk usage exceeds **85%**, the system triggers an **SNS Email Notification**.  
The Lambda function runs **daily** using an **EventBridge Scheduled Rule**.

## Architecture

```mermaid
flowchart LR
A[EC2 Instance] --> B[CloudWatch Agent]
B --> C[CloudWatch Metrics]
C --> D[Lambda Function]
D --> E[SNS Topic]
E --> F[Email Notification]
D <-->|Daily Trigger| G[EventBridge Rule]

## Components Used

| AWS Service           | Purpose                        |
| --------------------- | ------------------------------ |
| EC2 Instance (Ubuntu) | Server to monitor              |
| CloudWatch Agent      | Collects disk metrics          |
| CloudWatch Metrics    | Stores disk utilization metric |
| Lambda                | Reads metrics & sends alerts   |
| SNS Topic             | Sends notification emails      |
| EventBridge           | Schedules Lambda execution     |
| IAM Roles             | Permissions for EC2 & Lambda   |

## 1. EC2 Instance (Monitored Server)
Shows the Ubuntu EC2 instance (t2.micro) with the IAM role attached (e.g. Nikitha-EC2-CloudWatchAgentRole).

<img width="1280" height="692" alt="EC2 Instance Summary" src="https://github.com/user-attachments/assets/3b4d548c-69b8-4c16-bb0d-9eec7c2497df" />

<img width="1156" height="536" alt="image" src="https://github.com/user-attachments/assets/ae9c7991-43ee-441f-886e-b2d3685cdac1" />

## 2. CloudWatch Agent Status on EC2
Terminal output confirming that the amazon-cloudwatch-agent service is running and configured.

<img width="638" height="74" alt="image" src="https://github.com/user-attachments/assets/b66fd7f1-832a-4c4a-98da-a95448e6ddb3" />

## 3. SNS Topic and Subscription
SNS topic Nikitha-EC2DiskAlerts with an email subscription in Confirmed state.

<img width="1272" height="368" alt="Topic Created" src="https://github.com/user-attachments/assets/4b45ebde-2b25-4848-b43f-0d55c3385340" />

## 4. Lambda Function Configuration
Lambda function EC2DiskCheckLambda showing:

a. Runtime: Python 3.x
b. Environment variables (INSTANCE_ID, SNS_TOPIC_ARN, THRESHOLD)
c. Execution role with required permissions.

<img width="1073" height="629" alt="image" src="https://github.com/user-attachments/assets/458863c0-9264-4122-9ea3-7353c4e18b5e" />

<img width="1269" height="658" alt="IAM Lambda Role Created" src="https://github.com/user-attachments/assets/ca7709a4-2260-4c84-a8f1-d0bc136a0afc" />

## 5. Lambda Test Execution

Example of successful Lambda test execution.
In restricted environments, the response may be:

No disk metrics found for instance ... in the last 10 minutes.

<img width="830" height="309" alt="Successful Test 1" src="https://github.com/user-attachments/assets/b9f9786f-1fdd-4a01-af0d-94ab644703d2" />

## 6. EventBridge Scheduled Rule

EventBridge rule Daily-EC2-DiskCheck using *cron(0 0 * * ? ) to trigger the Lambda function daily.

<img width="1277" height="713" alt="image" src="https://github.com/user-attachments/assets/16e3dd0f-8a68-4aae-a962-153e2486759f" />

## Implementation Steps

### 1. Create an SNS Topic

Go to SNS → Topics → Create topic

Select Standard

Name: Nikitha-EC2DiskAlerts

Create topic

Create Subscription
Field	Value
Protocol	Email
Endpoint	Your email

Confirm the subscription from your email inbox.

### 2. Install CloudWatch Agent on EC2 (Ubuntu)
wget https://amazoncloudwatch-agent.s3.amazonaws.com/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
sudo dpkg -i amazon-cloudwatch-agent.deb

Create Agent Config
sudo tee /opt/aws/amazon-cloudwatch-agent/bin/config.json > /dev/null <<'EOF'
{
  "metrics": {
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}"
    },
    "metrics_collected": {
      "disk": {
        "measurement": [
          {"name": "disk_used_percent", "unit": "Percent"}
        ],
        "resources": ["*"],
        "ignore_file_system_types": ["sysfs", "devtmpfs"]
      }
    }
  }
}
EOF

Start Agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
-a start -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json -s
Note: In some shared training accounts, CloudWatch Agent may not be allowed to push custom metrics due to metadata/permission restrictions. In such cases, the CWAgent namespace may not appear and Lambda will log that no datapoints are available.

### 3. Create IAM Role for EC2

Create an IAM role for EC2 and attach:

CloudWatchAgentServerPolicy

AmazonSSMManagedInstanceCore

Attach this role to the monitored EC2 instance.

### 4. Create Lambda Function
| Field   | Value                |
| ------- | -------------------- |
| Name    | `EC2DiskCheckLambda` |
| Runtime | Python 3.12          |

Environment Variables
| Key           | Value                                              |
| ------------- | -------------------------------------------------- |
| INSTANCE_ID   | `i-008d5c0b8076ccc1c`                              |
| SNS_TOPIC_ARN | `arn:aws:sns:eu-west-2:xxxx:Nikitha-EC2DiskAlerts` |
| THRESHOLD     | `85`                                               |

Required IAM Permissions for Lambda

Attach policies to the Lambda execution role:
AWSLambdaBasicExecutionRole
CloudWatchReadOnlyAccess
AmazonSNSFullAccess

### 5. Lambda Function Code
import os
import boto3
from datetime import datetime, timedelta, timezone

cloudwatch = boto3.client('cloudwatch')
sns = boto3.client('sns')

INSTANCE_ID = os.environ['INSTANCE_ID']
SNS_TOPIC_ARN = os.environ['SNS_TOPIC_ARN']
THRESHOLD = float(os.environ.get('THRESHOLD', '85'))

def lambda_handler(event, context):
    end_time = datetime.now(timezone.utc)
    start_time = end_time - timedelta(minutes=10)

    response = cloudwatch.get_metric_statistics(
        Namespace='CWAgent',
        MetricName='disk_used_percent',
        Dimensions=[{'Name': 'InstanceId', 'Value': INSTANCE_ID}],
        StartTime=start_time,
        EndTime=end_time,
        Period=300,
        Statistics=['Average']
    )

    datapoints = response.get('Datapoints', [])

    if not datapoints:
        print(f"No disk metrics found for instance {INSTANCE_ID} in the last 10 minutes.")
        return {
            "statusCode": 200,
            "body": f"No disk metrics found for instance {INSTANCE_ID} in the last 10 minutes."
        }

    latest = sorted(datapoints, key=lambda x: x['Timestamp'])[-1]
    usage = latest['Average']

    if usage >= THRESHOLD:
        message = f"Disk usage alert! Instance {INSTANCE_ID} is at {usage:.2f}%"
        sns.publish(
            TopicArn=SNS_TOPIC_ARN,
            Message=message,
            Subject="EC2 Disk Alert"
        )
        return {"statusCode": 200, "body": "Alert sent"}

    return {"statusCode": 200, "body": f"Disk usage is OK at {usage:.2f}%."}

### 6. Create EventBridge Scheduled Rule

Open Amazon EventBridge → Rules → Create rule

Name: Daily-EC2-DiskCheck

Rule type: Schedule

Schedule expression:

cron(0 0 * * ? *)

Target: EC2DiskCheckLambda

### Testing

Run a Test event in Lambda.

Check the logs in CloudWatch Logs for:

“No disk metrics found…” (restricted environments), or

“Alert sent” when datapoints are present and above threshold.

Confirm email delivery via SNS topic.

### Notes / Observations

In restricted training environments, CloudWatch Agent may not have permission to push metrics, so get_metric_statistics returns no datapoints.
SNS and Lambda integration can be validated by sending a test message directly from Lambda.
The design is fully reusable for real production accounts where metrics are available.

### Final Outcome

Automated disk utilization monitoring using Lambda
Email alerts via SNS when utilization exceeds threshold
Daily execution scheduled via EventBridge





