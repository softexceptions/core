# SQLAlchemy 2.x Async — Patterns

## Engine und Session Setup

```python
# infrastructure/database/session.py
from sqlalchemy.ext.asyncio import (
    create_async_engine,
    AsyncSession,
    async_sessionmaker,
)

engine = create_async_engine(
    "sqlite+aiosqlite:///./app.db",  # oder postgresql+asyncpg://...
    echo=False,
)

AsyncSessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,  # Wichtig: Objekte nach commit nutzbar
)

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

## ORM-Modelle (bleiben in infrastructure/)

```python
# infrastructure/database/models.py
from sqlalchemy import String, DateTime, ForeignKey
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship
from datetime import datetime

class Base(DeclarativeBase):
    pass

class UserModel(Base):
    __tablename__ = "users"

    id: Mapped[str] = mapped_column(String, primary_key=True)
    email: Mapped[str] = mapped_column(String, unique=True, nullable=False)
    role: Mapped[str] = mapped_column(String, nullable=False)
    created_at: Mapped[datetime] = mapped_column(DateTime, nullable=False)
```

**Wichtig:** Typed Mapped-Columns (SQLAlchemy 2.x) statt alter `Column()`-Syntax.

## Repository-Implementierung

```python
# infrastructure/repositories/user_repository.py
from sqlalchemy import select, delete
from sqlalchemy.ext.asyncio import AsyncSession

class UserRepository(IUserRepository):
    def __init__(self, db: AsyncSession):
        self._db = db

    async def find_all(self) -> list[User]:
        result = await self._db.execute(select(UserModel))
        return [self._to_domain(m) for m in result.scalars().all()]

    async def find_by_id(self, user_id: UserId) -> User | None:
        result = await self._db.execute(
            select(UserModel).where(UserModel.id == str(user_id))
        )
        model = result.scalar_one_or_none()
        return self._to_domain(model) if model else None

    async def find_by_email(self, email: Email) -> User | None:
        result = await self._db.execute(
            select(UserModel).where(UserModel.email == str(email))
        )
        model = result.scalar_one_or_none()
        return self._to_domain(model) if model else None

    async def save(self, user: User) -> User:
        model = self._from_domain(user)
        self._db.add(model)
        await self._db.flush()  # flush statt commit — Session verwaltet Transaktion
        return self._to_domain(model)

    async def delete(self, user_id: UserId) -> None:
        await self._db.execute(
            delete(UserModel).where(UserModel.id == str(user_id))
        )

    def _to_domain(self, model: UserModel) -> User:
        return User(
            id=UserId(value=model.id),
            email=Email(value=model.email),
            role=model.role,
            created_at=model.created_at,
        )

    def _from_domain(self, user: User) -> UserModel:
        return UserModel(
            id=str(user.id),
            email=str(user.email),
            role=user.role,
            created_at=user.created_at,
        )
```

## Migrations mit Alembic

```bash
# Setup
alembic init alembic
```

```python
# alembic/env.py — wichtigste Anpassung
from infrastructure.database.models import Base
from infrastructure.database.session import engine

target_metadata = Base.metadata

def run_migrations_online():
    connectable = engine.sync_engine  # Alembic braucht sync engine
    with connectable.connect() as connection:
        context.configure(connection=connection, target_metadata=target_metadata)
        with context.begin_transaction():
            context.run_migrations()
```

```bash
alembic revision --autogenerate -m "add users table"
alembic upgrade head
```

## Häufige Fehler (SQLAlchemy 2.x)

| Fehler | Ursache | Fix |
|---|---|---|
| `greenlet_spawn` Error | Sync-Methode in async context | Alle DB-Calls mit `await` |
| `DetachedInstanceError` | `expire_on_commit=True` | `expire_on_commit=False` setzen |
| `MissingGreenlet` | `AsyncSession` in sync Code | Nie mix von sync/async |
| Kein `flush()` vor Relation | Fremdschlüssel unbekannt | `await session.flush()` vor Relation |

## In-Memory Repository für Tests

```python
# tests/fakes/in_memory_user_repository.py
class InMemoryUserRepository(IUserRepository):
    def __init__(self):
        self._store: dict[str, User] = {}

    async def find_all(self) -> list[User]:
        return list(self._store.values())

    async def find_by_id(self, user_id: UserId) -> User | None:
        return self._store.get(str(user_id))

    async def save(self, user: User) -> User:
        self._store[str(user.id)] = user
        return user

    async def delete(self, user_id: UserId) -> None:
        self._store.pop(str(user_id), None)
```

Kein echtes SQLAlchemy in Tests — sauber und schnell.
