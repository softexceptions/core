---
tags: [ressource, mcp, context7, python, dokumentation]
status: aktiv
date: 2026-05-07
---

# context7 — MCP-Dokumentationsserver

MCP-Server, der aktuelle Library-Dokumentation direkt in den Claude-Kontext lädt. Kein WebFetch, kein Trainingswissen — immer die echte, aktuelle Doku.

## Tools

| Tool | Funktion |
|---|---|
| `mcp__context7__resolve-library-id` | Library-ID ermitteln (z.B. `fastapi`, `sqlalchemy`) |
| `mcp__context7__query-docs` | Spezifische Doku-Abschnitte abrufen |

## Wann verwenden

Immer wenn ich Dokumentation zu Libraries oder Frameworks abrufe — auch für bekannte Libraries. Besonders kritisch bei Python-Projekten.

## Warum bei Python besonders wichtig

Das Python-Ökosystem hat häufige Breaking Changes zwischen Versionen:

| Library | Kritische Änderung |
|---|---|
| SQLAlchemy 2.x | Komplett neue async-API gegenüber 1.4 |
| Pydantic v2 | Neues Validierungsmodell, andere API |
| pydantic-settings | Seit v2 separates Paket, andere Imports |
| pytest-asyncio | `asyncio_mode` Konfiguration geändert |
| FastAPI | Lifespan-Events ersetzen `on_startup`/`on_shutdown` |

Ohne context7 → veralteter Code der nicht mehr läuft.

## Regel

> Trainingswissen für Library-APIs nie vertrauen — immer context7 zuerst.

## Verwandte Ressourcen

- [[MCP]] — MCP allgemein
- [[../Skills/python-solid/SKILL]] — Python Clean Architecture
- [[../../02 Projekte/ipNginx]] — Projekt das context7 aktiv nutzt
