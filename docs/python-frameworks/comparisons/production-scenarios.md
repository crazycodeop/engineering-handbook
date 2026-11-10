# Production Debugging & System Design Scenarios

> This guide explores 13 real-world production engineering incidents and scenario-based interview questions, breaking down the exact investigative steps, root causes, and production solutions.

---

# Scenario 1: The 5-Second API Endpoint

### Interviewer
> *"A critical API endpoint in your Django/DRF service was running in 40ms last month. Today, response times spiked to 5.2 seconds in production. How do you investigate, diagnose, and fix the issue?"*

### Simple Explanation
When an endpoint suddenly degrades from milliseconds to seconds, engineers should not start guessing or changing random lines of code. You need an observational methodology to divide the request lifecycle into measurable segments.

### How to Investigate Step by Step

```
Step 1: Check APM & Logs (Datadog / New Relic / Sentry)
        │
        ├── Is the bottleneck in Python CPU, Database, or Network I/O?
        │
        ├── CASE A: Database (90% of incidents)
        │     ├── Check query count: Is it executing 1 query or 800 queries? (N+1 query)
        │     └── Check query execution time: Is a single query missing an index? (Table Scan)
        │
        ├── CASE B: External Third-Party Call
        │     └── Is someone calling an unmemoized external payment or SMS API synchronously?
        │
        └── CASE C: Serialization Overhead
              └── Is DRF serializing 10,000 objects with nested SerializerMethodFields?
```

### The Common Production Culprit
Usually, a table grew from 500 rows to 500,000 rows. A query that used to perform a fast full-table scan in local development now locks up PostgreSQL because a column in the `WHERE` clause (like `status` or `created_at`) lacks an index.

### The Solution:
1. Run `EXPLAIN ANALYZE` on PostgreSQL to inspect the query plan.
2. Add a targeted database index (`db_index=True` or `Index(fields=['tenant', 'status'])`).
3. Add `select_related` or `prefetch_related` if an N+1 query appeared.
4. Verify using Django Silk or Django Debug Toolbar.

---

# Scenario 2: Identifying & Permanently Fixing N+1 Queries

### Interviewer
> *"How do you guarantee that a junior developer's code change doesn't introduce an N+1 query to production in the future?"*

### Solution: Automated Query Count Assertions

Instead of relying on manual code reviews, write automated unit tests that enforce query limits using Django's `assertNumQueries`:

```python
# tests/test_perf.py
from django.test import TestCase
from books.models import Author, Book

class PerformanceTests(TestCase):
    def test_book_list_query_count(self):
        # Create 50 books with authors
        author = Author.objects.create(name="Tolkien")
        for i in range(50):
            Book.objects.create(title=f"Book {i}", author=author)

        # Force authentication and call API
        with self.assertNumQueries(2):
            # Query 1: SELECT session/user
            # Query 2: SELECT books JOIN authors (select_related)
            response = self.client.get('/api/v1/books/')
            self.assertEqual(response.status_code, 200)
            self.assertEqual(len(response.data), 50)
```

If a developer later adds `book.author.profile.avatar` without updating `select_related`, the test immediately fails in the CI/CD pipeline before reaching production!

---

# Scenario 3: Database Connection Leaks & Exhaustion

### Interviewer
> *"At 2:00 AM, your application logs flood with `psycopg2.OperationalError: FATAL: remaining connection slots are reserved for non-replication superuser connections`. What happened, and how do you prevent it?"*

### Root Cause
PostgreSQL has a hard limit on open connections (default: 100).

This error occurs when application workers open database connections faster than they release them, or fail to close them during crashes:

```
Common Causes:
1. Unclosed raw database cursors (missing `with connection.cursor()` context manager).
2. Spawning hundreds of unmanaged background threads that each open a database connection.
3. Setting `CONN_MAX_AGE = None` in Django while running 20 autoscaled Gunicorn servers (20 servers × 8 workers = 160 connections > 100 max).
```

