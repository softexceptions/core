# Clean Architecture — Python Vertiefung

## Das Schichtenmodell

```
domain/          ← Kern — keine Abhängigkeiten nach außen
application/     ← Orchestrierung — hängt nur von domain/ ab
infrastructure/  ← Technik — hängt von domain/ ab, implementiert Interfaces
presentation/    ← HTTP — hängt von application/ ab
```

**Die goldene Regel:** Abhängigkeiten zeigen immer nach innen. `presentation/` kennt `application/`, aber nie umgekehrt.

## domain/ — Was gehört rein?

```
domain/
├── models/          # @dataclass(frozen=True) — unveränderliche Entities
├── value_objects/   # Pydantic BaseModel mit Validierung
└── repositories/    # Protocol-Interfaces — kein SQLAlchemy
```

### Entity vs. Value Object

| | Entity | Value Object |
|---|---|---|
| Identität | Hat eine ID | Hat keine ID |
| Vergleich | Über ID | Über alle Felder |
| Mutabilität | Kann sich ändern | Unveränderlich |
| Beispiel | `User(id=UserId(...))` | `Email(value="...")` |

```python
# Entity
@dataclass(frozen=True)
class User:
    id: UserId        # Identität
    email: Email
    role: str

    def is_admin(self) -> bool:
        return self.role == "admin"

# Value Object
class Email(BaseModel):
    model_config = ConfigDict(frozen=True)
    value: str

    @field_validator("value")
    @classmethod
    def validate(cls, v: str) -> str:
        if "@" not in v:
            raise ValueError("Ungültige E-Mail")
        return v.lower()
```

## application/ — Was gehört rein?

```
application/
├── services/    # Klassen mit Interface (IUserService / UserService)
└── dtos/        # Pydantic Request/Response-Modelle
```

**Regel:** Services orchestrieren — sie rufen Repositories auf, kombinieren Domain-Logik, aber enthalten keine SQL-Queries und kein FastAPI.

```python
class UserService(IUserService):
    def __init__(self, repo: IUserRepository):
        self._repo = repo

    async def create(self, dto: CreateUserDto) -> User:
        # 1. Domain-Objekt bauen (Validierung im Value Object)
        email = Email(value=dto.email)
        # 2. Geschäftsregel prüfen
        existing = await self._repo.find_by_email(email)
        if existing:
            raise ValueError("E-Mail bereits vergeben")
        # 3. Persistieren
        user = User(id=UserId.generate(), email=email, role=dto.role)
        return await self._repo.save(user)
```

## infrastructure/ — Was gehört rein?

```
infrastructure/
├── database/
│   ├── models.py      # SQLAlchemy ORM-Modelle (Base)
│   └── session.py     # AsyncEngine + AsyncSession-Factory
└── repositories/
    ├── user_repo.py   # SQLAlchemy-Implementierung
    └── in_memory/     # In-Memory für Tests
```

**Pflicht:** Jede Repository-Klasse hat `_to_domain()` und `_from_domain()` — keine Domain-Objekte in ORM-Modellen.

## presentation/ — Was gehört rein?

```
presentation/
├── api/
│   └── user_routes.py   # APIRouter — nur HTTP-Logik
└── dependencies.py      # FastAPI Depends()-Funktionen
```

**Thin Layer:** Route-Funktion = Request validieren → Service aufrufen → Response zurückgeben. Drei Zeilen, keine Logik.

## Häufige Schichtverletzungen

| Verstoß | Symptom | Fix |
|---|---|---|
| SQLAlchemy in domain/ | `from sqlalchemy import ...` | ORM-Modelle in infrastructure/ halten |
| FastAPI in application/ | `from fastapi import Depends` | DI in presentation/dependencies.py |
| Business-Logik in Route | if/else in Route-Funktion | In Service auslagern |
| Repository-Impl in domain/ | Konkrete Klasse statt Protocol | Protocol definieren, Impl in infra/ |

## Wann die Regeln brechen?

**Kleine Projekte (< 5 Routen):** Ein vereinfachtes Schema ist akzeptabel:
```
src/
├── models.py       # Domain + ORM zusammen (bewusste Ausnahme)
├── services.py     # Nur 1 Datei statt Ordner
└── main.py         # Routes + DI zusammen
```

Sobald das Projekt wächst, sofort auf die volle Struktur migrieren.
