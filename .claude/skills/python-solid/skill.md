---
name: python-solid
description: |
  SOLID principles for Python + FastAPI projects. Applies Clean Architecture with four distinct layers: Domain (models, value objects, interfaces), Application (services, use cases), Infrastructure (database, external APIs), and Presentation (FastAPI routes, Pydantic schemas). Uses dependency injection, type hints, and async/await patterns. Services are classes with ABC/Protocol interfaces, domain models use dataclasses, DTOs use Pydantic.
---

# Python FastAPI SOLID Architecture

You are now operating as a senior Python backend engineer who writes production-grade code following SOLID principles, Clean Architecture, and professional software design patterns.

## Core Principle

**Clean Architecture with SOLID principles for Python backend applications.**

Python FastAPI projects follow **four distinct layers**:
1. **Domain Layer** - Core business logic (framework-agnostic)
2. **Application Layer** - Use cases and services
3. **Infrastructure Layer** - External concerns (database, APIs)
4. **Presentation Layer** - API routes and schemas

## Architecture Rules

### Layer 1: Domain (Core Business Logic)

**Location:** `src/domain/`

**Components:**
- **Models:** Domain entities (dataclasses)
- **Value Objects:** Domain primitives (Pydantic models)
- **Repository Interfaces:** ABC or Protocol

**Rules:**
- NEVER import from FastAPI, SQLAlchemy, or external libraries
- Use only Python standard library + Pydantic for value objects
- Define interfaces (ABC/Protocol) for repositories
- All business rules live here
- Framework-agnostic code

**Example:**
```python
# domain/models/user.py
from dataclasses import dataclass
from datetime import datetime
from domain.value_objects import Email, UserId

@dataclass(frozen=True)
class User:
    id: UserId
    name: str
    email: Email
    role: str
    created_at: datetime

    def is_admin(self) -> bool:
        """Business logic: Check if user is admin."""
        return self.role == 'admin'

    def can_manage_users(self) -> bool:
        """Business rule."""
        return self.is_admin()

# domain/value_objects/email.py
from pydantic import BaseModel, field_validator

class Email(BaseModel):
    """Value object for email addresses."""
    value: str

    @field_validator('value')
    @classmethod
    def validate_email(cls, v: str) -> str:
        if '@' not in v or '.' not in v:
            raise ValueError('Invalid email address')
        return v.lower()

    def __str__(self) -> str:
        return self.value

# domain/repositories/user_repository.py
from abc import ABC, abstractmethod
from typing import Protocol

class IUserRepository(Protocol):
    """Repository interface (dependency inversion)."""
    async def find_all(self) -> list[User]:
        ...

    async def find_by_id(self, user_id: UserId) -> User | None:
        ...

    async def save(self, user: User) -> User:
        ...

    async def delete(self, user_id: UserId) -> None:
        ...
```

### Layer 2: Application (Use Cases & Services)

**Location:** `src/application/`

**Components:**
- **Services:** Business logic orchestration
- **DTOs:** Data transfer objects (Pydantic)
- **Use Cases:** Specific application operations

**Rules:**
- Can depend on Domain layer
- NEVER depend on Infrastructure or Presentation
- Use repository interfaces (not implementations)
- Orchestrate domain logic
- No framework code

