# FastAPI Dependency Injection

> This guide explains FastAPI's built-in Dependency Injection (DI) system, how `Depends()` resolves hierarchical dependency graphs, how `yield` manages database lifecycles, and how dependency overrides make testing effortless.

---

# 1. What is Dependency Injection?

## Interview Question

**What is Dependency Injection, and why does FastAPI's implementation eliminate so much boilerplate compared to traditional frameworks?**

---

### Answer

In traditional code, if a function needs an external service (like a database session, an authenticated user, or a third-party API client), the function instantiates that service directly:

```python
# TIGHTLY COUPLED (Difficult to test and maintain)
def get_user_orders(user_id: int):
    db = DatabaseConnection()       # Hardcoded creation
    auth = AuthService()            # Hardcoded creation
    user = auth.verify_token(...)
    return db.query(Order).filter_by(user_id=user.id).all()
```

If you want to write a unit test for `get_user_orders`, you cannot easily swap `DatabaseConnection` for a test database. You have to use messy monkey-patching.

**Dependency Injection (DI)** flips this relationship.

Instead of a function creating what it needs, the function **declares what it needs as parameters**, and an external framework (FastAPI) injects those dependencies automatically at runtime:

```python
from fastapi import FastAPI, Depends

app = FastAPI()

# A simple reusable dependency
def common_parameters(q: str | None = None, skip: int = 0, limit: int = 20):
    return {"q": q, "skip": skip, "limit": limit}

@app.get("/items/")
async def read_items(params: dict = Depends(common_parameters)):
    return params

@app.get("/users/")
async def read_users(params: dict = Depends(common_parameters)):
    return params
```

When a request arrives at `/items/?skip=10&limit=50`:
1. FastAPI inspects `read_items` and sees `Depends(common_parameters)`.
2. FastAPI inspects `common_parameters`, reads `skip=10` and `limit=50` from the query string, validates the types, and invokes `common_parameters`.
3. FastAPI takes the returned dictionary and passes it directly into `read_items` as `params`.

---

# 2. Yield Dependencies: Resource Cleanup & Database Sessions

## Interview Question

**How do `yield` dependencies work in FastAPI, and how do they guarantee database connection cleanup even when exceptions occur?**

---

### Answer

Many dependencies require a two-phase lifecycle:
1. **Setup**: Acquire a resource before the route runs (e.g. open a database session).
2. **Teardown**: Release the resource after the response is sent (e.g. close the database session, return connection to the pool).

In FastAPI, you implement this using the Python `yield` keyword:

```python
from typing import Generator
from sqlalchemy.orm import Session
from core.database import SessionLocal

# Database session dependency
def get_db() -> Generator[Session, None, None]:
    db = SessionLocal()  # 1. SETUP: Open database session
    try:
        yield db         # 2. HANDOFF: Inject db into route handler
    finally:
        db.close()       # 3. TEARDOWN: Runs AFTER response is generated!
```

---

### Execution Flow Under the Hood

```
Incoming HTTP Request
       │
       ▼
Calls get_db() up to yield
       │ ──► Creates SessionLocal()
       ▼
Injects `db` into route: async def get_users(db: Session = Depends(get_db))
       │ ──► Route queries database and returns data
       ▼
FastAPI sends HTTP Response to client
       │
       ▼
Resumes get_db() after yield
       │ ──► Executes `finally: db.close()`
       ▼
Database connection safely returned to pool!
```

### Why the `try ... finally` block is mandatory:

If your route handler raises an unexpected exception (e.g. `ZeroDivisionError` or custom `HTTPException`), the `finally` block is **guaranteed to execute**.

Without `finally`, an unhandled error would leave the database connection dangling in an unclosed state. Over time, this causes **connection pool exhaustion**, bringing down your database!

---

# 3. Hierarchical Dependency Trees & Sub-Dependencies

## Interview Question

**Can dependencies depend on other dependencies in FastAPI? How does FastAPI resolve the dependency graph?**

---

### Answer

Yes. FastAPI dependencies can be chained hierarchically to arbitrary depths.

