---
name: tdd
description: |
  Striktes Test-Driven Development (TDD) für Python + FastAPI (Backend), Vue 3 + TypeScript (Frontend) und Flutter + Dart (Mobile).
  Erzwingt den Red → Green → Refactor-Zyklus: Kein Produktionscode ohne vorherigen fehlschlagenden Test.
  Verwendet pytest + pytest-asyncio (Python), Vitest + Vue Test Utils (Vue), flutter_test + mocktail (Flutter/Dart).
  Integriert mit python-solid, vue-solid und flutter-solid: Tests spiegeln die Clean-Architecture-Schichtung wider.

  Verwende diesen Skill immer wenn:
  - Der Nutzer eine neue Funktion, einen Service, ein Repository oder eine Route implementieren möchte
  - Der Nutzer sagt "implementiere", "schreib", "erstelle", "füge hinzu" (Produktionscode)
  - Code-Review gefragt wird (prüfen ob Tests vor Code existieren)
  - Refactoring ansteht
  - Der Nutzer explizit TDD erwähnt
---

# TDD Skill — Red → Green → Refactor

Du bist jetzt ein Senior-Entwickler, der **strikt nach TDD** arbeitet. Kein Produktionscode ohne vorherigen fehlschlagenden Test. Kein Ausnahmen.

## Der Zyklus (STRIKT)

```
🔴 RED    → Schreibe einen fehlschlagenden Test
🟢 GREEN  → Schreibe minimalen Code, der den Test bestehen lässt
🔵 REFACTOR → Verbessere den Code ohne das Verhalten zu ändern
```

**Commit-Strategie:** Je Phase ein Commit.
```
git commit -m "test(red): <was getestet wird>"
git commit -m "feat(green): <minimale Implementierung>"
git commit -m "refactor: <was verbessert wurde>"
```

---

## Ablauf bei jeder Implementierungsaufgabe

### Schritt 1 — Anforderung analysieren
Bevor irgendetwas geschrieben wird:
- Was soll das Verhalten sein? (nicht die Implementierung)
- Welche Schicht betrifft das? (Domain / Application / Infrastructure / Presentation)
- Welche Edge Cases gibt es?

### Schritt 2 — 🔴 RED: Test zuerst
Schreibe den Test. Er **muss** fehlschlagen, weil der Produktionscode noch nicht existiert.
Zeige explizit den erwarteten Fehler: `ModuleNotFoundError` oder `AttributeError` ist der Beweis, dass TDD eingehalten wird.

### Schritt 3 — 🟢 GREEN: Minimale Implementierung
Schreibe **nur so viel Code wie nötig**, um den Test zu bestehen. Kein Gold-Plating.
"Minimal" bedeutet: Keine Features, die kein Test verlangt.

### Schritt 4 — 🔵 REFACTOR: Aufräumen
Jetzt darf verbessert werden — aber kein Test darf danach rot werden.

---

## Python + FastAPI (mit python-solid)

Die Test-Struktur spiegelt die Clean-Architecture-Schichten wider:

```
tests/
├── domain/          → Reine Unit-Tests, kein Framework
├── application/     → Services mit Mock-Repositories
├── infrastructure/  → Integrationstests (echte DB oder TestContainers)
└── api/             → End-to-End mit httpx.AsyncClient
```

### Domain Layer Tests (kein Framework, keine Mocks nötig)

```python
# 🔴 RED — tests/domain/test_user.py
import pytest
from domain.value_objects.email import Email  # existiert noch nicht → ModuleNotFoundError

def test_email_rejects_invalid_format():
    with pytest.raises(ValueError, match="Invalid email"):
        Email(value="kein-at-zeichen")

def test_email_normalizes_to_lowercase():
    email = Email(value="TEST@EXAMPLE.COM")
    assert str(email) == "test@example.com"
```

```python
# 🟢 GREEN — src/domain/value_objects/email.py
from pydantic import BaseModel

class Email(BaseModel):
    value: str

    def model_post_init(self, __context):
        if "@" not in self.value:
            raise ValueError("Invalid email")
        self.value = self.value.lower()

    def __str__(self) -> str:
        return self.value
```

### Application Layer Tests (Mock-Repository via Protocol)

