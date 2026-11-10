# FastAPI Async Programming & Concurrency

> This guide explores asynchronous programming in Python, how FastAPI executes `async def` vs `def` routes under the hood, the catastrophic pitfall of blocking the event loop, and how to handle background tasks.

---

# 1. Concurrency, Event Loops & Coroutines

## Interview Question

**How does asynchronous programming actually work in Python, and why does `await` improve server throughput without creating thousands of OS threads?**

---

### Answer

In traditional synchronous web frameworks (like standard Django or Flask), concurrency is achieved using **OS threads or processes**:

```
Request 1 ──► [ Worker Thread 1 ] ──► Queries DB (Thread sits idle waiting for 50ms)
Request 2 ──► [ Worker Thread 2 ] ──► Queries DB (Thread sits idle waiting for 50ms)
```

Each OS thread consumes 4–8MB of memory and incurs kernel context-switching overhead. If 500 requests arrive simultaneously, you would need 500 threads, quickly exhausting server RAM.

---

### The Asynchronous Event Loop

In an async framework like FastAPI (running on Uvicorn), thousands of requests run on a **single OS thread** managed by an **Event Loop**:

```
                       The Event Loop
                             │
            ┌────────────────┴────────────────┐
            ▼                                 ▼
   Request 1 (await DB)              Request 2 (await Stripe)
   "I'm waiting on PostgreSQL"       "I'm waiting on Network"
            │                                 │
            └─────────┬───────────────────────┘
                      ▼
   Event loop switches INSTANTLY to Request 3 (Computes JSON)
```

When Request 1 hits `await db.fetch()`, it pauses its own coroutine and yields control back to the event loop.

Instead of sitting idle, the event loop immediately picks up Request 2, Request 3, or accepts new incoming connections.

When PostgreSQL finishes sending data, the event loop resumes Request 1 right where it left off.

This is **Cooperative Multitasking**. A single server can easily handle 10,000 concurrent waiting connections with very low memory overhead.

---

# 2. `async def` vs normal `def`: Under the Hood

This is one of the most important technical interview questions for any FastAPI developer.

## Interview Question

**What is the difference between declaring a route with `async def` versus normal `def` in FastAPI? How does FastAPI execute each?**

---

### Answer

Many developers assume: *"FastAPI is an async framework, so I must put `async def` on every single route."*

**That assumption is dangerous and wrong.**

FastAPI treats `async def` and normal `def` routes very differently:

```
Route Declaration                  Where FastAPI Executes It
─────────────────                  ─────────────────────────
async def route_name(...)    ──►   Main Thread (Directly on the Event Loop)
def route_name(...)          ──►   External Thread Pool (Worker Threads)
```

---

### 1. What happens with `def` routes?

When FastAPI sees a regular synchronous function:

```python
@app.get("/sync-data")
def read_data():
    time.sleep(2)  # Blocking operation
    return {"status": "ok"}
```

FastAPI detects that this is **not** a coroutine.

To protect the main event loop from freezing, FastAPI automatically offloads this function to an external **thread pool** (using `anyio.to_thread.run_sync`).

The thread pool executes the blocking code in a background thread, while the main event loop remains free to serve other requests. When the thread finishes, FastAPI returns the HTTP response.

---

### 2. What happens with `async def` routes?

When FastAPI sees an `async def` function:

```python
@app.get("/async-data")
async def read_data():
    await asyncio.sleep(2)  # Non-blocking pause
    return {"status": "ok"}
```

FastAPI executes this function **directly on the main event loop thread**.

Because `await asyncio.sleep(2)` is non-blocking, it yields control back to the event loop cooperatively.

---

# 3. The Fatal Production Hazard: Blocking Code Inside `async def`

## Interview Question

**What happens if you execute blocking synchronous code (like `time.sleep()`, synchronous `requests.get()`, or synchronous database calls) inside an `async def` route?**

---

### The Disaster Scenario

Look at this innocent-looking route:

```python
# CATASTROPHIC BUG IN PRODUCTION
import requests  # Synchronous requests library!

@app.get("/user-stats/{user_id}")
async def get_stats(user_id: int):
    # This third-party external API takes 3 seconds to respond
    response = requests.get(f"https://slow-partner.com/api/{user_id}")
    return response.json()
```

### Why this kills your entire server:

