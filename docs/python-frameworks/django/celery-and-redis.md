# Celery & Redis Background Tasks in Django

> This guide provides a comprehensive, production-grade interview preparation handbook for asynchronous background task processing with **Celery**, in-memory caching and messaging with **Redis**, and their deep integration into **Django**.
>
> Written from the perspective of an experienced backend engineer who has designed, deployed, scaled, and debugged high-volume distributed task systems in production.

---

# 1. Why Background Tasks? (The Core Problem)

## Interview Question

**Why can't Django handle long-running operations directly inside views? What is a background task, and what problem does it actually solve?**

---

### Answer

To understand Celery and Redis, you first have to understand how web servers like Django, Nginx, and Gunicorn communicate.

When a user clicks a button in their browser, an HTTP request travels to your web server:

```
Client (Browser)                           Django Web Server (Gunicorn)
       │                                                 │
       │ ──► POST /checkout/ ──────────────────────────► │
       │                                                 │ ──► Saves order to DB (5ms)
       │                                                 │ ──► Generates Invoice PDF (2,500ms)
       │                                                 │ ──► Talks to SendGrid SMTP (3,000ms)
       │                                                 │ ──► Syncs with Salesforce API (4,000ms)
       │ ◄── 200 OK (After 9.5 SECONDS!) ─────────────── │
```

Look at what happened:

The customer stared at a spinning browser wheel for **9.5 seconds**.

Even worse, during those 9.5 seconds, that Gunicorn worker process was **completely frozen**. It could not accept any other requests from other users.

If you have 8 Gunicorn workers, and 8 customers click "Checkout" at the same moment, **your entire website freezes for everyone on the internet**.

New incoming requests queue up, hit Gunicorn's 30-second timeout, and Nginx starts returning **HTTP 504 Gateway Timeout**.

---

### What is a Background Task?

A background task is simply code that is executed **asynchronously, outside of the HTTP request-response cycle**.

The web request should do only what is strictly necessary to acknowledge the user (like saving the order to the database) and immediately return a response.

Everything else that takes time is offloaded to a separate worker process running in the background:

```
Client (Browser)                 Django (Web Worker)              Celery (Background Worker)
       │                                 │                                    │
       │ ──► POST /checkout/ ──────────► │                                    │
       │                                 │ ──► Save order to DB (5ms)        │
       │                                 │ ──► Enqueue task to Redis (2ms) ──►│
       │ ◄── 201 Created (in 15ms!) ─────│                                    │ ──► Generates PDF
       │                                                                      │ ──► Sends Email
       │                                                                      │ ──► Syncs CRM
```

Now the user gets an instant response in 15ms.

The heavy lifting happens in the background on separate machines or separate CPU cores. Your web server stays lightning-fast and responsive.

---

# 2. Celery Architecture & Internals

## Interview Question

**What is Celery, what are its core components, and what actually happens when you call `.delay()` on a task?**

---

### Answer

Celery is an **asynchronous distributed task queue** based on distributed message passing.

It consists of four distinct architectural components:

```
┌─────────────────┐       Task Message        ┌─────────────────┐
│     Producer    │ ────────────────────────► │     Broker      │
│  (Django App)   │        (JSON payload)     │     (Redis)     │
└─────────────────┘                           └─────────────────┘
                                                       │
                                                       │ Pulls task
                                                       ▼
┌─────────────────┐        Task Result        ┌─────────────────┐
│ Result Backend  │ ◄──────────────────────── │    Consumer     │
│ (Redis / PostgreSQL) (Status & Return Value)│ (Celery Worker) │
└─────────────────┘                           └─────────────────┘
```

1. **The Producer (Django Application)**:
   The code that creates and dispatches work. When your view calls `send_welcome_email.delay(user_id=42)`, Django acts as the producer.
2. **The Message Broker (Redis or RabbitMQ)**:
   The post office. Django doesn't talk directly to Celery workers. Django deposits a message into the broker. The broker holds tasks safely in a queue until a worker is ready to consume them.
3. **The Consumer (Celery Worker)**:
   An independent Python process running on your server (often managed by systemd or Docker). It continuously polls the message broker, pulls tasks off the queue, imports the Python function, and executes it.
