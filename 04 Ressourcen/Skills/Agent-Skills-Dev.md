---
tags: [ressource, skills, entwicklung]
---

# Agent Skills Entwicklung

Wie ich spezialisierte Skills für meine Projekte entwickle.

## Skill vs. Agent

| | Skill | Agent |
|---|---|---|
| Was | Wiederverwendbarer Prompt | Eigenständiger Subprozess |
| Aufruf | `/skill-name` im Chat | Über `Agent`-Tool von Claude |
| Tools | Erbt die des Eltern-Chats | Eigene Tool-Auswahl |
| Zustand | Kein eigener Zustand | Eigener Kontext |
| Einsatz | Spezialisierte Aufgaben | Parallelarbeit, Delegation |

## Dateiformat

Ein Skill ist eine Markdown-Datei mit YAML-Frontmatter:

```markdown
---
name: mein-skill
description: |
  Kurze Beschreibung was der Skill macht und wann Claude ihn nutzen soll.
  Diese Zeilen entscheiden, ob Claude den Skill automatisch triggert.
---

# Skill-Titel

Du agierst jetzt als [Rolle]. Deine Aufgabe ist [Ziel].

## Regeln

- Regel 1
- Regel 2

## Beispiel

[Konkretes Beispiel für Input → Output]
```

**Wichtig:** Die `description` im Frontmatter bestimmt, wann Claude den Skill automatisch aktiviert. Sie sollte präzise Trigger-Bedingungen enthalten.

## Installation

Skills liegen in `.claude/skills/<name>/skill.md` relativ zum Vault-Ordner:

```
core/
└── .claude/
    └── skills/
        ├── vue-solid/
        │   └── skill.md    ← Skill-Datei
        └── python-solid/
            └── skill.md
```

Aufruf im Chat: `/vue-solid` oder `/python-solid`

## Entwicklungs-Prozess

1. **Identifizieren** — Welche Aufgaben wiederhole ich häufig?
2. **Spezifizieren** — Rolle, Trigger-Bedingung, Regeln und Beispiel definieren
3. **Datei anlegen** — `.claude/skills/<name>/skill.md` erstellen
4. **Testen** — Im Chat mit `/skill-name` aufrufen und Verhalten prüfen
5. **Iterieren** — Description und Regeln verfeinern bis das Verhalten stimmt
6. **Dokumentieren** — Use Cases und Einschränkungen in `Skills.md` eintragen

## Konkretes Minimalbeispiel

Basierend auf dem vorhandenen `vue-solid`-Skill:

```markdown
---
name: vue-solid
description: |
  SOLID principles for Vue 3 + TypeScript projects.
  USE when: writing Vue services, composables, or components.
  Enforces: Business Logic (classes) → Adapter (composables) → Presentation (components).
---

# Vue 3 SOLID Architecture

Du agierst als Senior Vue 3 + TypeScript Engineer.

## Schichtenmodell

1. Business Logic (`src/services/`, `src/models/`) — SOLID zu 100%
2. Adapter (`src/composables/`) — Brücke zwischen Reaktivität und Logik
3. Presentation (`src/components/`) — Dünne UI-Schicht

## Regeln

- Services sind TypeScript-Klassen mit Interface
- Composables importieren niemals direkt aus Services-Implementierungen
- Components delegieren Logik an Composables
```

## Ideen für neue Skills

- **Python-Formatter** — Format-Checks und PEP8-Konformität
- **Vue-Component-Generator** — Neue Components nach SOLID-Schema
- **Test-Generator** — Tests aus bestehendem Code generieren
- **Architecture-Validator** — SOLID-Verletzungen im Code aufspüren

## Ressourcen

- [[Skills]] — Übersicht aller installierten Skills
- Claude Code Dokumentation: Skills-Abschnitt
