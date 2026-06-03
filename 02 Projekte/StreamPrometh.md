---
title: StreamPrometh
tags:
  - projekt
  - aktiv
  - rust
  - linux
  - schule
status: in-progress
stand: 2026-06-03
---

# StreamPrometh

> [!important] Nach `/init` in die CLAUDE.md des Projekts eintragen:
> `Projektbeschreibung: core/02 Projekte/StreamPrometh.md`

Rust-App für Linux: Bildschirm per WebRTC auf digitale Schulboards (Promethean, SMART Board, Prowise) spiegeln — ohne GNOME Network Displays, ohne Wi-Fi Direct, ohne Treiberproblem.

> [!info] Ziel
> Lehrkräfte auf Arch/Linux können ihren Laptop-Bildschirm per PIN-Code auf das Schulboard übertragen. Ein Befehl startet die App, PIN eingeben — fertig. Kein Windows, kein Adapter, kein Fummelei.

> [!warning] Warum nicht GNOME Network Displays?
> Verbindungsaufbau (P2P-Handshake + PIN) funktioniert unter Linux. Aber sobald das Promethean-Board das Signal „Sende jetzt" gibt, bricht GNOME ab — schwarzer Bildschirm. Grund: Promethean erwartet den Videostream **zwingend** auf UDP-Portbereich 19000–19100. Linux-Firewall blockiert diese Ports oder GNOME handelt einen anderen Port aus. Dieses Projekt löst das sauber.

## Technischer Ablauf (6 Schritte)

**Schritt 1 — Board meldet sich an**
Die Lehrkraft öffnet die „Screen Share"-App auf dem Promethean-Board. Das Board sendet ein Signal an `share.mypromethean.com`: „Ich bin online, mein Name ist ‚Klassenzimmer 3b', meine lokale IP ist 192.168.1.45."

**Schritt 2 — Cloud generiert PIN**
Der Promethean-Server erzeugt einen zufälligen 6-stelligen Code (z.B. `512 903`) und speichert: `512903 → Klassenzimmer 3b @ 192.168.1.45`.

**Schritt 3 — Board zeigt PIN an**
Das Board empfängt den Code und zeigt ihn groß im Display an.

**Schritt 4 — App sendet PIN**
Der Nutzer tippt den Code in die Rust-App auf dem Laptop. Die App schickt ihn an `share.mypromethean.com`.

**Schritt 5 — Matchmaking**
Der Cloud-Server findet das passende Board in seiner Datenbank und liefert die lokale IP-Adresse zurück.

**Schritt 6 — Stream startet**
Die Rust-App baut eine direkte Verbindung im Schulnetzwerk zur Board-IP auf und beginnt den Videostream — alles über das bestehende WLAN, kein Wi-Fi Direct nötig.

## Das Promethean-Warteraum-Problem

Promethean-Boards nutzen kein klassisches Miracast, sondern ein **„Waiting Room"-System**:

1. **PIN-Eingabe (klappt):** Laptop taucht als Name mit Punkt im Warteraum des Boards auf
2. **Warteraum:** Kein Bild — die Lehrkraft muss aktiv auf den Laptop-Namen tippen und „Teilen" drücken
3. **Freigabe:** Board sendet Signal „Sende jetzt" an den Laptop
4. **Stream:** Laptop muss sofort auf den richtigen Ports antworten

**Kritische Ports:**

| Port | Protokoll | Funktion |
|---|---|---|
| 7236 | TCP | Steuerkanal (Verbindungsaushandlung) |
| 7250 | TCP | Alternative Steuerung (MoI) |
| 19000–19100 | UDP | Videostrom — **zwingend dieser Bereich** |

> [!warning] Der Linux-Absturz-Grund
> GNOME Network Displays handelt den Videostream auf einem anderen Port aus oder die Firewall blockiert UDP 19000–19100. Promethean akzeptiert keinen anderen UDP-Port → schwarzer Bildschirm.

## Lösungsansatz: Miracast over Infrastructure + WebRTC

### Warum kein Wi-Fi Direct?

In Schulen nutzen Promethean-Boards **„Miracast over Infrastructure" (MoI)**: Der Datenverkehr läuft über die vorhandenen WLAN-Router der Schule — kein direktes Funknetzwerk zwischen Geräten. Das ist eine hervorragende Nachricht: Keine Wi-Fi Direct-Programmierung nötig, nur lokales Netzwerk.

### Primärer Ansatz: WebRTC

Promethean nutzt WebRTC für seine eigene Web-Lösung (`share.mypromethean.com`). Dasselbe Protokoll in der Rust-App verwenden:

- **Stabiler als Miracast** auf Linux
- **Kein Port-Problem** — WebRTC handelt Ports dynamisch aus
- **Gleiche API** wie die offizielle Web-Lösung

### Fallback: Miracast over Infrastructure

Falls WebRTC nicht funktioniert (Board unterstützt kein WebRTC direkt):

