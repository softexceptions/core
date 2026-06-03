---
tags: [kontext, coding, regeln, solid, tdd]
status: aktiv
date: 2026-05-25
---

# Coding-Regeln

Verbindliche Regeln für alle Projekte — unabhängig von Sprache oder Framework.

## Allgemein

1. **Objektorientiert wo möglich** — Verhalten gehört zur Datenstruktur, nicht als Freihand-Funktionen
2. **SOLID-Prinzipien** — Single Responsibility, Open/Closed, Liskov, Interface Segregation, Dependency Inversion
3. **TDD** — Kein Produktionscode ohne vorherigen fehlschlagenden Test. Reihenfolge: `/tdd` → Implementierung → SOLID-Review

## Reihenfolge bei jeder Implementierung

```
1. /tdd aktivieren → fehlschlagenden Test schreiben
2. Minimale Implementierung → Test grün
3. Refactoring → SOLID-Skill-Review
```

## Sprach-spezifische SOLID-Skills

| Sprache | Skill | Interface-Konzept |
|---|---|---|
| Python | `/python-solid` | `ABC` / `Protocol` |
| Vue 3 | `/vue-solid` | TypeScript `interface` |
| Flutter/Dart | `/flutter-solid` | `abstract class` |
| Rust | `/rust-solid` | `trait` |
| Java | `/java-solid` | `interface` (nativ) |
| C# | `/csharp-solid` | `interface` (nativ, I-Präfix) |

## TDD-Stack je Sprache

| Sprache | Framework | Test-Befehl |
|---|---|---|
| Python | pytest + pytest-asyncio | `pytest` |
| Vue 3 | Vitest + Vue Test Utils | `npm run test` |
| Flutter | flutter_test + mocktail | `flutter test` |
| Rust | cargo test + mockall | `cargo test` |
| Java | JUnit 5 + Mockito | `mvn test` / `./gradlew test` |
| C# | xUnit + Moq | `dotnet test` |

## Herkunft

Diese Regeln sind durch Feedback von Norbert entstanden — er hat mehrfach darauf hingewiesen, dass SOLID und TDD in allen Projekten gelten, auch wenn kein Skill explizit aufgerufen wurde.
