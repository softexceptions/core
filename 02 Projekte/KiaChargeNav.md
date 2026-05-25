---
title: KiaChargeNav
tags:
  - projekt
  - aktiv
status: in-progress
stand: 2026-05-24
---

# KiaChargeNav

> [!important] Nach `/init` in die CLAUDE.md des Projekts eintragen:
> `Projektbeschreibung: core/02 Projekte/KiaChargeNav.md`

Flutter-App für Android/iOS: Spracheingabe während der Fahrt → nächste Ladestation in Fahrtrichtung finden → Ziel per Send2Car ins Kia-Navi einspeisen.

> [!info] Ziel
> Das Fummelei am Infotainment während der Fahrt eliminieren. Sprachbefehl → App findet beste Station → Kia zeigt „Neues Ziel empfangen" — ein Knopfdruck zur Bestätigung im Auto genügt.

> [!tip] Wichtige Erkenntnis: Kein Routen-Transfer
> Es wird kein Routenplan übertragen, sondern nur ein **Ziel (Koordinaten + Name)** via Send2Car. Das Kia-Navi berechnet die Route selbst — basierend auf Standort, Verkehr und Ladestand.

## Technischer Ablauf

**Schritt 1 — Spracheingabe (primär) oder Button (fallback)**
Nutzer spricht: „Navigiere mich zur nächsten Shell-Ladestation" oder „EWE Go" etc.
Flutter `speech_to_text` erkennt den Text → Keyword-Matching extrahiert das Netzwerk (Shell → `Shell Recharge`, EWE Go → `EWE Go`, Ionity → `Ionity` usw.).
Alternativ: große Buttons für die häufigsten Netze als Fallback.

**Schritt 2 — Standort + Fahrtrichtung ermitteln**
Flutter liest GPS des Smartphones: Koordinaten **und Heading** (Fahrtrichtung in Grad).

> [!warning] Nicht Kia-API-GPS verwenden
> Die Kia-API liefert GPS-Daten verzögert (Batterie-Schonung). Das Smartphone-GPS ist die richtige Quelle.

**Schritt 3 — Ladestation finden (nächste in Fahrtrichtung)**
Anfrage an OpenChargeMap oder GoingElectric API mit:
- aktuellen Koordinaten
- Suchradius (z.B. 15 km)
- Netzwerk-Filter (z.B. `Shell Recharge`)

Aus den Treffern wird die Station gewählt, die **am nächsten in Fahrtrichtung** liegt. Dazu wird der Winkel zwischen Heading und der Richtung zur Station berechnet — Stationen mit kleinem Winkelabstand werden bevorzugt.

**Schritt 4 — Flutter quittiert**
App zeigt kurz: „Shell Recharge · Musterstraße · 4,2 km · wird gesendet…" → „✓ Gesendet"
Ablenkungsarm: große Schrift, auto-dismiss nach 3 Sekunden.

**Schritt 5 — Python sendet ans Fahrzeug**
FastAPI-Backend ruft `send_destination` (oder `send_poi`) der `hyundai-kia-connect-api` auf mit Latitude, Longitude und Stationsname.

**Effekt im Auto:** Popup im Display → „Neues Ziel empfangen — Als Zwischenziel hinzufügen?" → Fahrer bestätigt mit einem Tipp.

> [!tip] Zwischenziel vs. Ziel
> Ob die API ein echtes Zwischenziel (Waypoint) oder nur ein neues Ziel setzt, wird beim ersten Test geprüft. Das Display-Popup des Kia bietet „Als Zwischenziel" an — ob das die API oder das Navi-UI steuert, muss verifiziert werden.

> [!tip] Bonus: Batterie-Vorkonditionierung
> Wenn das Kia-Navi eine Schnellladestation als Ziel empfängt, aktiviert es automatisch die Akku-Vorkonditionierung (Heizung im Winter). Funktioniert nur via Send2Car — genau unser Weg.

## Architektur

```
Nutzer spricht: „Shell-Ladestation"
  │
  ▼
Flutter App
  ├── speech_to_text → Keyword-Matching → Netzwerk-Filter
  ├── GPS: Koordinaten + Heading (Fahrtrichtung)
  ├── OpenChargeMap / GoingElectric API
  │     └── Filter: Netzwerk + Radius → nächste in Fahrtrichtung
  ├── UI: Bestätigung anzeigen (auto-dismiss)
  └── HTTP POST → FastAPI Backend
                    └── hyundai-kia-connect-api
                          └── send_destination(lat, lon, name)
                                └── Kia Server (Mobilfunk)
                                      └── Popup im Fahrzeug-Display
```

