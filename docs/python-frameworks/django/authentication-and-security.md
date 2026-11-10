# Django Authentication, Sessions & Security

> This guide explains how Django secures web applications, how its authentication and session systems work internally, and the critical security vulnerabilities backend engineers must prevent.

---

# 1. The Authentication System & User Models

## Interview Question

**Why should you always create a custom user model at the very beginning of a Django project, and what is the difference between `AbstractUser` and `AbstractBaseUser`?**

---

### Answer

This is one of the most critical architectural decisions in Django.

By default, Django comes with `django.contrib.auth.models.User`. It includes fields like `username`, `first_name`, `last_name`, `email`, and `password`.

If you start a project using the default `User` model, and six months into production the client says:

> *"We want users to log in with their email address or phone number instead of a username."*

Changing the user model after migrations have already run in production is notoriously painful. It requires manually migrating foreign keys across dozens of tables, altering database constraints, and rewriting migration histories.

Django's official documentation explicitly recommends: **Always configure a custom user model before running your first migration.**

---

### `AbstractUser` vs `AbstractBaseUser`

When creating a custom user model, you inherit from one of two classes:

```
                       AbstractBaseUser
                     (Minimal foundation)
                              │
                              ├── Provides password hashing & last_login
                              └── You define ALL fields (email, is_active, etc.)
                              │
                              ▼
                         AbstractUser
                      (Standard Django user)
                              │
                              └── Adds username, first_name, last_name,
                                  email, is_staff, is_active, date_joined
```

### Approach 1: `AbstractUser` (Recommended for 90% of apps)

Use `AbstractUser` when you want all of Django's default behavior, permissions, and admin integration, but want the freedom to add extra fields (like `avatar`, `phone_number`, or `stripe_customer_id`):

```python
# accounts/models.py
from django.contrib.auth.models import AbstractUser
from django.db import models

class CustomUser(AbstractUser):
    phone_number = models.CharField(max_length=20, blank=True)
    is_verified = models.BooleanField(default=False)
```

Point Django to your custom model in `settings.py`:

```python
AUTH_USER_MODEL = 'accounts.CustomUser'
```

---

### Approach 2: `AbstractBaseUser` (For completely custom authentication)

Use `AbstractBaseUser` when you want complete control and want to eliminate fields like `username` entirely (e.g. logging in solely with an `email`):

```python
# accounts/models.py
from django.contrib.auth.models import AbstractBaseUser, BaseUserManager, PermissionsMixin
from django.db import models

class CustomUserManager(BaseUserManager):
    def create_user(self, email, password=None, **extra_fields):
        if not email:
            raise ValueError('Email address is required')
        email = self.normalize_email(email)
        user = self.model(email=email, **extra_fields)
        user.set_password(password)  # Hashes password securely
        user.save(using=self._db)
        return user

    def create_superuser(self, email, password=None, **extra_fields):
        extra_fields.setdefault('is_staff', True)
        extra_fields.setdefault('is_superuser', True)
        return self.create_user(email, password, **extra_fields)

class CustomUser(AbstractBaseUser, PermissionsMixin):
    email = models.EmailField(unique=True)
    is_active = models.BooleanField(default=True)
    is_staff = models.BooleanField(default=False)

    objects = CustomUserManager()

    USERNAME_FIELD = 'email'  # Identifies user during login
    REQUIRED_FIELDS = []      # Required when creating superuser via CLI
```

---

# 2. How Sessions & Cookies Work Under the Hood

## Interview Question

**Walk me through how Django sessions work. What is sent to the client, and where is the actual user data stored?**

---

### Answer

HTTP is a stateless protocol. The server does not remember who you are from one request to the next.

Django solves this through **Session-Based Authentication**:

```
Client (Browser)                                    Django Server
       │                                                  │
       │ ──► POST /login (username, password) ──────────► │
       │                                                  │ ──► Authenticates credentials
       │                                                  │ ──► Generates session_key = "abc123xyz"
       │                                                  │ ──► Stores data in DB/Redis:
       │                                                  │     session_key ──► { "_auth_user_id": 42 }
       │ ◄── Set-Cookie: sessionid=abc123xyz; HttpOnly ── │
       │                                                  │
       │ ──► GET /profile (Cookie: sessionid=abc123xyz) ─► │
       │                                                  │ ──► SessionMiddleware reads cookie
       │                                                  │ ──► Fetches { "_auth_user_id": 42 }
       │                                                  │ ──► Attaches request.user = User(id=42)
       │ ◄── 200 OK (Profile Data) ────────────────────── │
```

### Key Insight: The Cookie Contains ZERO Personal Data

The client cookie (`sessionid`) contains **only a random 32-character alphanumeric string** (the session key).

It does *not* contain the user's ID, username, role, or permissions.

All session payload data is serialized and stored on the **server side**.

---

### Where are sessions stored?

Configured via `SESSION_ENGINE` in `settings.py`:

1. **Database-backed (`django.contrib.sessions.backends.db`) [Default]**:
   - Stores session data in the `django_session` table.
   - Simple and requires no extra infrastructure, but every HTTP request executes a `SELECT` on the database to look up the session key.
2. **Cached Database (`django.contrib.sessions.backends.cached_db`)**:
   - Reads sessions from an in-memory cache (like Redis). Only falls back to the database on a cache miss. Writes update both cache and database.
3. **Pure Cache / Redis (`django.contrib.sessions.backends.cache`)**:
   - Stores sessions entirely in Redis. Sub-millisecond reads. Ideal for high-traffic production environments.

---

### Essential Production Cookie Flags

In an interview, mention these three cookie security attributes:

```python
# settings.py
SESSION_COOKIE_HTTPONLY = True  # Default: True
SESSION_COOKIE_SECURE = True    # Production requirement
SESSION_COOKIE_SAMESITE = 'Lax' # Default: 'Lax'
```

1. **`HttpOnly`**: Prevents client-side JavaScript (`document.cookie`) from accessing the session cookie. If your site has an XSS vulnerability, the attacker's script cannot steal the session cookie!
2. **`Secure`**: Instructs the browser to send the cookie **only over encrypted HTTPS connections**. Prevents man-in-the-middle sniffing on public Wi-Fi.
3. **`SameSite=Lax`**: Restricts the browser from sending the cookie on cross-site requests, providing strong automatic protection against Cross-Site Request Forgery (CSRF).

---

# 3. Password Hashing & Security

## Interview Question

**How does Django store user passwords securely, and what happens when the hashing algorithm is upgraded?**

---

### Answer

Django **never** stores plaintext passwords.

When a user registers or changes their password, Django calls `user.set_password('raw_password')`.

Under the hood, Django hashes the password using **PBKDF2 with a SHA-256 digest** (or Argon2 if installed) combined with a cryptographically secure random **salt** and thousands of hashing iterations:

```
String stored in database auth_user.password:
pbkdf2_sha256$720000$randomSaltString$computedHashString...
│              │      │                │
Algorithm      Work   Unique Salt      Hash Output
               Factor
```

### Why Salt?

A unique salt prevents **Rainbow Table attacks**. Even if two users choose the identical password `password123`, their stored password hashes in the database look completely different because their random salts are unique.

---

### Seamless Password Upgrades

Django's hashing architecture handles algorithm upgrades automatically.

Suppose Django increases the PBKDF2 iteration count from 600,000 to 720,000 in a new release.

When a user logs in:
1. Django checks the algorithm and iteration count stored in the password string prefix.
2. Django verifies the password against the old iteration count.
3. If the password matches, Django notices that the password was hashed using an older configuration.
4. Django **automatically re-hashes the password with the new, stronger settings and saves it to the database** transparently during that login!

---

# 4. Cross-Site Request Forgery (CSRF) Protection

## Interview Question

**What is a CSRF attack, and how does Django's `CsrfViewMiddleware` protect against it?**

---

### The Attack

