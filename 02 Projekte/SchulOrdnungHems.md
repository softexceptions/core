---
title: SchulOrdnungHems
tags:
  - projekt
  - vue
  - fastapi
  - aktiv
status: aktiv
stand: 2026-05-28
deployed: true
url: https://ordnung.softexceptions.com
---

# SchulOrdnungHems

Interaktive, mobile-first Landing Page für die Schulordnung der [[Heinrich-Emanuel-Merck-Schule]] (HEMS), Berufsschulzentrum Nord Darmstadt.

> [!info] Ziel
> Das PDF der Schulordnung ist für Berufsschüler:innen (16–25 Jahre) unpraktisch. Diese App macht die Inhalte mobil zugänglich, interaktiv und verständlich.

## Stack

| Schicht | Technologie |
|---|---|
| Frontend | Vue 3 + TypeScript + Vite + Tailwind CSS 3 |
| Backend | FastAPI + Python |
| Design | Claymorphism, Nunito (Headings) + DM Sans (Body) |

## Farbpalette (aus HEMS_Bild.webp)

| Token        | Hex       | Bedeutung                         |
| ------------ | --------- | --------------------------------- |
| `hems-navy`  | `#1a1d21` | Anthrazit-Schwarz (Metallfassade) |
| `hems-blue`  | `#2d6fa8` | Stahlblau — Primär-Akzent         |
| `hems-sky`   | `#6aadc8` | Hellblau (Himmel im Glas)         |
| `hems-steel` | `#4a6070` | Stahlgrau — Sekundär-Akzent       |
| `hems-glass` | `#8da8b8` | Glasgrau (Spiegelton)             |
| `hems-bg`    | `#f0f3f5` | Kühles Silberweiß                 |
| `hems-muted` | `#68808c` | Kühles Grau                       |

## Frontend-Architektur (SOLID)

```
src/services/interfaces/      → IQuizService, IChatbotService  (DIP)
src/services/implementations/ → QuizService, ChatbotService
src/services/container.ts     → InjectionKey + createServices()
src/composables/              → useFocusedSection, useQuiz, useChatbot, useScrollProgress
src/data/schulordnung.ts      → 13 Abschnitte, 8 Szenarien, 7 Quiz-Fragen, 10 Chatbot-Einträge
```

## Implementierte Features

- [x] Hero mit HEMS-Branding + 3 Leitsätze
- [x] Bottom-Navigation (mobil) / Top-Navigation (Desktop)
- [x] Scroll-Progress-Bar
- [x] Quicklinks-Dashboard — 8 Karten → öffnet Regel-Accordion per `useFocusedSection`
- [x] „Was passiert wenn…?" — Accordion mit 8 Szenarien
- [x] Schulordnung — 13 aufklappbare Abschnitte mit Fokus-Highlight
- [x] Quiz — 7 Fragen + Badge-System (Bronze / Silber / Gold)
- [x] Regelbasierter Chatbot mit Vorschlägen
- [x] QR-Code (nur Desktop, fix unten rechts, zeigt aktuelle URL)
- [x] Dev-Server mit `--host` für Netzwerkzugriff

## Bekannte Stolperfallen

> [!warning] CSS: opacity + animate-float-up
> `opacity-X` auf Karten mit `animate-float-up` (animation-fill-mode: both) wird überschrieben.  
> Immer `!opacity-X` (Tailwind important-Modifier) verwenden.

> [!warning] Tailwind-Config wird nicht hot-reloaded
> Bei Farbänderungen in `tailwind.config.js`: Dev-Server neu starten.

> [!tip] Hintergrundbild
> Bild auf `body` setzen (nicht `:root`), da `body` die opake Fallback-Farbe trägt.

## Start-Befehle

**Frontend** (Port 5173):
```
cd frontend
npm run dev
```

Mit Netzwerkzugriff für andere Geräte im LAN:
```
npm run dev -- --host
```

