---
tags: [projekt/aktiv, flutter, dart, waermepumpe, dokumentation]
status: aktiv
date: 2026-06-03
---

# BWT – Brauchwasser-Wärmepumpe App

## Ziel

Flutter/Dart-App als digitale Bedienungsanleitung für Brauchwasser-Wärmepumpen.
Primärnutzung: Jemand steht im Keller vor der Pumpe und braucht schnell Hilfe.
Sekundär: Herstellern präsentierbar (ecodesign, Wolf).

## Geräte

| Hersteller | Modell | Volumen | COP | Kältemittel |
|-----------|--------|---------|-----|------------|
| ecodesign | ED 300 WT | 296 l | 3,46 | R134a |
| Wolf | FHS-HE | 258 l | 3,15 | R290 |

Beide Geräte sind **baugleich**: identische Firmware, Menüstruktur, Fehlercodes.

## Quelldokumentation

- `/home/norbert/Code/bwp/ecodesign Brauchwasserwaermepumpe ED 300 WT Installations- und Betriebsanleitung.pdf`
- `/home/norbert/Code/bwp/Wolf-Warmwasser-Waermepumpe-FHS-HE-Betriebsanleitung.pdf`

## Technischer Stack

| Bereich | Entscheidung |
|--------|------------|
| Framework | Flutter 3.44 / Dart 3.12 |
| Architektur | Clean Architecture + SOLID |
| State Management | Riverpod 3.x (NotifierProvider) |
| DI | get_it 8.x |
| Navigation | go_router 14.x |
| OCR | google_mlkit_text_recognition (offline) |
| Live-OCR | Kamera-Stream (`startImageStream`, NV21) + Mehrheits-Voting über die interpretierte Bedeutung |
| Sprache | Deutsch (UI-Texte hartcodiert); `flutter_localizations` nur für die Lokalisierung der Flutter-eigenen Widgets |
| Plattform | Android (Release-Build mit R8/ProGuard) |

## Architektur

```
lib/
├── core/         # DI, Theme, i18n, Router
├── domain/       # Models, Repository-Interfaces, Use Cases (reines Dart)
├── data/         # Repository-Implementierungen, hardcodierte Gerätedaten
└── presentation/ # Screens, Widgets, Riverpod Providers
```

## Implementierte Features

