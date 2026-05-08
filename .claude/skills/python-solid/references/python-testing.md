# Testing — Python + FastAPI Strategien

## Grundstruktur

```
tests/
├── conftest.py          # Gemeinsame Fixtures
├── domain/              # Reine Unit-Tests — kein Framework
├── application/         # Services mit In-Memory-Repositories
├── infrastructure/      # Integrationstests — echte SQLite-DB
└── api/                 # End-to-End mit httpx.AsyncClient
```

## pytest-Konfiguration

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"     # Kein @pytest.mark.asyncio nötig
testpaths = ["tests"]
```

```python
# tests/conftest.py
import pytest
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from infrastructure.database.models import Base

@pytest.fixture
async def db_session():
    engine = create_async_engine("sqlite+aiosqlite:///:memory:")
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    session_factory = async_sessionmaker(engine, expire_on_commit=False)
    async with session_factory() as session:
        yield session
    await engine.dispose()
```

## Domain Tests (kein Framework)

```python
# tests/domain/test_email.py
import pytest
from domain.value_objects.email import Email

def test_email_normalisiert_zu_kleinbuchstaben():
    email = Email(value="TEST@EXAMPLE.COM")
    assert str(email) == "test@example.com"

def test_email_wirft_bei_ungueltigem_format():
    with pytest.raises(ValueError, match="Ungültige"):
        Email(value="kein-at-zeichen")

def test_user_ist_admin():
    from domain.models.user import User
    from domain.value_objects.user_id import UserId
    from datetime import datetime

    user = User(
        id=UserId.generate(),
        email=Email(value="admin@test.de"),
        role="admin",
        created_at=datetime.utcnow()
    )
    assert user.is_admin() is True
```

## Application Tests (In-Memory-Repository)

```python
# tests/application/test_user_service.py
import pytest
from application.services.user_service import UserService
from application.dtos.user_dto import CreateUserDto
from tests.fakes.in_memory_user_repository import InMemoryUserRepository

@pytest.fixture
def service():
    return UserService(user_repository=InMemoryUserRepository())

async def test_create_user_speichert_user(service):
    user = await service.create(CreateUserDto(name="Max", email="max@test.de", role="user"))
    assert user.name == "Max"
    assert str(user.email) == "max@test.de"

async def test_create_user_wirft_bei_duplikat(service):
    dto = CreateUserDto(name="Max", email="max@test.de", role="user")
    await service.create(dto)
    with pytest.raises(ValueError, match="bereits vergeben"):
        await service.create(dto)
```

## Infrastructure Tests (echte DB)

```python
# tests/infrastructure/test_user_repository.py
import pytest
from infrastructure.repositories.user_repository import UserRepository
from domain.models.user import User
from domain.value_objects.email import Email
from domain.value_objects.user_id import UserId
from datetime import datetime

async def test_save_und_find_by_id(db_session):
    repo = UserRepository(db_session)
    user = User(
        id=UserId.generate(),
        email=Email(value="test@test.de"),
        role="user",
        created_at=datetime.utcnow()
    )
    saved = await repo.save(user)
    found = await repo.find_by_id(saved.id)
    assert found is not None
    assert str(found.email) == "test@test.de"
```

## API Tests (httpx.AsyncClient)

```python
# tests/api/test_user_routes.py
import pytest
from httpx import AsyncClient, ASGITransport
from main import app
from presentation.dependencies import get_db
from tests.conftest import override_db_session

app.dependency_overrides[get_db] = override_db_session

async def test_create_user_gibt_201_zurueck():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as client:
        response = await client.post("/api/users", json={
            "name": "Max Mustermann",
            "email": "max@test.de",
            "role": "user"
        })
    assert response.status_code == 201
    assert response.json()["email"] == "max@test.de"

async def test_get_user_gibt_404_wenn_nicht_gefunden():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as client:
        response = await client.get("/api/users/nicht-vorhanden")
    assert response.status_code == 404
```

## In-Memory Fake (kein Mock)

```python
# tests/fakes/in_memory_user_repository.py
from domain.repositories.user_repository import IUserRepository
from domain.models.user import User
from domain.value_objects.user_id import UserId
from domain.value_objects.email import Email

class InMemoryUserRepository(IUserRepository):
    def __init__(self):
        self._store: dict[str, User] = {}

    async def find_all(self) -> list[User]:
        return list(self._store.values())

    async def find_by_id(self, user_id: UserId) -> User | None:
        return self._store.get(str(user_id))

    async def find_by_email(self, email: Email) -> User | None:
        return next((u for u in self._store.values() if str(u.email) == str(email)), None)

    async def save(self, user: User) -> User:
        self._store[str(user.id)] = user
        return user

    async def delete(self, user_id: UserId) -> None:
        self._store.pop(str(user_id), None)
```

**Warum Fake statt Mock?** Ein `InMemoryRepository` implementiert das Interface korrekt — es beweist Dependency Inversion. `unittest.mock.MagicMock` tut das nicht.

## Häufige Fehler

| Fehler | Ursache | Fix |
|---|---|---|
| `ScopeMismatch` Fixture-Fehler | async fixture ohne `asyncio_mode = "auto"` | pyproject.toml anpassen |
| Tests beeinflussen sich gegenseitig | Gemeinsamer Zustand in Fixture | Fixture pro Test neu erstellen |
| `greenlet_spawn` in Tests | Sync-Aufruf in async Test | Alle DB-Calls awaiten |
| Import-Fehler in Tests | Fehlende `__init__.py` | In jedem tests/-Unterordner anlegen |
