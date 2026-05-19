---
title:
  - SchulkonzeptHems
tags:
  - projekt
  - aktiv
  - vue
status: in-progress
stand:
  - 2026-05-19
deployed: true
url: https://schutz.softexceptions.com
---

# Schutzkonzept HEMS — Landing Page

> [!info] Ziel
> Eine responsive, barrierefreie Web-Seite, die das gesetzlich vorgeschriebene Schutzkonzept der Heinrich-Emanuel-Merck-Schule Darmstadt als "lebendes Dokument" abbildet. Lehrkräfte finden klare Navigation, Handlungsanleitungen und Kontakte — auf Mobilgeräten genauso wie am Desktop.

Landing Page für das **Schutzkonzept gegen Gewalt und sexuellen Missbrauch** der HEMS Darmstadt. Das Konzept ist gesetzlich vorgeschrieben (Abgabe bis Sommer 2026) und soll Lehrkräften Orientierung und konkrete Handlungsmöglichkeiten geben. Inhalte basieren auf `Schutzkonzept_HEMS_03-05-26-v2.pdf`.

## Farbpalette (aus HEMS_Bild.webp)

| Token | Hex | Bedeutung |
|---|---|---|
| `hems-navy` | `#1a1d21` | Anthrazit-Schwarz (Metallfassade) |
| `hems-blue` | `#2d6fa8` | Stahlblau — Primär-Akzent |
| `hems-sky` | `#6aadc8` | Hellblau (Himmel im Glas) |
| `hems-steel` | `#4a6070` | Stahlgrau — Sekundär-Akzent |
| `hems-glass` | `#8da8b8` | Glasgrau (Spiegelton) |
| `hems-bg` | `#f0f3f5` | Kühles Silberweiß |
| `hems-muted` | `#68808c` | Kühles Grau |

## Stack

| Schicht | Technologie |
|---|---|
| Framework | Vue 3 + TypeScript (strict) + Vite |
| Styling | Tailwind CSS 4 + HEMS-Tokens + `@tailwindcss/typography` |
| Routing | Vue Router 4 (inkl. 404 Catch-all) |
| Markdown | `import.meta.glob` + `marked` + `js-yaml` (custom frontmatter-Parser) |
| Tests | Vitest + Vue Test Utils + happy-dom |
| Fonts | Inter — **self-hosted** (`public/fonts/`) — DSGVO-konform |
| Deployment | `git push` lokal → `git pull && npm run build` auf LXC `192.168.2.79` |

## Architektur

SOLID-Schichten gemäß [[vue-solid]]-Skill:

```
src/
  services/interfaces/    → IKapitelService (async, Promise-basiert)
  services/               → StaticKapitelService (Phase 1) / ApiKapitelService (Phase 2)
  composables/            → useKapitel() — isLoading, error, try/catch/finally
  components/             → Thin Components
  views/                  → Routen-Views mit Lade/Fehler/Inhalt-Zuständen
  content/                → Markdown-Dateien mit Frontmatter
  router/                 → Vue Router + 404 Catch-all
```

> [!important] Interface ist async von Anfang an
> `IKapitelService.getAll()` gibt `Promise<KapitelMeta[]>` zurück — auch in Phase 1.
> Phase 2: nur `new ApiKapitelService()` statt `new StaticKapitelService()` in `useKapitel.ts`.

## Design

| Token | Wert |
|---|---|
| Primär | `#2d6fa8` (hems-blue) |
| Hintergrund | `#f0f3f5` (hems-bg) |
| Text | `#1a1d21` (hems-navy) |

- **Stil:** Minimalism — ruhig, klar, professionell (sensibles Thema)
- **Schriften:** Inter — self-hosted (`public/fonts/inter.woff2`), keine Google-Anfrage

## Besondere Komponenten