1. **Geräteauswahl** – ecodesign ED 300 WT oder Wolf FHS-HE
2. **Home** – Grid + Häufige Situationen (Kein Warmwasser, Fehler im Display, Einstellung ändern)
3. **Display auslesen** – **Live-Erkennung ohne Auslöser:** Telefon ruhig vors Display halten, der Kamera-Stream wird kontinuierlich gelesen; sobald ein Wert über mehrere Frames stabil ist, befüllt er automatisch das Textfeld. Manuelles Textfeld auf demselben Bildschirm als Korrektur/Fallback. Auswertung per Fuzzy-Matching gegen das geschlossene Display-Vokabular
4. **Menü-Browser** – 37 Menüpunkte (Haupt + Service + Wolf-Extras) mit Erklärung, Wertebereich, Schritt-für-Schritt
5. **Schnell-Hilfe** – 6 Szenarien: Kein Warmwasser, Wasser zu kalt, Grüne LED blinkt, Display zeigt 5 0 0, Legionellen-Warnung, Display zeigt nichts
6. **Volltextsuche** – über alle Inhalte inkl. funktionierende Vorschlags-Chips
7. **OCR-Auswertung (`AnalyzeDisplayUseCase`)** – zwei Stufen statt früherer Zeichenersetzung:
   - **Fuzzy-Label-Matching (#1):** Levenshtein-Distanz gegen alle 37 Menünamen, verglichen auf ganzen Wort-Tokens (Fenster passender Länge), Schwelle 0,34. Fängt OCR-Fehler ab, die feste Regeln nicht kennen (z. B. „Meldun9", „Meldunq", „Heldung", „T 50ll"). Bei zu großer Distanz: ehrlich kein Treffer.
   - **Erwartungsgesteuerte Wert-Korrektur (#3):** Das erkannte Label bestimmt den erwarteten Werttyp. Numerisch → Buchstabe-zu-Ziffer (S0→50, 6O→60) + Einheit aus dem Bereich; Enum (Bereich mit „/") → Fuzzy gegen erlaubte Werte (EIH→EIN, 8oost→Boost, 6AS→GAS). Bedeutungstragende Ziffern bleiben respektiert (EC LS2 ≠ EC LS1), negatives Vorzeichen erhalten.
8. **Live-Erkennung mit Stabilitäts-Voting (`stable_text_voter.dart`)** – statt eines Foto-Auslösers (der beim Antippen Verwacklung erzeugt) läuft der Kamera-Stream kontinuierlich durch ML Kit. Der `StableTextVoter` sammelt die letzten 10 Lesungen und übernimmt einen Wert erst ab 5 Stimmen. **Abgestimmt wird über die interpretierte Bedeutung** (`AnalyzeDisplayUseCase` → `Menüpunkt|Wert`), nicht über den rohen OCR-Text – der zappelt frame-zu-frame (`T Wasser` vs `T lWasser`) und käme sonst nie auf eine Mehrheit; Müll-Frames (`T`, leer) liefern keinen Menüpunkt und werden verworfen. Voter ist rein/testbar (keyOf injizierbar); Kamera/ML-Kit-Integration im Widget (`error_scanner_widget.dart`, `startImageStream` + NV21 + Rotations-Kompensation, App-Lifecycle-sicher). Löst zugleich Verwacklung **und** überstimmt sporadische `0`↔`9`-Fehler.
9. **Taschenlampe** – Toggle im Kamera-View (für dunklen Keller)
10. **App-Icon** – custom (Boiler + Wärmepumpen-Pfeil, App-Farben)
11. **Release-Build** – schneller Start, R8-Minifizierung, ProGuard-Regeln für ML Kit

## Daten

- **Fehlercodes**: 9 Codes (1–9, 10, 11) — Codes 10+11 unklar wie am Display dargestellt (nur 3 Einzel-Ziffernstellen vorhanden)
- **Fehler-Notation**: Im Handbuch steht „5/15" = LED 5 + LED 15 (Störungs-LED) auf der Platine — NICHT „5 von 15". Display zeigt „5 0 0".
- **Menüpunkte**: 21 Hauptmenü + 10 Service + 6 Wolf-Extras = 37 gesamt

## Design

- Material Design 3
- Farben: Tiefes Blau `#1A4A7A` + Orange `#E87422`
- Große Schrift (min. 16 sp) — keller-tauglich
- Dark Mode via Material 3

## Projektpfad

`/home/norbert/Code/bwp/bwt/`

## Verteilung

- **Firebase App Distribution** eingerichtet
- Projekt: `bwt-app-1cc0c`
- App-ID: `1:87344533521:android:5fb4e88aeec4f9254161b0`
- Tester hinzufügen + APK verteilen: siehe Auto-Memory `reference_firebase_bwt.md`

## Navigation (go_router)

**Flache Top-Level-Routen** statt verschachtelter Sub-Routen: `/device-selection`, `/home`, `/menu`, `/menu-item/:itemId`, `/error-lookup`, `/quick-help`, `/search`. Grund: Bei verschachtelten Routen (`/home/menu/:id`) baute `context.push()` aus einem Geschwister-Zweig (z. B. Suche) die ganze URL-Kette auf — der nie geöffnete Bedienmenü-Screen landete im Stack, Zurück wurde unvorhersehbar. Flach ⇒ jeder `push` fügt genau eine Seite hinzu.

**Regeln:**
- Vorwärts/tiefer: `context.push()`. Echte Resets (Geräteauswahl ↔ Home, Gerätewechsel): `context.go()`.
- **Such-Zurück-Verhalten** (gewünschtes UX): Aus einem Treffer führt der erste Zurück zurück zu den **Suchvorschlägen** (leeres Feld), erst der nächste Zurück ins Hauptmenü. Zwei Mechanismen, beide enden nach einem Zurück bei den Vorschlägen:
  - Menüpunkt-Treffer öffnet Detail-Screen → `push(...).then((_) => Suchfeld leeren)`.
  - Fehler-Treffer wird als `ErrorCard` **inline** in der Trefferliste gezeigt (kein Detail-Screen) → `PopScope` am Such-Screen fängt Zurück ab und leert bei aktivem Begriff zuerst das Feld.
- Such-Query liegt in einem globalen `NotifierProvider` (kein autoDispose) → bleibt beim Navigieren erhalten; bewusstes Leeren steuert die Rückkehr zu den Vorschlägen.

**Wichtig fürs Debugging:** Widget-Tests mit `tester.pageBack()` können trügerisch grün sein (poppt anders als der echte Pfeil/Geste am Gerät). Echtes Stack-Verhalten am Gerät mit `NavigatorObserver` + `debugPrint` über `adb logcat -s flutter:V` verifizieren (`debugLogDiagnostics` nutzt `dart:developer log()` und erscheint NICHT im logcat).

## ▶ Hier weiter (Stand 2026-06-02 Abend)

**Offener Akzeptanztest – bewusst noch NICHT committet** (Norbert committet erst nach bestandenem Test). Alle Änderungen liegen uncommittet im Working Tree, letzter Commit unverändert `c5aaa62`. Unit-Tests grün (74). Finales **Release-APK ist auf dem Poco F3 installiert**.

Morgen zu tun:
1. **Korrektur-Stepper gegentesten** (geht oben, ohne Keller): „Display auslesen" → Textfeld `T Soll 59` → mit `−` auf `50 °C`, Interpretation muss live mitgehen.
2. **OCR im Keller final prüfen**: „Kein Wert" weg? Werte stimmen bzw. schnell per Stepper korrigierbar? Phantom-„Meldung 9" weg?
3. Wenn Test bestanden: **committen** – Working Tree mischt mehrere Einheiten, sauber aufteilen: `chore:` i18n-Entfernung · `feat:` Live-OCR+Voting · `fix:` OCR-Wertkorrektur (Achse A+B) + Korrektur-Stepper. (Nichts pushen ohne Ansage.)

## Status (Stand 2026-06-02)

- [x] Projekt-Setup
- [x] **i18n-Gerüst entfernt** (2026-06-02): `AppLocalizations`/ARB/`generate:true` waren nie eingebunden (Delegate nicht registriert, `.of()` nirgends genutzt) – ~590 Zeilen toter Code raus. App ist bewusst deutsch-only (`supportedLocales: [de]`). Falls EN später nötig (Hersteller): sauber neu via gen-l10n aufsetzen
- [x] Domain Layer (TDD)
- [x] Data Layer (37 Menüpunkte, 9 Fehlercodes)
- [x] DI + Navigation + Theme
- [x] Alle Screens implementiert
- [x] OCR-Integration mit Display-Auswertung
- [x] OCR-Auswertung überarbeitet: Fuzzy-Label-Matching (#1) + erwartungsgesteuerte Wert-Korrektur (#3), per TDD (17 neue Tests)
- [x] Overflow auf Geräteauswahl-Screen behoben (LayoutBuilder + SingleChildScrollView + ConstrainedBox + IntrinsicHeight)
- [x] Navigations- & Such-Zurück-Verhalten korrigiert (mehrschichtig, siehe Abschnitt „Navigation" unten). Regressionstest `navigation_test.dart`. Technische Details in Auto-Memory `feedback_technical_gotchas.md`
- [x] **Bild-Aufbereitung erprobt und wieder verworfen** (Feldtest 2026-06-02): Grün-Kanal/Blur/Binarisierung verschlechterte die rohe ML-Kit-OCR drastisch (oft 0 erkannte Zeichen) statt sie zu verbessern; `image`-Dependency + `display_preprocessor.dart` entfernt. **Lehre:** schwarz-auf-grün ergibt in der Luminanz (Y≈0,59·G) bereits idealen Kontrast – die rohe OCR ist top, Rest-Verwechslungen gehören auf die Textebene. Belegt durch 8 reale Feld-Rohausgaben als Regressionstests im `AnalyzeDisplayUseCase` (alle grün, inkl. `T Legio Hus`→AUS fuzzy)
- [x] **Live-Erkennung statt Foto-Auslöser** (`stable_text_voter.dart` + Stream-Umbau, `startImageStream`/NV21): behebt Verwacklung; Mehrheits-Voting über die interpretierte Bedeutung überstimmt sporadische `0`↔`9`. App-Lifecycle-sicher (Kamera bei Hintergrund freigeben/neu aufbauen). Suite gesamt 65 grün
- [x] Release-Build lauffähig auf Poco F3
- [x] Firebase App Distribution eingerichtet
- [x] **OCR-Fehler „Achse A" behoben** (Feldtest-Beschwerde, 2026-06-02): zwei systematische Fehler auf der Textebene per TDD beseitigt (4 neue Tests, Suite gesamt 69 grün):
  - **Gradzeichen verfälschte die Zahl:** `°` als `o` fehlgelesen ⇒ `_letterToDigit` machte `50°C` → `500 °C`. Fix: Plausibilitätsgrenze aus `item.range` (z. B. T Soll max 62 °C, Temperatur-Default 99) – übersteigt der Wert die Obergrenze, wird die fälschlich angehängte Stelle von rechts gekürzt.
  - **Phantom-Fehler „Meldung 9":** das `g` in „Meldung" als separate `9` gelesen ⇒ Fehlercode 9 (Anode) ohne realen Fehler. Fix: Fehlercodes nur noch, wenn der Wert die echte **dreistellige** Meldungsform hat (≥ 3 Ziffern, „5 0 0"). Eine einzelne nachlaufende Ziffer wird verworfen; echte „9 0 0" bleibt erkannt.
- [x] **OCR „Achse B" behoben** (Feldtest 2026-06-02, datenbasiert): 94 reale Frame-Logs vom Poco F3 zeigten die wahre Hauptursache von „Kein Wert" – ML Kit liefert den Wert je nach Display-Layout mal **vor**, mal **nach** dem Label (`50 C\nT soll`). Der Parser suchte ihn nur *nach* dem Label. **Fix (Textebene, kein Bounding-Box-Umbau nötig):** Wert = alle Tokens außerhalb des Label-Fensters (beidseitig), erwartungsgesteuerte Korrektur pickt Zahl/Enum heraus; zusätzlich bei numerischer Erwartung ohne Ziffer ehrlich `null` statt Müll-Wert (`T sol1\nT` → kein „t"). 5 neue Tests aus echten Log-Strings, Suite gesamt 74 grün. **Lehre:** „erst messen" – das vermeintliche ML-Kit-Problem war zur Hälfte eine falsche Parser-Annahme.
- [x] **0↔9-Wurzel diagnostiziert + entschieden** (Display-Fotos 2026-06-02): Mess-Version speicherte bei jedem Treffer ein Display-Foto (`takePicture` → `getExternalStorageDirectory`, per `adb pull` gezogen; danach Mess-Gerüst inkl. `path_provider` wieder entfernt). Befund eindeutig: Das Display nutzt eine **Null mit Mittelpunkt** („dotted zero"), die ML Kits generisches Font-Modell systematisch als **9** liest (50→59, mehrheitlich → Voting machtlos). Ebenso ist das **`g` in „Meldung" ≈ `9`**. Das ist **Glyph-Mehrdeutigkeit, kein Auflösungsproblem** (im scharfen Foto ist die 0 für Menschen eindeutig) → ROI/Zoom hülfe kaum.
  - **Entscheidung (statt OCR zu erzwingen): Korrektur-UX.** OCR liefert den Vorschlag, der Nutzer berichtigt ihn mit großen +/−-Steppern (`error_lookup_screen.dart`, `_correctionBar`) – läuft durch dieselbe `analyze()`-Pipeline (`<Label> <Wert>`). Im Keller-Kontext (schnelle Hilfe, kein Messgerät) ausreichend und ehrlich. Editierbares Textfeld bleibt zusätzlich.
  - Offen falls je höchste Genauigkeit gefordert: Template-Matching auf den festen Dot-Matrix-Zeichensatz (baugleiche Geräte) – bewusst zurückgestellt.
- [ ] Korrektur-Stepper am Gerät gegentesten (UI, ohne Keller möglich)
- [ ] App gründlich testen vor Hersteller-Kontakt
- [ ] App gründlich testen vor Hersteller-Kontakt
- [ ] Hersteller kontaktieren (ecodesign / Wolf)