FastAPI builds a Directed Acyclic Graph (DAG) of all dependencies and resolves them from the leaves up to the root:

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer

app = FastAPI()
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

# Leaf Dependency 1: Extract and verify raw token string
def get_token_header(token: str = Depends(oauth2_scheme)) -> str:
    if not token:
        raise HTTPException(status_code=401, detail="Token missing")
    return token

# Sub-Dependency 2: Depends on Leaf 1 to fetch user from DB
def get_current_user(token: str = Depends(get_token_header), db: Session = Depends(get_db)):
    user = authenticate_user_from_token(token, db)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid credentials")
    return user

# Sub-Dependency 3: Depends on Sub-Dependency 2 to check roles
def get_current_active_admin(user: User = Depends(get_current_user)):
    if not user.is_admin:
        raise HTTPException(status_code=403, detail="Admin privileges required")
    return user

# Route Handler: Receives fully validated admin user
@app.delete("/system/purge")
async def purge_system(admin: User = Depends(get_current_active_admin)):
    return {"message": f"Purged by admin {admin.username}"}
```

```
                        purge_system
                             │
                             ▼
                 get_current_active_admin
                             │
                             ▼
                     get_current_user
                       /          \
                      ▼            ▼
             get_token_header    get_db
```

FastAPI resolves `get_token_header` and `get_db` first, passes their outputs to `get_current_user`, passes that output to `get_current_active_admin`, and finally executes `purge_system`.

---

# 4. Dependency Caching (`use_cache=True`)

## Interview Question

**What is dependency caching in FastAPI, and when would you disable it using `use_cache=False`?**

---

### Answer

By default, `Depends()` has `use_cache=True`.

If multiple dependencies in the same request tree require the same sub-dependency (for example, three different sub-dependencies all ask for `db: Session = Depends(get_db)`):

```
                     Route Handler
                      /         \
                     ▼           ▼
               Sub-Dep A       Sub-Dep B
                     \           /
                      ▼         ▼
                   get_db (Evaluated ONCE!)
```

With `use_cache=True`, FastAPI resolves `get_db()` **once per HTTP request**, caches the resulting session object in memory for the duration of that request, and passes that exact same session instance to both `Sub-Dep A` and `Sub-Dep B`.

---

### When to use `use_cache=False`

You disable caching when a dependency must produce a **fresh, independent value** every single time it is invoked within the same request:

```python
# Generates a unique transaction tracking ID for each operation
def generate_step_id():
    return str(uuid.uuid4())

@app.post("/multi-step")
def multi_step_task(
    step1: str = Depends(generate_step_id, use_cache=False),
    step2: str = Depends(generate_step_id, use_cache=False)
):
    # step1 and step2 are guaranteed to be different UUIDs!
    return {"step1": step1, "step2": step2}
```

---

# 5. Dependency Overrides: Testing Without Mocks

## Interview Question

**Why is `app.dependency_overrides` considered FastAPI's greatest superpower for automated testing?**

---

### Answer

In traditional frameworks, pointing tests to an in-memory SQLite database or mocking a third-party payment gateway requires complex patching (`unittest.mock.patch`).

FastAPI solves this cleanly through `app.dependency_overrides`.

Because all components (database sessions, authentication, external services) are dependencies, you can swap any dependency with a test double in two lines:

```python
# tests/test_api.py
from fastapi.testclient import TestClient
from main import app
from core.database import get_db

# Create an in-memory SQLite session for tests
def override_get_db():
    db = TestingSessionLocal()
    try:
        yield db
    finally:
        db.close()

# SWAP PRODUCTION DB WITH TEST DB
app.dependency_overrides[get_db] = override_get_db

client = TestClient(app)

def test_create_user():
    response = client.post("/users/", json={"username": "alice", "email": "alice@test.com"})
    assert response.status_code == 200

# Cleanup after tests
app.dependency_overrides.clear()
```

### Mocking Current User:
You can bypass authentication in unit tests completely:

```python
app.dependency_overrides[get_current_user] = lambda: User(id=1, username="test_admin", is_admin=True)
```

No monkey-patching. No global state mutation. Testing becomes fast, deterministic, and clean.