```python
# 🔴 RED — tests/application/test_user_service.py
import pytest
from application.services.user_service import UserService  # existiert noch nicht

class InMemoryUserRepository:
    """Testdouble — implementiert IUserRepository"""
    def __init__(self):
        self._users = []

    async def save(self, user):
        self._users.append(user)
        return user

    async def find_by_email(self, email):
        return next((u for u in self._users if str(u.email) == str(email)), None)

@pytest.mark.asyncio
async def test_create_user_stores_user():
    repo = InMemoryUserRepository()
    service = UserService(user_repository=repo)

    from application.dtos.user_dto import CreateUserDto
    user = await service.create(CreateUserDto(name="Max", email="max@test.de", role="user"))

    assert user.name == "Max"
    assert str(user.email) == "max@test.de"

@pytest.mark.asyncio
async def test_create_user_rejects_duplicate_email():
    repo = InMemoryUserRepository()
    service = UserService(user_repository=repo)

    from application.dtos.user_dto import CreateUserDto
    dto = CreateUserDto(name="Max", email="max@test.de", role="user")
    await service.create(dto)

    with pytest.raises(ValueError, match="already exists"):
        await service.create(dto)
```

### API Layer Tests

```python
# 🔴 RED — tests/api/test_user_routes.py
import pytest
from httpx import AsyncClient, ASGITransport
from main import app

@pytest.mark.asyncio
async def test_post_user_returns_201():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as client:
        response = await client.post("/api/users", json={
            "name": "Max Mustermann",
            "email": "max@test.de",
            "role": "user"
        })
    assert response.status_code == 201
    assert response.json()["email"] == "max@test.de"
```

### pytest-Konfiguration

```ini
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

```python
# tests/conftest.py
import pytest
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker
from infrastructure.database.models import Base

@pytest.fixture
async def db_session():
    engine = create_async_engine("sqlite+aiosqlite:///:memory:")
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    async_session = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)
    async with async_session() as session:
        yield session
    await engine.dispose()
```

---

## Vue 3 + TypeScript (Vitest + Vue Test Utils)

### TypeScript-Pflichtregeln (keine Ausnahmen)

- **Alle Dateien sind `.ts` oder `.vue` mit `<script setup lang="ts">`** — kein JavaScript
- **Kein `any`** — stattdessen `unknown` + Type Guard, oder ein konkretes Interface
- **Alle Funktionsparameter und Rückgabewerte explizit typisieren**
- **Interfaces für alle öffentlichen APIs** — auch für Testdoubles (`MockApiClient implements IApiClient`)
- **`as unknown as T` nur in Tests erlaubt** — niemals im Produktionscode
- **`tsconfig.json` mit `"strict": true`** — nicht verhandelbar

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "noImplicitReturns": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  }
}
```

**Im RED-Schritt gilt:** Typen im Test definieren bevor die Implementierung existiert — ein TypeScript-Compilerfehler ist genauso ein gültiger RED-Beweis wie ein Runtime-Fehler.

**Code-Review:** Wenn `any` im Vue-Code auftaucht → automatisch als TDD-Verletzung werten, da es ein nicht spezifiziertes Verhalten versteckt.

---

Integration mit **vue-solid**: Die drei Architektur-Schichten bestimmen die Test-Strategie.

```
tests/
├── unit/
│   ├── models/      → Value Objects, Domain-Modelle (kein Vue, kein Framework)
│   ├── services/    → Business-Logic-Klassen (kein Vue, MockApiClient)
│   ├── composables/ → Adapter-Schicht (Vue-Reaktivität mit Mock-Services)
│   └── components/  → Vue-Komponenten (mount, Mock-Composables oder Props)
└── e2e/             → Optional (Playwright)
```

**Schicht → Test-Regel:**
| Schicht | Vue-Import erlaubt? | Was wird gemockt? |
|---|---|---|
| `models/` | ❌ Nein | Nichts |
| `services/` | ❌ Nein | `IApiClient` via Mock-Klasse |
| `composables/` | ✅ Ja | `IUserService` via `vi.fn()` |
| `components/` | ✅ Ja | Composable-Rückgaben oder Props |

---

### Schicht 1: Model/Value-Object Tests (kein Framework)

```typescript
// 🔴 RED — tests/unit/models/Email.spec.ts
import { describe, it, expect } from 'vitest'
import { Email } from '@/models/Email'  // existiert noch nicht → ImportError

describe('Email', () => {
  it('rejects invalid format', () => {
    expect(() => Email.create('kein-at')).toThrow('Invalid email')
  })

  it('normalizes to lowercase', () => {
    const email = Email.create('TEST@EXAMPLE.COM')
    expect(email.toString()).toBe('test@example.com')
  })

  it('two emails with same value are equal', () => {
    const a = Email.create('max@test.de')
    const b = Email.create('max@test.de')
    expect(a.equals(b)).toBe(true)
  })
})
```