**Backend** (Port 8000):
```
cd backend
uvicorn main:app --reload
```

> [!tip] Proxy
> Vite leitet `/api` automatisch an `http://localhost:8000` weiter (`vite.config.ts`). Für API-Features (Chatbot, Quiz) müssen beide Server laufen.

## Dev-Server (LAN)

```
http://192.168.2.45:5173/    (mit --host Flag)
```

## Deployment (Proxmox LXC)

- [x] SSH eingerichtet
- [x] Nginx + Python3 + venv installiert (Debian 13)
- [x] Frontend gebaut (`npm run build`) und nach `/var/www/schulordnung/` übertragen
- [x] Backend nach `/opt/schulordnung/` übertragen, venv + Dependencies installiert
- [x] systemd-Service `schulordnung-backend` → gunicorn -w 4 + UvicornWorker auf Port 8000
- [x] Nginx konfiguriert: statische Dateien + `/api/`-Proxy zu FastAPI
- [x] App läuft unter **http://192.168.2.242:8125**
- [x] Frontend-Services (Quiz, Chatbot) auf echte API-Aufrufe umgestellt

### Workflow-Hinweis

> [!important] Norbert schaut die App immer auf dem LXC-Container an (http://192.168.2.242:8125)
> Der Dev-Server (localhost:5173) ist für ihn nicht relevant. Änderungen werden erst nach Build + Deploy sichtbar.

### Deploy-Befehl (Frontend)

```bash
cd /home/norbert/Code/SchulOrdnungHems/frontend && npm run build
scp -r /home/norbert/Code/SchulOrdnungHems/frontend/dist/* root@192.168.2.242:/var/www/schulordnung/
```

Danach im Browser **Strg+Shift+R** (Hard Reload) ausführen.

### Deploy-Befehl (Backend)

Bei Änderungen an einzelnen Backend-Dateien (z. B. `schulordnung_data.py`):

```bash
# 1. Geänderte Datei kopieren (Pfad anpassen falls andere Datei)
scp /home/norbert/Code/SchulOrdnungHems/backend/app/data/schulordnung_data.py \
    root@192.168.2.242:/opt/schulordnung/app/data/schulordnung_data.py

# 2. Service neu starten
ssh root@192.168.2.242 "systemctl restart schulordnung-backend"

# 3. Prüfen (API antwortet?)
curl -s http://192.168.2.242:8125/api/quiz/questions | python3 -m json.tool
```

> [!tip] API-Pfade
> Nginx entfernt `/api/` und leitet an FastAPI weiter. Korrekte Endpunkte:
> - `GET /api/quiz/questions` → Quiz-Fragen
> - `POST /api/quiz/submit` → Quiz auswerten
> - `GET /api/rules` → Schulordnungs-Abschnitte
> - `POST /api/chatbot` → Chatbot-Antwort

> [!tip] Nginx-Stolperfalle
> `sites-enabled/default` Symlink muss entfernt werden — sonst fängt der Default-Server alle Requests ab.
> `redirect_slashes=False` in `main.py` verhindert Redirect-Loop beim API-Proxy.

# Link

[Schulordnung](https://ordnung.softexceptions.com/)

## Letzte Änderungen (2026-06-23)

### PDF-Link zur offiziellen Schulordnung eingebaut

URL: `https://hems.de/wp-content/uploads/2026/06/2026_Schulordnung_HEMS.pdf`

Zwei Stellen in `RulesSection.vue`:
- **Header:** Clay-Badge `📄 Offizielles PDF` zwischen H2 und Filter-Hinweis — immer sichtbar
- **Abschluss-Karte:** Textlink `📄 Vollständige Schulordnung als PDF` unterhalb des Zitat-Satzes

## Letzte Änderungen (2026-06-22)

### Quiz — Fragen überarbeitet

- **Frage 3** komplett neu: „Wann kann ein Antrag auf Beurlaubung gestellt werden?" — 4 neue Antwortoptionen, richtig: „Nur in besonders begründeten Ausnahmefällen", Erklärung gekürzt
- **Frage 6** präzisiert: „Wie verlässt du den Unterrichtsraum am Ende des Schultages korrekt?"

### Quiz — Badge-System: kein_badge für 0 richtige Antworten

Vorher bekam man Bronze auch bei 0 richtigen Antworten (Catch-all). Neue Logik:

| Score | Badge |
|---|---|
| ≥ 85% | Gold |
| ≥ 60% | Silber |
| ≥ 1 richtig | Bronze |
| 0 richtig | Kein Badge (😔) |

Geänderte Dateien: `domain/models.py`, `presentation/schemas/quiz_schema.py`, `application/services.py` (Backend) + `types/index.ts`, `QuizSection.vue` (Frontend).

### Quicklinks — Beurlaubung durch Konsequenzen ersetzt

„Beurlaubung" und „Krank melden" zeigten beide auf §6 (Fehlzeiten). „Beurlaubung" wurde durch ⚖️ „Konsequenzen" (→ §12) ersetzt.

### Dunkelmodus — Zurück-Button

Text war kaum lesbar. Fix: `dark:bg-slate-600 dark:text-white` in `QuizSection.vue`.

### Szenarien — Texte aktualisiert

- **Verspätung:** Formulierung sachlicher, WebUntis-Tipp bleibt
- **Krank:** Hinweis auf schulformspezifische Sonderregelungen ergänzt
- **Beurlaubung:** Grundsatz „nicht möglich" vorangestellt, Ausnahmefälle (Beerdigung, Hochzeit) konkretisiert

### Chatbot — Waffen-Keyword getrennt

`waffe` und `messer` waren im gleichen `KeywordEntry` wie `alkohol`/`drogen` — Antwort erwähnte nur Drogen. Jetzt eigener Eintrag: „Das Mitbringen von Waffen und gefährlichen Gegenständen ist verboten."

## Letzte Änderungen (2026-05-28)

### Neue Schulordnung `_ab 08-2026_final.docx` eingearbeitet

Quelle: `Schulordnung der HEMS_ab 08-2026_final.docx` (verglichen mit `_neu_ab 08-2026.docx`). Beide Dateien liegen im Projektroot.

- **§ 2 Geltungsbereich:** „sonstiges Personal" ergänzt; „Gebäude" → „Gelände"
- **§ 3 Schulgelände:** Hinweis auf öffentliche Parkflächen am Nordeingang entfernt; quickInfo vereinfacht
- **§ 8 Schuleigentum:** „Schulbücher" → „Entliehene Schulmaterialien (Schulbücher, Tablets)"
- **§ 9 Verbote:** Rauchen-Ausnahme für Volljährige entfernt; Verbote-Formulierung vereinfacht
- **§ 10 Digitale Geräte:** Laptops ergänzt; „prinzipiell ausgeschaltet"
- **§ 11 Versicherungsschutz:** „nicht versichert" → „keine Haftung" (korrekte Formulierung)
- **§ 13 Ergänzende Ordnungen:** neuer Punkt „Schulformspezifische Regelungen bei Fehlzeiten"
- **Chatbot:** Rauchen (Volljährige-Satz entfernt), Handy (Laptops + prinzipiell ergänzt)

### „Unser Ziel"-Abschluss-Karte

Abschluss-Statement der Schulordnung als hervorgehobene Karte nach § 13 eingebaut (`RulesSection.vue`):
- `hems-navy` Hintergrund, weißer Text, Claymorphism-Stil
- ▲-Button rechts oben → scrollt via `scrollToTop()` zurück zum Hero

### Quiz — Robustheit ohne Backend

`useQuiz.ts` `onMounted` mit `try/catch/finally` abgesichert: Backend-Fehler setzen `isLoading = false`, `questions` bleibt leer.
`QuizSection.vue` zeigt drei States: Ladeindikator → „Backend nicht erreichbar" → Fragen.

### Dark Mode — Quiz-Antwort-Buttons

`dark:` Klassen in dynamischen `:class`-Objekten werden von Tailwind JIT nicht zuverlässig ins Bundle aufgenommen. Lösung: CSS-Klasse `quiz-option` + Regeln in `main.css` (nicht im Template).

### UI-Kleinigkeiten

- README komplett neu geschrieben (war falsch — beschrieb Lehrplan statt App)
- 🎯 → 🔍 bei WhatIfSection-Badge und StickyNav „Wenn?"-Eintrag
- WhatIfSection-Badge: `bg-hems-glass` → `bg-hems-steel` (konsistent mit Quicklinks)

## Letzte Änderungen (2026-05-12)

- **Quiz — Error-Handling ergänzt:** `useQuiz.ts` hatte kein `try/catch` in `submit()` — bei API-Fehlern passierte nichts (stilles Versagen). Ergänzt: `isSubmitting`-State (Button wird während des Calls deaktiviert), `submitError`-State (Fehlermeldung im UI), `try/catch/finally`-Block
- **QuizSection.vue — Lade-Spinner + Fehlermeldung:** "Auswerten"-Button zeigt während des Submits einen Spinner und ist deaktiviert; bei Fehler erscheint eine rote Meldung unterhalb der Navigation
- **Backend — Gunicorn mit 4 Workern:** systemd-Service von `uvicorn` (1 Worker) auf `gunicorn -w 4 -k uvicorn.workers.UvicornWorker` umgestellt → 1 Master + 4 Worker-Prozesse für echte Parallelität bei Gleichzeitig-Zugriff

> [!info] Hintergrund
> Beim Lehrer-Test (Gesamtkonferenz, ~40 Personen gleichzeitig) hat der "Auswerten"-Button bei manchen nicht reagiert. Ursache: fehlendes Error-Handling im Frontend + einzelner uvicorn-Worker. Lasttest mit `ab -n 40 -c 40` nach dem Fix: 0 Failed requests, längster Request 15ms.

## Letzte Änderungen (2026-05-05)

- **Navigation — Quiz durch Wenn? ersetzt:** Der `Quiz`-Button war redundant (Quiz ist bereits als Quicklink-Karte erreichbar). Ersetzt durch `🎯 Wenn?` → springt zur WhatIfSection (`id="whatif"`). Reihenfolge: Start | Wichtig | **Wenn?** | Regeln | Fragen | 🌙
- **Stundenplan — fehlende Pausen ergänzt:**
  - Pause `16:45–17:00` nach der 9./10. Stunde eingefügt
  - Pause `18:30–18:40` nach der 11./12. Stunde eingefügt

## Letzte Änderungen (2026-05-04)

- **Hero-Leitsätze klickbar:** Die drei Karten (Respektvolles Miteinander, Verantwortung übernehmen, Achtsam mit Ressourcen) klappen auf und zeigen je 4 Regeln aus der Schulordnung (Abschnitte 7, 5&6, 8) mit Quellenangabe


- **API-Anbindung:** QuizService + ChatbotService nutzen jetzt `fetch()` statt lokale Daten
- **Quiz-Fortschritt:** Balken basiert auf beantworteten Fragen (nicht Position) → startet bei 0%
- **Quicklinks:** Beim Schließen einer fokussierten Regelkarte → Scroll zurück zu Quicklinks
- **Backend-Fix:** q6 + q7 fehlten in `schulordnung_data.py` — ergänzt
- **CORS:** `http://192.168.2.242` zu erlaubten Origins hinzugefügt

## Ressourcen

- [[../04 Ressourcen/Git|Git]] — Aliasse, nützliche Befehle

## Claude-Verhaltensregeln

- **Vault first:** Bei jeder kontextbezogenen Frage zuerst `core/` lesen
- **Merke dir:** Löst immer internes Claude-Memory + Vault-Notiz aus
- **Standard-Ablage** für Entwicklungsstand: diese Datei
