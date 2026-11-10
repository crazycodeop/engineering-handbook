# Pytest Fixtures, Parametrization & Mocking

> This guide explores Pytest's most powerful features: dependency injection via fixtures, fixture scopes, `conftest.py`, data-driven testing with `parametrize`, monkeypatching, and mocking external services.

---

# 1. Pytest Fixtures Explained

## Interview Question

**What is a fixture in Pytest, how does it differ from traditional `setUp()` / `tearDown()` methods, and why are fixtures so powerful?**

---

### Answer

In traditional testing, if multiple tests need a sample user or database connection, you write:

```python
# Traditional unittest setUp (Rigid and monolithic)
class TestUser(unittest.TestCase):
    def setUp(self):
        self.user = User(name="Alice", role="admin")
        self.db = Database()
        self.s3 = S3Client()
```

The problem is that **every single test in that class gets all three objects**, whether it needs them or not. Furthermore, sharing setup logic across multiple test files requires deep, messy class inheritance hierarchies.

### Pytest Fixtures: Composable Dependency Injection

A **fixture** in Pytest is simply a Python function decorated with `@pytest.fixture` that prepares a resource, provides it to the test, and cleans it up afterward:

```python
import pytest

@pytest.fixture
def sample_user():
    return {"id": 1, "username": "alice", "is_admin": True}

# Tests declare what they need as function arguments!
def test_user_is_admin(sample_user):
    assert sample_user["is_admin"] is True

def test_user_has_username(sample_user):
    assert sample_user["username"] == "alice"
```

Pytest sees that `test_user_is_admin` takes an argument named `sample_user`, finds the matching fixture, executes it, and passes the return value into the test function.

---

# 2. Fixture Scopes & Lifecycle

## Interview Question

**What are the different fixture scopes in Pytest, and why is setting the wrong scope a common source of test pollution?**

---

### Answer

By default, every fixture runs once per test function. But creating expensive resources (like spinning up a Docker container or loading a machine learning model) for every single test is too slow.

Pytest provides five scopes:

```
Scope           Lifecycle                                    Best For
─────           ─────────                                    ────────
function        Created and destroyed for EVERY test         Clean state, database transactions (Default)
class           Created once per test class                  Grouping read-only tests on shared data
module          Created once per Python test file            File-level setup (e.g. parsing a large CSV)
package         Created once per test package directory      Package-level integration testing
session         Created ONCE for the entire test run         Starting test DB containers, Redis, or API clients
```

```python
# Session-scoped fixture: runs ONCE across the entire test suite
@pytest.fixture(scope="session")
def redis_client():
    client = redis.Redis(host="localhost", port=6379)
    yield client
    client.close()  # Cleans up when all tests finish!
```

### The Danger of Test Pollution

If you create a mutable object (like a dictionary or database table) inside a `session` or `module` scoped fixture, and Test A mutates it (`sample_data.pop('name')`), Test B will fail unexpectedly!

**Golden Rule**: If a test modifies state, use **`scope="function"`** to ensure complete isolation.

---

# 3. Setup and Teardown with `yield`

## Interview Question

**How do you write setup and teardown logic in a Pytest fixture? What happens if a test fails?**

---

### Answer

Use Python's `yield` keyword:

```python
@pytest.fixture
def temp_database_table():
    # 1. SETUP (Runs BEFORE the test)
    table = create_temporary_table("test_orders")
    
    # 2. HANDOFF (Injects resource into test)
    yield table
    
    # 3. TEARDOWN (Runs AFTER the test finishes)
    drop_temporary_table("test_orders")
```

Even if the test function raises an `AssertionError` or unhandled exception, Pytest **guarantees that the teardown code after `yield` will execute**, preventing leftover tables or orphaned connections.

---

# 4. Sharing Fixtures Globally: `conftest.py`

## Interview Question

**What is `conftest.py`, and why should you never manually import fixtures from it?**

---

### Answer

`conftest.py` is a special file recognized by Pytest as a **local plugin**.

Any fixture declared in a `conftest.py` file is **automatically available to all test files in that directory and all child directories** without importing:

```
tests/
├── conftest.py             → Global fixtures (database_engine, api_client)
├── unit/
│   └── test_models.py      → Automatically has access to conftest.py fixtures!
└── integration/
    ├── conftest.py         → Extra integration fixtures (auth_headers)
    └── test_payments.py
```