```typescript
// 🟢 GREEN — src/models/Email.ts
export class Email {
  private constructor(private readonly value: string) {}

  static create(value: string): Email {
    if (!value.includes('@')) throw new Error('Invalid email')
    return new Email(value.toLowerCase())
  }

  equals(other: Email): boolean {
    return this.value === other.value
  }

  toString(): string {
    return this.value
  }
}
```

---

### Schicht 2: Service Tests (kein Vue, MockApiClient)

```typescript
// 🔴 RED — tests/unit/services/UserService.spec.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { UserService } from '@/services/implementations/UserService'  // existiert noch nicht
import type { IApiClient } from '@/services/interfaces/IApiClient'

class MockApiClient implements IApiClient {
  private responses: Record<string, unknown> = {}

  setResponse(url: string, data: unknown) {
    this.responses[url] = data
  }

  async get<T>(url: string): Promise<T> {
    return this.responses[url] as T
  }

  async post<T>(_url: string, _data: unknown): Promise<T> {
    return this.responses['post'] as T
  }
}

describe('UserService', () => {
  let apiClient: MockApiClient
  let service: UserService

  beforeEach(() => {
    apiClient = new MockApiClient()
    service = new UserService(apiClient)
  })

  it('maps DTOs to domain User objects', async () => {
    apiClient.setResponse('/users', [{ id: '1', name: 'Max', email: 'max@test.de', role: 'user', created_at: new Date().toISOString() }])

    const users = await service.getAll()

    expect(users).toHaveLength(1)
    expect(users[0].name).toBe('Max')
    expect(users[0].email.toString()).toBe('max@test.de')
  })

  it('rejects invalid email on create', async () => {
    await expect(service.create({ name: 'Max', email: 'kein-at', role: 'user' }))
      .rejects.toThrow('Invalid email')
  })
})
```

---

### Schicht 3: Composable Tests (Vue-Reaktivität, Mock-Service)

```typescript
// 🔴 RED — tests/unit/composables/useUserManagement.spec.ts
import { describe, it, expect, vi } from 'vitest'
import { useUserManagement } from '@/composables/useUserManagement'  // existiert noch nicht
import type { IUserService } from '@/services/interfaces/IUserService'

const mockUser = { id: '1', name: 'Max', email: { toString: () => 'max@test.de' }, isAdmin: () => false }

describe('useUserManagement', () => {
  it('starts with empty users and not loading', () => {
    const mockService = { getAll: vi.fn(), create: vi.fn(), delete: vi.fn() } as unknown as IUserService
    const { users, loading } = useUserManagement(mockService)

    expect(users.value).toHaveLength(0)
    expect(loading.value).toBe(false)
  })

  it('loads users and sets loading state correctly', async () => {
    const mockService = { getAll: vi.fn().mockResolvedValue([mockUser]) } as unknown as IUserService
    const { users, loading, loadUsers } = useUserManagement(mockService)

    const promise = loadUsers()
    expect(loading.value).toBe(true)
    await promise
    expect(loading.value).toBe(false)
    expect(users.value).toHaveLength(1)
  })

  it('sets error on failed load', async () => {
    const mockService = { getAll: vi.fn().mockRejectedValue(new Error('Netzwerkfehler')) } as unknown as IUserService
    const { error, loadUsers } = useUserManagement(mockService)

    await loadUsers()

    expect(error.value).toBe('Netzwerkfehler')
  })
})
```

---

### Schicht 4: Komponenten-Tests (Props / data-testid)

```typescript
// 🔴 RED — tests/unit/components/UserList.spec.ts
import { describe, it, expect } from 'vitest'
import { mount } from '@vue/test-utils'
import UserList from '@/components/UserList.vue'  // existiert noch nicht

describe('UserList', () => {
  it('rendert einen Eintrag pro User', () => {
    const wrapper = mount(UserList, {
      props: { users: [mockUser, { ...mockUser, id: '2', name: 'Moritz' }] }
    })

    expect(wrapper.findAll('[data-testid="user-item"]')).toHaveLength(2)
    expect(wrapper.text()).toContain('Max')
  })

  it('zeigt Empty State wenn keine Users', () => {
    const wrapper = mount(UserList, { props: { users: [] } })
    expect(wrapper.find('[data-testid="empty-state"]').exists()).toBe(true)
  })

  it('emittet delete-Event mit User-ID', async () => {
    const wrapper = mount(UserList, { props: { users: [mockUser] } })
    await wrapper.find('[data-testid="delete-btn"]').trigger('click')
    expect(wrapper.emitted('delete')?.[0]).toEqual(['1'])
  })
})
```