### The Production Fix:
1. **Deploy PgBouncer**: Place PgBouncer between your web servers and PostgreSQL using **transaction-level pooling**. PgBouncer multiplexes 500 application workers over a small, dedicated pool of 20 physical database connections.
2. Set `CONN_MAX_AGE = 0` or a low value (e.g. 10 seconds) when running behind PgBouncer.
3. Ensure all SQLAlchemy sessions use `try ... finally: session.close()`.

---

# Scenario 4: Handling 10,000 Concurrent Requests in FastAPI

### Interviewer
> *"How would you architect a FastAPI service to handle 10,000 concurrent incoming requests without running out of RAM or overloading your database?"*

### The Architectural Blueprint

```
10,000 Concurrent Clients
           │
           ▼
[ Cloudflare / CDN ] ──► Serves static assets & caches public GET endpoints
           │
           ▼
[ Nginx Reverse Proxy ] ──► Buffers slow client uploads; rate-limits abuse
           │
           ▼
[ Gunicorn + Uvicorn Workers ] (e.g. 8 async worker processes on event loops)
           │
           ├── Redis Cache ──► Hot data served in 0.5ms (90% of read traffic)
           │
           ▼
[ PgBouncer Connection Pooler ] (Caps active DB connections to 30)
           │
           ▼
[ PostgreSQL Database ]
```

### Key Optimizations:
1. **Never open 10,000 DB connections**: Configure SQLAlchemy with `pool_size=20, max_overflow=10` and use PgBouncer.
2. **Cache Hot Paths**: Use Redis for frequently queried data. If 8,000 requests are querying the homepage catalog, serve it from Redis in sub-milliseconds.
3. **Offload Write Heavy Tasks**: For requests that mutate non-critical state (e.g. tracking clicks, logging audits), push tasks to a Celery/Redis queue and return an instant `202 Accepted`.

---

# Scenario 5: Tracing a Frozen Event Loop in FastAPI

### Interviewer
> *"A developer deployed a FastAPI route that caused the entire server to freeze intermittently. All other endpoints stopped responding. How do you find the offending code?"*

### Diagnosis
Someone put **blocking synchronous code inside an `async def` route**.

Because it was an `async def` function, Uvicorn executed it directly on the main event loop thread instead of the worker thread pool.

### How to Find It:
Enable Python's **`asyncio` debug mode**:

```bash
PYTHONASYNCIODEBUG=1 uvicorn main:app
```

Or configure it in code:

```python
import asyncio
loop = asyncio.get_event_loop()
loop.set_debug(True)
# Logs a warning if any coroutine blocks the event loop for longer than 100ms!
loop.slow_callback_duration = 0.1
```

When the slow endpoint executes, Python prints a clear traceback to the console:

```
Executing <Task ...> took 2.450 seconds!
File "routes/users.py", line 42, in get_stats
    response = requests.get(...)  <-- THE CULPRIT!
```

---

# Scenario 6: Structuring a Large Flask Application for a Team of 25

### Interviewer
> *"How would you structure a Flask application across 25 engineers to avoid merge conflicts and circular imports?"*

### Solution: Application Factory + Service-Oriented Blueprints

```
ecommerce/
├── __init__.py           → Extension initialization (db, migrate, csrf, login_manager)
├── app.py                → create_app() factory
├── config.py             → Environment-specific settings
│
├── modules/
│   ├── auth/             → Auth Blueprint, forms, services, models
│   ├── billing/          → Billing Blueprint, Stripe services, models
│   └── inventory/        → Inventory Blueprint, stock services, models
│
└── services/             → Shared domain business logic (Decoupled from HTTP!)
```

### Rules:
1. **Zero global `app`**: Everything is instantiated inside `create_app()`.
2. **Service Layer**: Views only parse JSON and pass data to pure Python service functions (`billing_service.charge_card()`). Business logic never lives inside route functions.
3. **Explicit Blueprint registration**: Each module exports a self-contained Blueprint.

