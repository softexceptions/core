---
tags: [ressource, skills, agents]
---

# Skills

Übersicht über Agent Skills und wie ich sie nutze für spezialisierte Aufgaben.

## Was sind Skills?

Skills sind spezialisierte, wiederverwendbare Funktionalitäten für Claude. Sie ermöglichen es, Claude neue Fähigkeiten beizubringen, ohne den Kern-Modell zu ändern.

## Installierte Skills

### Vault-Skills (Obsidian)

| Skill | Aufruf | Funktion |
|---|---|---|
| obsidian-markdown | automatisch | Obsidian-Flavored Markdown |
| obsidian-cli | automatisch | Vault-Operationen via CLI |
| obsidian-bases | automatisch | .base-Dateien, Views, Filter |
| json-canvas | automatisch | .canvas-Dateien, Mindmaps |
| defuddle | automatisch | Webseiten als Markdown |

### Entwicklungs-Skills

| Skill | Aufruf | Funktion |
|---|---|---|
| vue-solid | `/vue-solid` | Vue 3 + TypeScript, SOLID-Architektur |
| python-solid | `/python-solid` | Python + FastAPI, Clean Architecture |
| tdd | `/tdd` | Test-Driven Development |

Symlink-Befehle für neue Projekte: siehe jeweilige Referenz-Notiz im Skill-Ordner.

## Installierte Agenten

| Agent          | Aufruf            | Funktion                            |
| -------------- | ----------------- | ----------------------------------- |
| frontend-agent | `/frontend-agent` | Senior Vue 3 Entwickler:in          |
| backend-agent  | `/backend-agent`  | Senior Python/FastAPI Entwickler:in |

Referenz-Notizen: [[../../Agenten-Engineering/frontend-agent]] · [[../../Agenten-Engineering/backend-agent]]

## Ideen für weitere Skills

- Vue.js Component-Scaffold-Skill
- Code-Review-Skill
- Documentation-Generation-Skill

## Ressourcen

- [[Agent-Skills-Dev]] — Wie Skills entwickelt werden