### vitest-Konfiguration

```typescript
// vite.config.ts
export default defineConfig({
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./tests/setup.ts'],
  }
})
```

---

## Code-Review: TDD-Compliance prüfen

Wenn Code-Review angefragt wird, prüfe:

1. **Existiert ein Test für jede öffentliche Methode/Funktion?**
2. **Sind Tests unabhängig vom Framework?** (Domain/Service-Tests dürfen kein FastAPI/Vue importieren)
3. **Sind Mocks auf das Minimum reduziert?** (Zu viele Mocks = schlechtes Design-Signal)
4. **Testen die Tests Verhalten, nicht Implementierung?** (`.called_once()` ist OK, interne Variablen zu prüfen nicht)
5. **Folgen die Commits dem Red→Green→Refactor-Muster?**

### Checkliste (ausgeben bei Review)

```
TDD Code-Review Checkliste
--------------------------
[ ] Test existiert vor Produktionscode (git log prüfen)
[ ] Test war initial rot (dokumentiert im Commit-Message)
[ ] Produktionscode ist minimal (kein ungenutztes Feature)
[ ] Refactoring-Schritt ist sauber (kein gebrochener Test)
[ ] Test-Struktur spiegelt Schicht-Architektur wider
[ ] Mocks nur an Schichtgrenzen (keine internen Implementierungsdetails gemockt)
```

---

---

## Flutter + Dart (flutter_test + mocktail)

### Test-Stack

| Package | Zweck |
|---|---|
| `flutter_test` | Built-in, kein Extra-Package nötig |
| `mocktail` | Mocking ohne Code-Generator (bevorzugt gegenüber `mockito`) |

```yaml
# pubspec.yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
  mocktail: ^1.0.0
```

Tests ausführen: `flutter test`

### Test-Struktur (spiegelt flutter-solid-Schichten)

```
test/
├── domain/
│   ├── models/        → pure Dart, kein Flutter-Import, keine Mocks
│   └── use_cases/     → Use Cases mit Mock-Repositories (mocktail)
├── data/
│   └── repositories/  → Repository-Implementierungen mit Mock-DataSources
└── presentation/
    ├── controllers/   → Riverpod-Controller mit ProviderContainer
    └── widgets/       → Widget-Tests (testWidgets)
```

**Schicht → Test-Regel:**

| Schicht | Flutter-Import | Was wird gemockt? |
|---|---|---|
| `domain/models/` | ❌ Nein | Nichts |
| `domain/use_cases/` | ❌ Nein | `abstract class IRepository` via `mocktail` |
| `data/repositories/` | ❌ Nein | DataSource via `mocktail` |
| `presentation/controllers/` | ❌ Nein | Use Cases via `mocktail` + `ProviderContainer` |
| `presentation/widgets/` | ✅ Ja | Controller via Provider-Override |

---

### Layer 1: Domain Model Tests (kein Mock nötig)

```dart
// 🔴 RED — test/domain/models/charging_station_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:kia_charge_nav/domain/models/charging_station.dart'; // existiert noch nicht

void main() {
  group('ChargingStation.isInFront', () {
    late ChargingStation station;

    setUp(() {
      station = const ChargingStation(
        id: '1', name: 'Shell A5',
        network: 'Shell Recharge',
        latitude: 50.1, longitude: 8.7,
      );
    });

    test('returns true when station is within tolerance ahead', () {
      // Fahrtrichtung Nord (0°), Station bei 20° → innerhalb 45° Toleranz
      expect(station.isInFront(0, 20), isTrue);
    });

    test('returns false when station is behind', () {
      // Fahrtrichtung Nord (0°), Station bei 180° → hinter dem Fahrzeug
      expect(station.isInFront(0, 180), isFalse);
    });

    test('respects custom tolerance', () {
      // Enge Toleranz: 10°, Station bei 20° → außerhalb
      expect(station.isInFront(0, 20, toleranceDeg: 10), isFalse);
    });
  });
}
```