**Example:**
```python
# application/services/user_service.py
from abc import ABC, abstractmethod
from application.dtos import CreateUserDto, UpdateUserDto, UserResponseDto
from domain.models import User
from domain.repositories import IUserRepository
from domain.value_objects import Email, UserId

class IUserService(ABC):
    """Service interface."""
    @abstractmethod
    async def get_all(self) -> list[User]:
        ...

    @abstractmethod
    async def get_by_id(self, user_id: str) -> User | None:
        ...

    @abstractmethod
    async def create(self, data: CreateUserDto) -> User:
        ...

    @abstractmethod
    async def delete(self, user_id: str) -> None:
        ...

class UserService(IUserService):
    """User service implementation."""
    def __init__(
        self,
        user_repository: IUserRepository,
        email_validator: IValidator[Email] | None = None
    ):
        self._user_repository = user_repository
        self._email_validator = email_validator

    async def get_all(self) -> list[User]:
        """Get all users."""
        return await self._user_repository.find_all()

    async def get_by_id(self, user_id: str) -> User | None:
        """Get user by ID."""
        uid = UserId(value=user_id)
        return await self._user_repository.find_by_id(uid)

    async def create(self, data: CreateUserDto) -> User:
        """Create new user."""
        # Validate (domain logic)
        email = Email(value=data.email)

        if self._email_validator:
            result = self._email_validator.validate(email)
            if not result.is_valid:
                raise ValidationError(result.errors)

        # Create domain model
        user = User(
            id=UserId.generate(),
            name=data.name,
            email=email,
            role=data.role,
            created_at=datetime.utcnow()
        )

        # Persist
        return await self._user_repository.save(user)

    async def delete(self, user_id: str) -> None:
        """Delete user."""
        uid = UserId(value=user_id)
        await self._user_repository.delete(uid)

# application/dtos/user_dto.py
from pydantic import BaseModel, EmailStr
from datetime import datetime

class CreateUserDto(BaseModel):
    """DTO for creating users."""
    name: str
    email: EmailStr
    role: str = 'user'

class UpdateUserDto(BaseModel):
    """DTO for updating users."""
    name: str | None = None
    email: EmailStr | None = None
    role: str | None = None

class UserResponseDto(BaseModel):
    """DTO for API responses."""
    id: str
    name: str
    email: str
    role: str
    created_at: datetime

    @classmethod
    def from_domain(cls, user: User) -> 'UserResponseDto':
        """Convert domain model to DTO."""
        return cls(
            id=str(user.id),
            name=user.name,
            email=str(user.email),
            role=user.role,
            created_at=user.created_at
        )
```

### Layer 3: Infrastructure (External Concerns)

**Location:** `src/infrastructure/`

**Components:**
- **Repository Implementations:** SQLAlchemy, etc.
- **Database Models:** ORM models
- **External API Clients:** Third-party services
- **Migrations:** Alembic

**Rules:**
- Implements domain repository interfaces
- Contains all framework-specific code
- Translates between domain models and database models
- Handles persistence logic

**Example:**
```python
# infrastructure/database/models.py
from sqlalchemy import Column, String, DateTime
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class UserModel(Base):
    """SQLAlchemy model (ORM)."""
    __tablename__ = 'users'

    id = Column(String, primary_key=True)
    name = Column(String, nullable=False)
    email = Column(String, nullable=False, unique=True)
    role = Column(String, nullable=False)
    created_at = Column(DateTime, nullable=False)

# infrastructure/repositories/user_repository.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from domain.repositories import IUserRepository
from domain.models import User
from domain.value_objects import UserId, Email
from infrastructure.database.models import UserModel

class UserRepository(IUserRepository):
    """SQLAlchemy implementation of user repository."""
    def __init__(self, db: AsyncSession):
        self._db = db

    async def find_all(self) -> list[User]:
        """Find all users."""
        result = await self._db.execute(select(UserModel))
        user_models = result.scalars().all()
        return [self._to_domain(model) for model in user_models]

    async def find_by_id(self, user_id: UserId) -> User | None:
        """Find user by ID."""
        result = await self._db.execute(
            select(UserModel).where(UserModel.id == str(user_id))
        )
        user_model = result.scalar_one_or_none()
        return self._to_domain(user_model) if user_model else None

    async def save(self, user: User) -> User:
        """Save user."""
        user_model = self._from_domain(user)
        self._db.add(user_model)
        await self._db.commit()
        await self._db.refresh(user_model)
        return self._to_domain(user_model)

    async def delete(self, user_id: UserId) -> None:
        """Delete user."""
        result = await self._db.execute(
            select(UserModel).where(UserModel.id == str(user_id))
        )
        user_model = result.scalar_one_or_none()
        if user_model:
            await self._db.delete(user_model)
            await self._db.commit()

    def _to_domain(self, model: UserModel) -> User:
        """Convert ORM model to domain model."""
        return User(
            id=UserId(value=model.id),
            name=model.name,
            email=Email(value=model.email),
            role=model.role,
            created_at=model.created_at
        )

    def _from_domain(self, user: User) -> UserModel:
        """Convert domain model to ORM model."""
        return UserModel(
            id=str(user.id),
            name=user.name,
            email=str(user.email),
            role=user.role,
            created_at=user.created_at
        )
```

### Layer 4: Presentation (API Layer)

**Location:** `src/presentation/`

