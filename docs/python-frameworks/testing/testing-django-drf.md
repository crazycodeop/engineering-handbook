# Testing Django & Django REST Framework with Pytest

> This guide explains how to test Django and DRF applications using `pytest-django`, how to test database transactions, views, serializers, and permissions, and how to use Factory Boy for maintainable test data.

---

# 1. Setting Up `pytest-django`

## Interview Question

**How do you configure Pytest for a Django project, and why is `@pytest.mark.django_db` required?**

---

### Answer

Standard Pytest does not know about Django's settings or database engine.

We use **`pytest-django`**, the official Pytest integration plugin for Django:

```bash
pip install pytest-django
```

### Step 1: Configure `pytest.ini`

```ini
# pytest.ini
[pytest]
DJANGO_SETTINGS_MODULE = myproject.settings
python_files = tests.py test_*.py *_tests.py
addopts = --reuse-db
```

`--reuse-db`: Tells Pytest to create the test database schema once and reuse it across test runs, reducing test suite startup time from 15 seconds to 1 second!

---

### The `@pytest.mark.django_db` Marker

By default, Pytest **strictly forbids tests from touching the database**.

If a test queries the database without permission:

```python
# FAILS: pytest_django.exceptions.DatabaseAccessDenied: Database access not allowed
def test_user_count():
    assert User.objects.count() == 0
```

To grant database access, decorate the test with `@pytest.mark.django_db`:

```python
import pytest
from django.contrib.auth import get_user_model

User = get_user_model()

@pytest.mark.django_db
def test_create_user():
    user = User.objects.create_user(username="alice", email="alice@test.com")
    assert user.id is not None
    assert User.objects.count() == 1
```

### How `pytest-django` isolates tests:
Under the hood, `@pytest.mark.django_db` wraps every single test inside an **atomic database transaction**. When the test finishes, Pytest automatically issues a **`ROLLBACK`**, ensuring no test leaves leftover rows in the database.

---

# 2. Testing API Endpoints with DRF's `APIClient`

## Interview Question

**What is the difference between Django's native `client` and DRF's `APIClient`? How do you test authenticated endpoints efficiently?**

---

### Answer

Django's standard test `client` sends form-encoded data by default.

DRF provides **`APIClient`**, which includes first-class support for JSON payloads and authentication helpers:

```python
import pytest
from rest_framework.test import APIClient
from rest_framework import status
from django.contrib.auth import get_user_model

User = get_user_model()

@pytest.fixture
def api_client():
    return APIClient()

@pytest.fixture
def auth_user():
    return User.objects.create_user(username="alice", password="password123")
```

---

### Why you should use `client.force_authenticate()` in tests

When testing an authenticated endpoint:

```python
# SLOW ANTI-PATTERN: Logging in via HTTP POST for every test
def test_get_orders_slow(api_client):
    api_client.post('/api/token/', {'username': 'alice', 'password': 'password123'})
    response = api_client.get('/api/orders/')
```

Hashing passwords with PBKDF2/Argon2 takes 250ms per test.

Instead, use **`force_authenticate()`**:

```python
@pytest.mark.django_db
def test_authenticated_user_can_list_orders(api_client, auth_user):
    # Instant: Injects user into request without hashing passwords!
    api_client.force_authenticate(user=auth_user)

    response = api_client.get('/api/v1/orders/')
    assert response.status_code == status.HTTP_200_OK

@pytest.mark.django_db
def test_unauthenticated_request_is_rejected(api_client):
    response = api_client.get('/api/v1/orders/')
    assert response.status_code == status.HTTP_401_UNAUTHORIZED
```

---

# 3. Generating Test Data with Factory Boy

## Interview Question

**Why does creating model instances manually in test files become unmaintainable? How does Factory Boy solve this?**

---

### The Anti-Pattern: Manual Setup Duplication

If a test needs an `Order`, and an `Order` requires a `User`, a `ShippingAddress`, a `Warehouse`, and an `InventoryItem`:

```python
# Unmaintainable: 25 lines of setup in every single test!
def test_order_total():
    user = User.objects.create(username="u1", email="u1@test.com", ...)
    address = Address.objects.create(user=user, street="...", city="...", ...)
    warehouse = Warehouse.objects.create(name="Central", ...)
    item = Product.objects.create(warehouse=warehouse, name="Widget", price=10, ...)
    order = Order.objects.create(user=user, address=address, ...)
```

If you add a new required column `phone_number` to `User` next week, **all 200 tests across your codebase break immediately**!

---

### The Production Fix: Factory Boy

Factory Boy defines dynamic, reusable factories for your Django models:

```python
# tests/factories.py
import factory
from django.contrib.auth import get_user_model
from orders.models import Order, OrderItem

User = get_user_model()

class UserFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = User

    username = factory.Faker("user_name")
    email = factory.Faker("email")
    is_active = True

class OrderFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Order

    user = factory.SubFactory(UserFactory)  # Automatically creates a User!
    total_amount = 99.95
    status = "pending"
```

Now, generating realistic, fully populated test objects takes **one line**:

```python
@pytest.mark.django_db
def test_order_cancellation():
    # Creates an Order AND an associated User automatically!
    order = OrderFactory.create(status="pending")
    
    order.cancel()
    assert order.status == "canceled"
```

---

# 4. Testing DRF Serializers & Validation

## Interview Question

**How do you test a serializer's validation logic directly without making HTTP requests?**

---

### Answer

Testing serializers in isolation is fast because it requires zero HTTP overhead:

```python
import pytest
from orders.serializers import OrderCreateSerializer

def test_serializer_validation_success():
    payload = {
        "shipping_address": "123 Main St",
        "items": [{"product_id": 1, "quantity": 2}],
    }
    serializer = OrderCreateSerializer(data=payload)
    assert serializer.is_valid() is True
    assert serializer.validated_data["shipping_address"] == "123 Main St"

def test_serializer_rejects_negative_quantity():
    payload = {
        "shipping_address": "123 Main St",
        "items": [{"product_id": 1, "quantity": -5}],
    }
    serializer = OrderCreateSerializer(data=payload)
    
    # 1. Assert invalid
    assert serializer.is_valid() is False
    
    # 2. Assert exact error structure
    assert "items" in serializer.errors
    assert "quantity" in str(serializer.errors["items"])
```

---

# 5. Testing Celery Tasks in Django

## Interview Question

**How do you test Celery tasks in Django unit tests without running Redis or background worker processes?**

---

### Answer

Configure Celery to run in **eager mode** during tests:

```python
# In your test or conftest.py:
@pytest.fixture(autouse=True)
def eager_celery(settings):
    # Executes tasks synchronously in the same thread immediately!
    settings.CELERY_TASK_ALWAYS_EAGER = True
    settings.CELERY_TASK_EAGER_PROPAGATES = True
```

When `CELERY_TASK_ALWAYS_EAGER = True`, calling `my_task.delay(42)` does not touch Redis. It executes the Python function synchronously and returns an `EagerResult`, allowing you to assert database updates directly!