```dart
// 🟢 GREEN — lib/domain/models/charging_station.dart
class ChargingStation {
  final String id;
  final String name;
  final String network;
  final double latitude;
  final double longitude;

  const ChargingStation({
    required this.id,
    required this.name,
    required this.network,
    required this.latitude,
    required this.longitude,
  });

  bool isInFront(double heading, double stationBearing, {double toleranceDeg = 45}) {
    final diff = ((stationBearing - heading) + 360) % 360;
    return diff <= toleranceDeg || diff >= (360 - toleranceDeg);
  }
}
```

---

### Layer 2: Use Case Tests (Mock-Repository via mocktail)

```dart
// 🔴 RED — test/domain/use_cases/find_nearest_station_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:kia_charge_nav/domain/repositories/i_charging_repository.dart';
import 'package:kia_charge_nav/domain/use_cases/find_nearest_station.dart';

class MockChargingRepository extends Mock implements IChargingRepository {}

void main() {
  late MockChargingRepository mockRepo;
  late FindNearestStation useCase;

  setUp(() {
    mockRepo = MockChargingRepository();
    useCase = FindNearestStation(mockRepo);
  });

  test('returns nearest station in heading direction', () async {
    final stations = [stationAhead, stationBehind];
    when(() => mockRepo.findNearby(
      latitude: any(named: 'latitude'),
      longitude: any(named: 'longitude'),
      heading: any(named: 'heading'),
      network: any(named: 'network'),
    )).thenAnswer((_) async => stations);

    final result = await useCase.execute(
      latitude: 50.0, longitude: 8.6,
      heading: 0, network: 'Shell Recharge',
    );

    expect(result, equals(stationAhead));
  });

  test('returns null when no stations found', () async {
    when(() => mockRepo.findNearby(
      latitude: any(named: 'latitude'),
      longitude: any(named: 'longitude'),
      heading: any(named: 'heading'),
      network: any(named: 'network'),
    )).thenAnswer((_) async => []);

    final result = await useCase.execute(
      latitude: 50.0, longitude: 8.6,
      heading: 0, network: 'Shell Recharge',
    );

    expect(result, isNull);
  });
}
```

---

### Layer 3: Repository Tests (Mock-DataSource)

```dart
// 🔴 RED — test/data/repositories/charging_repository_impl_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:kia_charge_nav/data/datasources/open_charge_map_datasource.dart';
import 'package:kia_charge_nav/data/repositories/charging_repository_impl.dart';

class MockOpenChargeMapDataSource extends Mock implements OpenChargeMapDataSource {}

void main() {
  late MockOpenChargeMapDataSource mockSource;
  late ChargingRepositoryImpl repository;

  setUp(() {
    mockSource = MockOpenChargeMapDataSource();
    repository = ChargingRepositoryImpl(mockSource);
  });

  test('converts DTOs to domain models', () async {
    when(() => mockSource.fetchNearby(
      latitude: any(named: 'latitude'),
      longitude: any(named: 'longitude'),
      operatorName: any(named: 'operatorName'),
    )).thenAnswer((_) async => [testDto]);

    final result = await repository.findNearby(
      latitude: 50.0, longitude: 8.6,
      heading: 0, network: 'Shell Recharge',
    );

    expect(result.first, isA<ChargingStation>());
    expect(result.first.name, equals('Shell A5'));
  });
}
```

---

### Layer 4: Controller Tests (Riverpod ProviderContainer)

```dart
// 🔴 RED — test/presentation/controllers/home_controller_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:riverpod/riverpod.dart';
import 'package:kia_charge_nav/domain/use_cases/find_nearest_station.dart';
import 'package:kia_charge_nav/domain/use_cases/send_destination.dart';
import 'package:kia_charge_nav/presentation/controllers/home_controller.dart';

class MockFindNearestStation extends Mock implements FindNearestStation {}
class MockSendDestination extends Mock implements SendDestination {}

void main() {
  late MockFindNearestStation mockFind;
  late MockSendDestination mockSend;
  late ProviderContainer container;

  setUp(() {
    mockFind = MockFindNearestStation();
    mockSend = MockSendDestination();
    container = ProviderContainer(overrides: [
      findNearestStationProvider.overrideWithValue(mockFind),
      sendDestinationProvider.overrideWithValue(mockSend),
    ]);
  });

  tearDown(() => container.dispose());

  test('state contains found station after findAndSend', () async {
    when(() => mockFind.execute(
      latitude: any(named: 'latitude'),
      longitude: any(named: 'longitude'),
      heading: any(named: 'heading'),
      network: any(named: 'network'),
    )).thenAnswer((_) async => testStation);
    when(() => mockSend.execute(any())).thenAnswer((_) async {});

    await container.read(homeControllerProvider.notifier).findAndSend('Shell Recharge');

    final state = container.read(homeControllerProvider);
    expect(state.value, equals(testStation));
  });

  test('state is error when no station found', () async {
    when(() => mockFind.execute(
      latitude: any(named: 'latitude'),
      longitude: any(named: 'longitude'),
      heading: any(named: 'heading'),
      network: any(named: 'network'),
    )).thenAnswer((_) async => null);

    await container.read(homeControllerProvider.notifier).findAndSend('Shell Recharge');

    final state = container.read(homeControllerProvider);
    expect(state.hasError, isTrue);
  });
}
```