4. **The Result Backend (Redis / Database / Cache)**:
   A storage layer where Celery writes the final state (`SUCCESS`, `FAILURE`, `RETRY`) and the return value of the function so Django can inspect it later if needed.

---

### What happens behind the scenes when `.delay()` is called?

When you write:

```python
send_email_task.delay(user_id=42, subject="Welcome!")
```

Django does **not** execute the function. Here is the step-by-step internal execution:

1. **Task ID Generation**: Celery generates a unique UUID4 string for this task execution (e.g. `f47ac10b-58cc-4372-a567-0e02b2c3d479`).
2. **Message Serialization**: Celery takes the function name (`"myapp.tasks.send_email_task"`), the task ID, and the arguments (`args=[42]`, `kwargs={'subject': 'Welcome!'}`) and serializes them into a standardized JSON string.
3. **Broker Dispatch**: Django establishes a network connection to Redis and executes an `LPUSH` (or `RPUSH`) command, inserting the JSON message onto a Redis list named `celery`.
4. **Immediate Return**: `.delay()` returns an `AsyncResult` object containing the task ID immediately (typically in 1–2ms).
5. **Worker Consumption**: A Celery worker process waiting on Redis receives the task via a blocking pop (`BRPOP`), deserializes the JSON string, matches the task name to its registered Python functions, and executes `send_email_task(42, subject="Welcome!")`.
6. **Result Storage**: If a result backend is configured, the worker writes the return value into Redis under the key `celery-task-meta-<task_id>`.

---

### Task Serialization: Why JSON instead of Pickle?

In older versions of Celery, the default serializer was Python's `pickle`.

In modern production systems, **always use `JSON` serialization**:

```python
# settings.py
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_ACCEPT_CONTENT = ['json']
```

### Why Pickle is dangerous:
`pickle` allows serializing arbitrary Python code and bytecode objects. If an attacker gains access to your Redis broker, they can inject malicious pickled payloads that execute arbitrary shell commands when unpacked by the Celery worker!

JSON serialization only allows plain primitive data (strings, numbers, lists, dictionaries), completely neutralizing remote code execution vulnerabilities.

---

### Worker Concurrency & Worker Pools

When you launch a Celery worker:

```bash
celery -A myproject worker --concurrency=4
```

Celery spawns execution units according to the selected **pool**:

```
Pool Type       Mechanism                Best For
─────────       ─────────                ────────
prefork         Multi-process (fork)     CPU-bound tasks, standard Django code
threads         Multi-threading          I/O-bound tasks with low memory
gevent/eventlet Cooperative greenlets    High-concurrency network I/O (WebSockets, scrapers)
solo            Single thread/process    Local debugging, profiling
```

- **`prefork` (Default)**: Spawns multiple distinct Python operating system processes. Bypasses the Python Global Interpreter Lock (GIL). If one worker process crashes or leaks memory, the other worker processes continue running unharmed.
- **Prefetch Multiplier (`CELERYD_PREFETCH_MULTIPLIER`)**:
  How many tasks each worker process fetches from the broker in advance.
  - If you have many fast tasks (5ms each), a higher prefetch (e.g. `4`) improves throughput.
  - If you have long-running tasks (e.g. 5 minutes each), set `CELERYD_PREFETCH_MULTIPLIER = 1`. Otherwise, one worker might hog 10 heavy tasks while three other workers sit completely idle!

---

# 3. Redis: In-Memory Engine, Cache & Broker

## Interview Question

**What is Redis, why is it so fast, and how can the exact same database serve as both a cache and a message broker?**

---

### Answer

Redis stands for **REmote DIctionary Server**.

It is an open-source, in-memory key-value data structure store.

### Why is Redis so fast? (Sub-millisecond latency)

