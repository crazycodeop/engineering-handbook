# DRF API Features: Pagination, Filtering & Throttling

> This guide explores the production features of Django REST Framework: pagination strategies, search and filter backends, rate limiting (throttling), API versioning, and custom exception handling.

---

# 1. Pagination Strategies & Performance

## Interview Question

**Compare `PageNumberPagination` and `CursorPagination`. Why does page-number pagination become dangerously slow on large datasets, and how does cursor pagination solve it?**

---

### Answer

Pagination ensures endpoints don't return 500,000 rows in a single response, which would exhaust server memory and crash client apps.

DRF offers three primary pagination styles:

1. **`PageNumberPagination`** (`?page=2&page_size=20`)
2. **`LimitOffsetPagination`** (`?limit=20&offset=40`)
3. **`CursorPagination`** (`?cursor=cD0yMDI2...`)

---

### The Hidden Flaw with `PageNumberPagination` (Offset-based)

Under the hood, `PageNumberPagination` translates into SQL `OFFSET` and `LIMIT`:

```sql
-- Client requests page 5,000 with page_size = 20
SELECT * FROM orders 
ORDER BY created_at DESC 
LIMIT 20 OFFSET 100000;
```

### Why this is slow in production:

The database cannot magically jump to row 100,000. It must scan and discard the first 100,000 index entries to find the 20 requested rows. As the page number grows, the query gets slower and slower.

Furthermore, offset pagination suffers from **Page Drift**:
If a user is viewing Page 1, and 5 new orders are inserted into the database, navigating to Page 2 causes the user to see the last 5 orders from Page 1 a second time!

---

### The Solution: `CursorPagination` (Keyset Pagination)

`CursorPagination` does not use `OFFSET`. Instead, it uses an opaque, encrypted cursor that points to the exact timestamp or ID of the last item on the previous page:

```python
from rest_framework.pagination import CursorPagination

class OrderCursorPagination(CursorPagination):
    page_size = 20
    ordering = '-created_at'  # Must be an indexed, immutable field!
```

Django generates SQL using a `WHERE` clause:

```sql
-- Next page query
SELECT * FROM orders 
WHERE created_at < '2026-11-10 09:15:00' 
ORDER BY created_at DESC 
LIMIT 20;
```

### Why Cursor Pagination is superior:
1. **Constant $O(1)$ Time**: The database uses an index seek to jump directly to the target timestamp. Page 1,000 loads just as fast as Page 1.
2. **Zero Page Drift**: New records inserted at the top do not shift the cursor or cause duplicate records.

**The Trade-off**: Clients cannot jump directly to a random page (e.g. "Jump to page 47"). They can only navigate forward and backward (`next` / `previous`).

---

# 2. Filtering, Searching & Ordering

## Interview Question

**How do you configure declarative filtering, full-text searching, and sorting in DRF?**

---

### Answer

DRF uses pluggable filter backends configured globally or per-view:

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
        'rest_framework.filters.SearchFilter',
        'rest_framework.filters.OrderingFilter',
    ],
}
```

---

### 1. Exact & Range Filtering (`django-filter`)

```python
from django_filters.rest_framework import DjangoFilterBackend
from rest_framework.viewsets import ModelViewSet

class ProductViewSet(ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    filter_backends = [DjangoFilterBackend]
    filterset_fields = ['category', 'is_in_stock']
```

Clients can now query: `GET /products/?category=electronics&is_in_stock=true`

---

### 2. Search (`SearchFilter`)

```python
class ProductViewSet(ModelViewSet):
    ...
    filter_backends = [filters.SearchFilter]
    search_fields = ['name', 'description']
```

Clients query: `GET /products/?search=keyboard`

**Search Prefixes:**
- `'^name'`: Starts-with search (Uses `LIKE 'term%'`, fast if indexed).
- `'=name'`: Exact match.
- `'@name'`: Full-text search (PostgreSQL only).
- `'name'` (default): Case-insensitive contains (`ILIKE '%term%'`).

---

### 3. Ordering (`OrderingFilter`)

```python
class ProductViewSet(ModelViewSet):
    ...
    filter_backends = [filters.OrderingFilter]
    ordering_fields = ['price', 'created_at']
    ordering = ['-created_at']  # Default sorting
```

Clients query: `GET /products/?ordering=-price`

---

# 3. Throttling & Rate Limiting

## Interview Question

**How does DRF's rate limiting work, and how would you implement tiered rate limits for Free vs Premium users?**

---

### Answer

Throttling protects APIs against denial-of-service (DoS) attacks, brute-force password guessing, and API abuse.

DRF tracks request timestamps against client IPs (for anonymous users) or user IDs (for authenticated users), typically using Redis cache.

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/day',
        'user': '1000/hour',
        'premium': '10000/hour',
    }
}
```

When a user exceeds their limit, DRF returns **HTTP 429 Too Many Requests** along with a `Retry-After: 3600` header.

---

### Custom Tiered Throttling (Free vs Premium)

```python
from rest_framework.throttling import UserRateThrottle

class TieredRateThrottle(UserRateThrottle):
    def get_rate(self):
        # Determine the rate dynamically based on the user's subscription
        if self.request.user.is_authenticated and self.request.user.is_premium:
            return self.THROTTLE_RATES['premium']
        return self.THROTTLE_RATES['user']
```

Attach it to the view:

```python
class DataExportViewSet(ModelViewSet):
    throttle_classes = [TieredRateThrottle]
    ...
```

---

# 4. Custom Exception Handling

## Interview Question

**How do you write a custom exception handler in DRF to standardize API error formats across an entire organization?**

---

### Answer

By default, DRF error responses can be inconsistent. A validation error returns a dictionary of field errors:

```json
{"email": ["Enter a valid email address."]}
```

While an unauthenticated error returns:

```json
{"detail": "Authentication credentials were not provided."}
```

Frontend developers prefer a unified JSON structure for all errors.

You can intercept all exceptions using a **Custom Exception Handler**:

```python
# core/exceptions.py
from rest_framework.views import exception_handler
from rest_framework.response import Response
from rest_framework import status

def custom_exception_handler(exc, context):
    # 1. Call DRF's default exception handler first to get the standard response
    response = exception_handler(exc, context)

    # 2. If DRF handled it (400, 401, 403, 404, 429)
    if response is not None:
        custom_data = {
            "success": False,
            "error": {
                "status_code": response.status_code,
                "message": getattr(exc, 'default_detail', str(exc)),
                "details": response.data,
            }
        }
        response.data = custom_data

    # 3. If an unhandled Python exception occurred (500 Internal Server Error)
    else:
        custom_data = {
            "success": False,
            "error": {
                "status_code": 500,
                "message": "An unexpected server error occurred. Please try again later.",
                "details": None,
            }
        }
        response = Response(custom_data, status=status.HTTP_500_INTERNAL_SERVER_ERROR)

    return response
```

Register it in `settings.py`:

```python
REST_FRAMEWORK = {
    'EXCEPTION_HANDLER': 'core.exceptions.custom_exception_handler',
}
```

Now, every error across your entire platform returns a consistent, predictable contract.
