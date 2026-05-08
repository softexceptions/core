# Pydantic v2 — Patterns und Best Practices

## BaseModel Grundlagen

```python
from pydantic import BaseModel, Field, ConfigDict

class CreateUserDto(BaseModel):
    name: str = Field(min_length=2, max_length=100)
    email: str = Field(pattern=r"^[^@]+@[^@]+\.[^@]+$")
    role: str = Field(default="user")

    model_config = ConfigDict(
        str_strip_whitespace=True,   # Leerzeichen automatisch entfernen
        frozen=True,                 # Unveränderlich nach Erstellung
    )
```

## Field Validators (v2-Syntax)

```python
from pydantic import field_validator, model_validator

class Email(BaseModel):
    value: str

    @field_validator("value")
    @classmethod
    def normalize(cls, v: str) -> str:
        v = v.strip().lower()
        if "@" not in v:
            raise ValueError("Ungültige E-Mail-Adresse")
        return v

    def __str__(self) -> str:
        return self.value
```

**Wichtig:** In Pydantic v2 ist `@classmethod` bei `@field_validator` Pflicht. In v1 war das anders.

## Model Validators

```python
from pydantic import model_validator

class DateRange(BaseModel):
    start: date
    end: date

    @model_validator(mode="after")
    def check_range(self) -> "DateRange":
        if self.end <= self.start:
            raise ValueError("end muss nach start liegen")
        return self
```

## Computed Fields

```python
from pydantic import computed_field

class UserResponseDto(BaseModel):
    first_name: str
    last_name: str

    @computed_field
    @property
    def full_name(self) -> str:
        return f"{self.first_name} {self.last_name}"
```

## Settings mit pydantic-settings

```python
# pydantic-settings ist seit v2 ein separates Paket!
# pip install pydantic-settings

from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    DATABASE_URL: str
    API_PORT: int = 8000
    DEBUG: bool = False
    GEOIP_CITY_DB: str = "/opt/geoip/GeoLite2-City.mmdb"

    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
    )

from functools import lru_cache

@lru_cache
def get_settings() -> Settings:
    return Settings()
```

## ORM-Integration (from_attributes)

```python
class UserResponseDto(BaseModel):
    id: str
    email: str

    model_config = ConfigDict(from_attributes=True)

# Direkt aus SQLAlchemy-Modell erstellen:
dto = UserResponseDto.model_validate(user_orm_model)
```

## Serialisierung

```python
user = CreateUserDto(name="Max", email="max@test.de")

# Dict
user.model_dump()
user.model_dump(exclude={"password"})
user.model_dump(include={"name", "email"})

# JSON
user.model_dump_json()

# Aus Dict
CreateUserDto.model_validate({"name": "Max", "email": "max@test.de"})

# Aus JSON-String
CreateUserDto.model_validate_json('{"name": "Max", "email": "max@test.de"}')
```

## Wichtige v1 → v2 Änderungen

| v1 | v2 | Hinweis |
|---|---|---|
| `validator` | `field_validator` | `@classmethod` jetzt Pflicht |
| `root_validator` | `model_validator` | `mode="before"` oder `"after"` |
| `.dict()` | `.model_dump()` | Alte Methode noch verfügbar |
| `.json()` | `.model_dump_json()` | Alte Methode noch verfügbar |
| `class Config:` | `model_config = ConfigDict(...)` | |
| `pydantic.BaseSettings` | `pydantic_settings.BaseSettings` | Separates Paket |
| `orm_mode = True` | `from_attributes = True` | In ConfigDict |
