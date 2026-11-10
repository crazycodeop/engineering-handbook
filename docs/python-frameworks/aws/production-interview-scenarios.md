# Production Architecture & AWS Interview Scenarios

> This guide explores real-world AWS cloud architecture diagrams, production incident scenarios, debugging `AccessDenied` errors, handling AWS API throttling, and high-frequency interview questions for Python developers.

---

# 1. Core AWS Architecture Diagrams

---

### Diagram 1: Scalable File Upload with Presigned URLs

```
Client (Browser / Mobile)                 Django Backend                 Amazon S3 Bucket
       │                                         │                               │
       │ ──► 1. POST /api/upload-request ──────► │                               │
       │        (filename: "resume.pdf")         │ ──► Generates Presigned URL   │
       │ ◄── 2. Returns signed S3 URL ────────── │     (Valid for 15 minutes)    │
       │                                                                         │
       │ ──► 3. PUT https://bucket.s3.amazonaws.com/resume.pdf ─────────────────► │
       │        (Browser streams 50MB file DIRECTLY to S3!)                      │
       │ ◄── 4. HTTP 200 OK (Upload Complete) ────────────────────────────────── │
       │                                                                         │
       │ ──► 5. POST /api/confirm-upload ──────► │                               │
       │        (Triggers background processing) │ ──► Enqueues Celery task      │
```

**Why this matters in interviews**:
The Django application server never touches the raw file bytes. It consumes zero memory and zero bandwidth, preventing web worker starvation!

---

### Diagram 2: Decoupled Processing with SQS

```
Django Web Server (Producer)              Amazon SQS Queue           Python Worker (Consumer)
       │                                         │                               │
       │ ──► SQS.send_message() ───────────────► │                               │
       │     (JSON: {"task": "process_doc"})     │                               │
       │ ◄── Message ID returned (in 15ms) ───── │                               │
       │                                         │                               │
       │                                         │ ◄── Long polling (20s) ────── │
       │                                         │ ──► Delivers Message A ─────► │
       │                                         │     (Sets 60s timeout)        │
       │                                         │                               │ ──► Executes task
       │                                         │ ◄── SQS.delete_message() ──── │
       │                                         │     (Permanently removed)     │
```

---

### Diagram 3: Complete Document Processing Pipeline

```
User uploads document via Presigned URL ──► Amazon S3 Bucket
                                                   │
                                                   ▼
Django Web Server triggers task ──────────► Redis Broker
                                                   │
                                                   ▼
Celery Worker pulls job ──────────────────► Reads PDF from S3
                                                   │
                                                   ├── Parses text & extracts keywords
                                                   ├── Calls AI / LLM for summary
                                                   │
                                                   ▼
Saves final structured results ───────────► PostgreSQL Database
                                                   │
                                                   ▼
Updates Redis Result Cache ───────────────► Client polls and sees "Complete!"
```

---

# 2. Production Debugging & Incident Scenarios

---

### Scenario 1: Debugging `botocore.exceptions.ClientError: AccessDenied (403)`

**Interviewer**: *"Your Django backend tries to upload a file to S3 in production and crashes with `ClientError: An error occurred (AccessDenied) when calling the PutObject operation`. How do you systematically debug this?"*

**Answer**:
A 403 `AccessDenied` error in S3 can be caused by any one of **four separate security layers**:

```
Step 1: Check the IAM Policy
        ├── Does the IAM Role / User have explicit "s3:PutObject" permission?
        └── Check the Resource ARN: Is it "arn:aws:s3:::my-bucket/*" (with /*)?

Step 2: Check S3 Block Public Access
        └── Are public ACLs blocked at the bucket or account level?
            (If code sends acl='public-read', S3 drops it with 403!)

Step 3: Check the S3 Bucket Policy
        └── Does the bucket policy contain an explicit "Deny" statement overriding IAM?

Step 4: Check KMS Encryption Permissions (If using SSE-KMS)
        └── Does the IAM role have "kms:GenerateDataKey" permission on the KMS key?
```

---

### Scenario 2: Handling AWS API Throttling (`RequestLimitExceeded`)

**Interviewer**: *"During a Black Friday traffic spike, your backend starts throwing `ClientError: ThrottlingException` or `RequestLimitExceeded` when calling SQS or DynamoDB. How do you handle AWS rate limits?"*

**Answer**:
1. **Configuring Boto3 Adaptive Retries**:
   By default, Boto3 retries failed requests up to 5 times. In high-traffic systems, enable Boto3's modern **adaptive retry mode**:
   ```python
   from botocore.config import Config

   # Adaptive retry implements exponential backoff with client-side rate limiting
   config = Config(
       retries={
           'max_attempts': 10,
           'mode': 'adaptive'  # Automatically throttles requests client-side!
       }
   )
   sqs = boto3.client('sqs', config=config)
   ```
2. **Exponential Backoff with Full Jitter**:
   Never retry on a fixed interval (`time.sleep(2)`). If 500 workers all retry at second 2, they cause a **Thundering Herd** problem that keeps the AWS API throttled. Add random jitter:
   $$\text{Sleep} = \text{random}(0, 2^{\text{retries}})$$

---

### Scenario 3: What is an SQS Dead-Letter Queue (DLQ)?

**Interviewer**: *"What is a Dead-Letter Queue (DLQ), and why is it mandatory for asynchronous worker pipelines?"*

**Answer**:
Imagine a client submits a malformed JSON payload that causes your Python worker to crash with a `ZeroDivisionError`:

```
1. Worker receives Message A.
2. Worker crashes with unhandled exception.
3. Visibility timeout expires (30s).
4. Worker receives Message A again.
5. Worker crashes again!
... (Cycles forever, eating CPU and logging 10,000 exceptions a day!)
```

This is called a **Poison Pill Message**.

### The Solution: Dead-Letter Queue (DLQ)
You configure a Redrive Policy on your SQS queue:

```json
{
    "deadLetterTargetArn": "arn:aws:sqs:us-east-1:123456789012:orders-dlq",
    "maxReceiveCount": 3
}
```

If a message fails processing **3 times**, SQS automatically moves it off the main queue and into the **Dead-Letter Queue**.

Your main worker queue continues running smoothly, while your engineering team inspects the failed messages in the DLQ to fix bugs.
