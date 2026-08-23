---
tags: [ressource, skills, agents]
date: 2026-05-08
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
| [[vue-solid]] | `/vue-solid` | Vue 3 + TypeScript, SOLID-Architektur |
| [[python-solid]] | `/python-solid` | Python + FastAPI, Clean Architecture |
| [[flutter-solid]] | `/flutter-solid` | Flutter + Dart, Clean Architecture |
| [[rust-solid]] | `/rust-solid` | Rust, SOLID + Clean Architecture |
| [[java-solid]] | `/java-solid` | Java + Spring Boot, Clean Architecture |
| [[csharp-solid]] | `/csharp-solid` | C# + ASP.NET Core, Clean Architecture |
| [[tdd]] | `/tdd` | Test-Driven Development (Python, Vue, Flutter, Rust, Java, C#) |
| [[get-shit-done]] | `npx get-shit-done-cc@latest` | Workflow-Struktur, verhindert Kontextverlust |
| [[paul]] | `npx paul-framework` | Planungs-Framework für komplexe Projekte |

### Design-Skills (System-Skills, kein Symlink nötig)

| Skill | Aufruf | Funktion |
|---|---|---|
| ui-ux-pro-max | `/ui-ux-pro-max` | UI/UX für Web + Mobile inkl. Flutter — 50+ Stile, 161 Paletten, 99 UX-Richtlinien |
| frontend-design | `/frontend-design` | Hochwertige Web-Interfaces — Komponenten, Layouts, Poster |

Symlink-Befehle für neue Projekte: siehe die jeweils verlinkte Referenz-Notiz in der Tabelle oben.

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
