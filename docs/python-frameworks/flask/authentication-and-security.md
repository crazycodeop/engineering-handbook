# Flask Authentication & Web Security

> This guide explains Flask's session architecture, the critical security distinction between signed and encrypted cookies, user authentication with Flask-Login and JWT, and protections against CSRF, XSS, and SQL injection.

---

# 1. Flask Sessions: The Critical Security Distinction

## Interview Question

**How do default sessions work in Flask? Why is storing sensitive data in a default Flask session a catastrophic security vulnerability?**

---

### Answer

In Django, the session cookie contains only a random 32-character string (`sessionid`). All user data is stored on the server side in PostgreSQL or Redis.

**Flask does the exact opposite by default.**

In Flask, the session data is stored **directly inside the client's browser cookie**:

```python
from flask import Flask, session

app = Flask(__name__)
app.secret_key = "super-secret-key"

@app.route("/set-session")
def set_session():
    session["user_id"] = 42
    session["role"] = "admin"
    return "Session saved"
```

---

### The Critical Security Pitfall: Signed vs Encrypted

The session cookie sent to the client looks like this:

```
eyJyZXN1bHQiOiJhZG1pbiIsInVzZXJfaWQiOjQyfQ.ZZE9Lg.7dK...
```

Flask sessions are **cryptographically signed, NOT encrypted**.

- **Signing** means the server signs the payload with `secret_key` so the client cannot *tamper* with or change `"role": "admin"` to `"role": "superadmin"`.
- **However, signing provides ZERO secrecy!**

Anyone can copy that cookie string, paste it into a base64 decoder, and **read the entire session payload in plaintext**:

```json
{"role": "admin", "user_id": 42}
```

If a junior developer stores sensitive data in a default Flask session:

```python
# CATASTROPHIC BUG: Exposes sensitive data to the client!
session["api_token"] = "sk_live_123456789"
session["ssn"] = "123-45-6789"
```

The user or anyone inspecting network packets can immediately steal those secrets.

---

### The Production Fix: Server-Side Sessions (`Flask-Session`)

In production, use the `Flask-Session` extension to store session data in **Redis**:

```python
from flask import Flask, session
from flask_session import Session
import redis

app = Flask(__name__)
app.config["SESSION_TYPE"] = "redis"
app.config["SESSION_REDIS"] = redis.from_url("redis://localhost:6379")

Session(app)
```

Now, Flask behaves like Django: the cookie only contains a random session ID, and all actual session data lives securely inside Redis.

---

# 2. User Authentication with `Flask-Login`

## Interview Question

**How does `Flask-Login` manage user authentication state, and what is the role of the `@login_manager.user_loader` callback?**

---

### Answer

`Flask-Login` is the standard library for session-based user authentication in Flask.

### Step 1: User Model Implementation (`UserMixin`)

`Flask-Login` requires your user model to implement four properties:
- `is_authenticated`: True if user is logged in.
- `is_active`: True if account is enabled.
- `is_anonymous`: False for regular users.
- `get_id()`: Returns a unique string ID for the user.

Subclassing `UserMixin` provides all four automatically:

```python
from flask_login import UserMixin
from myproject import db

class User(db.Model, UserMixin):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True)
    password_hash = db.Column(db.String(255))
```

---

### Step 2: The `user_loader` Callback

When a logged-in user makes an HTTP request, `Flask-Login` reads the user's ID from the session cookie.

It doesn't know what database you are using. You must provide a **`user_loader` callback** that fetches the user model from the database:

```python
from flask_login import LoginManager

login_manager = LoginManager()

@login_manager.user_loader
def load_user(user_id):
    # Flask-Login passes user_id as a string; cast to int for DB query
    return User.query.get(int(user_id))
```

---

### Step 3: Logging In & Protecting Routes

```python
from flask_login import login_user, logout_user, login_required, current_user
from werkzeug.security import check_password_hash

@app.route("/login", methods=["POST"])
def login():
    username = request.form.get("username")
    password = request.form.get("password")
    
    user = User.query.filter_by(username=username).first()
    if user and check_password_hash(user.password_hash, password):
        login_user(user, remember=True)
        return redirect("/dashboard")
        
    return "Invalid credentials", 401

@app.route("/dashboard")
@login_required  # Automatically redirects unauthenticated users to login page!
def dashboard():
    return f"Welcome, {current_user.username}!"
```

---

# 3. Password Hashing with Werkzeug

## Interview Question

**How do you hash and verify passwords securely in Flask?**

---

### Answer

Always use Werkzeug's security module (`werkzeug.security`):

```python
from werkzeug.security import generate_password_hash, check_password_hash

# 1. Hashing a password during registration
raw_password = "user_secret_password"
hashed = generate_password_hash(raw_password, method="scrypt")

# 2. Verifying a password during login
is_valid = check_password_hash(hashed, "user_secret_password")  # True
is_wrong = check_password_hash(hashed, "wrong_password")        # False
```

Werkzeug automatically generates a cryptographically secure random salt for every password and uses modern adaptive hashers (`scrypt` or `pbkdf2:sha256`) to resist GPU brute-force attacks.

---

# 4. Web Security: CSRF, XSS & SQL Injection

## Interview Question

**How do you protect a Flask application from CSRF, XSS, and SQL Injection attacks?**

---

### 1. Cross-Site Request Forgery (CSRF)

Unlike Django, Flask does not include CSRF protection out of the box.

In production, you **must install `Flask-WTF`** and enable `CSRFProtect`:

```python
from flask_wtf.csrf import CSRFProtect

csrf = CSRFProtect(app)
```

In your HTML templates, inject the CSRF token into every `POST` form:

```html
<form method="POST">
    <input type="hidden" name="csrf_token" value="{{ csrf_token() }}"/>
    ...
</form>
```

For AJAX requests, configure your frontend to send the token in the `X-CSRFToken` HTTP header.

---

### 2. Cross-Site Scripting (XSS)

Flask's template engine (Jinja2) **automatically escapes all variables by default**.

If a user injects `<script>alert('xss')</script>`, Jinja2 renders:

```html
&lt;script&gt;alert(&#39;xss&#39;)&lt;/script&gt;
```

**Danger**: Never use `|safe` on user input:

```html
<!-- VULNERABLE TO STORED XSS -->
<div>{{ user_comment|safe }}</div>
```

---

### 3. SQL Injection

When using SQLAlchemy or Flask-SQLAlchemy, query parameters are automatically parameterized:

```python
# SAFE: Parameterized SQL generated
User.query.filter_by(username=untrusted_input).first()
```

The only way to introduce SQL injection in Flask is by concatenating strings in raw queries:

```python
# CRITICAL VULNERABILITY: Never use Python f-strings in raw SQL!
db.session.execute(f"SELECT * FROM users WHERE username = '{untrusted_input}'")

# SAFE: Pass parameters as a dictionary or tuple
db.session.execute(
    text("SELECT * FROM users WHERE username = :name"),
    {"name": untrusted_input}
)
```
