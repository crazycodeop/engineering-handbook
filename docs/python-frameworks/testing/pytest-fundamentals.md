# Pytest Fundamentals & Test Architecture

> This guide introduces automated testing in Python, explains why Pytest replaced Python's standard `unittest`, explores test discovery, and breaks down the core concepts backend developers need for interviews.

---

# 1. Why Automated Testing Matters

## Interview Question

**Why do engineering teams invest so heavily in automated testing? What is the difference between manual testing and automated testing?**

---

### Answer

When you build a new API endpoint locally, it's easy to open Postman or your browser, send a test payload, and verify that it returns `200 OK`.

That is **manual testing**.

Manual testing works when your project has 5 endpoints.

Now imagine your application has grown to:
- 150 API endpoints
- 40 database models
- 12 background Celery tasks
- 8 third-party integrations (Stripe, Twilio, Salesforce)

If a developer changes one line of code in the `User` model, **how do you know it didn't silently break checkout, billing, password reset, or inventory?**

You cannot manually click through 150 endpoints in Postman every time you make a commit.

```
Without Automated Tests:
Change code ──► Deploy to production ──► Cross fingers ──► Customer reports 500 error at 2 AM

With Automated Tests:
Change code ──► Run 800 tests in 10 seconds ──► CI catches regression ──► Fix before merge
```

Automated tests provide **confidence to refactor**. They act as executable documentation that proves the system behaves as intended.

---

# 2. Pytest vs Python's Built-in `unittest`

## Interview Question

**Why has Pytest become the de facto standard for testing in the Python ecosystem over the standard library's `unittest`?**

---

### Comparison: `unittest` vs `pytest`

`unittest` was ported from Java's JUnit in Python 2.1. It forces Java-style object-oriented boilerplate onto Python code:

```python
# Traditional unittest (Verbose, Java-style)
import unittest

class TestMathOperations(unittest.TestCase):
    def setUp(self):
        self.value = 10

    def test_addition(self):
        self.assertEqual(self.value + 5, 15)
        self.assertTrue(self.value > 0)
        self.assertIn(self.value, [10, 20, 30])
```

Here is the exact same test written in **Pytest**:

```python
# Modern Pytest (Clean, Idiomatic Python)
def test_addition():
    value = 10
    assert value + 5 == 15
    assert value > 0
    assert value in [10, 20, 30]
```

---

### Key Advantages of Pytest:

| Feature | `unittest` (Standard Library) | `pytest` (Industry Standard) |
|---|---|---|
| **Class Boilerplate** | Mandatory (`class TestFoo(TestCase):`) | Optional (Plain Python functions) |
| **Assertions** | Dozens of methods (`assertEqual`, `assertIsNone`) | **Single standard `assert` keyword** |
| **Setup & Teardown** | Rigid `setUp()` / `tearDown()` methods | **Composable, scoped Fixtures** (`@pytest.fixture`) |
| **Parametrization** | Clunky (requires subtests or 3rd-party) | Native `@pytest.mark.parametrize` |
| **Failure Output** | Basic message | Detailed AST-rewritten diff (shows intermediate values) |
| **Plugins** | None | 1,000+ plugins (`pytest-django`, `pytest-asyncio`, `pytest-cov`, `pytest-xdist`) |

---

### The Magic of Pytest Assertions: AST Rewriting

In standard Python, writing `assert a == b` just raises an empty `AssertionError` if it fails.

How does Pytest show rich failure details like this?

```
    def test_user_email():
        user = {"name": "Alice", "email": "alice@company.com"}
>       assert user["email"] == "alice@wrong.com"
E       AssertionError: assert 'alice@company.com' == 'alice@wrong.com'
E         - alice@wrong.com
E         + alice@company.com
```

Pytest uses **Abstract Syntax Tree (AST) Rewriting**.

Before executing your test files, Pytest intercepts the Python bytecode compilation and replaces standard `assert` statements with inspection routines that capture variable values on both sides of the comparison. You get crystal-clear failure reports without ever memorizing 30 different `assertSomething()` methods.

---

# 3. Test Discovery & Naming Conventions

## Interview Question

**How does Pytest discover tests in a repository? What naming conventions must you follow?**

---

### Answer

When you type `pytest` in your terminal, Pytest searches your current directory and all subdirectories recursively.

It discovers tests according to standard naming rules:

```
Test Directories:       tests/ or any directory not ignored
Test Files:             test_*.py or *_test.py
Test Functions:         def test_*(): (Must start with test_)
Test Classes (Optional):class Test*: (Must start with Test, no __init__ method)
```

---

### Standard Production Repository Structure

```
my_backend/
├── app/
│   ├── models.py
│   └── views.py
├── tests/
│   ├── conftest.py             → Shared fixtures across all tests
│   ├── unit/                   → Fast, isolated unit tests (No DB)
│   │   ├── test_models.py
│   │   └── test_serializers.py
│   └── integration/            → Slower tests (Database, API endpoints)
│       ├── test_auth_api.py
│       └── test_orders_api.py
├── pyproject.toml / pytest.ini → Pytest configuration
```

---

# 4. Running Tests from the Command Line

## Interview Question

**What are the most useful command-line flags when running Pytest in daily development and CI pipelines?**

---

### Essential CLI Flags

```bash
# 1. Run all tests with verbose output (shows each test name)
pytest -v

# 2. Stop immediately on the FIRST failure (don't waste time running remaining 500 tests)
pytest -x

# 3. Re-run ONLY the tests that failed in the last run
pytest --lf   # (--last-failed)

# 4. Run tests matching a keyword or substring
pytest -k "order_checkout"

# 5. Print standard output / print statements to console (disable stdout capture)
pytest -s

# 6. Show concise traceback (hides internal framework stack frames)
pytest --tb=short

# 7. Run tests in parallel across 4 CPU cores (requires pytest-xdist)
pytest -n 4
```

---

# 5. Configuring Pytest (`pytest.ini` / `pyproject.toml`)

Instead of passing command-line flags every time, configure Pytest in `pyproject.toml` or `pytest.ini`:

```ini
# pytest.ini
[pytest]
minversion = 8.0
testpaths = tests
python_files = test_*.py
addopts = 
    -v
    --strict-markers
    --tb=short
markers =
    slow: marks tests as slow (deselect with '-m "not slow"')
    integration: marks integration tests requiring external services
```

With `addopts = -v --tb=short`, running `pytest` automatically applies those flags every time.
