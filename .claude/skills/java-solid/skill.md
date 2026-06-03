---
name: java-solid
description: |
  SOLID principles for Java projects. Applies Clean Architecture with four distinct layers:
  Domain (entities, value objects, interfaces, use cases — no frameworks), Application (services,
  use cases), Infrastructure (repository implementations, external APIs, persistence), and
  Presentation (REST controllers, DTOs). Uses interface keyword natively, constructor injection
  (not field injection), Spring Boot for DI, Records for value objects (Java 16+), and
  custom exceptions for domain errors.

  Use /java-solid when:
  - Creating new Java classes, interfaces, services, or repositories
  - Reviewing Java code for SOLID violations
  - Planning package structure for a new Java/Spring Boot project
  - Adding a new data source or external dependency
---

# Java SOLID Architecture

Du bist jetzt ein Senior Java-Entwickler, der produktionsreifen Code nach SOLID-Prinzipien und Clean Architecture schreibt.

## Core Principle

**Clean Architecture mit SOLID für Java — vier distinkte Schichten.**

1. **Domain Layer** — Kernlogik (keine Spring-Annotationen, keine JPA-Annotationen)
2. **Application Layer** — Use Cases / Services (orchestriert Domain)
3. **Infrastructure Layer** — Repository-Implementierungen, externe APIs, JPA-Entities
4. **Presentation Layer** — REST-Controller, Request/Response-DTOs

**Java hat `interface` als natives Keyword — nutze es konsequent.**

---

## Architecture Rules

### Layer 1: Domain (Kernlogik)

**Location:** `src/main/java/com/example/domain/`

**Komponenten:**
- **Entities:** Kern-Domänenobjekte mit Geschäftslogik
- **Value Objects:** `record` — unveränderlich, validiert im Konstruktor
- **Interfaces:** `interface IRepository` — definiert Vertrag, keine Implementierung
- **Use Cases:** Eine Klasse pro Operation, ein `execute()`-Einstiegspunkt
- **Domain Exceptions:** Klare Fehlertypen ohne Framework-Abhängigkeit

**Regeln:**
- NIEMALS Spring-Annotationen (`@Service`, `@Repository`, `@Autowired`)
- NIEMALS JPA-Annotationen (`@Entity`, `@Column`)
- NIEMALS `javax.persistence` oder `jakarta.persistence` importieren
- Nur Java-Standardbibliothek + eigene Domain-Klassen

**Beispiel:**

```java
// domain/model/ChargingStation.java
public class ChargingStation {
    private final String id;
    private final String name;
    private final String network;
    private final double latitude;
    private final double longitude;

    public ChargingStation(String id, String name, String network,
                           double latitude, double longitude) {
        this.id = id;
        this.name = name;
        this.network = network;
        this.latitude = latitude;
        this.longitude = longitude;
    }

    // Geschäftslogik gehört zur Entität
    public boolean isInFront(double heading, double stationBearing) {
        double diff = ((stationBearing - heading) + 360) % 360;
        return diff <= 45 || diff >= 315;
    }

    public String getId() { return id; }
    public String getName() { return name; }
    public String getNetwork() { return network; }
    public double getLatitude() { return latitude; }
    public double getLongitude() { return longitude; }
}

// domain/model/Pin.java — Value Object mit record
public record Pin(String value) {
    public Pin {
        if (value == null || !value.matches("\\d{6}")) {
            throw new IllegalArgumentException("PIN muss 6 Ziffern haben: " + value);
        }
    }
}

// domain/exception/StationNotFoundException.java
public class StationNotFoundException extends RuntimeException {
    public StationNotFoundException(String network, double radius) {
        super("Keine " + network + "-Station in " + radius + " km Fahrtrichtung gefunden");
    }
}

// domain/repository/IChargingStationRepository.java
public interface IChargingStationRepository {
    List<ChargingStation> findNearby(double latitude, double longitude,
                                      double heading, String network,
                                      double radiusKm);
}

// domain/usecase/FindNearestStation.java
public class FindNearestStation {

    private final IChargingStationRepository repository;

    public FindNearestStation(IChargingStationRepository repository) {
        this.repository = repository;
    }

    public ChargingStation execute(double latitude, double longitude,
                                   double heading, String network) {
        var stations = repository.findNearby(latitude, longitude, heading, network, 15.0);
        return stations.stream()
                .findFirst()
                .orElseThrow(() -> new StationNotFoundException(network, 15.0));
    }
}
```

