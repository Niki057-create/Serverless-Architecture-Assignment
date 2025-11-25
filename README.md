## Archive Old Files from S3 to Glacier Using AWS Lambda and Boto3

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
