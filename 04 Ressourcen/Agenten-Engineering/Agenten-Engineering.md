---
tags: [ressource, agenten, engineering]
---

# Agenten Engineering

Comprehensive Guide zu Aufbau, Design und Orchestrierung von AI Agents für Softwareentwicklung.

## Definition

Ein Agent ist ein autonomes System, das:
- Anweisungen entgegennimmt
- Mit Tools/Skills arbeitet
- Kontext berücksichtigt
- Selbstständig Entscheidungen trifft
- Feedback-Schleifen hat

## Agent-Architektur

```
Benutzer-Anfrage
    ↓
[Agent-Loop]
├─ Verstehen (Prompt Understanding)
├─ Planen (What steps needed?)
├─ Ausführen (Execute tools/skills)
├─ Evaluieren (Is result good?)
└─ Iterieren (Feedback und Verbesserung)
    ↓
Ergebnis
```

## Für Softwareentwicklung

### Einsatzbereiche
- Code-Generierung
- Refactoring
- Testing und Debugging
- Architektur-Design
- Documentation-Erstellung

### Best Practices
- Klare Zieldefinition
- Robustes Error-Handling
- Validierung von Outputs
- Monitoring und Logging
- Feedback-Mechanismen

## Orchestrierungs-Muster

Claude Code agiert immer als **Senior Developer** — nie als Spezialist. Spezialisten werden gebrieft, liefern Implementierung, der Senior Dev reviewt.

```
Claude Code (Senior Dev)
  ├── /backend-agent   → implementiert + schreibt Tests (TDD)
  ├── /frontend-agent  → implementiert + schreibt Tests (TDD)
  ├── /python-solid    → Architektur-Wächter
  └── /tdd             → Skill, aktiviert TDD-Disziplin bei beiden Agenten
```

Dieses Muster ist projektübergreifend. Die konkrete Agenten-Auswahl variiert je Projekt — das Prinzip (Orchestrator + Spezialisten) bleibt immer gleich.

## Installierte Agenten

| Agent | Aufruf | Beschreibung |
|---|---|---|
| frontend-agent | `/frontend-agent` | Senior Vue 3 + TypeScript, SOLID Drei-Schichten-Modell |
| backend-agent | `/backend-agent` | Senior Python + FastAPI, Clean Architecture, Repository Pattern |

Alle Agent-Dateien liegen in `.claude/agents/` und werden per Symlink in Projekte eingebunden.
Referenz-Notizen: [[frontend-agent]] · [[backend-agent]]

## Agenten-Status prüfen

Vor dem Start einer Session prüfen, ob das Agent-Team aktiv ist:
```
"Ist das Agent-Team aktiv?"
```
Claude gibt Auskunft welche Agenten verfügbar und geladen sind.

## Agenten-Ideen (offen)

- **Code-Review-Agent** — Automatische Code-Reviews
- **Testing-Agent** — Test-Generierung und Execution
- **Documentation-Agent** — API-Dokumentation generieren
- **Architecture-Agent** — Design-Reviews und Suggestions

## Ressourcen

- [[../Skills/Agent-Skills-Dev]] — Wie Agenten entwickelt werden
- [[../Skills/Skills]] — Übersicht aller Skills
