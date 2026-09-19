# SQLAlchemy 2.0 Cheat Sheet

> Production-ready reference for intermediate Python developers. Covers Core + ORM in the 2.0 style. Assumes PostgreSQL by default; SQLite noted where relevant.

---

## 1. Installation

```bash
# Core + ORM + PostgreSQL driver (psycopg3)
pip install "sqlalchemy[asyncio]>=2.0" "psycopg[binary]"

# Async PostgreSQL driver (alternative)
pip install "sqlalchemy[asyncio]>=2.0" asyncpg

# SQLite (local development, no server)
pip install "sqlalchemy[asyncio]>=2.0" aiosqlite

# Alembic (migrations)
pip install alembic
```

`sqlalchemy[asyncio]` installs the greenlet dependency required for async. For sync-only apps, plain `sqlalchemy` suffices.

---

## 2. Engine, Connection, and Pooling

The `Engine` is the starting point — it holds the dialect and connection pool but does **not** open a DBAPI connection until first use (lazy initialization). All included dialects (except SQLite memory) use `QueuePool` by default, pre-configured with reasonable defaults.

```python
from sqlalchemy import create_engine

# PostgreSQL (psycopg3)
engine = create_engine(
    "postgresql+psycopg://user:pass@localhost:5432/mydb",
    pool_size=20,          # persistent connections
    max_overflow=10,       # extra connections under load
    pool_recycle=1800,     # recycle after 30 min (avoids stale connections)
    pool_timeout=30,       # seconds to wait for a connection
    echo=False,            # True to log SQL
)

# SQLite (local dev)
engine = create_engine("sqlite:///./dev.db", echo=True)
```

**Key classes & methods**

| Class / Method | What it does |
|---|---|
| `create_engine(url, **kw)` | Creates `Engine` with dialect + pool. |
| `Engine.connect()` | Returns a `Connection` (not auto-commit). |
| `Engine.begin()` | Returns a `Connection` wrapped in a transaction — commits on success, rolls back on exception. |
| `Engine.dispose()` | Closes all pooled connections; use on shutdown. |
| `QueuePool` | Default pool. Tune with `pool_size`, `max_overflow`, `pool_recycle`, `pool_timeout`. |
| `AsyncAdaptedQueuePool` | Used automatically by `create_async_engine` — `QueuePool` is not asyncio-compatible. |

---

## 3. Working Flow: Engine → Connection → Session → ORM

```
Engine (pool + dialect)
   │
   ├─ Core path:  Engine.connect() → Connection → execute(select(...))
   │
   └─ ORM path:   Session(engine)  → session.execute(select(Model)) → ORM objects
```

- **Core** is for when you want raw SQL-shaped control or work with tables directly.
- **ORM** is for when you want Python objects with identity, relationships, and unit-of-work persistence.
- A `Session` acquires a `Connection` from the pool automatically when it first needs to talk to the database.
- Both paths share the same `Engine` and `Connection` infrastructure.

**When to use Core vs ORM**

| Use Case | Prefer |
|---|---|
| Bulk inserts / raw SQL performance | Core (`insert()`, `text()`) |
| Simple CRUD with relationships | ORM |
| Complex analytical queries | Core or ORM with `select()` |
| Background jobs with minimal object overhead | Core |
| Domain-driven app with entities | ORM |

---

## 4. DeclarativeBase Models

SQLAlchemy 2.0 uses `DeclarativeBase` as the base class for all models. Every model subclasses it and defines `__tablename__`.

```python
from typing import List, Optional
from sqlalchemy import ForeignKey, String
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(30))
    fullname: Mapped[Optional[str]]

    addresses: Mapped[List["Address"]] = relationship(
        back_populates="user", cascade="all, delete-orphan"
    )

class Address(Base):
    __tablename__ = "address"

    id: Mapped[int] = mapped_column(primary_key=True)
    email_address: Mapped[str]
    user_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))

    user: Mapped["User"] = relationship(back_populates="addresses")
```

Create tables:

```python
Base.metadata.create_all(engine)   # creates all tables (dev only; use Alembic in prod)
```

---

## 5. `Mapped` and `mapped_column`

`Mapped[T]` declares the Python type. `mapped_column()` adds SQL-level configuration.

| `mapped_column` Parameter | Meaning |
|---|---|
| `primary_key=True` | Marks column as PK. |
| `ForeignKey("table.id")` | Defines a foreign key. |
| `String(30)`, `Integer`, etc. | SQL type. |
| `nullable=False` | NOT NULL constraint. |
| `default=...` | Python-side default (applied at INSERT). |
| `server_default=func.now()` | Server-side default (DDL). |
| `unique=True` | UNIQUE constraint. |
| `index=True` | Creates an index. |
| `onupdate=...` | Value applied on UPDATE. |

