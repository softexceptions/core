---
tags: [skill, java, solid, clean-architecture, spring]
status: aktiv
date: 2026-05-25
---

# java-solid — Referenz

Skill für SOLID-Architektur in Java-Projekten mit Clean Architecture.

## Aufruf

```
/java-solid
```

## Speicherort (funktionale Skill-Datei)

```
core/.claude/skills/java-solid/skill.md
```

**Symlink für ein neues Projekt:**

```
ln -s /home/norbert/Dokumente/core/.claude/skills/java-solid \
      /pfad/zum/projekt/.claude/skills/java-solid
```

## Vier Schichten

| Schicht | Location | Inhalt |
|---|---|---|
| Domain | `src/main/java/.../domain/` | Entities, Records (Value Objects), Interfaces, Use Cases — kein Framework |
| Application | `src/main/java/.../application/` | Services, Orchestrierung (`@Service`) |
| Infrastructure | `src/main/java/.../infrastructure/` | Repository-Impl., JPA, HTTP-Clients, DTOs |
| Presentation | `src/main/java/.../presentation/` | REST-Controller, Request/Response-DTOs |

## Java-Besonderheit: Interface ist nativ

Java hat `interface` als echtes Keyword — SOLID ist hier besonders natürlich:

```
interface IChargingStationRepository → Vertrag (Domain)
class OpenChargeMapRepository implements IChargingStationRepository → Impl (Infrastructure)
```

Konstruktor-Injection (NICHT `@Autowired` auf Felder):
```
public ChargingService(IChargingStationRepository repo) { this.repo = repo; }
```

## Verwandte Ressourcen

- [[tdd]] — TDD (Java: JUnit 5 + Mockito)
- [[csharp-solid]] — C# SOLID (Vergleich: sehr ähnlich, andere Konventionen)
- [[python-solid]] — Python SOLID (Vergleich: ABC/Protocol vs. interface)
- [[../Skills]] — Übersicht aller Skills
