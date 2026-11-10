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

# 5. Django Admin Optimization in Production

## Interview Question

**Why does the Django Admin frequently crash or become extremely slow on large database tables, and how do you optimize it?**

---

### Answer

The Django Admin is a great productivity tool, but out of the box, it can easily bring down a production database on tables with hundreds of thousands of rows.

### Mistake 1: Foreign Key Dropdowns

By default, for any `ForeignKey` field, the Django admin renders an HTML `<select>` dropdown menu containing **every single record in the foreign table**.

If your `Order` model has a ForeignKey to `User`, and you have 500,000 users, loading the "Add Order" admin page loads all 500,000 user rows into memory to build a massive HTML dropdown! The browser freezes and the server runs out of RAM.

**The Fix: `raw_id_fields` or `autocomplete_fields`**

```python
@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    # Replaces dropdown with a lightweight search popup or AJAX input
    raw_id_fields = ('user',)
    # OR
    autocomplete_fields = ('user',)
```

---

### Mistake 2: N+1 Queries in `list_display`

If your admin list displays related fields:

```python
@admin.register(Book)
class BookAdmin(admin.ModelAdmin):
    list_display = ('title', 'get_author_name')

    def get_author_name(self, obj):
        return obj.author.name  # N+1 query for every row in the admin list!
```

If the admin page displays 100 books, it executes 101 queries!

**The Fix: Override `get_queryset()`**

```python
@admin.register(Book)
class BookAdmin(admin.ModelAdmin):
    list_display = ('title', 'get_author_name')

    def get_queryset(self, request):
        # Eager load the author relationship in 1 query
        return super().get_queryset(request).select_related('author')

    def get_author_name(self, obj):
        return obj.author.name
```

---

# 6. Extending & Customizing the Django Admin

## Interview Question

**How do you customize the Django Admin for production workflows? How do you implement custom actions, inlines, colored status badges, and custom list filters?**

---

### Answer

The Django Admin is not just for superusers; back-office operations, customer support, and finance teams rely on it every day.

Out of the box, the default admin is basic. In production, we customize it extensively.

---

### 1. Custom Batch Actions (e.g. Export to CSV)

Suppose operations staff need to select multiple orders and mark them as shipped in bulk:

```python
from django.contrib import admin
from django.utils.translation import ngettext
from django.contrib import messages

@admin.action(description="Mark selected orders as Shipped")
def mark_as_shipped(modeladmin, request, queryset):
    # Atomic database bulk update
    updated_count = queryset.filter(status='pending').update(status='shipped')

    # Feedback banner for admin user
    modeladmin.message_user(
        request,
        ngettext(
            "%d order was successfully marked as shipped.",
            "%d orders were successfully marked as shipped.",
            updated_count,
        ) % updated_count,
        messages.SUCCESS,
    )

@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_display = ('id', 'user', 'status', 'total_amount')
    actions = [mark_as_shipped]
```

---

### 2. Inline Formsets: Managing Children on Parent Pages

If every `Order` has multiple `OrderItem` rows, navigating to a separate page to edit each item is frustrating.

Use `TabularInline` (compact table layout) or `StackedInline` (vertical form layout):

```python
class OrderItemInline(admin.TabularInline):
    model = OrderItem
    extra = 0          # Prevents showing 3 empty blank rows by default
    min_num = 1        # Enforces at least 1 item
    raw_id_fields = ('product',)

@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_display = ('id', 'user', 'status', 'created_at')
    inlines = [OrderItemInline]
```

Now, when opening an order, all related items are rendered and editable directly on that same page.

---

### 3. Colored Status Badges with `format_html`

Instead of plain text status words, color-coded badges help support staff quickly spot problem orders:

```python
from django.utils.html import format_html

@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_display = ('id', 'user', 'status_badge', 'total_amount')

    @admin.display(description="Status", ordering="status")
    def status_badge(self, obj):
        colors = {
            'completed': '#28a745',  # Green
            'pending': '#ffc107',    # Yellow
            'failed': '#dc3545',     # Red
            'refunded': '#6c757d',   # Grey
        }
        color = colors.get(obj.status, '#000')
        return format_html(
            '<span style="background-color: {}; color: white; padding: 3px 8px; border-radius: 4px; font-weight: bold;">{}</span>',
            color,
            obj.status.upper()
        )
```

---

### 4. Custom Filters with `SimpleListFilter`

When built-in `list_filter = ('status',)` is not enough (e.g. you want business-logic filters like "High Value Orders > \$1,000"):

```python
class HighValueFilter(admin.SimpleListFilter):
    title = 'order value tier'  # Title shown on the filter sidebar
    parameter_name = 'value_tier'

    def lookups(self, request, model_admin):
        return (
            ('high', 'High Value (> $1,000)'),
            ('standard', 'Standard (≤ $1,000)'),
        )

    def queryset(self, request, queryset):
        if self.value() == 'high':
            return queryset.filter(total_amount__gt=1000)
        if self.value() == 'standard':
            return queryset.filter(total_amount__lte=1000)
        return queryset

@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_filter = (HighValueFilter, 'status', 'created_at')
```

---

### 5. Custom Fieldsets & Readonly Locks

Organize cluttered forms into clean, collapsible sections, and prevent editing critical audit fields:

```python
@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    readonly_fields = ('created_at', 'updated_at', 'stripe_charge_id')

    fieldsets = (
        ("Customer Information", {
            'fields': ('user', 'shipping_address')
        }),
        ("Financial Details", {
            'fields': ('total_amount', 'stripe_charge_id', 'status')
        }),
        ("Timestamps & Audit", {
            'classes': ('collapse',),  # Collapsible drawer in the UI!
            'fields': ('created_at', 'updated_at')
        }),
    )

    # Disable deletion of already completed orders
    def has_delete_permission(self, request, obj=None):
        if obj and obj.status == 'completed':
            return False  # Protects financial records from deletion!
        return super().has_delete_permission(request, obj)
```

---

### 6. Securing the Admin in Production

1. **Change the URL**: Automated botnets constantly scan `/admin/` with brute-force dictionaries. Change the path in `urls.py` to something unexpected:
   ```python
   # urls.py
   urlpatterns = [
       path('internal-portal-992/', admin.site.urls),
   ]
   ```
2. **Two-Factor Authentication (2FA)**: Integrate `django-otp` or `django-two-factor-auth` so admin logins require TOTP authenticator codes.
3. **Admin Honeypot**: Point `/admin/` to `django-admin-honeypot`, which logs attacker IP addresses and alerts your security team when a bot attempts a login!