```python
from datetime import datetime
from sqlalchemy import func

class Post(Base):
    __tablename__ = "post"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200), index=True)
    body: Mapped[Optional[str]]
    created_at: Mapped[datetime] = mapped_column(
        server_default=func.now()
    )
    updated_at: Mapped[Optional[datetime]] = mapped_column(
        onupdate=func.now()
    )
```

---

## 6. Relationships

Use `relationship()` with `Mapped` annotations. Always use `back_populates` for bidirectional relationships.

```python
# One-to-many
class User(Base):
    __tablename__ = "user"
    id: Mapped[int] = mapped_column(primary_key=True)
    posts: Mapped[List["Post"]] = relationship(
        back_populates="author",
        cascade="all, delete-orphan",
    )

class Post(Base):
    __tablename__ = "post"
    id: Mapped[int] = mapped_column(primary_key=True)
    author_id: Mapped[int] = mapped_column(ForeignKey("user.id"))
    author: Mapped["User"] = relationship(back_populates="posts")
```

```python
# Many-to-many
from sqlalchemy import Table, Column

post_tag = Table(
    "post_tag", Base.metadata,
    Column("post_id", ForeignKey("post.id"), primary_key=True),
    Column("tag_id", ForeignKey("tag.id"), primary_key=True),
)

class Post(Base):
    __tablename__ = "post"
    id: Mapped[int] = mapped_column(primary_key=True)
    tags: Mapped[List["Tag"]] = relationship(secondary=post_tag, back_populates="posts")

class Tag(Base):
    __tablename__ = "tag"
    id: Mapped[int] = mapped_column(primary_key=True)
    posts: Mapped[List["Post"]] = relationship(secondary=post_tag, back_populates="tags")
```

| Relationship Parameter | Effect |
|---|---|
| `back_populates` | Bidirectional link to the other side. |
| `cascade="all, delete-orphan"` | Deleting parent deletes children. |
| `lazy="selectin"` | Eager-load via `SELECT ... IN`. |
| `lazy="joined"` | Eager-load via `JOIN`. |
| `lazy="raise"` | Raises if lazy load attempted (safety). |
| `secondary=table` | Many-to-many association table. |
| `uselist=False` | One-to-one scalar instead of collection. |

---

## 7. Session and Unit of Work

The `Session` is the ORM’s unit of work: it tracks pending changes and flushes them to the database at commit or query time.

```python
from sqlalchemy.orm import Session

with Session(engine) as session:
    user = User(name="alice", fullname="Alice Liddell")
    session.add(user)
    session.commit()
    print(user.id)  # populated after commit
```

**Session lifecycle**

| Method | What it does |
|---|---|
| `Session(engine)` | Creates a session bound to an engine. |
| `session.add(obj)` | Marks object as pending for INSERT/UPDATE. |
| `session.add_all([...])` | Adds multiple objects. |
| `session.flush()` | Sends pending SQL to DB without committing. |
| `session.commit()` | Flushes + commits transaction; expires objects by default. |
| `session.rollback()` | Rolls back current transaction. |
| `session.close()` | Releases connection back to pool. |
| `session.get(Model, pk)` | Fetches by primary key (identity map aware). |
| `session.refresh(obj)` | Reloads object from DB. |
| `session.expire(obj)` | Marks object’s attributes as stale. |

**Context manager pattern (recommended)**

```python
with Session(engine) as session:
    with session.begin():          # auto-commit / auto-rollback
        session.add(User(name="bob"))
    # commit happens here
# connection returned to pool
```

---

## 8. CRUD with `select()`

SQLAlchemy 2.0 uses `select()` for all SELECT queries — the legacy `Query` API is deprecated.

```python
from sqlalchemy import select

# CREATE
with Session(engine) as session:
    user = User(name="carol", fullname="Carol Danvers")
    session.add(user)
    session.commit()

# READ
with Session(engine) as session:
    stmt = select(User).where(User.name == "carol")
    user = session.execute(stmt).scalar_one_or_none()

# UPDATE (ORM unit-of-work)
with Session(engine) as session:
    user = session.get(User, 1)
    user.fullname = "Carol Danvers"
    session.commit()

# DELETE
with Session(engine) as session:
    user = session.get(User, 1)
    session.delete(user)
    session.commit()
```

