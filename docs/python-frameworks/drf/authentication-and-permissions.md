# DRF Authentication & Permissions

> This guide explains how Django REST Framework authenticates incoming requests, enforces view-level and object-level permissions, and manages JWTs in production.

---

# 1. Authentication vs Authorization

## Interview Question

**What is the difference between Authentication and Authorization, and in what order does DRF execute them?**

---

### Answer

Although people often confuse them, they answer two fundamentally different questions:

```
Authentication:  "Who are you?"
                  │
                  ▼
              Identity (request.user)
                  │
                  ▼
Authorization:   "Now that I know who you are,
                  are you allowed to perform this action?"
```

```
Incoming Request
       │
       ▼
1. Authentication (`authentication_classes`)
       │ ──► Inspects headers, cookies, or tokens
       │ ──► Sets `request.user` and `request.auth`
       ▼
2. Permissions (`permission_classes`)
       │ ──► Inspects `request.user` against required roles
       │ ──► Calls `has_permission(request, view)`
       ▼
3. View Logic (`get_object()` calls `has_object_permission()`)
```

---

### HTTP 401 Unauthorized vs HTTP 403 Forbidden

Interviewers frequently test candidates on this distinction:

- **HTTP 401 Unauthorized**: You have not proven who you are (e.g. no auth token was provided, or the token was expired/invalid). The client must authenticate.
- **HTTP 403 Forbidden**: We know who you are, but you do not have permission to access this resource (e.g. a regular customer trying to access an admin billing endpoint). Re-authenticating with the same credentials will not help.

---

# 2. Built-in Authentication Schemes

## Interview Question

**Compare `SessionAuthentication`, `TokenAuthentication`, and `JWTAuthentication`. What are the production trade-offs of each?**

---

### Comparison Matrix

| Feature | `SessionAuthentication` | `TokenAuthentication` | `JWTAuthentication` |
|---|---|---|---|
| **Mechanism** | Server-side session table + cookie | Static token in `authtoken_token` table | Signed cryptographic payload in header |
| **Header / Cookie** | `Cookie: sessionid=...` | `Authorization: Token <token>` | `Authorization: Bearer <token>` |
| **CSRF Protection** | Required (Enforced by DRF) | Not required | Not required |
| **DB Hits per Request** | 1 DB read per request (or Redis) | 1 DB read per request | **0 DB hits** (Stateless cryptographic verification) |
| **Expiration** | Handled by server | None (Static forever by default!) | Handled by token claims (`exp`) |
| **Best For** | Same-domain Single Page Apps | Simple internal scripts / CLI tools | Mobile apps & distributed microservices |

---

### Token Authentication Pitfalls in Production

DRF includes `rest_framework.authentication.TokenAuthentication`.

While easy to set up, it has two major flaws in production:
1. **Zero Expiration**: Tokens never expire unless you write custom scheduled cleanup jobs. If a token leaks, it is compromised forever.
2. **Database Bottleneck**: Every single API request executes `SELECT * FROM authtoken_token WHERE key = %s` to look up the user. At 2,000 requests per second, your database is flooded with token lookups.

---

# 3. JSON Web Tokens (JWT) & Token Rotation

## Interview Question

**How does JWT authentication work with `SimpleJWT`, and how do you solve the challenge of revoking a stateless JWT?**

---

### Answer

A JSON Web Token consists of three parts separated by dots: `Header.Payload.Signature`

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjo0MiwiZXhwIjoxNzAwMDAwMDAwfQ.signatureString...
│                                    │                                           │
Header (Algorithm)                   Payload (Claims: user_id, exp)               Cryptographic Signature
```

When a user logs in, the server generates a signed JWT using a secret key and returns it.

On subsequent requests, the client attaches:
```
Authorization: Bearer <token>
```

The server verifies the signature mathematically. It does **not** query the database. It trusts the claims inside the payload because tampering with the payload invalidates the signature.

---

### Access Tokens vs Refresh Tokens

Because JWTs are stateless, you cannot "delete" a token from the server once issued.

To mitigate security risks, `djangorestframework-simplejwt` uses two tokens:

1. **Access Token (Short-lived, e.g. 15 minutes)**:
   - Used for all standard API calls.
   - If intercepted by an attacker, the exposure window is only 15 minutes.
2. **Refresh Token (Long-lived, e.g. 7 days)**:
   - Used only to hit `POST /api/token/refresh/` to obtain a fresh access token.
   - Stored securely (e.g. in an `HttpOnly` cookie or secure mobile keychain).

### Token Rotation:
When the client uses a refresh token to get a new access token, SimpleJWT can also invalidate the old refresh token and issue a brand-new refresh token.

---

### The Revocation Challenge: How to Log Out or Ban a User

If JWTs are stateless, how do you immediately revoke access if a user clicks "Log out" or an admin bans a compromised account?

### Solution 1: Token Blacklisting
When a user logs out, the refresh token's unique ID (`jti`) is saved to a blacklist table (or fast Redis set). When `/token/refresh/` is called, SimpleJWT checks if the `jti` is blacklisted.

### Solution 2: User Token Version / Epoch
Add a `token_version = IntegerField(default=1)` column to your `User` model. Include `token_version` in the JWT payload. If an admin revokes all user sessions or the user changes their password, increment `token_version = 2`. Any token with `version=1` is immediately rejected.

---

# 4. View-Level vs Object-Level Permissions

## Interview Question

**What is the difference between `has_permission()` and `has_object_permission()`? Why does `has_object_permission()` not execute on a list view?**

---

### Answer

DRF permission classes implement two hooks:

```python
from rest_framework.permissions import BasePermission, SAFE_METHODS

class IsOwnerOrReadOnly(BasePermission):
    def has_permission(self, request, view):
        # 1. VIEW-LEVEL CHECK: Runs for EVERY request
        # (GET /posts/, POST /posts/, GET /posts/42/)
        return True

    def has_object_permission(self, request, view, obj):
        # 2. OBJECT-LEVEL CHECK: Runs ONLY when a single object is accessed
        # (GET /posts/42/, PUT /posts/42/, DELETE /posts/42/)
        if request.method in SAFE_METHODS:
            return True  # Read-only requests (GET, HEAD, OPTIONS) allowed for everyone

        # Write requests only allowed if the logged-in user is the author
        return obj.author == request.user
```

---

### Why `has_object_permission()` does NOT execute on List views

In a `GET /posts/` list request, DRF queries a `QuerySet` of 50 posts.

DRF does **not** call `has_object_permission()` 50 times inside the list loop. Doing so would destroy performance.

`has_object_permission()` is only triggered when `view.get_object()` is called (which happens in `retrieve()`, `update()`, and `destroy()`).

---

### How to filter list views by user permission

If you want users to see only their own records in a list view, you must override `get_queryset()`:

```python
class OrderViewSet(ModelViewSet):
    serializer_class = OrderSerializer
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        # Restrict the database query directly to the logged-in user's records!
        return Order.objects.filter(user=self.request.user)
```

Never rely on permissions to filter list querysets. Always filter at the database query level inside `get_queryset()`.
