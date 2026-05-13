---
title: [Projektname]
tags:
  - projekt
  - aktiv
status: in-progress
stand: [YYYY-MM-DD]
---

# [Projektname]

> [!important] Nach `/init` in die CLAUDE.md des Projekts eintragen:
> `Projektbeschreibung: core/02 Projekte/[Projektname].md`

[Kurzbeschreibung: Was ist das Projekt? Für wen? Warum?]

> [!info] Ziel
> [Hauptziel in 1–2 Sätzen — was soll am Ende anders sein?]

## Stack

| Schicht | Technologie |
|---|---|
| Frontend | [z.B. Vue 3 + TypeScript + Vite + Tailwind CSS] |
| Backend | [z.B. FastAPI + Python] |
| Datenbank | [z.B. SQLite / PostgreSQL / keins] |
| Deployment | [z.B. Proxmox LXC / Docker / Vercel] |

## Architektur

[Wichtige Muster und SOLID-Schichten — kurze Verzeichnisstruktur]

```
src/services/interfaces/       → Interfaces (DIP)
src/services/implementations/  → Implementierungen
src/composables/               → Adapter-Schicht (Vue-Reaktivität)
src/components/                → Thin Components
```

## Design

| Token | Wert | Bedeutung |
|---|---|---|
| Primär | `#000000` | ... |
| Hintergrund | `#000000` | ... |

- **Stil:** [z.B. Claymorphism / Glassmorphism / Minimalism]
- **Schriften:** [z.B. Nunito (Headings) + DM Sans (Body)]

## Agent-Team

Welche Agenten und Skills für dieses Projekt aktiv sind.

| Agent / Skill | Aufruf | Wann einsetzen |
|---|---|---|
| Frontend Agent | `/frontend-agent` | Vue-Komponenten, Composables, Services |
| Backend Agent | `/backend-agent` | FastAPI-Routen, Services, Tests |
| vue-solid | `/vue-solid` | SOLID-Review bei Vue-Dateien |
| python-solid | `/python-solid` | SOLID-Review bei Python-Dateien |

> [!tip] Nicht benötigte Zeilen einfach löschen oder ergänzen.

## Skills (automatisch anwenden)

- `.vue` Dateien → `/vue-solid` automatisch nach Implementierung
- `.py` Dateien → `/python-solid` automatisch nach Implementierung
- [weitere projektspezifische Regeln]

## Claude-Verhaltensregeln

- **Vault first:** Bei jeder Frage zuerst `core/` lesen
- **Modell:** Sonnet für Standard-Tasks, Opus für Architektur-Entscheidungen
- **Merke dir:** Löst immer internes Claude-Memory + Vault-Notiz aus
- [projektspezifische Abweichungen hier ergänzen]

## Nächste Aufgaben

Format: Aufgabe + Kontext + erwartetes Ergebnis — damit ich autonom arbeiten kann.

- [ ] **[Aufgabe]:** Was soll ich tun?
  **Kontext:** Warum / welche Einschränkung?
  **Ergebnis:** Was soll am Ende anders sein?

## Start-Befehle

**Frontend** (Port [XXXX]):
```bash
cd frontend && npm run dev
```

**Backend** (Port [XXXX]):
```bash
cd backend && uvicorn main:app --reload
```

> [!tip] Proxy
> [z.B. Vite leitet `/api` automatisch an `http://localhost:8000` weiter]

## Deployment

- [ ] [Schritt 1]
- [ ] [Schritt 2]

**Deploy-Befehl:**
```bash
# Beispiel
cd frontend && npm run build
scp -r dist/* user@server:/var/www/projekt/
```

**Erreichbar unter:** `http://[IP]:[Port]`

## Bekannte Stolperfallen

> [!warning] [Titel]
> [Beschreibung der Falle + wie man sie vermeidet]

> [!tip] [Titel]
> [Hilfreicher Hinweis]

## Changelog

### [YYYY-MM-DD]

- **[Feature/Fix]:** Was wurde geändert und warum?
