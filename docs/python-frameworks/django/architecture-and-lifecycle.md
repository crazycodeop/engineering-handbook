# Django Architecture & Request Lifecycle

> This handbook is not meant to teach Django from scratch.
>
> It's meant to teach you how experienced backend engineers think about Django during interviews.
>
> Every answer is written from the perspective of someone who has designed, built, deployed, debugged, and scaled Django applications in production.
>
> The focus is not on memorizing APIs.
>
> The focus is understanding why Django works the way it does, the trade-offs behind its design, and the kinds of production problems you'll eventually encounter.

---

# 1. Django Foundations

## Interview Question

**What is Django? Why would you choose Django over other frameworks like Flask or FastAPI?**

---

### Answer

I usually describe Django as a "batteries-included" backend framework.

Most frameworks give you a web router and let you assemble everything else yourself.

Django gives you an entire ecosystem.

Out of the box, you get:

- An Object-Relational Mapper (ORM)
- Built-in user authentication and session management
- An automatic admin dashboard
- A schema migration system
- A middleware pipeline
- Built-in protection against CSRF, XSS, and SQL injection
- Form handling and validation
- Caching abstraction

Instead of evaluating, choosing, and maintaining ten different third-party libraries, Django provides opinionated tools that are designed to work together seamlessly.

That is its single biggest superpower.

For small and medium-sized engineering teams, this dramatically increases delivery speed. Everyone follows the exact same conventions.

```
Standard Django App Structure
├── models.py       → Data definitions & database schema
├── views.py        → Request handling & business logic
├── urls.py         → Route declarations
├── admin.py        → Back-office admin configuration
└── tests.py        → Unit & integration tests
```

When a new engineer joins the team, they don't need a three-hour meeting to learn our bespoke folder layout. They already know where everything lives.

---

### When would you NOT use Django?

I don't think Django is the right hammer for every nail.

If I am building:

- A CRUD-heavy web app
- An internal admin portal or dashboard
- A SaaS backend with relational data and complex business rules
- An enterprise application requiring bulletproof user auth and permissions

Django is almost always my first choice.

However:

- If I'm building a tiny, single-purpose microservice that exposes only two endpoints, Django's overhead is unnecessary.
- If the application is heavily I/O-bound and requires tens of thousands of concurrent open connections (like a chat gateway or streaming telemetry service), FastAPI or Go might be a better fit because of their native async-first design.

Good backend engineering is about selecting the right tool for the problem, not worshipping a favorite framework.

---

## Interview Question

**Django says it follows the MVT pattern instead of MVC. What is the real difference?**

---

### Answer

In practice, MVT and MVC solve the exact same problem: separating data, business logic, and presentation.

The naming difference is mostly semantic.

Here is how they line up:

```
Traditional MVC              Django MVT
───────────────              ──────────
Model            ───────►    Model       (Data structure & database tables)
View             ───────►    Template    (HTML presentation layer)
Controller       ───────►    View        (Request processing & business logic)
```

In traditional MVC (like Ruby on Rails or Spring Boot), the **View** is the HTML page, and the **Controller** receives HTTP requests and orchestrates models.

Django flipped the terms:

- **Model**: Still the database model (`models.py`).
- **View**: Acts as the controller. It accepts the `HttpRequest`, talks to the ORM, executes business logic, and returns an `HttpResponse`.
- **Template**: The presentation layer (`.html` files with Django template tags).

Where did the controller go?

In Django, the controller's responsibilities are divided between:

1. The **URL Dispatcher** (`urls.py`) which directs the request.
2. The **Middleware pipeline** which intercepts the request.
3. The **View function or class** which executes the core logic.

In an interview, I emphasize that the naming distinction is trivial. What matters is the clean separation of concerns.

---

## Interview Question

**What is the difference between a Django Project and a Django Application?**

---

### Answer

Think of a **project** as the entire website or backend service, and an **application** as a self-contained module within that service.

```
Django Project: ecommerce_backend/
│
├── settings.py           → Global configuration & database connection
├── urls.py               → Root URL routing
├── wsgi.py / asgi.py     → Application server entry point
│
├── users/ (App 1)        → Handles authentication, profiles, permissions
├── products/ (App 2)     → Handles product catalog, categories, inventory
└── orders/ (App 3)       → Handles shopping carts, checkouts, payments
```

An app should ideally focus on a single domain.

You create a project once:

```bash
django-admin startproject config .
```

You create apps as features grow:

```bash
python manage.py startapp orders
```

In production, keeping apps decoupled makes testing easier. If the `orders` app needs to interact with `products`, it should import models cleanly or use service-layer functions rather than tangling dependencies.

---

# 2. The Django Request-Response Lifecycle

This is one of the most frequent senior backend interview questions.

> *"Walk me through step by step what happens when a user visits `GET /api/v1/orders/42`."*

Here is the mental model:

