# Framework & Architecture Comparisons

> This guide provides side-by-side architectural comparisons of Python backend frameworks, protocols, ORMs, and design patterns commonly evaluated in senior engineering interviews.

---

# 1. Django vs Flask vs FastAPI

## Interview Question

**If you were architecting a new Python backend today, how would you decide between Django, Flask, and FastAPI?**

---

### Comparison Matrix

| Criteria | Django | Flask | FastAPI |
|---|---|---|---|
| **Architectural Philosophy**| Monolithic & "Batteries-Included" | Minimalist Microframework | Type-Driven API Microframework |
| **Out-of-the-Box Components**| ORM, Admin, Auth, Migrations, Forms, Security | Web router & Jinja2 templates only | Web router, Pydantic validation, OpenAPI docs |
| **Protocol & Concurrency** | WSGI (with partial ASGI) | WSGI (Synchronous) | **Native ASGI (Asynchronous event loop)** |
| **Data Validation** | Django Forms / DRF Serializers | External (Marshmallow / Pydantic) | **Built-in Pydantic v2 (Rust core)** |
| **API Documentation** | Requires third-party plugin (`drf-spectacular`) | Requires third-party plugin (`flasgger`) | **Automatic Swagger UI (`/docs`) & ReDoc** |
| **Database Layer** | Opinionated Django ORM | Flexible (SQLAlchemy, Peewee, Mongo) | Flexible (Async SQLAlchemy, Tortoise, Mongo) |
| **Development Speed** | Fastest for full-stack monoliths & CRUD | Fast for small prototypes | Fastest for typed REST & microservices |

---

### How to Answer in an Interview: Decision Framework

When an interviewer asks: *"Which one is better?"*, never give a one-word answer. Explain the trade-offs:

1. **Choose Django when**:
   - You need a full-featured web application with a relational database, user authentication, and back-office administrative portals.
   - You have a large or growing team where standard conventions (`models.py`, `views.py`, `urls.py`) accelerate onboarding and eliminate bike-shedding.
   - You are building e-commerce, content management systems, internal business tooling, or SaaS platforms.

2. **Choose FastAPI when**:
   - You are building high-performance REST APIs or microservices consumed by mobile apps or modern Single Page Applications (React, Vue, Next.js).
   - Your system is I/O-intensive and requires thousands of concurrent open connections (WebSockets, streaming data, microservice orchestrations).
   - You are serving Machine Learning or AI models where type validation, JSON Schema, and async throughput are paramount.

3. **Choose Flask when**:
   - You need an ultra-lightweight microservice with minimal dependencies (e.g. AWS Lambda serverless functions).
   - You have legacy synchronous codebases or non-relational data stores where Django's opinionated ORM is a hindrance.
   - You want total architectural autonomy to assemble your own bespoke stack.

---

# 2. WSGI vs ASGI

## Interview Question

**What is the fundamental difference between WSGI and ASGI? Why did the Python ecosystem need a new standard?**

---

### Answer

```
WSGI (Web Server Gateway Interface - PEP 3333)
─────────────────────────────────────────────
• Synchronous request-response model.
• 1 incoming request = 1 OS worker thread/process occupied until completion.
• Cannot natively handle WebSockets, Server-Sent Events, or async coroutines.
• Typical Servers: Gunicorn, uWSGI.
```

```
ASGI (Asynchronous Server Gateway Interface)
────────────────────────────────────────────
• Asynchronous event-driven model.
• 1 worker process = 1 Event Loop handling thousands of concurrent requests.
• Native support for HTTP, WebSockets, long polling, and background tasks.
• Typical Servers: Uvicorn, Hypercorn, Daphne.
```

---

### Why WSGI struggled with modern web requirements:

Imagine an endpoint that streams live stock tickers or handles a chat room using WebSockets.

A WebSocket connection stays open for minutes or hours.

Under **WSGI**, if 50 users connect to a WebSocket chat room, they occupy 50 Gunicorn worker processes. Your entire server runs out of workers and rejects all incoming HTTP traffic!

Under **ASGI**, connections are managed on an event loop. A single Uvicorn process can maintain 20,000 idle WebSocket connections while using less than 100MB of RAM.

---

# 3. Django ORM vs SQLAlchemy 2.0

## Interview Question

**Compare the Active Record pattern used by Django ORM with the Data Mapper pattern used by SQLAlchemy. What are the production trade-offs?**

---

### Answer

