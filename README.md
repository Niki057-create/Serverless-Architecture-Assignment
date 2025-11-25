## 1. Archive Old Files from S3 to Glacier Using AWS Lambda and Boto3

### Objective: To automate the archival of files older than a certain age from an S3 bucket to Amazon Glacier for cost-effective storage.

### Boto3 Python Script/Lambda Code:

import boto3

from datetime import datetime, timezone, timedelta

s3 = boto3.client('s3')
bucket_name = 'nikitha-archive-demo-bucket'  # your bucket name

def lambda_handler(event, context):
    response = s3.list_objects_v2(Bucket=bucket_name)

    if 'Contents' not in response:
        print("No files in the bucket.")
        return
    
    now = datetime.now(timezone.utc)

    # For REAL use: age_limit = timedelta(days=180)
    # For TESTING: archive anything older than 10 seconds
    age_limit = timedelta(seconds=10)

    for obj in response['Contents']:
        key = obj['Key']
        last_modified = obj['LastModified']

        file_age = now - last_modified

        print(f"{key} last modified at {last_modified}, age = {file_age}")

        if file_age > age_limit:
            print(f"Archiving file: {key}")
            s3.copy_object(
                Bucket=bucket_name,
                Key=key,
                CopySource={'Bucket': bucket_name, 'Key': key},
                StorageClass='GLACIER',
                MetadataDirective='COPY'
            )
        else:
            print(f"File is recent, not archiving: {key}")


### Documentation with Screenshots: [S3_Glacier_Assignment_Formatted_With_Screenshots.docx](https://github.com/user-attachments/files/23750914/S3_Glacier_Assignment_Formatted_With_Screenshots.docx)

----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## 2.  Monitor Unencrypted S3 Buckets Using AWS Lambda and Boto3

### Objective: To enhance the AWS security posture by setting up a Lambda function that detects any S3 bucket without server-side encryption.

### Boto3 Python Script/Lambda Code:
import boto3

def lambda_handler(event, context):
    s3 = boto3.client('s3')

    print("Starting Lambda execution...")

    # List all buckets
    response = s3.list_buckets()
    buckets = response['Buckets']
    print(f"Total buckets in account: {len(buckets)}")

    unencrypted_buckets = []

    # 👇 For safety, only check first 20 buckets (you can change this number)
    buckets_to_check = buckets[:20]

    for bucket in buckets_to_check:
        bucket_name = bucket['Name']
        print(f"Checking bucket: {bucket_name}")

        try:
            enc = s3.get_bucket_encryption(Bucket=bucket_name)
            # If this call does NOT raise an error, encryption exists
            print(f"✅ {bucket_name} has encryption configured")
        except Exception:
            # If encryption is not enabled or config not found
            print(f"❌ {bucket_name} does NOT have server-side encryption")
            unencrypted_buckets.append(bucket_name)

    print("Buckets without server-side encryption:")
    for b in unencrypted_buckets:
        print(b)

    return {
        'unencrypted_buckets': unencrypted_buckets
    }

### Documentation with Screenshots: [Monitor Unencrypted S3 Buckets Using AWS Lambda and Boto3.docx](https://github.com/user-attachments/files/23751372/Monitor.Unencrypted.S3.Buckets.Using.AWS.Lambda.and.Boto3.docx)

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


## 3. Automated S3 Bucket Cleanup Using AWS Lambda and Boto3

### Objective: To gain experience with AWS Lambda and Boto3 by creating a Lambda function that will automatically clean up old files in an S3 bucket.

### Boto3 Python Script/Lambda Code:

import boto3

from datetime import datetime, timezone, timedelta

BUCKET_NAME = 'nikitha-s3-cleanup-bucket'  # change if needed
DAYS_THRESHOLD = 30

def lambda_handler(event, context):
    s3 = boto3.client('s3')
    cutoff = datetime.now(timezone.utc) - timedelta(days=DAYS_THRESHOLD)
    print(f"Running cleanup for bucket: {BUCKET_NAME}")
    print(f"Cutoff datetime (UTC): {cutoff.isoformat()}")

    deleted_files = []
    paginator = s3.get_paginator('list_objects_v2')

    # Iterate through all objects in the bucket
    for page in paginator.paginate(Bucket=BUCKET_NAME):
        for obj in page.get('Contents', []):
            key = obj['Key']
            last_modified = obj['LastModified']

            # Compare date to cutoff
            if last_modified < cutoff:
                print(f"Deleting: {key} (LastModified: {last_modified})")
                s3.delete_object(Bucket=BUCKET_NAME, Key=key)
                deleted_files.append(key)

    print(f"Deleted files: {deleted_files}")

    return {
        'statusCode': 200,
        'body': f"Deleted {len(deleted_files)} old files: {deleted_files}"
    }

### Documentation with Screenshots: [Automated S3 Bucket Cleanup Using AWS Lambda and Boto3.docx](https://github.com/user-attachments/files/23751674/Automated.S3.Bucket.Cleanup.Using.AWS.Lambda.and.Boto3.docx)

----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## 4. Automated SNS Alerts for EC2 Disk Space Utilization

### Objective: To set up a Lambda function that checks EC2 instances for disk space utilization, sending an SNS alert if utilization exceeds 85%.

### Boto3 Python Script/Lambda Code:

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
        print("No disk metrics yet. Possibly waiting for agent publish or permissions blocked.")
        return

    latest = sorted(datapoints, key=lambda x: x['Timestamp'])[-1]
    usage = latest['Average']

    if usage >= THRESHOLD:
        alert_message = f"Disk usage is {usage:.2f}% on instance {INSTANCE_ID}"
        sns.publish(TopicArn=SNS_TOPIC_ARN, Message=alert_message, Subject="EC2 Disk Alert")
        return {"status": "Alert sent"}

    return {"status": f"Disk OK ({usage:.2f}%)"}

### Documentation with Screenshots: [Automated SNS Alerts for EC2 Disk Space Utilization.docx](https://github.com/user-attachments/files/23752447/Automated.SNS.Alerts.for.EC2.Disk.Space.Utilization.docx)