1. **In-Memory Operations**: Traditional databases (PostgreSQL, MySQL) read and write to disk (NVMe/SSD). Disk I/O requires kernel system calls and mechanical latency. Redis stores its entire dataset in RAM. Reading from RAM is orders of magnitude faster than reading from SSD.
2. **Single-Threaded Event Loop**: Redis executes incoming commands on a single thread using non-blocking I/O multiplexing (`epoll` / `kqueue`). Because it is single-threaded, there are **zero thread locks, zero mutexes, zero semaphores, and zero race conditions** inside the Redis server engine. Every command executes deterministically and atomically.
3. **Optimized Low-Level C Structures**: Redis data types are implemented in ultra-optimized C (sds strings, ziplists, skiplists, hashtables) designed for minimum CPU cache misses.

---

### Redis Core Data Structures & Practical Use Cases

| Data Type | Description | Production Example |
|---|---|---|
| **String** | Binary-safe strings, integers, or serialized JSON (up to 512MB). | Caching API responses, user session tokens, rate limit counters (`INCR`). |
| **List** | Linked list of strings ordered by insertion. | **Task queues** (`LPUSH` to insert, `BRPOP` for workers to wait and consume). |
| **Set** | Unordered collection of unique strings. | Tracking unique visitors, tagging systems, mutual followers. |
| **Sorted Set (ZSet)**| Collection of unique members scored by a floating-point value. | **Leaderboards**, sliding window rate limiters, priority queues. |
| **Hash** | Mapping of string fields to string values (a mini dictionary). | Storing user profiles (`HSET user:42 name "Alice" email "a@a.com"`). |
| **Streams** | Append-only log with consumer groups (like a lightweight Kafka). | Audit logs, event sourcing, distributed messaging with multiple consumers. |
| **Pub/Sub** | Publisher-Subscriber message fanout. | Live chat notifications, WebSocket broadcasting. |

---

### Redis as Broker vs Redis as Result Backend

In Celery architecture, people often say *"We use Redis for Celery"*. But Redis actually performs two completely separate jobs:

```
Django App ──► [ Redis List: "celery" ] ──► Celery Worker
                    (The BROKER)
                         │
                         ▼
Django App ◄── [ Redis String: "celery-task-meta-UUID" ] ◄── Celery Worker
                    (The RESULT BACKEND)
```

1. **Redis as Broker**:
   - Holds tasks waiting to be processed.
   - Uses Redis **Lists** (`LPUSH` and `BRPOP`).
   - High write throughput; ephemeral (tasks are removed as soon as a worker picks them up).
2. **Redis as Result Backend**:
   - Stores the return values and exceptions of finished tasks.
   - Uses Redis **Strings** with keys like `celery-task-meta-<UUID>`.
   - Requires setting a Time-To-Live (TTL) so results automatically expire from RAM.

---

### Can you use the same Redis instance for both Broker and Cache?

**Technically yes, but in high-traffic production, NO.**

In production, you should run **separate Redis instances** (or at minimum separate Redis database numbers like DB 0 for broker and DB 1 for cache):

```python
# settings.py

# Celery Broker (Database 0)
CELERY_BROKER_URL = 'redis://localhost:6379/0'

# Celery Result Backend (Database 1)
CELERY_RESULT_BACKEND = 'redis://localhost:6379/1'

# Django Cache Framework (Database 2)
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://localhost:6379/2',
    }
}
```

### Why separation is mandatory in production:
1. **Eviction Policies Clash**:
   A cache needs an LRU eviction policy (`allkeys-lru`) so that when RAM fills up, Redis automatically deletes old cache keys to make room for new ones.
   If your Celery broker shares that Redis instance, Redis might **delete pending Celery task messages** to make room for an HTML cache fragment, silently dropping customer orders or emails!
2. **Failure Isolation**:
   If an unoptimized query dumps a 500MB dataset into the Django cache, causing Redis to run out of memory, your task broker will crash along with it.

---

# 4. Celery + Redis + Django Integration

## Interview Question

**Walk me through the standard setup of Celery with Django. What files are created, and how does Celery discover tasks across different apps?**

---

### Project Layout

```
my_project/
├── my_project/
│   ├── __init__.py         → Imports celery_app so @shared_task works
│   ├── celery.py           → Celery instance creation & configuration
│   ├── settings.py         → Celery broker & result backend settings
│   ├── urls.py
│   └── wsgi.py
└── orders/
    ├── models.py
    ├── views.py
    └── tasks.py            → Background tasks declared here
```

