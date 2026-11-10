# Advanced Django & Production Engineering

> This guide covers caching architectures, background task processing, custom middleware, Django admin optimization, and production deployment considerations.

---

# 1. Caching Strategies & Cache Invalidation

## Interview Question

**What caching strategies are available in Django, and how do you prevent a "Cache Stampede" in production?**

---

### Answer

Caching stores expensive computation or database query results in fast in-memory storage (typically Redis or Memcached) so subsequent requests don't hit the primary database.

Django provides caching at four distinct granularities:

```
Granularity               Use Case
───────────               ────────
1. Per-Site Cache         Entire pages cached for all visitors (e.g. public blogs)
2. Per-View Cache         Specific view endpoints cached with @cache_page
3. Template Fragment      Specific dynamic sections of an HTML page
4. Low-Level Cache API    Arbitrary Python objects, dictionaries, or QuerySet results
```

---

### Strategy 1: Per-View Caching

When an endpoint's response changes infrequently:

```python
from django.views.decorators.cache import cache_page

# Cache the output of this view for 15 minutes (900 seconds)
@cache_page(60 * 15)
def product_catalog(request):
    products = Product.objects.all().select_related('category')
    return render(request, 'catalog.html', {'products': products})
```

---

### Strategy 2: Low-Level Cache API (Most common in APIs)

In REST APIs or microservices, caching raw data or serialized dictionaries is much more flexible than caching full HTML pages:

```python
from django.core.cache import cache

def get_top_sellers():
    cache_key = "top_selling_products"
    data = cache.get(cache_key)

    if data is None:
        # Cache miss: expensive database query
        data = list(Product.objects.filter(is_featured=True)[:10].values('id', 'name', 'price'))
        # Store in Redis for 10 minutes
        cache.set(cache_key, data, timeout=600)

    return data
```

Django also provides `cache.get_or_set()`, which wraps this pattern in a single call:

```python
data = cache.get_or_set(
    'top_selling_products',
    lambda: list(Product.objects.filter(is_featured=True)[:10].values('id', 'name', 'price')),
    timeout=600
)
```

---

### What is a "Cache Stampede" (Dogpiling), and how do you prevent it?

Imagine your homepage receives 1,000 requests per second.

The product list is cached in Redis with a 10-minute TTL.

At minute 10:00, the cache key expires.

At minute 10:01, **1,000 concurrent requests all discover a cache miss simultaneously**.

All 1,000 requests query PostgreSQL at the exact same second to recalculate the exact same data. The database CPU spikes to 100%, connection pools exhaust, and the site crashes.

This is a **Cache Stampede**.

### Production Mitigations:

1. **Distributed Locks (Redis Lock)**:
   When a cache miss occurs, the first worker acquires an exclusive Redis lock to regenerate the data. The other 999 requests wait a few milliseconds or serve stale data until the lock releases.
2. **Probabilistic Early Expiration (XFetch)**:
   Instead of expiring abruptly, workers probabilistically refresh the cache as the TTL approaches zero, ensuring the key is renewed *before* it actually expires.
3. **Background Cache Warming**:
   Never let user requests calculate high-traffic keys. Run a Celery periodic beat task every 5 minutes to compute the data and push it into Redis. The user request *only* reads from cache and never queries the DB directly.

---

# 2. Static Files vs Media Files

## Interview Question

**What is the difference between Static and Media files in Django, and why is `collectstatic` required for production?**

---

### Answer

Although both are files served to the user, Django treats them very differently:

```
Static Files                                 Media Files
────────────                                 ───────────
Created by developers                        Uploaded by users at runtime
CSS, JavaScript, site logos, SVG icons       Avatars, PDFs, invoice receipts, photos
Committed to Git                             Never committed to Git
Stored locally in app /static/ folders       Stored in AWS S3, GCS, or object storage
Managed via STATIC_URL, STATIC_ROOT          Managed via MEDIA_URL, MEDIA_ROOT
```

---

### What does `collectstatic` do?

In local development, Django's dev server automatically finds static files inside each installed app's `static/` directory.

In production, having Nginx search through 30 different Python package folders on every CSS request would be horribly slow.

During production build / deployment, we run:

```bash
python manage.py collectstatic --no-input
```

`collectstatic` traverses all `INSTALLED_APPS` and copies every CSS, JS, and image asset into a single unified directory: `STATIC_ROOT`.

Nginx is then pointed directly to `STATIC_ROOT` to serve all assets with maximum performance:

```nginx
location /static/ {
    alias /var/www/myproject/staticfiles/;
    expires 30d;
    add_header Cache-Control "public, max-age=2592000, immutable";
}
```

---

# 3. Writing Custom Middleware

## Interview Question

**How do you write a custom Django middleware to attach a correlation ID (request ID) to every incoming request and outgoing response?**

---

### Answer

Modern Django middleware is structured as a callable class initialized once when the web server starts up:

```python
# core/middleware.py
import uuid
import time
import logging

logger = logging.getLogger(__name__)

class RequestTracingMiddleware:
    def __init__(self, get_response):
        # One-time configuration and initialization
        self.get_response = get_response

    def __call__(self, request):
        # 1. PRE-PROCESSING (Executes on incoming request)
        correlation_id = request.headers.get('X-Correlation-ID', str(uuid.uuid4()))
        request.correlation_id = correlation_id
        start_time = time.perf_counter()

        # 2. HANDOFF (Calls the next middleware or view)
        response = self.get_response(request)

        # 3. POST-PROCESSING (Executes on outgoing response)
        duration = time.perf_counter() - start_time
        response['X-Correlation-ID'] = correlation_id
        response['X-Response-Time'] = f"{duration:.4f}s"

        logger.info(
            f"Path={request.path} Method={request.method} "
            f"Status={response.status_code} Duration={duration:.4f}s ID={correlation_id}"
        )

        return response
```

Register it in `settings.py`:

```python
MIDDLEWARE = [
    'core.middleware.RequestTracingMiddleware',  # At or near the top
    'django.middleware.security.SecurityMiddleware',
    ...
]
```

---

# 4. Asynchronous Background Tasks with Celery

## Interview Question

**Why should you never send emails or call third-party APIs inside a Django view, and how do you coordinate Celery tasks with database transactions?**

---

### Answer

A standard WSGI worker (like Gunicorn) handles requests synchronously.

Suppose your checkout view sends a confirmation email using SendGrid:

```python
# ANTI-PATTERN: Blocking the HTTP worker
def checkout_view(request):
    order = create_order(request.user)
    send_confirmation_email(order)  # Takes 2.5 seconds to talk to SMTP!
    return JsonResponse({'status': 'ok'})
```

During those 2.5 seconds, that Gunicorn worker is completely blocked. It cannot accept any other incoming requests.

If 10 customers check out at the same time and you have 8 Gunicorn workers, your entire website freezes for all other visitors.

### The Fix: Background Tasks with Celery

Offload time-consuming work to a background worker queue:

```
Django App (Producer)
       │
       │ Enqueues message: {"task": "send_email", "order_id": 42}
       ▼
Message Broker (Redis / RabbitMQ)
       │
       ▼
Celery Worker Process (Consumer)
       │
       └── Talks to SMTP server in the background (User doesn't wait!)
```

---

### The Critical Production Trap: The `on_commit` Pattern

This is a favorite senior interview question.

Consider this buggy code:

```python
from django.db import transaction
from .tasks import process_order_task

def checkout(request):
    with transaction.atomic():
        order = Order.objects.create(user=request.user, total=100)
        # BUG: Enqueueing task INSIDE the transaction block!
        process_order_task.delay(order.id)
        
    return JsonResponse({'status': 'ok'})
```

### Why this crashes in production:

Celery workers run in separate processes and consume tasks almost instantaneously (often in under 2ms).

If the Celery worker picks up `process_order_task` before Django's database transaction has finished committing to PostgreSQL, the worker executes:

```python
order = Order.objects.get(id=order_id)
```

PostgreSQL's transaction isolation hides uncommitted rows. The worker gets `Order.DoesNotExist` and fails!

### The Solution: `transaction.on_commit()`

Use `transaction.on_commit()` to guarantee that the Celery task is dispatched **only after the database transaction has successfully committed**:

```python
def checkout(request):
    with transaction.atomic():
        order = Order.objects.create(user=request.user, total=100)
        
        # Safe: Fires ONLY after the DB transaction has committed!
        transaction.on_commit(lambda: process_order_task.delay(order.id))
        
    return JsonResponse({'status': 'ok'})
```

---

# 5. Django Admin Customization & Operations

For an in-depth guide on optimizing, customizing, and securing the Django Admin (including batch actions, colored status badges, inlines, custom filters, and performance tuning), see the dedicated chapter:

👉 **[Django Admin Customization, Performance & Security](admin-customization.md)**