```
Client (Browser / Mobile App)
          │  HTTPS
          ▼
┌────────────────────────────────────────┐
│ Reverse Proxy (Nginx / Cloudflare)     │ ──► Serves static files directly
└────────────────────────────────────────┘
          │  Unix Socket / HTTP
          ▼
┌────────────────────────────────────────┐
│ WSGI / ASGI Server (Gunicorn / Uvicorn)│ ──► Manages worker processes & threads
└────────────────────────────────────────┘
          │  WSGI / ASGI Callable
          ▼
┌────────────────────────────────────────┐
│ Django Middleware (Request Phase)      │ ──► Security, Session, Auth, CSRF
└────────────────────────────────────────┘
          │
          ▼
┌────────────────────────────────────────┐
│ URL Resolver (urls.py)                 │ ──► Matches regex/path to View
└────────────────────────────────────────┘
          │
          ▼
┌────────────────────────────────────────┐
│ View (FBV or CBV)                      │ ──► Executes business logic
│   ├── ORM (Lazy QuerySets)             │ ──► Generates SQL query
│   │     ▼                              │
│   │   Database (PostgreSQL / MySQL)    │ ──► Returns database rows
│   └── Model Instances                  │
└────────────────────────────────────────┘
          │  HttpResponse
          ▼
┌────────────────────────────────────────┐
│ Django Middleware (Response Phase)     │ ──► GZip, Cache headers, CORS
└────────────────────────────────────────┘
          │
          ▼
Client receives HTTP 200 OK + JSON / HTML
```

Let's dissect each stage.

---

## Stage 1: Reverse Proxy (Nginx)

### Why is Nginx sitting in front of Django?

Django is a web application framework, not a hardened, high-performance web server.

We place Nginx in front of Django for several critical reasons:

1. **Static and Media Files**: Django is slow at serving static files (CSS, JS, images). Nginx reads files directly from the disk cache or kernel memory using `sendfile` and serves them thousands of times faster without touching Python.
2. **SSL/TLS Termination**: Nginx handles HTTPS handshakes and decrypts incoming traffic, taking CPU load off the Python workers.
3. **Buffering Slow Clients**: If a mobile user on a 2G network uploads a 10MB photo, Nginx buffers the request completely before passing it to Python. If Django held an open worker thread for 60 seconds waiting for a slow client upload, all your application workers would quickly be exhausted.
4. **Load Balancing & Rate Limiting**: Nginx can distribute requests across multiple Gunicorn server nodes and drop malicious DDoS traffic before it reaches your backend.

---

## Stage 2: Application Server (Gunicorn / Uvicorn)

### Why can't Nginx directly run Django?

Nginx speaks HTTP. Django speaks Python.

Nginx cannot execute Python code directly. It needs a process manager that can load the Python interpreter, import your Django project, and invoke Python functions.

That process manager is **Gunicorn** (for WSGI) or **Uvicorn** (for ASGI).

```
Nginx (HTTP) ──► Unix Socket ──► Gunicorn Master Process
                                      │
                                      ├── Worker 1 (Python Interpreter + Django)
                                      ├── Worker 2 (Python Interpreter + Django)
                                      └── Worker 3 (Python Interpreter + Django)
```

Gunicorn manages a pool of worker processes. When a request arrives, Gunicorn hands it to an idle worker. If a worker crashes due to a memory leak or segfault, the Gunicorn master process automatically spins up a replacement worker.

---

### Why can't we use `python manage.py runserver` in production?

Interviewers love this question to test if a candidate understands production operations.

`runserver` is purely a local development convenience tool:

- It is single-threaded (or minimally threaded) and cannot handle concurrent users.
- It has not been audited for security vulnerabilities or denial-of-service resilience.
- It actively watches the filesystem for code changes, consuming CPU cycles.
- It has no worker process management, graceful restarts, or request timeout handling.

Deploying `runserver` to production will cause the server to freeze under even moderate traffic.

---

## Stage 3: The Protocol Bridge (WSGI vs ASGI)

### What is WSGI?

WSGI stands for **Web Server Gateway Interface** (PEP 3333).

It is a standardized specification that defines how a web server (like Gunicorn) talks to a Python application (like Django).

Under WSGI, the application is simply a callable object (usually a function or class with a `__call__` method) that takes two arguments:

```python
def application(environ, start_response):
    status = '200 OK'
    headers = [('Content-Type', 'text/plain')]
    start_response(status, headers)
    return [b"Hello World"]
```

WSGI is **synchronous**. A single request ties up a worker process or thread from start to finish. If your view performs a database query that takes 2 seconds, that worker cannot touch any other request for those 2 seconds.

---

### What is ASGI?

ASGI stands for **Asynchronous Server Gateway Interface**.

It is the modern, async-capable successor to WSGI. It allows Django to handle:

- Long-lived connections (WebSockets)
- Server-Sent Events (SSE)
- Asynchronous views that `await` network calls or external APIs without blocking other incoming requests.

```
WSGI: 1 Request = 1 Worker thread blocked until complete.
ASGI: 1 Worker event loop can interleave hundreds of concurrent waiting I/O requests.
```

---

## Stage 4: Django Middleware Pipeline

Once the request enters Django's `WSGIHandler` or `ASGIHandler`, Django converts the raw environment dictionary into an `HttpRequest` object.

The request now travels through the **Middleware stack**.

```python
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```

### How Middleware executes (The Onion Model)

