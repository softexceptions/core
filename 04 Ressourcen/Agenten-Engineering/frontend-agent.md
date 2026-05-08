---
tags: [agent, vue, frontend, solid]
status: aktiv
date: 2026-05-06
---

# Frontend Agent

Spezialisierter Claude-Code-Agent für Vue 3 + TypeScript Projekte nach SOLID-Architektur.

## Zweck

Spezialisierter Implementierungs-Agent für Vue 3 + TypeScript. Arbeitet unter der Koordination von Claude Code als Senior Developer — erhält klare Aufgaben, liefert Implementierung, die der Senior Dev reviewt.

## Arbeitsmodell

```
Claude Code (Senior Dev)
  └── briefed Frontend Agent
        └── implementiert Komponente / Composable / Service
              └── Senior Dev reviewt + nimmt ab
```

## Aufruf

```
/frontend-agent
```

Manueller Aufruf durch den Senior Dev — kein automatisches Triggern.

## Speicherort

Die Agent-Datei liegt im Vault und wird per Symlink in Projekte eingebunden:

```
core/.claude/agents/frontend-agent.md
```

**Symlink für ein neues Projekt anlegen:**

```bash
ln -s /home/norbert/Code/ipNINX/core/.claude/agents/frontend-agent.md \
      /pfad/zum/projekt/.claude/agents/frontend-agent.md
```

## Einsatzgebiete

- Neue Vue-Komponenten, Composables und Services erstellen
- Architekturentscheidungen im Frontend treffen
- Bestehenden Code auf SOLID-Verletzungen prüfen
- Refactoring nach dem Drei-Schichten-Modell

## Drei-Schichten-Modell (Kurzübersicht)

| Schicht | Ort | Regel |
|---|---|---|
| Business Logic | `services/`, `models/`, `validators/` | Reines TypeScript, kein Vue |
| Adapter | `composables/` | Vue-Reaktivität, delegiert an Services |
| Presentation | `components/` | Thin Components, kein Business-Code |

## Verwandte Ressourcen

- [[Skills/vue-solid/SKILL]] — Vollständige Architektur-Referenz
- [[Agent-Skills-Dev]] — Wie Agenten und Skills entwickelt werden
- [[Skills/Skills]] — Übersicht aller installierten Skills