---

### Step 1: Create `my_project/celery.py`

```python
# my_project/celery.py
import os
from celery import Celery

# Set default Django settings module for celery program
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'my_project.settings')

app = Celery('my_project')

# Read configuration from Django settings, using the CELERY_ namespace
app.config_from_object('django.conf:settings', namespace='CELERY')

# Automatically find tasks.py inside all installed Django apps
app.autodiscover_tasks()
```

---

### Step 2: Initialize in `my_project/__init__.py`

```python
# my_project/__init__.py
from .celery import app as celery_app

__all__ = ('celery_app',)
```

This guarantees that the Celery app is loaded when Django starts, allowing the `@shared_task` decorator to bind properly.

---

### Step 3: Configure `settings.py`

```python
# settings.py
CELERY_BROKER_URL = 'redis://localhost:6379/0'
CELERY_RESULT_BACKEND = 'redis://localhost:6379/1'
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TIMEZONE = 'UTC'

# Expire task results after 24 hours (prevents memory bloat in Redis)
CELERY_RESULT_EXPIRES = 86400

# Worker performance settings
CELERY_TASK_ACKS_LATE = True               # Re-queue task if worker crashes mid-execution
CELERY_WORKER_PREFETCH_MULTIPLIER = 1      # Fair task distribution for heavy jobs
```

---

### Step 4: Writing Tasks with `@shared_task`

```python
# orders/tasks.py
import logging
from celery import shared_task
from .models import Order

logger = logging.getLogger(__name__)

@shared_task(bind=True, max_retries=3, default_retry_delay=60)
def process_order_payment(self, order_id):
    try:
        order = Order.objects.get(id=order_id)
        # Process payment with Stripe ...
        logger.info(f"Payment successful for Order #{order.id}")
        return {"status": "success", "order_id": order.id}
    except ExternalPaymentGatewayError as exc:
        # Retry with exponential backoff: 60s, 120s, 240s
        logger.warning(f"Payment gateway down. Retrying Order #{order_id}...")
        raise self.retry(exc=exc, countdown=60 * (2 ** self.request.retries))
```

---

# 5. The #1 Production Trap: `transaction.on_commit()`

This is universally the most frequent senior Celery interview question.

## Interview Question

**What happens if you trigger a Celery task inside a Django view immediately after creating a model instance? Why does the worker raise `DoesNotExist`, and how does `transaction.on_commit()` fix it?**

---

### The Catastrophic Race Condition

Look closely at this view code:

```python
# BUGGY PRODUCTION CODE
from django.db import transaction
from .models import Order
from .tasks import send_order_confirmation

def checkout(request):
    with transaction.atomic():
        order = Order.objects.create(user=request.user, total=99.95)
        
        # Enqueueing the Celery task!
        send_order_confirmation.delay(order.id)
        
        # Some extra slow database logic ...
        update_inventory(request.user)

    return JsonResponse({"status": "order placed"})
```

### Why this crashes in production:

1. `with transaction.atomic():` opens an SQL database transaction (`BEGIN;`).
2. `Order.objects.create()` sends `INSERT INTO orders ...` to PostgreSQL. **The transaction has NOT committed yet!**
3. `send_order_confirmation.delay(order.id)` sends a message to Redis.
4. Redis hands the message to a Celery worker on another CPU core in **1 millisecond**.
5. The Celery worker runs:
   ```python
   order = Order.objects.get(id=order_id)
   ```
6. **Crash! `Order.DoesNotExist`!**

### Why couldn't the worker find the order?
Because of **Database Transaction Isolation (Read Committed)**!

PostgreSQL isolates uncommitted rows from all other database connections. Until Django's view finishes and sends `COMMIT;`, no other process on earth can see that new order row.

The Celery worker ran faster than the database could commit!

---

### The Solution: `transaction.on_commit()`

Always schedule tasks using Django's `transaction.on_commit()` hook:

```python
from django.db import transaction

def checkout(request):
    with transaction.atomic():
        order = Order.objects.create(user=request.user, total=99.95)
        update_inventory(request.user)
        
        # Guaranteed to execute ONLY AFTER the database transaction commits!
        transaction.on_commit(lambda: send_order_confirmation.delay(order.id))

    return JsonResponse({"status": "order placed"})
```