---

### Layer 2: Application (Services / Orchestrierung)

**Location:** `src/main/java/com/example/application/`

**Regeln:**
- Orchestriert Use Cases — keine eigene Geschäftslogik
- Darf Spring-Annotationen haben (`@Service`, `@Transactional`)
- Kennt nur Domain-Interfaces, nie Infrastructure-Klassen direkt

```java
// application/service/ChargingService.java
@Service
@Transactional(readOnly = true)
public class ChargingService {

    private final FindNearestStation findNearestStation;
    private final ISendDestinationService sendDestinationService;

    // Konstruktor-Injection — NICHT @Autowired auf Felder
    public ChargingService(IChargingStationRepository repository,
                           ISendDestinationService sendDestinationService) {
        this.findNearestStation = new FindNearestStation(repository);
        this.sendDestinationService = sendDestinationService;
    }

    public ChargingStation findAndSend(double latitude, double longitude,
                                       double heading, String network) {
        var station = findNearestStation.execute(latitude, longitude, heading, network);
        sendDestinationService.send(station);
        return station;
    }
}
```

---

### Layer 3: Infrastructure (Implementierungen & I/O)

**Location:** `src/main/java/com/example/infrastructure/`

**Regeln:**
- Implementiert Domain-Interfaces (`implements IChargingStationRepository`)
- Enthält JPA-Entities, HTTP-Clients, externe API-Calls
- Konvertiert DTOs/JPA-Entities → Domain-Modelle
- Keine Geschäftslogik — nur Datenzugriff

```java
// infrastructure/repository/OpenChargeMapRepository.java
@Repository
public class OpenChargeMapRepository implements IChargingStationRepository {

    private final RestTemplate restTemplate;
    private final String apiBaseUrl;

    public OpenChargeMapRepository(RestTemplate restTemplate,
                                   @Value("${ocm.api.url}") String apiBaseUrl) {
        this.restTemplate = restTemplate;
        this.apiBaseUrl = apiBaseUrl;
    }

    @Override
    public List<ChargingStation> findNearby(double latitude, double longitude,
                                             double heading, String network,
                                             double radiusKm) {
        var url = apiBaseUrl + "/poi/?latitude={lat}&longitude={lon}&distance={r}&operatorname={n}";
        var dtos = restTemplate.getForObject(url, StationDto[].class,
                latitude, longitude, radiusKm, network);

        return Arrays.stream(dtos != null ? dtos : new StationDto[0])
                .map(StationDto::toDomain)   // DTO → Domain
                .collect(Collectors.toList());
    }
}

// infrastructure/dto/StationDto.java
public record StationDto(String id, String name, String operator,
                         double latitude, double longitude) {
    // DTO → Domain-Modell: nur in Infrastructure erlaubt
    public ChargingStation toDomain() {
        return new ChargingStation(id, name, operator, latitude, longitude);
    }
}
```

---

### Layer 4: Presentation (Controller + DTOs)

**Location:** `src/main/java/com/example/presentation/`

**Regeln:**
- Thin — nur HTTP-Mapping, keine Logik
- Eigene Request/Response-DTOs — kein Domain-Objekt direkt zurückgeben
- Ruft Application Services auf

```java
// presentation/controller/ChargingController.java
@RestController
@RequestMapping("/api/charging")
public class ChargingController {

    private final ChargingService chargingService;

    public ChargingController(ChargingService chargingService) {
        this.chargingService = chargingService;
    }

    @PostMapping("/find-and-send")
    public ResponseEntity<StationResponse> findAndSend(@RequestBody FindStationRequest request) {
        var station = chargingService.findAndSend(
                request.latitude(), request.longitude(),
                request.heading(), request.network());
        return ResponseEntity.ok(StationResponse.from(station));
    }
}

// presentation/dto/FindStationRequest.java
public record FindStationRequest(double latitude, double longitude,
                                  double heading, String network) {}

// presentation/dto/StationResponse.java
public record StationResponse(String id, String name, String network) {
    public static StationResponse from(ChargingStation station) {
        return new StationResponse(station.getId(), station.getName(), station.getNetwork());
    }
}
```

---

## SOLID in Java

### Single Responsibility (SRP)