- TCP 7236/7250 für Steuerkanal öffnen
- H.264-Videostream auf UDP 19000–19100 senden
- Warteraum-Zustand implementieren (App wartet auf „Sende jetzt"-Signal)

## Architektur

```
Nutzer gibt PIN ein: „512 903"
  │
  ▼
Rust-App (StreamPrometh)
  ├── ashpd → XDG Screencast Portal → Wayland/X11 Frame-Capture
  ├── H.264-Encoding (GStreamer oder ffmpeg-sys)
  ├── PIN-Auth → share.mypromethean.com → Board-IP zurück
  ├── Warteraum-State Machine (idle → waiting → streaming)
  └── WebRTC-Stream → Board-IP (lokales Schulnetzwerk)
                └── Fallback: RTSP/RTP → UDP 19000–19100
```

## Stack

| Schicht | Technologie | Crate / Tool |
|---|---|---|
| Bildschirm abgreifen | XDG Desktop Portal (Wayland + X11) | `ashpd` |
| Video-Encoding | H.264 | `gstreamer` |
| WebRTC Peer-to-Peer | SDP/ICE/MediaStream | `webrtc-rs` |
| Signaling WebSocket | pre-signed WSS (kein SigV4 nötig!) | `tokio-tungstenite` |
| Service Discovery | `checkPanel` + `getChannel` | `reqwest` |
| UUID | Client-ID generieren | `uuid` (v4) |
| Fallback-Protokoll | RTSP / RTP | `rtp` + `rtcp` |
| Async Runtime | — | `tokio` |
| CLI / UI | Terminal-UI | `clap` (Args) + `ratatui` (TUI, optional) |

> [!success] Kein AWS SDK nötig!
> `servicediscovery.mypromethean.com/getChannel` liefert eine **fertig signierte** WebSocket-URL. Kein `aws-sdk-kinesisvideo`, kein `aws-sigv4`, kein STS. Nur `reqwest` + `tokio-tungstenite` + `webrtc-rs`.

> [!tip] ashpd ist der richtige Weg auf Arch/Wayland
> `ashpd` spricht XDG Desktop Portals — funktioniert auf GNOME, KDE, Hyprland und jedem anderen Wayland-Compositor. Kein X11-Hack nötig.

## State Machine (Warteraum)

```
idle
  │ PIN eingegeben
  ▼
authenticating → Fehler → idle (Toast)
  │ Board-IP erhalten
  ▼
waiting_for_approval ← Board zeigt Laptop im Warteraum
  │ Board-Signal „Sende jetzt"
  ▼
streaming ← H.264-Frame-Loop läuft
  │ Verbindung unterbrochen / Nutzer stoppt
  ▼
idle
```

> [!warning] Warteraum darf nicht abstürzen
> Zwischen PIN-Eingabe und Board-Freigabe können viele Sekunden vergehen. Die App muss in diesem Zustand stabil warten — kein Timeout, kein Panic.

## API-Analyse: Vollständiges Protokoll (2026-05-26)

> [!success] Chrome Extension v4.3.2 dekompiliert — vollständiges Protokoll dokumentiert
> Promethean Screen Share nutzt **kein direktes AWS SDK**. Der Laptop spricht nur mit `servicediscovery.mypromethean.com`. Kein AWS Credential-Handling nötig — der Server liefert eine pre-signierte WebSocket-URL.

### Vollständiger Protokoll-Flow

```
1. PIN-Eingabe
       │
       ▼
2. GET servicediscovery.mypromethean.com/checkPanel?panelId={PIN}
   Response: { name: "Klassenzimmer 3b", ... }   ← Board-Name für Warteraum-Anzeige
       │ Panel nicht vorhanden → Fehler anzeigen
       ▼
3. clientId = uuid.v4()   ← lokal generieren
       │
       ▼
4. GET servicediscovery.mypromethean.com/getChannel?panelId={PIN}&clientId={uuid}
   Response: {
     stunServer: { Uris: [...], Username: "...", Password: "..." },
     turnServers: [{ Uris: [...], Username: "...", Password: "..." }],
     signedUri: "wss://...kinesisvideo...amazonaws.com/..."   ← AWS KVS pre-signed URL
   }
       │
       ▼
5. WebSocket-Verbindung zu signedUri (tokio-tungstenite)
   → keine SigV4-Signierung nötig, URL ist bereits signiert
       │ WebSocket open
       ▼
6. Screen-Capture starten (ashpd)
   codec = H264 (hardcoded bevorzugt)
   bitrate = 5 Mbit/s
   SDP-Offer erstellen: { offerToReceiveAudio: true, offerToReceiveVideo: true }
       │
       ▼
7. SDP-Offer senden:
   { "action": "SDP_OFFER",
     "messagePayload": base64(JSON.stringify(offer)),
     "recipientClientId": undefined }
       │ Board antwortet mit SDP_ANSWER
       ▼
8. SDP-Answer empfangen:
   { "messageType": "SDP_ANSWER", "messagePayload": base64(...), "senderClientId": null }
   → setRemoteDescription(answer)
       │
       ▼
9. ICE Candidates tauschen:
   Senden: { "action": "ICE_CANDIDATE", "messagePayload": base64(candidate.toJSON()) }
   Empfangen: messageType "ICE_CANDIDATE" → addIceCandidate()
       │
       ▼
10. WebRTC P2P-Verbindung aktiv → Screen-Stream fließt zum Board
```

### API-Endpoints (aus Chrome Extension v4.3.2)

| Endpoint | Methode | Parameter | Beschreibung |
|---|---|---|---|
| `servicediscovery.mypromethean.com/checkPanel` | GET | `panelId={PIN}` | Board-Name abrufen, prüfen ob aktiv |
| `servicediscovery.mypromethean.com/getChannel` | GET | `panelId={PIN}&clientId={uuid}` | Signed URL + STUN/TURN holen |

> [!warning] Nur aus Schul-Netzwerk erreichbar
> `servicediscovery.mypromethean.com` ist hinter einem AWS ELB, der nur aus Schul-IP-Bereichen routet. Aus dem öffentlichen Internet antwortet der ELB immer mit `LB OK`. → Protokoll ist bekannt, aber testen nur im Schulnetz möglich.

### WebSocket-Nachrichten-Format

```jsonc
// Senden (Laptop → Board via KVS)
{
  "action": "SDP_OFFER",       // oder SDP_ANSWER, ICE_CANDIDATE
  "messagePayload": "<base64(JSON.stringify(sdp|iceCandidate))>",
  "recipientClientId": null    // für Master (Board) weglassen
}

// Empfangen (Board → Laptop via KVS)
{
  "messageType": "SDP_ANSWER", // oder ICE_CANDIDATE, CUSTOM_NAME_CHANGED
  "messagePayload": "<base64(JSON.stringify(sdp|iceCandidate))>",
  "senderClientId": null
}
```

### Standard STUN/TURN (Fallback wenn getChannel keine liefert)

```
stun:servicediscovery.mypromethean.com:3478
stun:servicediscovery.mypromethean.com?transport=udp
turn:servicediscovery.mypromethean.com:3478  (user: u1, pass: p1)
```

### Bekannte Ports

| Service | Protokoll | Ports |
|---|---|---|
| Service Discovery API | TCP | 443 |
| STUN/TURN | UDP | 3478 |
| AWS KVS Media (WebRTC) | UDP | 443, 3478, 19302–19309, 49152–65535 |
| Miracast-Fallback Steuerung | TCP | 7236, 7250 |
| Miracast-Fallback Video | UDP | 19000–19100 |

## Nächste Aufgaben

> [!success] Kurswechsel 2026-06-03: zurück zu WebRTC/PIN — lokal End-to-End grün (siehe Changelog Session 9)

- [x] **WebRTC-Stack reaktiviert** (`session.rs` neu, `main.rs` PIN-GUI, Dev-Override) — 51 Tests grün (2026-06-03)
- [x] **Lokaler End-to-End-Test gegen board_sim** — 7634+ RTP-Pakete, VA-API aktiv (2026-06-03)
- [x] **VA-API-Fallback nach `state.rs` portiert** (2026-06-03) — `run_encoder` in `run_pipeline` aufgeteilt, bei `vah264enc`-Fehler einmalig x264enc-Retry; `fallback_encoder()` (2 Tests) verhindert Endlosschleife. 53 Tests grün.
- [x] **Version 0.2.2 + Release-Tarball** gebaut (2026-06-03) — `streamprometh-0.2.2.tar.gz`; `install.sh` vom Wi-Fi-Direct-/wpa_supplicant-Check auf WLAN-/Portal-Hinweis umgestellt; Binary loggt „0.2.2"
- [ ] **`StreamPrometh-Anleitung.pdf` neu erzeugen** — beschreibt noch den alten Auto-Discovery-Weg, nicht die PIN-Eingabe (nur für Kollegen nötig, nicht für eigenen Test)
- [ ] **Schultest mit echtem PIN** — Warteraum-Anzeige (clientId=Name) verifizieren
- [ ] **Aufräumen nach erfolgreichem Schultest:** schlafende WFD-Module (`mice.rs`, `p2p_discovery.rs`, `streaming.rs`) + `board_sim.rs` entscheiden/entfernen

---

- [x] **Promethean-API: Chrome Extension v4.3.2 dekompiliert** — vollständiges Protokoll dokumentiert (2026-05-26)
- [x] **Rust-Projekt angelegt:** `streamprometh/` mit Cargo.toml, Modulstruktur und `cargo check` grün (2026-05-26)
- [x] **Discovery-Client implementiert:** `checkPanel` + `getChannel` mit 9 mockito-Tests, alle grün (2026-05-26)
- [x] **Signaling-Client implementiert:** WebSocket Actor-Pattern, `SignalingSender`, 7 Tests grün (2026-05-26)
- [x] **State Machine implementiert:** `state.rs` mit PeerConnection, ashpd Screen-Capture, GStreamer-Encoder (2026-05-26)
- [x] **Board-Simulator implementiert:** `src/bin/board_sim.rs` — WebSocket-Server, SDP/ICE-Austausch, RTP-Frame-Zähler (2026-05-26)
- [x] **Board-Simulator lokal ausführen:** 1512–3173 RTP-Pakete empfangen — Stream funktioniert End-to-End (2026-05-26)
- [x] **Architekturwechsel zu Wi-Fi Direct (MICE/WFD):** GTK4-GUI, P2P Discovery, WFD-Handshake (2026-05-27/28)
- [x] **Stage 3: GStreamer H.264 → RTP/UDP Streaming implementiert** — `streaming.rs`, SOLID/TDD, VA-API Fallback (2026-05-28)
- [x] **Distributions-Paket:** `dist.sh` → `streamprometh-0.2.0.tar.gz`, `install.sh` für Arch/Debian/Fedora (2026-05-28)
- [x] **Kollegenanleitung:** `create_pdf.py` → `StreamPrometh-Anleitung.pdf`, 3 Seiten A4 (2026-05-28)
- [x] **Erster Schultest (2026-05-29):** Board erkannt ✅, aber P2P-Aktivierung Timeout (20s) — Laptop kam nicht in den Warteraum
- [ ] **VA-API verifizieren:** Im Schultest prüfen ob Intel VA-API aktiv ist — Statuszeile zeigt „Hardware-Encoding aktiv (VA-API)"
- [ ] **Latenz messen:** CPU-Last bei 1080p/30fps mit Software- vs. Hardware-Encoding vergleichen
- [x] ~~**Code aufräumen:** Alte WebRTC-Fragmente entfernen~~ — **HINFÄLLIG seit Pivot 2026-06-03.** Genau umgekehrt: `discovery.rs`/`signaling.rs`/`state.rs`/`board_sim.rs` sind jetzt der **aktive** Pfad. Zu entfernen sind stattdessen die schlafenden WFD-Module (siehe Aufräum-Eintrag oben).
- [ ] **Fehlerbehandlung aushärten:** Reconnect bei Verbindungsabbruch, saubere Fehler bei fehlendem Portal

## Board-Simulator — Plan (nächste Session)

Ziel: Die `streamprometh`-App ohne echtes Promethean-Board lokal testen.

Da `SignalingClient` nur eine WebSocket-URL bekommt, kann ein lokaler Server das Board vollständig simulieren:

```
streamprometh (unser Code)       board-sim (neues Binary)
       │                                │
       │── WS-Connect ────────────────►│  tokio-tungstenite accept_async
       │── SDP_OFFER (base64) ────────►│  webrtc-rs: create_answer()
       │◄── SDP_ANSWER (base64) ───────│  set_remote_description(offer)
       │── ICE_CANDIDATE ─────────────►│  add_ice_candidate()
       │◄── ICE_CANDIDATE ─────────────│  on_ice_candidate → senden
       │                                │
       │      WebRTC P2P aktiv          │
       │── H264-Frames (RTP) ──────────►│  on_track() → Frame-Zähler
       │                                │  ✅ "X Frames empfangen"
```

### Implementierung

**Neues Binary:** `src/bin/board_sim.rs`

```
[[bin]]
name = "board-sim"
path = "src/bin/board_sim.rs"
```

**Ablauf:**
1. `TcpListener::bind("127.0.0.1:0")` → zufälliger Port
2. `accept_async()` → WebSocket-Verbindung annehmen
3. Auf `SDP_OFFER`-Nachricht warten (base64 dekodieren)
4. `webrtc-rs` PeerConnection als Answerer aufbauen (kein STUN/TURN nötig — lokal)
5. `create_answer()` → `set_local_description()` → base64-kodiert als `SDP_ANSWER` senden
6. ICE Candidates via `on_ice_candidate` senden / empfangen
7. `on_track()` → H264-Frames zählen und loggen
8. Nach N Frames oder Timeout: Status ausgeben

**Starten:**
```bash
# Terminal 1: Simulator starten
cargo run --bin board-sim

# Terminal 2: App gegen Simulator
cargo run -- 000000 --signaling-url ws://127.0.0.1:{PORT}
```

> Dafür braucht `main.rs` einen `--signaling-url`-Override-Flag, der `getChannel` überspringt.

## Agent-Team

| Agent / Skill | Aufruf | Wann einsetzen |
|---|---|---|
| backend-agent | `/backend-agent` | Netzwerk-Logik, Service-Schichten, API-Design |
| rust-solid | `/rust-solid` | Rust-Architektur, Traits, Use Cases, Repositories, SOLID-Review |
| tdd | `/tdd` | Vor jeder Implementierung — Test zuerst |

> [!info] Reihenfolge beachten
> TDD kommt **vor** dem Produktionscode — erst `/tdd` aktivieren, dann implementieren, dann `/rust-solid`-Review.

## Coding-Regeln (immer anwenden)

> [!important] OOP + SOLID + TDD — gilt für jede `.rs`-Datei
> Diese Regeln gelten verbindlich, auch wenn kein Skill explizit aufgerufen wurde.

### Objektorientierung in Rust

- Verhalten gehört zur Datenstruktur: `impl`-Blöcke statt Freihand-Funktionen
- Traits als Interfaces — Abhängigkeiten immer auf Trait-Ebene, nicht auf konkreten Typen
- Konstruktoren als `new()` oder Builder-Pattern

### SOLID in Rust

| Prinzip | Rust-Umsetzung |
|---|---|
| **S** — Single Responsibility | Jede Struct hat eine Aufgabe; Logik nicht in `main.rs` |
| **O** — Open/Closed | Traits für Erweiterbarkeit statt `if`/`match` |
| **L** — Liskov | Trait-Implementierungen halten den Vertrag — kein `unimplemented!()` in Produktion |
| **I** — Interface Segregation | Kleine, fokussierte Traits (kein Allzweck-Trait) |
| **D** — Dependency Inversion | `fn new(dep: impl TraitX)` statt `fn new(dep: KonkreterTyp)` |

### TDD-Zyklus

```
1. cargo test → Test schreiben (schlägt fehl ✗)
2. Minimaler Produktionscode → Test grün ✓
3. Refactoring → cargo test bleibt grün ✓
```

Kein Produktionscode ohne vorher fehlschlagenden Test.

## Bekannte Stolperfallen

> [!warning] Wayland-Portals brauchen einen laufenden Portal-Dienst
> `xdg-desktop-portal` muss auf dem System laufen. Auf Arch manuell prüfen: `systemctl --user status xdg-desktop-portal`

> [!success] Promethean nutzt AWS KVS WebRTC — kein Black-Box-Protokoll
> Der Signaling-Teil ist Standard-AWS-API (offizielle Rust-Crates vorhanden). Einziges Black-Box-Element: der `servicediscovery.mypromethean.com`-Endpoint (PIN → AWS-Credentials + ChannelARN). Dieser muss per DevTools erschlossen werden.

> [!warning] H.264-Encoding-Latenz
> Bildschirm-Spiegeln ist latenzempfindlich. Software-Encoding (CPU) kann bei 1080p/60fps zu langsam sein. GStreamer mit Hardware-Encoding (`vaapi` für AMD/Intel) prüfen.

> [!warning] UDP 19000–19100 in der Schul-Firewall
> Schulnetzwerke haben oft restriktive Firewalls. Dieser UDP-Portbereich muss im Schulnetzwerk erlaubt sein — nicht nur auf dem Laptop. IT-Admin fragen.

> [!tip] Netzwerk-Analyse als erster Schritt
> Browser-DevTools (Network-Tab) auf `share.mypromethean.com` öffnen und PIN-Eingabe beobachten — so lernt man die API kennen, ohne zu raten.

## Ressourcen

- [ashpd (XDG Desktop Portal)](https://github.com/bilelmoussaoui/ashpd)
- [webrtc-rs](https://github.com/webrtc-rs/webrtc)
- [Promethean Screen Share](https://share.mypromethean.com)
- [GStreamer Rust Bindings](https://gitlab.freedesktop.org/gstreamer/gstreamer-rs)
- [ratatui (TUI)](https://ratatui.rs)
- [aws-sdk-kinesisvideo (Rust)](https://crates.io/crates/aws-sdk-kinesisvideo)
- [aws-sdk-kinesisvideosignaling (Rust)](https://crates.io/crates/aws-sdk-kinesisvideosignaling)
- [AWS KVS WebRTC JS SDK (Referenz für Protokoll)](https://github.com/awslabs/amazon-kinesis-video-streams-webrtc-sdk-js)
- [AWS KVS ConnectAsViewer API](https://docs.aws.amazon.com/kinesisvideostreams-webrtc-dg/latest/devguide/ConnectAsViewer.html)

## Implementierungsstand (Code-Referenz) — v0.2.0

```
streamprometh/src/
├── main.rs           — GTK4 GUI: BoardListe, Name-Feld, Verbinden/Trennen, Status-Spinner
├── p2p_discovery.rs  — Wi-Fi Direct Discovery via NetworkManager D-Bus (BoardEvent: Found/Lost/Unavailable)
├── mice.rs           — WFD/MICE Handshake: RTSP TCP 7236 + Verbindungsaufbau
├── streaming.rs      — GStreamer H.264 → RTP/UDP 19000 (SOLID: Traits + Generics, 10 Tests)
├── state.rs          — ashpd Screen-Capture + choose_h264_encoder() (VA-API / x264 Fallback)
├── discovery.rs      — (alt) DiscoveryClient: checkPanel + getChannel
├── signaling.rs      — (alt) SignalingClient: WebSocket Actor-Pattern
└── bin/
    └── board_sim.rs  — Board-Simulator (alt, WebRTC)
```

### Wichtige Dateien für den Schultest

| Datei | Zweck |
|---|---|
| `streamprometh-0.2.0.tar.gz` | Distributions-Paket (USB-Stick) |
| `install.sh` | Multi-Distro Install (Arch/Debian/Fedora), prüft wpa_supplicant |
| `StreamPrometh-Anleitung.pdf` | Kollegenanleitung, 3 Seiten A4 |
| `~/.local/share/streamprometh/streamprometh.log` | Log-Datei (täglich rotiert) |

### Streaming-Pipeline (streaming.rs)

```
pipewiresrc fd={fd} path={node_id} do-timestamp=true
! video/x-raw,framerate=30/1
! videoconvert
! [vah264enc (Hardware) ODER x264enc tune=zerolatency (Software-Fallback)]
! h264parse config-interval=-1
! rtph264pay pt=96 config-interval=-1
! udpsink host={board_ip} port=19000 sync=false async=false
```

- VA-API (`vah264enc`) wird automatisch versucht; bei Fehler → Retry mit `x264enc`
- `STREAMPROMETH_NO_VAAPI=1` erzwingt Software-Encoding
- Statuszeile zeigt Encoding-Art + ggf. Install-Hinweis für fehlenden VA-API-Treiber

### SOLID-Architektur in streaming.rs

```rust
trait ScreenCapture  { async fn start() -> Result<(OwnedFd, u32)> }
trait PipelineFactory { fn build(enc, pre, fd, node_id, board_ip, port) -> String }

struct Streamer<C: ScreenCapture, P: PipelineFactory> { ... }
// AshpdCapture → ruft crate::state::start_screen_capture()
// H264RtpPipeline → detect() wählt Encoder; with_encoder() für Tests ohne GStreamer-Init
```

### Abhängigkeiten (Cargo.toml)

| Crate | Version | Zweck |
|---|---|---|
| `tokio` | 1 | Async Runtime (rt-multi-thread, sync, signal) |
| `reqwest` | 0.13 | HTTP: checkPanel + getChannel |
| `tokio-tungstenite` | 0.29 | WebSocket: KVS Signaling |
| `futures-util` | 0.3 | SinkExt/StreamExt für WS |
| `webrtc` | 0.17 | RTCPeerConnection, SDP, ICE, Tracks |
| `gstreamer` | 0.23 | GStreamer-Core (Pipeline, Bus) |
| `gstreamer-app` | 0.23 | AppSink (Frame-Extraktion) |
| `ashpd` | 0.9 | XDG Screencast Portal (PipeWire-fd) |
| `serde` / `serde_json` | 1 | JSON-Serialisierung |
| `base64` | 0.22 | Signaling-Payload kodieren |
| `uuid` | 1 | Client-ID v4 |
| `clap` | 4 | CLI |
| `anyhow` | 1 | Fehlerbehandlung |
| `tracing` + `tracing-subscriber` | 0.1/0.3 | Logging |
| `mockito` | 1 (dev) | HTTP-Mock für Tests |

### GStreamer-Pipeline (state.rs: run_encoder)

```
pipewiresrc fd={pipewire_fd} path={node_id}
! videoconvert
! x264enc tune=zerolatency bitrate=5000 key-int-max=30
! h264parse config-interval=1
! video/x-h264,stream-format=byte-stream,alignment=au
! appsink name=sink sync=false max-buffers=2 drop=true
```

- Frames landen via `mpsc::channel` im Tokio-Task
- `TrackLocalStaticSample::write_sample(&Sample { data, duration, .. })` sendet an webrtc-rs
- Bus-Monitor in `spawn_blocking` erkennt EOS/Fehler
- `std::mem::forget(session/proxy)` hält ashpd-Session bis Prozessende am Leben

### Bekannte Einschränkungen

- **ashpd-Session-Lifecycle:** `std::mem::forget` für proxy + session — GStreamer liest den fd direkt. Cleanup erst bei Prozessende. Für sauberes Reconnect später refactorn.
- **x264enc (CPU):** Software-Encoding — bei 1080p/60fps möglicherweise zu langsam. `vaapih264enc` als VAAPI-Alternative evaluieren.
- **Nur im Schulnetz testbar:** `servicediscovery.mypromethean.com` antwortet öffentlich immer mit `LB OK`.
- **gstreamer 0.23 / GStreamer 1.28 auf System:** `gst-plugins-ugly` (für x264enc) und `gst-plugins-pipewire` (pipewiresrc) müssen installiert sein.

## Changelog

### 2026-06-03 (Session 9) — Architektur-Rückkehr zu WebRTC/PIN, lokal End-to-End grün

**Dritter Schultest (Wi-Fi Direct) — endgültige Sackgasse bestätigt:**
- v0.2.1: `AddAndActivateConnection2` kehrt jetzt in 47 ms zurück (der Timeout-Fix wirkte), bleibt aber bei „warte auf ACTIVATED" hängen. Die WFD-Assoziation erreicht `ACTIVATED` nie — `wait_for_activation` hat keinen Timeout → erneut stiller Hänger, nur eine Stufe später.
- **Erkenntnis:** Der NetworkManager-basierte Wi-Fi-Direct-Weg ist auf Linux exakt die GNOME-Network-Displays-Wand, gegen die das Projekt von Anfang an antrat. Das Windows-„Board-anklicken"-Erlebnis ist Miracast = ein zertifizierter OS-/Treiber-Stack, den Linux nicht hat. Diesen Weg weiter zu verfolgen = die kaputte Schicht nachbauen.

**Strategieentscheidung: zurück zum WebRTC-/PIN-Weg (Promethean Screen Share / Cloud).**
- Am Board bestätigt: der **6-stellige PIN wird angeboten** → der Browser-WebRTC-Weg ist offen. Er umgeht die Funkschicht komplett und läuft als normaler IP-Verkehr über das Schul-WLAN.
- Der WebRTC-Stack (`discovery.rs` / `signaling.rs` / `state.rs`) lag seit Session 5 als „toter Code" vor — vollständig und getestet. **Reaktiviert statt neu gebaut.**

**Umsetzung (TDD/SOLID):**
- **Neues Modul `session.rs`:** `connect` (PIN → checkPanel → getChannel → Signaling → Stream) + `run_channel`, hängt von Traits (`DiscoveryService`, `SignalingTransport`) ab → mit Mock testbar. 3 Tests grün (rot→grün dokumentiert).
- **`main.rs`:** GUI von Board-Liste auf **PIN-Feld** umgebaut; Dev-Override `STREAMPROMETH_SIGNALING_URL` überspringt die Cloud (board_sim-Test ohne Schulnetz).
- **`state.rs`:** Warteraum-Statusmeldung + `STREAM:`-Präfix (Fenster minimiert erst beim echten Stream, nicht im Warteraum).
- **Wi-Fi-Direct-Module** (`mice.rs`, `p2p_discovery.rs`, `streaming.rs`) als **schlafend** markiert (`#[allow(dead_code)]`) — bleiben als Rückfallebene, gelöscht wird erst nach erfolgreichem Schultest.
- **51 Tests grün** (vorher 48), `cargo build` sauber.

**Lokaler End-to-End-Test gegen `board_sim`: ERFOLGREICH ✅**
- Signaling → ICE `connected` → Screencast-Portal (PipeWire-Node) → `vah264enc` (VA-API) → **7634+ RTP-Pakete, steigend**. Der komplette Weg trägt.

**Härtung + Paketierung (alles in dieser Session erledigt):**
- ✅ **VA-API→x264-Fallback** nach `state.rs` portiert: `run_encoder` in `run_pipeline` aufgeteilt, bei `vah264enc`-Fehler einmalig x264enc-Retry (`fd` über beide Versuche gehalten); reine Funktion `fallback_encoder()` mit 2 Tests verhindert Endlosschleife. **53 Tests grün.**
- ✅ **Version 0.2.2** + `streamprometh-0.2.2.tar.gz` gebaut (Binary loggt „0.2.2"). `install.sh` vom irreführenden wpa_supplicant-/Wi-Fi-Direct-Check auf WLAN-/Portal-Hinweis umgestellt. `Cargo.toml`-Beschreibung „Miracast" → „WebRTC/PIN".
- ✅ **Schultest-Spickzettel** erzeugt: `SCHULTEST-SPICKZETTEL.md` + `StreamPrometh-Schultest-Spickzettel.pdf` (1 Seite A4, via `spickzettel_pdf.py`).

**Offen für nächste Session (nach dem Schultest):**
- Schultest mit echtem PIN — `clientId = Name` → Warteraum-Anzeige am Board verifizieren (Hypothese aus Session 4). Diagnose über `~/.local/share/streamprometh/streamprometh.log`.
- Optional vorab: x264-Gegencheck `STREAMPROMETH_NO_VAAPI=1` gegen board_sim.
- `StreamPrometh-Anleitung.pdf` (Kollegen) beschreibt noch den alten Auto-Discovery-Weg → vor Weitergabe neu erzeugen.
- Nach Erfolg: schlafende WFD-Module (`mice.rs`, `p2p_discovery.rs`, `streaming.rs`) + ggf. `board_sim.rs` entscheiden/entfernen.

---

### 2026-05-29 (Session 8) — Zweiter Schultest gescheitert + Diagnose-Härtung (v0.2.1)

**Schultest-Ergebnis:**
- ActivPanels werden in der Liste angezeigt, Auswahl + „Verbinden" funktioniert
- App bleibt bei „Verbinde per Wi-Fi Direct mit ActivPanel-AFXNR9…" stehen — **kein Warteraum, keine Fehlermeldung, nichts**
- Logdatei weiterhin nicht auffindbar
- `iw phy` bestätigt: Hardware kann P2P (P2P-client/GO/device) — Problem liegt nicht an der Hardware

**Root-Cause-Analyse:**
- **Log fehlte**, weil `tracing_appender::non_blocking` den Puffer erst beim Programmende flusht. Die App *hing* (endete nie) → Guard nie gedroppt → Logzeilen blieben im RAM. Gemeinsame Wurzel mit dem Hänger: **die App hängt, statt mit Fehler abzubrechen.**
- **Hänger** in `mice.rs`: `add_and_activate_connection2` hatte **keinen Timeout** (nur `wait_for_activation` danach). NetworkManager kehrt erst nach abgeschlossenem **WPS-Provisioning** zurück — erwartet das Board PIN/Bestätigung auf der Tafel, kehrt der Call nie zurück. Deckt sich mit dem bekannten GNOME-Network-Displays-Problem.

**Fixes (v0.2.1):**
- `main.rs`: `non_blocking` entfernt → `RollingFileAppender` schreibt direkt/blockierend; zweites Log-Ziel stderr (Registry mit zwei fmt-Layern); Panic-Hook → Logdatei; Log-Pfad per `eprintln!`
- `mice.rs`: `add_and_activate_connection2` in `timeout(P2P_TIMEOUT, …)` gekapselt mit klarer WPS-Fehlermeldung; `info`-Logs vor/nach dem Call
- Version auf **0.2.1** gebumpt — Startzeile loggt die Version → eindeutige Erkennung der laufenden Binary
- `streamprometh-0.2.1.tar.gz` gebaut, 48 Tests grün

**Offene Architekturfrage (WPS):** Die App baut Wi-Fi Direct über NM auf wie GNOME Network Displays → wiederholt evtl. denselben Fehler. Optionen: **A** wpa_supplicant P2P direkt (expliziter PBC), **B** MICE-over-Infrastructure (mDNS + direkter TCP, kein WPS, scheitert evtl. an AP-Isolation), **C** Promethean Cloud/WebRTC (PIN, offizieller Weg). **Erst messen, dann entscheiden.**

**Nächster Schultest — Diagnose-Plan:**
- App testen → Log liegt jetzt sicher unter `~/.local/share/streamprometh/streamprometh.log`
- Parallel: `journalctl -u NetworkManager -f` + `sudo journalctl -u wpa_supplicant -f`
- Am Board beobachten: Erscheint ein **PIN-Dialog / „Zulassen?"-Abfrage** auf der Tafel?

---

### 2026-05-29 (Session 7) — Erster Schultest + Bugfixes

**Schultest-Ergebnis:**
- Board wurde erkannt (P2P Discovery ✅)
- Laptop kam nicht in den Warteraum: `P2P-Aktivierung Timeout (20s): deadline has elapsed`
- Log-Datei war nicht auffindbar (`~/.local/share/streamprometh/` existierte nicht)

**Fix 1 — P2P-Timeout (`mice.rs`):**
- `P2P_TIMEOUT` von 20s auf 60s erhöht (Wi-Fi Direct P2P-Handshake + DHCP dauert 30–50s)
- Polling (`state().await` alle 500ms) durch D-Bus Property-Change-Signal ersetzt (`receive_state_changed()` → sofortige Reaktion)
- `futures_util::StreamExt` Import ergänzt

**Fix 2 — Log-Verzeichnis (`main.rs`):**
- `create_dir_all` vor `tracing_appender::rolling::daily` — Verzeichnis wird jetzt automatisch angelegt
- `tracing_appender` erstellt das Verzeichnis **nicht** selbst, schreibt stillschweigend nichts

**Release:** `streamprometh-0.2.0.tar.gz` neu gebaut (gleiche Versionsnummer, neue Fixes)

---



### 2026-05-28 (Session 6) — v0.2.0 fertig, Schultest vorbereitet

**Architektur-Pivot abgeschlossen:** Von PIN/WebRTC zu Wi-Fi Direct (MICE/WFD). Die App erkennt Boards automatisch im Netzwerk — kein PIN mehr nötig.

**Stage 3: Streaming (streaming.rs)**
- SOLID/TDD-Architektur erklärt und implementiert: `ScreenCapture` + `PipelineFactory` Traits, `Streamer<C, P>` generisch
- GStreamer-Pipeline: `pipewiresrc → vah264enc/x264enc → h264parse → rtph264pay → udpsink` auf Port 19000
- VA-API Fallback: Automatischer Retry mit x264enc wenn vah264enc-Pipeline fehlschlägt
- `STREAMPROMETH_NO_VAAPI=1` Umgebungsvariable für erzwungenes Software-Encoding
- 10 Unit-Tests ohne GStreamer-Init (pipeline string validation)
- Statuszeile zeigt Encoding-Art + Install-Hint für fehlenden VA-API-Treiber

**mice.rs Bug-Fix:** `read_rtsp_message` hatte `??;` + `unreachable!()` — hat bei jedem Aufruf den RtspMessage-Wert verworfen und dann Panic ausgelöst. Fix: `?` statt `??;` + `unreachable!()`.

**GTK4 GUI verbessert:**
- `BoardEvent::Unavailable` zeigt kontextspezifische Fehlermeldungen (wpa_supplicant vs iwd vs NetworkManager)
- Fenster minimiert beim Stream-Start, öffnet sich wieder bei Fehler oder Verbindungsende

**Install & Distribution:**
- `install.sh` überarbeitet: Multi-Distro (Arch/Debian/Fedora), prüft wpa_supplicant vs iwd, zeigt VA-API Treiber-Hinweise
- `dist.sh` erstellt → `streamprometh-0.2.0.tar.gz` auf USB-Stick, bereit für Schultest
- Deinstallation: `sudo ./uninstall.sh`

**Kollegenanleitung PDF:**
- `create_pdf.py` → `StreamPrometh-Anleitung.pdf` (3 Seiten A4, reportlab)
- Enthält: Systemvoraussetzungen, Installation, 6-Schritt-Bedienungsanleitung, Hardware-Encoding Tabelle, Fehlerbehebung, Log-Analyse

**Schultest morgen:** App läuft auf Desktop, lädt Release-Binary. Laptop in Schule muss aus Paket installieren (`sudo ./install.sh`). Log liegt unter `~/.local/share/streamprometh/streamprometh.log`.

---

### 2026-05-26 (Session 4)

- **GTK4-GUI implementiert:** PIN-Eingabe, Name-Eingabe (mit Hostname-Vorbelegung), Verbinden-Button, Spinner, Status-Label
- **Name-Hypothese:** `clientId` in `getChannel` auf den eingegebenen Namen gesetzt (statt UUID) — AWS KVS leitet `senderClientId` ans Board weiter, Board zeigt ihn im Warteraum. Muss im Schulnetz verifiziert werden.
- **rustls-Fix:** `ring`-CryptoProvider muss vor GTK/Tokio initialisiert werden — in `main()` vor allem anderen aufrufen
- **Release-Binary gebaut:** `target/release/streamprometh` — auf USB-Stick kopiert, bereit für Schultest
- **Schultest vorbereitet:** `RUST_LOG=info ./streamprometh 2>&1 | tee schultest.log` — Log für Diagnose mitschreiben

### 2026-05-26 (Session 3)

- **Board-Simulator implementiert:** `src/bin/board_sim.rs` — TcpListener auf zufälligem Port, WebSocket `accept_async`, SDP/ICE-Austausch als webrtc-rs Answerer, `on_track` RTP-Pakete zählen
- **`main.rs` erweitert:** `--signaling-url`-Flag überspringt `checkPanel`/`getChannel` — für lokale Tests ohne Schulnetz
- **Cargo.toml:** `[[bin]] board-sim` hinzugefügt — `cargo run --bin board-sim`
- **16/16 Tests weiterhin grün**

### 2026-05-26 (Session 2)

- **`signaling.rs` implementiert:** WebSocket Actor-Pattern (`SignalingClient` + `SignalingSender`), 7 Unit-Tests mit lokalem `TcpListener + accept_async` grün
- **`state.rs` implementiert:** vollständige State Machine — `SignalingClient`, `RTCPeerConnection` (webrtc-rs 0.17), ashpd Screencast-Portal, GStreamer-Pipeline
- **GStreamer-Integration:** `pipewiresrc → x264enc → h264parse → appsink` → `TrackLocalStaticSample::write_sample`
- **GStreamer 0.23 + GStreamer 1.28.3** auf System vorhanden: `pipewiresrc`, `x264enc`, `h264parse` alle verfügbar
- **16/16 Tests grün** nach allen Änderungen

### 2026-05-26 (Session 1)

- **Rust-Projekt angelegt:** `streamprometh/` mit Cargo.toml + Modulstruktur (`discovery`, `signaling`, `state`), `cargo check` grün
- **Chrome Extension v4.3.2 dekompiliert:** Vollständiges Protokoll aus JS-Source extrahiert
- **Stack vereinfacht:** Kein AWS SDK nötig — `getChannel` liefert pre-signierte WebSocket-URL direkt
- **Zwei API-Endpoints dokumentiert:** `GET /checkPanel?panelId=` + `GET /getChannel?panelId=&clientId=`
- **WebSocket-Format dokumentiert:** `{ action, messagePayload: base64(JSON), recipientClientId }`
- **Viewer sendet SDP-Offer** (nicht das Board!) — H264, 5 Mbit/s, trickle ICE
- **Board nur aus Schulnetz erreichbar** — `servicediscovery.mypromethean.com` blockiert öffentliche IPs

### 2026-05-25

- **Projekt angelegt:** Beschreibung, technischer Ablauf, Stack und Aufgaben definiert
- **Problem analysiert:** GNOME Network Displays scheitert an UDP 19000–19100 und Promethean-Warteraum-Protokoll
- **Lösungsansatz:** WebRTC primär (wie Promethean Web), Miracast over Infrastructure als Fallback
- **Stack-Entscheidung:** `ashpd` für Wayland-Screencast, `webrtc-rs` für Stream, `reqwest` für PIN-API