Now, if the database transaction succeeds, Django commits the row first, and *then* pushes the message to Redis. The worker is guaranteed to find the row.

If an exception occurs and the database rolls back, the lambda function never executes, preventing phantom background tasks!

---

# 6. Idempotency & Delivery Guarantees

## Interview Question

**Does Celery guarantee that a task will execute exactly once? How do you make a Celery task idempotent in production?**

---

### Answer: The Myth of "Exactly-Once" Execution

In distributed computing, **there is no such thing as guaranteed exactly-once delivery across network boundaries**.

Celery operates under **At-Least-Once delivery** (when `task_acks_late = True`):

```
1. Celery Worker pulls task from Redis.
2. Worker finishes executing the task (e.g. charges customer credit card).
3. Right before worker sends confirmation ACK back to Redis, the server loses power!
4. Redis never receives the ACK.
5. Redis assumes the worker died and hands the exact same task to Worker 2.
6. Worker 2 charges the customer credit card a SECOND TIME!
```

If your tasks are not **idempotent**, your system will eventually charge customers twice, send duplicate emails, or create duplicate invoices.

---

### What is an Idempotent Task?

An operation is **idempotent** if running it once produces the exact same business outcome as running it five times:

$$\text{Operation}(x) = \text{Operation}(\text{Operation}(x))$$

---

### How to Build Idempotent Tasks with Redis Locks

Use an atomic **Redis distributed lock** and track completed operations:

```python
from celery import shared_task
from django.core.cache import cache
from .models import Payment

@shared_task(bind=True)
def charge_customer_order(self, order_id):
    # Unique lock key for this specific business operation
    lock_key = f"lock:payment:order:{order_id}"
    
    # 1. Attempt to acquire an atomic Redis lock with a 60-second TTL
    # cache.add() uses Redis SETNX: returns True ONLY if key did not exist!
    acquired = cache.add(lock_key, "locked", timeout=60)
    
    if not acquired:
        logger.warning(f"Task for Order #{order_id} is already being processed. Skipping duplicate.")
        return "Already processing"

    try:
        # 2. Check if business operation was already finalized in DB
        payment = Payment.objects.filter(order_id=order_id, status='completed').first()
        if payment:
            return "Already completed"

        # 3. Perform the non-idempotent operation (Charge Stripe)
        charge = stripe.Charge.create(amount=5000, currency="usd", ...)
        
        # 4. Save state
        Payment.objects.create(order_id=order_id, status='completed', charge_id=charge.id)
        return "Success"

    finally:
        # 5. Release lock
        cache.delete(lock_key)
```

Now, even if network retries or worker restarts cause the task to be delivered three times, the lock and database state checks guarantee the customer is charged exactly once.

---

# 7. Celery Workflows: Canvas (Chains, Groups, Chords)

## Interview Question

**How do you coordinate dependent tasks in Celery? What is the difference between a `chain`, a `group`, and a `chord`?**

---

### Answer

In real applications, complex jobs consist of multiple sub-tasks that depend on each other. Celery provides **Canvas** primitives to compose workflows:

---

### 1. Chain: Sequential Execution ($A \rightarrow B \rightarrow C$)

Each task runs one after the other. The return value of task A is automatically passed as the first argument to task B:

```python
from celery import chain
from .tasks import fetch_user_data, sanitize_data, generate_report

# Runs fetch -> sanitize -> generate sequentially
workflow = chain(
    fetch_user_data.s(user_id=42),
    sanitize_data.s(),
    generate_report.s()
)
workflow.apply_async()
```

---

### 2. Group: Parallel Execution ($A \parallel B \parallel C$)

Executes multiple tasks concurrently in parallel across available workers:

```python
from celery import group
from .tasks import process_image

# Processes 10 images concurrently across available worker cores
job = group(process_image.s(image_id) for image_id in image_ids)
job.apply_async()
```

---

### 3. Chord: Parallel Execution + Callback Barrier

