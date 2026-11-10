# FastAPI Database Integration & Production Deployment

> This guide explains how to integrate SQLAlchemy 2.0 (async), manage migrations with Alembic, handle modern application lifespan events, write async tests, and deploy FastAPI using Gunicorn and Uvicorn.

---

# 1. Async SQLAlchemy 2.0 Integration

## Interview Question

**How do you connect FastAPI to PostgreSQL using asynchronous SQLAlchemy 2.0, and how does connection pooling work?**

---

### Answer

FastAPI is completely agnostic about database access. The industry-standard ORM for FastAPI is **SQLAlchemy 2.0**.

Using async SQLAlchemy requires an asynchronous driver such as `asyncpg` for PostgreSQL or `aiosqlite` for SQLite.

---

### Step 1: Database Engine & Session Factory

```python
# core/database.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from sqlalchemy.orm import DeclarativeBase
from typing import AsyncGenerator

DATABASE_URL = "postgresql+asyncpg://postgres:secret@localhost:5432/mydb"

# 1. Create the async engine with connection pooling
engine = create_async_engine(
    DATABASE_URL,
    echo=False,           # Set True to log raw SQL queries during debugging
    pool_size=10,         # Maximum persistent connections maintained in the pool
    max_overflow=20,      # Extra connections created if pool_size is exhausted
    pool_timeout=30,      # Seconds to wait for a connection before raising an error
    pool_recycle=1800,    # Reconnect connections older than 30 mins (avoids dropped sockets)
)

# 2. Sessionmaker for creating fresh async sessions
AsyncSessionLocal = async_sessionmaker(
    bind=engine,
    class_=AsyncSession,
    expire_on_commit=False,  # Critical: Prevents attributes from expiring after commit!
)

# 3. Declarative Base for models
class Base(DeclarativeBase):
    pass

# 4. Dependency for FastAPI routes
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        try:
            yield session
        except Exception:
            await session.rollback()
            raise
```

---

### Step 2: Defining Models (SQLAlchemy 2.0 Type Syntax)

Modern SQLAlchemy 2.0 uses type annotations with `Mapped` and `mapped_column`:

```python
# models.py
from sqlalchemy import String, Integer, DateTime, func
from sqlalchemy.orm import Mapped, mapped_column
from core.database import Base
from datetime import datetime

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, index=True)
    username: Mapped[str] = mapped_column(String(50), unique=True, index=True)
    email: Mapped[str] = mapped_column(String(100), unique=True)
    created_at: Mapped[datetime] = mapped_column(DateTime, server_default=func.now())
```

---

### Step 3: Querying the Database in Routes

```python
# routes/users.py
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from core.database import get_db
from models import User

router = APIRouter()

@router.get("/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    # 2.0-style declarative select query
    query = select(User).where(User.id == user_id)
    result = await db.execute(query)
    user = result.scalar_one_or_none()

    if user is None:
        raise HTTPException(status_code=404, detail="User not found")

    return {"id": user.id, "username": user.username, "email": user.email}
```

---

# 2. Database Migrations with Alembic

## Interview Question

**Why does FastAPI not have built-in database migrations like Django, and how do you configure Alembic for an async SQLAlchemy project?**

---

### Answer

Django is a monolithic framework that bundles an ORM and migration tool.

FastAPI is a focused web framework. It leaves database modeling and migrations to dedicated database libraries.

The standard tool for SQLAlchemy migrations is **Alembic**.

### Workflow:

1. **Initialize Alembic with async template**:
   ```bash
   alembic init -t async migrations
   ```
2. **Point `migrations/env.py` to your models**:
   Import your `Base` class so Alembic knows your model schema:
   ```python
   # migrations/env.py
   from core.database import Base
   from models import User  # Must import models so metadata registers!
   target_metadata = Base.metadata
   ```
3. **Generate a migration revision**:
   Alembic compares your Python models against the live database schema:
   ```bash
   alembic revision --autogenerate -m "create users table"
   ```
4. **Apply migration to the database**:
   ```bash
   alembic upgrade head
   ```

---

# 3. Modern Application Lifespan Events

## Interview Question

**FastAPI deprecated `@app.on_event("startup")` and `@app.on_event("shutdown")`. How do you manage application lifecycles using the modern `lifespan` context manager?**

---

### Answer

In modern FastAPI (Starlette 0.28+), application setup and teardown are handled using an **async context manager** passed into the `FastAPI(lifespan=...)` constructor:

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from core.database import engine
import redis.asyncio as redis

# Global cache client
redis_client = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    # 1. STARTUP LOGIC: Runs before accepting requests
    global redis_client
    print("Application starting up... Initializing connection pools")
    redis_client = redis.from_url("redis://localhost:6379", decode_responses=True)
    
    yield  # Application is now running and serving requests!

    # 2. SHUTDOWN LOGIC: Runs when server stops
    print("Application shutting down... Cleaning up resources")
    await redis_client.close()
    await engine.dispose()  # Safely closes all pooled DB connections

app = FastAPI(lifespan=lifespan)
```

### Why this is better than `on_event`:
- All setup and cleanup logic lives in a single, cohesive function.
- Exceptions during startup can be caught cleanly.
- Type-safe and fully compliant with modern ASGI standards.

---

# 4. Production Deployment: Gunicorn + Uvicorn Workers

## Interview Question

**How do you deploy a FastAPI application to production? Why run Gunicorn if Uvicorn is already an ASGI server?**

---

### Answer

In production, running raw Uvicorn like this:

```bash
uvicorn main:app --port 8000  # BAD for production
```

runs only **a single Python process**. If that process crashes or encounters a memory spike, your entire API goes down.

Furthermore, a single Python process can only utilize **one CPU core** due to the GIL. If your server has 8 CPU cores, 7 cores sit completely idle!

### The Production Architecture: Gunicorn as Process Manager

In production, we use **Gunicorn as the master process manager** and instruct it to use **Uvicorn's worker class**:

```
                              Nginx (Reverse Proxy)
                                       │
                                       ▼
                       Gunicorn (Master Process Manager)
                                       │
                ┌──────────────────────┼──────────────────────┐
                ▼                      ▼                      ▼
        Uvicorn Worker 1       Uvicorn Worker 2       Uvicorn Worker 3
         (Event Loop 1)         (Event Loop 2)         (Event Loop 3)
```

### The Command:

```bash
gunicorn main:app \
    --workers 4 \
    --worker-class uvicorn.workers.UvicornWorker \
    --bind 0.0.0.0:8000 \
    --timeout 120 \
    --access-logfile - \
    --error-logfile -
```

### What Gunicorn provides:
1. **Multi-Core Scaling**: Spins up multiple independent worker processes to utilize all CPU cores.
2. **Crash Recovery**: If an unhandled exception or segfault kills Worker 2, the Gunicorn master process immediately spawns a fresh worker to replace it.
3. **Graceful Zero-Downtime Reloads**: Reloads code and workers without dropping active customer connections during deployments (`kill -HUP <master_pid>`).

### How many workers should you run?
The standard rule of thumb for I/O-bound async applications:

$$\text{Workers} = (2 \times \text{CPU Cores}) + 1$$

On a 4-core virtual machine: $(2 \times 4) + 1 = 9$ workers.