```java
// ✅ GUT — jede Klasse eine Aufgabe
class FindNearestStation { /* nur Stationen suchen */ }
class SendDestination     { /* nur ans Auto senden */ }
class ChargingStation     { /* nur Stations-Daten + Logik */ }

// ❌ SCHLECHT — God-Klasse
class ChargingManager {
    public void findValidateAndSendAndLog() { ... }
}
```

### Open/Closed (OCP)

```java
// ✅ GUT — Erweiterung via neue Implementierung
interface IChargingStationRepository { ... }
class OpenChargeMapRepository implements IChargingStationRepository { ... }
class GoingElectricRepository  implements IChargingStationRepository { ... }
// Neue Quelle → kein bestehender Code ändert sich

// ❌ SCHLECHT
if (source.equals("ocm")) { ... }
else if (source.equals("ge")) { ... }
```

### Liskov Substitution (LSP)

```java
// ✅ GUT — alle Implementierungen austauschbar
interface IChargingStationRepository { ... }
class OpenChargeMapRepository implements IChargingStationRepository { ... } // Produktion
class InMemoryChargingRepository  implements IChargingStationRepository { ... } // Tests
// Use Case funktioniert mit BEIDEN
```

### Interface Segregation (ISP)

```java
// ✅ GUT — kleine, fokussierte Interfaces
interface IStationFinder {
    List<ChargingStation> findNearby(double lat, double lon, double heading, String network, double radius);
}

interface IStationSender {
    void send(ChargingStation station);
}

// ❌ SCHLECHT — Alles in einem Interface
interface IChargingApi {
    List<ChargingStation> findNearby(...);
    void send(ChargingStation station);
    void updateConfig(Config config);   // FindNearestStation braucht das nicht
}
```

### Dependency Inversion (DIP)

```java
// ✅ GUT — abhängt vom Interface
public class FindNearestStation {
    private final IChargingStationRepository repository; // Interface

    public FindNearestStation(IChargingStationRepository repository) {
        this.repository = repository; // Konstruktor-Injection
    }
}

// ❌ SCHLECHT — direkter Typ, hardcoded
public class FindNearestStation {
    private final OpenChargeMapRepository repository = new OpenChargeMapRepository(); // konkret
}
```

---

## Dependency Injection (Spring Boot)

```java
// Konstruktor-Injection (IMMER bevorzugen)
@Service
public class ChargingService {
    private final IChargingStationRepository repository;
    private final ISendDestinationService sendService;

    public ChargingService(IChargingStationRepository repository,
                           ISendDestinationService sendService) {
        this.repository = repository;
        this.sendService = sendService;
    }
}

// ❌ NIEMALS Field-Injection
@Service
public class ChargingService {
    @Autowired
    private IChargingStationRepository repository; // nicht testbar, nicht final
}
```

---

## File Structure

```
src/main/java/com/example/
├── domain/
│   ├── exception/
│   │   └── StationNotFoundException.java
│   ├── model/
│   │   ├── ChargingStation.java
│   │   └── Pin.java                     (record — Value Object)
│   ├── repository/
│   │   └── IChargingStationRepository.java  (interface)
│   └── usecase/
│       └── FindNearestStation.java
├── application/
│   └── service/
│       └── ChargingService.java
├── infrastructure/
│   ├── dto/
│   │   └── StationDto.java              (record + toDomain())
│   └── repository/
│       └── OpenChargeMapRepository.java
└── presentation/
    ├── controller/
    │   └── ChargingController.java
    └── dto/
        ├── FindStationRequest.java      (record)
        └── StationResponse.java        (record + from())
```

---

## Best Practices

### ✅ DO:
1. `interface IRepository` für jede externe Abhängigkeit
2. Konstruktor-Injection überall — kein `@Autowired` auf Felder
3. Use Cases: eine Klasse, eine `execute()`-Methode
4. `record` für Value Objects und DTOs (Java 16+)
5. DTOs nur in Presentation- und Infrastructure-Layer
6. Domain-Exceptions für Geschäftsfehler (kein `RuntimeException` direkt)

### ❌ DON'T:
1. Kein Spring in Domain-Layer (`@Service`, `@Autowired`, `@Entity`)
2. Kein `new` für Dependencies — immer injizieren
3. Keine Geschäftslogik in Controllern oder Repository-Implementierungen
4. Kein Domain-Objekt direkt als HTTP-Response zurückgeben
5. Kein `static` für Abhängigkeiten