Executes a group of tasks in parallel. When **all** of them finish, Celery collects their return values into a list and triggers a final callback task:

```python
from celery import chord
from .tasks import fetch_store_sales, calculate_total_quarterly_revenue

# 1. Fetch sales for 4 stores in parallel
header = [fetch_store_sales.s(store_id=i) for i in [1, 2, 3, 4]]

# 2. When all 4 finish, sum them up in the callback
callback = calculate_total_quarterly_revenue.s()

chord(header)(callback)
```

---

# 8. Periodic Tasks with Celery Beat

## Interview Question

**What is Celery Beat, and why must you run only ONE instance of Celery Beat in production?**

---

### Answer

Celery Beat is a **scheduler**.

Think of Celery Beat as a distributed cron service. It wakes up at specified intervals (e.g. every hour, or every night at 2:00 AM) and enqueues task messages into the Redis broker for workers to execute.

```
Celery Beat (Scheduler)
       │ Every 30 minutes
       ▼
Pushes task to Redis Broker
       │
       ▼
Worker 1 / Worker 2 executes task
```

```python
# settings.py
from celery.schedules import crontab

CELERY_BEAT_SCHEDULE = {
    # Run database cleanup every night at midnight UTC
    'nightly_database_cleanup': {
        'task': 'core.tasks.cleanup_expired_sessions',
        'schedule': crontab(hour=0, minute=0),
    },
    # Sync exchange rates every 15 minutes
    'sync_currency_rates': {
        'task': 'billing.tasks.sync_forex_rates',
        'schedule': 900.0,  # 900 seconds
    },
}
```

---

### The Production Warning: Never run multiple Beat instances!

You can scale Celery workers to 100 instances across 20 servers.

**However, you must run strictly ONE single instance of Celery Beat.**

If you run 3 Celery Beat instances across 3 web containers, all 3 schedulers will wake up at midnight and push 3 duplicate cleanup tasks to Redis!

---

# 9. Nine Practical Production Project Examples

Here are complete, practical implementations of the 9 most common production background task scenarios.

---

### Example 1: Sending Transactional Emails

**Why Background Task?** Talking to SMTP or external APIs (SendGrid/Mailgun) takes 1–3 seconds and fails frequently on network blips.

```python
# tasks.py
from celery import shared_task
from django.core.mail import send_mail

@shared_task(bind=True, max_retries=3, default_retry_delay=30)
def send_welcome_email(self, user_email, user_name):
    try:
        send_mail(
            subject="Welcome to Our Platform!",
            message=f"Hi {user_name}, thanks for signing up.",
            from_email="no-reply@company.com",
            recipient_list=[user_email],
            fail_silently=False,
        )
    except Exception as exc:
        raise self.retry(exc=exc)
```

---

### Example 2: Generating Large PDFs / Invoices

**Why Background Task?** Compiling HTML/CSS into PDF using WeasyPrint or ReportLab requires high CPU and 5–10 seconds.

```python
# tasks.py
from celery import shared_task
from .models import Invoice
from .services import pdf_generator, s3_storage

@shared_task
def generate_invoice_pdf(invoice_id):
    invoice = Invoice.objects.get(id=invoice_id)
    pdf_bytes = pdf_generator.render_to_pdf("invoice_template.html", {"invoice": invoice})
    
    # Upload to AWS S3
    s3_url = s3_storage.upload_file(f"invoices/{invoice.number}.pdf", pdf_bytes)
    
    invoice.pdf_url = s3_url
    invoice.status = 'ready'
    invoice.save(update_fields=['pdf_url', 'status'])
    return s3_url
```

---

### Example 3: Processing Uploaded Images (Resizing & Thumbnails)

**Why Background Task?** Parsing 20MB photos blocks the GIL and exhausts web worker memory.

```python
# tasks.py
from PIL import Image
from celery import shared_task
from .models import UserProfile

@shared_task
def generate_profile_thumbnails(profile_id):
    profile = UserProfile.objects.get(id=profile_id)
    with Image.open(profile.original_avatar.path) as img:
        img.thumbnail((200, 200))
        thumbnail_path = f"thumbnails/{profile.id}.jpg"
        img.save(thumbnail_path, "JPEG")
        
        profile.thumbnail = thumbnail_path
        profile.save(update_fields=['thumbnail'])
```

