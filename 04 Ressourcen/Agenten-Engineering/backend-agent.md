---
tags: [agent, python, fastapi, solid, backend]
status: aktiv
date: 2026-05-06
---

# Backend Agent

Spezialisierter Claude-Code-Agent für Python + FastAPI Projekte nach Clean Architecture und SOLID.

## Zweck

Spezialisierter Implementierungs-Agent für Python + FastAPI. Arbeitet unter der Koordination von Claude Code als Senior Developer — erhält klare Aufgaben, liefert Implementierung, die der Senior Dev reviewt.

## Arbeitsmodell

```
Claude Code (Senior Dev)
  └── briefed Backend Agent
        └── implementiert Schicht / Feature
              └── Senior Dev reviewt + nimmt ab
```

## Aufruf

```
/backend-agent
```

Manueller Aufruf durch den Senior Dev — kein automatisches Triggern. Für TDD zusätzlich `/tdd` aufrufen.

## Speicherort (funktionale Agent-Datei)

```
core/.claude/agents/backend-agent.md
```

**Symlink für ein neues Projekt:**

```bash
ln -s /home/norbert/Code/ipNINX/core/.claude/agents/backend-agent.md \
      /pfad/zum/projekt/.claude/agents/backend-agent.md
```

## Einsatzgebiete

- Neue FastAPI-Routen, Services und Repositories erstellen
- Domain-Modelle und Value Objects definieren
- Repository Pattern mit SQLAlchemy async einrichten
- Architekturentscheidungen im Backend treffen
- Bestehenden Code auf Clean-Architecture-Verletzungen prüfen

## Vier-Schichten-Modell

| Schicht | Ort | Regel |
|---|---|---|
| Domain | `domain/` | Kein FastAPI, kein SQLAlchemy |
| Application | `application/` | Nur Domain-Abhängigkeiten |
| Infrastructure | `infrastructure/` | SQLAlchemy, externe APIs |
| Presentation | `presentation/` | Thin Layer, nur HTTP-Concerns |

## Datenbank-Strategie

SQLAlchemy async + Repository Pattern (Option A):
- `IUserRepository` als `Protocol` in `domain/`
- `UserRepository` (SQLAlchemy) in `infrastructure/`
- `InMemoryUserRepository` für Tests — kein Mocking nötig

## Verwandte Ressourcen

- [[../Skills/python-solid/SKILL]] — Vollständige Architektur-Referenz
- [[../Skills/tdd/SKILL]] — TDD (separat aufrufen mit /tdd)
- [[frontend-agent]] — Frontend-Pendant (Vue 3 + TypeScript)
- [[../Agent-Skills-Dev]] — Agenten-Entwicklung allgemein
