# FastAPI Fundamentals & Architecture

> This guide introduces FastAPI, its ASGI foundation, Uvicorn server architecture, route definitions, parameter binding, and request lifecycle.

---

# 1. What is FastAPI & Why Choose It?

## Interview Question

**What is FastAPI, who created it, and why has it become so popular compared to traditional frameworks like Django and Flask?**

---

### Answer

FastAPI is a modern, high-performance web framework for building APIs with Python, created by **Sebastián Ramírez** in 2018.

It was designed from the ground up to solve the shortcomings of earlier Python frameworks when building JSON APIs:

```
Django:   Fantastic for full-stack monoliths, but heavy for lightweight microservices.
Flask:    Minimal and flexible, but lacks built-in data validation and native async support.
FastAPI:  Combines extreme performance (Starlette), type safety (Pydantic), and automatic documentation (OpenAPI).
```

### The Three Pillars of FastAPI

FastAPI is built directly on top of two battle-tested libraries:

```
┌────────────────────────────────────────────────────────┐
│                        FastAPI                         │
├───────────────────────────┬────────────────────────────┤
│         Starlette         │          Pydantic          │
│   (Web routing, ASGI,     │   (Data validation,        │
│   WebSockets, middleware) │   serialization, schemas)  │
└───────────────────────────┴────────────────────────────┘
```

1. **Starlette**: Provides the high-speed ASGI routing engine, HTTP request/response handling, WebSocket support, and event loops.
2. **Pydantic**: Provides data parsing, validation, serialization, and JSON Schema generation based entirely on standard Python type annotations.
3. **OpenAPI / JSON Schema**: Because every route parameter and request body uses standard Python type hints, FastAPI automatically generates interactive documentation (`/docs` using Swagger UI and `/redoc` using ReDoc) with zero extra configuration.

---

### FastAPI vs Django vs Flask

| Feature | Django | Flask | FastAPI |
|---|---|---|---|
| **Philosophy** | Batteries-included monolith | Microframework (Minimal core) | API-first, type-driven microframework |
| **Protocol** | WSGI (with partial ASGI) | WSGI | **Native ASGI** |
| **Data Validation** | Django Forms / DRF Serializers | External (Marshmallow / manual) | **Native Pydantic** |
| **Concurrency** | Synchronous (Multi-threaded workers) | Synchronous | **Asynchronous (Event loop + Thread pool)** |
| **API Docs** | Third-party (`drf-spectacular`) | Third-party (`flasgger`) | **Automatic & Built-in** (`/docs`, `/redoc`) |
| **Best For** | Admin dashboards, full SaaS, relational apps | Small utilities, server-side rendered apps | High-concurrency REST APIs, ML model serving |

---

# 2. ASGI Architecture & Uvicorn

## Interview Question

**What is ASGI, and what is Uvicorn's role in running a FastAPI application?**

---

### Answer

WSGI was designed when web applications were synchronous: one request occupied one worker thread until complete.

**ASGI (Asynchronous Server Gateway Interface)** is the modern standard that allows Python applications to handle asynchronous I/O, WebSockets, and long-lived HTTP connections concurrently on an event loop.

### What is Uvicorn?

FastAPI is just the application logic. It does not listen on a network socket or parse raw TCP packets.