- `InterventionsAkkordeon.vue` — Kapitel 1.4 (Fall A–D) als **2×2-Karten-Grid** (mobil einspaltig), Klick öffnet Detail-Panel; farblich: sky/orange/blue/yellow (Dark Mode: neutral `gray-800`); Emoji: 🏠👥👨‍🏫🪪; **Podcast-Bereich steht über den Schritten**; **Triggerwarnung:** zweistufig — kompakte Zeile → ausgeklappt (mit X zum Zuklappen) → Bestätigung; `triggerAusgeklappt` + `triggerBestaetigt` + `triggerEinklappen()` (alle als `Set<number>`-Refs); nativer `<audio>`-Player; **Transkript:** Vue-State `transkriptOffen`-Set (kein `<details>`), Chevron-Rotation per `:class`-Binding; Play-Hinweis Fade-out via `gespielt`-Set
- `KontaktKarte.vue` — für Kapitel 4+5 mit Name, Angebote, E-Mail, Tel, Website
- `LadeIndikator.vue` + `FehlerMeldung.vue` — UI-Zustände (Retry-Button)
- ~~`src/utils/ikonen.ts`~~ — nur noch in `InterventionsAkkordeon` nicht mehr nötig (Emojis ersetzt alle SVG-Icons in den Akkordeon-Zeilen)

## Frontmatter-Felder (vollständig)

```yaml
nr: 1
titel: Vorgehen bei Verdacht
slug: verdacht
beschreibung: Kurzsatz für Übersichtsseiten-Karte
emoji: "🔍"                            # Karten-Emoji auf HomeView (01–05)
abschnittsEmoji: ["🔍", "💭", "🗣️", "🗺️", "🛡️"]  # Index = H2-Reihenfolge
faelle:          # nur Kapitel 1
  - titel: "Fall A: …"
    schritte: […]
    podcast:                           # optional — nur wenn Audio vorhanden
      datei: /audio/fall-a.m4a
      triggerwarnung: "…"
      transkriptDatei: fall-a          # → src/content/transcripts/fall-a.md
kontakte:        # Kapitel 4+5
  - name: …
    email: …
```

## H2-Akkordeon-Logik (`KapitelView.vue`)

- HTML wird per Regex in `intro` (vor erstem H2) + `abschnitte[]` aufgeteilt
- **Exklusives Akkordeon:** `offen = ref<string | null>(null)` — nur ein Abschnitt gleichzeitig offen
- Abschnitt mit `<!-- faelle -->` Marker wird nie als Akkordeon gerendert → immer sichtbar mit eingebetteter `InterventionsAkkordeon`-Komponente
- Emoji-Rendering: `abschnittsEmoji[i]` direkt als `<span>` (kein SVG mehr), 60% Deckkraft geschlossen / 100% offen
- **Wichtig:** `parseKapitel()` in `StaticKapitelService.ts` muss `abschnittsEmoji` zurückgeben (nicht nur `parseMeta()`)

## TDD

Nur für die **Service-Schicht** — Komponenten werden visuell geprüft.

- `StaticKapitelService.spec.ts` — Parsing, Sortierung, Fehlerfälle
- `useKapitel.spec.ts` — reaktive Daten, isLoading/error-Verhalten
- Phase 2: dieselben Tests laufen gegen `ApiKapitelService`

## Phasen

### Phase 1 — Content (aktuell)

