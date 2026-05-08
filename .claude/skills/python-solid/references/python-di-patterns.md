# Dependency Injection — Python + FastAPI Patterns

## Das Grundprinzip

FastAPI löst Abhängigkeiten über `Depends()`. Alles was eine Route braucht, wird injiziert — nie direkt instanziiert.

```python
# ❌ BAD — direkt instanziiert
@router.get("/users")
async def get_users():
    db = AsyncSession(engine)         # Hartcodiert
    repo = UserRepository(db)
    service = UserService(repo)
    return await service.get_all()

# ✅ GOOD — injiziert
@router.get("/users")
async def get_users(service: IUserService = Depends(get_user_service)):
    return await service.get_all()
```

## Dependency-Kette aufbauen

```python
# presentation/dependencies.py

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        yield session

def get_user_repository(db: AsyncSession = Depends(get_db)) -> IUserRepository:
    return UserRepository(db)

def get_user_service(
    repo: IUserRepository = Depends(get_user_repository)
) -> IUserService:
    return UserService(repo)
```

**Reihenfolge:** `get_db` → `get_user_repository` → `get_user_service` → Route. FastAPI löst die Kette automatisch auf.

## Scoped Sessions (Request-Scope)

Jede HTTP-Anfrage bekommt ihre eigene Session — wird nach dem Request automatisch geschlossen.

```python
# infrastructure/database/session.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

engine = create_async_engine(settings.DATABASE_URL, echo=False)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

## Lifespan — Startup/Shutdown

Seit FastAPI 0.95+ ersetzen Lifespan-Events `on_startup`/`on_shutdown`:

```python
# main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    await engine.dispose(close=False)   # Engine initialisieren
    yield
    # Shutdown
    await engine.dispose()              # Verbindungen schließen

app = FastAPI(lifespan=lifespan)
```

## Globale Dependencies (Router-Ebene)

```python
# Gilt für alle Routen in diesem Router
router = APIRouter(
    prefix="/admin",
    dependencies=[Depends(require_admin_user)]
)
```

## Testing: Dependencies überschreiben

```python
# tests/conftest.py
from main import app
from presentation.dependencies import get_db

async def override_get_db():
    async with test_engine_session() as session:
        yield session

app.dependency_overrides[get_db] = override_get_db
```

## Settings als Dependency

```python
# infrastructure/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    DATABASE_URL: str
    API_KEY: str
    DEBUG: bool = False

    model_config = SettingsConfigDict(env_file=".env")

from functools import lru_cache

@lru_cache
def get_settings() -> Settings:
    return Settings()

# In Route
@router.get("/info")
async def info(settings: Settings = Depends(get_settings)):
    return {"debug": settings.DEBUG}
```

`@lru_cache` stellt sicher, dass Settings nur einmal geladen werden.

## Häufige Fehler

| Fehler | Problem | Fix |
|---|---|---|
| `AsyncSession` direkt im Service | Schichtverletzung | Via `Depends()` injizieren |
| `Depends()` außerhalb von FastAPI | Funktioniert nicht | Nur in Route-Funktionen verwenden |
| Session nicht schließen | Connection-Leak | `async with` / `yield` verwenden |
| Settings jedes Mal neu laden | Langsam | `@lru_cache` auf `get_settings()` |