**Components:**
- **API Routes:** FastAPI routers
- **Schemas:** Pydantic request/response models
- **Dependencies:** FastAPI dependency injection

**Rules:**
- Thin layer - delegates to services
- Handles HTTP concerns (status codes, headers)
- Validates input with Pydantic
- Returns DTOs as responses
- NO business logic

**Example:**
```python
# presentation/api/user_routes.py
from fastapi import APIRouter, Depends, HTTPException, status
from application.services import IUserService
from application.dtos import CreateUserDto, UpdateUserDto, UserResponseDto
from presentation.dependencies import get_user_service

router = APIRouter(prefix="/api/users", tags=["users"])

@router.get("/", response_model=list[UserResponseDto])
async def get_users(
    user_service: IUserService = Depends(get_user_service)
):
    """Get all users."""
    users = await user_service.get_all()
    return [UserResponseDto.from_domain(u) for u in users]

@router.get("/{user_id}", response_model=UserResponseDto)
async def get_user(
    user_id: str,
    user_service: IUserService = Depends(get_user_service)
):
    """Get user by ID."""
    user = await user_service.get_by_id(user_id)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="User not found"
        )
    return UserResponseDto.from_domain(user)

@router.post(
    "/",
    response_model=UserResponseDto,
    status_code=status.HTTP_201_CREATED
)
async def create_user(
    data: CreateUserDto,
    user_service: IUserService = Depends(get_user_service)
):
    """Create new user."""
    try:
        user = await user_service.create(data)
        return UserResponseDto.from_domain(user)
    except ValidationError as e:
        raise HTTPException(
            status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
            detail=str(e)
        )

@router.delete("/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_user(
    user_id: str,
    user_service: IUserService = Depends(get_user_service)
):
    """Delete user."""
    await user_service.delete(user_id)

# presentation/dependencies.py
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from application.services import UserService, IUserService
from infrastructure.repositories import UserRepository
from infrastructure.database import get_db

def get_user_service(
    db: AsyncSession = Depends(get_db)
) -> IUserService:
    """Dependency injection for user service."""
    user_repository = UserRepository(db)
    return UserService(user_repository)
```

## SOLID Principles in Python

### Single Responsibility Principle (SRP)

```python
# ✅ GOOD - Each class has one responsibility
class Email:
    """Only validates and represents email."""
    ...

class UserService:
    """Only handles user operations."""
    ...

class UserRepository:
    """Only handles user persistence."""
    ...

# ❌ BAD - God class
class UserManager:
    def validate_email(self): ...
    def save_to_database(self): ...
    def send_welcome_email(self): ...
    def generate_report(self): ...
```

### Open/Closed Principle (OCP)

```python
# ✅ GOOD - Strategy pattern
from abc import ABC, abstractmethod

class INotificationService(ABC):
    @abstractmethod
    async def send(self, message: str) -> None:
        ...

class EmailNotificationService(INotificationService):
    async def send(self, message: str) -> None:
        # Send email
        ...

class SmsNotificationService(INotificationService):
    async def send(self, message: str) -> None:
        # Send SMS
        ...

# Add new notification types without modifying existing code
```

### Liskov Substitution Principle (LSP)

```python
# ✅ GOOD - All implementations are substitutable
class IUserRepository(Protocol):
    async def find_all(self) -> list[User]: ...

class SqlAlchemyUserRepository:
    async def find_all(self) -> list[User]: ...

class InMemoryUserRepository:
    async def find_all(self) -> list[User]: ...

# UserService works with ANY implementation
service = UserService(SqlAlchemyUserRepository(db))
service = UserService(InMemoryUserRepository())  # For testing
```

### Interface Segregation Principle (ISP)

```python
# ✅ GOOD - Segregated interfaces
class IUserReader(Protocol):
    async def find_all(self) -> list[User]: ...
    async def find_by_id(self, user_id: UserId) -> User | None: ...

class IUserWriter(Protocol):
    async def save(self, user: User) -> User: ...
    async def delete(self, user_id: UserId) -> None: ...

# Full repository combines both
class IUserRepository(IUserReader, IUserWriter, Protocol):
    pass

# Services can depend on only what they need
class UserListService:
    def __init__(self, reader: IUserReader):
        self._reader = reader  # Only needs read access
```

### Dependency Inversion Principle (DIP)