---

### Example 4: Calling Slow Third-Party APIs with Circuit Breaking

**Why Background Task?** Protects user requests from freezing when external partners degrade.

```python
# tasks.py
import httpx
from celery import shared_task

@shared_task(bind=True, max_retries=5)
def verify_tax_id_external(self, vendor_id, tax_id):
    url = f"https://api.tax-authority.gov/verify/{tax_id}"
    try:
        response = httpx.get(url, timeout=5.0)
        response.raise_for_status()
        return response.json()
    except (httpx.RequestError, httpx.HTTPStatusError) as exc:
        # Exponential backoff retry: 10s, 20s, 40s, 80s
        countdown = 10 * (2 ** self.request.retries)
        raise self.retry(exc=exc, countdown=countdown)
```

---

### Example 5: Bulk Notification Dispatch (Chunking 100,000 Users)

**Why Background Task?** Looping through 100,000 push notification tokens in a view crashes the server.

```python
# tasks.py
from celery import shared_task
from .models import DeviceToken
from .services import push_service

@shared_task
def send_bulk_push_notification(notification_message):
    # Process in chunks of 1,000 using iterator() to keep memory low
    tokens = DeviceToken.objects.filter(is_active=True).values_list('token', flat=True)
    batch = []
    
    for token in tokens.iterator(chunk_size=1000):
        batch.append(token)
        if len(batch) >= 1000:
            push_service.send_batch(batch, notification_message)
            batch.clear()
            
    if batch:
        push_service.send_batch(batch, notification_message)
```

---

### Example 6: Scheduled Database Cleanup (Celery Beat)

**Why Background Task?** Expired sessions and orphaned draft carts accumulate in PostgreSQL.

```python
# tasks.py
from celery import shared_task
from django.utils import timezone
from datetime import timedelta
from .models import Cart

@shared_task
def delete_stale_abandoned_carts():
    cutoff = timezone.now() - timedelta(days=30)
    # Bulk delete directly in SQL without instantiating model objects
    deleted_count, _ = Cart.objects.filter(is_completed=False, updated_at__lt=cutoff).delete()
    return f"Purged {deleted_count} abandoned carts."
```

---

### Example 7: AI / LLM Document Analysis & Embeddings

**Why Background Task?** OpenAI / Anthropic API calls take 5 to 30 seconds to stream or analyze text.

```python
# tasks.py
from celery import shared_task
from .models import Document
from .services import llm_service

@shared_task(time_limit=120)  # Hard kill after 2 minutes
def analyze_document_with_ai(document_id):
    doc = Document.objects.get(id=document_id)
    doc.status = 'processing'
    doc.save(update_fields=['status'])
    
    summary = llm_service.generate_summary(doc.raw_text)
    embeddings = llm_service.create_embeddings(doc.raw_text)
    
    doc.summary = summary
    doc.embedding_vector = embeddings
    doc.status = 'completed'
    doc.save(update_fields=['summary', 'embedding_vector', 'status'])
```

---

### Example 8: Heavy Financial Reporting & Data Aggregation

**Why Background Task?** Aggregating 5 million transaction rows locks tables and exceeds HTTP timeouts.

```python
# tasks.py
import pandas as pd
from celery import shared_task
from django.db import connection
from .models import FinancialReport

@shared_task
def compile_annual_financial_report(report_id, year):
    report = FinancialReport.objects.get(id=report_id)
    
    # Execute analytical SQL aggregation
    query = "SELECT month, SUM(revenue), SUM(expenses) FROM ledger WHERE year = %s GROUP BY month"
    df = pd.read_sql(query, connection, params=[year])
    
    # Perform analytical computation
    summary_metrics = {
        "annual_gross": float(df['sum'].sum()),
        "profit_margin": float((df['sum'] - df['expenses']).sum() / df['sum'].sum())
    }
    
    report.metrics = summary_metrics
    report.status = 'generated'
    report.save(update_fields=['metrics', 'status'])
```

---

### Example 9: Resilient Multi-Stage Order Fulfillment with Retries

