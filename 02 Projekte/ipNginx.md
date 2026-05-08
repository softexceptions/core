---
tags: [projekt, python, fastapi, nginx, vue, ki-entwicklung]
status: aktiv
date: 2026-05-07
---

# ipNINX

GoAccess-Ersatz: Echtzeit-NGINX-Log-Analyzer mit Geolocation, Bedrohungserkennung, Session-Analyse und Vue 3-Dashboard. Deployed als systemd-Service im LXC-Container — keine Cloud-Abhängigkeiten.

## Ziel

NGINX wird so konfiguriert, dass es Logs direkt im **JSON-Format** ausgibt — kein Regex-Parser nötig, nur `json.loads()`. Das Dashboard zeigt Traffic, Bedrohungen, Geo-Verteilung und Performance in Echtzeit via SSE.

## Projektpfad

```
/home/norbert/Code/ipNINX/
```

## Arbeitsmodell

Orchestrierungs-Muster: [[../../04 Ressourcen/Agenten-Engineering/Agenten-Engineering|Agenten-Engineering]]

Aktive Agenten in diesem Projekt:
- `/backend-agent` + `/tdd` — Implementierung und Tests
- `/python-solid` — Architektur-Wächter
- `/vue-solid` — Frontend-Implementierung

## Architektur (4 Schichten)

```
domain/          ← LogEntry, GeoInfo, ThreatInfo, SessionInfo (@dataclass frozen)
                   ILogRepository, IGeoEnricher, IThreatDetector (Protocol)
application/     ← LogService, StatsService, GeoService, ThreatService,
                   SessionService, AnomalyService
infrastructure/  ← SQLAlchemy async, log_watcher, log_parser (json.loads),
                   geo_enricher (GeoLite2), ua_parser, owasp_detector
presentation/    ← FastAPI-Routen, SSE /events, dependencies.py
frontend/        ← Vue 3 + TypeScript + Tailwind + ECharts
```

## Feature-Scope

| Feature | Priorität |
|---|---|
| Log-Parsing (NGINX JSON) | Kern |
| Basis-Statistiken (Top-IPs, Status, Traffic) | Kern |
| Performance-Analyse (p50/p95/p99) | Hoch |
| Geolocation (MaxMind GeoLite2, offline) | Hoch |
| User-Agent-Analyse (Browser, OS, Bot) | Hoch |
| Bedrohungserkennung (SQL/XSS/Scanner/Brute-Force) | Hoch |
| Web-Dashboard (Vue 3 + SSE, Echtzeit) | Hoch |
| Session-Rekonstruktion (Verweildauer, Bounce) | Mittel |
| Anomalie-Erkennung (avg + 3σ) | Mittel |
| IP-Reputation (AbuseIPDB) | Optional |

## API-Endpunkte

| Endpunkt | Beschreibung |
|---|---|
| `GET /logs` | Alle Log-Einträge, paginiert |
| `GET /stats` | Top-IPs, Status-Verteilung, Requests/Stunde |
| `GET /geo/summary` | Traffic nach Land, Stadt, ASN |
| `GET /threats` | Erkannte Bedrohungen |
| `GET /threats/summary` | Anzahl pro Bedrohungstyp |
| `GET /sessions` | Rekonstruierte Sessions |
| `GET /anomalies` | Traffic-Anomalien |
| `GET /visitors/summary` | Browser, OS, Gerät, Bounce-Rate |
| `GET /performance` | Antwortzeiten p50/p95/p99 |
| `GET /events` | SSE-Stream (Echtzeit) |

## Dashboard-Layout

```
┌─────────────────────────────────────────────────────────┐
│  ipNINX                                    🔴 Live       │
├──────────┬───────────┬────────────┬─────────────────────┤
│ Requests │ Bedrohungen│ Besucher  │ Ø Antwortzeit        │
├──────────┴───────────┴────────────┴─────────────────────┤
│ Traffic/Stunde (ECharts)                                 │
├─────────────────────────┬───────────────────────────────┤
│ Weltkarte Heatmap       │ Top-Länder / Browser / Gerät  │
│ (ECharts Choropleth)    │                               │
├─────────────────────────┴───────────────────────────────┤
│ Bedrohungs-Feed (SSE)   │ Log-Stream (SSE)               │
└─────────────────────────┴───────────────────────────────┘
```

