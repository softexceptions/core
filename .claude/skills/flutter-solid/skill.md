---
name: flutter-solid
description: |
  SOLID principles for Flutter + Dart projects. Applies Clean Architecture with three distinct layers: Domain (models, abstract repositories, use cases), Data (repository implementations, data sources, DTOs), and Presentation (thin Widgets, Riverpod Controllers). Uses abstract classes as interfaces (Dart has no interface keyword), constructor injection, and get_it for DI. Widgets are always thin — all logic lives in Use Cases and Repositories.
---

# Flutter + Dart SOLID Architecture

You are now operating as a senior Flutter + Dart engineer who writes production-grade code following SOLID principles, Clean Architecture, and professional software design patterns.

## Core Principle

**Clean Architecture with SOLID for Flutter — three distinct layers.**

1. **Domain Layer** — Core business logic (no Flutter, no http, no external packages)
2. **Data Layer** — Repository implementations, API clients, DTOs
3. **Presentation Layer** — Thin Widgets, Riverpod Controllers (state only, no logic)

**Dart has no `interface` keyword. Use `abstract class` instead.**

---

## Architecture Rules

### Layer 1: Domain (Core Business Logic)

**Location:** `lib/domain/`

**Components:**
- **Models:** Immutable domain entities (plain Dart classes or `freezed`)
- **Abstract Repositories:** `abstract class IRepository` — define contracts, no implementation
- **Use Cases:** One class per operation (`FindNearestStation`, `SendDestination`)

**Rules:**
- NEVER import `package:flutter/...`, `package:http/...`, or any external API package
- Only pure Dart — standard library allowed
- All business rules live here
- Abstract classes define the contract (DIP)

**Example:**
```dart
// domain/models/charging_station.dart
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

  // Business logic: is this station in front of us?
  bool isInFront(double heading, double stationBearing, {double toleranceDeg = 45}) {
    final diff = ((stationBearing - heading) + 360) % 360;
    return diff <= toleranceDeg || diff >= (360 - toleranceDeg);
  }
}

// domain/repositories/i_charging_repository.dart
abstract class IChargingRepository {
  Future<List<ChargingStation>> findNearby({
    required double latitude,
    required double longitude,
    required double heading,
    required String network,
    int radiusKm = 15,
  });
}

// domain/repositories/i_kia_repository.dart
abstract class IKiaRepository {
  Future<void> sendDestination({
    required double latitude,
    required double longitude,
    required String name,
  });
}

// domain/use_cases/find_nearest_station.dart
class FindNearestStation {
  final IChargingRepository _chargingRepository;

  FindNearestStation(this._chargingRepository);

  Future<ChargingStation?> execute({
    required double latitude,
    required double longitude,
    required double heading,
    required String network,
  }) async {
    final stations = await _chargingRepository.findNearby(
      latitude: latitude,
      longitude: longitude,
      heading: heading,
      network: network,
    );
    // Return single best result — nearest in heading direction
    return stations.isEmpty ? null : stations.first;
  }
}

// domain/use_cases/send_destination.dart
class SendDestination {
  final IKiaRepository _kiaRepository;

  SendDestination(this._kiaRepository);

  Future<void> execute(ChargingStation station) async {
    await _kiaRepository.sendDestination(
      latitude: station.latitude,
      longitude: station.longitude,
      name: station.name,
    );
  }
}
```

---

### Layer 2: Data (Implementations & External APIs)

**Location:** `lib/data/`

**Components:**
- **Repository Implementations:** Implement domain interfaces
- **Data Sources:** HTTP clients for external APIs
- **DTOs:** JSON parsing — never leak into domain layer

**Rules:**
- Implements domain abstract classes
- Handles JSON parsing, HTTP errors, retries
- Converts DTOs → Domain models (never return raw DTOs)
- No business logic — only data access

