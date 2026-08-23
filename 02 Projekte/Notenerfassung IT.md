---
title: Notenerfassung IT
tags:
  - projekt/aktiv
  - schule/it
status: in-progress
date: 2026-06-16
updated: 2026-06-23
---

# Notenerfassung IT

Webanwendung zur Notenerfassung für Lehrer, mit automatischer Übertragung nach LUSD (hessisches Schulverwaltungssystem).

## Architektur

### Backend (Flask + SQLite)

Sauber nach [[SOLID-Prinzipien]] geschichtet:

| Schicht | Dateien |
|---|---|
| Models | `Student`, `Subject`, `Grade`, `SubjectHours` |
| Repositories | Interface + Implementierung (SQLAlchemy) |
| Services | `GradeCalculationService`, `StudentService`, `SubjectService`, `SeedService` |
| API | Flask Blueprints: students, subjects, grades, calculations |
| Schemas | Marshmallow-Serialisierung |

- **Datenbank:** SQLite, gespeichert in Docker-Volume `db-data`
- **Migrations:** Alembic (1 angewandte Version: `585e00a39bb6`)
- **Stack:** Flask 3.1, SQLAlchemy, Gunicorn, Flask-CORS

### Frontend (Vue 3 + TypeScript)

SOLID-Struktur: `Interfaces → Implementations → Composables → Views`

**Views:**
- `GradeEntryView` — Notenerfassung pro Schüler
- `StudentManagementView` — Schülerverwaltung
- `HoursSettingsView` — Stunden-Einstellungen pro Fach

**API-Calls:** Relativ auf `/api` (Proxy via nginx) → produktionstauglich

### Docker Compose

```
backend  → Gunicorn auf Port 5000
frontend → nginx auf Port 80 (served + Proxy zu /api)
```

> [!warning] Kein HTTPS, kein Reverse Proxy
> Das aktuelle `docker-compose.yml` hat kein TLS und keinen externen Reverse Proxy. Muss für Proxmox-Deployment ergänzt werden.

---

## LUSD-Integration

### Alte Architektur (lokal, wird abgelöst)

- `lusd_bridge.py` — HTTP-Server auf `localhost:5001`, steuert LUSD per CDP/Playwright
- `start.sh` / `stop.sh` — Starten Docker, Bridge und Chromium mit `--remote-debugging-port=9222`
- Button „→ LUSD" in `GradeEntryView.vue` ruft hardcoded `http://localhost:5001` auf

**Problem:** Bitwarden-Extension verweigert Verbindung wenn CDP-Debug-Port aktiv ist. Nicht skalierbar für mehrere Lehrer.

### Neue Zielarchitektur

1. **App auf Proxmox** (LXC, Docker, HTTPS via Reverse Proxy)
2. **Browser-Extension** (Manifest V3) — läuft im normalen Browser, kein Debug-Modus nötig

**LUSD-Details für Extension:**
- Login: `myapplications.microsoft.com` (Microsoft App Proxy + Azure AD + Authy 2FA)
- Noten-iframe: `SchuelerForm.aspx`
- Eingabe-Widgets: Telerik RadComboBox → `click(3)` + type + Tab
- Fach-Mapping: Deutsch→D, Englisch→E, Politik→POWI, Sport→SPO, Reli/Ethik→RKA, LF1–12→LF01–LF12

---

## Roadmap

### Phase 1 — Proxmox-Deployment

#### Schritt 1: LXC anlegen (Ansible)

- [x] Ansible-Playbook erstellt: `~/homelab-ansible/proxmox-setups/create_notenIT_lxc.yml`
- [x] Playbook erfolgreich ausgeführt — LXC angelegt und gestartet
- [x] Container per SSH erreichbar, Ansible-Ping erfolgreich (`pong`)

**LXC-Konfiguration (läuft):**
- VMID: `901`, Node: `pve`, Hostname: `noten-it`, IP: `192.168.2.6`
- Debian 13, 10 GB, 1 Core, 1024 MB RAM, 512 MB Swap
- `nesting=1` aktiviert (Pflicht für Docker), `unprivileged: true`, `onboot: true`
- In `hosts.ini` eingetragen als `noten_it_lxc`

