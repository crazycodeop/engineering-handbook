# Flask Fundamentals & Core Architecture

> This guide introduces Flask, explains why it is classified as a microframework, explores its WSGI foundation (Werkzeug) and templating engine (Jinja2), and covers routing and parameter handling.

---

# 1. What is Flask & Why is it Called a "Microframework"?

## Interview Question

**What is Flask, and what does the term "microframework" actually mean? Does it mean Flask is only suitable for small projects?**

---

### Answer

Flask is a lightweight Python web framework created by **Armin Ronacher** in 2010 under the Pallets Projects.

The term **micro** does not mean your application must be small, nor does it mean Flask lacks functionality.

It means Flask's **core is intentionally minimal and unopinionated**:

```
Django:   Includes ORM, Admin, Auth, Migrations, Forms, Caching (Batteries-included).
Flask:    Includes Web Router (Werkzeug) and Template Engine (Jinja2). Everything else is up to you.
```

In Flask:
- There is no default database or ORM (you choose SQLAlchemy, Peewee, or raw SQL).
- There is no default authentication system (you choose Flask-Login, Flask-Security, or JWT).
- There is no default form validation or migration engine.

### Why do engineering teams choose Flask?

1. **Total Freedom & Flexibility**: You are not forced into Django's design patterns or ORM conventions. You choose the exact libraries that fit your architecture.
2. **Simplicity & Low Overhead**: You can write an entire working microservice in a single 15-line Python file.
3. **No Unused Bloat**: Your application memory footprint is tiny, which makes Flask great for serverless functions (AWS Lambda), lightweight microservices, and custom backend proxies.

---

### Flask vs Django vs FastAPI

| Dimension | Django | Flask | FastAPI |
|---|---|---|---|
| **Architecture** | Monolithic & Opinionated | Microframework (Minimal) | Microframework (Typed APIs) |
| **Data Layer** | Built-in Django ORM | User choice (Typically SQLAlchemy) | User choice (Typically SQLAlchemy) |
| **Primary Use** | Full-stack apps, SaaS, Admin | Custom services, microservices | High-concurrency REST APIs |
| **Protocol** | WSGI (synchronous) | WSGI (synchronous) | ASGI (asynchronous) |
| **Learning Curve** | High (Lots of conventions) | Low (Start in 5 minutes) | Moderate (Type hints & async) |

---

# 2. Application Structure & The WSGI Foundation

## Interview Question

**How does a basic Flask application work, and what role does Werkzeug play?**

---

### Answer

Flask is essentially a wrapper around two foundational libraries:

1. **Werkzeug**: A comprehensive WSGI web application library that handles HTTP routing, request/response objects, cookie handling, and debugging.
2. **Jinja2**: A fast, sandboxed Python templating engine.

---

### The Minimal Application

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route("/")
def home():
    return {"message": "Hello from Flask"}

@app.route("/items/<int:item_id>", methods=["GET", "PUT"])
def item_detail(item_id):
    if request.method == "PUT":
        data = request.get_json()
        return jsonify({"item_id": item_id, "updated": data}), 200
    
    return jsonify({"item_id": item_id, "name": "Sample Item"})

if __name__ == "__main__":
    app.run(debug=True, port=5000)
```

### Why `__name__` is passed to `Flask(__name__)`
Flask uses `__name__` (the current Python module name) to determine the root path of the project. This allows Flask to locate static files (`/static`) and templates (`/templates`) relative to where the application file lives.

---

# 3. Routing & Parameter Handling

## Interview Question

**How does Flask extract path variables, query string parameters, form data, and JSON bodies from incoming requests?**

---

### Answer

In Flask, incoming request information is accessed via the global-looking `request` proxy object:

```python
from flask import request
```

*(Note: Although `request` looks like a global variable, it is actually a thread-local proxy that safely points to the specific request being handled by the current thread).*

---

### 1. Path Parameters (URL Converters)

Path variables are declared using `<converter:variable_name>` in `@app.route`:

```python
@app.route("/users/<int:user_id>/posts/<slug:post_slug>")
def show_post(user_id, post_slug):
    # user_id is automatically cast to an int!
    return f"User {user_id}, Post {post_slug}"
```

**Built-in Converters:**
- `string` (default): Accepts any text without slashes.
- `int`: Accepts positive integers and casts to Python `int`.
- `float`: Accepts floating-point values.
- `path`: Accepts text including slashes (e.g. `/files/<path:filepath>`).
- `uuid`: Accepts UUID strings.

---

### 2. Query Parameters (`request.args`)

Query strings (`/search?q=python&limit=10`) are accessed via `request.args`:

```python
@app.route("/search")
def search():
    query = request.args.get("q", default="", type=str)
    limit = request.args.get("limit", default=20, type=int)
    return jsonify({"query": query, "limit": limit})
```

`request.args.get()` allows providing default values and automatic type casting (`type=int`).

---

### 3. JSON Request Bodies (`request.get_json()`)

When a client sends `Content-Type: application/json`:

```python
@app.route("/api/orders", methods=["POST"])
def create_order():
    # Parses JSON body into a Python dict; returns None or 400 if malformed
    data = request.get_json(silent=False)
    
    if not data or "item_id" not in data:
        return jsonify({"error": "Missing item_id"}), 400

    return jsonify({"status": "created", "order_id": 123}), 201
```

---

### 4. HTML Form Data (`request.form`)

When submitting standard HTML forms (`application/x-www-form-urlencoded` or `multipart/form-data`):

```python
@app.route("/login", methods=["POST"])
def login():
    username = request.form.get("username")
    password = request.form.get("password")
    return f"Logging in {username}"
```

---

### Quick Reference: Where data lives in `request`

| Payload Type | Source in Flask | Example URL / Header |
|---|---|---|
| **Path Variable** | View function arguments | `/users/<int:id>` |
| **Query Parameters** | `request.args` | `/users?page=2` |
| **JSON Body** | `request.get_json()` | `Content-Type: application/json` |
| **HTML Form** | `request.form` | Standard `<form method="POST">` |
| **File Uploads** | `request.files` | `<input type="file" name="avatar">` |
| **HTTP Headers** | `request.headers` | `request.headers.get("Authorization")` |
| **Cookies** | `request.cookies` | `request.cookies.get("session_id")` |

---

# 4. Responses, Status Codes & Redirections

## Interview Question

**What can a Flask view function return, and how do you construct custom HTTP headers and status codes?**

---

### Answer

A Flask view function can return:

1. **A String**: Automatically wrapped in an HTTP 200 `text/html` response.
2. **A Dictionary or List**: Automatically converted to JSON via `jsonify()` with `application/json` headers (Flask 1.1+).
3. **A Tuple**: `(response_body, status_code, headers)`.
4. **A Response Object**: Created via `make_response()`.

```python
from flask import make_response, redirect, url_for

# 1. Returning a tuple with custom status code and headers
@app.route("/created")
def created():
    return {"status": "success"}, 201, {"X-Custom-Header": "MyValue"}

# 2. Explicit Response object
@app.route("/custom-cookie")
def set_cookie_view():
    resp = make_response({"message": "Cookie set"})
    resp.set_cookie("user_pref", "dark_mode", httponly=True, secure=True)
    return resp

# 3. Redirecting
@app.route("/old-path")
def old_path():
    # Redirects dynamically using the view function name
    return redirect(url_for("home"), code=301)
```

Always use `url_for("function_name")` when redirecting rather than hardcoding URL strings. If the route changes in the future, `url_for` automatically uses the updated path.