**Example:**
```dart
// data/dtos/charging_station_dto.dart
class ChargingStationDto {
  final String id;
  final String name;
  final String operatorTitle;
  final double latitude;
  final double longitude;

  const ChargingStationDto({
    required this.id,
    required this.name,
    required this.operatorTitle,
    required this.latitude,
    required this.longitude,
  });

  factory ChargingStationDto.fromJson(Map<String, dynamic> json) {
    return ChargingStationDto(
      id: json['ID'].toString(),
      name: json['AddressInfo']['Title'] ?? '',
      operatorTitle: json['OperatorInfo']?['Title'] ?? '',
      latitude: json['AddressInfo']['Latitude'],
      longitude: json['AddressInfo']['Longitude'],
    );
  }

  // Convert DTO → Domain model (only in data layer)
  ChargingStation toDomain() {
    return ChargingStation(
      id: id,
      name: name,
      network: operatorTitle,
      latitude: latitude,
      longitude: longitude,
    );
  }
}

// data/datasources/open_charge_map_datasource.dart
class OpenChargeMapDataSource {
  final http.Client _client;
  static const _baseUrl = 'https://api.openchargemap.io/v3';

  OpenChargeMapDataSource(this._client);

  Future<List<ChargingStationDto>> fetchNearby({
    required double latitude,
    required double longitude,
    required String operatorName,
    int radiusKm = 15,
  }) async {
    final uri = Uri.parse('$_baseUrl/poi/').replace(queryParameters: {
      'latitude': latitude.toString(),
      'longitude': longitude.toString(),
      'distance': radiusKm.toString(),
      'operatorname': operatorName,
      'maxresults': '20',
      'output': 'json',
    });

    final response = await _client.get(uri);
    if (response.statusCode != 200) {
      throw Exception('OpenChargeMap API error: ${response.statusCode}');
    }

    final List<dynamic> data = jsonDecode(response.body);
    return data.map((json) => ChargingStationDto.fromJson(json)).toList();
  }
}

// data/repositories/charging_repository_impl.dart
class ChargingRepositoryImpl implements IChargingRepository {
  final OpenChargeMapDataSource _dataSource;

  ChargingRepositoryImpl(this._dataSource);

  @override
  Future<List<ChargingStation>> findNearby({
    required double latitude,
    required double longitude,
    required double heading,
    required String network,
    int radiusKm = 15,
  }) async {
    final dtos = await _dataSource.fetchNearby(
      latitude: latitude,
      longitude: longitude,
      operatorName: network,
      radiusKm: radiusKm,
    );

    final stations = dtos.map((dto) => dto.toDomain()).toList();

    // Sort: nearest in heading direction first
    stations.sort((a, b) {
      final bearingA = _bearing(latitude, longitude, a.latitude, a.longitude);
      final bearingB = _bearing(latitude, longitude, b.latitude, b.longitude);
      final diffA = ((bearingA - heading) + 360) % 360;
      final diffB = ((bearingB - heading) + 360) % 360;
      return diffA.compareTo(diffB);
    });

    return stations;
  }

  double _bearing(double lat1, double lon1, double lat2, double lon2) {
    final dLon = (lon2 - lon1) * pi / 180;
    final lat1Rad = lat1 * pi / 180;
    final lat2Rad = lat2 * pi / 180;
    final y = sin(dLon) * cos(lat2Rad);
    final x = cos(lat1Rad) * sin(lat2Rad) - sin(lat1Rad) * cos(lat2Rad) * cos(dLon);
    return (atan2(y, x) * 180 / pi + 360) % 360;
  }
}
```

---

### Layer 3: Presentation (Widgets + Riverpod Controllers)

**Location:** `lib/presentation/`

**Components:**
- **Pages:** Thin — only wiring, no logic
- **Widgets:** Thin — only UI, delegate everything
- **Controllers (Riverpod):** State management only — delegate to Use Cases

**Rules:**
- Widgets contain NO business logic, NO direct API calls
- Controllers delegate all operations to Use Cases
- Controllers manage loading/error state
- Inject dependencies via `ref.read` (Riverpod) or constructor

**Example:**
```dart
// presentation/controllers/home_controller.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

@riverpod
class HomeController extends _$HomeController {
  @override
  AsyncValue<ChargingStation?> build() => const AsyncValue.data(null);

  Future<void> findAndSend(String network) async {
    state = const AsyncValue.loading();

    try {
      // Get dependencies via DI
      final findNearest = ref.read(findNearestStationProvider);
      final sendDestination = ref.read(sendDestinationProvider);
      final location = ref.read(locationServiceProvider);

      // Get current position + heading
      final position = await location.getCurrentPosition();

      // Use Case 1: find station
      final station = await findNearest.execute(
        latitude: position.latitude,
        longitude: position.longitude,
        heading: position.heading,
        network: network,
      );

      if (station == null) {
        state = AsyncValue.error('Keine Station gefunden', StackTrace.current);
        return;
      }

      // Use Case 2: send to car
      await sendDestination.execute(station);

      state = AsyncValue.data(station);
    } catch (e, stack) {
      state = AsyncValue.error(e, stack);
    }
  }
}

// presentation/pages/home_page.dart
class HomePage extends ConsumerWidget {
  const HomePage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(homeControllerProvider);

    return Scaffold(
      body: SafeArea(
        child: Column(
          children: [
            // Status feedback
            state.when(
              data: (station) => station != null
                  ? StatusCard(station: station)
                  : const SizedBox.shrink(),
              loading: () => const LoadingOverlay(),
              error: (e, _) => ErrorBanner(message: e.toString()),
            ),
            const Spacer(),
            // Network buttons + mic button
            NetworkButtonGrid(
              onNetworkSelected: (network) =>
                  ref.read(homeControllerProvider.notifier).findAndSend(network),
            ),
            const MicButton(),
          ],
        ),
      ),
    );
  }
}
```