> [!note] Ansible-Konventionen im Repo
> Repo: `~/homelab-ansible/` — nutzt `api_password` (kein Token-Muster), Vault in `geheimnisse.yml`, Python-Interpreter auf Proxmox-Host: `/opt/ansible-venv/bin/python`
> SSH-Verbindung zum Container erfordert explizit `-i /root/.ssh/id_rsa` (pubkey wurde injiziert, aber Standard-Auth greift nicht automatisch)

#### Schritt 2: Ansible-Struktur auf Rollen umgebaut (2026-06-19)

- [x] `homelab-ansible/` auf Rollen-Architektur umgestellt (Option B: Rollen ins bestehende Repo)
- [x] `roles/proxmox_lxc/` — parametrisierbar, wiederverwendbar für andere LXC
- [x] `roles/docker_install/` — deb822-Format (Debian 13 kompatibel, kein apt_key)
- [x] `roles/app_deploy/` — rsync + `docker compose up -d --build`
- [x] `group_vars/proxmox_host.yml` + `group_vars/lxc_containers.yml` angelegt
- [x] `site.yml` als Haupt-Einstiegspunkt mit Tags (`--tags lxc/docker/app`)
- [x] Docker 29.6.0 läuft im LXC

> [!note] Ansible-Konventionen im Repo (aktualisiert)
> `~/homelab-ansible/` — Vault-Secrets in `geheimnisse.yml` (werden in `proxmox_lxc`-Rolle per `include_vars` geladen, nicht in `site.yml` — so läuft `--tags docker/app` ohne `--ask-vault-pass`)
> SSH-Key von lokalem Rechner (`/home/norbert/.ssh/id_rsa`) ist in LXC 901 eingetragen

#### Schritt 3: App deployen + NPM einrichten (2026-06-19)

- [x] App-Code per rsync auf LXC übertragen, Docker-Stack gestartet
- [x] `backend/.dockerignore` angelegt — `data/` und `*.db` ausgeschlossen (verhindert alte DB im Image)
- [x] Nginx Proxy Manager eingerichtet: `notenit.softexceptions.com` → `192.168.2.6:80`
- [x] Let's Encrypt Zertifikat aktiv, HTTPS erzwungen

#### Schritt 4: SOLID-Refactoring + CORS-Härtung (2026-06-19)

- [x] CORS eingeschränkt auf `https://notenit.softexceptions.com` (war `*`)
- [x] `backend/app/services/lusd_service.py` — `ILusdService` Protocol, alle 3 Repos via Constructor injiziert (DIP)
- [x] 11 TDD-Tests in `backend/tests/test_lusd_service.py` — FakeRepos, Unit- und Integrationstests, alle grün
- [x] `backend/.dockerignore` — `data/`, `*.db`, `*.sqlite`, `.env` ausgeschlossen

### Phase 2 — Browser-Extension (2026-06-19 implementiert)

#### Architektur: Window-Nachrichten-Bus

```
GradeEntryView.vue
  └─ postMessage({type:'LUSD_TRANSFER', studentId})
       └─ bridge.js (Content Script auf notenit.softexceptions.com)
            └─ chrome.runtime.sendMessage → background.js (Service Worker)
                 └─ fetch /api/students/{id}/lusd-data
                 └─ chrome.webNavigation.getAllFrames → SchuelerForm.aspx-Frame
                      └─ chrome.tabs.sendMessage → content.js
                           └─ Telerik RadComboBox befüllen (80ms-Delay)
                 └─ Ergebnis zurück an bridge.js → postMessage({type:'LUSD_RESULT'})
```

#### Implementierte Dateien

| Datei | Zweck |
|---|---|
| `browser-extension/manifest.json` | Manifest V3, Permissions: tabs, webNavigation |
| `browser-extension/bridge.js` | Content Script auf App-Domain — postMessage-Brücke |
| `browser-extension/background.js` | Service Worker — API-Call + Tab-Routing |
| `browser-extension/content.js` | Läuft in LUSD-Frames — Telerik-Felder befüllen |
| `browser-extension/popup.html/.js` | Standalone-Popup für manuelle Übertragung |

