---
tags:
  - skill
  - workflow
  - claude-code
status: aktiv
date: 2026-05-22
---

# PAUL — Referenz

Strukturiertes Planungs-Framework für Claude Code. Geeignet für komplexe Architekturen und langfristige Projekte.

> [!info] Workflow
> **Discuss → Research → Plan → Execute → Verify**
> Mehr Gründlichkeit als GSD — für Aufgaben, bei denen Planung wichtiger ist als Geschwindigkeit.

> [!warning] Installationspfad unterscheidet sich von GSD
> PAUL installiert nach `~/.claude/commands/paul/` (global) bzw. `.claude/commands/paul/` (lokal) —
> **nicht** nach `~/.claude/skills/` wie GSD und andere Skills.

**GitHub:** [ChristopherKahler/paul](https://github.com/ChristopherKahler/paul)

## Installation

### Interaktiv (empfohlen — fragt nach Global oder Lokal)

```
npx paul-framework
```

### Global (alle Projekte)

```
npx paul-framework --global
```

### Lokal (nur aktuelles Projekt)

```
npx paul-framework --local
```

### Update

```
npx paul-framework@latest
```

## Verifikation

Nach der Installation in Claude Code prüfen:

```
/paul:help
```

## Wichtige Slash-Commands

| Befehl | Funktion |
|---|---|
| `/paul:init` | Projekt initialisieren + Anforderungen erfassen |
| `/paul:plan [phase]` | Ausführbaren Plan erstellen |
| `/paul:apply [path]` | Genehmigten Plan ausführen |
| `/paul:unify [path]` | Schleife schließen + abgleichen |
| `/paul:progress [context]` | Status und nächsten Schritt prüfen |

## Deinstallation

### Global

```
rm -rf ~/.claude/commands/paul/
```

### Lokal

```
rm -rf .claude/commands/paul/
```

> [!tip] Problem: Commands nicht gefunden?
> Claude Code neu starten — Slash-Commands werden beim Start geladen.

## Verwandte Ressourcen

- [[../../03 Bereiche/KI-Softwareentwicklung/GSD-vs-PAUL]] — Wann GSD, wann PAUL?
- [[get-shit-done]] — Alternative (schneller, iterativer)
- [[Skills]] — Übersicht aller Skills
