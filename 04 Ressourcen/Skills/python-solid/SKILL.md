---
tags: [skill, python, solid, backend]
status: aktiv
date: 2026-05-06
---

# python-solid — Referenz

Spezialisierter Skill für Python + FastAPI nach Clean Architecture und SOLID.

## Aufruf

```
/python-solid
```

## Speicherort (funktionale Skill-Datei)

```
core/.claude/skills/python-solid/skill.md
```

**Symlink für ein neues Projekt:**

```bash
ln -s /home/norbert/Code/ipNINX/core/.claude/skills/python-solid \
      /pfad/zum/projekt/.claude/skills/python-solid
```

## Einsatzgebiete

- FastAPI-Routen, Services und Repositories erstellen
- Domain-Modelle und Value Objects definieren
- Dependency Injection mit FastAPI einrichten
- Clean-Architecture-Schichtung durchsetzen

## Vier-Schichten-Modell

| Schicht | Ort | Inhalt |
|---|---|---|
| Domain | `domain/` | Models, Value Objects, Interfaces |
| Application | `application/` | Services, Use Cases |
| Infrastructure | `infrastructure/` | DB, externe APIs |
| Presentation | `presentation/` | FastAPI-Routen, Pydantic-Schemas |

## Verwandte Ressourcen

- [[../Agent-Skills-Dev]] — Skill-Entwicklung allgemein
- [[../Skills]] — Übersicht aller Skills
- [[../tdd/SKILL]] — TDD-Skill (integriert mit python-solid)
