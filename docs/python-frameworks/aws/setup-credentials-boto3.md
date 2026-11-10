# AWS Setup, Local Credentials & Boto3 Configuration

> This guide provides a complete, step-by-step setup walkthrough for configuring AWS credentials for Python development, working with named profiles, and enforcing security best practices.

---

# 1. Step-by-Step AWS Setup for Developers

## Step 1: Install Boto3 and AWS CLI

In your terminal or virtual environment:

```bash
# 1. Install Boto3
pip install boto3

# 2. Verify AWS CLI is installed
aws --version
```

---

## Step 2: Configure Local Credentials with `aws configure`

Run the interactive configuration wizard:

```bash
aws configure
```

It prompts for four values:

```
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: ap-south-1
Default output format [None]: json
```

### What actually happened on your machine?

The AWS CLI created two configuration files in your user home directory:

1. **`~/.aws/credentials`** (Stores sensitive secrets):
   ```ini
   [default]
   aws_access_key_id = AKIAIOSFODNN7EXAMPLE
   aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
   ```
2. **`~/.aws/config`** (Stores non-sensitive settings):
   ```ini
   [default]
   region = ap-south-1
   output = json
   ```

Now, any Python script using Boto3 on your machine automatically discovers these credentials without any hardcoding!

---

# 2. Named Profiles: Managing Dev, Staging & Prod

## Interview Question

**How do you switch between different AWS accounts (e.g. Personal vs Company, or Dev vs Production) in Boto3?**

---

### Answer

In professional development, you frequently work across multiple AWS accounts.

You define **Named Profiles** in `~/.aws/credentials`:

```ini
[default]
aws_access_key_id = AKIA_DEFAULT_DEV...
aws_secret_access_key = secret_dev...

[company-prod]
aws_access_key_id = AKIA_PROD_ACCOUNT...
aws_secret_access_key = secret_prod...
```

---

### Using Named Profiles in Boto3

Create a dedicated `boto3.Session`:

```python
import boto3

# Explicitly load the company-prod profile
session = boto3.Session(profile_name="company-prod", region_name="us-east-1")

# Create clients using that specific session
s3 = session.client("s3")
response = s3.list_buckets()
```

---

# 3. Testing AWS Connectivity in Python

Use AWS STS (Security Token Service) to verify that your credentials are valid and active:

```python
import boto3
from botocore.exceptions import NoCredentialsError, ClientError

def verify_aws_connection():
    try:
        # STS get_caller_identity is the AWS equivalent of "whoami"
        sts = boto3.client("sts")
        identity = sts.get_caller_identity()
        
        print("Successfully authenticated to AWS!")
        print(f"Account ID: {identity['Account']}")
        print(f"User ARN:   {identity['Arn']}")
        return True

    except NoCredentialsError:
        print("ERROR: No AWS credentials found. Run 'aws configure' or set environment variables.")
        return False
    except ClientError as e:
        print(f"ERROR: AWS rejected credentials: {e.response['Error']['Message']}")
        return False

if __name__ == "__main__":
    verify_aws_connection()
```

---

# 4. Critical Security Rules: Never Leak Credentials

## Interview Question

**What happens if an AWS Access Key is accidentally committed to a public GitHub repository? How do you prevent credential leaks?**

---

### The 5-Minute Attack

Automated botnets scan GitHub's public commit firehose 24 hours a day, 7 days a week for strings matching `AKIA...`.

If you push an active AWS access key to GitHub:
1. Within **90 seconds**, an automated bot detects the commit.
2. The bot uses your credentials to spin up 50 expensive GPU instances (`p3.16xlarge`) across multiple AWS regions to mine cryptocurrency.
3. By the time you notice 4 hours later, your AWS account has accrued **\$12,000+ in unauthorized EC2 charges**!

---

### The Defense Checklist

1. **Never Hardcode Secrets**: Never write `aws_secret_access_key = "..."` inside Python files.
2. **Add to `.gitignore`**:
   ```gitignore
   .env
   *.pem
   ~/.aws/
   credentials
   ```
3. **Use Pre-Commit Hooks**: Install `detect-secrets` or `trufflehog` to scan staged files locally before `git commit` allows the commit to proceed.
4. **Use IAM Roles on Servers**: On AWS EC2 or ECS, never configure keys. Use IAM instance roles.