**Result methods**

| Method | Returns |
|---|---|
| `session.execute(stmt)` | `Result` object. |
| `session.scalars(stmt)` | First column of each row (ORM entities). |
| `result.scalars().all()` | List of ORM objects. |
| `result.scalar_one()` | Exactly one scalar; raises if zero or many. |
| `result.scalar_one_or_none()` | One or `None`. |
| `result.first()` | First row or `None`. |
| `result.one()` | Exactly one row; raises otherwise. |

---

## 9. Filtering

```python
from sqlalchemy import select, and_, or_, not_, func

stmt = select(User).where(User.name == "alice")
stmt = select(User).where(and_(User.is_active == True, User.age >= 18))
stmt = select(User).where(or_(User.role == "admin", User.role == "moderator"))
stmt = select(User).where(User.name.like("a%"))
stmt = select(User).where(User.email.ilike("%@example.com"))
stmt = select(User).where(User.id.in_([1, 2, 3]))
stmt = select(User).where(User.deleted_at.is_(None))
stmt = select(User).order_by(User.created_at.desc()).limit(10).offset(20)
```

| Operator | SQL Equivalent |
|---|---|
| `==` | `=` |
| `!=` | `<>` |
| `.like()` / `.ilike()` | `LIKE` / `ILIKE` |
| `.in_([...])` | `IN (...)` |
| `.is_(None)` | `IS NULL` |
| `.is_not(None)` | `IS NOT NULL` |
| `.between(a, b)` | `BETWEEN` |
| `func.lower(col)` | `LOWER(col)` |
| `and_()`, `or_()`, `not_()` | `AND`, `OR`, `NOT` |

---

## 10. Joins

```python
from sqlalchemy import select

# Implicit join via relationship
stmt = (
    select(User, Address)
    .join(Address, User.id == Address.user_id)
    .where(User.name == "alice")
)

# Join via relationship attribute
stmt = select(User).join(User.addresses).where(Address.email_address.like("%@example.com"))

# Outer join
stmt = select(User).outerjoin(User.addresses)

# Join with subquery
subq = select(func.count(Address.id)).where(Address.user_id == User.id).scalar_subquery()
stmt = select(User.name, subq.label("address_count"))
```

---

## 11. Aggregation

```python
from sqlalchemy import select, func

# Count
stmt = select(func.count(User.id)).where(User.is_active == True)
count = session.scalar(stmt)

# Group by + having
stmt = (
    select(User.name, func.count(Address.id).label("addr_count"))
    .join(Address, User.id == Address.user_id)
    .group_by(User.name)
    .having(func.count(Address.id) > 1)
)

# Sum / Avg / Min / Max
stmt = select(
    func.sum(Order.amount),
    func.avg(Order.amount),
    func.min(Order.created_at),
    func.max(Order.created_at),
)
```

---

## 12. Eager Loading

Lazy loading emits extra SELECTs on attribute access. Eager loading loads related objects up front.

```python
from sqlalchemy.orm import selectinload, joinedload

# selectinload — best for collections (one-to-many, many-to-many)
stmt = select(User).options(selectinload(User.addresses))

# joinedload — best for scalar relationships (many-to-one)
stmt = select(Address).options(joinedload(Address.user))

# Nested
stmt = select(User).options(
    selectinload(User.addresses).joinedload(Address.user)
)
```

| Strategy | SQL Emitted | Best For |
|---|---|---|
| `selectinload()` | Extra `SELECT ... IN (...)` | Collections. |
| `joinedload()` | `LEFT OUTER JOIN` | Scalar / many-to-one. |
| `lazy="selectin"` on relationship | Same as `selectinload()` | Default eager for a relationship. |
| `lazy="joined"` on relationship | Same as `joinedload()` | Always eager scalar. |
| `lazy="raise"` | None; raises if accessed | Guard against N+1. |

**Async note:** lazy loading is unsafe in async — always eager-load everything you plan to serialize, or use `AsyncAttrs` + `awaitable_attrs` (SQLAlchemy 2.0.13+).

---

## 13. Async

```python
import asyncio
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy import select

engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost:5432/mydb",
    echo=False,
    pool_size=5,
    max_overflow=10,
)

AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_user(user_id: int):
    async with AsyncSessionLocal() as session:
        result = await session.execute(select(User).where(User.id == user_id))
        return result.scalar_one_or_none()

asyncio.run(get_user(1))
```

**Async session essentials**