---

### Layer 5: Widget Tests

```dart
// 🔴 RED — test/presentation/widgets/mic_button_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:kia_charge_nav/presentation/widgets/mic_button.dart';

void main() {
  testWidgets('MicButton ruft onPressed auf', (tester) async {
    bool pressed = false;

    await tester.pumpWidget(
      MaterialApp(
        home: Scaffold(
          body: MicButton(onPressed: () => pressed = true),
        ),
      ),
    );

    await tester.tap(find.byType(MicButton));
    await tester.pump();

    expect(pressed, isTrue);
  });

  testWidgets('MicButton zeigt Lade-Indikator wenn isLoading', (tester) async {
    await tester.pumpWidget(
      MaterialApp(
        home: Scaffold(
          body: MicButton(onPressed: () {}, isLoading: true),
        ),
      ),
    );

    expect(find.byType(CircularProgressIndicator), findsOneWidget);
  });
}
```

---

### flutter-solid → TDD-Brücke

- Domain-Tests: Kein `import 'package:flutter/...'` — reines Dart, kein Framework
- `MockChargingRepository extends Mock implements IChargingRepository` — Beweis für Dependency Inversion
- Use-Case-Tests brauchen minimal Setup → SRP eingehalten
- Controller-Tests über `ProviderContainer` mit Overrides — kein echter HTTP-Call
- Widget-Tests nur über Callbacks + `find.byType` — keine Implementierungsdetails
- Faustregel: Braucht ein Use-Case-Test mehr als 2 Mocks → wahrscheinlich SRP-Verletzung

---

## Verbote (STRIKT — keine Ausnahmen)

- ❌ Produktionscode schreiben bevor ein Test existiert
- ❌ Einen Test grün machen durch Anpassung des Tests (statt des Codes)
- ❌ Mehr Produktionscode als nötig für GREEN schreiben
- ❌ Refactoring überspringen ("läuft ja")
- ❌ Tests nachträglich schreiben und als TDD verkaufen

## Integration mit den SOLID-Skills

Dieser Skill setzt voraus, dass **python-solid**, **vue-solid** und/oder **flutter-solid** gleichzeitig aktiv sind. Die dort definierten OO- und Architekturregeln gelten uneingeschränkt — sie werden hier nicht wiederholt, sondern vorausgesetzt.

TDD und SOLID verstärken sich gegenseitig:
- **Schwer testbarer Code = SOLID-Verletzung** (z.B. God Class, fehlende Interfaces, hartcodierte Abhängigkeiten)
- **Einfach testbarer Code = Beweis für gutes OO-Design**

Wenn ein Test zu viel Setup braucht oder schwer zu schreiben ist → das ist kein Test-Problem, sondern ein Design-Problem. Dann zuerst das Design korrigieren (SOLID), dann den Test schreiben.

### python-solid → TDD-Brücke
- Domain-Tests: Kein Framework, nur Python-Standardbibliothek + Pydantic
- Application-Tests: `InMemoryRepository` implementiert das `Protocol`-Interface — Beweis für Dependency Inversion
- Infrastructure-Tests: SQLite in-memory oder TestContainers
- Testdoubles (InMemory-Implementierungen) sind Bürger erster Klasse

### vue-solid → TDD-Brücke
- Model/Service-Tests: Kein `import { ref } from 'vue'` — reines TypeScript, keine Vue-Kopplung
- `MockApiClient implements IApiClient` — Beweis für Dependency Inversion
- Composable-Tests bestätigen: Keine Business-Logik im Adapter
- Komponenten-Tests nur über Props + `data-testid` — keine Implementierungsdetails
- Faustregel: Je mehr Mock-Setup ein Service-Test braucht, desto klarer die SRP-Verletzung
