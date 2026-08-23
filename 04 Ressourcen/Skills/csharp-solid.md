---
tags: [skill, csharp, solid, clean-architecture, aspnet]
status: aktiv
date: 2026-05-25
---

# csharp-solid — Referenz

Skill für SOLID-Architektur in C#-Projekten mit Clean Architecture.

## Aufruf

```
/csharp-solid
```

## Speicherort (funktionale Skill-Datei)

```
core/.claude/skills/csharp-solid/skill.md
```

**Symlink für ein neues Projekt:**

```
ln -s /home/norbert/Dokumente/core/.claude/skills/csharp-solid \
      /pfad/zum/projekt/.claude/skills/csharp-solid
```

## Vier Schichten

| Schicht | Location | Inhalt |
|---|---|---|
| Domain | `src/Domain/` | Entities, Records (Value Objects), Interfaces, Use Cases — kein Framework |
| Application | `src/Application/` | Services, Orchestrierung |
| Infrastructure | `src/Infrastructure/` | Repository-Impl., EF Core, HTTP-Clients, DTOs |
| Presentation | `src/Presentation/` oder `src/Api/` | ASP.NET Core Controller, Minimal APIs, DTOs |

## C#-Besonderheit: I-Präfix Konvention

C# hat `interface` nativ — I-Präfix ist Namenskonvention (anders als Java, wo es optional ist):

```
interface IChargingStationRepository → Vertrag (Domain)
class OpenChargeMapRepository : IChargingStationRepository → Impl (Infrastructure)
```

DI-Registrierung in `Program.cs`:
```
builder.Services.AddScoped<IChargingStationRepository, OpenChargeMapRepository>();
```

`CancellationToken` immer als letzten Parameter in async-Methoden.

## Verwandte Ressourcen

- [[tdd]] — TDD (C#: xUnit + Moq)
- [[java-solid]] — Java SOLID (Vergleich: sehr ähnlich, andere Konventionen)
- [[python-solid]] — Python SOLID (Vergleich: ABC/Protocol vs. interface)
- [[../Skills]] — Übersicht aller Skills