- [x] Vite-Projekt initialisieren + Abhängigkeiten installieren
- [x] Tailwind + HEMS-Tokens konfigurieren
- [x] Inter Fonts self-hosten (DSGVO)
- [x] Favicon (SVG, HEMS-Orange + Fingerbalken) → `public/favicon.svg`
- [x] Typen + IKapitelService Interface
- [x] TDD: Tests schreiben, dann StaticKapitelService implementieren
- [x] useKapitel Composable (isLoading, error, try/catch/finally)
- [x] Vue Router + 404 Route
- [x] Layout + AppNavigation (responsiv, Mobile-first)
- [x] HomeView + KapitelView (alle 3 Zustände: Laden/Fehler/Inhalt)
- [x] KontaktKarte + InterventionsAkkordeon
- [x] PDF-Inhalt → 5 Markdown-Dateien mit Frontmatter
- [x] vue-solid-Review (alle .vue-Dateien geprüft)
- [x] ui-ux-pro-max Review (Kontrast, Barrierefreiheit) — WCAG-AA-Kontrast-Fixes angewendet
- [x] Dark Mode — `useDarkMode.ts` Singleton, localStorage, Toggle ☀️/🌙 als Emoji, alle Komponenten mit `dark:`-Klassen
- [x] Hintergrundbild — `hems-bild3.jpg` als globaler App-Hintergrund mit dynamischem Overlay
- [x] Build + nginx-Snippet dokumentieren

### Phase 2a — Audio (teilweise implementiert)

> [!success] Deployment abgeschlossen
> Seite live unter **https://schutz.softexceptions.com** (LXC `192.168.2.79`, Port 8080, Nginx Proxy Manager `192.168.2.78`)

- [x] Audio-Dateien komprimiert: `Fall_A.m4a` 38 MB → **15 MB**, `Fall_B.m4a` 31 MB → **12 MB** (96 kbps AAC via ffmpeg, Originale erhalten)
- [x] `public/audio/fall-a.m4a`, `public/audio/fall-b.m4a`
- [x] Typen: `PodcastInfo` + `InterventionsFall.podcast?`
- [x] Transkripte: `src/content/transcripts/fall-a.md`, `fall-b.md` (editierbar ohne Code)
- [x] Frontmatter Fall A + B: `podcast`-Feld mit Datei, Triggerwarnung, Transkript-Schlüssel
- [x] `InterventionsAkkordeon.vue`: 🎧 Badge, Triggerwarnung, `<audio controls preload="metadata">`, Transkript (`<details>`), Play-Hinweis mit Fade-out
- [x] Fall C + D Audio: komprimiert (je 48 MB → **18 MB**), Transkripte aus ODT-Dokumenten, Frontmatter ergänzt
- [x] Nginx-Konfiguration für Audio-Streaming auf Proxmox LXC

> [!info] Nginx für Audio-Streaming (Proxmox LXC)
> ```nginx
> location /audio/ {
>     add_header Accept-Ranges bytes always;
>     gzip off;
>     add_header Cache-Control "public, max-age=2592000, immutable" always;
>     sendfile on;
>     tcp_nopush on;
> }
> ```

### Phase 2b — REST-API + Audio (geplant, noch nicht bauen)

**REST-API (FastAPI + SQLite):**

| Endpunkt | Beschreibung |
|---|---|
| `GET /api/kapitel` | Alle Kapitel |
| `GET /api/kapitel/{slug}` | Einzelnes Kapitel |
| `PUT /api/kapitel/{slug}` 🔒 | Admin: Text bearbeiten |
| `GET /api/audio` | Alle Audio-Szenarien |
| `GET /api/audio/{id}` | Audio + Transkript |
| `POST /api/audio` 🔒 | Admin: Audio hochladen |

**Audio-Formate** (je mit Triggerwarnung + Transkript + nativem Player):
- **Format A:** Gespielte Szene (3–5 Min.) + Einordnung
- **Format B:** Fiktiver Erfahrungsbericht (2–3 Min.)
- **Format C:** Fiktives Beratungsgespräch (5 Min.)

> [!warning] Kein externes Audio-Hosting
> Player direkt auf der Seite — kein Spotify, kein Apple Podcasts.

## Agent-Team

| Agent / Skill | Wann einsetzen |
|---|---|
| `vue-solid` | Nach jeder `.vue`-Datei automatisch |
| `frontend-design` | Beim Erstellen von Komponenten |
| `ui-ux-pro-max` | Einmal zu Beginn (Designsystem) + Review am Ende |
| `context7` | Bei jedem Library-API-Aufruf automatisch |