Think of middleware as concentric layers around an onion. Your view is at the center:

```
Incoming Request ──► [ SecurityMiddleware ]
                        └──► [ SessionMiddleware ]
                                └──► [ AuthenticationMiddleware ]
                                        └──► [ View Function ]
                                                    │
Outgoing Response ◄── [ SecurityMiddleware ]        │
                        ┌─── [ SessionMiddleware ] ◄┘
                        └─── [ AuthenticationMiddleware ]
```

1. **Request Phase (Top to Bottom)**:
   - `SessionMiddleware` reads the session cookie and attaches `request.session`.
   - `AuthenticationMiddleware` reads the user ID from `request.session`, fetches the user from the database or cache, and attaches `request.user`.
   - `CsrfViewMiddleware` checks whether a `POST`/`PUT`/`DELETE` request includes a valid CSRF token.
2. **View Execution**:
   - The view runs and returns an `HttpResponse`.
3. **Response Phase (Bottom to Top)**:
   - Each middleware receives the response on the way out. For example, `SecurityMiddleware` attaches security headers (`X-Content-Type-Options`, HSTS) and `SessionMiddleware` writes back modified session cookies.

> **Crucial Rule**: Middleware order in `settings.py` matters. If `AuthenticationMiddleware` appears *before* `SessionMiddleware`, Django crashes with an `ImproperlyConfigured` error because authentication depends on `request.session` already existing.

---

## Stage 5: URL Resolution (`urls.py`)

After the request passes all incoming middleware, Django passes `request.path_info` to the root URLconf defined in `ROOT_URLCONF`.

Django iterates sequentially through the `urlpatterns` list:

```python
# urls.py
urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/v1/orders/<int:order_id>/', views.order_detail, name='order-detail'),
]
```

1. Django uses path converters (like `<int:order_id>`) or regular expressions (`re_path`) to match the URL.
2. When a match is found, Django captures the parameters and converts `<int:order_id>` directly into a Python integer (`42`).
3. If no route matches, Django raises `Http404`, bypassing the view entirely.

---

## Stage 6: View Execution & ORM

The matched view function or class-based view method executes:

```python
def order_detail(request, order_id):
    # 1. Check permissions
    if not request.user.is_authenticated:
        return HttpResponseForbidden()
    
    # 2. Query the database using the ORM
    order = get_object_or_404(Order, id=order_id, user=request.user)
    
    # 3. Return the response
    return JsonResponse({
        "id": order.id,
        "total": str(order.total_amount),
        "status": order.status
    })
```

- If an unhandled exception occurs inside your view, Django halts and executes the `process_exception` hooks of your middleware from bottom to top.
- Otherwise, the view returns an `HttpResponse`, which bubbles back up through the middleware response chain and returns through Gunicorn and Nginx to the user.

---

# 3. Settings & Configuration Best Practices

## Interview Question

**How do you manage Django settings across development, staging, and production environments?**

---

### Answer

One of the worst production anti-patterns is keeping a single `settings.py` file with comments like `# uncomment for production`.

In production, I follow the **Twelve-Factor App** methodology: store configuration in the environment, not in code.

### Pattern 1: Environment Variables with `django-environ`

Never commit secrets (`SECRET_KEY`, database passwords, Stripe API keys) to Git.

```python
# settings.py
import environ

env = environ.Env(
    DEBUG=(bool, False)
)

# Read .env file if it exists (for local dev)
environ.Env.read_env()

SECRET_KEY = env('SECRET_KEY')
DEBUG = env('DEBUG')

DATABASES = {
    'default': env.db('DATABASE_URL')
}
```

Locally, developers use a `.env` file:
```bash
DEBUG=True
SECRET_KEY=local-dev-insecure-key
DATABASE_URL=postgres://postgres:postgres@localhost:5432/mydb
```

In production (Kubernetes, AWS ECS, or Heroku), secrets are injected into the environment container directly.

---

### Pattern 2: Split Settings Files

For large projects with substantial differences between local, test, and production configs:

```
config/
├── settings/
│   ├── __init__.py
│   ├── base.py         → Shared configuration (INSTALLED_APPS, MIDDLEWARE)
│   ├── local.py        → DEBUG=True, console email backend, Django Debug Toolbar
│   ├── test.py         → In-memory database, fast password hashers
│   └── production.py   → S3 storage, Redis cache, strict cookie security
```

Set `DJANGO_SETTINGS_MODULE=config.settings.production` in your production deployment.

---

## Production Checklist: Things that must NEVER be in Production

1. `DEBUG = True`: If an error occurs, Django displays a traceback revealing environment variables, database credentials, and source code to anyone on the web.
2. Hardcoded `SECRET_KEY`: Anyone with the key can forge session cookies and bypass authentication.
3. `ALLOWED_HOSTS = ['*']`: Leaves the application vulnerable to HTTP Host Header poisoning attacks (e.g. password reset poisonings). Always specify exact domains (`['api.mycompany.com']`).
4. Plain HTTP Cookies: In production, `SESSION_COOKIE_SECURE = True` and `CSRF_COOKIE_SECURE = True` ensure browsers only transmit authentication cookies over encrypted HTTPS.