---

## SOLID in Dart

### Single Responsibility (SRP)

```dart
// ✅ GOOD — each class one responsibility
class FindNearestStation { /* only finds stations */ }
class SendDestination    { /* only sends to car */ }
class ChargingStation    { /* only represents station data */ }

// ❌ BAD — god class
class ChargingManager {
  Future<void> findAndValidateAndSendAndLog() { ... }
}
```

### Open/Closed (OCP)

```dart
// ✅ GOOD — extend via new implementations
abstract class IChargingRepository { ... }
class OpenChargeMapRepository implements IChargingRepository { ... }
class GoingElectricRepository  implements IChargingRepository { ... }
// Add new source → no existing code changes

// ❌ BAD — modify existing code for each source
if (source == 'opencharge') { ... }
else if (source == 'goingelectric') { ... }
```

### Liskov Substitution (LSP)

```dart
// ✅ GOOD — all implementations interchangeable
abstract class IKiaRepository { Future<void> sendDestination(...); }

class KiaConnectRepository    implements IKiaRepository { ... } // real
class MockKiaRepository       implements IKiaRepository { ... } // test

// Use Case works with BOTH
final useCase = SendDestination(MockKiaRepository()); // for tests
```

### Interface Segregation (ISP)

```dart
// ✅ GOOD — small focused interfaces
abstract class ILocationReader {
  Future<Position> getCurrentPosition();
}

abstract class ILocationStream {
  Stream<Position> watchPosition();
}

// Controllers only depend on what they need
class HomeController {
  HomeController(this._locationReader); // only needs read, not stream
  final ILocationReader _locationReader;
}
```

### Dependency Inversion (DIP)

```dart
// ✅ GOOD — depends on abstraction
class FindNearestStation {
  FindNearestStation(this._repository); // ← abstract class
  final IChargingRepository _repository;
}

// ❌ BAD — depends on concrete class
class FindNearestStation {
  final repo = OpenChargeMapRepository(); // ← concrete, hardcoded
}
```

---

## Dependency Injection (get_it)

```dart
// lib/di/injection.dart
import 'package:get_it/get_it.dart';

final getIt = GetIt.instance;

void setupDependencies() {
  // Data sources
  getIt.registerLazySingleton(() => http.Client());
  getIt.registerLazySingleton(() => OpenChargeMapDataSource(getIt()));
  getIt.registerLazySingleton(() => KiaApiDataSource(getIt()));

  // Repositories
  getIt.registerLazySingleton<IChargingRepository>(
    () => ChargingRepositoryImpl(getIt()),
  );
  getIt.registerLazySingleton<IKiaRepository>(
    () => KiaRepositoryImpl(getIt()),
  );

  // Use Cases
  getIt.registerFactory(() => FindNearestStation(getIt()));
  getIt.registerFactory(() => SendDestination(getIt()));
}
```

---

## File Structure

```
lib/
├── main.dart                          # App entry + DI setup
├── domain/
│   ├── models/
│   │   └── charging_station.dart      # Domain entity
│   ├── repositories/
│   │   ├── i_charging_repository.dart # Abstract (interface)
│   │   └── i_kia_repository.dart      # Abstract (interface)
│   └── use_cases/
│       ├── find_nearest_station.dart  # One class per operation
│       └── send_destination.dart
├── data/
│   ├── dtos/
│   │   └── charging_station_dto.dart  # JSON parsing
│   ├── datasources/
│   │   ├── open_charge_map_datasource.dart
│   │   └── kia_api_datasource.dart
│   └── repositories/
│       ├── charging_repository_impl.dart
│       └── kia_repository_impl.dart
├── presentation/
│   ├── controllers/
│   │   └── home_controller.dart       # Riverpod — state only
│   ├── pages/
│   │   └── home_page.dart             # Thin wiring
│   └── widgets/
│       ├── mic_button.dart
│       ├── network_button_grid.dart
│       ├── status_card.dart
│       └── loading_overlay.dart
└── di/
    └── injection.dart                 # get_it setup
```

---

## Best Practices

### ✅ DO:
1. `abstract class IRepository` for every external dependency
2. Constructor injection everywhere
3. Use Cases: one class, one public method `execute()`
4. Controllers delegate to Use Cases — no business logic
5. DTOs only in data layer — convert to domain models before returning
6. Widgets are stateless when possible
7. Test Use Cases with mock repositories

### ❌ DON'T:
1. No Flutter imports in domain layer
2. No `http` calls in presentation layer
3. No business logic in Widgets or Controllers
4. No `static` service access — always inject
5. No DTOs leaking into domain or presentation

---

## When to use this Skill

Use `/flutter-solid` when:
- Creating new Dart classes, repositories, or use cases
- Writing Riverpod controllers or providers
- Reviewing Flutter code for SOLID violations
- Planning feature architecture
- Adding a new data source (new API, local storage)
