# 🐍 Python Backend Frameworks Handbook

Most framework tutorials teach you syntax.

They show you how to define a route, how to create a model, and how to return a JSON response.

That's fine for getting started.

But senior backend interviews rarely ask you to recite API signatures.

Instead, they ask questions like:

- Why did your team choose Django instead of FastAPI?
- What actually happens under the hood between Nginx, Gunicorn, and your WSGI application?
- Why is an endpoint taking 4 seconds to respond, and how do you find the hidden N+1 query?
- What happens when someone executes blocking synchronous code inside an `async def` route in FastAPI?
- How does Flask manage request contexts using thread-local storage, and why does that break inside background threads?
- When two users try to update the exact same bank balance at the exact same millisecond, how do you prevent lost updates?

This section is written from the perspective of an engineer who builds, debugs, optimizes, and scales real-world Python backend applications.

Every topic is explained in simple everyday English. No academic hand-waving. No unnecessary jargon.

---

## What We Cover

```
Python Backend Frameworks
│
├── 1. Django
│   ├── Architecture & Request-Response Lifecycle
│   ├── Views, URL Routing & Templates
│   ├── Models & Deep ORM Internals (N+1, select_related vs prefetch_related, Transactions)
│   ├── Authentication, Sessions & Security Best Practices
│   └── Advanced Django (Caching, Celery, Signals, Middleware, Production Deployment)
│
├── 2. Django REST Framework (DRF)
│   ├── Fundamentals & Views (APIView, GenericAPIView, ViewSets)
│   ├── Serializers & Validation Pipelines
│   ├── Authentication, Permissions & Object-Level Access
│   ├── Pagination, Filtering, Throttling & Versioning
│   └── Advanced DRF (Custom Actions, Bulk Operations, Queryset Optimization, Testing)
│
├── 3. FastAPI
│   ├── Fundamentals, Routing & ASGI Architecture
│   ├── Pydantic v2 & Data Validation
│   ├── Dependency Injection System (`Depends`, Yield Dependencies)
│   ├── Async Programming, Event Loops & Concurrency Pitfalls
│   ├── Authentication & Security (OAuth2, JWT, Scopes)
│   └── Database Integration (SQLAlchemy Async, Migrations) & Production Deployment
│
├── 4. Flask
│   ├── Microframework Fundamentals & Minimal Philosophy
│   ├── Application & Request Contexts (`current_app`, `g`, `request`)
│   ├── Database Integration (Flask-SQLAlchemy) & Extensions
│   ├── Authentication & Web Security
│   └── Blueprints, Application Factory & Production Best Practices
│
└── 5. Comparisons & Production Scenarios
    ├── Head-to-Head Architectural Comparisons (WSGI vs ASGI, ORM vs SQLAlchemy, etc.)
    └── Real-World Production Debugging Incidents & Interview Scenarios
```

---

## How to Read This Handbook

Every interview topic follows the same natural progression:

```
Real Problem
↓
Why does this exist?
↓
How the framework actually works under the hood
↓
Practical Code Example
↓
Production Perspective & Trade-offs
↓
Likely Interview Follow-up Questions
```

Start with the framework you use every day, or jump directly into **Comparisons & Scenarios** to see how architectural trade-offs look when designing backend systems.
