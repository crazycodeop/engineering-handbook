# Advanced Flask & Production Engineering

> This guide covers WSGI middleware, the crucial decorator pitfall with `functools.wraps`, building REST APIs, automated testing with pytest, Celery task coordination, and Gunicorn production deployment.

---

# 1. WSGI Middleware & Reverse Proxies (`ProxyFix`)

## Interview Question

**How does WSGI middleware work in Flask, and why is `ProxyFix` mandatory when deploying behind Nginx or AWS load balancers?**

---

### Answer

Flask applications are standard WSGI callables. The application object has an internal `app.wsgi_app` attribute that actually receives requests from Gunicorn.

WSGI middleware works by wrapping that callable inside another callable:

```
Gunicorn ──► [ Middleware 1 ] ──► [ Middleware 2 ] ──► app.wsgi_app (Flask)
```

---

### Why `ProxyFix` is Mandatory Behind Nginx

When your application sits behind a reverse proxy (like Nginx, Cloudflare, or an AWS Application Load Balancer), Nginx establishes a new TCP connection to Gunicorn.

If you inspect the incoming request in Flask:

```python
client_ip = request.remote_addr  # Will be '127.0.0.1' (Nginx's IP!), not the actual user!
is_secure = request.is_secure    # False! Because Nginx decrypted HTTPS and forwarded HTTP!
```

If your code checks `request.remote_addr` for rate limiting or security audits, you are rate-limiting `127.0.0.1`, which blocks all users! If you redirect users to HTTPS, Flask creates an infinite redirect loop because it thinks every request is insecure HTTP!

### The Solution: Werkzeug's `ProxyFix`

`ProxyFix` reads the `X-Forwarded-For` and `X-Forwarded-Proto` headers injected by Nginx and replaces `request.remote_addr` and `request.scheme` with the real client values:

```python
from werkzeug.middleware.proxy_fix import ProxyFix
from flask import Flask

app = Flask(__name__)

# Wrap the WSGI app with ProxyFix
# x_for=1: Trust 1 proxy hop for client IP (Nginx)
# x_proto=1: Trust 1 proxy hop for protocol (http vs https)
app.wsgi_app = ProxyFix(app.wsgi_app, x_for=1, x_proto=1)
```

---

# 2. Writing Custom Decorators & The `functools.wraps` Bug

## Interview Question

**What happens if you write a custom Flask view decorator without `@functools.wraps`? What error does Flask raise?**

---

### Answer

Suppose you write a custom `@admin_required` decorator:

```python
# BUGGY IMPLEMENTATION (Missing @wraps)
def admin_required(f):
    def wrapper(*args, **kwargs):
        if not current_user.is_admin:
            abort(403)
        return f(*args, **kwargs)
    return wrapper
```

Now you apply it to two routes:

```python
@app.route("/admin/users")
@admin_required
def manage_users():
    return "Users"

@app.route("/admin/settings")
@admin_required
def manage_settings():
    return "Settings"
```

### The Crash:
When you start the application, Flask crashes with:

```
AssertionError: View function mapping is overwriting an existing endpoint function: wrapper
```

### Why it crashes:
Flask uses the function's `__name__` as the unique endpoint identifier in its routing table.

Because the decorator replaced both `manage_users` and `manage_settings` with `wrapper`, both functions now have `__name__ == "wrapper"`.

Flask sees the second route registering an endpoint named `"wrapper"`, assumes you registered the exact same view twice, and raises an `AssertionError`.

---

### The Fix: Always use `@functools.wraps`

```python
import functools
from flask import abort
from flask_login import current_user

def admin_required(f):
    @functools.wraps(f)  # Preserves f's original __name__, docstrings, and signature!
    def wrapper(*args, **kwargs):
        if not current_user.is_authenticated or not current_user.is_admin:
            abort(403)
        return f(*args, **kwargs)
    return wrapper
```

`@functools.wraps(f)` copies the original function's name and metadata onto `wrapper`, keeping Flask's endpoint names unique (`"manage_users"` and `"manage_settings"`).

---

# 3. Testing Flask Applications with `pytest`

## Interview Question

**How do you write clean, isolated unit and integration tests for a Flask application using `pytest`?**

---

### Answer

Using the Application Factory pattern, we define test fixtures in `conftest.py`:

```python
# tests/conftest.py
import pytest
from myproject import create_app, db

@pytest.fixture
def app():
    # 1. Create app with in-memory SQLite config
    app = create_app("config.TestingConfig")

    with app.app_context():
        db.create_all()  # Build fresh tables
        yield app
        db.session.remove()
        db.drop_all()    # Clean up tables after test

@pytest.fixture
def client(app):
    # Returns Flask's built-in test client
    return app.test_client()
```

---

### Writing Test Cases

```python
# tests/test_orders.py
def test_get_orders_unauthorized(client):
    response = client.get("/api/v1/orders")
    assert response.status_code == 401

def test_create_order_success(client, app):
    # Test POST request with JSON
    payload = {"item_id": 42, "quantity": 2}
    response = client.post("/api/v1/orders", json=payload)

    assert response.status_code == 201
    assert response.json["status"] == "created"
```

---

# 4. Background Tasks with Celery & Application Context

## Interview Question

**How do you integrate Celery with Flask, and why must Celery tasks run within `app.app_context()`?**

---

### Answer

Celery worker processes run outside of Flask's HTTP request lifecycle.

If a Celery task needs to query the database using `User.query.all()`, it will crash with `RuntimeError: Working outside of application context`.

The standard Pallets-recommended pattern is to subclass Celery's `Task` class so every task execution automatically pushes an application context:

```python
# myproject/celery_utils.py
from celery import Celery, Task

def make_celery(app):
    class FlaskTask(Task):
        def __call__(self, *args, **kwargs):
            # Automatically push app context during task execution!
            with app.app_context():
                return self.run(*args, **kwargs)

    celery_app = Celery(app.import_name, task_cls=FlaskTask)
    celery_app.config_from_object(app.config["CELERY"])
    celery_app.set_default()
    app.extensions["celery"] = celery_app
    return celery_app
```

Now, any Celery task can access `db.session` or `current_app.config` without manual context management:

```python
from celery import shared_task
from myproject.models import Order
from myproject import db

@shared_task
def send_order_receipt(order_id):
    # Database query works seamlessly!
    order = Order.query.get(order_id)
    send_email(order.user.email, f"Receipt for Order {order.id}")
```

---

# 5. Production Deployment with Gunicorn

## Interview Question

**How do you configure Gunicorn for a production Flask deployment? What worker type should you choose?**

---

### Answer

Flask's built-in `app.run()` server is single-threaded and unsafe for production.

In production, run Flask using **Gunicorn**:

```bash
gunicorn "myproject:create_app()" \
    --workers 4 \
    --worker-class gthread \
    --threads 2 \
    --bind 0.0.0.0:8000 \
    --access-logfile - \
    --error-logfile -
```

### Choosing the Right Worker Type:

1. **`sync` (Default)**:
   - Each worker process handles 1 request at a time.
   - Best for CPU-bound tasks or simple apps with fast database queries.
2. **`gthread` (Threaded Workers)**:
   - Workers spawn OS threads to handle multiple concurrent requests.
   - Low memory overhead, excellent for applications waiting on database or file I/O.
3. **`gevent` (Async Greenlets)**:
   - Uses cooperative greenlet coroutines for high-concurrency long polling or WebSockets.
