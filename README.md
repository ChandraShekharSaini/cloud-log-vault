# CloudLogVault — Cost-Optimized AWS Log Archival

A **cost-optimized server log archival system** that automatically collects logs from **EC2 through Amazon CloudWatch Logs**, processes them using **AWS Lambda**, compresses them with **Gzip**, and archives them to **Amazon S3 every 24 hours**.

## 🚀 Project Overview

In this project, application/server logs generated on an **EC2 instance** are sent to **Amazon CloudWatch Logs** for monitoring.

Instead of keeping all historical logs in CloudWatch for a long period, an **AWS Lambda function runs every 24 hours** and retrieves the previous 24 hours of logs.

The Lambda function:

1. Retrieves CloudWatch log streams.
2. Fetches the previous 24 hours of log events.
3. Converts the logs into JSON.
4. Compresses the logs using Gzip.
5. Uploads the compressed `.json.gz` files to Amazon S3.
6. Stores the archived logs using timestamp-based filenames.

This approach helps **optimize logging costs** by using S3 for long-term log retention while keeping CloudWatch primarily for monitoring and recent log analysis.

## 🏗️ Architecture

![Architecture](https://github.com/ChandraShekharSaini/aws-event-driven-deployment/blob/e4070955dd860654d2d1a71460b725b321ba5b80/images/cloud-log-vault.png)

## 💰 Cost Optimization

The main goal of this project is **log-storage cost optimization**.

* CloudWatch Logs are useful for **real-time monitoring and troubleshooting**.
* Historical logs can be archived to **Amazon S3**, which is generally more economical for long-term storage.
* Logs are **Gzip compressed** before uploading to reduce storage size.
* Lambda performs the archival automatically without requiring a continuously running server.
* The archival process runs **once every 24 hours**, reducing unnecessary processing.

### Cost-Optimized Flow

```text
EC2
 │
 ▼
CloudWatch Logs
 │
 │ 24-hour logs
 ▼
Lambda
 │
 │ Gzip compression
 ▼
S3
 │
 └── Long-term archival
```

## ⚙️ Technologies Used

* **Amazon EC2** — Generates application/server logs
* **Amazon CloudWatch Logs** — Centralized log collection
* **AWS Lambda** — Serverless log archival
* **Amazon S3** — Long-term log storage
* **Python** — Lambda application
* **Boto3** — AWS SDK for Python
* **Gzip** — Log compression
* **IAM** — Access control
* **EventBridge Scheduler/Rule** — 24-hour Lambda scheduling

## 📁 Project Structure

```text
cloudlogvault/
│
├── lambda_function.py
├── iam-policy.json
├── README.md
└── .gitignore
```

## 🔧 Lambda Function

The Lambda function retrieves logs from the configured CloudWatch Log Group and uploads compressed archives to S3.

```python
import boto3
import gzip
import json
import time

logs_client = boto3.client('logs')
s3_client = boto3.client('s3')

S3_BUCKET_NAME = 'YOUR-S3-BUCKET'
LOG_GROUP = 'LOG-FROM-EC2'
TIME_WINDOW_HOURS = 24


def lambda_handler(event, context):

    start_time = int(
        (time.time() - TIME_WINDOW_HOURS * 3600) * 1000
    )

    end_time = int(time.time() * 1000)

    paginator = logs_client.get_paginator(
        'describe_log_streams'
    )

    page_iterator = paginator.paginate(
        logGroupName=LOG_GROUP
    )

    uploaded_files = []

    for page in page_iterator:

        for stream in page['logStreams']:

            log_stream_name = stream['logStreamName']

            try:

                response = logs_client.get_log_events(
                    logGroupName=LOG_GROUP,
                    logStreamName=log_stream_name,
                    startTime=start_time,
                    endTime=end_time,
                    limit=10000
                )

                log_events = response.get('events', [])

                if not log_events:
                    continue

                log_data = json.dumps(
                    log_events,
                    default=str
                )

                compressed_log_data = gzip.compress(
                    log_data.encode('utf-8')
                )

                timestamp = int(time.time())

                file_name = (
                    f'cloudwatch-logs/'
                    f'{log_stream_name}-'
                    f'{timestamp}.json.gz'
                )

                s3_client.put_object(
                    Bucket=S3_BUCKET_NAME,
                    Key=file_name,
                    Body=compressed_log_data,
                    ContentType='application/gzip'
                )

                uploaded_files.append(file_name)

            except Exception as e:

                print(
                    f"Error processing stream "
                    f"{log_stream_name}: {e}"
                )

    return {
        'statusCode': 200,
        'body': json.dumps({
            'message':
                f'Successfully uploaded '
                f'{len(uploaded_files)} log files to S3.',
            'files': uploaded_files
        })
    }
```

## 🔐 Required IAM Permissions

The Lambda execution role requires permissions to read CloudWatch Logs and write objects to S3.

Example permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:DescribeLogStreams",
        "logs:GetLogEvents"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::YOUR-S3-BUCKET/cloudwatch-logs/*"
    }
  ]
}
```

> For production, replace `Resource: "*"` with the specific CloudWatch Log Group ARN where practical.

## ⏰ 24-Hour Automation

Create an **Amazon EventBridge schedule/rule** to invoke the Lambda function every 24 hours.

Example schedule:

```text
rate(24 hours)
```

Flow:

```text
EventBridge
     │
     │ Every 24 Hours
     ▼
AWS Lambda
     │
     ▼
CloudWatch Logs
     │
     ▼
Gzip Compression
     │
     ▼
Amazon S3
```

## 📦 S3 Archive Structure

Archived files are stored using the following structure:

```text
cloudwatch-logs/
│
├── stream-1-1756900000.json.gz
├── stream-2-1756900000.json.gz
├── stream-1-1756986400.json.gz
└── stream-2-1756986400.json.gz
```

The timestamp helps identify when each archive was created.

## 🧪 Testing

You can test the Lambda manually using the AWS Lambda console.

Expected response:

```json
{
  "statusCode": 200,
  "body": {
    "message": "Successfully uploaded log files to S3.",
    "files": [
      "cloudwatch-logs/stream-1-xxxxx.json.gz"
    ]
  }
}
```

Then verify the S3 bucket:

```text
S3
└── cloudwatch-logs/
    └── *.json.gz
```

## ⚠️ Troubleshooting

### ResourceNotFoundException

If you receive:

```text
ResourceNotFoundException:
The specified log group does not exist.
```

Check the configured Log Group:

```python
LOG_GROUP = 'LOG-FROM-EC2'
```

Verify that:

* The CloudWatch Log Group exists.
* The Log Group name is correct.
* Lambda and CloudWatch Logs are in the **same AWS Region**.
* Lambda IAM role has CloudWatch Logs permissions.

You can verify the Log Group using:

```bash
aws logs describe-log-groups \
  --log-group-name-prefix LOG-FROM-EC2
```

## 🎯 Key Features

* ✅ Automated 24-hour log archival
* ✅ EC2 log centralization
* ✅ CloudWatch Logs integration
* ✅ Serverless processing with Lambda
* ✅ Multiple log stream support
* ✅ Pagination support
* ✅ JSON log conversion
* ✅ Gzip compression
* ✅ S3 long-term archival
* ✅ IAM-based security
* ✅ Cost-optimized log retention

## 📌 Resume Description

**CloudLogVault — Cost-Optimized AWS Log Archival**

> Built an automated **EC2 → CloudWatch Logs → Lambda → S3** pipeline that archives the previous 24 hours of logs to compressed S3 storage, reducing long-term logging storage costs through **Gzip compression and cost-effective S3 retention**.

## 👨‍💻 Skills Demonstrated

```text
AWS
├── EC2
├── CloudWatch Logs
├── Lambda
├── S3
├── EventBridge
└── IAM

Programming
├── Python
├── Boto3
├── JSON
└── Gzip

DevOps
├── Automation
├── Log Management
├── Cost Optimization
└── Serverless Architecture
```