#### Frontend SOLID-Refactoring

```
ILusdExtensionService (Interface)
  └─ LusdExtensionService (Implementation, postMessage-Logik)
       └─ useLusdTransfer (Composable, Vue-Reaktivität)
            └─ GradeEntryView.vue (inject + Composable, 25 Zeilen Script)
```

- [x] `frontend/src/services/interfaces/ILusdExtensionService.ts`
- [x] `frontend/src/services/implementations/LusdExtensionService.ts`
- [x] `frontend/src/composables/useLusdTransfer.ts`
- [x] `frontend/src/services/container.ts` — `LusdExtensionServiceKey` hinzugefügt
- [x] `frontend/src/views/GradeEntryView.vue` — auf inject + Composable umgestellt

#### Schritt 5: Mobile-first Refactoring (2026-06-19)

- [x] `AppHeader.vue` — 3-Zeilen-Layout auf Mobile (Titel+Toggle | Nav | Select), 1 Zeile auf Desktop
- [x] `GradeTable.vue` + `GradeRow.vue` — Spalten „Stunden" und „Gewichtet" auf Mobile ausgeblendet (`hidden md:table-cell`)
- [x] `GradeInput.vue` — Touch-Target h-10 (40px), volle Breite auf Mobile
- [x] `GradeEntryView.vue` — Schülername + LUSD-Button stacken auf Mobile, Button volle Breite
- [x] `StudentManagementView.vue` — Formular stackt auf Mobile, Klasse-Spalte ausgeblendet, Buttons py-2
- [x] `HoursSettingsView.vue` — Number-Inputs py-2 auf Mobile, Aktion-Spalte ausgeblendet
- [x] `AverageDisplay.vue` — „Gesamt&shy;durchschnitt" mit softem Trennzeichen, bricht auf Mobile korrekt um
- [x] Deployed auf `https://notenit.softexceptions.com` — mobil getestet, sieht gut aus

#### Schritt 6: Firefox-Support via webextension-polyfill (2026-06-19)

- [x] `browser-polyfill.js` heruntergeladen (Mozilla, 10 KB)
- [x] `manifest.json` — `browser_specific_settings` für Firefox (Gecko-ID `notenerfassung@softexceptions.com`, `strict_min_version: "128.0"`), Polyfill als erstes Script in `content_scripts`
- [x] `background.js` — `importScripts('browser-polyfill.js')` + alle `chrome.*` → `browser.*`
- [x] `bridge.js`, `content.js`, `popup.js` — alle `chrome.*` → `browser.*`
- [x] `popup.html` — Polyfill-Script vor `popup.js` eingebunden

> [!note] Warum Firefox 128 als Mindestversion?
> Erst ab FF 128 unterstützt Firefox Service Worker in Manifest V3 vollständig.
> Der Polyfill erkennt automatisch den Browser — in Firefox nutzt er native `browser.*`-APIs, in Chrome wrappt er `chrome.*`.

> [!warning] Morgen testen
> **Chrome:**
> 1. `chrome://extensions/` → Entwicklermodus → „Entpackte Erweiterung laden" → Ordner `browser-extension/`
> 2. Extension-ID notieren → in `backend/app/__init__.py` zu CORS-Origins:
>    `origins=["https://notenit.softexceptions.com", "chrome-extension://DEINE_ID"]`
> 3. Nach CORS-Fix: `ansible-playbook -i hosts.ini site.yml --tags app`
> 4. Live-Test: Schüler auswählen → „→ LUSD"-Button → Telerik-Inputs prüfen
>
> **Firefox:**
> 1. `about:debugging` → „Dieser Firefox" → „Temporäres Add-on laden" → `manifest.json` im Ordner `browser-extension/`
> 2. Gleicher Live-Test wie Chrome

### Phase 3 — Extension-Verteilung (2026-06-20)

#### Build-System: @crxjs → esbuild (2026-06-20)