1. You declared the function with `async def`.
2. FastAPI runs it **directly on the main event loop thread**.
3. Inside, you called `requests.get()`. This is **synchronous, blocking I/O**. It does *not* have an `await` and does *not* yield control.
4. **The entire event loop freezes solid for 3 seconds.**

During those 3 seconds:
- Zero other requests can be processed.
- Other `async def` routes cannot run.
- Health-check probes (`/healthz`) fail.
- Kubernetes thinks your container died and restarts it.

If 10 users hit this endpoint concurrently, your server becomes completely unresponsive for 30 seconds!

---

### How to Fix It (The Two Correct Approaches)

### Approach A: Use Async Libraries with `await` (Preferred)

Replace synchronous libraries with async equivalents:

```python
import httpx

@app.get("/user-stats/{user_id}")
async def get_stats(user_id: int):
    async with httpx.AsyncClient() as client:
        # Non-blocking: yields control to the event loop while waiting for network
        response = await client.get(f"https://slow-partner.com/api/{user_id}")
        return response.json()
```

---

### Approach B: Use Normal `def` for Synchronous Code

If you must use a library that has no async version (e.g. legacy AWS SDK or synchronous database driver), **remove `async`**:

```python
import requests

# FastAPI automatically moves this to a worker thread! Event loop never freezes!
@app.get("/user-stats/{user_id}")
def get_stats(user_id: int):
    response = requests.get(f"https://slow-partner.com/api/{user_id}")
    return response.json()
```

---

### Summary Rule for Senior Interviews

```
If your code uses `await` on non-blocking async calls  ──► Use `async def`
If your code uses blocking I/O (sync DB, requests, etc.) ──► Use normal `def`
NEVER put blocking synchronous calls inside `async def`!
```

---

# 4. CPU-Bound Tasks vs I/O-Bound Tasks

## Interview Question

**Does `async/await` speed up CPU-intensive tasks like image processing or machine learning model inference?**

---

### Answer

**No.**

`async/await` only optimizes **I/O-bound operations** (waiting on databases, network calls, disk reads).

A CPU-bound task (e.g. resizing a 20MB image, password hashing, training a scikit-learn model) requires continuous CPU computation.

Because of Python's **Global Interpreter Lock (GIL)**, a single Python process can only execute CPU instructions on one core at a time. Putting `async def` on a CPU-heavy calculation will still peg the core and block the event loop.

### How to handle CPU-bound workloads in FastAPI:
1. **Thread Pool**: Use normal `def` or `anyio.to_thread.run_sync()`. (Releases the event loop, but still limited by the GIL).
2. **Process Pool (`concurrent.futures.ProcessPoolExecutor`)**: Distributes CPU computation across separate Python interpreter processes, bypassing the GIL.
3. **Distributed Worker (Celery / RQ)**: Best for production. Enqueue the image processing task to a Celery worker pool and return a `202 Accepted` response with a task ID.

---

# 5. FastAPI `BackgroundTasks`

## Interview Question

**How does FastAPI's `BackgroundTasks` work, and when should you choose Celery instead?**

---

### Answer

FastAPI includes a built-in `BackgroundTasks` utility for running lightweight jobs **after the HTTP response has already been sent to the client**:

```python
from fastapi import FastAPI, BackgroundTasks

app = FastAPI()

def write_audit_log(email: str, message: str):
    with open("audit.log", "a") as f:
        f.write(f"User {email}: {message}\n")

@app.post("/register/")
async def register(email: str, background_tasks: BackgroundTasks):
    # Register user in DB ...
    
    # Schedule task to run AFTER response is returned
    background_tasks.add_task(write_audit_log, email, "Account created")
    
    # Client receives response immediately!
    return {"message": "Registration successful"}
```

---

### When to use `BackgroundTasks` vs Celery

| Feature | `BackgroundTasks` | Celery |
|---|---|---|
| **Infrastructure** | Zero extra infrastructure (Runs in same FastAPI process) | Requires Message Broker (Redis/RabbitMQ) + Worker processes |
| **Durability** | **In-memory only**. If server restarts or pod crashes, pending tasks are lost! | **Persistent**. Tasks remain safely queued in Redis/RabbitMQ |
| **Retries & Monitoring**| No built-in retry mechanism or failure tracking | Sophisticated exponential backoff, rate limits, Flower UI |
| **Best For** | Fast, non-critical tasks (writing a log line, updating an in-memory counter) | Heavy jobs, sending emails, generating PDFs, financial transactions |
