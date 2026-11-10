# Essential AWS Services for Backend Developers: SES, EC2, CloudWatch & Secrets Manager

> This guide explores the essential AWS ecosystem services every Python backend engineer encounters: transactional email with SES, server credentials with EC2, logging with CloudWatch, and secret management.

---

# 1. Amazon SES (Simple Email Service)

## Interview Question

**How do you send transactional emails from Python using Amazon SES? Why does your initial setup fail to send emails to external users?**

---

### Answer

Amazon SES (Simple Email Service) is a managed email service designed for high deliverability (preventing your emails from going into the spam folder).

---

### Sending an Email with Boto3

```python
import boto3
from botocore.exceptions import ClientError

ses = boto3.client('ses', region_name='us-east-1')

def send_transactional_email(to_email, subject, body_html):
    try:
        response = ses.send_email(
            Source='no-reply@mycompany.com',  # Must be a verified domain!
            Destination={'ToAddresses': [to_email]},
            Message={
                'Subject': {'Data': subject, 'Charset': 'UTF-8'},
                'Body': {
                    'Html': {'Data': body_html, 'Charset': 'UTF-8'}
                }
            }
        )
        return response['MessageId']
    except ClientError as e:
        print(f"Failed to send email: {e.response['Error']['Message']}")
        return None
```

---

### The #1 Gotcha: The SES Sandbox

When you create a new AWS account, your SES service is placed in the **SES Sandbox**:

- You can **only** send emails to email addresses that have been manually verified in the AWS console!
- If you try to send to a customer (`customer@gmail.com`), SES rejects the request with: `Email address is not verified`.

### How to Fix:
You must submit a request to AWS Support to **Request Production Access**, providing your bounce/complaint handling procedures and proving you are not sending spam.

---

# 2. Amazon EC2 & IAM Instance Profiles

## Interview Question

**How does a Django application running on an EC2 instance access S3 or SQS without storing any access keys in `.env` files or settings?**

---

### Answer

This is one of the most fundamental cloud security interview questions.

Instead of creating an IAM user with permanent access keys, you attach an **IAM Role** to the EC2 instance via an **Instance Profile**:

```
┌────────────────────────────────────────────────────────┐
│                      EC2 Instance                      │
│                                                        │
│   Django Application                                  │
│          │                                             │
│          ▼ Calls boto3.client('s3')                    │
│   Boto3 checks credentials chain                      │
│          │                                             │
│          ▼ Hits local Link-Local IP                    │
│   http://169.254.169.254/latest/meta-data/             │
│   (Instance Metadata Service - IMDSv2)                 │
│          │                                             │
│          ▼ Returns temporary STS token                 │
└──────────┼─────────────────────────────────────────────┘
           │
           ▼ Authenticates to Amazon S3
   Amazon S3 Bucket
```

1. You create an IAM Role granting S3 read/write permissions.
2. In the AWS console, you attach this role to your EC2 virtual machine.
3. When Boto3 initializes, it automatically queries the internal AWS link-local IP (`169.254.169.254`).
4. AWS returns **temporary credentials** (an access key, secret key, and security token) valid for 6 hours.
5. Boto3 automatically refreshes the temporary token before it expires.

**Security Payoff**: Even if an attacker reads your entire hard drive, there are **zero permanent AWS access keys** to steal!

---

# 3. AWS Secrets Manager vs Parameter Store

## Interview Question

**How do you securely fetch database passwords and API keys at runtime using Boto3? What is the difference between Secrets Manager and Systems Manager Parameter Store?**

---

### Answer

Storing production database passwords in `.env` files on disk is risky if a server is compromised or backups leak.

Instead, your application fetches secrets dynamically into memory at startup:

```python
import boto3
import json

def get_secret(secret_name, region_name="us-east-1"):
    client = boto3.client("secretsmanager", region_name=region_name)
    response = client.get_secret_value(SecretId=secret_name)
    
    # Secrets are stored as a JSON string
    return json.loads(response["SecretString"])

# In settings.py:
secrets = get_secret("production/ecommerce/database")
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': secrets['DB_NAME'],
        'USER': secrets['DB_USER'],
        'PASSWORD': secrets['DB_PASSWORD'],
        'HOST': secrets['DB_HOST'],
        'PORT': secrets['DB_PORT'],
    }
}
```

---

### Secrets Manager vs Systems Manager Parameter Store

| Feature | AWS Secrets Manager | SSM Parameter Store |
|---|---|---|
| **Primary Use** | Highly sensitive secrets (Database credentials, API keys) | Configuration values, environment toggles, strings |
| **Automatic Rotation** | Built-in automatic rotation for RDS, Redshift, DocumentDB | No built-in automatic rotation |
| **Cost** | \$0.40 per secret per month | **Free tier** for standard parameters |
| **Best For** | Database credentials requiring scheduled rotation | General app configuration (`DEBUG=False`, service URLs) |

---

# 4. Amazon CloudWatch: Centralized Application Logging

## Interview Question

**How do you forward Django or FastAPI application logs to AWS CloudWatch in real-time?**

---

### Answer

When deploying applications across 5 autoscaling servers, logging to local text files (`/var/log/app.log`) is useless because logs are scattered across different machines.

You forward logs to **AWS CloudWatch Logs** using the Python **`watchtower`** library:

```python
# settings.py
import logging
import watchtower
import boto3

boto3_session = boto3.Session(region_name="us-east-1")

LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'handlers': {
        'watchtower': {
            'class': 'watchtower.CloudWatchLogHandler',
            'boto3_session': boto3_session,
            'log_group': 'ProductionBackendLogs',
            'stream_name': 'django-app',
        },
    },
    'loggers': {
        'django': {
            'handlers': ['watchtower'],
            'level': 'INFO',
        },
    },
}
```

Now, every `logger.info()` or `logger.error()` is streamed to CloudWatch in real-time, allowing you to set up alarms that ping Slack or PagerDuty when error rates spike!