- [x] @crxjs/vite-plugin entfernt — erzeugte dynamischen Loader der in gepackten .crx nicht funktionierte
- [x] Neues Build-Skript `build.mjs` mit esbuild direkt
  - `bridge.ts` + `content.ts` → IIFE (self-contained, kein dynamischer Import)
  - `background.ts` → ES-Modul (Service Worker)
- [x] `manifest.json` aktualisiert: direkte Dateipfade (`bridge.js`, `content.js`, `background.js`)

#### Extension-Detection: PostMessage → DOM-Attribut (2026-06-20)

- [x] `bridge.ts` setzt beim Laden `document.documentElement.setAttribute('data-notenerfassung-active', 'true')`
- [x] `ExtensionDetectionService.ts` prüft nur noch dieses Attribut (synchron, kein Timeout)
- [x] Tests angepasst (6 Tests, alle grün)

#### Extension-ID (2026-06-20)

- Aktuelle ID: `ahfmlgccaajcjabmapikobkgjleobhdb` (aus `notenerfassung.pem`)
- CORS in `backend/app/__init__.py` auf neue ID aktualisiert
- `.pem`-Datei in Vaultwarden gesichert

#### Extension-Download in der App (2026-06-20)

- [x] `/extension`-Seite: Download-Button → `notenerfassung.zip` (nicht .crx, wegen Browser-Interception)
- [x] Backend-Endpunkt `GET /api/extension/download` mit `Cache-Control: no-store`
- [x] Installationsanleitung: ZIP umbenennen zu .crx → Drag & Drop auf `chrome://extensions/` → F5

> [!important] F5 nach Installation nötig
> Content Scripts werden nur in neue Seitenladungen injiziert. Nach der Extension-Installation immer F5 auf der App-Seite drücken.

> [!warning] Chromium 149: gepackte .crx blockiert Content Scripts
> Bei sideloaded .crx (nicht aus Web Store) blockiert Chromium 149 Content Scripts trotz aktiviertem Toggle.
> Workaround: „Entpakte Erweiterung laden" → dist/-Ordner. .crx-Download funktioniert nur auf älteren Versionen oder echtem Chrome.

#### Chrome Web Store Vorbereitung (2026-06-20)

