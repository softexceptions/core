---
name: csharp-solid
description: |
  SOLID principles for C# projects. Applies Clean Architecture with four distinct layers:
  Domain (entities, value objects, interfaces, use cases — no framework dependencies), Application
  (services, use cases), Infrastructure (repository implementations, EF Core, external APIs),
  and Presentation (ASP.NET Core controllers, DTOs, minimal APIs). Uses interface keyword natively
  (I-prefix convention), constructor injection via Microsoft.Extensions.DependencyInjection,
  records for value objects (C# 9+), and custom exceptions or Result<T> for domain errors.

  Use /csharp-solid when:
  - Creating new C# classes, interfaces, services, or repositories
  - Reviewing C# code for SOLID violations
  - Planning namespace structure for a new C#/ASP.NET Core project
  - Adding a new data source or external dependency
---

# C# SOLID Architecture

Du bist jetzt ein Senior C#-Entwickler, der produktionsreifen Code nach SOLID-Prinzipien und Clean Architecture schreibt.

## Core Principle

**Clean Architecture mit SOLID für C# — vier distinkte Schichten.**

1. **Domain Layer** — Kernlogik (keine ASP.NET-, EF-Core-, oder Framework-Abhängigkeiten)
2. **Application Layer** — Use Cases / Services (orchestriert Domain)
3. **Infrastructure Layer** — Repository-Implementierungen, EF Core, externe APIs
4. **Presentation Layer** — ASP.NET Core Controller, Minimal APIs, Request/Response-DTOs

**C# hat `interface` als natives Keyword — `I`-Präfix ist Konvention (z.B. `IRepository`).**

---

## Architecture Rules

### Layer 1: Domain (Kernlogik)

**Location:** `src/Domain/`

**Komponenten:**
- **Entities:** Kern-Domänenobjekte mit Geschäftslogik
- **Value Objects:** `record` — unveränderlich, validiert im Konstruktor
- **Interfaces:** `interface IRepository` — definiert Vertrag, keine Implementierung
- **Use Cases:** Eine Klasse pro Operation, ein `ExecuteAsync()`-Einstiegspunkt
- **Domain Exceptions:** Klare Fehlertypen ohne Framework-Abhängigkeit

**Regeln:**
- NIEMALS `[ApiController]`, `[HttpGet]`, EF-Core-Attribute
- NIEMALS `using Microsoft.AspNetCore` oder `using Microsoft.EntityFrameworkCore`
- Nur `System.*` und eigene Domain-Klassen

**Beispiel:**

```csharp
// Domain/Models/ChargingStation.cs
public class ChargingStation
{
    public string Id { get; }
    public string Name { get; }
    public string Network { get; }
    public double Latitude { get; }
    public double Longitude { get; }

    public ChargingStation(string id, string name, string network,
                           double latitude, double longitude)
    {
        Id = id;
        Name = name;
        Network = network;
        Latitude = latitude;
        Longitude = longitude;
    }

    // Geschäftslogik gehört zur Entität
    public bool IsInFront(double heading, double stationBearing)
    {
        var diff = ((stationBearing - heading) + 360) % 360;
        return diff <= 45 || diff >= 315;
    }
}

// Domain/Models/Pin.cs — Value Object mit record
public record Pin
{
    public string Value { get; }

    public Pin(string value)
    {
        if (string.IsNullOrEmpty(value) || !System.Text.RegularExpressions.Regex.IsMatch(value, @"^\d{6}$"))
            throw new ArgumentException($"PIN muss 6 Ziffern haben: {value}");
        Value = value;
    }
}

// Domain/Exceptions/StationNotFoundException.cs
public class StationNotFoundException : Exception
{
    public StationNotFoundException(string network, double radius)
        : base($"Keine {network}-Station in {radius} km Fahrtrichtung gefunden") { }
}

// Domain/Repositories/IChargingStationRepository.cs
public interface IChargingStationRepository
{
    Task<IReadOnlyList<ChargingStation>> FindNearbyAsync(
        double latitude, double longitude,
        double heading, string network,
        double radiusKm,
        CancellationToken ct = default);
}

// Domain/UseCases/FindNearestStation.cs
public class FindNearestStation
{
    private readonly IChargingStationRepository _repository;

    public FindNearestStation(IChargingStationRepository repository)
    {
        _repository = repository;
    }

    public async Task<ChargingStation> ExecuteAsync(
        double latitude, double longitude,
        double heading, string network,
        CancellationToken ct = default)
    {
        var stations = await _repository.FindNearbyAsync(
            latitude, longitude, heading, network, radiusKm: 15.0, ct);

        return stations.FirstOrDefault()
            ?? throw new StationNotFoundException(network, 15.0);
    }
}
```

---

### Layer 2: Application (Services / Orchestrierung)

**Location:** `src/Application/`

**Regeln:**
- Orchestriert Use Cases — keine eigene Geschäftslogik
- Darf `using Microsoft.Extensions.Logging` verwenden
- Kennt nur Domain-Interfaces, nie Infrastructure-Klassen direkt

```csharp
// Application/Services/ChargingService.cs
public class ChargingService : IChargingService
{
    private readonly FindNearestStation _findNearest;
    private readonly ISendDestinationService _sendService;

    // Konstruktor-Injection — NICHT Property Injection
    public ChargingService(
        IChargingStationRepository repository,
        ISendDestinationService sendService)
    {
        _findNearest = new FindNearestStation(repository);
        _sendService = sendService;
    }

    public async Task<ChargingStation> FindAndSendAsync(
        double latitude, double longitude,
        double heading, string network,
        CancellationToken ct = default)
    {
        var station = await _findNearest.ExecuteAsync(
            latitude, longitude, heading, network, ct);
        await _sendService.SendAsync(station, ct);
        return station;
    }
}
```

---

### Layer 3: Infrastructure (Implementierungen & I/O)

**Location:** `src/Infrastructure/`

**Regeln:**
- Implementiert Domain-Interfaces (`IChargingStationRepository`)
- Enthält EF-Core-Entities, HTTP-Clients, externe API-Calls
- Konvertiert DTOs/EF-Entities → Domain-Modelle
- Keine Geschäftslogik — nur Datenzugriff

```csharp
// Infrastructure/Repositories/OpenChargeMapRepository.cs
public class OpenChargeMapRepository : IChargingStationRepository
{
    private readonly HttpClient _httpClient;

    public OpenChargeMapRepository(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }

    public async Task<IReadOnlyList<ChargingStation>> FindNearbyAsync(
        double latitude, double longitude,
        double heading, string network,
        double radiusKm,
        CancellationToken ct = default)
    {
        var url = $"/v3/poi/?latitude={latitude}&longitude={longitude}" +
                  $"&distance={radiusKm}&operatorname={network}";

        var dtos = await _httpClient.GetFromJsonAsync<StationDto[]>(url, ct)
                   ?? Array.Empty<StationDto>();

        return dtos.Select(dto => dto.ToDomain()).ToList().AsReadOnly();
    }
}

// Infrastructure/Dtos/StationDto.cs
public record StationDto(string Id, string Name, string Operator,
                         double Latitude, double Longitude)
{
    // DTO → Domain: nur in Infrastructure erlaubt
    public ChargingStation ToDomain() =>
        new(Id, Name, Operator, Latitude, Longitude);
}
```

---

### Layer 4: Presentation (Controller + DTOs)

**Location:** `src/Presentation/` oder `src/Api/`

**Regeln:**
- Thin — nur HTTP-Mapping, keine Logik
- Eigene Request/Response-DTOs — kein Domain-Objekt direkt zurückgeben
- Ruft Application Services auf

```csharp
// Presentation/Controllers/ChargingController.cs
[ApiController]
[Route("api/[controller]")]
public class ChargingController : ControllerBase
{
    private readonly IChargingService _chargingService;

    public ChargingController(IChargingService chargingService)
    {
        _chargingService = chargingService;
    }

    [HttpPost("find-and-send")]
    public async Task<ActionResult<StationResponse>> FindAndSend(
        [FromBody] FindStationRequest request,
        CancellationToken ct)
    {
        var station = await _chargingService.FindAndSendAsync(
            request.Latitude, request.Longitude,
            request.Heading, request.Network, ct);

        return Ok(StationResponse.From(station));
    }
}

// Presentation/Dtos/FindStationRequest.cs
public record FindStationRequest(
    double Latitude, double Longitude,
    double Heading, string Network);

// Presentation/Dtos/StationResponse.cs
public record StationResponse(string Id, string Name, string Network)
{
    public static StationResponse From(ChargingStation s) =>
        new(s.Id, s.Name, s.Network);
}
```

---

## DI-Registrierung (Program.cs / Startup.cs)

```csharp
// Program.cs
builder.Services.AddScoped<IChargingStationRepository, OpenChargeMapRepository>();
builder.Services.AddScoped<ISendDestinationService, KiaConnectService>();
builder.Services.AddScoped<IChargingService, ChargingService>();

// HttpClient für externe APIs
builder.Services.AddHttpClient<OpenChargeMapRepository>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration["OCM:BaseUrl"]!);
});
```

---

## SOLID in C#

### Single Responsibility (SRP)

```csharp
// ✅ GUT — jede Klasse eine Aufgabe
class FindNearestStation { /* nur Stationen suchen */ }
class SendDestination     { /* nur ans Auto senden */ }
class ChargingStation     { /* nur Stations-Daten + Logik */ }

// ❌ SCHLECHT — God-Klasse
class ChargingManager {
    public async Task FindValidateAndSendAndLog() { ... }
}
```

### Open/Closed (OCP)

```csharp
// ✅ GUT — Erweiterung via neue Implementierung
interface IChargingStationRepository { ... }
class OpenChargeMapRepository : IChargingStationRepository { ... }
class GoingElectricRepository  : IChargingStationRepository { ... }
// Neue Quelle → kein bestehender Code ändert sich

// ❌ SCHLECHT
if (source == "ocm") { ... }
else if (source == "ge") { ... }
```

### Liskov Substitution (LSP)

```csharp
// ✅ GUT — alle Implementierungen austauschbar
IChargingStationRepository repo = new OpenChargeMapRepository(...); // Produktion
IChargingStationRepository mock = new Mock<IChargingStationRepository>().Object; // Tests
// FindNearestStation funktioniert mit BEIDEN
```

### Interface Segregation (ISP)

```csharp
// ✅ GUT — kleine, fokussierte Interfaces
public interface IStationFinder
{
    Task<IReadOnlyList<ChargingStation>> FindNearbyAsync(...);
}

public interface IStationSender
{
    Task SendAsync(ChargingStation station, CancellationToken ct);
}

// ❌ SCHLECHT — Alles in einem Interface
public interface IChargingApi
{
    Task<IReadOnlyList<ChargingStation>> FindNearbyAsync(...);
    Task SendAsync(ChargingStation station, CancellationToken ct);
    Task UpdateConfigAsync(Config config);  // FindNearestStation braucht das nicht
}
```

### Dependency Inversion (DIP)

```csharp
// ✅ GUT — abhängt vom Interface
public class FindNearestStation
{
    private readonly IChargingStationRepository _repository; // Interface

    public FindNearestStation(IChargingStationRepository repository)
        => _repository = repository; // Konstruktor-Injection
}

// ❌ SCHLECHT — direkter Typ, hardcoded
public class FindNearestStation
{
    private readonly OpenChargeMapRepository _repo = new(...); // konkret
}
```

---

## File Structure

```
src/
├── Domain/
│   ├── Exceptions/
│   │   └── StationNotFoundException.cs
│   ├── Models/
│   │   ├── ChargingStation.cs
│   │   └── Pin.cs                        (record — Value Object)
│   ├── Repositories/
│   │   └── IChargingStationRepository.cs (interface)
│   └── UseCases/
│       └── FindNearestStation.cs
├── Application/
│   ├── Interfaces/
│   │   └── IChargingService.cs
│   └── Services/
│       └── ChargingService.cs
├── Infrastructure/
│   ├── Dtos/
│   │   └── StationDto.cs                 (record + ToDomain())
│   └── Repositories/
│       └── OpenChargeMapRepository.cs
└── Presentation/
    ├── Controllers/
    │   └── ChargingController.cs
    └── Dtos/
        ├── FindStationRequest.cs         (record)
        └── StationResponse.cs           (record + From())
```

---

## Best Practices

### ✅ DO:
1. `interface IXyz` für jede externe Abhängigkeit (I-Präfix ist C#-Konvention)
2. Konstruktor-Injection überall — kein Property Injection
3. Use Cases: eine Klasse, eine `ExecuteAsync()`-Methode
4. `record` für Value Objects und DTOs (C# 9+)
5. `CancellationToken` in alle async-Methoden als letzten Parameter
6. Domain-Exceptions für Geschäftsfehler

### ❌ DON'T:
1. Kein `[ApiController]` oder EF-Core im Domain-Layer
2. Kein `new` für Dependencies — immer DI-Container
3. Keine Geschäftslogik in Controllern oder Repository-Implementierungen
4. Kein Domain-Objekt direkt als HTTP-Response zurückgeben
5. Kein `static` für Abhängigkeiten
6. Kein `async void` — immer `async Task`
