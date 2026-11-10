# Advanced Testing Concepts & Senior Interview Scenarios

> This guide explores advanced testing patterns: the testing pyramid, when NOT to mock, test isolation, race-condition testing, optimizing slow test suites, and essential interview scenarios.

---

# 1. The Testing Pyramid: Unit vs Integration vs E2E

## Interview Question

**Explain the Testing Pyramid. What percentage of your test suite should be unit tests versus integration tests?**

---

### Answer

The Testing Pyramid is an architectural guideline for balancing test speed, cost, and confidence:

```
          / \
         / E2E \        ──► End-to-End Tests (10%)
        /───────\           (Playwright / Selenium / Full environment)
       / Integr- \      ──► Integration Tests (30%)
      /   ation   \         (Real database, API endpoints, Redis)
     /─────────────\
    /     Unit      \   ──► Unit Tests (60%)
   /                 \      (Fast, isolated functions, business logic)
```

1. **Unit Tests (Fastest & Cheapest)**:
   - Test a single function, model method, or calculation in complete isolation.
   - Run in-memory in microseconds. Zero database, zero network.
2. **Integration Tests (Medium Speed & Realism)**:
   - Test multiple components working together (e.g. an API endpoint querying PostgreSQL and serializing JSON).
   - Catch schema mismatches, serialization errors, and transaction bugs.
3. **End-to-End (E2E) Tests (Slowest & Most Fragile)**:
   - Test the entire system from the browser UI down to the backend.
   - Slow to run, expensive to maintain, and prone to network flakiness.

---

# 2. When NOT to Mock: The Over-Mocking Anti-Pattern

## Interview Question

**Why is over-mocking dangerous in backend test suites? When should you mock, and when should you use real components?**

---

### Answer

Mocking replaces a real object with a fake that simulates its behavior.

### The Disaster of Over-Mocking:
If a developer mocks the database:

```python
# OVER-MOCKED ANTI-PATTERN (Tests nothing real!)
@patch('myapp.models.User.objects.filter')
def test_active_users(mock_filter):
    mock_filter.return_value = [User(username="alice")]
    result = get_active_users()
    assert len(result) == 1
```

This test proves **nothing**. If your SQL query has a syntax error or references a non-existent column, this test still passes!

You are testing the mock, not your software.

---

### The Production Rule:

```
DO NOT Mock:
• The Database (Use an isolated test database with automatic transaction rollback!)
• Framework Internals (Django ORM, FastAPI routing, Pydantic validation)
• Internal Business Logic

DO Mock:
• External Third-Party APIs (Stripe, Twilio, SendGrid, Salesforce)
• Hardware / Cloud Resources (AWS S3, SQS, SES)
• Non-Deterministic Factors (System time: freeze_gun, random numbers)
```

---

# 3. Optimizing Slow Test Suites

## Interview Question

**Your team's test suite grew to 2,000 tests and now takes 18 minutes to run in CI. How do you optimize it to finish in under 2 minutes?**

---

### Answer

1. **Parallelize with `pytest-xdist`**:
   Run tests concurrently across all CPU cores:
   ```bash
   pytest -n auto  # Spawns 1 worker per CPU core
   ```
2. **Reuse the Database Schema**:
   Use `--reuse-db` so Django doesn't recreate all tables from scratch on every run.
3. **Identify the Slowest Tests**:
   Run Pytest with `--durations=10` to print the 10 slowest tests in the suite. You will usually find 5 tests that sleep or download large files.
4. **Bypass Password Hashing in Unit Tests**:
   In Django, override `PASSWORD_HASHERS = ['django.contrib.auth.hashers.MD5PasswordHasher']` during tests. MD5 takes 0.1ms compared to PBKDF2's 250ms!
5. **Keep Unit Tests Truly In-Memory**:
   Do not decorate pure helper functions with `@pytest.mark.django_db` if they don't actually query tables.