- [x] Icons erstellt: `icons/icon16.png`, `icon48.png`, `icon128.png` (blaues „N" auf blauem Grund)
- [x] `browser_specific_settings` (Gecko) aus Manifest entfernt für Chrome-Web-Store-Version
- [x] ZIP-Paket erstellt: `notenerfassung-webstore.zip` (manifest.json direkt im Root)
- [x] Datenschutzseite angelegt: `https://notenit.softexceptions.com/privacy`
- [x] Google-Entwicklerregistrierung abgeschlossen
- [x] Extension im Chrome Web Store eingereicht (wartet auf Prüfung, 1–3 Werktage)

#### Phase 4 — Sicherheit + Chrome Web Store Fix (2026-06-21)

##### Privacy-Policy-Fix
- [x] Problem: Basic Auth schützte auch `/privacy` → Google-Crawler bekam 401
- [x] `frontend/public/privacy.html` — standalone statische HTML-Seite (kein Vue, kein Auth nötig)
- [x] `nginx.conf` — `location = /privacy { auth_basic off; }` vor dem Auth-Block
- [x] Chrome Web Store: neue Version eingereicht mit Verweis auf Datenschutz-URL

##### API-Key-Schutz
- [x] Problem: `/api/` war ohne Auth öffentlich zugänglich (Schülerdaten frei abrufbar)
- [x] `backend/app/api/auth.py` — `require_api_key()` Middleware (prüft `X-Api-Key` Header)
- [x] `backend/app/__init__.py` — Hook auf alle geschützten Blueprints (students, subjects, grades, calculations)
- [x] `docker-compose.yml` — `API_KEY` Umgebungsvariable für Backend und Frontend
- [x] `nginx.conf` → Template (`${API_KEY}`) — nginx setzt Key automatisch beim Proxyen zum Backend
- [x] `frontend/Dockerfile` — Template-Mechanismus: `nginx.conf` → `/etc/nginx/templates/default.conf.template`
- [x] API-Key in Vaultwarden gesichert

##### Extension: API-Key-Integration
- [x] `src/services/LusdApiClient.ts` — `X-Api-Key` Header bei jedem fetch
- [x] `src/background.ts` — Key aus `browser.storage.sync` lesen
- [x] Neues Popup (`popup.html` + `src/popup.ts`) — Einstellungs-UI zum Eintragen des Keys
- [x] `manifest.json` — `storage` Permission + `default_popup: "popup.html"` hinzugefügt
- [x] `build.mjs` — Popup-Kompilierung + Kopieren ergänzt
- [x] `dist.crx` via Chrome „Erweiterung packen" erstellt (mit `dist.pem`)
- [x] `backend/static/notenerfassung.crx` aktualisiert
- [x] `notenerfassung-webstore.zip` neu erstellt und im Web Store eingereicht

> [!success] Web Store Extension aktiv (2026-06-23)
> CORS-Origin für Web Store ID `gnfmipbngblllffpolcmbgcokooiedpl` eingetragen und deployt. Live-Test erfolgreich — Transfer zur LUSD funktioniert über die Web Store Extension.

> [!note] Installationsweg für Kollegen (bis Web Store freigegeben)
> Download-Button auf `/extension` → `notenerfassung.zip` → umbenennen zu `.crx` → Drag & Drop auf `chrome://extensions/`
> Danach Key im Extension-Popup (Einstellungen-Tab) eintragen — Key steht in Vaultwarden

#### Basic Auth (2026-06-20)

- [x] `frontend/.htpasswd` erstellt: `lehrer` / `-lusd-` (apr1-Hash)
- [x] `nginx.conf` — Basic Auth auf `/` (schützt Web-UI, `/api/` bleibt offen für Extension)
- [x] `frontend/Dockerfile` — `.htpasswd` wird nach `/etc/nginx/.htpasswd` kopiert
- [x] Deployed (Ansible `--tags app`)

**Zielgruppe Browser:**
| Browser | Lösung |
|---|---|
| Chrome Windows | Chrome Web Store |
| Edge Windows | Chrome Web Store (kann Chrome-Extensions installieren) |
| Firefox | Mozilla AMO (noch ausstehend) |
| Chromium Linux | dist/-Ordner als unpacked Extension |

---

## Phase 5 — Session-Lock (2026-06-21 implementiert)

### Architektur

```
App.vue (onMounted → checkStatus)
  ├─ state = 'checking'  → "Verbinde …" Ladescreen
  ├─ state = 'entering'  → SessionEntryModal (Name eingeben)
  ├─ state = 'active'    → SessionBar + normales App-Layout
  └─ state = 'waiting'   → SessionWaitView (automatisches Polling alle 30 Sek)

useSession (Composable — Zustandsmaschine)
  └─ ISessionService / SessionService (fetch → /api/session/*)
```

### Backend

| Datei | Zweck |
|---|---|
| `backend/app/services/session_service.py` | `ISessionService` (Protocol) + `SessionService` (In-Memory) |
| `backend/app/api/session.py` | Blueprint `/api/session/{status,lock,heartbeat,release}` |
| `backend/tests/test_session_service.py` | 19 TDD-Tests (11 Unit + 8 Integration), alle grün |

**Entscheidungen:**
- In-Memory (kein DB) — für 2–3 Lehrer ausreichend, kein Migrations-Overhead
- `_now()` mit timezone-aware UTC — kein deprecated `utcnow()`
- `math.ceil()` für Minuten → 30 bleibt 30 trotz Millisekunden-Delay
- Nebenbefund: Pre-existing Bug in `__init__.py` (`bp.before_request` nach Registration) gefixt — auth jetzt via `@app.before_request` mit Prefix-Filter
- **Gunicorn `--workers 1`** — In-Memory-State verträgt sich nicht mit mehreren Prozessen (Split-Brain); für 2–3 Lehrer reicht 1 Worker
- **Timeout: 30 Minuten** — bei Browser-Absturz automatische Freigabe nach 30 Min.

### Frontend

| Datei | Zweck |
|---|---|
| `frontend/src/types/session.ts` | `SessionStatus`, `AcquireResult`, `SessionState` Union |
| `frontend/src/services/interfaces/ISessionService.ts` | Interface |
| `frontend/src/services/implementations/SessionService.ts` | axios-basierte Implementierung |
| `frontend/src/composables/useSession.ts` | Zustandsmaschine + Heartbeat/Polling/Countdown-Timer + beforeunload |
| `frontend/src/components/session/SessionEntryModal.vue` | Name-Eingabe-Modal |
| `frontend/src/components/session/SessionBar.vue` | Blaue Leiste: Name, Countdown, Verlängern, Fertig + rote Ablauf-Warnung |
| `frontend/src/components/session/SessionWaitView.vue` | Wartebildschirm mit Polling-Hinweis |

**Token-Verwaltung:** `sessionStorage` (wird bei Browser-Schließen gelöscht — gewollt)

**Browser-Schließen:** `beforeunload` + `fetch keepalive` sendet Release zuverlässig ans Backend.
`sendBeacon` wäre ungeeignet — kann keine Custom-Header setzen, `X-Api-Key` fehlt dann.
(nginx injiziert den Key NICHT — er steckt als `VITE_API_KEY` im axios-Client des Bundles)

**Ablauf-Warnung:** Rotes pulsierendes Banner bei ≤ 5 Min. verbleibend (`showExpiryWarning` Computed).

**Activity-based Heartbeat:** Heartbeat wird nur gesendet wenn in den letzten 5 Min. Aktivität
war (mousemove / click / keydown / touchstart). Bei Inaktivität kein Heartbeat → Session läuft
nach 30 Min. serverseitig ab. Ohne diese Logik blieb die Session unbegrenzt aktiv.

**Verhalten:**

| Szenario | Ergebnis |
|---|---|
| Lehrer arbeitet aktiv | Heartbeat alle 5 Min., Session bleibt |
| Lehrer geht weg (keine Eingabe) | Kein Heartbeat, Countdown läuft ab |
| Nach 30 Min. Inaktivität | Session frei, Kollege kann einsteigen |
| Tab/Browser normal geschlossen | Sofort frei (keepalive-Request) |
| Browser abgestürzt / Strom weg | Nach 30 Min. automatisch frei |
| „Fertig" geklickt | Sofort frei |
| „Verlängern" geklickt | Sofort Heartbeat + Activity-Reset |

---

## Entwicklungsregeln (2026-06-16 festgelegt)

### Arbeitsweise
- **TDD** — jedes Feature nach Red → Green → Refactor, kein Produktionscode ohne fehlschlagenden Test
- **SOLID** — `/python-solid` bei `.py`, `/vue-solid` bei `.vue` (automatisch)
- **API-Stil** — REST bleibt (GraphQL wurde erwogen und verworfen: Datenmodell zu flach, Extension-Code würde unnötig komplexer)

### Verfügbare Skills im Projekt

| Skill | Verfügbar über | Zweck |
|---|---|---|
| `/tdd` | Projekt-Symlink | TDD-Zyklus |
| `/python-solid` | Global | Flask-Backend |
| `/vue-solid` | Global | Vue-Frontend |
| `/obsidian-markdown` | Global (System) | Vault-Notizen |
| `/obsidian-cli` | Global (System) | Vault-Operationen |

### Dokumentation
- context7 wird für alle Library-Docs genutzt (Flask, Vue, Alembic, Vite, etc.) — kein Trainingswissen

---

## Deploy-Workflow (ab 2026-06-23)

```
1. Desktop: Code ändern + git push origin main
2. SSH auf ansible-control (192.168.2.101):
   ansible-playbook playbooks/noten_it.yml --ask-vault-pass --tags app
```

> [!important] Nicht mehr vom Desktop deployen
> Deployment läuft ausschließlich vom `ansible-control`-LXC (192.168.2.101). Der LXC holt den Code selbst per `git pull` aus GitHub. Rsync vom Desktop ist abgelöst.

Detaillierte Ansible-Doku: [[Ansible]]

---

## Sonstiges

- Passwörter: selbst gehostetes [[Vaultwarden]]
- Klasse: BS12IT (hardcoded als Default in `Student`-Model)
