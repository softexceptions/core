---
tags: [skill, vue, solid, frontend]
status: aktiv
date: 2026-05-06
---

# vue-solid — Referenz

Spezialisierter Skill für Vue 3 + TypeScript nach SOLID-Architektur.

## Aufruf

```
/vue-solid
```

## Speicherort (funktionale Skill-Datei)

```
core/.claude/skills/vue-solid/skill.md
```

**Symlink für ein neues Projekt:**

```bash
ln -s /home/norbert/Code/ipNINX/core/.claude/skills/vue-solid \
      /pfad/zum/projekt/.claude/skills/vue-solid
```

## Einsatzgebiete

- Vue-Komponenten, Composables und Services erstellen
- Architekturentscheidungen im Frontend
- Refactoring nach Drei-Schichten-Modell
- Code-Reviews auf SOLID-Verletzungen

## Drei-Schichten-Modell

| Schicht | Ort | Regel |
|---|---|---|
| Business Logic | `services/`, `models/`, `validators/` | Reines TypeScript, kein Vue |
| Adapter | `composables/` | Vue-Reaktivität, delegiert an Services |
| Presentation | `components/` | Thin Components, kein Business-Code |

## Verwandte Ressourcen

- [[../Agent-Skills-Dev]] — Skill-Entwicklung allgemein
- [[../Skills]] — Übersicht aller Skills
- [[../../Agenten-Engineering/frontend-agent]] — Frontend-Agent (nutzt diese Architektur)
