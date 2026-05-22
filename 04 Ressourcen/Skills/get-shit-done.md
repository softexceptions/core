---
tags:
  - skill
  - workflow
  - claude-code
status: aktiv
date: 2026-05-22
---

# get-shit-done — Referenz

Workflow-Skill für Claude Code. Strukturiert Arbeitsabläufe und verhindert, dass Claude in Endlosschleifen oder Kontextverlust gerät.

> [!info] Quelle
> Diagramm: `GitShitDone.excalidraw` unter `/home/norbert/Code/excalDraw/`

**GitHub:** [gsd-build/get-shit-done](https://github.com/gsd-build/get-shit-done)

## Installation

### Global (einmalig, gilt für alle Projekte)

```
npx skills add https://github.com/shoootyou/get-shit-done-multi --skill get-shit-done
```

### Lokal (nur für ein Projekt)

**Claude Code Variante:**

```
npx get-shit-done-cc@latest
```

**Multi-Variante:**

```
npx get-shit-done-multi
```

> [!tip] Wann welche Variante?
> **Global** = einmal einrichten, überall verfügbar via `~/.claude/`.
> **Lokal** = nur im aktuellen Projektordner unter `.claude/`.

## Deinstallation

### Global

```
rm -rf ~/.claude/skills/gsd-* ~/.claude/agents/gsd-* ~/.claude/get-shit-done/
```

### Lokal

```
rm -rf .claude/skills/gsd-* .claude/agents/gsd-* .claude/get-shit-done/
```

## Verwandte Ressourcen

- [[Skills]] — Übersicht aller Skills
- [[Agent-Skills-Dev]] — Skill-Entwicklung allgemein
