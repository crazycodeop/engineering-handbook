# FastAPI Authentication & Security

> This guide explains how FastAPI implements OAuth2 with JWT Bearer tokens, role-based access control with scopes, password hashing, CORS configuration, and security middleware.

---

# 1. OAuth2 with Password Flow & JWT Tokens

## Interview Question

**How does FastAPI implement OAuth2 with Bearer tokens, and how does `OAuth2PasswordBearer` integrate with the Swagger `/docs` UI?**

---

### Answer

FastAPI provides security abstractions in `fastapi.security` that conform directly to the OpenAPI specification.

The standard pattern for web and mobile APIs is the **OAuth2 Password flow with Bearer JWT tokens**:

```
Client (Browser / Mobile)                           FastAPI Backend
       │                                                   │
       │ ──► POST /token (username, password) ───────────► │
       │                                                   │ ──► Verifies password hash (bcrypt)
       │                                                   │ ──► Generates signed JWT
       │ ◄── 200 OK {"access_token": "...", "token_type": "bearer"}
       │                                                   │
       │ ──► GET /users/me ──────────────────────────────► │
       │     (Header: Authorization: Bearer <jwt>)         │ ──► OAuth2PasswordBearer extracts token
       │                                                   │ ──► Verifies signature & claims
       │ ◄── 200 OK {"username": "alice"} ──────────────── │
```

---

### Step 1: The `OAuth2PasswordBearer` Scheme

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
import jwt
from datetime import datetime, timedelta, timezone

app = FastAPI()

# Declares where the client can obtain a token (Enables the Authorize button in /docs!)
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

SECRET_KEY = "your-256-bit-secret-key-stored-in-env"
ALGORITHM = "HS256"
```

Notice `tokenUrl="token"`.

When you open `/docs` in your browser, FastAPI sees `oauth2_scheme` and automatically renders a green **"Authorize"** button in Swagger UI. Clicking it opens a modal where you enter a username and password, which Swagger posts to `/token` to fetch and store the token for all subsequent interactive requests!

---

### Step 2: The Login Endpoint (`/token`)

FastAPI provides `OAuth2PasswordRequestForm` to parse form-encoded credentials:

```python
@app.post("/token")
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    user = authenticate_user(form_data.username, form_data.password)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )

    # Generate JWT
    expires_delta = timedelta(minutes=30)
    expire = datetime.now(timezone.utc) + expires_delta
    payload = {"sub": user.username, "exp": expire}
    
    access_token = jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)
    return {"access_token": access_token, "token_type": "bearer"}
```

---

### Step 3: The `get_current_user` Dependency

```python
async def get_current_user(token: str = Depends(oauth2_scheme)):
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise credentials_exception
    except jwt.PyJWTError:
        raise credentials_exception

    user = get_user_from_db(username)
    if user is None:
        raise credentials_exception
    return user
```

---

### Step 4: Protecting Routes

Now, protecting any route requires only adding the dependency:

```python
@app.get("/users/me")
async def read_current_user(current_user: User = Depends(get_current_user)):
    return {"username": current_user.username, "email": current_user.email}
```

If the token is missing, expired, or invalid, FastAPI automatically halts execution and returns an HTTP 401 response with proper `WWW-Authenticate: Bearer` headers.

---

# 2. Scopes & Role-Based Access Control (RBAC)

## Interview Question

**How do OAuth2 Scopes work in FastAPI, and how do you implement fine-grained permissions?**

---

### Answer

Authentication verifies who a user is. **Scopes** restrict what an access token can do (e.g. `read:users`, `write:orders`).

FastAPI provides `Security` and `SecurityScopes` to enforce scopes declaratively:

```python
from fastapi import Security
from fastapi.security import SecurityScopes

async def get_current_user(
    security_scopes: SecurityScopes,
    token: str = Depends(oauth2_scheme)
):
    payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
    token_scopes = payload.get("scopes", [])
    
    # Verify that the token has all required scopes for this specific endpoint
    for scope in security_scopes.scopes:
        if scope not in token_scopes:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Not enough permissions: Missing scope '{scope}'",
            )
            
    return get_user_from_db(payload.get("sub"))
```

In your endpoints, use `Security()` to declare required scopes:

```python
# Accessible by any valid authenticated user
@app.get("/items/")
async def read_items(user: User = Security(get_current_user, scopes=["items:read"])):
    return {"items": ["item1", "item2"]}

# Accessible ONLY if token has the 'items:write' scope
@app.post("/items/")
async def create_item(user: User = Security(get_current_user, scopes=["items:write"])):
    return {"status": "created"}
```

---

# 3. Cross-Origin Resource Sharing (CORS)

## Interview Question

**What is CORS, what causes CORS errors in modern Single Page Apps, and how do you configure `CORSMiddleware` in FastAPI?**

---

### Answer

CORS (Cross-Origin Resource Sharing) is a browser security mechanism that restricts a web page hosted on one origin (e.g. `https://myapp.com`) from making AJAX/Fetch requests to a backend hosted on a different origin (e.g. `https://api.myapp.com`).

When a frontend makes a cross-origin request with custom headers or mutation methods (`POST`, `PUT`, `DELETE`), the browser sends an initial **Preflight `OPTIONS` request** asking:

> *"Is `https://myapp.com` allowed to send a `PUT` request with an `Authorization` header?"*

If the backend does not reply with the proper `Access-Control-Allow-Origin` headers, the browser blocks the request and throws a CORS error in the frontend console.

---

### Configuring `CORSMiddleware` in FastAPI

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# Allowed frontend domains
origins = [
    "https://myapp.com",
    "https://staging.myapp.com",
    "http://localhost:3000",  # Local React / Vue development
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,          # Specific origins allowed
    allow_credentials=True,        # Allow cookies & authorization headers
    allow_methods=["*"],           # Allow GET, POST, PUT, DELETE, etc.
    allow_headers=["*"],           # Allow custom headers
)
```

> **Production Warning**: Never set `allow_origins=["*"]` when `allow_credentials=True`. Browsers strictly reject wildcards on authenticated requests. Always list exact allowed domains.
