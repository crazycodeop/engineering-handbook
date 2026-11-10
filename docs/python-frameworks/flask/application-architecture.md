# Flask Application Architecture: Contexts & Blueprints

> This guide explores the internal architecture of Flask, the Application and Request Contexts, the Application Factory pattern, Blueprints, and request lifecycle hooks.

---

# 1. The Two Contexts of Flask

This is one of the most famous and fundamental senior Flask interview questions.

## Interview Question

**Why does Flask have both an "Application Context" and a "Request Context"? What causes the dreaded `RuntimeError: Working outside of application context` error?**

---

### The Problem Flask Solved: Global Variables in Multi-Threaded Servers

In your code, you import `request` and `current_app` as if they are global variables:

```python
from flask import request, current_app
```

Now imagine Gunicorn is running with 8 worker threads.

At the exact same millisecond:
- Thread 1 handles Request A from User Alice.
- Thread 2 handles Request B from User Bob.

If `request` were a true Python global variable, Bob's request data would overwrite Alice's request data!

Flask solves this using **Thread-Local Storage** (`werkzeug.local.LocalProxy`).

When your code accesses `request.args`, the `LocalProxy` inspects the ID of the current operating system thread, looks up the specific request state allocated to that thread, and returns Alice's data to Thread 1 and Bob's data to Thread 2.

---

### The Two Contexts

Flask divides request state into two separate stacks:

```
┌────────────────────────────────────────────────────────┐
│                   Application Context                  │
│   (Created when handling a request, or manually pushed)│
│                                                        │
│   • current_app: Points to the active Flask instance   │
│   • g: Request-scoped scratchpad for temporary data    │
└────────────────────────────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│                     Request Context                    │
│   (Created when an HTTP connection arrives)            │
│                                                        │
│   • request: The incoming HTTP payload, headers, URLs  │
│   • session: The client's signed session dictionary    │
└────────────────────────────────────────────────────────┘
```

---

### Why does Application Context exist separately from Request Context?

Why not just combine everything into the Request Context?

Because there are many scenarios where you need the application's configuration, database engine, or logging settings **without an HTTP request being present**:

- Running Celery background tasks
- Running database migration scripts
- Executing CLI commands (`flask db upgrade`, `flask create-admin`)
- Running unit tests

---

### The Dreaded Error: `RuntimeError: Working outside of application context`

Suppose you write a script to query the database outside of a view:

```python
# test_script.py
from myproject import db
from myproject.models import User

# CRASHES WITH: RuntimeError: Working outside of application context.
users = User.query.all()
```

### Why it crashes:
`User.query` needs to know what database URL to connect to. That configuration lives inside `current_app.config`.

Because no HTTP request is happening, no Application Context was automatically pushed onto the thread stack. `current_app` points to nothing!

### The Fix: Push the context manually using `with app.app_context()`

```python
from myproject import create_app, db
from myproject.models import User

app = create_app()

with app.app_context():
    # Application context is active inside this block!
    users = User.query.all()
    print(users)
```

---

# 2. What is `g`?

## Interview Question

**What is Flask's `g` object, and how is it used during the request lifecycle?**

---

### Answer

`g` stands for **global**, but it is **strictly request-scoped**.

Think of `g` as a scratchpad dictionary that lives for the duration of a single HTTP request and is destroyed when the request finishes:

```python
from flask import g, request

@app.before_request
def authenticate():
    auth_header = request.headers.get("Authorization")
    user = verify_token_and_get_user(auth_header)
    # Store authenticated user on g
    g.user = user

@app.route("/profile")
def profile():
    # Access the user stored during before_request
    if g.user is None:
        return {"error": "Unauthorized"}, 401
    return {"username": g.user.username}
```

### Common uses for `g`:
- Storing the currently authenticated user (`g.user`)
- Storing request start time to measure endpoint latency
- Storing an open database transaction or connection to reuse across multiple helper functions within the same request

---

# 3. The Application Factory Pattern (`create_app`)

## Interview Question

**Why is defining a global `app = Flask(__name__)` considered an anti-pattern in production? How does the Application Factory pattern solve this?**

---

### The Anti-Pattern: Global App Instance

In beginner tutorials, applications are structured like this:

```python
# app.py
from flask import Flask
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///mydb.db'
db = SQLAlchemy(app)  # Tightly bound to global app!
```

### Why this breaks in production:
1. **Circular Imports**: Views need to import `db` from `app.py`, while `app.py` needs to import views to register routes. As the project grows, circular import errors crash Python.
2. **Testing Nightmares**: You cannot run one test against an in-memory SQLite database and another test against PostgreSQL. The global `app` is configured once when Python starts.
3. **Multiple Instances**: You cannot run multiple instances of the app (e.g. testing different configurations) in the same process.

---

### The Production Standard: Application Factory

Instead of creating a global `app`, wrap initialization inside a function called `create_app()`:

```
my_app/
├── __init__.py           → Application Factory & Extension setup
├── config.py             → Config classes (DevConfig, ProdConfig, TestConfig)
├── models.py             → SQLAlchemy models
└── blueprints/
    ├── auth.py           → Authentication routes
    └── api.py            → Business logic routes
```

### Step 1: Initialize Extensions Unbound

```python
# my_app/__init__.py
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate

# Initialize extensions without attaching to an app yet!
db = SQLAlchemy()
migrate = Migrate()

def create_app(config_name="config.DevConfig"):
    app = Flask(__name__)
    app.config.from_object(config_name)

    # Bind extensions to this specific app instance
    db.init_app(app)
    migrate.init_app(app, db)

    # Register Blueprints
    from my_app.blueprints.auth import auth_bp
    from my_app.blueprints.api import api_bp

    app.register_blueprint(auth_bp, url_prefix="/auth")
    app.register_blueprint(api_bp, url_prefix="/api/v1")

    return app
```

Now, your test suite can spin up completely isolated app instances:

```python
# tests/conftest.py
import pytest
from my_app import create_app, db

@pytest.fixture
def app():
    app = create_app("config.TestConfig")  # Isolated test database!
    with app.app_context():
        db.create_all()
        yield app
        db.drop_all()
```

---

# 4. Blueprints: Modularizing Features

## Interview Question

**What is a Flask Blueprint, and how do you use Blueprints to organize large applications?**

---

### Answer

A **Blueprint** is a way to organize a group of related views, static files, templates, and error handlers into a self-contained module.

Think of a Blueprint as an application component waiting to be registered to a main Flask application:

```python
# my_app/blueprints/auth.py
from flask import Blueprint, render_template, request, redirect, url_for

# Define the blueprint
auth_bp = Blueprint("auth", __name__, template_folder="templates")

@auth_bp.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        # ... authenticate ...
        return redirect(url_for("auth.dashboard"))
    return render_template("auth/login.html")

@auth_bp.route("/dashboard")
def dashboard():
    return "User Dashboard"
```

Notice `url_for("auth.dashboard")`.

When reversing URLs registered inside a Blueprint, you prefix the view name with the Blueprint's name: `blueprint_name.view_function_name`.

---

# 5. Request Lifecycle Hooks

## Interview Question

**Walk me through the execution order of `before_request`, `after_request`, and `teardown_request`. Which one runs if an unhandled exception occurs?**

---

### Answer

Flask provides lifecycle hooks to intercept requests before, during, and after view execution:

```
Incoming Request
       │
       ▼
@app.before_request  ──► Authentication, rate limiting, request timing
       │
       ▼
View Function        ──► Returns Response (or raises Exception)
       │
       ▼
@app.after_request   ──► Runs ONLY if view succeeded! Modifies headers (CORS, cache)
       │
       ▼
Response sent to client
       │
       ▼
@app.teardown_request ──► Runs ALWAYS (even on crashes). Database cleanup!
```

---

### Key Differences:

1. **`before_request`**:
   - Executes before every request.
   - If it returns a value (e.g. returning a 401 response on invalid auth), **view execution is aborted immediately**.
2. **`after_request(response)`**:
   - Must take `response` and must return a `response` object.
   - Executes **only if the view completed without raising an unhandled exception**.
3. **`teardown_request(exception)`**:
   - Executes **always**, even if an unhandled 500 error crashed the view!
   - Takes `exception` as an argument (will be `None` if no error occurred).
   - This is the only safe place to close database connections or clean up locks.