**Uvicorn** is a lightning-fast ASGI web server built on `uvloop` (an ultra-fast C-based event loop implementation) and `httptools` (Node.js's HTTP parser ported to C).

```
Client (HTTP/HTTPS)
       │
       ▼
Uvicorn (ASGI Web Server)
       │ Parses HTTP headers and manages TCP connections on an event loop
       ▼
FastAPI Application Callable: async def app(scope, receive, send)
```

To run a FastAPI application locally:

```bash
uvicorn main:app --reload --port 8000
```

- `main`: The Python file (`main.py`).
- `app`: The `FastAPI()` instance created inside `main.py`.
- `--reload`: Automatically restarts the server when code files change (development only).

---

# 3. Routes & Parameter Binding

## Interview Question

**How does FastAPI differentiate between Path parameters, Query parameters, and Request bodies without requiring boilerplate parsing code?**

---

### Answer

FastAPI uses **Python type annotations** to automatically determine where a parameter comes from.

Look at this single endpoint:

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    price: float

@app.put("/items/{item_id}")
async def update_item(item_id: int, item: Item, q: str | None = None, limit: int = 10):
    return {
        "item_id": item_id,
        "item": item,
        "q": q,
        "limit": limit
    }
```

FastAPI inspects the signature and inspects the URL path:

1. **Path Parameter (`item_id: int`)**:
   - Because `{item_id}` appears in the route path (`/items/{item_id}`), FastAPI extracts it from the URL.
   - It validates that `item_id` can be converted to an integer. If someone visits `/items/foo`, FastAPI immediately returns an HTTP 422 Unprocessable Entity error with a clear message: `"value is not a valid integer"`.
2. **Request Body (`item: Item`)**:
   - Because `Item` is a Pydantic `BaseModel` subclass, FastAPI reads the incoming HTTP request body, parses the JSON, validates all fields, and converts it into a typed `Item` instance.
3. **Query Parameters (`q: str | None = None`, `limit: int = 10`)**:
   - Because `q` and `limit` are singular types not present in the URL path, FastAPI treats them as query parameters in the URL query string (e.g. `/items/42?limit=25&q=electronics`).
   - Default values (`None`, `10`) automatically define whether the parameter is optional or required.

---

# 4. Response Models & Data Filtering

## Interview Question

**What is `response_model`, and why is it a critical security feature in FastAPI?**

---

### Answer

In many APIs, database models contain sensitive fields (such as `hashed_password`, internal tracking IDs, or credit card tokens).

`response_model` defines the output contract for an endpoint:

```python
from fastapi import FastAPI
from pydantic import BaseModel, EmailStr

app = FastAPI()

# Input model (Includes sensitive password)
class UserIn(BaseModel):
    username: str
    password: str
    email: EmailStr

# Output model (Excludes sensitive password!)
class UserOut(BaseModel):
    username: str
    email: EmailStr

@app.post("/users/", response_model=UserOut)
async def create_user(user: UserIn):
    # Imagine user is saved to DB and returned as a dict containing hashed_password
    db_user = {
        "username": user.username,
        "email": user.email,
        "hashed_password": "super-secret-hash-12345"
    }
    # FastAPI filters the outgoing dictionary through UserOut!
    return db_user
```

### Why `response_model` is critical:
1. **Security (Prevents Accidental Data Leaks)**: Even though `db_user` contains `hashed_password`, FastAPI filters the output through `UserOut`. The returned JSON **only** contains `username` and `email`. The password hash is never exposed over the network!
2. **Data Serialization**: Converts complex ORM models or datetimes into compliant JSON primitives.
3. **OpenAPI Schema Accuracy**: Generates exact response definitions in the interactive `/docs` UI.

---

# 5. Modularizing Large Applications with `APIRouter`

## Interview Question

**How do you structure large, production-grade FastAPI applications using `APIRouter`?**

---

### Answer

In a real enterprise application, putting all endpoints in a single `main.py` is unmaintainable.

FastAPI provides `APIRouter` (similar to Flask Blueprints or Django Apps) to group related routes into dedicated modules:

```
my_project/
├── main.py               → App initialization & router inclusion
├── core/
│   ├── config.py         → Pydantic Settings
│   └── database.py       → SQLAlchemy session setup
└── routers/
    ├── users.py          → User endpoints
    ├── orders.py         → Order endpoints
    └── products.py       → Product endpoints
```

### 1. Define Routes in a Submodule

```python
# routers/users.py
from fastapi import APIRouter

router = APIRouter(
    prefix="/users",
    tags=["Users"],
    responses={404: {"description": "User not found"}},
)

@router.get("/")
async def list_users():
    return [{"username": "alice"}, {"username": "bob"}]

@router.get("/{user_id}")
async def get_user(user_id: int):
    return {"user_id": user_id}
```

---

### 2. Include Routers in `main.py`

```python
# main.py
from fastapi import FastAPI
from routers import users, orders, products

app = FastAPI(title="E-Commerce API", version="1.0.0")

app.include_router(users.router)
app.include_router(orders.router, prefix="/api/v1")
app.include_router(products.router, prefix="/api/v1")
```

Every router retains its own independent prefixes, tags, and dependencies, keeping the codebase organized as teams scale.