---

# Scenario 7: Implementing Role-Based Access Control (RBAC)

### Interviewer
> *"How do you design a flexible, scalable RBAC system for an API where users have multiple roles (Admin, Manager, Member) and roles have granular permissions?"*

### Architecture: The Three-Tier Model

```
User  ──────►  UserRoles  ──────►  Role  ──────►  RolePermissions  ──────►  Permission
Alice          (Assigned)          Manager        (Has)                     "orders:refund"
```

1. **Never hardcode role checks in views**: Avoid `if user.role == "Admin"`. Six months later, product introduces "SuperAdmin" or "FinanceManager", and you have to update 50 files.
2. **Check Permissions, not Roles**: Check `if user.has_perm("orders:refund")`.
3. **FastAPI Dependency Enforcement**:
   ```python
   def require_permission(perm: str):
       def dependency(user: User = Depends(get_current_user)):
           if not user.has_permission(perm):
               raise HTTPException(status_code=403, detail="Forbidden")
           return user
       return dependency

   @app.post("/refund")
   def refund(user: User = Depends(require_permission("orders:refund"))):
       ...
   ```

---

# Scenario 8: Preventing Duplicate Requests (Idempotency)

### Interviewer
> *"A user clicks 'Pay Now' on a checkout page. Their phone network blips, so their client retries the request 3 seconds later. How do you prevent the customer from being charged twice?"*

### Solution: Idempotency Keys with Redis & DB Constraints

```
Client sends: POST /api/payments/ (Header: Idempotency-Key: "uuid-1234-abcd")
                      │
                      ▼
[ Idempotency Middleware / Dependency ]
                      │
                      ├── 1. Check Redis: Has key "idempotency:uuid-1234-abcd" been seen?
                      │     │
                      │     ├── YES (Completed): Return cached previous response immediately!
                      │     ├── YES (In Progress): Return HTTP 409 Conflict ("Transaction in progress")
                      │     └── NO: Acquire Redis lock with 60s TTL
                      │
                      ▼
[ Execute Payment via Stripe & Save to PostgreSQL ]
                      │
                      ▼
[ Store response in Redis under "idempotency:uuid-1234-abcd" ]
                      │
                      ▼
Return HTTP 200 OK
```

### Database Safety Net:
Add a `unique=True` constraint on the `idempotency_key` column in the `Payment` table. Even if Redis crashes, the database engine physically rejects the duplicate insert!

---

# Scenario 9: Handling Slow External APIs Without Blocking

### Interviewer
> *"Your API needs to call a third-party credit check service that takes between 2 to 10 seconds to respond. How do you prevent this from freezing your Python application?"*

### Solution:

1. **Set Strict Timeouts**: Never make an HTTP call without timeouts:
   ```python
   # Times out after 3 seconds rather than hanging forever
   httpx.get("https://external-credit.com", timeout=3.0)
   ```
2. **Use Async I/O (FastAPI)**: By using `await client.get()`, the Python worker yields control to the event loop, allowing thousands of other users to use the server while waiting for the credit check.
3. **Asynchronous Webhook / Celery Workflow**: For operations taking over 5 seconds:
   - Accept request: `POST /credit-check`
   - Enqueue Celery task and return `202 Accepted` with a `task_id`.
   - The frontend polls `GET /credit-check/{task_id}` or receives a WebSocket event when the job completes.

---

# Scenario 10: Investigating Memory Leaks in Long-Running Workers

### Interviewer
> *"You notice that Gunicorn worker processes start with 150MB of RAM, but after 24 hours of production traffic, each worker consumes 1.8GB of RAM until the server crashes. How do you debug and fix this?"*

### Common Root Causes:
1. **Django `DEBUG = True` left in production**:
   When `DEBUG = True`, Django automatically saves every single SQL query executed into a Python list: `connection.queries`. In a busy application, this list grows indefinitely in memory!
