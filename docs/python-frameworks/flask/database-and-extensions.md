# Flask Database Integration: SQLAlchemy & Migrations

> This guide explains how Flask-SQLAlchemy manages models and scoped sessions, how to migrate schemas with Flask-Migrate, how to prevent N+1 queries with eager loading, and how connection pools work.

---

# 1. Flask-SQLAlchemy & Scoped Sessions

## Interview Question

**How does Flask-SQLAlchemy manage database sessions across multiple concurrent requests? What is a "Scoped Session"?**

---

### Answer

In plain SQLAlchemy, developers must manually create, track, and close database sessions for every thread:

```python
# Raw SQLAlchemy (Requires manual thread safety)
Session = sessionmaker(bind=engine)
session = Session()
# ... do work ...
session.close()
```

If you forget to close a session in raw SQLAlchemy, connections leak, and your database eventually locks up.

**Flask-SQLAlchemy** solves this by providing a **Scoped Session** (`db.session`):

```python
from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy()

class User(db.Model):
    __tablename__ = "users"
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
```

---

### How Scoped Sessions Work Under the Hood

Under the hood, `db.session` is a **thread-local registry**:

```
Thread 1 (Request from Alice) ──► db.session ──► Unique Session Object A
Thread 2 (Request from Bob)   ──► db.session ──► Unique Session Object B
```

- When Thread 1 touches `db.session`, Flask-SQLAlchemy checks if a session exists for this thread. If not, it opens one.
- When Thread 2 touches `db.session`, it receives its own independent session.
- **Automatic Teardown**: When the HTTP request finishes, Flask-SQLAlchemy automatically hooks into `@app.teardown_appcontext` and calls `db.session.remove()`.

This guarantees that:
1. Two concurrent requests never share or corrupt each other's transactions.
2. Database connections are always safely returned to the connection pool at the end of every request.

---

# 2. Database Migrations with Flask-Migrate

## Interview Question

**Why should you never use `db.create_all()` in production? How does Flask-Migrate manage database evolution?**

---

### Answer

In introductory Flask tutorials, you often see:

```python
with app.app_context():
    db.create_all()  # NEVER DO THIS IN PRODUCTION
```

`db.create_all()` only creates tables that **do not already exist**.

If you add a new column `phone_number` to an existing `User` model, or rename a table, `db.create_all()` does **absolutely nothing** because the table already exists in PostgreSQL.

If you drop into PostgreSQL and manually run `ALTER TABLE users ADD COLUMN phone_number VARCHAR;`:
- Your changes are not tracked in Git.
- Other developers on your team cannot replicate your database state.
- Staging and production databases diverge, leading to deployment failures.

---

### The Production Standard: Flask-Migrate (Alembic)

Flask-Migrate is an extension that wraps **Alembic** directly into the Flask CLI.

```bash
# 1. Initialize migration directory (Run once per project)
flask db init

# 2. Generate migration script by comparing Python models with DB
flask db migrate -m "add phone_number to user"

# 3. Apply the migration to the database
flask db upgrade

# 4. Rollback the last migration (if something goes wrong)
flask db downgrade
```

Every migration file is checked into Git as a version-controlled Python script containing both `upgrade()` and `downgrade()` functions.

---

# 3. Preventing N+1 Queries in SQLAlchemy

## Interview Question

**What is the N+1 query problem in SQLAlchemy, and how do `joinedload()` and `selectinload()` solve it?**

---

### The Problem: Default Lazy Loading

Consider an author-book relationship:

```python
class Author(db.Model):
    __tablename__ = "authors"
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100))
    books = db.relationship("Book", backref="author", lazy="select")

class Book(db.Model):
    __tablename__ = "books"
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200))
    author_id = db.Column(db.Integer, db.ForeignKey("authors.id"))
```

Notice `lazy="select"`. This is SQLAlchemy's default loading strategy.

If you query all authors and print their books:

```python
authors = Author.query.all()  # 1 Query: SELECT * FROM authors;

for author in authors:
    for book in author.books:  # N Queries: SELECT * FROM books WHERE author_id = ?
        print(book.title)
```

If you have 100 authors, this triggers **101 database queries** (N+1 disaster).

---

### The Fix: Eager Loading

SQLAlchemy provides loading options inside `sqlalchemy.orm`:

### Option 1: `joinedload` (SQL `LEFT OUTER JOIN`)

```python
from sqlalchemy.orm import joinedload

# 1 Single SQL query with a JOIN
authors = Author.query.options(joinedload(Author.books)).all()
```

Django equivalent: `select_related`.

**Best for**: Single-valued relationships (`OneToOne` or `ManyToOne`, like `Book` $\rightarrow$ `Author`).

---

### Option 2: `selectinload` (Two SQL Queries with `IN (...)`)

```python
from sqlalchemy.orm import selectinload

# 2 Separate SQL queries:
# Query 1: SELECT * FROM authors;
# Query 2: SELECT * FROM books WHERE author_id IN (1, 2, 3, ...);
authors = Author.query.options(selectinload(Author.books)).all()
```

Django equivalent: `prefetch_related`.

**Best for**: Multi-valued relationships (`OneToMany` or `ManyToMany`, like `Author` $\rightarrow$ `books`). It avoids the Cartesian product explosion of huge SQL joins.

---

# 4. Database Transactions & Atomic Blocks

## Interview Question

**How do you guarantee atomic multi-table updates in Flask-SQLAlchemy?**

---

### Answer

Always wrap multi-step database mutations inside a transaction block:

```python
@app.route("/transfer", methods=["POST"])
def transfer_funds():
    sender_id = request.json.get("sender_id")
    receiver_id = request.json.get("receiver_id")
    amount = request.json.get("amount")

    try:
        # Atomic transaction context manager
        with db.session.begin():
            sender = User.query.filter_by(id=sender_id).with_for_update().first()
            receiver = User.query.filter_by(id=receiver_id).with_for_update().first()

            if sender.balance < amount:
                raise ValueError("Insufficient funds")

            sender.balance -= amount
            receiver.balance += amount
            
            # db.session.commit() is called automatically when exiting the with block!

        return {"status": "transfer completed"}

    except Exception as e:
        # If any error occurs, rollback is executed automatically!
        return {"error": str(e)}, 400
```

Notice `.with_for_update()`. This adds `SELECT ... FOR UPDATE` in SQL, placing a pessimistic row lock to prevent race conditions during balance updates.