## Start-Befehl

```bash
cd /home/norbert/Code/Schutzkonzept_HEMS
npm run dev     # → http://localhost:5173 (lokal) + http://192.168.2.45:5173 (Netzwerk/Handy)
                # Port kann nach Neustart wechseln → mit `ss -tlnp | grep 517` prüfen
npm run test    # → Vitest
npm run build   # → dist/
```

> [!tip] Netzwerk-Zugriff
> `vite.config.ts` hat dauerhaft `server: { host: true }` — kein `--host`-Flag nötig. Handy und PC müssen im selben WLAN sein.

## Deployment

### Lokal → GitHub → LXC (Standard-Workflow)

**1. Lokal committen und pushen:**
```bash
git add .
git commit -m "feat: Beschreibung der Änderung"
git push -u origin main
```

**2. Auf dem LXC deployen** (SSH zu `root@192.168.2.79`):
```bash
cd /var/www/Schutzkonzept_HEMS
git pull
npm run build
```

Das war's — nginx serviert automatisch das neue `dist/`.

### 3. Nginx auf dem Proxmox LXC

**Installation (Debian/Ubuntu LXC):**
```bash
apt update && apt install nginx -y
```

**Konfigurationsdatei** `/etc/nginx/sites-available/schutzkonzept`:

```nginx
server {
    listen 80;
    server_name _;                        # oder konkrete IP / Domain

    root /var/www/schutzkonzept;
    index index.html;

    # Gzip für Text-Assets (JS, CSS, HTML)
    gzip on;
    gzip_types text/html text/css application/javascript application/json;
    gzip_min_length 1024;

    # SPA-Routing — alle Pfade auf index.html
    location / {
        try_files $uri /index.html;
    }

    # Audio-Streaming — kritisch für parallele Nutzer
    location /audio/ {
        gzip off;                         # bereits komprimiert — kein gzip

        add_header Accept-Ranges bytes always;
        add_header Cache-Control "public, max-age=2592000, immutable" always;

        sendfile on;                      # Zero-Copy: Kernel streamt direkt
        tcp_nopush on;                    # TCP-Pakete bündeln

        types {
            audio/mp4 m4a;               # MIME-Typ für .m4a explizit setzen
        }
    }

    # Statische Assets — 1 Jahr Cache (Vite erzeugt Content-Hash im Dateinamen)
    location ~* \.(js|css|woff2|png|jpg|webp|svg|ico)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }
}
```

**Aktivieren und starten:**
```bash
ln -s /etc/nginx/sites-available/schutzkonzept /etc/nginx/sites-enabled/
nginx -t                                  # Konfiguration testen
systemctl reload nginx
```

> [!info] Warum `sendfile on` so wichtig ist
> Ohne `sendfile` liest nginx die Audiodatei in den Userspace und schreibt sie in den Socket — zwei Kopien. Mit `sendfile` übergibt der Kernel die Datei direkt an den Socket (Zero-Copy). Bei mehreren gleichzeitigen Nutzern macht das den entscheidenden Unterschied.

> [!warning] Dev-Server vs. Produktion
> Der Vite-Dev-Server kann bei laufendem Audio-Stream andere Verbindungen (z. B. Smartphone) blockieren. Das ist ein bekanntes Node.js-Dev-Server-Limit und **tritt in Produktion mit nginx nicht auf**.

## Bekannte Stolperfallen

> [!warning] Sensibles Thema
> Keine Bilder von Betroffenen, keine reißerischen Formulierungen. Ton: sachlich, unterstützend, klar.

> [!tip] Mobile First
> Lehrkräfte nutzen primär Mobilgeräte — immer zuerst auf kleinen Viewports testen.

> [!warning] DSGVO
> Keine externen Ressourcen ohne Einwilligung — Fonts und Assets self-hosted.

## Changelog

### 2026-05-19 (Session 8)