## Stack

| Schicht           | Technologie                           |
| ----------------- | ------------------------------------- |
| Mobile App        | Flutter + Dart                        |
| Spracheingabe     | `speech_to_text` (Flutter Package)    |
| GPS + Heading     | `geolocator` (Flutter Package)        |
| Ladestation-Daten | OpenChargeMap API + GoingElectric API |
| Backend           | FastAPI + Python                      |
| Kia-Anbindung     | `hyundai-kia-connect-api` (PyPI)      |
| Datenbank         | keins (vorerst)                       |
| Deployment        | Proxmox LXC / Docker                  |

## Unterstützte Ladenetzwerke (geplant)

### Deutschland (primär)
- Shell Recharge
- EWE Go
- Ionity
- EnBW
- Aral Pulse
- Fastned
- Tesla Supercharger

### International (Ausland)
- ChargePoint (Europa + USA)
- Allego (Benelux, DE, FR, AT)
- Osprey (UK)
- Electrify America (USA)
- Blink (USA)

> [!tip] Konfiguration statt Hardcoding
> Netzwerke als Konfigurationsliste anlegen — nicht hardcoden. Neue Netze werden nur in der Keyword-Liste und im UI ergänzt, kein Code-Umbau.

## Auslands-Betrieb

Die App muss auch im Ausland ohne manuelle Anpassungen funktionieren. Betroffen sind vier Bereiche:

### 1 — Ladestation-APIs

| Komponente | Inland (DE) | Ausland |
|---|---|---|
| GoingElectric | Primärquelle ✅ | Europaweite Abdeckung ✅ — aber DE-fokussiert |
| OpenChargeMap | Fallback | **Primärquelle im Ausland** — weltweite Daten |

**Strategie:** Backend prüft anhand der Koordinaten, ob die Position in Deutschland liegt. Außerhalb DE wird OpenChargeMap bevorzugt, GoingElectric als Fallback.

```
DE-Koordinaten  → GoingElectric → OCM (Fallback)
EU/Welt         → OpenChargeMap → GoingElectric (Fallback)
```

> [!info] Koordinaten-Check im Backend
> Einfache Bounding-Box für Deutschland (47.3°N–55.0°N, 6.0°E–15.0°E) reicht aus — keine externe Geocoding-API nötig.

### 2 — Kia-API Region

`Region.Europe` in der `hyundai-kia-connect-api` deckt alle EU-Länder ab — kein Umschalten beim Fahren ins Ausland nötig.

> [!warning] Außerhalb der EU
> Kia US / CA nutzen andere Server (`Region.USA`, `Region.Canada`). Für Nicht-EU-Reisen müsste die Region manuell in der Backend-`.env` umgestellt werden. Vorerst kein automatischer Wechsel geplant.

### 3 — Keyword-Matching (Spracheingabe)

Internationale Netzwerke kommen mit eigenen Namen — die `network_matcher.dart` muss erweitert werden:

| Netzwerk | Keywords |
|---|---|
| ChargePoint | „chargepoint", „charge point" |
| Allego | „allego" |
| Electrify America | „electrify america", „electrify" |
| Tesla | „tesla", „supercharger", „tesla supercharger" |
| Osprey | „osprey" |

> [!tip] Länder-adaptive Button-Anzeige (Idee)
> Später: GPS-Land erkennen → häufigste Netze des Landes als Buttons einblenden. Für v1 reicht die erweiterte Keyword-Liste.

### 4 — Backend-Erreichbarkeit

Das Backend läuft auf `https://kia.softexceptions.com` — erreichbar über Mobilfunk-Roaming aus dem gesamten EU-Ausland. EU-Roaming ist seit 2017 ohne Aufpreis. Außerhalb der EU: normaler Datenverbrauch über Roaming.

> [!tip] Offline-Fallback (Idee für später)
> Letzte erfolgreiche Station im Gerät cachen — bei Tunneln oder schlechtem Netz zumindest den letzten Suchbegriff anzeigen.

## Kia-API

### Inoffizielle Schnittstellen (Reverse Engineering)

