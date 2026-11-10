# Amazon S3 Storage & Presigned URLs with Boto3

> This guide provides a deep dive into Amazon Simple Storage Service (S3) using Boto3: file uploads, streaming, multipart handling, and the architecture of presigned URLs for scalable direct-to-S3 uploads.

---

# 1. Amazon S3 Core Concepts

## Interview Question

**What is Amazon S3? How does object storage differ from a traditional hierarchical filesystem?**

---

### Answer

Amazon S3 (Simple Storage Service) is an **object storage service** designed for 99.999999999% (11 nines) of data durability.

In a traditional filesystem (like Linux or Windows):
- Files are organized in a hierarchical tree of nested folders and directories.
- Modifying a file rewrites specific disk blocks.

In **S3 Object Storage**:
- The storage structure is completely **flat**. There are no real "directories" or "folders".
- An object consists of a **Bucket** (the top-level container) and a **Key** (the unique string identifier).
- When you see a key named `documents/2026/invoices/inv_101.pdf`, the slashes (`/`) are simply characters in the key string! The AWS console renders them as folders for human convenience.
- Objects are **immutable**. You cannot edit the first 10 bytes of an S3 file. You replace the entire object.

---

# 2. File Operations: Uploading & Downloading

## Interview Question

**What is the difference between `upload_file`, `upload_fileobj`, and `put_object` in Boto3?**

---

### Comparison of Upload Methods

```python
import boto3
from io import BytesIO

s3 = boto3.client('s3')

# 1. upload_file: Uploads a file directly from local DISK
s3.upload_file('/tmp/report.pdf', 'my-company-bucket', 'reports/annual_2026.pdf')

# 2. upload_fileobj: Uploads a file-like object directly from MEMORY (BytesIO)
in_memory_file = BytesIO(b"Hello world byte content")
s3.upload_fileobj(in_memory_file, 'my-company-bucket', 'greetings.txt')

# 3. put_object: Low-level API call (Loads entire payload into RAM at once)
s3.put_object(
    Bucket='my-company-bucket',
    Key='config.json',
    Body=b'{"version": 1}',
    ContentType='application/json'
)
```

### Which should you use in production?
- **Always prefer `upload_file` and `upload_fileobj`**: They use Boto3's internal transfer manager, which automatically handles **multipart uploads, retries, and multi-threading** for large files.
- **Avoid `put_object` for large files**: `put_object` attempts to send the entire payload in a single HTTP request. If the file is 500MB, it consumes 500MB of RAM and fails if a network drop occurs.

---

# 3. Handling Large Files with Multipart Uploads

## Interview Question

**How does Boto3 upload a 5 GB file to S3 without running out of server memory or failing on network blips?**

---

### Answer

For files larger than 100MB, uploading in a single HTTP request is unreliable. If the connection drops at 99%, the entire 5 GB must be re-sent from the beginning.

Boto3 solves this with **Multipart Uploads** using `TransferConfig`:

```
                      5 GB File on Disk
                             │
     ┌───────────────────────┼───────────────────────┐
     ▼                       ▼                       ▼
Part 1 (50MB)           Part 2 (50MB)           Part 3 (50MB) ...
     │                       │                       │
     ▼ (Thread 1)            ▼ (Thread 2)            ▼ (Thread 3)
     └───────────────────────┬───────────────────────┘
                             ▼
               Uploaded concurrently to S3
                             │
                             ▼
               S3 reassembles the complete 5GB file
```

```python
import boto3
from boto3.s3.transfer import TransferConfig

s3 = boto3.client('s3')

# Configure multipart upload behavior
config = TransferConfig(
    multipart_threshold=100 * 1024 * 1024,  # Use multipart if file > 100MB
    max_concurrency=10,                      # Upload using 10 concurrent threads
    multipart_chunksize=25 * 1024 * 1024,   # Each chunk is 25MB
    use_threads=True
)

s3.upload_file('huge_database_backup.iso', 'my-bucket', 'backup.iso', Config=config)
```

If Part 4 drops due to Wi-Fi packet loss, Boto3 retries **only Part 4**, saving bandwidth and time.

---

# 4. Presigned URLs: Scalable Direct-to-S3 Uploads

This is one of the most critical system design interview topics for backend engineers.

## Interview Question

**Why should web servers never handle multi-gigabyte file uploads directly? How do Presigned URLs work?**

---

### The Architecture Problem: Uploading Through the Backend

```
Client (Browser) ──────► Django / FastAPI Server ──────► Amazon S3
   (1 GB Video)             (Server holds 1GB RAM)          (Final Storage)
                             (Worker frozen for 3 mins!)
```

If 10 users upload a 1 GB video simultaneously:
- 10 application worker processes are completely tied up for 3 minutes streaming bytes.
- Your web server runs out of memory or Gunicorn workers, and your entire website crashes for all other users!

---

### The Production Solution: Presigned URLs

A **Presigned URL** is a temporary, cryptographically signed URL generated by your backend that grants permission to upload directly to a specific S3 bucket key:

```
Client (Browser)                     Django Backend                      Amazon S3
       │                                   │                                 │
       │ ──► 1. POST /api/upload-request ─►│                                 │
       │        {"filename": "video.mp4"}  │ ──► Generates Presigned URL     │
       │ ◄── 2. Returns signed S3 URL ──── │     (Expires in 15 mins)        │
       │                                                                     │
       │ ──► 3. PUT https://bucket.s3.amazonaws.com/video.mp4 ─────────────► │
       │        (Browser streams 1GB DIRECTLY to S3!)                        │
       │ ◄── 4. 200 OK (Upload Complete!) ────────────────────────────────── │
       │                                                                     │
       │ ──► 5. POST /api/confirm-upload ─►│                                 │
```

Notice: **The 1 GB video file NEVER touches your Python backend server.**

Your backend only spent 5 milliseconds generating a signed URL. Zero server RAM consumed. Zero worker threads blocked.

---

### Generating a Presigned PUT URL in Python

```python
import boto3
from botocore.exceptions import ClientError

def generate_presigned_upload_url(bucket_name, object_key, expiration=900):
    """Generates a presigned URL that allows a client to upload directly to S3 via HTTP PUT"""
    s3 = boto3.client('s3')
    try:
        url = s3.generate_presigned_url(
            ClientMethod='put_object',
            Params={
                'Bucket': bucket_name,
                'Key': object_key,
                'ContentType': 'application/pdf',
            },
            ExpiresIn=expiration  # URL expires in 15 minutes (900 seconds)
        )
        return url
    except ClientError as e:
        logger.error(f"Failed to generate presigned URL: {e}")
        return None
```

The frontend can now execute:

```javascript
// Browser JavaScript: Direct S3 Upload
await fetch(presignedUrl, {
    method: 'PUT',
    headers: { 'Content-Type': 'application/pdf' },
    body: fileObject
});
```

---

### Generating a Presigned Download URL (Securing Private Content)

If you run a SaaS that sells paid digital downloads (e.g. video courses, medical records, or PDF invoices), the S3 bucket must be **completely private**.

When an authorized user requests the file, generate a temporary signed download link:

```python
def generate_presigned_download_url(bucket_name, object_key, expiration=300):
    """Generates a temporary download link valid for 5 minutes"""
    s3 = boto3.client('s3')
    return s3.generate_presigned_url(
        ClientMethod='get_object',
        Params={'Bucket': bucket_name, 'Key': object_key},
        ExpiresIn=expiration
    )
```

The user receives a link valid for 5 minutes. If they share the link with an unauthorized friend after 5 minutes, S3 rejects the request with **HTTP 403 Access Denied**.