**Why Background Task?** A real e-commerce checkout involves payment, inventory reservation, and warehouse ERP notification.

```python
# tasks.py
from celery import shared_task
from django.db import transaction
from .models import Order

@shared_task(bind=True, max_retries=5)
def fulfill_order_step(self, order_id):
    try:
        with transaction.atomic():
            order = Order.objects.select_for_update().get(id=order_id)
            if order.fulfillment_status == 'fulfilled':
                return "Already fulfilled"
                
            # Call 3rd-party logistics warehouse ERP
            warehouse_client.dispatch(order.id, order.items)
            
            order.fulfillment_status = 'fulfilled'
            order.save(update_fields=['fulfillment_status'])
            
    except WarehouseNetworkError as exc:
        # Retry with jitter to prevent thundering herd on warehouse server
        countdown = (2 ** self.request.retries) + random.uniform(1, 5)
        raise self.retry(exc=exc, countdown=countdown)
```

---

# 10. Senior Interview Questions & Production Scenarios

---

### Scenario 1: Why is your Celery queue growing indefinitely?
**Interviewer**: *"You look at Datadog and see your Redis queue size has grown from 10 tasks to 80,000 tasks over the last 3 hours. How do you troubleshoot?"*

**Answer**:
1. **Check Task Ingestion Rate vs Worker Processing Rate**: Are producers pushing 100 tasks/second while 4 workers can only process 10 tasks/second? (Scale up worker instances).
2. **Check for Unhandled Blocking Calls**: Did someone deploy a task that makes an external HTTP request without a timeout? If all worker threads are stuck waiting for a hung network connection, worker throughput drops to zero!
3. **Inspect the Deadlock / Heavy Task**: Use `celery -A myproject inspect active` to see what each worker process is currently executing.
4. **Task Routing / Priority Inversion**: Is a 30-minute CSV export running on the same queue as instant 50ms user emails? Heavy tasks starve fast tasks. Split tasks into separate queues: `emails` and `reports`.

---

### Scenario 2: What happens when a Celery worker process runs out of memory?
**Interviewer**: *"Python doesn't always release memory back to the OS immediately after processing large objects. How do you prevent Celery workers from memory bloating?"*

**Answer**:
Configure worker lifecycle limits in `settings.py`:

```python
# Automatically terminate and recreate a worker process after it serves 1,000 tasks
CELERY_WORKER_MAX_TASKS_PER_CHILD = 1000

# Automatically terminate and recreate a worker process if it exceeds 300MB of RAM
CELERY_WORKER_MAX_MEMORY_PER_CHILD = 300000  # 300MB in KB
```

When a child worker process finishes its 1,000th task or exceeds 300MB, the Celery master process terminates it cleanly and spawns a fresh worker child with clean RAM.

---

### Scenario 3: Task Time Limits (Soft vs Hard)
**Interviewer**: *"What is the difference between `time_limit` and `soft_time_limit`?"*

**Answer**:
- **`soft_time_limit`**: When reached, Celery raises a `SoftTimeLimitExceeded` exception inside the task. The task can catch this exception, log an error, clean up open files or database locks, and exit gracefully.
- **`time_limit`**: The hard limit. When reached, the operating system kernel sends `SIGKILL` (-9) to the worker process immediately. No cleanup code runs. Always configure both:

```python
# Allow 60 seconds soft limit to clean up, hard kill at 75 seconds
@shared_task(soft_time_limit=60, time_limit=75)
def scrape_website(url):
    try:
        ...
    except SoftTimeLimitExceeded:
        logger.error("Scraping took too long. Cleaning up resources...")
        save_partial_progress()
```

---

### Scenario 4: Flower — Production Task Monitoring
**Interviewer**: *"How do you monitor Celery workers in real-time in production?"*

**Answer**:
Deploy **Flower** (`celery -A myproject flower --port=5555`).

Flower is a real-time web-based monitoring dashboard for Celery that provides:
- Live worker status (online, offline, active threads).
- Task graphs and throughput rates.
- Interactive task termination and rate-limit adjustment without restarting workers.
- Historical execution times and exception stack traces.