```python
# ✅ GOOD - Depends on abstraction
class UserService:
    def __init__(
        self,
        user_repository: IUserRepository,  # ← Interface
        validator: IValidator[User]        # ← Interface
    ):
        self._user_repository = user_repository
        self._validator = validator

# ❌ BAD - Depends on concrete implementation
class UserService:
    def __init__(self):
        self._user_repository = SqlAlchemyUserRepository()  # ← Concrete
        self._validator = UserValidator()                  # ← Concrete
```

## File Structure

```
src/
├── main.py                          # FastAPI app + DI setup
├── domain/
│   ├── models/
│   │   └── user.py                  # Domain entities
│   ├── value_objects/
│   │   ├── email.py                 # Value objects
│   │   └── user_id.py
│   └── repositories/
│       └── user_repository.py       # Repository interfaces
├── application/
│   ├── services/
│   │   └── user_service.py          # Business logic
│   └── dtos/
│       └── user_dto.py              # Data transfer objects
├── infrastructure/
│   ├── database/
│   │   ├── models.py                # SQLAlchemy models
│   │   └── session.py               # Database connection
│   └── repositories/
│       └── user_repository_impl.py  # Repository implementations
├── presentation/
│   ├── api/
│   │   └── user_routes.py           # FastAPI routes
│   ├── schemas/
│   │   └── user_schema.py           # Pydantic schemas (if different from DTOs)
│   └── dependencies.py              # FastAPI dependencies
└── di/
    └── container.py                 # Dependency injection container
```

## Dependency Injection

```python
# di/container.py
from dependency_injector import containers, providers
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from infrastructure.repositories import UserRepository
from application.services import UserService

class Container(containers.DeclarativeContainer):
    config = providers.Configuration()

    # Database
    engine = providers.Singleton(
        create_async_engine,
        config.database.url,
        echo=config.database.echo
    )

    # Repositories
    user_repository = providers.Factory(
        UserRepository,
        db=providers.Callable(AsyncSession, bind=engine)
    )

    # Services
    user_service = providers.Factory(
        UserService,
        user_repository=user_repository
    )

# main.py
from fastapi import FastAPI
from di.container import Container

def create_app() -> FastAPI:
    app = FastAPI()

    # Setup container
    container = Container()
    container.config.database.url.from_env('DATABASE_URL')
    container.config.database.echo.from_env('DB_ECHO', as_=bool)

    app.container = container

    return app
```

## Testing Strategy

### Domain Tests (No Framework)

```python
# tests/domain/test_user.py
import pytest
from domain.models import User
from domain.value_objects import Email, UserId

def test_user_is_admin():
    user = User(
        id=UserId.generate(),
        name='Admin',
        email=Email(value='admin@test.com'),
        role='admin',
        created_at=datetime.utcnow()
    )

    assert user.is_admin() is True

def test_email_validation():
    with pytest.raises(ValueError):
        Email(value='invalid-email')
```

### Service Tests

```python
# tests/application/test_user_service.py
import pytest
from application.services import UserService
from tests.mocks import MockUserRepository

@pytest.mark.asyncio
async def test_get_all_users():
    repository = MockUserRepository()
    service = UserService(repository)

    users = await service.get_all()

    assert len(users) == 2
```

### API Tests

```python
# tests/api/test_user_routes.py
import pytest
from httpx import AsyncClient
from main import app

@pytest.mark.asyncio
async def test_get_users():
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/api/users")

    assert response.status_code == 200
    assert len(response.json()) > 0
```

## Best Practices Summary

### ✅ DO:

1. **Use type hints everywhere**
2. **ABC/Protocol for interfaces**
3. **Dataclasses for domain models**
4. **Pydantic for DTOs and value objects**
5. **Async/await for I/O operations**
6. **Dependency injection via constructor**
7. **Private attributes with underscore**

### ❌ DON'T:

1. **No business logic in routes**
2. **No SQLAlchemy in domain layer**
3. **No FastAPI in services**
4. **No `Any` types**
5. **No God classes**

For detailed guidance, see reference documentation:
- [Clean Architecture](references/python-architecture.md)
- [Dependency Injection](references/python-di-patterns.md)
- [FastAPI Integration](references/python-fastapi.md)
- [SQLAlchemy Patterns](references/python-sqlalchemy.md)
- [Pydantic v2](references/python-pydantic.md)
- [Testing Strategies](references/python-testing.md)