### Why you should NEVER `import conftest`:
Pytest discovers `conftest.py` automatically. If you write `from conftest import my_fixture`, Python re-imports the module as a separate instance, breaking Pytest's internal dependency resolution and causing duplicate registration errors.

---

# 5. Data-Driven Testing: `@pytest.mark.parametrize`

## Interview Question

**How do you test multiple inputs and expected outputs without writing 10 repetitive test functions?**

---

### Answer

Use `@pytest.mark.parametrize`:

```python
import pytest

def is_valid_email(email: str) -> bool:
    return "@" in email and "." in email.split("@")[-1]

@pytest.mark.parametrize("email,expected", [
    ("alice@company.com", True),
    ("bob.smith@domain.org", True),
    ("invalid-email", False),
    ("missing-at.com", False),
    ("@no-username.com", False),
])
def test_email_validation(email, expected):
    assert is_valid_email(email) is expected
```

When you run `pytest -v`, Pytest executes **5 distinct, independent tests**:

```
test_validation.py::test_email_validation[alice@company.com-True] PASSED
test_validation.py::test_email_validation[bob.smith@domain.org-True] PASSED
test_validation.py::test_email_validation[invalid-email-False] PASSED
test_validation.py::test_email_validation[missing-at.com-False] PASSED
test_validation.py::test_email_validation[@no-username.com-False] PASSED
```

If one case fails, the other 4 still run and pass.

---

# 6. Testing Exceptions with `pytest.raises`

## Interview Question

**How do you assert that a function raises a specific exception, and how do you verify the exception message?**

---

### Answer

Use the `pytest.raises` context manager:

```python
import pytest

def withdraw_balance(account_balance, amount):
    if amount <= 0:
        raise ValueError("Withdrawal amount must be strictly positive")
    if amount > account_balance:
        raise InsufficientFundsError("Cannot withdraw more than current balance")
    return account_balance - amount

def test_negative_withdrawal_raises_error():
    # 1. Verify specific exception type is raised
    with pytest.raises(ValueError) as exc_info:
        withdraw_balance(account_balance=100, amount=-50)

    # 2. Verify the exact exception error message
    assert "strictly positive" in str(exc_info.value)
```

If the function does *not* raise `ValueError`, the test fails immediately with: `Failed: DID NOT RAISE <class 'ValueError'>`.

---

# 7. Mocking vs Monkeypatching

## Interview Question

**What is the difference between `monkeypatch` and `unittest.mock.patch`? When should you use which?**

---

### Answer

```
Tool                         Best For
────                         ────────
monkeypatch (Built-in)       Setting environment variables, altering dictionaries, patching simple attributes
unittest.mock (patch, Mock)  Verifying call counts, inspecting arguments, replacing complex class methods
```

---

### 1. Using `monkeypatch` for Environment Variables

```python
def test_stripe_api_key_loading(monkeypatch):
    # Safely injects test environment variable; automatically restored after test!
    monkeypatch.setenv("STRIPE_SECRET_KEY", "sk_test_fake_12345")
    
    config = load_payment_config()
    assert config.stripe_key == "sk_test_fake_12345"
```

---

### 2. Using `unittest.mock.patch` for External Services

Suppose your checkout function calls an external credit card gateway:

```python
from unittest.mock import patch

def test_checkout_calls_payment_gateway():
    # Intercept payment_gateway.charge
    with patch("myapp.services.payment_gateway.charge") as mock_charge:
        # Configure return value of the mock
        mock_charge.return_value = {"status": "success", "transaction_id": "tx_999"}

        result = process_checkout(user_id=1, amount=50.0)

        # 1. Assert result is correct
        assert result["status"] == "success"
        
        # 2. Verify that the external payment gateway was actually invoked!
        mock_charge.assert_called_once_with(user_id=1, amount=50.0)
```

### `Mock` vs `MagicMock`:
- **`Mock`**: A generic dummy object that tracks calls, returns values, and records arguments.
- **`MagicMock`**: A subclass of `Mock` that automatically implements Python magic/dunder methods (like `__len__`, `__iter__`, `__enter__`, `__exit__`). If you are mocking a context manager (`with service:`) or an iterable, use `MagicMock`.
