---
tags: [skill, tdd, testing, python, vue]
status: aktiv
date: 2026-05-06
---

# tdd — Referenz

Skill für striktes Test-Driven Development nach Red → Green → Refactor.

## Aufruf

```
/tdd
```

## Speicherort (funktionale Skill-Datei)

```
core/.claude/skills/tdd/skill.md
```

**Symlink für ein neues Projekt:**

```bash
ln -s /home/norbert/Code/ipNINX/core/.claude/skills/tdd \
      /pfad/zum/projekt/.claude/skills/tdd
```

## Einsatzgebiete

- Neue Features testgetrieben entwickeln (kein Produktionscode ohne fehlschlagenden Test)
- TDD-Zyklus einhalten: Red → Green → Refactor
- Tests für Services, Repositories und Routen schreiben

## Test-Stack

| Kontext | Framework |
|---|---|
| Python / FastAPI | pytest + pytest-asyncio |
| Vue 3 / TypeScript | Vitest + Vue Test Utils |

## Verwandte Ressourcen

- [[../python-solid/SKILL]] — Clean Architecture (TDD-Tests spiegeln die Schichten wider)
- [[../vue-solid/SKILL]] — Vue SOLID (Frontend-Tests)
- [[../Agent-Skills-Dev]] — Skill-Entwicklung allgemein
- [[../Skills]] — Übersicht aller Skills
