---
tags: [ressource, claude, ai]
---

# Claude

Übersicht und Ressourcen zu Claude, dem KI-Modell von Anthropic, das ich täglich zur Entwicklung nutze.

## Überblick

Claude ist ein großes Sprachmodell (LLM), das ich für verschiedenste Entwicklungsaufgaben einsetze. Es kann Code generieren, debuggen, erklären und komplexe Probleme lösen helfen.

## Claude Versionen

- **Claude Haiku 4.5** — Schnell, für einfache Aufgaben
- **Claude Sonnet 4.6** — Balanced, Standard für meiste Aufgaben
- **Claude Opus 4.7** — Powerful, für komplexe Architektur-Entscheidungen

## Kernfähigkeiten

- Code-Generierung und Refactoring
- Problem-Analyse und Debugging
- Dokumentation und Erklärungen
- Architektur-Design
- Test-Schreiben
- Prompt Engineering Feedback

## Integration

- [[Claude Code]] — IDE-Integration
- [[MCP|Model Context Protocol]] — Externe Tools und Datenquellen
- [[Skills]] — Benutzerdefinierte Fähigkeiten
- [[Agenten|Agent Engineering]] — Autonome Systeme

## Ressourcen & Links

- [Claude Dokumentation](https://docs.anthropic.com)
- [[Prompt-Engineering]] — Best Practices
- [[03 Bereiche/Unterricht/Schüler-Onboarding|Schüler-Onboarding]] — Wie man anfängt

## Neues Projekt starten

1. Projektordner anlegen: `/home/norbert/Code/[Projektname]`
2. Symlink anlegen: `ln -s /home/norbert/Dokumente/core core`
3. `04 Ressourcen/Claude/Projektvorlage.md` kopieren → `02 Projekte/[Projektname].md`
4. Vorlage ausfüllen: Ziel, Stack, Architektur, Agent-Team, Skills, Start-Befehle
5. Erste Aufgaben unter `## Nächste Aufgaben` eintragen
6. Claude Code starten + `/init` ausführen → `CLAUDE.md` wird generiert
7. In die generierte `CLAUDE.md` eintragen:
   ```
   Projektbeschreibung: core/02 Projekte/[Projektname].md
   ```

> [!important] Reihenfolge beachten
> Vorlage **vor** dem ersten Claude-Start ausfüllen — sonst fehlt mir der Projektkontext beim Start.

> [!tip] Aufgaben-Format
> Jede Aufgabe braucht drei Teile damit ich autonom arbeiten kann:
> `**Aufgabe:** Was — **Kontext:** Warum — **Ergebnis:** Was soll anders sein`

## Meine Erkenntnisse

-