```
Active Record (Django ORM)                Data Mapper (SQLAlchemy)
──────────────────────────                ────────────────────────
Model class represents table AND query.   Model class is purely a data schema.
user = User.objects.get(id=1)             Query logic is separated into Session.
user.name = "Bob"                         session.execute(select(User).where(...))
user.save()  # Instance saves itself!     session.commit()  # Session manages persistence!
```

---

### Comparison Matrix

| Feature | Django ORM (Active Record) | SQLAlchemy 2.0 (Data Mapper) |
|---|---|---|
| **Design Pattern** | Active Record | Data Mapper / Unit of Work |
| **Coupling** | High: Model instances are directly bound to the database | Low: Models are pure Python classes; persistence is handled by `Session` |
| **Learning Curve** | Extremely low (Intuitive syntax) | Moderate (Requires understanding sessions and engines) |
| **Query Complexity** | Excellent for 90% of CRUD; clumsy for window functions and CTEs | Industry gold-standard; can express virtually any complex SQL query |
| **Async Support** | Partial (Async wrappers over sync DB calls) | **First-class native async** (`create_async_engine`) |
| **Transactions** | Auto-commit by default; `transaction.atomic()` | Explicit transaction lifecycle managed by `Session` |

---

# 4. DRF Serializers vs Pydantic Models

## Interview Question

**How does data validation and serialization in DRF differ from Pydantic in FastAPI?**

---

### Answer

```
DRF Serializer                                Pydantic BaseModel (FastAPI)
──────────────                                ────────────────────────────
• Deeply coupled to Django's ORM.             • Standalone library (Zero framework coupling).
• Imperative validation:                      • Declarative validation:
  validate_<field>(), validate()                Type annotations (int, str, EmailStr).
• Slower: Written in pure Python.             • Lightning fast: Core engine written in Rust.
• Automatically inspects Django models.       • Validates any arbitrary dictionary or object.
• Generates DRF validation error dicts.       • Generates standard JSON Schema.
```

### The Key Difference:
In DRF, a `ModelSerializer` is explicitly built to translate between Django models and JSON.

In FastAPI, Pydantic has zero awareness of databases. It validates raw incoming JSON and produces typed Python objects. Connecting Pydantic to a database is done explicitly via SQLAlchemy or SQLModel.

---

# 5. Django Signals vs Explicit Service Layer

## Interview Question

**Why do experienced backend engineers often reject Django Signals in favor of an explicit Service Layer?**

---

### Answer

```python
# APPROACH 1: Django Signal (Implicit & Hidden)
@receiver(post_save, sender=Order)
def on_order_created(sender, instance, created, **kwargs):
    if created:
        send_order_email(instance)
        update_inventory(instance)
        track_analytics(instance)
```

```python
# APPROACH 2: Service Layer (Explicit & Transparent)
# services/order_service.py
def create_order(user, cart_items):
    with transaction.atomic():
        order = Order.objects.create(user=user)
        deduct_inventory(order, cart_items)
        transaction.on_commit(lambda: send_order_email_task.delay(order.id))
        transaction.on_commit(lambda: track_analytics_task.delay(order.id))
    return order
```

### Why Senior Engineers Prefer the Service Layer:
1. **Traceability**: In the Service Layer, anyone reading `create_order()` sees the exact sequence of actions in plain Python. With signals, side effects happen magically in the background, making code reviews and debugging painful.
2. **Transaction Timing**: Signals fire *before* transactions commit, causing race conditions with Celery. The Service Layer allows using `transaction.on_commit()` cleanly.
3. **Testability**: You can test the service function directly without mocking framework signal dispatchers.

---

# 6. Session-Based Authentication vs JWTs

## Interview Question

**When should you use server-side sessions vs stateless JSON Web Tokens (JWTs)?**

---

### Comparison Matrix

| Dimension | Server-Side Sessions (Django / Flask) | Stateless JWTs (FastAPI / DRF) |
|---|---|---|
| **Storage Location** | Server (PostgreSQL or Redis) | Client (Local storage, memory, or cookie) |
| **Token Size** | Tiny (32-character random string) | Large (300–800 bytes base64 payload) |
| **Revocation** | **Instant**: Delete row from Redis/DB | **Difficult**: Requires token blacklisting or expiry waiting |
| **Scalability** | Requires central session store (Redis) | **Stateless**: Validated mathematically without DB |
| **CSRF Risk** | Vulnerable if using browser cookies (Requires CSRF tokens) | Immune if stored in memory / custom Authorization header |
| **Best For** | Server-rendered websites, monolithic web apps | Mobile apps, distributed microservices, single-sign-on (SSO) |
