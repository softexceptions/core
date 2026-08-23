---
tags: [skill, flutter, dart, solid, mobile]
status: aktiv
date: 2026-05-24
---

# flutter-solid — Referenz

Spezialisierter Skill für Flutter + Dart nach Clean Architecture und SOLID.

## Aufruf

```
/flutter-solid
```

## Speicherort (funktionale Skill-Datei)

```
core/.claude/skills/flutter-solid/skill.md
```

**Symlink für ein neues Projekt:**

```
ln -s /home/norbert/Dokumente/core/.claude/skills/flutter-solid \
      /pfad/zum/projekt/.claude/skills/flutter-solid
```

## Einsatzgebiete

- Flutter-Widgets, Riverpod-Controller und Use Cases erstellen
- Abstract Repositories und Implementierungen definieren
- Dependency Injection mit `get_it` einrichten
- Clean-Architecture-Schichtung in Dart durchsetzen
- Code-Reviews auf SOLID-Verletzungen

## Drei-Schichten-Modell

| Schicht | Ort | Inhalt |
|---|---|---|
| Domain | `lib/domain/` | Models, `abstract class IRepository`, Use Cases |
| Data | `lib/data/` | Repository-Implementierungen, DataSources, DTOs |
| Presentation | `lib/presentation/` | Thin Widgets, Riverpod Controller (nur State) |

## Dart-Besonderheit: kein `interface`

Dart kennt kein `interface`-Schlüsselwort. Stattdessen:

```
abstract class IChargingRepository {
  Future<List<ChargingStation>> findNearby(...);
}
```

Klassen implementieren mit `implements IChargingRepository`.

## Verwandte Ressourcen

- [[../Agent-Skills-Dev]] — Skill-Entwicklung allgemein
- [[../Skills]] — Übersicht aller Skills
- [[../../../02 Projekte/KiaChargeNav]] — Erstes Projekt mit diesem Skill