- **Fall C + D Audio eingebunden:**
  - Originale (je 48 MB) komprimiert auf **18 MB** (96 kbps AAC via ffmpeg)
  - `public/audio/fall-c.m4a`, `public/audio/fall-d.m4a`
  - Transkripte aus `Fall_C.odt` und `Fall_D.odt` extrahiert → `src/content/transcripts/fall-c.md`, `fall-d.md` (Sprecher-Format → **Moderator**/**Expertin** vereinheitlicht)
  - Frontmatter `01-verdacht.md`: `podcast`-Feld (Datei, Triggerwarnung, Transkript-Schlüssel) für Fall C und D ergänzt

- **Quiz-Karte auf Startseite** (`HomeView.vue`): 6. Karte im Grid, kein Kapitel-Nr., stattdessen kleines `Quiz`-Badge (gleiche Typografie wie „Schnellzugriff"), Emoji 🧠, Text „Wissenscheck", CTA „Quiz starten", Link `/quiz`

- **Logo-Layout** (`AppNavigation.vue`): „Schutzkonzept" rechts vom Logo statt darunter — `flex items-center gap-3`, Logo mit `shrink-0`, Text mit `whitespace-nowrap`; gilt für Desktop-Sidebar und Mobile-Header

- **Bezeichnung vereinheitlicht**: „Wissen testen" → **„Wissenscheck"** in Desktop-Sidebar und Mobile-Menü

### 2026-05-19 (Session 7)

- **HEMS-Logo komplett überarbeitet:**
  - EPS-Originaldatei → SVG via Ghostscript + `pdftocairo` (Vektorgrafik, kein weißer Hintergrund-Hack mehr)
  - HEMS-Schriftfarbe: Rot → Orange (`#F5921E`)
  - "Heinrich-Emanuel-Merck-Schule" als `<text>`-Element ergänzt (war in EPS nicht vorhanden)
  - Beide Texte (Schulname + Darmstadt) bündig am gleichen linken Rand ausgerichtet
  - Schulname per SVG `<g transform="scale(0.655, 1)">` auf HEMS-Breite gestreckt (rsvg unterstützt `textLength` nicht)
  - Weißer Hintergrund als `<rect>` direkt im SVG (15pt Padding rundum) — ersetzt `dark:bg-white/90`-Hack
  - Keine `rx/ry`-Rundung im SVG (transparente Ecken → graue Artefakte auf Sidebar-Hintergrund); Rundung via CSS `rounded`-Klasse
  - `AppNavigation.vue`: Logo-Klasse vereinfacht zu `h-8 w-auto rounded bg-white` (beide Modi)

- **Mobile Header repariert:** `<HemsLogo>`-Komponente war nicht mehr importiert → Logo + "Schutzkonzept"-Text unsichtbar; ersetzt durch `<img src="/hems-logo.svg">` identisch zur Desktop-Sidebar

- **Podcast UX:**
  - Podcast-Bereich in `InterventionsAkkordeon.vue` über die Schritte-Liste verschoben
  - Triggerwarnung zweistufig: kompakte Zeile → ausgeklappter Volltext → Bestätigung
  - Triggerwarnung wieder zuklappbar: X-Button + `triggerEinklappen()`-Funktion

- **Dark Mode — Akkordeon komplett überarbeitet:**
  - Fall-Karten und Detail-Panels: neutrale Grau-Töne (`dark:bg-gray-800`, `dark:border-gray-700`) statt getönter Hintergründe (kein Orange/Gelb/Blau mehr im ganzen Fenster)
  - Karten-Titel: `dark:text-gray-100`, Vorschautext: `dark:text-gray-400`, Podcast-Badge: `dark:text-hems-sky`
  - Detail-Panel-Titel: `dark:text-gray-100`, Schritte: `dark:text-gray-300`, Transkript: `dark:text-gray-200`
  - Triggerwarnung: `dark:bg-gray-700 dark:border-gray-600`, Titeltext: `dark:text-amber-300`, Fließtext: `dark:text-gray-200`, Bestätigen-Button: `dark:bg-gray-600 dark:text-gray-100`

### 2026-05-19 (Session 6)

- **Deployment:** Produktiv live unter `https://schutz.softexceptions.com`
- **LXC:** Proxmox LXC `SchutzHEMS` — IP `192.168.2.79`, Debian, Node.js 22, nginx auf Port 8080
- **Repo auf LXC:** `/var/www/Schutzkonzept_HEMS/` — `git clone` + `npm ci` + `npm run build` → `dist/`
- **Nginx-Config:** `/etc/nginx/sites-available/schutzkonzept` — SPA-Routing, Audio-Streaming (`sendfile on`, `gzip off`), Asset-Cache 1y
- **Reverse Proxy:** Nginx Proxy Manager LXC `192.168.2.78` → leitet `https://schutz.softexceptions.com` auf `192.168.2.79:8080`
- **Updates künftig:** `git pull && npm run build` auf dem LXC

### 2026-05-19 (Session 5)

- **Quiz-Feature implementiert:** 50 Fragen in `src/content/quiz.json` (15 MC, 15 Szenario, 12 Richtig/Falsch, 8 Zuordnung); SOLID-Architektur mit `IQuizService`, `StaticQuizService`, `useQuiz`-Composable; Route `/quiz`; Komponenten `QuizFrage.vue` + `QuizAuswertung.vue` + `QuizView.vue`; 🧠-Link in Desktop-Sidebar und Mobile-Menü
- **Quiz-Ablauf:** Start → zufällig 3–7 Fragen → Auswertung mit SVG-Kreis-Score + Detailansicht (Erklärungen); Abbrechen-Button während Fragen-Phase
- **Quiz Shuffle-Bug behoben:** `richtig-falsch`-Fragen waren vom Option-Mischen ausgenommen → bei `richtig: 0` stand "Richtig" immer an erster Stelle und war korrekt → erkennbares Muster; Fix: Ausnahme entfernt, alle Typen gleichbehandelt
- **Phase 2b-Vorbereitung:** `IQuizService`-Interface ermöglicht spätere Migration auf `GET /api/quiz` ohne Composable-Änderung

### 2026-05-19 (Session 4)

- **Fall-Karten Labels:** Beschriftung von "A/B/C/D" auf "Fall A/B/C/D" geändert (`text-xl` statt `text-3xl`)
- **Dark Mode Farben:** `--color-gray-900: #1c2535`, `--color-gray-800: #2d3f56` in `style.css` (`@theme`)
- **Emoji auf Kapitel-Karten:** `emoji`-Frontmatter-Feld (🔍🛡️📋💬🤝), Typ erweitert, Service liest es aus, `KapitelCard.vue` zeigt es oben rechts (opacity 60% → 100% bei Hover)
- **Sidebar komplett überarbeitet:**
  - Emojis + Nummern (`🔍 01 Vorgehen bei Verdacht`) in allen Nav-Links
  - Aktiv-Indikator: `border-l-2 border-hems-blue bg-white` (sichtbar gegen `bg-hems-bg`)
  - "KAPITEL"-Label über der Liste
  - Mausrad-Navigation zwischen Kapiteln (`@wheel.prevent`, 600 ms Cooldown)
  - Hellmodus: `bg-hems-bg` Sidebar, `bg-white` Logo-Bereich, `h-1 bg-hems-blue` Top-Akzentstreifen, `border-hems-glass/70` Trennlinie
  - Hover: `bg-white/70`, Nummern: `text-hems-steel/50`
- **Podcast-Feature (Phase 2a):**
  - Audio komprimiert: Fall A 38 MB → 15 MB, Fall B 31 MB → 12 MB (96 kbps AAC)
  - `public/audio/fall-a.m4a`, `public/audio/fall-b.m4a`
  - Transkripte aus ODT extrahiert → `src/content/transcripts/fall-a.md`, `fall-b.md`
  - Neue Typen: `PodcastInfo`, `InterventionsFall.podcast?`
  - `InterventionsAkkordeon.vue`: 🎧 Badge auf Karten, Triggerwarnung (Set-basiert, muss aktiv bestätigt werden), `<audio controls preload="metadata">`, Transkript via `import.meta.glob` + `marked`, Play-Hinweis mit 300ms Fade-out (`gespielt`-Set)
- **Transkript-Chevron:** `group-open:` existiert in Tailwind 4.3 nicht → Vue-State-Lösung mit `transkriptOffen`-Set + `transkriptUmschalten()`; Chevron rotiert 180° via `:class`-Binding; `summary { list-style: none }` in `style.css`
- **Transkript-Textfarbe:** `prose`-Plugin überschreibt Farbklassen mit höherer Spezifizität → `prose` entfernt, direkt `text-hems-navy` (panels sind immer hell, kein Dark-Mode-Override nötig)
- **Transkript-Hover:** `dark:hover:text-white` war auf hellem Panel-Hintergrund unsichtbar → `hover:text-hems-blue dark:hover:text-hems-sky`
- **Nginx-Deployment** vollständig dokumentiert im Vault (SPA-Routing, Audio-Streaming, Asset-Cache, `sendfile on`)
- **Dark Mode Lesbarkeit** — mehrere zu blasse Elemente nachgebessert:
  - Nav-Link-Labels: `gray-400` → `gray-300`
  - Nummern (01–05): `gray-600` → `gray-400`
  - "KAPITEL"-Label: `gray-500` → `gray-400`
  - "Übersicht"-Link (`KapitelView.vue`): `text-hems-muted` + `dark:text-gray-400` + `dark:hover:text-hems-sky`
  - "Kapitel XX"-Label (`KapitelView.vue`): `gray-500` → `gray-400`

### 2026-05-18 (Session 3)

- **Dark Mode:** `useDarkMode.ts` als Singleton-Composable (module-level `ref`, localStorage + System-Präferenz); `@variant dark (&:is(.dark *))` in `style.css`; Toggle-Button in Sidebar-Footer (Desktop) und Mobile Header — **Emojis** `☀️` / `🌙` (kein SVG); `hintergrundStyle` in `App.vue` wechselt Hintergrund-Overlay zwischen Hell `rgba(240,243,245,0.88)` und Dunkel `rgba(10,12,15,0.90)`; alle Komponenten mit `dark:`-Klassen versehen (gray-900 Karten, gray-700 Rahmen, weißer Text)
- **Hintergrundbild:** `hems-bild3.jpg` als globales Hintergrundbild in `App.vue` (blasser Overlay darüber)
- **HEMS-Favicon:** `public/favicon.svg` — HEMS-Orange (#F5921E) Hintergrund + 4 weiße Fingerbalken

### 2026-05-18 (Session 2)

- **Textoverflow-Fix Fall-Karten:** `min-w-0 break-words` auf `<span>` in den `<li>`-Elementen des Detail-Panels (Flex-Container ignorieren ohne `min-w-0` die Elternbreite)
- **Sticky Header Mobile:** `sticky top-0 z-50` auf `<header>` + `h-svh` statt `min-h-svh` auf äußerem Container in `App.vue` → `<main>` scrollt intern, Header bleibt fest
- **Schnellzugriff-Karte:** Neue prominente Karte auf der Startseite (`HomeView.vue`) mit direktem Link zum Interventionsplan (Kapitel 1); Emoji 🗺️, blauer Rahmen, Pfeil-Icon
- **Emojis statt SVG-Icons:**
  - Akkordeon-Zeilen: `abschnittsEmoji`-Frontmatter (ersetzt `abschnittsIcons`); alle 4 Kapitel mit Emojis befüllt (01–04); gerendert als `<span>` mit `aria-hidden`
  - Fall A–D Karten: `emoji`-Feld in `farben`-Array (🏠👥👨‍🏫🪪) statt SVG-Pfad; IKONEN-Import aus `InterventionsAkkordeon` entfernt
  - Schnellzugriff: 🗺️ Emoji statt SVG-Warndreieck
- **Fall D Farbe:** Slate → Yellow (`border-yellow-200 bg-yellow-50` etc.)
- **HEMS-Logo:** Aus `Schutzkonzept_HEMS_03-05-26-v2.pdf` (Seite 1) per `pdfimages` extrahiert → `public/hems-logo.png`; in Desktop-Sidebar und Mobile-Header eingebaut (je `h-8 w-auto`, inline mit "Schutzkonzept"-Text)
- **Silbentrennung:** `hyphens: auto` + `-webkit-hyphens: auto` in `body`; `.hyphens-auto` CSS-Override in `style.css` ergänzt `-webkit-hyphens: auto` (Tailwind generiert das nicht); `lang="de"` explizit auf `v-html`-Containern und `<ol>` in Akkordeon; `break-words` + `hyphens-auto` kombiniert (break-words als Overflow-Fallback, hyphens für Trennstrich)
- **Scroll-to-Top:** `mainRef` als Ref in `App.vue`; Route-Watcher scrollt `<main>` bei Routenwechsel nach oben; Logo-Klick ruft `scrollNachOben()` auf (auch wenn bereits auf Startseite)

### 2026-05-18 (Session 1)

- **H2-Akkordeon:** Alle Kapitel-Abschnitte werden als exklusives Akkordeon dargestellt (nur ein Abschnitt gleichzeitig offen); `offen = ref<string | null>(null)`
- **Icons — Akkordeon-Zeilen:** `src/utils/ikonen.ts` als zentrales Heroicons-v2-Outline-Registry; `abschnittsIcons`-Frontmatter-Feld pro Kapitel (alle 5 Kapitel befüllt)
- **Icons — Fall-Karten:** Fall A–D haben Mini-Icons (home/users/academic-cap/identification) oben rechts in den Karten
- **Bug behoben:** `parseKapitel()` in `StaticKapitelService.ts` gab `abschnittsIcons` nicht zurück — Icons fehlten in der Detailansicht
- **Kurzbeschreibungen:** `beschreibung`-Frontmatter-Feld in allen 5 Kapiteln → erscheint auf der Übersichtsseite unter dem Kapitel-Titel
- **Netzwerkzugriff:** `vite.config.ts` dauerhaft mit `server: { host: true }` — Handy-Zugriff via WLAN ohne `--host`-Flag

### 2026-05-15

- **vue-solid-Review:** Alle 10 `.vue`-Dateien geprüft — SOLID-konform
- **Bug behoben:** `RouterView :key="$route.fullPath"` — verhindert Wiederverwendung der Komponente beim Kapitelwechsel (slug-Navigation fehlerhaft ohne Key)
- **Barrierefreiheit:** `aria-expanded` am Akkordeon ergänzt; `text-hems-muted` → `text-hems-steel` für kleine Labels (WCAG AA: 5.69:1 statt 3.72:1)
- **Stack-Korrektur:** `gray-matter` → `js-yaml` (gray-matter nutzt Node.js Buffer, nicht browser-kompatibel)

### 2026-05-14

- **Planung abgeschlossen:** Stack, Architektur, TDD-Strategie, Error-Handling, REST-API Phase 2 vollständig geplant
- **Entscheidungen:** Vue 3 + TS + Vite, kein CMS, Inter self-hosted, Interventionsplan als Akkordeon
- **IKapitelService async** von Anfang an — sichert Phase-2-Migration ohne Interface-Änderung
- **Phase 1 vollständig implementiert:** 13/13 Tests grün, alle 5 Kapitel digitalisiert, Dev-Server läuft
