---
tags: [bereich, unterricht, programmiersprachen, architektur]
date: 2026-06-22
---

# Sprachwahl begründen — Fallbeispiel StreamPrometh

Lehreinheit zur Frage: **Wie begründet man die Wahl einer Programmiersprache für ein konkretes Projekt?** Reales Fallbeispiel: das Projekt [[StreamPrometh]] (Rust-CLI/GUI zum Bildschirmspiegeln auf Promethean-Schulboards unter Linux).

## Lernziel

Die Schüler:innen sollen **Anforderungen vor Technologie** setzen: nicht „welche Sprache ist die beste?", sondern „welche Sprache deckt **dieses** Anforderungsbündel am vollständigsten ab?". Sie lernen, eine Entscheidung anhand expliziter Achsen abzuwägen — und sie ehrlich gegen die Kosten zu rechnen.

## Didaktischer Kern (eine Botschaft)

> Eine Sprachwahl ist eine **Abwägung über mehrere Achsen gleichzeitig**, kein Sieg auf einer einzelnen Achse. Die „beste" Sprache pro Einzelkriterium gewinnt selten — die beste **Kombinationsabdeckung** gewinnt.

## Schritt 1 — Anforderungsprofil scharfstellen (vor jedem Sprachvergleich!)

Am Beispiel StreamPrometh:

1. **Nativer Linux-Desktop-Stack** — Screen-Capture (XDG-Portal + PipeWire), H.264 (GStreamer), GUI (GTK4), D-Bus. → eine **C-Welt** (GNOME/freedesktop).
2. **WebRTC** — SDP/ICE/DTLS/RTP, latenzsensitiv.
3. **Verteilung auf Schul-PCs** — USB-Stick rein, **kein Runtime** soll dort nötig sein.
4. **Zuverlässigkeit** — läuft vor einer Klasse; Absturz/Hänger ist teuer.

**Lehrpunkt:** Erst dieses Profil macht den Vergleich überhaupt aussagekräftig. Ohne Profil ist „Rust vs. Python" Geschmackssache.

## Schritt 2 — Vergleichstabelle gegen realistische Alternativen

| Achse | **Rust** | C/C++ | Python | Go | TS/Electron |
|---|---|---|---|---|---|
| Single-Binary auf Schul-PC | ✅ | ✅ unsicher | ❌ Interpreter+venv | ✅ | ❌ ~150 MB |
| Native Bindings (GStreamer/GTK/PipeWire/D-Bus) | ✅ erstklassig | ✅ (ist C) | ✅ PyGObject | ⚠️ cgo-Reibung | ❌ schwach (Wayland) |
| WebRTC-Reife | ⚠️ webrtc-rs | ✅ libwebrtc | ⚠️ aiortc | ✅✅ pion | ✅✅ libwebrtc |
| Echtzeit/Latenz (kein GC) | ✅ | ✅ | ❌ GIL | ⚠️ GC-Pausen | ⚠️ GC+IPC |
| Speichersicherheit | ✅ erzwungen | ❌ manuell | ✅ (GC) | ✅ (GC) | ✅ (GC) |
| Lerntempo / Komplexität | ❌ steil | ❌ steil+gefährlich | ✅ schnell | ✅ einfach | ✅ schnell |

## Schritt 3 — Die Begründung erzählen (nicht nur tabellieren)

- **Gegen C/C++:** gleicher nativer Zugriff + Single-Binary, aber **Sicherheit ohne GC**. Konkret: Der Datei-Deskriptor der Bildschirmaufnahme muss über zwei Encoder-Versuche (VA-API → Software-Fallback) am Leben bleiben — in Rust **erzwingt der Compiler** das, in C wäre es ein sporadischer Crash.
- **Gegen Python:** PyGObject ist gut, aber Verteilung (Interpreter+Deps), GIL/GC (Echtzeit) und fehlende statische Typsicherheit (Fehler erst vor der Klasse) sprechen dagegen.
- **Gegen Go (stärkster Gegner!):** Go's `pion` ist die **reifere** WebRTC-Implementierung — `webrtc-rs` ist sogar ein Port davon. Wäre WebRTC die *einzige* Achse, gewänne Go. Aber bei GTK/Portal/PipeWire (cgo-Reibung, unvollständige Bindings) verliert Go. → **Rust ist der bessere Allrounder.**
- **Gegen TS/Electron:** verlockend, weil Prometheans eigene Lösung eine Chrome-Extension ist (reifstes WebRTC). Aber 150-MB-Runtime und — entscheidend — die Wayland-Screen-Capture aus Electron ist genau das, was unzuverlässig versagt.

## Schritt 4 — Ehrlich bleiben: Wo die Wahl Geld kostet

Der wertvollste Lehrteil — **keine Technologie ist gratis**:

- **Lernkurve real:** Rusts Lifetime-/Trait-Gymnastik (z. B. `'static`-Futures für WebRTC-Callbacks) gibt es in Go/Python nicht.
- **Sicherheit ≠ Korrektheit:** Die echten Bugs im Projekt waren **keine Speicherfehler**, sondern Logik-/Integrationsfehler (D-Bus-Call ohne Timeout → Hänger; gepufferter Logger → verschwundene Logs). Rust verhindert eine *Klasse* von Fehlern, nicht *alle*.
- **Unreife Abhängigkeit:** `webrtc-rs` ist weniger erprobt → eigener Board-Simulator zum Testen nötig.

## Fazit-Satz für die Tafel

> Rust gewinnt hier nicht, weil es auf einer Achse am besten ist, sondern weil es **alle vier Anforderungen gleichzeitig** am vollständigsten abdeckt: C-nahe Kontrolle + Single-Binary + Echtzeit + erzwungene Sicherheit.

## Schüler-Übung

1. Eigenes Projekt nehmen, **Anforderungsprofil** (3–5 Achsen) aufschreiben — *bevor* an Sprachen gedacht wird.
2. Vergleichstabelle gegen 3 realistische Kandidaten ausfüllen (✅/⚠️/❌ + ein Stichwort).
3. **Den stärksten Gegner benennen** und erklären, warum man sich *trotzdem* anders entschieden hat.
4. Eine ehrliche „Kosten"-Zeile: Was kostet meine Wahl (Lernkurve, Reife, Tooling)?

**Bewertungsfokus:** Nicht die gewählte Sprache, sondern die **Qualität der Begründung** und die Ehrlichkeit der Gegenrechnung.

---

Verwandt: [[StreamPrometh]] · [[REST-API-Architektur]]
