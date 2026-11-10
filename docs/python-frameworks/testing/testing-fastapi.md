# Testing FastAPI with Pytest & HTTPX

> This guide explains how to test FastAPI applications using `TestClient` and `httpx.AsyncClient`, how to isolate database sessions using `dependency_overrides`, and how to test async endpoints without mocks.

---

# 1. FastAPI `TestClient` (Synchronous Testing)

## Interview Question

**What is FastAPI's `TestClient`, how does it work under the hood, and how do you write your first API test?**

---

### Answer

FastAPI includes a built-in `TestClient` (inherited from Starlette, powered by `httpx`):

```python
# tests/test_items.py
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_read_root():
    response = client.get("/")
    assert response.status_code == 200
    assert response.json() == {"message": "Hello World"}

def test_create_item_validation_error():
    # Sending invalid payload (price is negative)
    response = client.post("/items/", json={"name": "Book", "price": -10})
    
    # FastAPI automatically validates via Pydantic and returns 422!
    assert response.status_code == 422
    assert response.json()["detail"][0]["loc"] == ["body", "price"]
```

### How `TestClient` works:
`TestClient` does **not** open a network socket or start a live Uvicorn server on a port. It passes requests directly in-memory to the ASGI application callable, making tests run in milliseconds.

---

# 2. Async Testing with `pytest-asyncio` & `AsyncClient`

## Interview Question

**When should you use `httpx.AsyncClient` instead of `TestClient`? How do you test native async endpoints?**

---

### Answer

Use `TestClient` for 90% of standard API tests.

However, if your tests need to:
- Use `await` directly inside the test function
- Test WebSockets or streaming responses
- Use an async database session (like async SQLAlchemy with `asyncpg`)

You must use **`httpx.AsyncClient`** combined with **`pytest-asyncio`**:

```bash
pip install pytest-asyncio httpx
```

```python
# tests/test_async_api.py
import pytest
from httpx import AsyncClient, ASGITransport
from main import app

@pytest.mark.asyncio
async def test_async_endpoint():
    # ASGITransport passes requests in-memory to the async ASGI app
    transport = ASGITransport(app=app)
    
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        response = await client.get("/async-data/")
        
        assert response.status_code == 200
        assert response.json()["status"] == "ok"
```

---

# 3. Database Isolation via `dependency_overrides`

## Interview Question

**Why does FastAPI not need complex mocking for database sessions? How does `app.dependency_overrides` work?**

---

### Answer

In traditional frameworks, pointing tests to an in-memory database requires monkey-patching database engine variables.

In FastAPI, database sessions are injected via dependencies (`db: Session = Depends(get_db)`).

You can swap the production database with an in-memory SQLite database using **`app.dependency_overrides`**:

```python
# tests/conftest.py
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from main import app
from core.database import Base, get_db

# 1. Create an isolated in-memory SQLite test database
SQLALCHEMY_TEST_DATABASE_URL = "sqlite:///:memory:"
engine = create_engine(SQLALCHEMY_TEST_DATABASE_URL, connect_args={"check_same_thread": False})
TestingSessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

@pytest.fixture(scope="function")
def db_session():
    # Create all tables in memory
    Base.metadata.create_all(bind=engine)
    session = TestingSessionLocal()
    
    try:
        yield session
    finally:
        session.close()
        Base.metadata.drop_all(bind=engine)  # Clean up after test!

@pytest.fixture(scope="function", autouse=True)
def override_db(db_session):
    # 2. Swap get_db with our test session!
    def _override_get_db():
        try:
            yield db_session
        finally:
            pass

    app.dependency_overrides[get_db] = _override_get_db
    yield
    app.dependency_overrides.clear()  # Clean up after test!
```

Now, every test automatically executes against a pristine, freshly created in-memory SQLite database that is completely destroyed after each test!

---

# 4. Mocking Authentication in FastAPI Tests

## Interview Question

**How do you test authenticated FastAPI routes without generating real JWTs?**

---

### Answer

Override the `get_current_user` dependency:

```python
from main import app
from core.security import get_current_user
from models import User

def test_admin_dashboard(client):
    # Create a mock authenticated user
    fake_admin = User(id=1, username="admin_alice", is_admin=True)

    # Bypass JWT authentication completely!
    app.dependency_overrides[get_current_user] = lambda: fake_admin

    response = client.get("/admin/dashboard/")
    assert response.status_code == 200
    assert "admin_alice" in response.text

    # Clear override
    app.dependency_overrides.clear()
```