2. **Global Lists / Dictionaries**:
   Appending objects or cached results to a module-level list that is never cleared.
3. **Evaluating Massive QuerySets**:
   Running `list(User.objects.all())` on 1,000,000 rows populates `_result_cache`.

### How to Fix:
1. Set `DEBUG = False`.
2. Use `.iterator()` when processing large batches of records.
3. Use Gunicorn's `--max-requests 1000` and `--max-requests-jitter 100`. This instructs Gunicorn to gracefully restart each worker process after it has served 1,000 requests, automatically releasing all accumulated memory back to the operating system.

---

# Scenario 11: Preventing Lost Updates (Concurrent Mutations)

### Interviewer
> *"Two administrators open the same product inventory page at the same time. Both see stock = 10. Admin A changes stock to 15. Admin B changes stock to 8. Admin B clicks Save second. How do you prevent Admin B from silently overwriting Admin A's change?"*

### Approach 1: Pessimistic Locking (`select_for_update`)
Lock the row in the database during the edit transaction. Works well for fast programmatic operations (like balance transfers), but bad for human UI workflows (you cannot hold a database lock while a human types on a form).

### Approach 2: Optimistic Locking with Version Numbers (Industry Standard)
Add a `version = IntegerField(default=1)` column to the model:

```python
# When Admin A reads the product:
# product.id = 1, product.stock = 10, product.version = 1

# When Admin A saves:
rows_updated = Product.objects.filter(
    id=1, 
    version=1  # Only update if version has NOT changed!
).update(
    stock=15, 
    version=F('version') + 1
)

if rows_updated == 0:
    raise ConflictError("The product was modified by another user while you were editing. Please refresh.")
```

When Admin B attempts to save with `version=1`, `rows_updated` is `0` because Admin A already incremented `version` to `2`. Admin B is cleanly warned instead of silently overwriting data!

---

# Scenario 12: Deep Pagination on 50 Million Rows

### Interviewer
> *"Your `orders` table has 50 million records. Visiting `?page=1` takes 12ms. Visiting `?page=500000` takes 18 seconds or times out. Why does this happen, and how do you redesign the API?"*

### The Flaw with Offset Pagination
`LIMIT 20 OFFSET 10000000` requires PostgreSQL to scan 10,000,020 rows on disk and discard the first 10,000,000.

### The Fix: Keyset / Cursor Pagination
Replace `page` with a cursor based on an indexed column (e.g. `id` or `created_at`):

```sql
-- Client sends: ?cursor=10000020
SELECT * FROM orders 
WHERE id > 10000020 
ORDER BY id ASC 
LIMIT 20;
```

PostgreSQL performs a B-Tree index seek directly to `id = 10000020` in **0.1ms**.

Query execution time remains constant whether querying page 1 or page 500,000!

---

# Scenario 13: Local Works, Production Fails Intermittently

### Interviewer
> *"A developer submits a feature that passes all unit tests and works flawlessly on their local machine with `sqlite3` and `runserver`. In production (Docker, Nginx, Gunicorn, PostgreSQL), 5% of requests randomly fail with 502 Bad Gateway. What are the common causes?"*

### Common Culprits:
1. **Different Database Engines**: SQLite is forgiving of type coercion, missing foreign key constraints, and case sensitivity. PostgreSQL enforces strict type checking and case sensitivity. (Always use Docker with PostgreSQL for local dev!).
2. **Reverse Proxy Timeouts**: Gunicorn's default timeout is 30 seconds. If an unindexed query in production takes 31 seconds, Gunicorn abruptly kills the worker process, and Nginx returns `502 Bad Gateway`.
3. **Database Connection Limits**: SQLite allows infinite local in-process connections. In production, 8 Gunicorn workers quickly hit PostgreSQL's `max_connections` limit under load.
4. **Missing Static Files**: `DEBUG=False` in production causes Django to stop serving static files. If `collectstatic` was not run or Nginx is not configured to serve `/static/`, the frontend assets fail to load.