Imagine a user is logged into their bank at `bank.com`. The bank uses cookie-based session authentication.

While still logged in, the user visits a malicious website: `evil.com`.

`evil.com` contains a hidden HTML form that automatically submits on page load:

```html
<!-- Hosted on evil.com -->
<form action="https://bank.com/transfer/" method="POST" id="hack">
    <input type="hidden" name="recipient" value="attacker">
    <input type="hidden" name="amount" value="10000">
</form>
<script>document.getElementById('hack').submit();</script>
```

When the browser sends the `POST` request to `bank.com`, the browser automatically attaches the user's `sessionid` cookie for `bank.com`.

`bank.com` receives the request, sees a valid session cookie, assumes the user initiated it, and transfers \$10,000 to the attacker!

This is **Cross-Site Request Forgery (CSRF)**.

---

### How Django Prevents CSRF

Django uses the **Synchronizer Token Pattern**:

1. When a user requests a page containing a form, Django generates a secret token and sends it two ways:
   - As a cookie (`csrftoken`).
   - Injected into the form via `{% csrf_token %}` as a hidden input field:
     ```html
     <input type="hidden" name="csrfmiddlewaretoken" value="K3b7...d9X">
     ```
2. When the user submits the form, `CsrfViewMiddleware` intercepts the `POST` request.
3. Django compares the token submitted in the form body (or `X-CSRFToken` HTTP header) with the secret token in the cookie.
4. Because the Same-Origin Policy forbids `evil.com` from reading cookies belonging to `bank.com`, `evil.com` cannot know the secret token.
5. If the submitted token is missing or does not match, Django halts immediately with an **HTTP 403 Forbidden** error.

```
Safe Methods: GET, HEAD, OPTIONS (Read-only, should never mutate data)
Protected Methods: POST, PUT, PATCH, DELETE (Mutate state, require CSRF token)
```

> **Interview Follow-up**: *Why do pure token-based APIs (like mobile apps using JWTs in the `Authorization: Bearer <token>` header) not need CSRF protection?*
>
> Because mobile apps and SPA clients don't use ambient browser cookie transmission. The client must explicitly attach the token in an HTTP header using JavaScript. A malicious third-party site cannot force the browser to inject custom headers on cross-site form submissions.

---

# 5. SQL Injection & XSS Defenses

## Interview Question

**How does Django protect against SQL Injection, and where can developers accidentally introduce a vulnerability?**

---

### Answer

The Django ORM prevents SQL injection by default through **Query Parameterization**.

When you write:

```python
user_input = "alice' OR '1'='1"
User.objects.filter(username=user_input)
```

Django does **not** construct SQL through string formatting:

```sql
-- DANGEROUS / VULNERABLE:
SELECT * FROM auth_user WHERE username = 'alice' OR '1'='1';
```

Instead, Django sends parameterized placeholders to the database driver:

```sql
-- SAFE / PARAMETERIZED:
SELECT * FROM auth_user WHERE username = %s;
-- Parameters: ("alice' OR '1'='1",)
```

The database treats `user_input` purely as a literal string value, not as executable SQL syntax. The attack fails completely.

---

### Where developers introduce SQL Injection in Django

Vulnerabilities occur when developers write **raw SQL with Python string formatting**:

```python
# CRITICAL VULNERABILITY: Python f-strings in raw queries
user_id = request.GET.get('id')
users = User.objects.raw(f"SELECT * FROM auth_user WHERE id = {user_id}")
```

If an attacker passes `id=1 OR 1=1`, they dump the entire user database.

### The Secure Way: Always pass parameters as a tuple

```python
# SECURE: Django passes parameters safely to the driver
users = User.objects.raw("SELECT * FROM auth_user WHERE id = %s", [user_id])
```

The same rule applies when using `connection.cursor()`:

```python
from django.db import connection

with connection.cursor() as cursor:
    cursor.execute("SELECT * FROM auth_user WHERE email = %s", [email])
    row = cursor.fetchone()
```