| Method | Notes |
|---|---|
| `await session.execute(stmt)` | Always await. |
| `await session.commit()` | Always await. |
| `await session.rollback()` | Always await. |
| `await session.refresh(obj)` | Always await. |
| `session.add(obj)` | **Not** awaited (sync). |
| `session.delete(obj)` | **Not** awaited — use `await session.execute(delete(...))` for safety. |

`expire_on_commit=False` is strongly recommended for async to avoid implicit I/O on attribute access after commit.

---

## 14. Transactions

**Core transaction** — `Engine.begin()` auto-commits on success, rolls back on exception.

```python
with engine.begin() as conn:
    conn.execute(text("INSERT INTO user (name) VALUES (:name)"), {"name": "dave"})
```

**ORM transaction** — `Session.begin()` gives explicit transaction boundaries.

```python
with Session(engine) as session:
    with session.begin():
        session.add(User(name="eve"))
        session.add(Address(email_address="eve@example.com", user_id=1))
    # commit here
```

**Nested transaction (SAVEPOINT)**

```python
with Session(engine) as session:
    with session.begin():
        session.add(User(name="frank"))
        try:
            with session.begin_nested():   # SAVEPOINT
                session.add(User(name="frank"))  # duplicate -> fails
        except IntegrityError:
            pass  # outer transaction survives
        session.add(User(name="grace"))
```

---

## 15. Alembic Migrations

```bash
# Initialize (run once)
alembic init migrations

# Configure alembic.ini
# sqlalchemy.url = postgresql+psycopg://user:pass@localhost:5432/mydb

# Point env.py at your models
# from myapp.models import Base
# target_metadata = Base.metadata

# Generate a migration from model changes
alembic revision --autogenerate -m "add user table"

# Apply migrations
alembic upgrade head

# Roll back one migration
alembic downgrade -1
```

**Async Alembic:** in `env.py`, use `create_async_engine` and `run_async_migrations()` with `async with engine.begin() as conn: await conn.run_sync(...)`.

Always review autogenerated migrations — Alembic may miss custom constraints, enums, or server-side defaults.

---

## 16. Common Pitfalls

| Pitfall | Fix |
|---|---|
| Using legacy `session.query()` | Use `select()` + `session.execute()` — `Query` is deprecated in 2.0. |
| Lazy loading in async | Eager-load with `selectinload`/`joinedload`, or use `AsyncAttrs`. |
| Sharing a Session across threads/tasks | One session per request/task; use `scoped_session` or `async_sessionmaker`. |
| Forgetting `expire_on_commit=False` in async | Set it on `async_sessionmaker` to avoid implicit I/O. |
| `await session.delete()` | `session.delete()` is sync; use `await session.execute(delete(...))` for bulk. |
| Not calling `session.rollback()` after flush failure | Always rollback in the exception handler to reset the session. |
| Using `create_all()` in production | Use Alembic for schema changes. |
| N+1 queries | Detect with SQL logging; fix with `selectinload()`. |
| String `relationship()` args evaluated with `eval()` | Never pass untrusted input to relationship strings. |

---

## 17. Best Practices

- **One `Engine` per app**, created at startup, disposed at shutdown.
- **One `Session` per request / per unit of work** — never share across threads or async tasks.
- **Use `Session.begin()`** (or `async with session.begin()`) for explicit transaction scoping.
- **Prefer `select()` over `Query`** everywhere.
- **Use `mapped_column()`** with type annotations — it enables static type checking.
- **Eager-load everything in async** — lazy loading raises `MissingGreenlet`.
- **Use Alembic for all schema changes** in production; `create_all()` is for dev only.
- **Tune the pool** for your workload: `pool_size`, `max_overflow`, `pool_recycle`, `pool_timeout`.
- **Enable SQL echo** (`echo=True`) during development to spot N+1 and inefficiencies.
- **Use `AsyncAdaptedQueuePool`** implicitly via `create_async_engine` — never force `QueuePool` on async.

---

## 18. TL;DR — 5 Most Important Concepts

1. **Engine → Connection → Session** is the flow. Engine is lazy; Session manages transactions and identity.
2. **Use `select()` + `session.execute()` / `session.scalars()`** — the legacy `Query` API is gone in 2.0.
3. **`DeclarativeBase` + `Mapped` + `mapped_column`** is the 2.0 model style; `back_populates` defines bidirectional relationships.
4. **Eager-load relationships** (`selectinload` for collections, `joinedload` for scalars) — mandatory in async, critical for performance in sync.
5. **Alembic owns schema migrations**; `create_all()` is for local dev only. Tune the connection pool for production.
