# Testing Flask Applications with Pytest

> This guide explains how to test Flask applications using the Application Factory pattern, manage application and request contexts in tests, and test database operations with Flask-SQLAlchemy.

---

# 1. Flask Test Client & Application Factory

## Interview Question

**How do you configure Pytest fixtures for a Flask application that uses the Application Factory pattern?**

---

### Answer

When using the Application Factory pattern (`create_app()`), we define reusable fixtures in `conftest.py`:

```python
# tests/conftest.py
import pytest
from my_app import create_app, db

@pytest.fixture(scope="session")
def app():
    # 1. Create app with testing configuration (in-memory SQLite)
    app = create_app("config.TestingConfig")
    
    with app.app_context():
        yield app

@pytest.fixture(scope="function")
def client(app):
    # 2. Return Flask's native test client
    return app.test_client()

@pytest.fixture(scope="function")
def db_session(app):
    # 3. Create fresh database tables for every test
    with app.app_context():
        db.create_all()
        yield db.session
        db.session.remove()
        db.drop_all()  # Clean up after test!
```

---

# 2. Writing Endpoint Tests

## Interview Question

**How do you test GET and POST endpoints in Flask, and how do you assert JSON responses?**

---

### Answer

```python
# tests/test_orders.py
def test_get_orders_empty(client, db_session):
    response = client.get("/api/v1/orders")
    
    assert response.status_code == 200
    assert response.get_json() == []

def test_create_order_success(client, db_session):
    payload = {"item_id": 42, "quantity": 2}
    
    response = client.post(
        "/api/v1/orders",
        json=payload,
        headers={"Content-Type": "application/json"}
    )
    
    assert response.status_code == 201
    data = response.get_json()
    assert data["item_id"] == 42
    assert data["status"] == "created"
```

---

# 3. Managing Contexts in Flask Tests

## Interview Question

**What is the difference between `app.app_context()` and `app.test_request_context()` when writing tests?**

---

### Answer

Flask requires contexts to access global proxies:

```python
# 1. Application Context: Needed for DB and current_app
def test_create_user_model(app, db_session):
    with app.app_context():
        user = User(username="alice", email="alice@test.com")
        db_session.add(user)
        db_session.commit()
        assert User.query.count() == 1

# 2. Request Context: Needed to test functions that inspect request, g, or session
def test_request_headers_utility(app):
    with app.test_request_context("/profile", headers={"User-Agent": "MobileApp"}):
        # request is active inside this block!
        from flask import request
        assert request.headers.get("User-Agent") == "MobileApp"
```
