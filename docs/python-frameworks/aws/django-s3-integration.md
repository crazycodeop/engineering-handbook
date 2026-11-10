# Django & Amazon S3 Integration with `django-storages`

> This guide explains how to store user uploads and media files in Amazon S3 using `django-storages`, how to secure private documents, and how to test S3 interactions using Moto without touching real AWS infrastructure.

---

# 1. Configuring `django-storages` in Django 4.2+

## Interview Question

**How do you configure Django to use Amazon S3 for media file storage? What is the modern Django 4.2+ `STORAGES` setting?**

---

### Answer

In production, web containers (Docker, Heroku, AWS ECS) have **ephemeral filesystems**. If your Django application saves uploaded avatars to `/media/avatars/` on local disk, those files are permanently erased every time a new container deploys!

We offload media files to Amazon S3 using **`django-storages`** and **`boto3`**:

```bash
pip install django-storages boto3
```

Add `'storages'` to `INSTALLED_APPS` in `settings.py`.

---

### Modern Django 4.2+ `STORAGES` Configuration

In Django 4.2+, the legacy `DEFAULT_FILE_STORAGE` setting was replaced by the **`STORAGES` dictionary**:

```python
# settings.py
import os

AWS_STORAGE_BUCKET_NAME = os.environ.get('AWS_STORAGE_BUCKET_NAME')
AWS_S3_REGION_NAME = os.environ.get('AWS_S3_REGION_NAME', 'us-east-1')

# Optional custom CDN domain (e.g. CloudFront)
AWS_S3_CUSTOM_DOMAIN = f"{AWS_STORAGE_BUCKET_NAME}.s3.amazonaws.com"

STORAGES = {
    # 1. Default Storage (User-uploaded media files like avatars, documents)
    "default": {
        "BACKEND": "storages.backends.s3.S3Storage",
        "OPTIONS": {
            "bucket_name": AWS_STORAGE_BUCKET_NAME,
            "region_name": AWS_S3_REGION_NAME,
            "default_acl": "private",  # Secure: Objects are private by default!
            "file_overwrite": False,    # Appends unique hash if file with same name exists
            "querystring_auth": True,   # Automatically generates presigned URLs for private files!
            "querystring_expire": 3600, # Links expire after 1 hour
        },
    },
    # 2. Static Files Storage (CSS, JavaScript, Images)
    "staticfiles": {
        "BACKEND": "django.contrib.staticfiles.storage.StaticFilesStorage",
    },
}

MEDIA_URL = f"https://{AWS_S3_CUSTOM_DOMAIN}/media/"
```

Now, any standard Django model field automatically writes directly to S3:

```python
# models.py
class UserProfile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    avatar = models.ImageField(upload_to="avatars/%Y/%m/")
    tax_document = models.FileField(upload_to="taxes/")
```

When `profile.avatar.save()` runs, Django uploads the bytes to S3 and saves only the relative path string (`avatars/2026/11/photo.jpg`) in PostgreSQL!

---

# 2. Private Media vs Public Media

## Interview Question

**How do you configure Django so that user avatars are publicly accessible, but customer tax documents require authenticated, expiring URLs?**

---

### Answer

Subclass `S3Storage` to create distinct storage backends:

```python
# core/storages.py
from storages.backends.s3 import S3Storage

# 1. Public Storage: Avatars, product photos (CDN cached, no signatures)
class PublicMediaStorage(S3Storage):
    location = 'public-media'
    default_acl = 'public-read'
    file_overwrite = False
    querystring_auth = False  # Direct URLs: https://mybucket.s3.amazonaws.com/public-media/...

# 2. Private Storage: Invoices, medical records, tax PDFs
class PrivateMediaStorage(S3Storage):
    location = 'private-documents'
    default_acl = 'private'
    file_overwrite = False
    querystring_auth = True   # Automatically appends ?AWSAccessKeyId=...&Signature=...
    querystring_expire = 300  # Links expire in 5 minutes!
```

In your models, specify the exact storage backend per field:

```python
# models.py
from core.storages import PublicMediaStorage, PrivateMediaStorage

class CustomerAccount(models.Model):
    # Public: Anyone can see the avatar via CDN
    avatar = models.ImageField(storage=PublicMediaStorage())

    # Private: Accessing .url generates an authenticated, signed link valid for 5 mins!
    w2_form = models.FileField(storage=PrivateMediaStorage())
```

---

# 3. Testing S3 in Django with Moto

## Interview Question

**Why should automated tests never connect to real AWS S3 buckets? How do you test S3 uploads using Moto?**

---

### Answer

Connecting to real AWS during unit tests causes:
- Slow tests (500ms network latency per test).
- Flaky tests (fails if internet drops).
- Financial costs (AWS charges for `PUT` and `GET` requests).
- Accidental data contamination.

We use **`moto`**, the official mock AWS library that intercepts Boto3 HTTP calls and simulates an in-memory AWS cloud locally:

```bash
pip install moto[s3]
```

```python
# tests/test_s3_upload.py
import pytest
import boto3
from moto import mock_aws
from django.core.files.uploadedfile import SimpleUploadedFile
from myapp.models import CustomerAccount

@pytest.mark.django_db
@mock_aws  # Intercepts all Boto3/S3 network calls in-memory!
def test_user_avatar_upload():
    # 1. Create simulated test bucket in memory
    s3 = boto3.client('s3', region_name='us-east-1')
    s3.create_bucket(Bucket='test-company-bucket')

    # 2. Simulate file upload in Django
    uploaded_file = SimpleUploadedFile("avatar.jpg", b"fake-jpeg-binary-data", content_type="image/jpeg")
    account = CustomerAccount.objects.create(avatar=uploaded_file)

    # 3. Verify object was created in the simulated S3 bucket!
    assert account.avatar.name is not None
    s3_objects = s3.list_objects_v2(Bucket='test-company-bucket')
    assert len(s3_objects['Contents']) == 1
    assert "avatar" in s3_objects['Contents'][0]['Key']
```

Zero network calls. Zero AWS charges. Complete test isolation running in 20 milliseconds!
