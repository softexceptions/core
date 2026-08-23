---
tags: [ressource, claude, excalidraw, diagramme]
date: 2026-05-10
---

# Excalidraw-Workflow mit Claude

## Kernregel: Iterativ, nicht monolithisch

Claude soll Excalidraw-Diagramme **abschnittsweise** aufbauen — nie das gesamte JSON in einem Durchgang generieren. Das führt zu weniger Fehlern und schnelleren Ergebnissen.

## Vorgehen

1. **Grundstruktur** (Titel, Column-Header, Trennlinien) → rendern → prüfen
2. **Eine Sektion / Phase nach der anderen** → rendern → prüfen
3. **Fix-Iteration** nur für konkrete sichtbare Probleme

## Häufige Fehlerquellen

| Problem | Ursache | Fix |
|---|---|---|
| Arrow-Labels abgeschnitten | `width`/`height` zu klein | min. `width: 120`, `height: 25` bei `fontSize 14` |
| Text von Hintergrund verdeckt | Falsche Z-Order im Array | Hintergrund-Rect **vor** den darüber liegenden Elementen |
| PNG zeigt Hieroglyphen | Lokaler Renderer fehlen Fonts | Finalen Export auf excalidraw.com machen |

## Export

- **Bearbeitung / Kontrolle**: `.excalidraw`-Datei direkt in excalidraw.com öffnen
- **Finale PNG/SVG**: In excalidraw.com exportieren (Fonts korrekt eingebettet)
- **Lokaler Renderer** (`render_excalidraw.py`): Nur zur schnellen Struktur-Kontrolle während der Erstellung — Font-Rendering ist kaputt (Hieroglyphen)

## Lokaler Renderer — Setup & Hintergrund

**Warum lokales Bundle statt CDN?**
`render_template.html` importierte Excalidraw ursprünglich von `esm.sh` (CDN). Zwei Probleme:
- `file://`-URLs in Playwright blockieren HTTPS-Imports (Mixed Content)
- `esm.sh` lieferte `@braintree/sanitize-url` mit 404 → Timeout nach 30 s

Fix (2026-05-10): `excalidraw-bundle.js` lokal mit esbuild gebaut, liegt in `references/`. Kein CDN, kein Single Point of Failure, deutlich schneller (2 s statt Timeout).

**Bundle neu bauen** (falls `excalidraw-bundle.js` fehlt oder veraltet):
```
cd /tmp && mkdir excalidraw_render && cd excalidraw_render
npm init -y && npm install @excalidraw/excalidraw
echo 'import { exportToSvg } from "@excalidraw/excalidraw"; window.__excalidrawExportToSvg = exportToSvg; window.__moduleReady = true;' > entry.js
npx esbuild entry.js --bundle --format=iife --platform=browser \
  --define:process.env.NODE_ENV='"production"' \
  --outfile=~/.claude/skills/excalidraw-diagram/references/excalidraw-bundle.js
```