## Implementierung starten

| Phrase | Was passiert |
|---|---|
| **„Phase 1 starten"** | Backend-Agent + TDD: Domain-Modelle + JSON-Parser |
| **„Phase 2 starten"** | Geo-Enricher + UA-Parser |
| **„Phase 3 starten"** | Threat-, Session-, Anomalie-, Performance-Service |
| **„Phase 4 starten"** | FastAPI-Routen + SSE Event-Bus |
| **„Phase 5 starten"** | Vue-Solid-Agent: Frontend + Dashboard |
| **„Phase 6 starten"** | Deployment: systemd, NGINX, setup.sh |

## TDD-Reihenfolge (6 Phasen)

### Phase 1 — Domain & Parser
- [ ] `tests/domain/test_models.py`
- [ ] `src/domain/models.py`
- [ ] `tests/infrastructure/test_log_parser.py`
- [ ] `src/infrastructure/watcher/log_parser.py`

### Phase 2 — Anreicherung
- [ ] `tests/infrastructure/test_geo_enricher.py`
- [ ] `src/infrastructure/enricher/geo_enricher.py`
- [ ] `tests/infrastructure/test_ua_parser.py`
- [ ] `src/infrastructure/enricher/ua_parser.py`

### Phase 3 — Analyse
- [ ] Threat-Service (OWASP-Regex, Brute-Force)
- [ ] Session-Service
- [ ] Anomalie-Service (avg + 3σ)
- [ ] Performance-Service (Perzentile)

### Phase 4 — Backend API
- [ ] Alle FastAPI-Routen inkl. SSE `/events`
- [ ] SSE Event-Bus

### Phase 5 — Frontend
- [ ] Vite + Vue 3 + Tailwind + ECharts aufsetzen
- [ ] Services/Interfaces → EventStreamService zuerst
- [ ] Widgets: StatCard → TrafficChart → LogStream → ThreatsFeed → GeoMap → VisitorStats → PerformanceWidget
- [ ] AnomalyAlert-Overlay
- [ ] Vitest-Tests

### Phase 6 — Deployment
- [ ] systemd-Service
- [ ] NGINX-Konfiguration (SSE-Buffering deaktivieren für `/api/events`)
- [ ] `setup.sh` (venv, GeoLite2, NGINX-Config)
- [ ] Deployment auf SchulOrdnungHems LXC (Port 8001)

## Konfiguration (.env)

```env
NGINX_LOG_PATH=/var/log/nginx/access.log
DATABASE_URL=sqlite+aiosqlite:///./ipnginx.db
API_PORT=8000
GEOIP_CITY_DB=/opt/geoip/GeoLite2-City.mmdb
GEOIP_ASN_DB=/opt/geoip/GeoLite2-ASN.mmdb
ABUSEIPDB_KEY=
ANOMALY_THRESHOLD=3.0
BRUTE_FORCE_THRESHOLD=20
```

## Deployment (LXC)

```
Browser → http://192.168.2.242:8001
  └── NGINX
        ├── /       → /var/www/ipnginx/dist/   (Vue-App)
        └── /api/   → FastAPI Port 8000
```

Erster Einsatz: SchulOrdnungHems (bestehender LXC-Container).

## Abhängigkeiten

```toml
# Backend
fastapi, uvicorn, sqlalchemy[asyncio], aiosqlite, aiofiles
pydantic, pydantic-settings, geoip2, user-agents, httpx
pytest, pytest-asyncio, httpx (dev)

# Frontend
vue, typescript, vite, tailwindcss
echarts, vue-echarts, pinia, vitest
```

## Verifikation (End-to-End)

1. `systemctl start ipnginx` → Service läuft
2. Testzeile an `/var/log/nginx/access.log` anhängen
3. `GET /logs` → Eintrag mit Geo + UA-Daten
4. `GET /threats` → SQL-Injection-Test schlägt an
5. `GET /dashboard` → Vue-App im Browser sichtbar

## Vollständiger Plan

Interner Plan-File: `/home/norbert/.claude/plans/zeige-mir-bitte-unseren-fluffy-yeti.md`