| Option | Sprache | Kosten | Funktion |
|---|---|---|---|
| [hyundai-kia-connect-api](https://github.com/Hyundai-Kia-Connect/hyundai_kia_connect_api) | Python | Kostenlos | `send_destination` / `set_navigation` — **aktiv genutzt ✅** |
| [bluelinky](https://github.com/Hacksore/bluelinky) | Node.js | Kostenlos | Alternative zu hyundai-kia-connect-api, ähnlicher Funktionsumfang |
| [kia_uvo](https://github.com/Hyundai-Kia-Connect/kia_uvo) | Python (HACS) | Kostenlos | Home Assistant Integration — nicht für direkten API-Einsatz gedacht |

> [!warning] Risiko inoffizielle APIs
> Basieren auf Reverse Engineering der Kia Connect App — können bei Kia App-Updates oder Server-Änderungen brechen. `hyundai-kia-connect-api` wird aktiv gepflegt (v4.x, Mai 2026).

### Offizielle / Kommerzielle Schnittstellen

| Option | Typ | Kosten | Funktion |
|---|---|---|---|
| [Smartcar API](https://smartcar.com/brand/kia) | Kommerziell | Freemium | `send-destination` Endpoint, EU-Support, SDKs für Python/Node/Java |
| [High Mobility](https://www.high-mobility.com/car-data/kia) | Kommerziell | Freemium | Telematics-Daten, standardisiertes Format (HMKit) |
| Kia Connect Business API | B2B / Fleet | Auf Anfrage | Fleet-Management, nur für gewerbliche Partner |

**Empfehlung:** `hyundai-kia-connect-api` für dieses Projekt — kostenlos, EU-Support, `set_navigation` bereits getestet ✅. Smartcar als kommerzielle Fallback-Option wenn Stabilität kritisch wird.

## Design

> [!info] UI-Kontext für `/ui-ux-pro-max`
> Diese Anforderungen beim Skill-Aufruf mitgeben: `/ui-ux-pro-max — Kontext: core/02 Projekte/KiaChargeNav.md`

**Nutzungskontext:** Fahr-App — Interaktion während der Fahrt, ein Finger, kein Blick länger als 1 Sekunde.

### UI-Anforderungen

| Anforderung | Begründung |
|---|---|
| Touch-Targets min. 72px | Treffsicherheit beim Fahren |
| Ein-Finger-Bedienung | Lenkrad in der anderen Hand |
| Auto-dismiss nach 3 Sek. | Kein aktives Wegtippen nötig |
| Hoher Kontrast | Lesbarkeit bei Sonneneinstrahlung |
| Maximale 2 Interaktionen pro Vorgang | Ablenkungsminimierung |
| Große, klare Schrift | min. 18sp Fließtext |

### Stil-Richtung

- **Dark Mode** als Standard — weniger Blendung nachts
- **Wenige Farben** — Primär + Status (Erfolg/Fehler)
- **Netzwerk-Buttons** mit Logo-Farbe des jeweiligen Anbieters (Shell-Gelb, Ionity-Lila, …)
- Kein Schnickschnack — jedes Element muss einen Zweck haben

## Agent-Team

| Agent / Skill | Aufruf | Wann einsetzen |
|---|---|---|
| Backend Agent | `/backend-agent` | FastAPI-Routen, KiaConnectService, Token-Handling |
| python-solid | `/python-solid` | SOLID-Review bei Python-Dateien |
| flutter-solid | `/flutter-solid` | Flutter-Architektur, Widgets, Use Cases, Repositories |
| tdd | `/tdd` | Vor jeder Implementierung — Test zuerst |
| ui-ux-pro-max | `/ui-ux-pro-max` | UI/UX-Design — Stile, Farben, Komponenten, UX-Review |

## Skills (automatisch anwenden)

- `.dart` Dateien → `/tdd` **vor** der Implementierung, `/flutter-solid` **nach** der Implementierung
- `.py` Dateien → `/tdd` **vor** der Implementierung, `/python-solid` **nach** der Implementierung

> [!tip] Reihenfolge beachten
> TDD kommt **vor** dem Produktionscode — erst `/tdd` aktivieren, dann implementieren, dann SOLID-Review.

## Claude-Verhaltensregeln

- **Vault first:** Projektkontext immer aus `core/02 Projekte/KiaChargeNav.md` lesen
- **context7 Pflicht:** Vor jeder Library-Nutzung aktuelle Docs holen — nie auf Trainingswissen verlassen

| Library | context7 | Anmerkung |
|---|---|---|
| `riverpod` | ✅ Immer | API ändert sich aktiv zwischen Versionen |
| `speech_to_text` | ✅ Immer | Plattform-Konfiguration (Android/iOS) variiert |
| `geolocator` | ✅ Immer | Permissions-Setup ändert sich |
| `mocktail` | ✅ Immer | Syntax-Änderungen möglich |
| `FastAPI` | ✅ Immer | DI-Patterns und async-Verhalten |
| `hyundai-kia-connect-api` | ⚠️ Nicht verfügbar | Direkt GitHub-Repo lesen: [Link](https://github.com/Hyundai-Kia-Connect/hyundai_kia_connect_api) |

## Fehlerbehandlung

Nur der Erfolgsfall ist schön — diese Fälle müssen explizit behandelt werden:

| Fehlerfall | Verhalten in der App |
|---|---|
| Kein Internet (Tunnel) | Toast: „Kein Netz — bitte später versuchen" — kein Absturz |
| Kia-API antwortet nicht | Timeout nach 10 Sek., Toast: „Fahrzeug nicht erreichbar" |
| Keine Station im Radius | Toast: „Keine Shell-Station in 15 km Fahrtrichtung gefunden" |
| Backend nicht erreichbar | Toast: „Backend offline" — App bleibt bedienbar |
| Sprache nicht erkannt | Fallback auf Button-Auswahl — kein stiller Fehler |

> [!tip] Grundregel
> Die App darf während der Fahrt niemals einfrieren oder einen leeren Bildschirm zeigen. Jeder Fehler braucht eine sichtbare, verständliche Meldung — auto-dismiss nach 4 Sekunden.

## Nächste Aufgaben

- [x] **Kia-API testen:** `hyundai-kia-connect-api` mit einfachem Python-Script verbinden.
  **Ergebnis:** Login ✅, Fahrzeug-ID ausgelesen ✅, `set_navigation` mit `POIInfo` sendet erfolgreich (Message-ID erhalten) ✅
  Popup-Verhalten (Zwischenziel vs. Hauptziel) → erst im Auto verifizierbar, wenn Flutter-App steht.

- [x] **Projektstruktur aufsetzen** nach [[04 Ressourcen/Claude/Neues Projekt anlegen]].
  **Ergebnis:** `app/` (Flutter 3.44.0) + `backend/` (FastAPI) angelegt, Packages installiert.

- [x] **Backend-Authentifizierung definieren:** Wie authentifiziert sich die Flutter-App am FastAPI-Backend?
  **Ergebnis:** API-Key via `X-Api-Key` Header. Key liegt in `app/.env` (BACKEND_API_KEY) und `backend/.env` (API_KEY). FastAPI prüft Header auf beiden Endpoints `/find` und `/send`. Flutter lädt Key per `flutter_dotenv` beim Start.

- [x] **Ladestation-API erkunden:** Netzwerk-Filter + Heading-basierte Sortierung implementieren.
  **Ergebnis:** `backend/services/charging_service.py` — `find_nearest(lat, lon, heading, network, radius_km)` ✅
  Primärquelle: GoingElectric (726 Netzwerke, exzellente DE-Abdeckung). Fallback: OpenChargeMap.
  Heading-Logik: Stationen ≤60° von Fahrtrichtung bevorzugt, Fallback auf nächste wenn Korridor leer.
  Getestete Netze aus Hamburg (alle gefunden): Shell, IONITY, EnBW, Aral, EWE Go, Fastned, Tesla.

- [x] **Flutter speech_to_text einbinden:** Spracheingabe → Keyword → Netzwerk-Mapping.
  **Ergebnis:** `lib/services/network_matcher.dart` (reines Dart, 6 Unit-Tests ✅) + `lib/services/speech_service.dart` (wraps SpeechToText, nutzt neue `SpeechListenOptions`-API). Android-Permissions in `AndroidManifest.xml` eingetragen. `flutter analyze` clean.

- [x] **Flutter-UI bauen:** Ablenkungsarm, große Flächen, auto-dismiss Bestätigung.
  **Ergebnis:** Vollständige UI implementiert — `HomeScreen` mit State-Machine (`idle → listening → processing → confirming → error`), `MicButton` (pulsierend beim Hören), `NetworkChip`-Grid (7 Schnell-Netze mit Brandfarben), `StationCard` (auto-dismiss 3s / 4s), Error-Toast. Dark Mode, min. 72px Touch-Targets, max. 2 Interaktionen.

## Deployment

### Android (primärer Weg)

**Phase 1 — Entwicklung**
Einmalig auf dem Smartphone: Entwickleroptionen aktivieren (7× auf Build-Nummer tippen) → USB-Debugging einschalten.

```
# Über USB
flutter run

# Über WLAN — kabellos (Android 11+)
flutter run --device-id <wireless-device-id>
```

**Phase 2 — Tägliche Nutzung (APK sideloaden)**
```
flutter build apk --release
```
APK liegt unter `build/app/outputs/flutter-apk/app-release.apk` → auf Telefon übertragen → installieren (einmalig „Unbekannte Quellen" erlauben).

**Phase 3 — Updates (optional)**
APK auf eigenem Server (Proxmox/Docker) ablegen → per Browser-Download aktualisieren. Kein Play Store nötig.

---

### iOS (Hinweis für spätere Nutzung)

Flutter-Code läuft auf iOS ohne Änderungen — aber der **Build für iOS benötigt macOS mit Xcode**. Lösung von Linux aus: **Codemagic** (Cloud-CI/CD speziell für Flutter).

> [!info] Codemagic — iOS-Build ohne Mac
> 1. Code auf GitHub/GitLab pushen
> 2. [Codemagic](https://codemagic.io) verbindet sich mit dem Repository
> 3. Codemagic baut die iOS-App auf einem Mac in der Cloud
> 4. Ergebnis: `.ipa`-Datei → via TestFlight auf iPhone installieren
>
> **Free Tier:** 500 Build-Minuten/Monat — für ein persönliches Projekt ausreichend.

## Bekannte Stolperfallen

> [!warning] Kia EU vs. US API
> Immer `Region.Europe` setzen — falsche Region führt zu Authentifizierungsfehlern.

> [!warning] Kia-Zugangsdaten nur im Backend
> Nie in der Flutter-App speichern. Die App spricht nur mit dem eigenen FastAPI-Backend.

> [!info] Kia-Popup: Erster Live-Test 2026-05-24
> **Empfang funktioniert ✅** — Kia hat beide `set_navigation`-Aufrufe aus der Entwicklung gespeichert und beim Autostart als „Zwei Ziele erhalten" angezeigt.
> **Aber:** Beim Tippen auf „Ziele anzeigen" war alles weg — vermutlich weil keine aktive Navigation lief.
> **Erkenntnis:** Kia speichert Nachrichten bis das Auto gestartet wird (kein sofortiger Verfall). Für die volle Funktion (Ziel übernehmen) muss wahrscheinlich eine aktive Navigation laufen oder der Test direkt beim Losfahren gemacht werden.
> **Nächster Test:** App während der Fahrt benutzen — dann erscheint das Popup im richtigen Kontext.

> [!warning] Fahrzeug-ID erforderlich
> Die `hyundai-kia-connect-api` benötigt eine Vehicle-ID, wenn mehrere Fahrzeuge im Kia-Account sind. ID beim ersten API-Test auslesen und in der Backend-Konfiguration (`.env`) festhalten.

> [!warning] Backend-Authentifizierung nicht vergessen
> Die Flutter-App muss sich am eigenen FastAPI-Backend authentifizieren — sonst ist der `/send`-Endpoint offen zugänglich. Lösung: API-Key in `.env`, Flutter sendet ihn als Header mit.

> [!tip] Heading-Stabilisierung bei Kurvenfahrt
> Der GPS-Heading springt auf kurvenreichen Strecken. Lösung: Durchschnitt der letzten 5 Sekunden GPS-Readings verwenden statt Momentanwert — ergibt eine stabile Fahrtrichtung.

> [!tip] Suchradius konfigurierbar halten
> 15 km ist ein guter Standard. Auf der Autobahn sind 30–50 km sinnvoller, in der Stadt reichen 5 km. Als Konfigurationswert anlegen, nicht hardcoden.

## Kia API — Vehicle-Felder (hyundai-kia-connect-api v4.14.1)

Alle Felder sind im `Vehicle`-Objekt nach `vm.update_all_vehicles_with_cached_state()` verfügbar.

### Fahrzeugstatus
| Attribut | Typ | Bedeutung |
|---|---|---|
| `engine_is_running` | `bool` | Motor läuft → für Send-Check relevant |
| `is_locked` | `bool` | Fahrzeug verriegelt |
| `car_battery_percentage` | `int` | 12V-Batterie (%) |
| `dtc_count` / `dtc_descriptions` | `int` / `dict` | OBD-Fehlercodes |
| `smart_key_battery_warning_is_on` | `bool` | Schlüsselbatterie schwach |

### Batterie & Laden (EV)
| Attribut | Typ | Bedeutung |
|---|---|---|
| `ev_battery_percentage` | `int` | Ladestand (%) |
| `ev_battery_is_charging` | `bool` | Lädt gerade |
| `ev_battery_is_plugged_in` | `bool` | Kabel eingesteckt |
| `ev_battery_soh_percentage` | `int` | Batteriegesundheit (%) |
| `ev_driving_range` | `float` | Reichweite (km) |
| `ev_battery_precondition_enabled` | `bool` | Vorkonditionierung aktiv |
| `ev_estimated_fast_charge_duration` | `int` | Schnellladezeit bis Ziel (min) |
| `ev_battery_winter_mode` | `bool` | Wintermodus aktiv |

### Klima
| Attribut | Typ | Bedeutung |
|---|---|---|
| `air_control_is_on` | `bool` | Klimaanlage an |
| `air_temperature` | `float` | Innentemperatur |
| `outside_temperature` | `float` | Außentemperatur |
| `defrost_is_on` | `bool` | Frontscheibe enteist |
| `steering_wheel_heater_is_on` | `bool` | Lenkradheizung |
| `front_left/right_seat_status` | `str` | Sitzheizung/-kühlung |

### Türen & Fenster
| Attribut | Typ | Bedeutung |
|---|---|---|
| `front/back_left/right_door_is_open` | `bool` | Tür offen |
| `trunk_is_open` | `bool` | Kofferraum offen |
| `hood_is_open` | `bool` | Motorhaube offen |
| `sunroof_is_open` | `bool` | Schiebedach offen |

### Warnungen
| Attribut | Typ | Bedeutung |
|---|---|---|
| `washer_fluid_warning_is_on` | `bool` | Scheibenwasser leer |
| `brake_fluid_warning_is_on` | `bool` | Bremsflüssigkeit |
| `fuel_level_is_low` | `bool` | Kraftstoff niedrig |

### Position & Fahrten
| Attribut | Typ | Bedeutung |
|---|---|---|
| `location_latitude` / `location_longitude` | `float` | GPS-Position |
| `odometer` | `float` | Kilometerstand |
| `total_driving_range` | `float` | Gesamtreichweite |
| `day_trip_info` / `month_trip_info` | Objekt | Tages-/Monatsfahrten |
| `daily_stats` | Objekt | Energieverbrauch täglich |

## Ladestation-Datenquellen

### GoingElectric API (primär)
- Deutsche API, sehr gute DE-Abdeckung
- Braucht API-Key (`GOINGELECTRIC_API_KEY` in `.env`)
- Kennt exakte Netzwerknamen (Shell Recharge, IONITY, EnBW usw.)
- Endpoint: `https://api.goingelectric.de/chargepoints/`

### OpenChargeMap API (Fallback)
- Internationale API, weltweite Abdeckung
- Funktioniert auch ohne API-Key (dann niedrigeres Rate-Limit)
- Wird nur genutzt wenn GoingElectric keinen Treffer liefert oder nicht erreichbar ist
- Endpoint: `https://api.openchargemap.io/v3/poi/`

### Ablauf in `find_nearest()`
```
→ GoingElectric → Station gefunden? → zurückgeben ✅
→ GoingElectric → nichts/Fehler → OpenChargeMap → Station gefunden? → zurückgeben ✅
→ beide leer → None → App zeigt "Keine Station gefunden"
```

## Ressourcen

- [hyundai-kia-connect-api GitHub](https://github.com/Hyundai-Kia-Connect/hyundai_kia_connect_api)
- [hyundai-kia-connect-api PyPI](https://pypi.org/project/hyundai-kia-connect-api/)
- [OpenChargeMap API](https://openchargemap.org/site/develop/api)
- [GoingElectric API](https://www.goingelectric.de/stromtankstellen/api/)
- [Flutter speech_to_text](https://pub.dev/packages/speech_to_text)
- [Flutter geolocator](https://pub.dev/packages/geolocator)

## Changelog

### 2026-05-24

- **Projekt angelegt:** Stack, Ablauf und API-Optionen definiert
- **Stack-Entscheidung:** Flutter (nicht Vue 3) — Mobile, GPS, Spracheingabe
- **UI-Entscheidung:** Spracheingabe primär, Buttons als Fallback
- **Stationsauswahl:** Nächste in Fahrtrichtung (Heading-basiert), nicht nur nächste
- **Kia-API erfolgreich getestet:** Login, Fahrzeugdaten und `set_navigation` funktionieren
  - Fahrzeug: **EV6**
  - Vehicle-ID: `efdfdb39-5d6e-4555-8b58-4fcaeaf0fae9`
  - SOC beim Test: 68%
  - API-Methode: `vm.set_navigation(vehicle_id, [POIInfo(...)])` → liefert Message-ID
  - PIN wird für Remote-Befehle benötigt: `0208`
  - Passwort-Format: mit runden Klammern `(!!Ng020860!!)`
  - headless Login der Library funktioniert mit `encryptedPassword=false` + korrektem Passwort
  - Client-ID `fdc85c00-...` war zeitweise als "abusing request" geblockt — mit korrektem Passwort dennoch erfolgreich
- **Zwischenziel-Verhalten:** Noch offen — erst im Auto testbar wenn Flutter-App fertig

### 2026-05-24 (Fortsetzung)

- **speech_to_text integriert:**
  - `lib/services/network_matcher.dart` — reines Dart, priorisierte Keyword-Liste (mehrteilige Ausdrücke zuerst), 6 Unit-Tests ✅
  - `lib/services/speech_service.dart` — wraps `SpeechToText`, nutzt neue `SpeechListenOptions`-API (alt: direkte Parameter waren deprecated), lokalisiert auf `de_DE`, 10s Timeout, 3s Pause-Erkennung
  - `AndroidManifest.xml` — `RECORD_AUDIO`, `INTERNET`, `BLUETOOTH_CONNECT` (+ Legacy BT mit `maxSdkVersion=30`)
  - `flutter analyze` clean, alle Tests grün
- **Flutter-UI komplett implementiert:**
  - `lib/app_theme.dart` — Dark-Mode-Theme, Brandfarben (Shell-Gelb, IONITY-Lila, …), Netzwerk-Labels
  - `lib/models/station.dart` — Datenklasse mit `fromJson`
  - `lib/services/backend_service.dart` — HTTP-Client (findStation, sendToKia), `BackendException`, per `http.Client` mockbar
  - `lib/services/location_service.dart` — GPS + Heading via geolocator, Permission-Flow
  - `lib/screens/home_screen.dart` — State-Machine (idle/listening/processing/confirming/sendError), Dependency Injection
  - `lib/widgets/mic_button.dart` — 120px, Puls-Animation beim Hören, Cyan-Glow
  - `lib/widgets/network_chip.dart` — 72px Höhe, Brandfarbe, Kontrast-Check
  - `lib/widgets/station_card.dart` — Bottom-Overlay, sending/sent/error-Status, Tap-to-dismiss
  - `app/.env` — BACKEND_URL + BACKEND_API_KEY (in gitignore)
  - AndroidManifest: `ACCESS_FINE_LOCATION`, `ACCESS_COARSE_LOCATION` ergänzt
  - 11 Unit-Tests ✅, `flutter analyze` clean
- **Gesamter App-Flow implementiert:**
  1. Mic-Tipp (oder Netzwerk-Chip) → Sprache → Keyword-Matching → Netzwerk erkannt
  2. GPS-Position + Heading holen (geolocator)
  3. Backend `/find` → nächste Station in Fahrtrichtung
  4. Backend `/send` → Station an Kia senden
  5. StationCard: „Wird gesendet…" → „✓ Gesendet" → 3s auto-dismiss
  6. Fehler jeglicher Art: roter Toast → 4s auto-dismiss → zurück zu idle
- **Stand 2026-05-24:** Alle Tasks abgeschlossen. App ist bereit für ersten Geräte-Test.
- **Nächster Schritt:** `flutter run` auf Android-Gerät (USB-Debugging einschalten → `! flutter devices`), Backend lokal starten, Endpunkt in `app/.env` ggf. auf LAN-IP anpassen

- **Erster Geräte-Test erfolgreich:**
  - App auf Xiaomi Mi 11 installiert und gestartet ✅
  - Backend auf `192.168.2.45:8000` erreichbar ✅
  - Stationssuche funktioniert (Heading-Logik korrekt getestet) ✅
  - Kia hat beide Test-`set_navigation`-Aufrufe gespeichert → beim Autostart „Zwei Ziele erhalten" angezeigt ✅
  - „Ziele anzeigen" zeigte nichts → wahrscheinlich braucht Kia eine aktive Navigation für volle Funktion
  - Java 26 → Java 21 LTS nötig für Gradle-Kompatibilität (gotcha dokumentiert)
  - Android SDK musste manuell installiert werden: cmdline-tools → platform-tools → platforms;android-36 → ndk;28.2
- **Nächster Schritt:** LXC auf Proxmox → Backend permanent deployed für mobilen Einsatz im Auto

### 2026-05-24 — Backend Deployment abgeschlossen

- **LXC bereitgestellt** auf Proxmox (Debian Trixie)
- **Deployment-Weg:** GitHub (`softexceptions/KiaChargeNav`) → `git clone` auf LXC → `deploy/setup.sh`
- **Stack auf LXC:**
  - uvicorn (FastAPI) als systemd-Service (`kiachargenav.service`), startet automatisch beim Boot
  - nginx als Reverse Proxy auf Port 9081
  - Nginx Proxy Manager → HTTPS via Let's Encrypt → `https://kia.softexceptions.com`
  - UniFi Dream Machine Pro: Port 443 extern → NPM intern
- **DRY_RUN-Modus eingebaut:** `DRY_RUN=true` in `.env` → kein echter Kia-Send, App verhält sich identisch. Umschalten: `sed -i 's/DRY_RUN=true/DRY_RUN=false/' /opt/kiachargenav/backend/.env && systemctl restart kiachargenav`
- **App-URL:** `BACKEND_URL=https://kia.softexceptions.com` in `app/.env`
- **Test über Mobilfunk erfolgreich ✅** — App findet Stationen und sendet (Dry-Run) von extern
- **Nächster Schritt:** DRY_RUN=false setzen und ersten echten Send-Test während der Fahrt durchführen

### 2026-05-25 — App-Toggle + Fehlerbehandlung

- **App-seitiger Test/Live-Toggle eingebaut:**
  - Badge oben rechts im HomeScreen — einmal tippen schaltet um
  - Orange „Test": `/send` wird nicht aufgerufen — nur Stationssuche läuft
  - Grün „Live": sendet wirklich ans Auto
  - Zustand wird in `SharedPreferences` (`dry_run`) gespeichert → bleibt nach Neustart erhalten
  - Standard: Test-Modus aktiv
- **Für echten Betrieb müssen beide Schalter stimmen:**
  1. App-Toggle auf „Live"
  2. Server: `DRY_RUN=false` in `/opt/kiachargenav/backend/.env` + `systemctl restart kiachargenav`
- **Fehlerbehandlung lückenlos abgesichert:**
  - Flutter: `catch (_)` nach GPS-Fehler und nach `sendToKia` — kein stiller Absturz mehr
  - `LocationService`: `getCurrentPosition` in try/catch — wirft `LocationException` statt zu crashen
  - Backend `/find`: try/except um `find_nearest()` → 503 mit Meldung statt 500
  - `charging_service`: GoingElectric + OCM API-Fehler geben `None` zurück → Fallback greift
- **App-Icon:** Navigationspfeil + Blitz, Cyan auf Dark, generiert mit `flutter_launcher_icons` (alle 5 Android-Größen)

### 2026-05-25 — Auslands-Unterstützung

- **Anforderung ergänzt:** App muss im Ausland ohne manuelle Eingriffe funktionieren
- **API-Strategie:** Koordinaten-Check im Backend (DE-Bounding-Box) → GoingElectric primär in DE, OpenChargeMap primär im Ausland
- **Kia-API:** `Region.Europe` deckt alle EU-Länder ab — kein Umschalten nötig
- **Keyword-Matcher:** internationale Netzwerke ergänzt (ChargePoint, Allego, Tesla, Electrify America, Osprey)
- **Ladenetzwerke:** internationale Netze in die Netzwerk-Liste aufgenommen
