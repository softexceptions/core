---
tags:
  - plan
  - workflow
  - claude-code
  - vault
status: abgeschlossen
date: 2026-05-07
---

# Plan: „Merke dir den Plan" — Vault-Session-Routine

## Kontext

Claude Code-Pläne sollen direkt im Vault gespeichert werden können. Der Befehl „Merke dir den Plan in meinem core" löst eine definierte Session-Routine aus, die den aktuellen Plan als strukturierte Notiz ablegt.

## Änderung

**Datei:** [[CLAUDE]] (Vault-Root)  
**Abschnitt:** `Session-Routinen` → neuer Abschnitt `### Plan merken`

## Schritte

- [x] Neue Routine `### Plan merken` in `core/CLAUDE.md` einfügen
- [x] Trigger-Phrasen definieren
- [x] Ablage-Logik definieren (Projekt → Bereich → Inbox)
- [x] Notiz-Format mit YAML-Frontmatter, Checkboxen, Verifikation festlegen
- [x] Erste Plan-Notiz nach neuer Routine erstellen (diese Datei)

## Ablage-Logik

| Kontext | Ablageort |
|---|---|
| Projekt-Plan | `02 Projekte/[Projektname]/[Titel].md` |
| Bereichs-Plan | `03 Bereiche/[Bereich]/[Titel].md` |
| Unklar | `01 Inbox/[Titel].md` |

## Verifikation

1. „Merke dir den Plan" sagen, wenn ein Plan aktiv ist
2. Notiz erscheint am richtigen Ort im Vault
3. YAML-Frontmatter korrekt: `tags`, `status`, `date`
4. Wikilink im zugehörigen Projekt oder Bereich gesetzt

## Quelle

Erstellt von [[Claude Code]] am 2026-05-07
