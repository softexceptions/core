---
tags: [projekt/aktiv, victron, node-red, energie, homeassistant, mqtt, pv]
status: aktiv
date: 2026-07-05
updated: 2026-08-23
---

# Victron Node-RED

Node-RED-Flows für die Victron-Anlage, Repo: `~/Code/victron_node_red`.

> [!info] Umbau läuft (Etappe 1 von 4, Stand 23.08.2026)
> Diese Notiz wird in einen Ordner mit Themennotizen und einem Journal aufgeteilt.
> **Erledigt:** Frontmatter ergänzt, projektfremde Themen ausgelagert.
> **Offen:** Journal abtrennen, Themennotizen destillieren, MOC anlegen.

**Ausgelagert am 23.08.2026:**
- [[Brauchwasser-Wärmepumpe]] — Stillstandsverlust, Stufentest, Zirkulation, Pumpenrichtung
- [[homelab-monitoring]] — BW-WP-Datenpfad (InfluxDB/Grafana), Zigbee-Dosen- und Coordinator-Störungen
- [[homelab-infrastruktur]] — UniFi-Firmware-Vorfall 18.08., UDM-SSH-Runbook, HA von außen (NPM/Alexa)

## Setup
- Node-RED läuft in einem **Proxmox-LXC-Container** (nicht auf dem Cerbo GX, um ihn zu entlasten).
- Victron-Nodes sprechen den Cerbo per **D-Bus über TCP** an: auf dem GX `InsecureDbusOverTcp=1`, im Container `NODE_RED_DBUS_ADDRESS=<cerbo-ip>:78`.
- Hardware: Cerbo GX, Gridmeter VM-3P75CT.
- Daneben existiert ein **Venus-MQTT-Keepalive-Setup in HA** (Switch + Automationen), das **nur bei Dashboard-Aufruf** aktiviert wird und danach automatisch stoppt — der Venus-MQTT-Pfad liefert also absichtlich nicht dauerhaft. Für kontinuierliche Daten (z.B. Energie-Dashboard) daher immer den D-Bus-über-TCP-Pfad via Node-RED nutzen, nie Venus-MQTT.

## Wartung: Anlage sicher ab- und zuschalten

> ⚠️ **Sicherheit:** Arbeiten an der festen Elektroinstallation gehören in die Hand einer Elektrofachkraft. Die **5 Sicherheitsregeln** beachten: freischalten → gegen Wiedereinschalten sichern → **Spannungsfreiheit messen** → erden/kurzschließen → benachbarte Teile abdecken. **PV-Module/-Strings führen bei Tageslicht IMMER Spannung** — ein „ausgeschalteter" Wechselrichter/MPPT ist DC-seitig NICHT spannungsfrei. Ein Schalter erzeugt nur eine **Trennstelle**; spannungsfrei ist ausschließlich der allseitig (von PV, Batterie, Netz) getrennte Abschnitt. Am Arbeitsort immer **messen**, nie aus der Schalterstellung schließen.

**Anlagen-Topologie (2026-07-07 bestätigt):** Fronius Symo 17.5-3-M am **AC-IN** der 3× MultiPlus-II (→ netzabhängig, kein Insel-/Frequenzregelbetrieb, kein Backup-PV); 2× SmartSolar MPPT DC-gekoppelt an der Batterie; Batterie mit Batrium-BMS/Schütz; Cerbo GX über **separates 24-V-Netzteil an 230 V AC** (geht NICHT mit dem Batterie-Trennschalter aus, muss separat stromlos); Node-RED im Proxmox-LXC (stromunabhängig von der Anlage, hängt per D-Bus-TCP am Cerbo).

**Grundprinzip:** Abschalten = Erzeuger zuerst, Speicher (Batterie) zuletzt. Einschalten = umgekehrt. Vor jeder mechanischen DC-Trennung erst **lastfrei** machen (Software-Aus bzw. AC-Trennung) → kein DC-Lichtbogen (DC hat keinen Nulldurchgang!). **Fronius zuerst aus, zuletzt an.**

### Abschalten (für Wartung)
1. **Node-RED stoppen** im LXC (`systemctl stop nodered`) — verhindert false-reset-Statistiken beim späteren Reconnect.
2. **Haus-AC-Lasten** wegnehmen (Sicherungsautomaten einzeln aus).
3. **Fronius AC freischalten** (RCD/LS des Fronius-Abgangs, **allpolig**) → Anti-Islanding, Fronius stellt ab, DC-Seite wird lastfrei. Gegen Wiedereinschalten sichern.
4. **Fronius DC-Trennschalter** öffnen (jetzt lastfrei → lichtbogenfrei).
5. **PV-Strings im GAK** Richtung Fronius trennen (lastfrei; MC4-Stecker nie unter Last!).
6. **MultiPlus (3er-Verbund) ausschalten** — zentral über den **Cerbo GX** (Remote-Schalter im Gerätemenü). Die 3 Multi sind EIN VE.Bus-System; **nicht** einzeln oder „nur den Master" am Frontschalter schalten → VE.Bus-Fehler (Error 17/3). **Frontschalter aller 3 bleiben auf Stellung `I` (On)** — nicht `II` (Charger only), nicht `0` (Off). **Vor** dem Netz, sonst unnötiger Inselbetrieb-Wechsel. (Remote-Off = nur Standby; spannungsfrei erst durch Netz-/Batterie-Trennung.)
7. **Netz-Hauptschalter (AC-In)** freischalten — danach ist die AC-In-Schiene tot.
8. **MPPT 1 + 2 in VictronConnect auf „Aus"** (kein Lade-/Entladestrom). Nur falls an der PV-DC-Seite gearbeitet wird: **PV-Trenner der MPPT** öffnen, danach **Batterieseite** — Reihenfolge **PV → Batterie** einhalten (SmartSolar nie PV-only).
9. **Batterie** trennen (DC-Hauptschalter / Batrium-Schütz).
10. **Cerbo GX** separat stromlos machen (LS des 230-V-Netzteil-Kreises).
11. Vor Arbeiten: an jeder Arbeitsstelle **Spannungsfreiheit messen**. Fronius-Gehäuse erst nach **5 min Kondensator-Entladezeit** öffnen (klassische Symo-Serie; genaue Zeit steht auf dem Gerät). Modulseite bleibt bei Licht heiß.

### Einschalten (nach Wartung) — umgekehrt
1. **Batterie** zuschalten (DC-Hauptschalter / Batrium-Schütz).
2. **Cerbo GX** einschalten, sobald sein 230-V-Kreis Spannung hat, warten bis vollständig hochgefahren (⚠️ je nachdem, woher die 230 V kommen, ggf. erst nach Netz/MultiPlus verfügbar — vor Ort prüfen).
3. **MPPT** DC verbinden (**Batterie → dann PV**), in VictronConnect wieder auf „Ein".
4. **Netz-Hauptschalter (AC-In)** zuschalten.
5. **MultiPlus** einschalten — zentral über den **Cerbo GX** (Remote On), Frontschalter aller 3 auf Stellung `I` (On). Synchronisieren direkt aufs anliegende Netz (kein Insel-Zwischenschritt), warten bis stabil.
6. **Fronius** DC zuschalten (Module → DC-Trennschalter), dann **AC (RCD/LS)** → er synchronisiert auf die anliegende AC-In-Spannung. Fronius kommt zuletzt.
7. **Haus-AC-Lasten** wieder zuschalten.
8. **Warten**, bis Cerbo/VRM alle Geräte + Werte zeigt und das System stabil ist.
9. **Node-RED starten** (`systemctl start nodered`) — erst jetzt, damit keine Teilwerte in die retained-Topics/Statistik laufen.

**Vor Ort noch zu verifizieren:** exakte Bezeichnung/Reihenfolge der physischen Trennschalter; woher die 230 V für das Cerbo-Netzteil kommen (eigener Hauskreis oder Anlagen-AC?); SmartSolar-Anschlussreihenfolge laut Handbuch. Quelle Entladezeit: Fronius Symo/Eco Bedienungsanleitung (5 min).

## Planung: Whole-home-Backup & Batterie-Ausbau (2026-07-07)

**Ziel:** Notstromfähigkeit für das ganze Haus — Hauslasten über den **AC-Out** der MultiPlus führen (aktuell hängt alles am AC-IN → kein Backup).

### Ist-Stand / Topologie (Auslegungsdaten)
- **3× MultiPlus-II 48/10000** (3-Phasen, EIN VE.Bus-System): je **8.000 W** Dauerleistung @25 °C → **~24 kW** Insel gesamt (bei 40 °C ~19–20 kW); **Transfer-Relais 100 A/Phase**; Wechselrichter-Ausgangsstrom 36 A/Phase.
- **Fronius Symo 17.5-3-M am AC-IN** → netzabhängig, **im Inselbetrieb NICHT verfügbar**.
- **2× SmartSolar MPPT 250/70** (~5 kWp, DC an der Batterie) → einzige PV-Quelle im Notstrom.
- **Batterie:** aktuell ~15 kWh (280 Ah/48 V), **im Ausbau auf ~30 kWh** (2. Pack à 15 kWh, parallel).
- **BMS:** **1× WatchMon Core + 2× CellMate K9** = EIN BMS für die Gesamtbatterie → ein CAN-BMS am Cerbo (DVCC sauber), ein UDP-Datenstrom fürs Monitoring. (Frühere Fehlannahme „zwei getrennte K7" verworfen.)

### Bewertung
- **Leistung reicht locker:** Die 3× 10000 tragen das ganze Haus inkl. 11-kW-Wallbox. Durchgangsstrom (100 A/Phase) liegt über jedem üblichen Hausanschluss, Inselleistung 24 kW ist komfortabel. → Hürden „Durchgangsstrom" und „Inselleistung" **entfallen**.
- **Fronius-Umzug auf AC-Out (für PV im Notstrom): aktuell NICHT möglich.** Grund = Batterie-Aufnahmefähigkeit: 17,5 kW ÷ 48 V ≈ **365 A** Ladestrom bei plötzlichem Lastabwurf; sicher bei 0,2–0,3 C → **~1.200–1.800 Ah ≈ 60–100 kWh** Batterie nötig. 30 kWh = ~0,65 C → **noch zu klein**. Schwelle für Umzug: **~60–100 kWh**.
- **Notstrom-Konzept jetzt:** Whole-home über AC-Out, **Fronius bleibt AC-IN**, PV im Notstrom nur ~5 kWp (MPPT), Autonomie = Batteriekapazität (~30 kWh) + MPPT-Nachladung. → **Lastabwurf für Wallbox/Großverbraucher** im Inselbetrieb einplanen (sonst Batterie in <1 h leer).

### Batterie-Ausbaupfad
- Jetzt +15 kWh → 30 kWh: verdoppelte Notstrom-Autonomie, niedrigere C-Raten, längere Lebensdauer.
- Erweiterung = **weitere CellMate K9 am selben WatchMon Core** (bleibt EIN BMS). Nach dem Ausbau: **Gesamtkapazität im Core neu einstellen** (SoC/Coulomb), **Shunt misst Gesamtstrom** (Position!), gleicher SoC/Spannung beim Parallelverbinden, **Sicherung pro Pack**, Balancing über alle Zellen.
- Ab ~60–100 kWh wird der Fronius-Umzug auf AC-Out möglich → volle 22,5 kWp PV auch im Notstrom.

### Offene Punkte für die Elektrofachkraft / Fachplanung
- Verteilerumbau: Netz → AC-In → AC-Out → Hausverteilung, mit **Wartungs-Bypass** (Haus direkt aufs Netz bei Multi-Wartung).
- **N-PE-Erdungsrelais-Konzept** für FI-Schutz im Inselbetrieb (kritisch! internes Erdungsrelais der MultiPlus, sauberes TN-System).
- **Lastmanagement/Lastabwurf** für Wallbox + Großverbraucher im Notstrom.
- Symmetrische Lastverteilung auf die 3 Phasen; Hausanschluss-Absicherung bestätigen (bei 100 A Transfer unkritisch).

### Monitoring-Konsequenz
Bleibt einfach: **ein WatchMon Core = ein UDP-Datenstrom** → Batrium-Alarm-Flow + HA-Sensoren unverändert, nur Kapazitätsbezug (15→30 kWh) anpassen, wo hinterlegt.

**Quellen:** [MultiPlus-II Datenblatt (48/10000: 8000 W, Transfer 100 A)](https://www.victronenergy.com/upload/documents/Datasheet-MultiPlus-II-inverter-charger-EN-.pdf) · [External Transfer Switch Applikation](https://www.victronenergy.com/upload/documents/MultiPlus-II_External_Transfer_Switch_application/179843-MultiPlus-II_external_transfer_switch_application-pdf-en.pdf) · [Batrium Victron/DVCC Wiki](https://wiki.batrium.com/integration/victron)

## Speicher-Wirkungsgrad: korrekte Kennzahl (2026-07-07)

**Anlass:** `sensor.speicher_wirkungsgrad_monat` (HA-Template `AC-Abgabe_Monat / Batterie-geladen_Monat`) zeigte 184 %. Zwei Fehler: (1) Monats-`utility_meter`-Reset → SoC-Carryover über die Monatsgrenze; (2) **prinzipiell**: Der Zähler `AC-Abgabe` (vebus `InverterToAcIn1 + InverterToAcOut`) zählt **jede** DC→AC-Wandlung — bei DC-gekoppelter MPPT-PV also auch **PV-Durchleitung**, die nie durch die Batterie lief. Deshalb kann der Quotient >100 % werden → es ist kein Wirkungsgrad. (Der Fronius am AC-IN stört nicht, er läuft nie durch die vebus-Wandlung.)

**Korrekte Zerlegung:** `AC-Roundtrip = DC-Roundtrip × Wechselrichter-Entlade-Wirkungsgrad ≈ 96,9 % × ~94 % ≈ 91 %`.
- **DC-Roundtrip** = `sensor.batterie_wirkungsgrad` (entladen/geladen, Lifetime, gleicher Batrium-Shunt) = 96,9 % — war schon sauber.
- **Wechselrichter-Entlade-Wirkungsgrad** — neu gemessen, nur wenn **DC-PV (MPPT) ≈ 0 UND Batterie entlädt**: dann stammt die AC-Abgabe ausschließlich aus der Batterie (keine PV-Durchleitung). Physikalisch eindeutig, praktisch = Nachtstunden.

**Umsetzung (Node-RED, Flow „Victron Speicher-Wirkungsgrad (MQTT)", Live-ID `fa26bca6047459ed`, Variante B):**
- Datenquellen: Rohsignale **direkt vom D-Bus** via `victron-input-*` (autark, projektkonform) — `/History/DischargedEnergy` + `/History/ChargedEnergy` (`com.victronenergy.battery/512`), `/Dc/Pv/Power` + `/Dc/Battery/Power` (`com.victronenergy.system`), je mit nachgeschaltetem `change`-Node für ein eindeutiges Topic. Über MQTT kommt die **gehärtete** Summe `victron/multiplus/ac_out_energy` (AC-Abgabe, DC→AC) — sonst müsste man die vebus-Summe + Härtung duplizieren.
- Function `clspx0func000001` (1 Ausgang → 1 gemeinsamer MQTT-Out, leeres Topic) akkumuliert `accAc`/`accDc` (AC-Abgabe ÷ Batterie-Entladung), **gegatet** auf `Dc/Pv/Power < 30 W` UND `Dc/Battery/Power < −50 W` (entlädt). Tick alle 60 s. **Wichtig:** beide Zuwächse pro Tick mit `>= 0` summieren — NICHT auf `dDc>0` gaten (grober 0,1-kWh-Entladezähler würde die feine AC-Abgabe sonst unterzählen → war der 17-%-Bug). Akku persistiert über retained `victron/battery/_eff_acc`; Reset-Inject `clspx0reset00001` (Topic `reset`).
- Publiziert **erst ab >1 kWh** gesammelter Entlade-Energie → sonst `unknown`.
- **Zwei HA-Sensoren** via MQTT-Discovery:
  - `sensor.victron_speicher_wechselrichter_entlade_wirkungsgrad` (Entlade-Wandlung DC→AC; nachts gemessen, ~80–90 % — Nacht-Last niedrig → Eigenverbrauch drückt, daher unter Datenblatt).
  - `sensor.victron_speicher_ac_roundtrip_inkl_wandlung` — Anzeigename **„PV-Speicher-Nutzgrad (DC → AC)"** = DC-Roundtrip × Entlade-Wandlung, ~91 %. Entity-ID bewusst unverändert (nur Friendly-Name), damit die Gauge auf `victron-sys` nicht bricht.
- **Lade-/Netz-Roundtrip-Sensoren entfernt (2026-07-08):** AC→DC-Laden gibt es zwar (Fronius via MultiPlus bei Schlechtwetter — Cerbo → System-Setup → Ladekontrolle, Ladestrom 5 A → 60 A), ist aber **nicht sauber isolierbar**: tagsüber immer mit DC-gekoppeltem MPPT-Laden vermischt (`charged`-Zähler trennt die Quellen nicht), und ein sauberes MPPT=0-Fenster gäbe es nur beim seltenen Netz-Nachtladen. Statt schlecht zu messen → **Referenzwert** MultiPlus-Ladewandlung ~94 % → Fronius-geladener Roundtrip ≈ 0,94 × 0,97 × ~0,90 ≈ **82 %** (betrifft nur den Schlechtwetter-Anteil).
- Didaktik-Skizze: `docs/speicher-wirkungsgrad.excalidraw` — Lehrdiagramm „was man MISST, was man ABSCHÄTZT": Messgrenze (Batterie-Shunt), Verlust-Klassifikation (gemessen / schon im DC-Roundtrip / upstream MPPT / geschätzt Fronius) + Messtechnik-Lektion (Messgrenze kennen, nur Isolierbares messen, Rest begründen).
- **Reproduzierbarkeit (Generatoren im Repo, 2026-07-08):** Flow-JSON und Skizze werden aus Python-Generatoren erzeugt, nicht von Hand gepflegt:
  - `tools/gen_eff_flow.py` → schreibt `flows/victron-speicher-wirkungsgrad-mqtt.json` **und** `eff_deploy.json` (PUT-Payload, ins CWD). Live-Deploy: `curl -X PUT http://192.168.2.80:1880/flow/fa26bca6047459ed -H 'Content-Type: application/json' --data-binary @eff_deploy.json`. (`eff_deploy.json` ist nur temporär — nicht committen.)
  - `tools/gen_excalidraw.py` → schreibt `docs/speicher-wirkungsgrad.excalidraw`.
  - Also: Änderung immer im Generator machen, neu ausführen, dann deployen — nicht direkt in der JSON editieren.

**Warum nicht als HA-Template-Helper:** Erst versucht, scheiterte an zwei HA-Zicken — der Config-Flow verwirft den `availability`-Key (nicht im Schema), und ein Template-Sensor mit **leerem Render überspringt das Update** (behält den letzten Wert). Eine initial gerenderte 0,0 blieb dadurch kleben. In Node-RED/MQTT gilt „kein Publish = unknown" nativ → sauberer. Template-Helper wieder gelöscht.

**Caveat:** Gemessen wird nur bei niedriger Nacht-Last → spiegelt den Wechselrichter-Wirkungsgrad im typischen Entlade-Betrieb (Abend/Nacht), nicht den Peak bei Volllast.

**Erledigt:** Alter `sensor.speicher_wirkungsgrad_monat` (Fehlkennzahl, 184 %) am 2026-07-07 gelöscht — ersetzt durch den korrekten AC-Roundtrip. Falls ein Dashboard-Card ihn referenzierte, dort auf `sensor.victron_speicher_ac_roundtrip_inkl_wandlung` umstellen.

### Stand & Verifikation (2026-07-08) — ✅ VERIFIZIERT

**Bug gefixt (07.07. abends):** Sensoren zeigten 17 % statt ~90 %. Ursache: grober Entladezähler (`/History/DischargedEnergy`, 0,1-kWh-Stufen) vs. feiner AC-Abgabe-Zähler; Akkumulation gatete auf `dDc>0` → AC nur in Sprung-Minuten gezählt → unterzählt. **Fix:** beide Zuwächse pro Tick `>=0` summieren. Verschmutzter Akku per Reset genullt.

**Vereinfacht (08.07.):** Lade-/Netz-Roundtrip-Sensoren entfernt (AC→DC-Laden nicht isolierbar, s. o.) → es bleiben **zwei** Sensoren. Flow: Function `clspx0func000001` → gemeinsamer MQTT-Out `clspx0out_pub001` (leeres Topic), Tick `clspx0tick00001`, **Reset `clspx0reset00001`**. Persistenz retained `victron/battery/_eff_acc` (JSON accAc/accDc).

**Reset-Befehl:** `curl -X POST http://192.168.2.80:1880/inject/clspx0reset00001` (nullt Akku + löscht die 2 retained Efficiency-Topics).

**Verifikation abgeschlossen (2026-07-08):** Nach der Entlade-Nacht 07.→08.07. haben sich die Sensoren selbst überschrieben und zeigen jetzt:
- `victron_speicher_wechselrichter_entlade_wirkungsgrad` = **72,9 %**
- `victron_speicher_ac_roundtrip_inkl_wandlung` = **71,7 %** (= 72,9 % × 98,4 %, Verrechnung stimmt exakt)
- `batterie_wirkungsgrad` (DC-Roundtrip) = **98,4 %**

**5-Minuten-Gegenprüfung (Nachtfenster 23:00–06:25, PV=0):** Σ `victron_multiplus_ac_abgabe` = 3,62 kWh ÷ Σ `victron_batterie_entladen` = 4,90 kWh = **73,9 %** roh — trifft den Sensor (72,9 %) auf ~1 pp (Rest = Fensterbeginn erst 23:00 + 0,1-kWh-Quantisierung des Entladezählers). **→ Kein Bug, Wert ist real.** Der 17-%-Bug ist damit endgültig erledigt.

**Warum 73 % real sind (kein Defekt):** Reine Teillast-Physik. Verlust 4,90−3,62 = 1,28 kWh über ~7,4 h = **~173 W Dauerverlust** bei nur ~490 W AC-Nutzlast. Der 3×-MultiPlus-II-Verbund (30 kW Nenn) versorgt nachts <2 % seiner Nennlast; der last­unabhängige Eigenverbrauch (~75–90 W, 3 Geräte) + Teillast-Wandlung dominiert prozentual. Bei höherer Last steigt der η stark: ~2 kW → ~91 %, ~5 kW → ~93 % (fixer Overhead bleibt, wird prozentual klein). **Der Nacht-Wert ist der ungünstigste Betriebspunkt, nicht der Alltags-Nutzgrad.** Beantwortet nebenbei die alte Grundlast-Frage: ~173 W mittlerer Verlust > reiner Eigenverbrauch, enthält also Eigenverbrauch + Teillast-Wandlung.

**Gauge-Schwelle offen (Entscheidung von Norbert ausstehend):** `victron-sys`-Gauge „PV-Speicher-Nutzgrad" steht auf grün ≥ 88 → zeigt beim realen ~72-%-Nachtwert dauerhaft rot/gelb. Optionen: (a) Schwelle an Nacht-Realität senken (~70), (b) Sensor erst ab höherer Entladeleistung gaten (näher ans Datenblatt), (c) unverändert lassen (Farbe kosmetisch). Norbert hat das im Kontext der Wärme-/Arbitrage-Diskussion (s. u.) zurückgestellt.

## Wärmewende & Speicher-Arbitrage (2026-07-08) — Wirtschaftlichkeit

**Kontext:** Norbert will weg vom Öl, hat zwei Split-Klimageräte (bisher nur Sommer-Kühlbetrieb gelaufen, Winter-Heizbetrieb noch ungetestet) und überlegt Speicher-Arbitrage mit dynamischem Tarif. „Arbitrage" = Gewinn aus Preisdifferenz desselben Guts; hier über die **Zeit** (nachts billig laden, tagsüber teuren Netzbezug vermeiden). Der Speicher ist das Transportmittel, der Wirkungsgrad der „Fahrpreis": lohnt nur, wenn **η_roundtrip > p_nacht / p_tag**.

**Haus-Grundlast (rekonstruiert aus Batterie-Entladung, Netzbezug nachts ~0 → ESS deckt alles):**
- Absolute Grundlast (tiefe Nacht): **~250–300 W AC**; mittlere Nachtlast ~450–550 W; Abendlast (Kochen) ~1–2 kW.
- Konsequenz für den Ausspeicher-η: Grundlast-Stunden ~74–78 %, Abend/höhere Last ~88–91 %. Energie-gewichtet real ~82–86 %, außer in reinen Grundlast-Nächten (dann die gemessenen ~74 %).

**Split-Klima-Verbrauch (Zigbee-Steckdosen, Sommer-Kühlbetrieb 30 d):**
- SK Arbeitszimmer/Küche (`sensor.0xa085e3fffebc4574_power`): 219,2 kWh Lebenszeit, Peak ~750–1000 W (max 1564).
- SK Wohnzimmer (`sensor.0xa085e3fffebd16b8_power`): 99,6 kWh, Peak ~1280–1340 W (max 1896).
- Installierte el. Spitze zusammen ~2,3–2,9 kW. **Winter-Heizbetrieb liegt damit im GUTEN Wechselrichter-η-Bereich** (~88–91 %) — die Heizlast lässt den fixen Teillast-Overhead prozentual verschwinden. Achtung: Kühl-Daten, Heizen bei Kälte zieht anders (COP sinkt, Abtauzyklen) → reale JAZ erst diesen Winter messbar.

**Öl vs. Wärmepumpe — Wärmepreis pro kWh (die faire Kennzahl):**
- Öl: 3.500 l/Jahr, Angebot 114,56 €/100 l = **4.009,65 €**. Heizöl ~10 kWh/l → 35.000 kWh Brennstoff; bei Kessel 85 % = 29.750 kWh Nutzwärme → **~13,5 ct/kWh Wärme** (Spanne 12,1 bei Brennwert / 14,3 bei 80 %).
- Split-WP: Wärmepreis = Strompreis ÷ JAZ. Bei Normaltarif **0,3494 €/kWh** (Netzbetrieb Hirschberg): JAZ 3,0 → 11,6 ct; **JAZ 3,5 → 10,0 ct**; JAZ 4,0 → 8,7 ct.
- **Kernbefund: Schon mit dem heutigen Normaltarif OHNE jeden Trick schlägt die Split-WP (JAZ ≥ 3,5 → ~10 ct) das Öl (~13,5 ct).** Ersparnis pro kWh-Preis (bei gleichbleibender Wärmemenge): JAZ 3,0 −545 €, JAZ 3,5 −1.040 €, JAZ 4,0 −1.411 €. ~~**Realistische Gesamtersparnis aber höher (~2.000–2.800 €/Jahr, s. Erkenntnis 3), weil die Splits mehr Öl-Menge ersetzen als die reine Preisdifferenz-Rechnung annimmt** — die kWh-Preis-Ersparnis ist nur der Effekt auf den ersetzten Anteil.~~
  > ❌ **Rechenfehler, korrigiert 13.08.2026.** Die „2.000–2.800 €" sind die **Brutto-Öl-Einsparung ohne Gegenrechnung des Stroms**, den die Splits für dieselbe Wärme verbrauchen. Beispiel 70 % Deckung: Öl sinkt um 2.807 €, aber es kommen 5.950 kWh Strom (2.079 €) dazu → **netto ~730 €**, nicht 2.800 €. Die Preisdifferenz-Rechnung eine Zeile darüber (−545/−1.040/−1.411 €) ist dagegen **richtig**. ⚠️ Auch die Preisbasis ist überholt (114,56 €/100 l). → aktueller Stand im Abschnitt **„Öl vs. Wärmepumpe: Stand 08/2026"** am Dateiende.

**Drei Erkenntnisse:**
1. **Der große Hebel ist der WP-Wechsel (Faktor JAZ), NICHT die Arbitrage.** Die Batterie-Arbitrage bringt in der Hochpreisregion wenig: Roundtrip 0,83 = 20 % Aufschlag frisst die geringe Tag/Nacht-Spreizung fast auf (Nacht 25 ct über Speicher = 8,6 ct Wärme ≈ direkter Normaltarif 10 ct). Lohnt erst ab Nachttarif Richtung 20 ct.
2. **Für die WP schlägt thermische Speicherung die Batterie:** direkt im billigen dynamischen Fenster heizen (Gebäudemasse/Puffer, ~100 % „Roundtrip") statt Umweg über die 83-%-Batterie. Batterie → PV-Eigenverbrauch + nicht-verschiebbare Lasten.
3. **Mengen-Deckung — Leistung ≠ Jahresarbeit (korrigiert 2026-07-08):** Die frühere „35–52 %"-Zahl war eine reine Leistung-×-Volllaststunden-Rechnung und damit zu pessimistisch. Entscheidend ist die **Jahres-Wärmearbeit** (Heizgradtag-Verteilung): Die milde Übergangszeit + Durchschnitts-Wintertage dominieren mengenmäßig, die wenigen Frost-Tage tragen wenig zur Gesamtmenge bei. Nach **Bivalenz-Prinzip** deckt eine WP mit ~50 % der Spitzenlast typisch **75–85 % der Jahres-Heizarbeit**; der Ölkessel bleibt nur Spitzenlast an den kältesten Tagen. Öl-Rest realistisch: 70 % Deckung → ~1.050 l (−2.807 €), 80 % → ~700 l (−3.208 €).

**Rahmenbedingungen:** Dynamischer Tarif noch nicht möglich (Smartmeter fehlt → dann Modul 3 / dynamische Netztarife); dynamische Preise in der Region generell hoch. Netzbetreiber: Netzbetrieb Hirschberg. Bezug 0,3494 €/kWh, Einspeisung 0,07382 €/kWh.

**Warmwasser bereits elektrifiziert (2026-07-08 bestätigt):** Die **Brauchwasser-Wärmepumpe läuft schon** → Warmwasser hängt nicht mehr am Öl. Damit greift der **Sommer-Effekt** bereits: Öl-WW im Sommer war besonders verschwenderisch (Kessel taktet nur für WW, Nutzungsgrad fällt von ~85 % auf ~55–65 % durch Bereitschafts-/Stillstandsverluste; grob ~320–510 l/Jahr = 370–580 € allein fürs WW). Der Ölkessel kann im Sommer komplett aus (Sommerabschaltung). **Offen/zu klären:** Enthielt der 3.500-l-Wert noch WW (dann ist der Sommer-Anteil eine bereits realisierte Einsparung) oder ist er schon reiner Heizbedarf? → per Tankablesung nach Saison bzw. Beobachtung, ob diesen Sommer überhaupt noch Öl fließt.

**Offene Messung (diesen Winter selbst schließbar):**
- **Reale JAZ der Splits im Heizbetrieb** — wichtigste Zahl, die ganze Rechnung hängt daran (Zigbee-Steckdosen `sensor.0xa085e3fffebc4574_power` / `..._d16b8_power` messen Strom bereits; Wärme grob über Öl-Minderverbrauch gegenrechnen). BW-WP-Verbrauch erfasst Norbert ebenfalls in Kürze — idealer Zeitverschiebungs-Kandidat (WW-Speicher träge → nachts/mittags im günstigen Fenster heizen).

## Fronius-Ladedeckel-Automatik (SoC-gesteuert) (2026-07-08) ✅

**Ziel:** Den AC→DC-Ladepfad über die MultiPlus (Fronius-Überschuss/Netz → Batterie) wetterabhängig steuern statt statisch 5 A. MPPTs laden immer ungebremst.

**Schlüssel-Erkenntnis (live verifiziert):** Der DVCC-Parameter „Maximaler Ladestrom" (Cerbo → System-Setup → Ladekontrolle = `com.victronenergy.settings /Settings/SystemSetup/MaxChargeCurrent`, float A) heißt zwar „systemweit", drosselt die DC-gekoppelten MPPTs aber faktisch NICHT (sie speisen direkt in den DC-Bus). Beleg: bei 5 A eingestellt lädt die Batterie mit 1.836 W / 7 kWh-Tag über MPPT. Er wirkt praktisch als **Fronius-/AC-Ladedeckel** — genau der gewünschte Hebel. (Norbert bestätigt: MaxChargeCurrent bestimmt, was der Fronius obendrauf zu den MPPTs legt.)

**Logik** (Flow „Fronius-Ladedeckel (Auto)", Repo `flows/victron-ladedeckel-auto.json`, Nodes `ldeck0*`):
- kontinuierlich: SoC (`/Dc/Battery/Soc`) → `flow.soc`.
- **14:00**: SoC < 60 % → 50 A (Fronius hilft), sonst 5 A.
- **00:00**: Reset → 5 A (garantiert morgens MPPT-Vorrang, kein verlustreiches Fronius-Vorladen; DC-Laden ~99 % vs. AC-Laden ~91 %).
- Sicherheits-Gate: nur 5/50 A erlaubt (ungültige Werte verworfen + geloggt). Guard „nur bei Änderung" **entfernt (08.07.)** → schreibt bedingungslos, robust gegen manuelle Cerbo-Eingriffe/Node-RED-Neustart (die frühere Stale-Cache-Falle: der Guard verglich gegen internen Merker, nicht gegen echten Cerbo-Wert). Editor-Buttons „Sofort 50 A / 5 A".
- **HA-Bedienung (08.07.):** Switch `switch.victron_system_fronius_ladehilfe` („Fronius-Ladehilfe", Device Victron System) via MQTT Discovery über Broker „Proxmox Mqtt" (`0057f46f75e31fea`). an=50A / aus=5A, Command-Topic `victron/ladedeckel/set`, **retained State-Feedback** `victron/ladedeckel/state` → Switch zeigt Ist-Zustand, Auto-Trigger bewegen ihn mit. Dashboard `victron-sys` → neue Sektion „Ladesteuerung" (Tile). **HA hängt Device-Namen an die entity_id** (Discovery-`name` „Fronius-Ladehilfe" + Device „Victron System" → `switch.victron_system_fronius_ladehilfe`, nicht `switch.fronius_ladehilfe`).
- **2026-07-08 Ende-zu-Ende getestet** ✅ (Werte wechseln am Cerbo). Handgebauter `victron-output-settings`-Node (kein Vorbild im System) funktioniert. State initial „unknown" bis zum ersten Schaltvorgang (Switch einmal betätigen zum Synchronisieren).

**Warum SoC-Checkpoint statt Solcast-Prognose:** Der Ist-SoC um 14:00 integriert bereits Vormittagsertrag + Hauslast — robuster als Prognose, kein MPPT/Fronius-Skalierungsproblem (Solcast trennt nur nach Dach Ost/West `sensor.solcast_pv_forecast_pv_anlage_ost|west`, nicht nach Ladepfad; MPPT-Anteil nur über 20-%-Skalierung von `..._verbleibende_leistung_heute` schätzbar). Ausgangsfall „trüber Morgen/sonniger Nachmittag" gelöst: bis 14:00 bleibt 5 A → kein Vorladen, der Nachmittag füllt per MPPT.

**Offen / Kalibrierung:**
- Schwelle SoC < 60 % gilt für 15-kWh-Akku (= 6 kWh Restbedarf um 14:00, entspricht MPPT-Nachmittagsertrag-Annahme ~6 kWh). **Nach Akku-Ausbau auf 30 kWh anpassen** — besser in kWh-Restbedarf denken; echten MPPT-Nachmittagsertrag über `victron_mppt_1/2_ertrag` ab 14:00 messen und Schwelle nachziehen.
- Erste automatische 14:00-Auslösung an einem trüben Tag beobachten (schaltet auf 50 A? Fronius lädt nach?).

## Ziel
Gridmeter **VM-3P75CT** in Home Assistant einbinden, insbesondere fürs Energie-Dashboard.

## Stand 2026-07-05
- **Norbert nutzt MQTT als systemweiten Standard** — HA-Anbindung daher per MQTT (Discovery), nicht über node-red-contrib-home-assistant-websocket. Der Websocket-Ansatz (`flows/victron-grid-energie-ha.json`) wurde verworfen, weil er die Node-RED-Companion-Integration (HACS) erfordert hätte.
- Aktueller Flow: `flows/victron-grid-energie-mqtt.json` — Zähler → retained MQTT-Topics `victron/grid/energy_forward|reverse`, Discovery-Configs retained nach `homeassistant/sensor/victron_*/config`.
- **Entscheidung:** Fürs HA-Energie-Dashboard nicht die zappelnde Momentanleistung glätten, sondern die eingebauten Energiezähler des Gridmeters nutzen: `/Ac/Energy/Forward` (Netzbezug, kWh) und `/Ac/Energy/Reverse` (Einspeisung, kWh). Die sind monoton und ruhig; das Dashboard braucht ohnehin kWh mit `state_class: total_increasing`, keine Watt.
- Flow erstellt: `flows/victron-grid-energie-ha.json` — 2× `victron-input-gridmeter` (onlyChanges) → 2× `ha-sensor` (device_class energy, state_class total_increasing) via node-red-contrib-home-assistant-websocket.
- Nach Import nötig: Gerät in den Victron-Nodes wählen, HA-Server in den Entity-Configs wählen; HA braucht die **Node-RED-Companion**-Integration (HACS).
- Vorheriger Ansatz (smooth-Node + 5 s Verzögerung auf `/Ac/Power`) verworfen.

## Stand 2026-07-06 — Ziel erreicht ✅
- MQTT-Flow läuft Ende-zu-Ende: `sensor.victron_vm_3p75ct_victron_netzbezug` (2.657,84 kWh) und `..._netzeinspeisung` (24.915,21 kWh) liefern Werte.
- Beide Sensoren sind als Grid-Quelle im HA-Energie-Dashboard eingetragen.
- Lernerfahrung: `onlyChanges` + nächtlicher ESS-Betrieb → erster Wert kann auf sich warten lassen (Zähler tickt nur alle 0,01 kWh); „unknown" direkt nach Deploy ist kein Fehler.

## PV-Erweiterung (2026-07-06) ✅
- Hardware: **2 MPPT-Laderegler + 1 Fronius PV-Wechselrichter** (AC-gekoppelt).
- Flow `flows/victron-pv-energie-mqtt.json` deployed: MPPTs via `/Yield/User`, Fronius via `/Ac/Energy/Forward`, Topics `victron/pv/...`.
- Sensoren `sensor.victron_mppt_1_ertrag`, `sensor.victron_mppt_2_ertrag`, `sensor.fronius_pv_energie` als Solarquellen im Energie-Dashboard eingetragen. Nachts noch „unknown" (erwartet) — erster Wert kommt bei Sonnenaufgang, dann verschwinden auch die Validierungswarnungen.
- **Gotcha bestätigt (Quellcode geprüft):** Victron-Input-Nodes mit `onlyChanges` senden erst bei einer D-Bus-Wertänderung — der Node-Status zeigt Werte an, ohne dass Nachrichten fließen. Kein Fehler, nur Geduld nötig.

## Batterie-Erweiterung (2026-07-06) ✅
- Flow `flows/victron-batterie-energie-mqtt.json` deployed: Batterie `com.victronenergy.battery/512`, `/History/ChargedEnergy|DischargedEnergy`, `/Soc` → Topics `victron/battery/...`.
- Batterie-Quelle im Energie-Dashboard eingetragen (geladen/entladen/SoC). Entladen + SoC liefern; „Geladen" bleibt bis zur ersten Ladephase (Sonnenaufgang) „unknown" — erwartet.
- **Debug-Infrastruktur entdeckt:** Node-RED-Admin-API offen unter `http://192.168.2.80:1880` (kein adminAuth) — /flows lesen/deployen, /context/global lesen, /victron/services. MQTT-Broker 192.168.2.233 verlangt Auth (anonym rc=Not authorized); zum Mitlesen temporären mqtt-in-Probe-Flow via Admin-API deployen (Broker-Config `0057f46f75e31fea` „Proxmox Mqtt"). HA-seitig: MQTT-Debug via Integration-Diagnostics (zeigt empfangene Messages pro Entität).

## Geglätteter Leistungssensor (2026-07-06) ✅
- Direkt per Admin-API auf den Grid-Tab deployed (Nodes `clpwr0*`, auch in `flows/victron-grid-leistung-glatt.json`): `/Ac/Power` → EMA (alpha 0.1) → Rate-Limit 1/10 s → `victron/grid/power_smooth` → `sensor.victron_vm_3p75ct_netzleistung_geglattet` (W, measurement). Für Live-Dashboards, NICHT fürs Energie-Dashboard.

## MQTT-Broker-Landkarte (2026-07-06)
- **192.168.2.233** — Proxmox-Mosquitto (HA + Node-RED), **Auth erforderlich** (anonym: Not authorized)
- **192.168.2.29** — openWB 2.x, anonym lesbar. EV-Zähler: `openWB/chargepoint/1/get/imported` in **Wh** (÷1000!). openWB sieht auch Netzzähler (counter/14), Victron-Batterie (bat/16), PV (pv/21) → macht eigenes PV-Überschussladen. Wallbox-Hardware: 192.168.2.145.
- **192.168.2.181** — Cerbo GX Venus-MQTT (`N/c0619ab31b3b/...`), anonym lesbar, Keepalive-gebunden

## E-Auto / openWB (2026-07-06) ✅
- Per Admin-API deployed (Tab „openWB EV → HA Energie (MQTT)", Nodes `clev0*`/`clevbroker*`, Repo-Kopie `flows/openwb-ev-energie-mqtt.json`): neuer Broker-Config „openWB Mqtt" (192.168.2.29, anonym), `openWB/chargepoint/get/imported` (Wh, Aggregat über alle Ladepunkte) ÷1000 → rbe → `victron/ev/energy` retained + Discovery.
- `sensor.openwb_geladen` liefert (6.797,92 kWh) und ist als **device_consumption** „E-Auto (openWB)" im Energie-Dashboard eingetragen.

## openWB-Ladeprotokoll dokumentiert Ladevorgang nicht (2026-07-23) — Ursache per MQTT diagnostiziert, Fix offen
**Anlass:** Norbert bemerkte, dass das openWB-Ladeprotokoll (UI) eine Ladung nicht anzeigte, obwohl geladen wurde.

**Live-Diagnose per MQTT** (Broker `192.168.2.29`, anonym lesbar — kein SSH-Zugang zum openWB-Host vorhanden, daher kein Blick ins App-/System-Log möglich): Kurzer `paho-mqtt`-Subscribe auf `openWB/#` (temp. venv unter `/tmp/mqttenv`, kein pip-System-Eingriff) zeigt, dass der Ladevorgang real stattfand — heute **11:33–15:57 Uhr, 32,54 kWh, 99,6 % PV-Anteil, Kia EV6**, sichtbar sowohl im Tageszähler `daily_imported` (32.538 Wh) als auch im retained Segment-Log-Eintrag `openWB/chargepoint/1/set/log`.

**Befund:** Genau dieser Log-Eintrag hat drei `null`-Felder, die für einen vollständigen Ladeprotokoll-Eintrag normalerweise gefüllt sein müssten: `soc_at_end`, `range_at_end`, `timestamp_start_charging`. Auffällig: `connected_vehicle/soc` lieferte fast zeitgleich (±0,2 s zum Log-`end`-Zeitstempel) einen validen SoC-Wert (79,71 %) — das Fahrzeug hat also geantwortet, der Wert landete nur nicht im Log-Eintrag.

**Korrektur (2026-07-23, Norbert bestätigt): Fahrzeug ist noch nicht abgesteckt.** Das stützt jetzt die zweite Erklärung stärker als vermutet: Der gefundene `set/log`-Eintrag ist wahrscheinlich nur ein **Zwischen-Segment** (ausgelöst durch den Chargemode-Wechsel auf „stop" um 15:57), kein finaler Sitzungs-Abschluss — die `null`-Felder (`soc_at_end`, `range_at_end`, `timestamp_start_charging`) wären dann konsequent, weil diese erst beim tatsächlichen Ausstecken final geschrieben werden. Die Race-Condition-Theorie (Cloud-SoC-Abfrage kommt zu spät) ist damit NICHT ausgeschlossen, aber nicht mehr die wahrscheinlichste Erklärung für den fehlenden UI-Eintrag.

**Nächster Schritt (einfachster Test):** Nach dem Ausstecken prüfen, ob der Ladevorgang im Ladeprotokoll erscheint. Erscheint er → kein Bug, nur „Sitzung war einfach noch offen". Erscheint er weiterhin NICHT → dann wieder `openWB/chargepoint/1/set/log` per MQTT prüfen, ob die Felder auch nach dem Ausstecken `null` bleiben — das wäre dann der stärkere Beleg für einen echten Log-Writer-Fehler.

**Offen:** Root-Cause-Bestätigung (falls nach Ausstecken immer noch kein Eintrag) braucht SSH-/Log-Zugriff auf den openWB-Host (`192.168.2.29`, Wallbox-Hardware `192.168.2.145`) — bisher kein Zugang hinterlegt. Technisches Detail zusätzlich in der Auto-Memory (`feedback_technical_gotchas.md`) festgehalten, als Diagnosemuster (Topic `set/log` auf `null`-Pflichtfelder prüfen — aber erst NACH dem Ausstecken als Bug-Indiz werten, vorher ist es erwartetes Zwischen-Segment-Verhalten).

### SSH-Zugang zur openWB-Box (2026-07-23) — vorbereitet, zurückgestellt bis Ladung fertig
**Befund:** `192.168.2.29` hat Port 22 offen, aber SSH akzeptiert nur Public-Key-Auth (kein Passwort-Prompt, direkt „Permission denied (publickey)" mit Norberts vorhandenen Keys `id_rsa`/`id_ed25519`) — kein Key von Norberts Rechner ist dort hinterlegt. Kein SSH-Config-Eintrag für openWB vorhanden (anders als `Homeassistant`-Host in `~/.ssh/config`).

**Geplanter Weg (SD-Karte ziehbar, von Norbert bestätigt):** Box stromlos, microSD entnehmen, an einem Linux-Rechner mounten, eigenen Public Key manuell in `/home/openwb/.ssh/authorized_keys` auf der **ext4-Root-Partition** eintragen (Berechtigungen 700/600, Ownership auf UID/GID aus `/etc/passwd` der Karte setzen — NICHT auf die boot-FAT32-Partition, dort ist nur `ssh`-Datei-Trick für reines SSH-Aktivieren relevant, keine Nutzer-Keys). Danach Karte zurück, Box hochfahren, `ssh openwb@192.168.2.29` sollte direkt ohne Passwort funktionieren.

**Warum zurückgestellt:** Norbert hat gerade eine Ladung laufen (23.07. abends) — Box stromlos machen würde die Ladung unterbrechen und ist ohne sauberes Shutdown ein (kleines) Risiko für die SD-Karte. **Geplant für morgen**, sobald keine Ladung aktiv ist.

## Sankey-Karte (2026-07-06) — eingebaut und auf Wunsch wieder ENTFERNT
- Energie-Karten (`energy-date-selection`, `energy-distribution`, `energy-sankey`, `energy-devices-graph`) waren im Dashboard „System" (`dashboard-system`) — Norbert hat nach Ansicht des Vorbild-Screenshots **alles wieder entfernen lassen** (Original-Zustand verifiziert wiederhergestellt, config_hash identisch). Vermutlich neuer Anlauf mit größerem Wurf geplant (Vorbild: volles Energie-Board mit Sankey inkl. vieler Geräte-Äste, CO2 Signal Grün/Fossil, Forecast.Solar, Carbon-Gauge).
- Wissen (Doku-verifiziert): `energy-distribution` zeigt NUR Netz/Solar/Batterie/Haus, nie individuelle Geräte. E-Auto erscheint im Sankey-Ast/Devices-Graph erst nach Ladung im Zeitraum. Auto als Kreis in Flussansicht: HACS „Power Flow Card Plus" (openWB publisht `chargepoint/get/power`).
- Offene Bausteine für das Vorbild: CO2-Signal-Integration (API-Key co2signal.com), Forecast.Solar (Anlagendaten), Geräte-Inventur (vorhandene kWh-Sensoren von Plugs/Shellys suchen und als device_consumption eintragen).

## Bereichskarte „Leistung" (2026-07-06) ✅
- Pseudo-Bereich `energie` angelegt (später umbenannt in „**Leistung**" — area_id bleibt `energie`; ebenso Dashboard-Titel „Leistung (Sys)", View „Leistung", Karte „Leistungsfluss live" — durchgängige Leistungs-Terminologie auf Norberts Wunsch) (Etage „System" wie `victron`, Icon mdi:lightning-bolt, Bild = **Data-URI-SVG** direkt im Area-Registry — kein Upload/Hosting nötig; SVG-Quelle im Scratchpad erzeugt, Motiv: Sonne, PV-Haus, Batterie, Strommast, E-Auto).
- Area-Karte im Dashboard „System" ergänzt, identischer Aufbau wie Victron-Karte (display_type picture, columns 12/rows 3), **navigiert zu `/energie-sys`**.

## Power Flow Card Plus (2026-07-06) ✅
- HACS: `flixlix/power-flow-card-plus` v0.3.7 installiert (Browser-Cache-Hinweis!).
- Live-Leistungssensoren via Node-RED-Tab „Victron Leistung → HA (MQTT)" (Repo: `flows/victron-leistung-mqtt.json`): `sensor.victron_system_pv_leistung` (system `/Dc/Pv/Power` + Fronius `/Ac/Power`, Summe), `sensor.victron_batterie_leistung` (system `/Dc/Battery/Power`, **positiv = laden**), `sensor.openwb_ladeleistung` (openWB `chargepoint/get/power`). Alle rateLimit 5 s, W, measurement.
- Neues Dashboard **`energie-sys`** „Energie (Sys)" (nicht in Sidebar, wie victron-sys): PFCP-Karte mit grid=geglättete Netzleistung, solar=PV-Summe, battery+SoC, individual=E-Auto. Kartentitel bewusst „Leistungsfluss live" (zeigt W, nicht kWh). **Gotcha:** PFCP-Batterie erwartet positiv = Entladen („Haus konsumiert aus dem Knoten", gleiche Lesart wie beim Grid-Eintrag), Victron liefert positiv = Laden → `invert_state: true` ist **nötig und bleibt**. (Am 2026-07-06 kurzzeitig fälschlich entfernt und wieder hergestellt — siehe unten.)

## Batrium-Schütz-Alarm → Pushover (2026-07-06) ✅ KOMPLETT
- **Ziel:** Pushover-Meldung, wenn das Batrium den Schütz auslöst. Umsetzung auf dem WatchMon-Tab (Nodes `clbtrm*`, Repo `flows/batrium-schuetz-pushover.json`).
- **Zwei Grundprobleme entdeckt und behoben:**
  1. **UDP-Port-Konflikt:** Der Test-Tab „UDP Akku" hielt Port 18542, der WatchMon-Listener bekam NICHTS. Duplikat-Listener deaktiviert (nicht gelöscht), WatchMon-Listener neu gebunden → 404 Pakete/20 s von 192.168.2.103.
  2. **Firmware-Protokollwechsel:** Batrium sendet mit aktueller FW (≥2.15, live: FW-Feld 3588) NEUE Message-IDs — der komplette 1.0.30-Decoder des WatchMon-Tabs ist tot (3F33/3E32/6131… kommen nie). Live gemessen: 3233 LiveDisplay (sporadisch!), 3f34 StatusShunt (300 ms), 5732 SystemDiscovery (2 s), 4733 StatusControlLogic (~3 s), 415a/4232 Zelldaten. Spez: github.com/Batrium/WatchMonUdpListener (payload/*.js). Bits sind LSB-first (live verifiziert).
- **Architektur (wie besprochen):** Pushover DIREKT aus Node-RED (kürzeste Kette, hängt nicht an HA/Broker) + Status parallel als MQTT-Sensor nach HA.
- **Pipeline:** udp 18542 → Topic-Function → Switch: Ausgang 7 (`5732`, primär, alle 2 s) + neue Regel `3233` (reicher: SoC/Leistung) → Decoder → „Schütz-Wächter": Flankenerkennung auf `SystemOpStatus`. Neue Regel `4733` → Critical-Flags in Flow-Kontext (`batrium_critical`) als Auslöse-Grund für die Meldung.
- **Meldungen:** CriticalOffline → Prio 2 (Emergency, retry 60 s/expire 1 h, Sirene); CriticalPending → Prio 1; Rückkehr → Entwarnung Prio 0. Jeweils mit Zellspannungen, Temp, ggf. SoC/W und Auslöser-Flags.
- **HA:** `sensor.victron_batterie_batrium_status` (zeigt „Discharging" ✓) + `binary_sensor.victron_batterie_batrium_schutz_alarm` (device_class problem, off ✓) via Discovery, Topics `victron/battery/batrium_status|batrium_problem` retained, Device „Victron Batterie".
- **Neuer SystemOpStatus-Enum (FW ≥2.15):** 0 Simulator, 1 Idle, 2 Discharging, 3 Empty, 4 Charging, 5 Full, 6 Timeout, 7 CriticalPending, 8 CriticalOffline, 9 MqttOffline, 10 AuthSetup, 11 ShuntTimeout — Charging/Discharging sind gegenüber alt VERTAUSCHT.
- **Selbsttest eingebaut (2026-07-07) ✅:** Zwei Auslöser für eine Prio-0-Testmeldung durch die echte Kette: (a) Inject „Test: Pushover-Zustellung" im Editor (WatchMon-Tab), (b) HA-Button `button.victron_batterie_batrium_alarm_testen` („Batrium-Alarm testen", Device Victron Batterie) via MQTT Discovery → Topic `victron/battery/batrium_test` → mqtt-in → „Testmeldung bauen". Meldung enthält Auslöseweg + aktuellen Batrium-Status (Wächter legt ihn in Flow-Kontext `batrium_status`). „Pushover-Antwort prüfen" schreibt letztes Sende-Ergebnis nach Flow-Kontext `pushover_last` (ok/statusCode/time). Ende-zu-Ende verifiziert per Button-Druck: HTTP 200 ✓. **Gotcha dabei:** Norberts Editor-Deploy (stale Browser-Tab) hatte den SupplyVoltLo-Filter im 4733-Decoder zurückgerollt — wiederhergestellt; Regel: nach API-Deploys Editor-Tabs erst neu laden (F5), sonst merged der nächste Editor-Deploy alte Stände zurück. Repo-Kopie enthält Platzhalter statt Credentials (Scrub beim Sync + Assert).
- **Dashboard-Sektion „Batrium (BMS)“ in `victron-sys` (2026-07-07) ✅:** Tiles für Status, Schütz-Alarm und Test-Button (tap = button.press, „Pushover-Test senden“, 12 Spalten; Status/Alarm je 6). Neuer config_hash 98ca39c21a7eb32d.
- ~~OFFEN (1)~~ ERLEDIGT: Pushover-Credentials von Norbert eingetragen (App-Token + User-Key, je 30 Zeichen, stehen NUR im Live-Flow — die Repo-Kopie behält bewusst die Platzhalter, bei künftigen Repo-Syncs Credentials ausnehmen!). Testmeldung Prio 0 durch die echte Kette gesendet: HTTP 200, status 1, Empfang auf dem Handy von Norbert bestätigt ✓ (2026-07-06 spätabends). (2) ~~Kurioses Flag `hasCriticalSupplyVoltLo`~~ GEKLÄRT (Msg_4f33 ControlCriticalSetup passiv mitgelesen): Kriterium ist DEAKTIVIERT (MonitorSupplyLo=Off), Schwelle steht auf 40 V (Default für pack-gespeisten WatchMon) vs. 13,2-V-Versorgung → Roh-Vergleich dauerhaft true, aber wirkungslos. Flag wird im 4733-Decoder jetzt gefiltert. Aktive Critical-Kriterien: Zellspannung 2,80/3,60 V, Zelltemp −10/55 °C, Pack 44,8/57,0 V; Mode=Auto, AutoRecovery=An (→ Entwarnungs-Meldung wird real feuern). (3) Watchdog („WatchMon stumm") bewusst abgewählt. (4) Rest des WatchMon-Tabs (Zell-Globals/Chart) weiterhin tot — Kandidat für „Stufe B"-Modernisierung.

## BW-WP in den Energie-Dashboards ergänzt (2026-07-09) ✅

Datenpfad (InfluxDB, Grafana-Dashboard „BW-WP Heizungskeller“): → [[homelab-monitoring#Brauchwasser-WP: Messstelle, Datenpfad, Zigbee-Störungen]] (23.08.2026).

- **PFCP „Leistungsfluss live"** (`energie-sys`): BW-WP als vierter individual-Kreis ergänzt — Name **„BW WP"**, `sensor.0xa085e3fffeb7c870_power`, mdi:heat-pump, display_zero (wie E-Auto/SK AZ/SK WZ).
- **EFCP „Energiebilanz"** (`energiebilanz-sys`): BW-WP ebenso als vierter individual-Eintrag — „BW WP", `sensor.0xa085e3fffeb7c870_energy` (kWh-Zähler, EFCP rechnet Zeitraum-Summen über die Energie-Collection), mdi:heat-pump, display_zero.

## BW-WP zeigte in der Energiebilanz nichts (2026-07-10) — Datenpfad gelöst (5-min-Poll ✅), Karten-Anzeige noch offen

**Symptom (Norbert):** In `energiebilanz-sys` (EFCP, 8 Kreise) zeigte der BW-WP-Kreis nichts an, obwohl er anfangs Werte hatte. Karten-Config war korrekt (4. individual-Eintrag, `display_zero: true`).

**Ursache:** Die Steckdose „BrauchwasserWP" ist ein **Shelly 1PM Gen 4** (Zigbee via Z2M) — **identische Hardware wie die SK-Dosen**. Sie war >1 Tag `unavailable` (8.7. → 9.7. 20:18), beim Wiedereinbuchen startete ihr Energie-Zähler bei 0 (Geräte-Neustart/Re-Join) und das **Zigbee-Attribut-Reporting für den Energie-Zähler ging dabei verloren**: Leistung (W) meldete weiter (Firmware-Default), der kWh-Zähler kam nur noch auf aktives `get` (deshalb brauchte auch das InfluxDB-Setup am 9.7. den „get-Anstoß"). Folge: Zähler fror bei 0,45 kWh ein (letzter Selbst-Report 9.7. 23:33), zwei WP-Läufe (~0,84 kWh) blieben ungemeldet → Tages-`change` in den Statistiken = 0 → EFCP-Kreis leer.

**Referenz SK-Dosen (warum es dort läuft):** Kein Extra-Mechanismus — normales Zigbee-Reporting, beim Pairing von Z2M ins Gerät geschrieben: `seMetering/currentSummDelivered`, min 10 s, max 65000 s, Änderungsschwelle 0,1 kWh → die bekannten 0,1-kWh-Schritte (~alle 28 min bei 211 W).

**Fix (2026-07-10):**
1. Manueller Poll `zigbee2mqtt/BrauchwasserWP/get` `{"energy": ""}` → Zähler sprang 0,45 → **1,29 kWh** (Diagnose-Beweis; heutige ~0,84 kWh landen statistisch in der 09:55-Stunde, nicht in den echten Laufzeit-Stunden).
2. `zigbee2mqtt/bridge/request/device/configure` `{"id": "BrauchwasserWP"}` → Antwort `status: ok`; danach via retained `zigbee2mqtt/bridge/devices` verifiziert: `configured_reportings` jetzt **identisch zu SK AZ/WZ** (inkl. seMetering-Regel).
3. Linkqualität aktuell gut (~204/255) — der Tages-Ausfall war eher Join-/Geräteproblem als Funkloch; beobachten, ob `unavailable`-Aussetzer wiederkehren (dann Zigbee-Router Richtung Keller).

**Live-Test GESCHEITERT → Poll-Flow als Dauerlösung (2026-07-10 mittags) ✅:**
- Beweis-Lauf 10:25–11:10 (596 W, ≈0,44 kWh): Leistung meldete alle 10–30 s, der **Energie-Zähler blieb den kompletten Lauf bei 1,29 kWh eingefroren** — die Shelly-Firmware honoriert das (korrekt eingetragene!) seMetering-Reporting schlicht nicht. Warum die SK-Dosen (gleiches Modell) von selbst melden: unklar, vermutlich Firmware-Unterschied (`update.installed_version = -1`, OTA-Stand unbekannt — ggf. mal Firmware-Update der BW-Dose probieren, könnte den Poll überflüssig machen).
- Kontroll-Poll danach: 1,29 → **1,71 kWh** (+0,42, deckt sich mit dem Lauf) — intern zählt die Dose korrekt, sie pusht nur nicht.
- **Dauerlösung deployed:** Node-RED-Tab „**Zigbee BW-WP Energie-Poll**" (Live-ID `c0cf881d9a69a14b`, Repo `flows/zigbee-bwwp-energie-poll.json`): inject alle 300 s (+ einmalig 30 s nach Start) → mqtt-out `zigbee2mqtt/BrauchwasserWP/get` `{"energy": ""}` über Broker „Proxmox Mqtt". Antwortzeit der Dose ~100 ms. **Verifiziert:** automatischer Zyklus läuft exakt im 5-min-Takt (11:23:12/11:28:12), Antwort kommt auf `zigbee2mqtt/BrauchwasserWP` an → HA-Sensor/Energiebilanz bekommen künftig max. 5 min alte Stände.
- **Deploy-Gotcha:** Nach dem POST /flow des Poll-Tabs publizierte der mqtt-out-Node zunächst NICHT (Inject HTTP 200, aber nichts am Broker); erst nach dem nächsten Deploy (weiterer Flow-POST) hing er an der Broker-Verbindung. Bei per Admin-API neu angelegten mqtt-out-Nodes das erste Publish immer per Mitlese-Probe verifizieren.
- Temporäre Debug-Flows („TEMP Z2M Probe", „TEMP Poll-Debug") wieder gelöscht, Session-Watcher gestoppt.

**UNGELÖSTER REST (Stand 2026-07-10 mittags) — ✅ ERLEDIGT 2026-07-18 (siehe Sektion „EFCP-Anzeige-Bug behoben"):** Die EFCP-Karte zeigt für BW WP weiterhin **840 Wh**, obwohl die Langzeit-Statistik nachweislich **1,26 kWh** für heute enthält (5-min-Zellen: +0,84 um 09:55, +0,42 um 11:15 — per API verifiziert) — **auch nach Seiten-Reload** (Erklärungsversuch „Energie-Collection-Refresh" hat sich damit NICHT bestätigt). Datenpfad bis in die Statistik ist sauber; es hakt zwischen Statistik und Karten-Anzeige. **Norbert spielt ein Update auf** (nicht näher spezifiziert — HA-Core und/oder Shelly-Firmware), danach neuer Test. Prüfschritte für den Retest: (1) Reload → zeigt BW-WP-Kreis 1,26 kWh (bzw. aktuellen Tages-change)? (2) Falls nicht: eingebautes Energie-Dashboard (Geräteverbrauch BW-WP) vergleichen — trennt EFCP-Bug von Energie-Collection-Problem; (3) EFCP v0.2.3 auf Update prüfen (HACS). Poll-Flow läuft unabhängig davon korrekt weiter.

**Nebenbefund PFCP (andere Karte, `energie-sys` „Leistungsfluss live"):** Erste Fehlspur der Diagnose, aber echtes Verhalten — PFCP v0.3.7 rendert bei **Kartenbreite < 359 px nur die ersten 2** individual-Kreise (Bundle-Logik: `allow_layout_break || width >= 359 ? 4 : 2`, dann `filter(has).slice(0,T)`; ResizeObserver misst live). Auf Desktop alle 4 sichtbar, auf schmalen Screens fliegen SK WZ + BW WP raus (Reihenfolge = Config-Reihenfolge, kein Sortieren ohne `sort_individual_devices`). Fix bei Bedarf: `allow_layout_break: true` an der Karte — **noch nicht gesetzt** (Norberts eigentliches Anliegen war die EFCP-Karte).

## BrauchwasserWP-Dose schaltet sich selbst aus (2026-07-12) — Fall geschlossen

→ Verschoben nach [[homelab-monitoring#Brauchwasser-WP: Messstelle, Datenpfad, Zigbee-Störungen]] (23.08.2026).

## Zigbee-Vormittags-Ausfall 15.07. — Coordinator-Socket-Timeouts (2026-07-15)

→ Verschoben nach [[homelab-monitoring#Brauchwasser-WP: Messstelle, Datenpfad, Zigbee-Störungen]] (23.08.2026).

## EFCP-Anzeige-Bug behoben + SK-WZ-„20-Wh"-Fehlalarm (2026-07-18) ✅

**Anlass (Norbert):** SK WZ zeigte in der Energiebilanz (`energiebilanz-sys`, EFCP) nur 20 Wh, obwohl die Anlage seit Stunden lief.

**Auflösung — kein Fehler im Datenpfad:** Die Datumswahl der Karte stand auf dem **17.07.** — und gestern lief die SK WZ real gar nicht (Leistung ganztags 0 W, einziger Zähler-Tick +0,02 kWh = exakt 20 Wh um 14:05, Standby-Drift). Nach Umschalten auf den 18.07. zeigte die Karte 500 Wh — korrekt: Anlage lief erst ab ~12:15 (Anlauf-Peak 1.347 W, danach Inverter-Modulation auf ~190–220 W Mittel), Zähler tickt sauber in 0,1-kWh-Schritten alle ~30 min. **Merke für Inverter-Splits:** ~200 W Dauerlast nach dem Abkühlen ist normal — „läuft seit Stunden" heißt nicht „mehrere kWh".

**Nebenbefund — der offene EFCP-Anzeige-Bug vom 10.07. ist behoben:** Alle drei Geräte-Kreise zeigen jetzt exakt die Langzeit-Statistik (stundengenau nachgerechnet): SK WZ 0,5 kWh, SK AZ 0,8 kWh (0,1 um 7 Uhr + Läufe ab 12 Uhr), BW-WP 2,0 kWh (typisches Zyklus-Muster à 0,4–0,5 kWh nachts/4/8/12 Uhr). Das von Norbert eingespielte Update (um den 15.07., nicht näher spezifiziert — HA-Core und/oder EFCP) hat den Hänger „Karte zeigt stale Werte trotz korrekter Statistik" beseitigt. Retest aus der BW-WP-Sektion vom 10.07. damit abgeschlossen.

**Diagnose-Reihenfolge, die sich bewährt hat:** (1) Roh-Zähler + `last_updated` prüfen (Dose eingefroren?) → (2) Leistungs-Historie (lief das Gerät überhaupt im angezeigten Zeitraum?) → (3) Langzeit-Statistik stundengenau summieren → (4) erst dann Karten-/Collection-Bug vermuten. Hier war es Schritt 2: der „falsche" Wert war korrekt, nur der Zeitraum ein anderer.

## Batrium CriticalPending bei SoC 100 % — Analyse, Ladespannungs-Tuning, PDF (2026-07-18) — Umsetzung geplant 19.07.

**Anlass:** Zweimal binnen einer Woche Pushover „Abschaltung droht (CriticalPending)" bei vollem Akku (2. Vorfall 18.07. 12:51, Dauer **3 s**, AutoRecovery + Entwarnung gesendet — erster echter Ernstfall der Alarmkette, sie funktionierte einwandfrei; Schütz fiel nie).

**Messbefund (HA 1-s-Daten):** 12:49:50 Ladeburst **+26 A** in den vollen Akku → Pack-Überschwinger **56,06 V** (Ø 3,50 V/Zelle) über die Decke 55,8; ~100 s später tippt eine Einzelzelle kurz über 3,60 V (CV12) → Pending 12:51:40–43.

**Diagnose-Verlauf (wichtig, mit Korrektur):**
1. Erste These „invertierte Leiter" (Wiki-Beispiel-Cutout 3,65 > Critical 3,60 aus 4f33-Auslesung) — **widerlegt** durch den **Volt Guide** (Batrium-Portal → Teams → System → Volt Guide): CV10 steht real auf **3,55**, Ordnung korrekt.
2. Tatsächliche Ursache: **Kompression** — Ladeziel RV9 3,49 (55,80) → Cutout 3,55 → Critical 3,60 = nur **110 mV** Gesamtreserve, davon **50 mV** Cutout→Critical (Wiki-Beispiel: 150 mV), im steilsten Kurvenbereich. Burst ≈ 26 mV/s → Zelle durchläuft die Reserve schneller als die träge 30-kW-Flotte abregeln kann. **Cutout feuert korrekt, wirkt zu spät.**

**Schlüssel-Erkenntnisse:**
- **Volt Guide = Spiegel der eigenen Konfiguration, KEINE Vorgabe** („Guide" = Landkarte; Beweis: zeigt die eigenen Werte 3,55/3,60 statt der Wiki-Beispiele 3,65/3,80 und wandert nach Änderungen mit). Batriums echte „Muster-Einstellung" = Chemie-Presets (Batt Type) + Default-Buttons je Tab; es gibt KEIN offizielles Empfehlungs-Dokument. Normativ bleibt das Zell-Datenblatt (3,65 V max).
- **Kürzelsystem:** CVx = Zellschwellen (Eingabe Zell-Volt), **RVx = Remote-Ladeziele (Eingabe PACK-Volt!)**, QVx Shunt, CAx Bypass-Strom/-mAh, CTx/BTx Temp, BSx SoC. Auf dem Remote-Tab gibt es keinen CV-Wert — „Dynamic Volt Max" = RV9, als 55,2 eintragen (3,45 ist nur ÷16-Anzeige).
- Wiki-CV12 3,80 > Datenblatt-Max ist kein Widerspruch: Havarie-Detektor („Lader ignoriert Cutout"), dank fast senkrechter Kurve oberhalb 3,65 schneller Indikator — Norberts 3,60 ist die konservativere Philosophie, dann muss die Betriebsleiter aber darunter Platz finden.

**Beschlossene Maßnahme (Stufe 1, nur zwei Werte, geplant 19.07.):**
- Control Logic → Remote: **RV9 „Dynamic Volt Max" 55,8 → 55,2 V** (beide Spalten Normal+Limited, Pack-Wert; vorher Default→Save wegen Dezimal-Bug)
- Hardware → CellMon: **CV9 „Bypass Volt" 3,47 → 3,42 V** + **Device Sync** (zwingend zusammen mit RV9, sonst Balancing/CA3-Vollerkennung tot → SoC-Drift; Fenster Plateau−CV9 wächst 20→30 mV)
- Stufe 2 (optional): Offset 0,2 / CV10 3,53 / CV14 3,45 / CV8 3,40 / CV11 3,55 / CV15 3,38 / CCL 140 A. **Nicht anfassen:** CV12 3,60, Pack 57,0, Entladeseite.

**Dokument:** `docs/batrium-ladespannung-tuning.pdf` (14 Seiten, 5 eigene Grafiken, Aufgaben-Checkliste mit Vorher/Nachher-Volt-Guide-Tabelle auf S. 1) — Generator `tools/gen_batrium_doc.py` (Konvention: Änderungen im Generator, dann Chromium `--headless --print-to-pdf`).

**Offen (nach Umstellung):**
1. Volt Guide: RV9 `3.45|55.20`, CV9 `3.42|54.72`, Rest unverändert.
2. **CVL am Cerbo prüfen** (D-Bus `com.victronenergy.battery/512 /Info/MaxChargeVoltage` ≈ 55,2 via Node-RED) — Claude übernimmt.
3. Sonnentag: `victron_batterie_spannung` flacht bei ~55,2 ab, kein Alarm; über einige Tage: SoC erreicht weiter 100 % (CA3 greift).
4. Danach Fall schließen. Gauge-Schwellen-Frage „PV-Speicher-Nutzgrad" unabhängig davon weiter offen.

**App-Stolperfallen (Details in Auto-Memory):** Remote-Werte uint16-skaliert (Dezimal-Bug → Default→Save-Zyklus, notfalls Template „Advanced"), Editieren nur im Technician-Mode, CellMon/Charging-Save laut Wiki nur über **USB** (WiFi → Fehlermeldung) — Kandidat für Norberts Android-Editierproblem.

## CO₂-Gauge zeigt nachts 69 % — Anzeige-Artefakt, kein Datenfehler (2026-07-19) ✅ ZU

**Anlass:** Karte „Verbrauchte CO₂-neutrale Energie" (`energy-carbon-consumed-gauge`, siehe [[#Electricity Maps / CO2 (2026-07-06) ✅]]) zeigte um 1 Uhr nachts 69 % — bei praktisch autarker Anlage (Netzbezug ~0,1 kWh/Tag) unplausibel.

**Befund (quellcode-verifiziert + nachgerechnet):** Die Karten-Formel `(1 − fossil / (Netzbezug + max(0, Solar − Einspeisung))) × 100` **ignoriert die Batterie komplett**. Nachts (Solar = 0) besteht der Nenner nur aus dem minimalen Netzbezug (~0,01 kWh) → die Gauge zeigt exakt den Netz-Mix von Electricity Maps (100 − 30,79 % fossil ≈ 69 %), obwohl das Haus real aus dem PV-geladenen Akku läuft. Tagsüber und in der Monatsansicht korrekt ~99–100 %. Sensoren, Statistiken und Energie-Dashboard-Konfiguration alle geprüft und in Ordnung — **nichts zu reparieren**.

**Option (nicht beauftragt):** Batterie-bewusste Kennzahl in Node-RED/MQTT rechnen, falls die nächtliche Anzeige stört. Details der Stolperfalle in der Auto-Memory (`feedback_technical_gotchas.md`).

## Nächster Schritt
- Verifizieren (tagsüber): PV-Sensoren (Sonnenaufgang), Batterie „Geladen" (erste Ladephase), E-Auto-Balken (erste Ladung), Validierungswarnungen im Energie-Dashboard sollten alle weg sein; Sankey-Karte zeigt dann Flüsse inkl. E-Auto-Ast.

## Energy Flow Card Plus / Energiebilanz (2026-07-06) ✅
- HACS: `flixlix/energy-flow-card-plus` v0.2.3 installiert. Karte bindet an die **Energie-Collection** (Zeitraum via energy-date-selection), Entities = kumulierte kWh-Statistiken.
- **PV-Gesamtzähler** via Node-RED (PV-Tab, Repo `flows/victron-pv-summe-mqtt.json`): abonniert die eigenen retained Topics (mppt1+mppt2+fronius), Summe → `victron/pv/energy_total` → `sensor.victron_system_pv_energie_gesamt` (EFCP-solar nimmt nur EINE Entität). Sendet erst, wenn alle 3 Quellen geliefert haben.
- Dashboard **`energiebilanz-sys`** „Energie (Sys)": date-selection + EFCP (grid=bezug/einspeisung, solar=PV-Summe, battery: consumption=**entladen**/production=**geladen** ⚠️ Richtung bei erster Ladung prüfen, individual=E-Auto+SK AZ+SK WZ).
- Zweiter Pseudo-Bereich **`energie_2`** „Energie" (Tag-Version des SVG, mdi:meter-electric) → Kachel unter System navigiert zu `/energiebilanz-sys`.
- System-Dashboard hat jetzt 3 Bild-Kacheln: Victron → victron-sys, **Leistung** (SVG: dunkler Tacho mit grün/rotem Bogen, Zeiger, „WATT") → energie-sys (PFCP, W), **Energie** (SVG: helles Balkendiagramm mit Sonne, „kWh") → energiebilanz-sys (EFCP, kWh). Bilder als Data-URI im Area-Registry; SVG-Quellen im Scratchpad (leistung-tacho.svg, energie-bilanz.svg).

## PFCP-Batterie-Vorzeichen: Verifikation mit Doppel-Irrtum (2026-07-06) ✅
- **Endergebnis: das `invert_state: true` von gestern war korrekt und bleibt.** Zwischenzeitlich wurde es entfernt (Fehldiagnose) und nach Live-Gegenprobe wiederhergestellt; `energie-sys` ist wieder auf dem Original-Stand (config_hash `f5ed49ce9e8f3624`).
- **Die Falle:** Die PFCP/power-flow-card-README beschreibt die Batterie-Entität nur als „negative values for production and positive values for consumption" — mehrdeutig. Falsche Lesart: „Batterie konsumiert" = lädt. Richtige Lesart (Quervergleich mit dem Grid-Eintrag, der identisch formuliert ist und bei dem positiv unstrittig Netzbezug bedeutet): **„das Haus konsumiert aus dem Knoten"** → Batterie positiv = **Entladen**. Victron (`/Dc/Battery/Power`) liefert positiv = Laden → Konventionen entgegengesetzt → Invert nötig.
- **Beweis in beiden Richtungen:** nachts (Entladung, Roh-Wert negativ) stimmt die Anzeige nur MIT Invert; morgens beim Laden (Roh +450…750 W, SoC 56→61 % steigend) zeigte die Karte OHNE Invert fälschlich Akku→Haus.
- **Auslöser der Fehldiagnose:** Norberts Frage „Akku→Haus obwohl er lädt?" bezog sich auf die **Energieverteilung** — dort ist das korrekt (Zeitraum-Summen: 3,4 kWh Nacht-Entladung), nicht auf die PFCP-Karte. Dazu kam eine KI-Doku-Zusammenfassung, die die mehrdeutige README-Stelle falsch auflöste.
- **Lehren:** (1) Vorzeichen-Logik immer in beiden Richtungen gegen Rohwert + SoC-Verlauf testen. (2) Bei mehrdeutigen Doku-Formulierungen analoge, eindeutige Einträge desselben Schemas als Referenz nehmen (hier: Grid). (3) Erst klären, **welche Karte** gemeint ist — Energieverteilung/EFCP zeigen kWh-Summen des Zeitraums, PFCP zeigt Momentan-Watt.
- Kontext-Fund am selben Tag: fehlende Fluss-Punkte in der HA-Energieverteilung auf dem Smartphone lagen an Android-Bedienungshilfe „Animationen entfernen" (`prefers-reduced-motion`; Core-Karte respektiert es, PFCP nicht) — Details in der Auto-Memory.

## Anlagen-Stammdaten (2026-07-06, von Norbert bestätigt + OSM-verifiziert)
- **Gesamt: 25,2 kWp**, Satteldach **30°**, Ost/West. Firstachse 13,8° NNO (aus OSM-Gebäudeumriss 15,7×9 m, Friedrich-Ebert-Str. 17, 69493 Hirschberg-Großsachsen, way 151647625; Standort 49.51404, 8.65993).
- Dachflächen-Azimut (OSM-berechnet, ersetzt Norberts Schätzung −80/+110): **Ost −76°** zu Süd (Kompass 104°), **West +104°** zu Süd (Kompass 284°).
- **Fronius Symo 17.5-3-M** (AC 17,5 kW): 10,08 kWp Ost + 10,08 kWp West (2 MPP-Tracker).
- **2× SmartSolar MPPT 250/70** (DC, an Batterie): „MPPT Ost" (Instanz 278) 2,52 kWp, „MPPT West" (277) 2,52 kWp. 31-Tage-Peaks: Ost 3.098 W, West 2.828 W (per Venus-MQTT `/History/Daily/n/MaxPower` ausgelesen — Keepalive an `R/c0619ab31b3b/keepalive`, dann `N/.../solarcharger/#`).
- Pro Seite gesamt: **12,6 kWp Ost, 12,6 kWp West**.

## Forecast.Solar (2026-07-06) ✅
- Zwei Core-Integration-Einträge (Weg ohne API-Key — die neue Subentry-Variante „mehrere Flächen in einem Eintrag" erzwingt ab Fläche 2 einen API-Key): **„Ost-Dach"** (Entry `01KWV8XCEAX9D33V5S05BP72YP`, 30°/104°/12.600 Wp) und **„West-Dach"** (`01KWV945MHPPGSX92G7VB128RP`, 30°/284°/12.600 Wp). Azimut in HA = Kompass-Grad (0=N), nicht „relativ zu Süd"!
- Beide Entries via `config_entry_solar_forecast` an der Fronius-Solarquelle in den Energie-Einstellungen verknüpft (Summe = Gesamtanlage; Zuordnung zu welcher Quelle ist kosmetisch, Dashboard summiert die Entry-Menge).
- Entitäten: `sensor.energy_production_today[_2]`, `..._tomorrow`, `sensor.power_production_now`, Peak-Zeiten; Suffix `_2` = West-Dach. Erste Plausibilität: 52,3 kWh Tagesprognose, Peaks Ost 12:00 / West 18:00 — ok; um 10:56 lag Ist (10,42 kWh) 12 % über Prognose-bis-jetzt (9,3) — gut.
- **Summen-Helfer** (min_max type sum, Bereich energie_2): `sensor.pv_prognose_heute` (= today + today_2) und `sensor.pv_prognose_morgen`. Als Tile-Kacheln (6 Spalten, amber) im `energiebilanz-sys` zwischen Datumswahl und EFCP. Lese-Gotcha: Forecast-Sensoren existieren doppelt (ohne Suffix=Ost, `_2`=West) und haben vier Zeitbasen (today/remaining/current_hour/now) — für Vergleiche immer beide addieren und Zeitbasis beachten.

## Solcast (2026-07-06) ✅ — ersetzt Forecast.Solar als primäre PV-Prognose
- Anlass: Forecast.Solar prognostizierte 52,3 kWh bei realen ~100+; VRM (nutzt Solcast) sagte 89–109. Nach Umstieg: **Solcast 104,4 kWh** (Ost 49,8 + West 54,6) — deckungsgleich mit VRM.
- **Hobbyist-Account** (kostenlos, nicht-kommerziell, max. 2 Sites, 10 API-Abrufe/Tag): Sites „PV Ost" und „PV West". ⚠️ **Solcast-Azimut-Konvention: gegen den Uhrzeigersinn positiv!** 0=N, Ost=−90, West=+90 → PV Ost = **−104**, PV West = **+76**. Die Integrations-Warnung „Azimut ungewöhnlich für Nordhalbkugel" bei +76 ist ein False Positive (West-Dach zeigt real WNW, 14° nördlich von West wegen Firstdrehung) — ignorieren.
- HACS: `BJReplay/ha-solcast-solar` v4.5.2 (Domain `solcast_solar`, Entry `01KWVFKHV222SJYGC57BZ0760Q`). Nach Install: HA-Neustart + Browser-Cache! API-Key im Account (einer für beide Sites, Integration findet Sites automatisch).
- Energie-Dashboard-Prognoselinie auf Solcast umgehängt (`config_entry_solar_forecast` an Fronius-Quelle), Kacheln in `energiebilanz-sys` zeigen jetzt `sensor.solcast_pv_forecast_prognose_heute`/`_morgen`. Weitere nützliche Sensoren: `..._verbleibende_leistung_heute` (kWh Rest), `..._aktuelle_leistung`, Peak-Zeiten, `..._verwendete_api_abrufe`.
- **Forecast.Solar läuft parallel weiter** (Einträge + min_max-Helfer `pv_prognose_heute/morgen` zeigen weiter FS-Werte) für den Vergleich; nach dem Sonnentage-Test 9.–11.07. entscheiden und Verlierer entfernen (FS-Einträge + Helfer aufräumen oder Helfer auf Solcast umziehen).

## Electricity Maps / CO2 (2026-07-06) ✅
- Core-Integration „Electricity Maps" (ehem. CO2 Signal), Free Tier (electricitymaps.com/free-tier, 50 Abrufe/h, Zone DE). Sensoren: `sensor.electricity_maps_co2_intensitat` (g CO₂eq/kWh) und `sensor.electricity_maps_anteil_fossiler_energietrager` (%).
- Energie-Dashboard nutzt die Integration automatisch (Grün/Fossil-Split des Netzbezugs, Blatt-Symbol). Zusätzlich `energy-carbon-consumed-gauge` (6 Spalten) in `energiebilanz-sys` hinter den Prognose-Kacheln.
- Erste Werte: 99 g/kWh, 7,45 % fossil (Sonntagmittag). Damit sind vom Vorbild-Board erledigt: Sankey-Grundlage, Geräte (E-Auto+2×SK), PV-Prognose (Solcast), CO2. Offen nur noch: weitere Geräte-Inventur, ggf. großes Energie-Board zusammenstellen.

## victron-sys saniert / Keepalive stillgelegt (2026-07-06) ✅ — „Stufe A"
- **Drei neue kontinuierliche Sensoren** im Node-RED-Leistungs-Tab (Repo `flows/victron-leistung-mqtt.json`, Nodes `clpowA*`): `sensor.victron_system_mppt_ost_leistung` (solarcharger/278 `/Yield/Power`), `..._mppt_west_leistung` (277), `..._fronius_leistung` (zweiter Draht am bestehenden Fronius-Node). Topics `victron/pv/mppt_ost_power|mppt_west_power|fronius_power`, Device „Victron System".
- **Dashboard `victron-sys`** auf kontinuierliche Sensoren umgestellt (SoC-Gauge → `victron_batterie_ladestand`, PV gesamt → `victron_system_pv_leistung`, Einzelgraphen → neue Sensoren); Heading „Umfeld"/Kühlschrank → „PV & Batterie"/mdi:solar-power. Graphen sind ab jetzt lückenlos.
- **Venus-MQTT-Keepalive stillgelegt** (reversibel): beide Automationen aus (`victron_keepalive_ein_bei_dashboard_aufruf`, `..._automatisch_stoppen`), `switch.victron_keepalive_switch` aus, alte MQTT-Sensoren registry-disabled (`batterie_soc`, `fronius_power`, `mqtt_west_power`, `mqtt_ost_power`). Historie bleibt erhalten.
- Rest-Aufräumen (optional, benötigt Zugriff auf HA-configuration.yaml): Template-Sensor `pv_gesamtleistung` (ohne unique_id, nicht im Registry) und ggf. die MQTT-YAML-Definitionen des Keepalive-Setups entfernen; `update.victronmqttkeepalive_update` deutet auf ein HACS-Paket „VictronMQTTKeepalive" hin, das dann auch deinstalliert werden kann.
- **Nachzügler gefunden und migriert:** Haus-Dashboard-Badges nutzten `sensor.batterie_soc` (→ `victron_batterie_ladestand`) sowie `sensor.akku_spannung`/`akku_strom` — letztere waren ENTGEGEN erster Diagnose ebenfalls Venus-MQTT-Keepalive-Sensoren (froren exakt 60 s nach der letzten Keepalive-Probe ein; der WatchMon-Tab decodiert Batrium-UDP nur in Node-RED-Globals, publiziert aber NICHT nach HA). Neue D-Bus-Sensoren `sensor.victron_batterie_spannung`/`..._strom` (Batterie-Tab, Nodes `clbat*`, `/Dc/0/Voltage|Current`, rateLimit 5 s), Badges umgehängt, alte Entities disabled. Merksatz: Nach Stilllegung einer Quelle die Verbraucher über `last_updated`-Frische verifizieren, nicht über plausible Werte.
- Cerbo-ADC: Tank-/Temperatureingänge laut Norbert **nicht belegt** — ignorieren.
- „Stufe B" (System-Gesundheits-Board: Batrium-Zellwerte, CCL/DCL, Multiplus-Status, Netz je Phase, Alarme+Notifications) besprochen, noch nicht beauftragt.

## Batterie-Wirkungsgrad-Gauges (2026-07-06) ✅
- **Klärung:** Batrium `ChargedEnergy/DischargedEnergy` sind Lebenszeit-Zähler (kein „letzter Zyklus"). Batterie: 280 Ah ≈ 15 kWh (vebus `/Dc/0/Capacity`). Anlage ist 3-Phasen-Verbund aus **3× MultiPlus-II** (ein vebus-Service 276).
- **Gauge 1 „Batterie-Wirkungsgrad (DC, Lebenszeit)"**: `sensor.batterie_wirkungsgrad` (Template-Helfer) = entladen÷geladen = **96,8 %**. Misst nur Zellebene, Unschärfe durch SoC-Differenz schrumpft mit wachsenden Zählern.
- **Gauge 2 „PV-Speicher-Nutzgrad (DC → AC)"**: zeigt jetzt `sensor.victron_speicher_ac_roundtrip_inkl_wandlung` (Node-RED-gemessen, s. Abschnitt Speicher-Wirkungsgrad), Skala 70–100, grün ≥ 88. — Früher `sensor.speicher_wirkungsgrad_monat` (AC-Abgabe(Monat) ÷ Batterie-geladen(Monat)); **am 07.07. gelöscht**, weil dieser Quotient die PV-Durchleitung mitzählte und dadurch >100 % zeigte. Die zwei utility_meter `victron_batterie_batterie_geladen_monat` / `victron_multiplus_multiplus_ac_abgabe_monat` sind seither ungenutzt (können bei Gelegenheit entfernt werden).
- **Neue Quell-Sensoren** (Tab „Victron MultiPlus → HA (MQTT)", Repo `flows/victron-multiplus-energie-mqtt.json`, Nodes `clmpx0*`): `sensor.victron_multiplus_ac_abgabe` (56,45 kWh = vebus `Energy/InverterToAcIn1`+`InverterToAcOut`, onlyChanges=false + Summen-Function die auf beide Quellen wartet) und `..._ac_aufnahme` (1,73 kWh = `AcIn1ToInverter`).
- ⚠️ **Zeitbasen-Gotcha:** vebus-Zähler (56 kWh) und Batrium-Zähler (350 kWh) haben verschiedene Start-Epochen — Lebenszeit-Quotienten NUR innerhalb derselben Quelle, quellenübergreifend nur über utility_meter-Fenster!
- **Erkenntnis nebenbei:** Batterie lädt fast ausschließlich MPPT-direkt (nur 1,7 kWh je über AC→DC) — Wandlungsverluste betreffen praktisch nur die Entladerichtung.
- **Nacht-Messung offen:** Morgen früh Δ(Batrium entladen) − Δ(MultiPlus AC-Abgabe) über die Nacht ÷ Stunden = realer Grundverbrauch des 3er-Verbunds in W (Norberts Frage: mehr als 25 W? Datenblatt sagt 60–90 W).

## Begriffe / Hausgeräte
- **„SK" = Split-Klimaanlage** (nicht Stromkreis!). Zwei Stück, per Zigbee-Messsteckdose erfasst: `sensor.0xa085e3fffebc4574_power` (**SK AZ** = Arbeitszimmer/Küche) und `sensor.0xa085e3fffebd16b8_power` (**SK WZ** = Wohnzimmer). Beide als individual-Kreise in der PFCP-Karte (`energie-sys`), Icon mdi:air-conditioner. Ihre kWh-Zähler (`..._energy`) sind Kandidaten für device_consumption im Energie-Dashboard (noch nicht eingetragen).

## Erledigt (Ergänzungen)
- Bereichskarte „KG Heizungsraum" (2026-07-09): Bestehender Bereich `heizungsraum` (Etage Keller) bekam `Bilder/Heizung.webp` als Bild (Data-URI-webp im Area-Registry, ~19 KB) und eine picture-Area-Karte im Dashboard `dashboard-keller` (eigene Sektion, 12 Spalten/4 Zeilen, analog Vorratsraum). Unter-Dashboard `kg-heizungskeller` „KG Heizungskeller" ergänzt (Subview mit `back_path: /dashboard-keller` → Zurück-Pfeil; Karte navigiert dorthin). Inhalt (2026-07-09): Sektion „Brauchwasser-Wärmepumpe" im Stil von `eg-arbeitszimmer` (Norberts Vorgabe nach erstem Tile-Versuch): **Gauge** Momentanleistung (`sensor.0xa085e3fffeb7c870_power`, Zigbee-Messsteckdose „BrauchwasserWP"; 0–2000 W, gelb ab 700, rot ab 1500 — BW-WP-Kompressor ~550 W, Heizstab-Reserve) + **statistics-graph** „Tagesverbrauch" (`..._energy`, change/day, 7 Tage, Balken). Gerät dem Bereich `heizungsraum` zugewiesen. (Ein zwischenzeitlich angelegter utility_meter-Helfer `sensor.brauchwasserwp_tagesverbrauch` wurde wieder gelöscht — der statistics-graph deckt den Tagesverbrauch direkt ab.) BW-WP-Verbrauchserfassung damit gestartet (→ Zeitverschiebungs-Kandidat aus der Wärmewende-Analyse). Zusätzlich als device_consumption „BW-WP (Brauchwasser-Wärmepumpe)" (`sensor.0xa085e3fffeb7c870_energy`) ins Energie-Dashboard eingetragen — Geräteverbrauch zeigt jetzt 4 Geräte: E-Auto, SK AZ, SK WZ, BW-WP. Hintergrund: Wärmewende-Thema (Öl → Split-WP).
- Strompreise im Energie-Dashboard (2026-07-06): Bezug 0,3494 €/kWh, Einspeisevergütung 0,07382 €/kWh.
- Tacho-SVG der Leistung-Kachel: Zeiger auf die grüne Einspeisungs-Seite gedreht (rotate -38°).
- SK AZ + SK WZ (2026-07-06) als device_consumption im Energie-Dashboard eingetragen („SK AZ (Klima Arbeitszimmer)" / „SK WZ (Klima Wohnzimmer)", `sensor.0xa085e3fffebc4574_energy` / `sensor.0xa085e3fffebd16b8_energy`) — Geräteverbrauch zeigt jetzt E-Auto + beide Klimaanlagen. Da die Zigbee-Zähler bereits Langzeit-Statistiken haben, erscheinen auch vergangene Tage rückwirkend im Geräte-Diagramm.
- **Totband 40 W** im EMA-Node der geglätteten Netzleistung (2026-07-06): `if (Math.abs(msg.payload) < 40) msg.payload = 0;` nach der EMA, vor dem Rate-Limit. Schwelle 40 statt der ursprünglich angedachten 25, weil die Nachtdaten Rausch-Ausreißer bis −49 W zeigten (typisch ±30). Unterdrückt das nächtliche Richtungs-Geflacker der Punkte in der PFCP-Karte; echte Ereignisse (±200…600 W Regelspitzen) bleiben sichtbar. Per Admin-API deployed (partial deploy, Node `clpwr0emafunc001`), Repo-Kopie `flows/victron-grid-leistung-glatt.json` synchron. Sichtprüfung: heute Nacht sollte die Karte am Netz-Ast ruhige 0 W zeigen.

## PV-Summe: Phantom-Erzeugung im Energiebilanz-Board gefixt (2026-07-07) ✅ (Statistik-Reparatur offen)
- **Symptom (Norbert):** `energiebilanz-sys` (EFCP) zeigte für 07.07. absurde Solar-Werte; das eingebaute HA-Energie-Board (Einstellungen→Energie) blieb korrekt. Norberts Schlüsselfrage „warum ist HA-Energie ok?" führte zur Auflösung.
- **Warum HA-Board sauber bleibt:** Dort sind die **drei Einzelzähler** direkt als Solarquelle eingetragen (`victron_mppt_1_ertrag`, `_2_ertrag`, `fronius_pv_energie`); HA summiert intern, `unavailable` wird ignoriert (≠ Wertsturz). EFCP nimmt pro Kategorie nur EINE Entität → dafür der Aggregat-Sensor `victron_system_pv_energie_gesamt`. Diese **zusätzliche Node-RED-Aggregationsschicht** ist die fehleranfällige Stelle.
- **Ursache:** Der Aggregat-Sensor sackte 04:28 für 0,2 s von 35951 auf **7044** (= nur mppt1+mppt2, Fronius-Anteil 28908 fehlte) → HA wertete den Wiederanstieg als `total_increasing`-Reset → Tages-`change` 35.952 kWh statt real ~2. **Physischer Auslöser (Norberts Hinweis):** Der **Fronius schaltet nach Sonnenuntergang ab** (D-Bus-Service `com.victronenergy.pvinverter` verschwindet, `sensor.fronius_pv_energie` friert ~21:13 ein). Beim nächtlichen Redeploy/Reconnect lieferte der Fronius-Node leer/0 → `Number(payload)||0` = 0 in die Summe. **Wiederkehrendes Risiko** (jede Nacht Fronius weg), kein Einzelfall.
- **Fix live deployed** (Admin-API `PUT /flow/f8e3c4d5e6f7a803`, Repo-Kopien `victron-pv-energie-mqtt.json` + `victron-pv-summe-mqtt.json` synchron): **(A)** neuer Fronius-Guard-Node `clfrng0guard0001` vor dem retained-mqtt-out — `!Number.isFinite(v) || v<=0 → return null` (retained-Topic behält den letzten guten Wert). **(B)** Summenknoten `clsum0func00001`: `isFinite`-Check statt `||0` (verwirft fehlende Werte, defaultet nicht auf 0) + **Monotonie-Wächter** (letzte Summe in Context, `sum < last → return null`, weil Lebenszeit-Zähler nie fallen). Verifiziert: nach Redeploy läuft die Summe glatt weiter (35953,53 → 35954,39), **kein Dip**.
- **Andere Aggregat-Sensoren geprüft + MultiPlus präventiv gehärtet (2026-07-07):** Systematischer Check aller Node-RED-Summen. Einziger weiterer echter Kandidat (`total_increasing` aus Node-RED-Summe): `sensor.victron_multiplus_ac_abgabe` (Summe InverterToAcIn1+InverterToAcOut, Tab `clmpx0tab0000001`, Node `clmpx0sum000001`) — codegleiche Anfälligkeit (`||0`, kein Wächter), aber **noch nie ausgelöst** (beide Quellen am selben immer-online vebus/276, kein nächtliches Abschalten wie beim Fronius). Präventiv gehärtet: `isFinite`-Check + Monotonie-Wächter (kein Guard-Node nötig, da keine Quelle verschwindet). Statistik war sauber, keine Reparatur nötig. Nicht betroffen: PV-Leistungssumme (`measurement`, kein Reset möglich), AC-Aufnahme/Grid/Batterie/openWB/Einzel-MPPTs (Einzelquellen, keine Summe).
- **Statistik-Reparatur erledigt (2026-07-07, verifiziert):** HA bietet keinen Service `recorder.adjust_sum_statistics` (nur WebSocket hinter Entwicklerwerkzeuge → Statistiken; Service-Call gibt 400 mit irreführendem „entity_id"-Hinweis). **Der UI-Dialog „Eine Statistik anpassen" arbeitet auf 5-Minuten-Zellen; „Neuer Wert" = absoluter Zuwachs (change) DIESER Zelle in kWh — kein Delta, kein Minus, keine Stunde!** Der Phantom saß in der 5-Min-Zelle **04:25–04:30** (change 35.951,55, Glitch um 04:28:35). Korrektur: diese Zelle + eine bei den ersten Versuchen versehentlich verstellte Zelle **04:05–04:10** je auf „Neuer Wert" = **0** (nachts real 0 Ertrag). Ergebnis: Tages-`change` 07.07. jetzt **3,49 kWh** (state 35.955,04) statt 35.952; kumulierte Summe zurück auf 99,69; Rest-Rauschen 1,5 Wh vernachlässigbar. Energiebilanz stimmt wieder.

## Geplant (zurückgestellt bis Herbst): Hochlast-Gauge + Fix/Variabel-Verlustmodell (2026-07-10)

**Anlass:** E3DC-Vergleich („>95 %" = Datenblatt-Peak vs. unsere gemessenen 73,6 % Nacht-Roundtrip). Befund: kein Technik-Nachteil, sondern andere Kennzahl — realer E3DC-Nachtbetrieb liegt laut PV-Forum ebenfalls bei ~80 % und schlechter. Einziger struktureller Nachteil unserer Anlage: Überdimensionierung (3× 10 kVA fürs Whole-home-Backup → ~173 W Fixverlust nachts, davon Eigenverbrauch ~2 kWh/Tag ≈ 83 W).

**Beschlossenes Design (noch NICHT umgesetzt — Hochsommer, Hochlast-Fenster zu selten):**
1. **Hochlast-Gauge:** zweites Akkumulator-Paar (`accAcHi`/`accDcHi`) im Wirkungsgrad-Flow, Gate wie gehabt (PV < 30 W, entlädt) **plus Entladeleistung > 1,5 kW**. Dritter Discovery-Sensor „Entlade-Wirkungsgrad Hochlast" + Gauge in `victron-sys`. Erwartung ~88–91 %. Publikation erst ab > 1 kWh (sonst unknown). **Umsetzung im Generator `tools/gen_eff_flow.py`, nie direkt im Flow-JSON!**
2. **Datenlage:** Sommer dünn (dunkel erst ~22 Uhr; nur späte Spülmaschine/Trockner). Ab Herbst füllt es sich täglich (Kochen bei Dunkelheit + **Split-Klimas im Heizbetrieb 1–3 kW aus dem Akku**) — liefert genau den η am Heizlast-Betriebspunkt für die Wärmepreis-Rechnung. Schwelle bewusst 1,5 kW (datenblatt-vergleichbar); Alternative 1,2 kW (BW-WP+Grundlast fiele rein, aber näher an Mittellast) verworfen. E-Auto aus dem Akku wäre zwar Messfutter, ist aber unerwünschtes Szenario (doppelter Roundtrip, 15 kWh in ~3 h leer).
3. **Fix/Variabel-Verlustmodell statt „Eigenverbrauch herausrechnen":** Bereinigen um angenommene 2 kWh/Tag ergäbe nachts ~84–86 % Wandlungs-η, wäre aber Modell statt Messung (Annahme-Konstante!) und für Wirtschaftlichkeit falsch (Eigenverbrauch ist real bezahlt). Stattdessen: Verlust(P) = P_fix + k·P. **Die zwei Bins liefern zwei Punkte der Verlustgeraden → P_fix und k empirisch lösbar** — dann ist der Eigenverbrauch gemessen statt geschätzt, und der Roundtrip für jedes Durchsatz-Szenario (30-kWh-Ausbau, Winterheizung, Arbitrage) berechenbar. Kern-Einsicht: η ist Funktion des Durchsatzes; 2 kWh Fix bei 5 kWh Nacht-Durchsatz = 29 % Overhead → 73 %; bei 15–20 kWh Winter-Durchsatz ~9 % → 85–88 % von allein.
4. Referenz-Nacht zum Nachrechnen: 4,90 kWh DC-Entladung, 3,62 kWh AC über 7,4 h → 73,9 %; bereinigt 3,62/(4,90−0,61) = 84,4 %.

**Trigger für Umsetzung:** Herbstbeginn / erste Heiztage, oder wenn Norbert es früher will. Aufwand klein (Generator + Deploy + 1 Gauge). Damit erledigt sich auch die offene **Gauge-Schwellen-Frage** (grün ≥ 88): Das bestehende Gauge bleibt der ehrliche Grundlast-Wert, das neue zeigt den vergleichbaren Hochlast-Wert.

## Wandlungsverluste-Zähler: Differenz VRM ↔ HA/openWB erklärt (2026-07-25) ✅ live

**Anlass (Norberts Frage):** VRM zeigte Gesamtverbrauch **18,9 kWh**, HA-Hausverbrauch **23,1 kWh**, openWB **23,7 kWh** — warum die große Differenz zu VRM?

**Befund: kein Messfehler, sondern unterschiedliche Zählerebene.** Beide Zahlen aus den HA-Statistiken exakt nachgerechnet (25.07., Tageswerte: Netzbezug 0,07 / Einspeisung 106,21 / MPPT 27,92 / Fronius 103,70 / Batt +7,70 −5,30 / Multi AC-Abgabe 21,46 − Aufnahme 0,44):
- **HA** (Residual-Bilanz): `0,07 − 106,21 + 131,62 + 5,30 − 7,70 = 23,08` ✓
- **VRM** (rein AC-seitig): `(0,07 − 106,21) + 103,70 + (21,46 − 0,44) = 18,58` ✓
- **Differenz = Wandlungsverluste:** DC in den Multi `27,92 + 5,30 − 7,70 = 25,52` gegen AC heraus `21,02` → **4,50 kWh, η ≈ 82 %**. Gegenprobe: `23,08 − 4,50 = 18,58`. Stabil, nicht zufällig — 24.07.: 32,72 (HA) vs. 27,62 (VRM), Verlust 5,10 kWh.
- **Kern:** MPPT- und Batterie-Zähler sind **DC-seitig**, Gridmeter und Lasten **AC-seitig**. HA kennt keine Kategorie „Verluste" und rechnet den Hausverbrauch als Residuum → alles Nicht-Zuordenbare landet dort. **openWB rechnet dieselbe DC-Bilanz** (darum 23,7 nahe HA, nicht nahe VRM); die 0,6 kWh Rest sind Abtast-/Quellenunterschiede. Deckt sich mit dem bekannten Entlade-Wirkungsgrad-Sensor (74,6 %) und der Fixverlust-Analyse oben (~173 W nachts).
- **Welcher Wert stimmt:** VRM 18,9 = was die Verbraucher wirklich zogen. HA/openWB 23,1 = was das System dafür bereitstellen musste (für Autarkie/Kosten die ehrlichere Zahl, nur falsch beschriftet).

**Gebaut:** `tools/gen_losses_flow.py` → `flows/victron-wandlungsverluste-mqtt.json` (17 Nodes, Tab „Victron Wandlungsverluste (MQTT)", IDs `cllos0*`). Publiziert `victron/losses/conversion_energy` (kWh, retained) + Discovery-Sensor `victron_wandlungsverluste` „Wandlungsverluste (DC → AC)" am bestehenden Gerät **Victron Speicher**.
- **Quellen: ausschließlich vorhandene retained MQTT-Topics** (`victron/pv/mppt1|mppt2/yield`, `victron/battery/charged|discharged`, `victron/multiplus/ac_out_energy|ac_in_energy`) — keine D-Bus-Nodes, nach Import sofort lauffähig, kein Nachkonfigurieren von Services.
- **Delta-Verfahren statt Absolutwerten:** Die Zähler haben verschiedene Nullpunkte/Zeitbasen (Batrium vs. vebus vs. MPPT `/Yield/User`) — nur Zuwächse im selben 300-s-Fenster sind vergleichbar.
- **Wichtigste Design-Entscheidung — nicht pro Tick clampen:** Die Quellzähler springen grob (Batrium 0,1 kWh), einzelne Ticks rauschen ins Negative. Ein `max(0, …)` je Tick würde nur die negativen Rauschticks abschneiden → **systematische Überschätzung** (Rectification Bias). Stattdessen `accDc`/`accAc` **roh** akkumulieren und erst die **Ausgabe** monoton halten (`pub = max(pub, accDc − accAc)`), weil HA für `total_increasing` Monotonie braucht. Simulation: **153 von 288 Ticks** hätten gedippt — der Wächter ist real nötig, nicht theoretisch (vgl. PV-Summen-Phantom vom 07.07.).
- **Robustheit:** Rücksprung/Sprung einer Quelle (Batrium-Reset, stale D-Bus) macht das Fenster ungültig → verwerfen, mit neuem `prev` weiter. Persistenz über retained `victron/losses/_acc` (Akkustand + `prev` + `ts`); `prev` wird nur bei Downtime < 15 min übernommen, sonst Lücke statt Riesen-Delta. „Arm"-Inject nach 15 s, damit der Erststart ohne retained `_acc` nicht blockiert und kein Tick eine 0 über den gespeicherten Stand schreibt. Manueller RESET-Inject vorhanden.
- **Getestet:** `tools/test_losses_flow.js` (Node, ohne Node-RED lauffähig) — simulierter Tag reproduziert 4,48 kWh (erwartet 4,50; Rest = Quantisierung), Monotonie, Zähler-Rücksprung, Restore/Downtime, Anlaufverhalten, Reset. Alle grün.

**Deployed (2026-07-25 21:42):** `POST /flow` auf 192.168.2.80:1880 → Node-RED vergab eine **eigene Tab-ID `b8e14530c32f6796`** (die im Payload mitgegebene `cllos0tab0000001` wurde ignoriert; Node-IDs `cllos0*` blieben erhalten). Generator + Repo-File auf die Live-ID nachgezogen, Abgleich Repo↔Live verifiziert (IDs + Function-Code identisch). Broker-Config `0057f46f75e31fea` ist global (`z=None`) und wurde bewusst **nicht** mitgesendet → keine Duplizierung. Die bekannte Deploy-Gotcha (frischer mqtt-out publiziert beim ersten Mal nicht) trat **nicht** ein — Discovery + erstes Publish kamen sofort durch.
- **Anlauf verifiziert** über `GET /context/node/cllos0func00001`: alle sechs Quellen gefüllt, `prev` gesetzt, `restored: true`. Erster Tick (21:42) nur `prev`-Initialisierung → Sensor 0. Zweiter Tick (21:47) akkumulierte: `accDc 0,10 / accAc 0,05 / pub 0,05`.
- ⚠️ **Einzelne Ticks sind NICHT aussagekräftig:** Der erste Zuwachs war ein Batrium-Sprung (0,1 kWh) gegen 0,05 kWh AC — hochgerechnet absurde 14 kWh/Tag. Das ist reine Quantisierung; erst über Stunden mittelt sich der Wert auf die erwarteten ~4–5 kWh/Tag ein. Nebeneffekt des Monotonie-Wächters: Ein überzeichnender erster Tick bleibt als kleiner Einmal-Offset stehen (hier 0,05 kWh ≈ 1 % eines Tageswerts) — bewusst in Kauf genommen, Alternative wäre der viel schädlichere Rectification Bias.
- **Energie-Dashboard:** per `add_device` eingetragen als „Wandlungsverluste (MultiPlus DC → AC)", 4 → 5 Geräte, die bestehenden vier intakt (vorher `dry_run` geprüft — `save_prefs` ersetzt pro Schlüssel den ganzen Block und löscht sonst still). Die Warnung `statistics_not_defined` direkt nach dem Schreiben war nur das noch fehlende erste Statistik-Intervall; nach dem 5-min-Zyklus (21:40) sind Statistiken da und `energy/validate` meldet **keine** Fehler mehr.
- **Erwartung zur Sichtprüfung:** Der Hausverbrauch im Energie-Dashboard bleibt in der Summe gleich, aber ~4–5 kWh/Tag erscheinen jetzt als eigener Geräte-Posten „Wandlungsverluste" → der Rest entspricht ≈ dem VRM-Wert. Rückwirkend zeigt das Geräte-Diagramm nichts (Zähler startet heute bei 0), anders als damals bei den Zigbee-Zählern mit vorhandener Historie.
- Der Sensor liefert nebenbei die **empirische Gesamtverlust-Zahl** für das Fix/Variabel-Verlustmodell oben.

**Abgrenzung — was der Zähler misst (Norberts Rückfrage 2026-07-25):** ausschließlich die Verluste **im MultiPlus**, also zwischen DC-Bus (MPPT + Batterieklemmen) und AC-Klemmen:
- ✅ **enthalten:** Wandlung DC→AC (Wechselrichten) und AC→DC (Netzladen); **Eigenverbrauch/Standby der 3 MultiPlus** (Elektronik/Lüfter, 24/7, ~173 W — dominiert nachts); kleine DC-Leitungsverluste.
- ❌ **NICHT enthalten: der Batterie-Roundtrip.** Batrium misst `charged`/`discharged` beide **an den Klemmen**; der chemische Verlust zeigt sich darin, dass `discharged` kleiner ausfällt — diese Energie erreicht den Wechselrichter nie und taucht in `accDc` gar nicht erst auf. Sie fehlt der Bilanz, statt als Wandlungsverlust gezählt zu werden. Den DC-Roundtrip misst weiterhin `sensor.batterie_wirkungsgrad` (96,9–98,4 %).
- ❌ ebenfalls draußen: Fronius-Wandlung (AC-seitig gemessen, Verluste liegen davor), MPPT-Wandlung (Zähler sitzt schon auf der DC-Ausgangsseite), hausinterne Leitungsverluste.
- **Merksatz:** Der Zähler misst die Wärme, die die drei MultiPlus abgeben — und exakt den Betrag, um den HA mehr „Hausverbrauch" ausweist als VRM.

⚠️ **VRM zeigt den Tagesverbrauch an zwei Stellen mit unterschiedlicher Genauigkeit** (2026-07-25, 22:40): eine Ansicht gerundet („20 kWh"), eine genauer („19,7 kWh"). Die gerundete Zahl ergab eine scheinbar **schrumpfende** Differenz zu HA (4,2 → 3,8) — physikalisch unmöglich, da Verluste kumulativ sind. **Beim Vergleich immer die Ansicht mit Nachkommastelle nehmen.** Ein sinkender Verlust-Betrag über den Tag ist immer ein Ablese-/Rundungsartefakt, nie eine echte Messung.
**Datenblatt-Abgleich MultiPlus-II 48/10000/140-100 (2026-07-25):** Werte aus dem offiziellen Victron-Datenblatt (8k/10k/15k, AUS-Ausgabe — gleiche Hardware):

| | pro Gerät | ×3 |
|---|---|---|
| Nulllast normal | 38 W | **114 W** |
| Nulllast AES | 27 W | 81 W |
| Nulllast Search | 4 W | 12 W |
| max. Wirkungsgrad | 96 % | — |

> ⚠️ **Die folgenden fünf Punkte sind in der Aufteilung überholt (12.08.2026).** Der Leerlaufsockel ist inzwischen per Regression gemessen: **174–200 W statt 114 W**. Die Zerlegung unten unterschätzt daher den Standby-Anteil (real 80–92 % statt 60 %) und überschätzt die Wandlungsverluste. → Abschnitt **„Leerlaufverlust: Stand der Messungen"** am Dateiende.

- **114 W × 24 h = 2,74 kWh/Tag reiner Standby** — rund **60 % des gemessenen Tagesverlusts von ~4,6 kWh**. Der Löwenanteil ist also Eigenverbrauch, nicht Wandlung.
- **Saubere Zerlegung des Tages** (26,2 kWh DC-Durchsatz): Standby 2,74 kWh + Wandlung ~1,9 kWh (≈93 % effektiv, plausibel unter dem 96-%-Peak) = **4,6 kWh** ✓ deckt sich mit der Messung.
- **Messungen streuen um den Datenblattwert:** Abend 25.07. (538 W Durchsatz, 1,3 h) ≈ 100 W gemessen vs. ~136 W erwartet; Referenz-Nacht (662 W, 7,4 h) 173 W vs. ~140 W erwartet. Kurzmessungen < 2 h sind wegen der 0,1-kWh-Batrium-Stufung nicht auflösend genug — nur Mehrstunden-Fenster taugen.
- **Für das Fix/Variabel-Verlustmodell:** `P_fix` liegt jetzt aus zwei Quellen vor — eigene Schätzung 173 W, Datenblatt 114 W. Realistisch dazwischen (Laborwert optimistisch; im ESS-Netzparallelbetrieb regelt der Multi permanent nach, Search/AES praktisch nicht wirksam).
- **Kostenbezifferung der Überdimensionierung:** 114 W Dauerlast ≈ **1000 kWh/Jahr ≈ 350 €** (bei 34,94 ct) — der bezifferte Preis der Whole-home-Backup-Fähigkeit, unabhängig vom Energiedurchsatz.

- **Restabweichung ~0,5 kWh/Tag (2,5 %) zwischen VRM und der Nachrechnung ist methodisch, kein Fehler:** VRM integriert laufend Momentanleistungen (W → Wh), die Nachrechnung nutzt die vebus-Lebenszeitzähler (`/Energy/InverterToAcIn1` etc.), die der MultiPlus nur in groben Schritten fortschreibt. Über einen Tag summiert sich das zu einem kleinen systematischen Versatz — dieselbe Ursache wie beim bekannten Zeitbasis-Hinweis im MultiPlus-Flow.
### ⛔ EFCP-Karte: harte Grenze von 4 individual-Geräten (2026-07-25, im Code verifiziert)

Versuch, die Verluste als fünften `individual`-Eintrag in `energiebilanz-sys` aufzunehmen, **scheiterte** — Norbert sah E-Auto/SK AZ/SK WZ/BW WP, aber keine Verluste. Ursache im installierten Kartencode gefunden (`/config/www/community/energy-flow-card-plus/energy-flow-card-plus.js`, HACS v0.2.3 = neueste, kein Update verfügbar):

```js
["left-top","left-bottom","right-top","right-bottom"][i]
```

**`energy-flow-card-plus` hat genau VIER Positionen für individual-Geräte**, adressiert über den Array-Index. Ab dem 5. Eintrag liefert das Positions-Array `undefined` → der Posten wird **stillschweigend nicht gerendert**, ohne Fehler oder Warnung. Kein Cache-Problem, Reload hilft nicht.

- Die Migrations-Warnung im Code („`individual1`/`individual2` sind jetzt ein einzelnes `individual`-Array") ist irreführend: Das Array-Schema stimmt, aber die **Anzahl bleibt auf 4 begrenzt**. Norberts 4 Plätze waren voll.
- Der wirkungslose 5. Eintrag wurde wieder entfernt, Dashboard-Config-Hash zurück auf den Ausgangswert `09984b22239aaa08` (Config = das, was tatsächlich dargestellt wird).
- **Diagnoseweg zum Merken:** Bei „Custom Card zeigt Eintrag nicht" direkt in die installierte JS greppen (SSH auf die HA-VM, `/config/www/community/<card>/`) statt Cache/Config zu vermuten — Positions-Arrays und Slices sind auch minifiziert gut auffindbar.

**Gelöst (Norberts Wahl: eigene Karte):** `statistic`-Karte in derselben Sektion, eingefügt **vor** der Flusskarte neben den `energy-carbon-consumed-gauge` (der belegt nur 6 der 12 Spalten — die Lücke war schon da, kein Layout-Umbau):

```yaml
type: statistic
entity: sensor.victron_speicher_wandlungsverluste_dc_ac
stat_type: change
period: energy_date_selection
name: Wandlungsverluste
icon: mdi:flash-alert
grid_options: {columns: 6}
```

- **`period: energy_date_selection` ist der Schlüssel** (aus der Card-Doku, nicht aus dem Gedächtnis): Die Karte hängt sich an den `energy-date-selection`-Wähler oben in der View — wechselt Norbert auf „gestern"/„diese Woche", zieht die Verlust-Zahl mit, synchron zur Flusskarte. Ein festes `period: {calendar: {period: day}}` wäre entkoppelt gewesen.
- **`stat_type: change` ist zwingend**, weil der Sensor ein Lebenszeit-Zähler (`total_increasing`) ist — eine `tile`-Karte hätte den kumulierten Gesamtstand gezeigt (heute 0,07 kWh, in einem Jahr ~1600 kWh), nicht den Tagesverlust. Gleiche Logik wie beim BW-WP-`statistics-graph`.
- Einfügung per `python_transform` **relativ zur EFCP-Karte gesucht** statt an fixem Index — überlebt Umsortieren.
- Dashboard-Config-Hash jetzt `a2217486d9b46d4b`. Flusskarte unverändert bei ihren 4 individual-Einträgen.
- Optionaler Ausbau später: zusätzlich ein `statistics-graph` (change/day, 7 Tage, Balken) nach dem BW-WP-Muster, um den Tagesverlauf der Verluste zu sehen.

⚠️ **Topologie-Vorbehalt (gilt unabhängig davon):** EFCP hat keine Verlust-Kategorie, ein individual-Posten hängt optisch als *Verbraucher am Haus-Ast*. Physikalisch entstehen die Verluste aber **vor** dem Haus (im MultiPlus zwischen DC und AC) — sie müssten am Batterie-/Solar-Ast abgehen. Bilanziell wäre der Abzug korrekt, die Position im Flussbild aber eine Vereinfachung. Beim Erklären (Unterricht!) dazusagen.

## Offen
- **Grafana „BW-WP Heizungskeller": Reset-Tag 12.07. bleibt ohne Tagesbalken** — der Zähler-Reset durchs Firmware-Update (7,41 → 0, 13:13 Uhr) lässt `difference(nonNegative: true)` den Tag verwerfen; Datenpfad HA → InfluxDB → Grafana nachweislich intakt, ab 13.07. Selbstheilung. Entscheidung offen: Panels auf reset-festes Muster umbauen (erst Punkt-Differenz, dann Tages-Summe) — Diagnose, Nachrechnung und Query-Muster in [[homelab-monitoring]] (Session 2026-07-12 + Stolperfalle „Zähler-Reset reißt ein Tagesloch").
- ~~**BW WP in der EFCP-Energiebilanz zeigt 840 Wh statt 1,26 kWh**~~ ✅ ERLEDIGT 2026-07-18: Nach Norberts Update zeigt die EFCP-Karte wieder exakt die Statistik-Werte (Sektion „EFCP-Anzeige-Bug behoben").
- **Prognose-Vergleich bis ~11.07.** (Solcast ist seit 06.07. primär): Tages-Ist (`victron_system_pv_energie_gesamt`) vs. Solcast (`solcast_pv_forecast_prognose_heute`) vs. Forecast.Solar (min_max-Helfer `pv_prognose_heute`) — heute Abend erster Datenpunkt (Solcast 104,4 / FS 52,3 / VRM 89–109). Nach den Sonnentagen 9.–11.07. (Met.no: sonnig, Klartag ≈ 130–150) Verlierer entfernen: FS-Integrationseinträge + ggf. Helfer auf Solcast umziehen oder löschen.
- **Odenwald-Verdacht** (Horizont Ost 4–6°, ~30–45 min späte Morgensonne, geschätzt 2–5 % Ost-Tagesertrag) gilt auch für Solcast: nach 1–2 Wochen prüfen, ob Ist morgens systematisch unter Prognose startet. Solcast Hobbyist hat allerdings keine Damping-Option — wäre dann nur dokumentierte Erwartungskorrektur.
- **Tages-Verifikation** (Rest): E-Auto-Balken nach erster Ladung; PFCP Netz-Richtung visuell bestätigen (Konvention stimmt laut Doku überein: Victron positiv=Bezug = PFCP-Erwartung; aktuell Einspeisung, Punkte müssen Haus→Netz laufen). ✅ Erledigt am 2026-07-06: PV-Zähler + PV-Summe liefern, Batterie „Geladen" liefert, PFCP-Batterie-Richtung in beiden Richtungen verifiziert (`invert_state: true` bleibt, siehe unten), EFCP-Batterie-Zuordnung (consumption=entladen) per Grid-Analogie bestätigt.
- Optional: CO2 Signal (Grün/Fossil + Carbon-Gauge), Forecast.Solar (PV-Prognose), weitere Geräte-Inventur — für das große Energie-Board nach Vorbild-Screenshot.
- Kosmetik: Anzeigename doppelt „Victron" („Victron VM-3P75CT Victron Netzbezug") — bei Gelegenheit Discovery-Namen kürzen (Grid-Flow).
- **rateLimit der Victron-Nodes greift nicht** (Spannung/Strom/Leistung updaten 1×/s trotz rateLimit 5000): Semantik prüfen (evtl. nur mit onlyChanges=false wirksam?) oder Delay-Nodes (rate 1/5s, drop) nachrüsten — Recorder-Volumen. Anzeige-Präzision im Registry gesetzt: Spannung 2, Strom 1 Nachkommastellen.

## Dashboard-Kosmetik „PV & Batterie" (2026-07-25) ✅

**Anlass:** Norbert wollte die Sensor-Karten der Sektion „PV & Batterie" im Dashboard `victron-sys` (View „Victron (Sys)") lesbarer beschriften — die Card-Titel zeigten bisher den vollen Entity-Friendly-Name inkl. Geräte-Präfix.

- Vier `sensor`-Karten (Entities `sensor.victron_system_fronius_leistung`, `..._mppt_west_leistung`, `..._mppt_ost_leistung`, `..._pv_leistung`) per `name`-Override umbenannt: Präfix „Victron System" entfernt, „Leistung" → „(P)".
- Endstand: **Fronius (P)**, **MPPT West (P)**, **MPPT Ost (P)**, **PV Σ (P)** (bei PV zusätzlich Summenzeichen Σ eingefügt, da dieser Wert die Gesamt-PV-Leistung ist).
- Nur kosmetisch (Card-`name`), Entity-IDs/Discovery unverändert — keine Auswirkung auf Statistik oder Energie-Dashboard.
- Umsetzung direkt über HA-MCP (`ha_config_set_dashboard`, `python_transform`), nicht über die Flow-JSON-Generatoren (betrifft nur Lovelace-Storage, keinen Node-RED-Flow).

## Wassersensor-Leck-Alarm (2026-07-25) ✅

**Ziel:** Push-Nachricht aufs Handy, wenn der Aqara Water Leak Sensor („Wasser Sensor", Zigbee2MQTT, `binary_sensor.0x00158d008bbda3f9_water_leak`) ein Leck meldet — analog zum bestehenden Batrium-Schütz-Alarm.

**Design-Entscheidung:** Bestehende Pushover-Pipeline auf dem Tab „WatchMon" **wiederverwendet statt dupliziert** — kein neuer Tab, kein zweiter Pushover-Token. Neue Nodes (`clwtr0*`) verdrahten direkt in den vorhandenen Knoten „Pushover-Request bauen" (`clbtrmpush00001`, siehe `flows/batrium-schuetz-pushover.json`).

- **`clwtr0mqtt00001`** (mqtt in) — Topic `zigbee2mqtt/Wasser Sensor` (aus HA-`mqtt_topic_hint` abgeleitet, Datatype `json`), Broker „Proxmox Mqtt" (`0057f46f75e31fea`).
- **`clwtr0func00001`** (function „Wassersensor-Wächter (Leck)") — reagiert nur auf *Statuswechsel* von `water_leak` (Context-Dedupe wie beim Schütz-Wächter, kein Spam bei jeder MQTT-Nachricht): `true` → 🚱 Alarm Prio 2 (Emergency, Sirene, Retry 60 s/Expire 1 h); Rückkehr auf `false` → ✅ Entwarnung Prio 0. Kein Alarm beim Node-RED-Start, außer der Sensor ist zu dem Zeitpunkt bereits nass.
- **Bewusst minimal gehalten** (Norberts Wahl): nur Push, **kein** zusätzliches retained MQTT/HA-Status-Topic, keine Dashboard-Karte — anders als beim Batrium-Alarm, der zusätzlich Status+Discovery nach HA spiegelt.
- Deployed per Node-RED Admin-API (`PUT /flow/703f4458.ff2d1c`, Tab „WatchMon"). Repo-Kopie der 3 neuen Nodes: `flows/wassersensor-leck-pushover.json` (die gemeinsam genutzten Pushover/HTTP-Nodes bleiben in `batrium-schuetz-pushover.json`, nicht dupliziert).
- **Kleiner Deploy-Fehler selbst gefunden+korrigiert:** Erster Deploy landete versehentlich mit 🚩 (Flag) statt 🚱 (Kein-Trinkwasser-Symbol) im Titel (Surrogate-Pair-Verwechslung `🚩` vs. `🚱`) — per zweitem PUT korrigiert, verifiziert.
- **Live verifiziert (2026-07-25):** Norbert hat den Sensor real ausgelöst — Push kam an. Topic `zigbee2mqtt/Wasser Sensor` war korrekt, keine weitere Korrektur nötig.

## AC-Lasten-Prüfung: Addition, Saldierung, Grundlast, BW-WP-Stillstandsverlust (2026-07-26) ✅

**Anlass:** Norberts Verdacht „meine AC-Lasten werden falsch addiert". Prüfung direkt am Cerbo (`192.168.2.181`) per D-Bus über TCP (`dbus-send --bus=tcp:host=...,port=78`) — funktioniert von Norberts Arbeitsrechner aus, nicht nur aus dem Node-RED-LXC. Nützlich für Ad-hoc-Diagnose ohne Node-RED-Umweg.

### 1. AC-Lasten werden korrekt addiert — kein Fehler

GX-Formel: `AC-Lasten = Netzzähler − MultiPlus_AcIn + PvOnGrid`. In zwei Betriebszuständen gegengerechnet:

- **Nachts:** Netz −18 W, Multi AcIn −462 W → berechnet 444 W, GX zeigt 524 W (Restdifferenz = Sampling-Versatz zwischen zwei D-Bus-Abfragen). Bilanzprobe mit Batterie −671 W bei η≈0,78 geht auf.
- **Mittags (25.07. 13:55):** Netz −14.280 W, Fronius +12.082 W, MPPT 3.307 W DC → AC-Lasten ≈ 1 kW. Plausibel, kein Doppelzählen.
- **Pro Phase live:** Netz L1 +63/L2 −38/L3 −43, Multi AcIn L1 −175/L2 −156/L3 −141 → AC-Lasten L1 244/L2 120/L3 131. Minuswerte auf L2/L3 sind korrekt: der Multi speist gleichmäßig dreiphasig ein, die Hauslast hängt ungleich verteilt dran.

**Topologie bestätigt:** `/Settings/Fronius/Inverters/I114_2060671/Position = 0` (= AC-Eingang 1). Der Fronius sitzt hinter dem VM-3P75CT, wird also einmal weggerechnet und einmal wieder addiert → hebt sich auf. Stünde Position auf 1 (AC-Ausgang), erschienen mittags ~12 kW Phantomlast.

`PvOnGrid` ist nachts *invalid* (nicht 0), weil der Fronius komplett vom Netz geht und der `com.victronenergy.pvinverter`-Service vom Bus verschwindet. Normal, kein Fehler.

### 2. VM-3P75CT saldiert korrekt — mit Zählerstands-Beweis

Der Zähler führt Phasenzähler **und** Summenzähler. Wären die Summen bloße Additionen, müssten sie gleich sein. Sind sie nicht:

```
           L1        L2        L3     Σ Phasen   Summenzähler   Differenz
Bezug    1543.88  1549.46   831.75    3925.09      2660.83     1264.26
Einsp.   8867.74  9033.28 10196.10   28097.12     26832.83     1264.29
```

**Beide Differenzen identisch (1264,3 kWh, Abweichung 0,03 kWh = 0,002 %)** — das ist die Signatur echter Saldierung: Energie, die zeitgleich auf einer Phase bezogen und auf einer anderen eingespeist wurde, hebt sich im Summenzähler auf. Netto-Saldo auf beiden Rechenwegen −24172,0 kWh.

**Wichtig für die Flows:** `flows/victron-grid-energie-mqtt.json` liest `/Ac/Energy/Forward` und `/Ac/Energy/Reverse` — die **saldierten Summenzähler**, nicht `/Ac/L1..L3/Energy/*`. Hätte der Flow die Phasenzähler summiert, stünden 3925 statt 2661 kWh Bezug im Energie-Dashboard (**+48 %**). Passt außerdem zum saldierenden EVU-Zweirichtungszähler. Bei allen künftigen Dreiphasen-Zählern dieselbe Falle prüfen.

**Kosmetik-Falle:** Die `Consumption/Lx/Current`-Werte des GX sind Unsinn (231 W bei angeblich 4,03 A / 238 V). Der GX subtrahiert Ströme als vorzeichenbehaftete Skalare, was bei unterschiedlichen Phasenlagen unzulässig ist. **Nur den Watt-Werten trauen, nie den Ampere-Werten.**

### 3. HA-Dashboard liegt strukturell ~5 kWh/Tag über den GX-AC-Lasten

Unterschiedliche Definitionen, kein Fehler:
- HA: `Netz + PV(MPPT+Fronius) + Batt_entladen − Batt_geladen` — nimmt den **DC**-Ertrag der MPPTs, als käme er verlustfrei auf AC an.
- GX: `Netz + Fronius + Multi_AC-Abgabe − Multi_AC-Aufnahme` — nimmt, was der MultiPlus real AC-seitig liefert.

7-Tage-Vergleich (19.–25.07.): HA 185,11 kWh vs. GX 149,53 kWh → **Differenz 35,58 kWh = +23,8 %**. Pro Tag konstant **5,0–5,2 kWh**, unabhängig davon, ob das E-Auto lud (23.07.: 32,5 kWh) — die Konstanz beweist Offset, nicht Additionsfehler. Multi-Wirkungsgrad daraus 80–82 % (niedrig, aber plausibel: 3× 8300 VA Nennleistung bei ~1 kW mittlerer Last → Leerlaufanteil dominiert). Der Sensor `victron_speicher_wandlungsverluste_dc_ac` fängt genau diese Größe ab und lief korrekt an (0,07/0,18/0,16 kWh in den ersten Abendstunden).

### 4. Grundlast-Bilanz — 281 W ungemessen

62 echte Nachtstunden (Netz < 20 W, reiner Batteriebetrieb, 19.–26.07.), AC-Last rekonstruiert aus `−Batterieleistung × 0,78` (η aus zwei Live-Messpaaren):

```
Minimum 342 W | 25 % 387 W | Median 506 W | 75 % 632 W | Max 1245 W (BW-WP-Takte)
```

Ruhigster Tag (20.07.): 10,70 kWh AC-Lasten = **446 W Tagesmittel** — praktisch gleich dem Nacht-Median. **Das Haus hat kaum ein Lastprofil, die Grundlast IST der Verbrauch.** Hochrechnung ~4.400 kWh/Jahr.

Bilanz des ruhigsten Tages über **Tagesenergie** (nicht Momentanwerte!):
```
AC-Lasten gesamt        10.70 kWh   446 W
  BW-WP                  3.01 kWh   125 W
  Gefrierschrank         0.42 kWh    17 W
  Kühltruhe Keller       0.33 kWh    14 W
  Klima Arbeitszimmer    0.20 kWh     8 W
  gemessen               3.96 kWh   165 W
  NICHT gemessen         6.74 kWh   281 W
```

Norberts Geräteliste (2 Kühlschränke ungemessen, Lenovo-Server 24/7, Cerbo, openWB/RPi2, Zigbee-Stick) deckt ohne PC nur **74–161 W** der 281 W ab. **Lücke 120–207 W.** Nicht genannt, aber zwangsläufig vorhanden: Netzwerk (Router/Switch/APs, 20–50 W), Batrium-BMS (5–15 W), Fronius-Nachtverbrauch, die Zigbee-Geräte selbst (30 Stück × 0,3–1 W), Standby-Kram. Der MultiPlus zählt **nicht** dazu (DC-seitig = die 5 kWh Wandlungsverluste, nicht doppelt zählen).

**Freie Messhardware:** `sensor.steckdose_alexa_power` und `sensor.bad_power` melden über 7 Tage durchgehend Mittel 0,00 und Max 0,00 W → nichts dran oder aus. Kandidaten zum Umstecken an die beiden Kühlschränke (größter blinder Fleck).

**Wirtschaftlichkeit ehrlich:** Norbert ist massiver Netto-Einspeiser (Juli 2026: 2,99 kWh Bezug gegen 1.917,61 kWh Einspeisung). Grundlast-Einsparung wird daher **nicht** mit 0,3494 €/kWh Bezugstarif bewertet, sondern mit 0,07382 €/kWh Einspeisevergütung. 100 W weniger = 876 kWh/Jahr = **64,67 € (Sommer)** bis 306 € (wenn alles Winterbezug wäre). HA-Statistik reicht nur bis Juli 2026 zurück → **Winteranteil nicht bestimmbar**, realistisch 80–130 €/Jahr pro 100 W. Der bessere Grund ist Batteriedurchsatz + Winter-Netzbezug, nicht die Sommer-Vergütung.

### 5. BW-WP: 54 % des Stroms halten nur die Temperatur

Taktprofil 25.07. (Tag mit lückenloser Erfassung, siehe Stolperfalle unten): **7 Takte à ~50 min bei konstant 510–660 W**, Abstände streng 3,0–3,9 h **rund um die Uhr**, inklusive 02:06 und 23:08.

- **Heizstab läuft nicht** — der zöge 1500–2000 W. Reiner Wärmepumpenbetrieb, kein Defekt.
- Rekonstruktion: 6,25 h × 580 W = 3,63 kWh gegen Zähler 3,61 kWh (**0,5 % Abweichung** → Erfassung an dem Tag vollständig, Analyse belastbar).
- **Streuung über 16 Tage nur 11 %** (Mittel 3,19 kWh, σ 0,35). Echter Warmwasserbedarf eines 4-Personen-Haushalts streut 25–40 %. Ein Verbrauch, der kaum auf das Zapfverhalten reagiert, wird nicht vom Zapfen bestimmt.
- **Nacht gegen Tag:** 23:57–05:57 = 1 Takt in 6 h = 0,081 kWh/h; 05:57–23:57 = 6 Takte in 18 h = 0,161 kWh/h. Nachts halb so oft — aber sie läuft. Zweite Nacht (auf 23.07.) unabhängig gegengerechnet: **0,087 kWh/h**, bestätigt.

```
Stillstandsverlust       1,93 kWh/Tag el. = 5,8 kWh th. = 242 W dauerhaft
Echte WW-Nutzung         1,68 kWh/Tag el. = 5,0 kWh th. = ~108 l bei 40 K
-> 54 % des BW-WP-Stroms halten nur die Temperatur
```

Der **54/46-Split ist COP-unabhängig** (Verhältnis zweier Taktraten); nur die absoluten kWh-thermisch- und Liter-Zahlen hängen an der COP-3-Annahme. Guter 300-l-Speicher verliert 60–100 W → Norbert liegt **Faktor ~2,9 darüber**. 108 l/Tag für 4 Personen ist eher **wenig** (typisch 120–200 l) — die Personenzahl entlastet die Zahl also nicht, sie belastet sie.

**Ursache (von Norbert bestätigte Randbedingungen):** ungedämmte Rohre vorhanden, Zirkulationspumpe **neu** und **bedarfsgesteuert**. (Die Ursache wurde später widerlegt — verkehrte Pumpenrichtung, siehe [[Brauchwasser-Wärmepumpe]].) Damit ist die Pumpe als Verursacher raus — die 242 W wurden nachts gemessen, während sie steht. Bleibt **Schwerkraftzirkulation** durch die stillstehende Zirkulationsleitung und/oder Einrohrzirkulation im blanken Vorlauf.

Rechnerische Gegenprobe bei 55 °C Wasser / 18 °C Keller (ΔT 37 K): blankes DN15 27,8 W/m, DN20 33,3 W/m, DN25 40,7 W/m → **6–9 m warmes blankes Rohr erklären die 242 W vollständig**. Gilt aber nur bei *aktiver Strömung* — stehendes Wasser im Rohr kühlt einmal ab und isoliert dann. Dass die 242 W dauerhaft anliegen, beweist Bewegung im Rohr trotz stehender Pumpe.

**Wichtige Feinheit für die Sanierung:** Ein einfacher **Rückflussverhinderer stoppt Schwerkraftzirkulation nicht** — die läuft in *derselben* Richtung, in die die Pumpe fördert, für ein Rückschlagventil also in Durchlassrichtung. Nötig ist eine **Schwerkraftbremse** (federbelastet, definierter Öffnungsdruck ~20–30 mbar; Auftrieb liegt darunter, Pumpe überwindet ihn mühelos). Bei Einrohrzirkulation im Vorlauf hilft auch die nicht — dort nur eine nach unten gezogene Rohrschleife („Schwanenhals") direkt am Speicherausgang plus Dämmung.

**Maßnahmenplan (Reihenfolge zählt):** erst Schwerkraftbremse/Schleife (kappt den Kreis), dann dämmen — speichernah zuerst (höchstes ΔT). Nur dämmen verlangsamt den Verlust, beseitigt ihn nicht. Erwartung: 242 → ~100 W = **1,14 kWh/Tag = 415 kWh/Jahr**, Material ~50 € Dämmung + ~30 € Bremse. In Geld bei Einspeiser-Situation nur ~30–60 €/Jahr; der eigentliche Gewinn sind die entfallenden Nachttakte (02:06 + 23:08 ≈ 1 kWh/Tag aus der Batterie) und ~400 kWh weniger Winter-Netzbezug.

**Offener Test (kostenlos, entscheidet die Ursache):** spätabends bei stehender Pumpe und ohne Zapfung die Rohre anfassen. Noch mehrere Meter vom Speicher entfernt warm → Schwerkraftzirkulation bestätigt. Nur die ersten 30–50 cm warm → Rohre sind es nicht, dann sitzt der Verlust in der Speicherdämmung. **Zweiter, annahmefreier Test:** an einem Tag, an dem alle weg sind, ist der BW-WP-Tageswert direkt der reine Stillstandsverlust.

### ⚠️ Stolperfalle: Zigbee-Meldelücken erzeugen Phantom-Dauerläufe

Beim Gegenprüfen zweier weiterer Nächte erschienen 4-Stunden-Dauerläufe der BW-WP. **Artefakt.** Die Zigbee-Steckdose meldet nur bei Wertänderung; bleibt sie stumm, hält HA den letzten Wert:

```
23.07. 00:33 -> 03:39   186 min stumm  (letzter Wert: 2 W)
23.07. 14:53 -> 17:18   145 min stumm  (letzter Wert: 2 W)
24.07. 02:30 -> 04:59   149 min stumm  (letzter Wert: 577 W)
```

Takterkennung über die Leistungskurve zog daraus einen 255-min-Lauf. Gegenprobe am **Energiezähler**: 24.07. hatte 3,70 kWh, der „Dauerlauf" allein wäre 2,42 kWh gewesen — geht nicht auf. Umgekehrt können bei Stille auf 2 W auch **ganze Takte unbemerkt bleiben**.

**Regel:** Bei Zigbee-Steckdosen ist der **Energiezähler die Ground Truth, nicht die Leistungskurve.** Leistungsbasierte Analysen immer gegen `change` des Energiezählers gegenrechnen — stimmt die Summe auf wenige Prozent, war die Erfassung lückenlos (25.07.: 0,5 % → belastbar), sonst verwerfen.

**Für Norbert praktisch:** Der Gauge „Momentanleistung" auf dem Dashboard `kg-heizungskeller` kann **stundenlang veraltete Werte** zeigen (z. B. 577 W bei längst abgeschaltetem Gerät). Dem `statistics-graph` daneben (Tagesverbrauch, `change`) kann man trauen. Ggf. eine `availability`-/Alterungsprüfung am Gauge ergänzen — siehe auch die bekannte Falle „HA-Template-Helper überspringt State-Update bei leerem Render" in [[homelab-monitoring]].

### Offen aus dieser Session
- **Rohr-Anfass-Test** durch Norbert (entscheidet Schwerkraftzirkulation vs. Speicherdämmung).
- **Zwei freie Zigbee-Dosen an die Kühlschränke** umstecken (`steckdose_alexa`, `bad`).
- **Sicherungskreis-Test** zur Eingrenzung der 120–207 W unerklärter Grundlast (Batterieleistung nachts als Anzeige reicht, kein neuer Sensor nötig).
- **Angeboten, noch nicht gebaut:** AC-Lasten-Flow (`/Ac/Consumption/L1..L3/Power` → MQTT → HA), damit die GX-Größe in der Historie liegt statt nur per D-Bus abfragbar. Der ursprünglich mit angebotene „Grundlast-Sensor" wurde **verworfen** — sobald der AC-Lasten-Sensor existiert, ist das nächtliche Minimum ein `statistics`-Helper (`characteristic: value_min`, `max_age: 24h`) direkt in HA, kein Flow nötig. Datenbeschaffung aus dem D-Bus gehört nach Node-RED, statistische Ableitung aus vorhandenen Sensoren nach HA.

## Versuchsaufbau: BW-WP-Stillstandsverlust — Stufentest Pumpe → Hahn (2026-07-26)

→ Verschoben nach [[Brauchwasser-Wärmepumpe]] (23.08.2026).

## Idee „nachts kleiner MultiPlus 5000, tags die 3× 10000" — geprüft, ⛔ so nicht umsetzbar (2026-07-26)

**Anlass:** Norbert hatte eine Google-/Gemini-Antwortkette recherchiert, die vorschlägt, die drei MultiPlus-II 10000 nachts per Node-RED in Standby zu schicken und einen **vorhandenen MultiPlus 5000** (Hardware ist da!) die Nacht-Grundlast übernehmen zu lassen — Ziel: den Fixverlust von ~114–173 W loswerden. Bitte um Prüfung, ob das geht.

**Ergebnis: Die Zielrichtung ist richtig, die vorgeschlagene Umsetzung funktioniert nicht.** Ein harter Blocker, plus ein betrieblich gefährlicher Ratschlag darin.

### ⛔ Der Showstopper: ein GX-Gerät regelt nur EIN VE.Bus-System

Die Google-Antwort behauptet, der Cerbo erkenne dann „zwei Systeme" (3-phasig + 1-phasig) und man könne beide getrennt ansprechen. **Erkennen ja, regeln nein.** Victron-Doku (Cerbo/Venus GX, „Connecting Victron products"):

- Am eingebauten VE.Bus-Port darf **genau ein** VE.Bus-System hängen.
- Ein zweites System über **MK3-USB** erscheint in der Geräteliste und zählt in den VRM-kWh-Statistiken mit, aber: **„DVCC and ESS control applies only to the system connected directly to the built-in VE.Bus ports."** Zweitsysteme folgen ihrer internen Konfiguration.

→ Der 5000er am MK3-USB bekäme **keinen Grid-Setpoint, keine Nullausregelung, keine DVCC-/BMS-Limits**. Genau die Funktion, um die es geht, fehlt. (Neuere Firmware hat lediglich *limited CCL control* für ein zweites VE.Bus-System ergänzt — Ladestrombegrenzung, nicht ESS-Regelung.)

**Theoretischer Weg, von dem ich abrate:** ein **zweites GX-Gerät** (Raspberry Pi mit Venus OS, ~60 €) exklusiv für den 5000er. Dann bräuchte es dort aber (a) einen Zähler-Feed — der VM-3P75CT hängt am Cerbo, also `dbus-mqtt-grid` mit per Node-RED nachgebauten Zählerwerten, und (b) einen **BMS-Feed** — Batrium hängt am Cerbo, sonst regelt der 5000er ohne Kenntnis der Zell-/Temperaturgrenzen an derselben Batterie (`dbus-mqtt-battery`). Zwei ESS-Regler auf einem Zähler und einer Batterie, gekoppelt über MQTT-Latenz. Das ist die Sorte Konstruktion, die genau dann versagt, wenn niemand hinschaut → **nicht bauen.**

### ⚠️ Gefährlich in der Vorlage: „Modus 2 oder 4" als gleichwertig

Die Antwort empfiehlt mehrfach `Off (4)` **oder** `Charger Only (2)` als austauschbar. Sind sie nicht:

- **Charger Only (2):** Wechselrichter aus, **Transferrelais bleibt geschlossen** → AC-Out läuft weiter aus dem Netz.
- **Off (4):** Transferrelais **öffnet** → AC-Out spannungslos.

Heute hängen die Hauslasten am **AC-IN** (siehe Topologie 2026-07-07), also wäre „Off" momentan harmlos. **Nach dem geplanten Whole-home-Backup-Umbau (Lasten an den AC-Out) wäre „Off" jede Nacht ein Hausausfall.** Falls je eine Nacht-Abschaltautomatik gebaut wird: ausschließlich `Charger Only`, und der Umbau macht diese Unterscheidung sicherheitsrelevant. Ebenso falsch in der Vorlage: „es gibt keine mechanischen Schaltkontakte" — bei Off↔On schaltet das Transferrelais real (unkritisch bei ~700 Zyklen/Jahr, aber die Begründung stimmt nicht).

### Weitere Fehler in der Vorlage

- ~~**„150–200 W Eigenverbrauch"** vermischt Leerlaufsockel und Gesamtverlust. Datenblatt: 3 × 38 W = 114 W Nulllast; die gemessenen ~173 W der Referenznacht sind **Sockel + Teillast-Wandlung**. Zufällig ähnliche Zahl, falsch benannt.~~
  > ❌ **Dieser Kritikpunkt war selbst falsch (widerlegt 12.08.2026).** Die Regression über 72 Stunden weist den Sockel bei **174 W** aus (Gegenprobe 200 W) — die 173 W der Referenznacht waren also der Sockel, nicht „Sockel + Teillast". **Die Vorlage lag mit „150–200 W" richtig**, das Datenblatt mit 114 W ist der zu optimistische Laborwert. → Abschnitt **„Leerlaufverlust: Stand der Messungen"** am Dateiende.
- **„1,5–2 kWh Ersparnis pro Nacht"** → realistisch ~1,1–1,3 kWh (Referenznacht: 1,28 kWh Verlust über 7,4 h).
- **AES/Search Mode** als Hebel: im netzparallelen ESS praktisch wirkungslos — war hier schon notiert (Zeile „Search/AES praktisch nicht wirksam"). Search Mode ist für Inselbetrieb ohne Netz gedacht; im ESS regelt der Multi permanent gegen den Grid-Setpoint.
- **„Grid Setpoint auf −10 W stellen, dann regelt es ruhiger"** → ein negativer Setpoint ist **aktive Dauereinspeisung** aus der Batterie, keine Regelcharakteristik. In DE unerwünscht. Nach *oben* (nachts z. B. 250 W) wäre sinnvoll — aber wegen weniger gewandelter Leistung, nicht weil „die Multis thermisch herunterregeln".
- **Schieflast/Anmeldung komplett übersehen:** ein einphasiger 5000er erzeugt bis ~4 kVA Unsymmetrie; **VDE-AR-N 4105 zieht bei 4,6 kVA die Grenze**, und ein zusätzlicher Speicher-Wechselrichter ist beim Netzbetreiber (Netzbetrieb Hirschberg) anmeldepflichtig.

### ✅ Was an der Vorlage stimmte

Fronius am AC-IN → **Faktor-1-Regel gilt nicht** (kein Risiko bei plötzlicher Einspeisung). Software-Standby schadet der Elektronik nicht. MPPT-Vorrang beim Laden ist richtig (haben wir, siehe Ladedeckel-Automatik). Und: **ESS „Multiphase regulation" auf „Summe aller Phasen"** ist ein echter, kostenloser Hebel — bei „Individual phase" kann auf L1 eingespeist und gleichzeitig auf L2 geladen werden = Doppelwandlung ohne Nutzen. **→ Einstellung am Cerbo noch zu verifizieren** (bei saldierendem Zähler gehört sie auf „Summe aller Phasen"; der von Google zitierte Forumslink empfahl das Gegenteil und wurde falsch verwendet).

### 💡 Die eigentliche Erkenntnis: im Sommer lohnt Abschalten NICHT

Der Punkt, den beide Vorlagen-Varianten übersehen — gerechnet mit Norberts echten Tarifen (Bezug 34,94 ct, Einspeisung 7,382 ct) und der Referenznacht (4,90 kWh DC → 3,62 kWh AC):

| | Batterie gespart | Netz gekauft | Netto |
|---|---|---|---|
| **Sommer** (Batterie-kWh = entgangene Einspeisung 7,38 ct) | 4,90 × 7,38 = **0,36 €** | 3,62 × 34,94 = **1,26 €** | **−0,90 €/Nacht** |
| **Winter** (Batterie-kWh = vermiedener Bezug 34,94 ct) | 4,90 × 34,94 = **1,71 €** | 3,62 × 34,94 = **1,26 €** | **+0,45 €/Nacht** |

**Solange PV im Überfluss da ist, sind die 114 W Fixverlust fast gratis** — sie fressen Überschuss, der sonst für 7,38 ct rausgegangen wäre. Wehtun tun sie nur im Winter, wenn jede Batterie-kWh teuer erkauft ist. Eine Abschaltautomatik müsste also **am PV-Ertrag der letzten Tage bzw. saisonal** hängen, nicht an Tag/Nacht. Realistischer Jahresnutzen: **~150 Winternächte × 0,45 € ≈ 70 €** — nicht die ~350 €/Jahr, die die reine Fixverlust-Rechnung (Zeile „Kostenbezifferung der Überdimensionierung") suggeriert. Die 350 € sind der *physikalische* Verlust, nicht der *wirtschaftlich hebbare* Anteil.

### Konsequenz / Stand

- **Der 5000er bleibt ungenutzt in der Ecke.** Kein zweites GX, keine Zweitsystem-Bastelei. Wenn ein Verwendungszweck, dann als eigenständiges Insel-/Notstromsystem an anderer Stelle — nicht als Nacht-Ergänzung an dieser Batterie.
- **Nicht neu messen nötig:** Der Gesamtverlust läuft seit 25.07. als Sensor (`victron_speicher_wandlungsverluste_dc_ac`). Die Isolierung von `P_fix` ist als **Fix/Variabel-Verlustmodell** mit zwei Bins schon designt und für den Herbst terminiert (Abschnitt 2026-07-10) — das liefert P_fix gemessen statt geschätzt und beantwortet die Frage „wie viel wäre überhaupt hebbar" endgültig.
- **Wenn überhaupt eine Automatik**, dann ohne jede Hardware: die 3× 10000 per Node-RED auf `Charger Only` (D-Bus `com.victronenergy.vebus`, `/Mode`) — aber erst **nach** dem Fix/Variabel-Modell und **nur** mit Winter-/Ertragsbedingung, sonst kostet sie Geld. Vor dem Whole-home-Umbau harmlos, danach zwingend `2` statt `4`.
- **Offener Prüfpunkt:** ESS-Einstellung „Mehrphasige Regulierung" am Cerbo kontrollieren (soll: Summe aller Phasen). Einziger Sofort-Hebel ohne Nebenwirkung.

**Quellen:** [Cerbo GX – Connecting Victron products](https://www.victronenergy.com/media/pg/Cerbo_GX/en/connecting-victron-products.html) · [Venus GX – Connecting Victron products](https://www.victronenergy.com/media/pg/Venus_GX/en/connecting-victron-products.html) · [Venus OS & multiple VE.Bus systems (Community)](https://community.victronenergy.com/t/question-about-venus-os-and-multiple-ve-bus-systems/53294) · [ESS Design & Installation Manual](https://www.victronenergy.com/upload/documents/Energy_Storage_System/6292-ESS_design_and_installation_manual-pdf-en.pdf)

---

## 2026-08-12 — Wandlungsverluste explodiert: VE.Bus-Zähler-Reset + blockierter Monotonie-Wächter

**Symptom:** `sensor.victron_speicher_wandlungsverluste_dc_ac` sprang von stabilen **4,7–5,3 kWh/Tag** (14 Tage lang) auf 8,48 kWh. Stundenwerte: bis 15:00 normal 0,1–0,35 kWh, ab **16:00 dann 1,99 und 1,94 kWh/h**.

**Ursachenkette (drei Glieder):**

1. **Auslöser am Cerbo:** Die VE.Bus-Energiezähler wurden gegen **15:47–15:51** zurückgesetzt (Signatur eines Multi-Firmware-Updates oder „Reset historical data"). `AcIn1ToInverter` sprang von 22,94 auf **0**. MPPT-, Batterie- und Gridmeter-Zähler blieben unberührt — deshalb war es kein GX-weites Ereignis.
2. **Verstärker im MultiPlus-Flow:** Der Monotonie-Wächter (`if (sum < last - 0.001) return null;`) unterdrückte danach **jeden** Publish, weil die zurückgesetzte Summe nie wieder über den alten Stand kam. `ac_out_energy` fror bei **814,27 kWh** ein — für Monate, nicht für Minuten. Beweis: `last_reported` des HA-Sensors stand exakt auf 15:47:40, obwohl die vebus-Node weiter Events lieferte.
3. **Folgefehler im Verluste-Flow:** Dessen Sanity-Check `d[k] >= 0` akzeptiert ein **Null-Delta klaglos** — eine tote Quelle ist so nicht von „es fließt gerade nichts" zu unterscheiden. Mit stehender AC-Seite wurde die komplette DC-Energie als Verlust gebucht. Die ~1,95 kWh/h waren exakt die DC-PV-Leistung des Nachmittags: **der Zähler maß Ertrag, nicht Verlust.**

**Fix (beide Flows, mit Tests):**

- **Neuer Generator `tools/gen_multiplus_flow.py`** — der Wächter-Code lag vorher handgepflegt im JSON und musste für beide Zweige gelten. Kernidee der neuen Logik: nicht Rohwert gegen Rohwert prüfen, sondern **Kandidat (roh + Offset) gegen den zuletzt publizierten Wert**. Damit sind Neustart, Downtime und Zähler-Reset derselbe Fall: `offset += Sprunghöhe`, die Ausgabe macht dort weiter, wo sie stand. Rückgänge werden erst nach **Bestätigungsfrist** akzeptiert (großer Sprung 2 min, kleiner Dip 30 min) — sonst könnte ein einzelnes stale-0-Sample den Zähler um seinen ganzen Lebenszeitstand verfälschen. Offset liegt retained auf `victron/multiplus/_acout_state` / `_acin_state`, übersteht also Neustarts. Gilt jetzt für **beide** Zähler (AC-Aufnahme hatte vorher gar keinen Schutz).
- **`gen_losses_flow.py`:** Plausibilitätsprüfung — nennenswerte DC-Bewegung (> 0,05 kWh/Tick) bei *gleichzeitig* null Bewegung auf beiden AC-Zählern ⇒ Fenster verwerfen, `node.warn`, roter Node-Status. Lieber eine Lücke als ein Fehlwert.
- **Tests:** `tools/test_multiplus_guard.js` (neu, simulierte Uhr für die Fristen) und `tools/test_losses_flow.js` (erweitert um Stall-Fälle und Leerlauf-Gegenprobe). Beide grün.

**Beim Deploy beachten:** Der Rohzähler steht nach dem Reset bei ~1 kWh, HA aber bei 814,27. Damit der Sensor stetig bleibt, vor dem Deploy einmalig den Anker setzen:
`mosquitto_pub -t victron/multiplus/_acout_state -r -m '{"offset":0,"lastOut":814.27,"ts":0}'`
Der Wächter überbrückt den Rest dann nach 2 Minuten selbst. Ohne diesen Schritt startet der HA-Zähler bei ~1 kWh neu (für die Statistik unkritisch, HA behandelt total_increasing-Resets korrekt).

**Noch offen:** Die heute fälschlich gebuchten ~4–5 kWh stecken weiterhin im Zählerstand (93,3 kWh) und in der Langzeitstatistik — Korrektur über Entwicklerwerkzeuge → Statistiken bzw. RESET-Inject im Flow. Ebenfalls betroffen war der Wirkungsgrad-Flow, der am selben Topic hängt.

**Nachtrag Statistik-Korrektur (20:55):** Stunden 15:00–19:00 per WebSocket `recorder/adjust_sum_statistics` korrigiert (−0,171 / −1,750 / −1,709 / −1,240 / −1,241 kWh). Referenz war das Stundenprofil vom 11.08., skaliert am Verhältnis der heute noch sauberen Stunden 00:00–14:59 (Faktor 0,9619 — die Tage weichen nur 3,8 % voneinander ab). **Tageswert damit von 10,357 auf 4,246 kWh**, Vortage zum Vergleich 4,66–5,13. Der WS-Weg braucht `adjustment` als **Delta** (die UI dagegen will den absoluten Zellwert — nicht verwechseln) und arbeitet auf vollen Stunden-Buckets, Folgesummen verschieben sich automatisch mit.

**Zählerstand bewusst NICHT abgesenkt:** 96,412 → 89,088 wären 92,4 % des alten Werts und lägen damit **über** HA's 90-%-Schwelle für die Reset-Erkennung bei `total_increasing`. HA hätte den Sprung folglich nicht als Zähler-Reset, sondern als **negativen Zuwachs** gebucht und die gerade korrigierte Statistik ein zweites Mal um 7,3 kWh nach unten gezogen. Der absolute Stand eines Lebenszeitzählers hat ohnehin keine eigene Bedeutung — relevant sind die Zuwächse, und die stehen jetzt richtig. Flow-State (`accDc` 435,14 / `accAc` 338,73 / `pub` 96,41) bleibt konsistent zum HA-Sensor und rechnet ab dem Deploy korrekt weiter.

**Deploy + exakte Schlussbilanz (21:00–21:10):** Beide Flows per Admin-API deployed (`PUT /flow/<tabid>`, Backup der 271 Live-Nodes vorher gezogen). Um **21:03:57** hat der Wächter den Reset überbrückt: `{"offset":806.57,"lastOut":814.27,"raw":7.7}` → `ac_out_energy` blieb bei 814,27 stehen und wächst seither wieder (814,29 → 814,33). Der Verluste-Flow bucht wieder beidseitig (`accAc` 338,7278 → 338,7878).

**Der Rohzähler-Stand von 7,70 kWh machte die Korrektur exakt statt geschätzt:** Der Phantom-Verlust ist per Bilanz genau die nie gegengerechnete AC-Abgabe — Fehler = ΔaccAc_wahr = 7,70 kWh (acIn blieb über die ganze Ausfallzeit 0). Davon waren 6,109 bereits über das Vortagesprofil korrigiert; die restlichen **1,591 kWh** auf die 20:00-Stunde gebucht. Bemerkenswert: die Profil-Schätzung lag mit 7,57 vs. 7,70 kWh fast punktgenau — **das Verfahren „Vortagesprofil, skaliert an den sauberen Stunden" ist also belastbar**, wenn keine exakte Bilanz verfügbar ist. Tageswert final 4,304 kWh (21:08) → hochgerechnet ~4,9 zum Tagesende, Vortage 4,66–5,13.

## 2026-08-12 (Teil 2) — Ursache des Zähler-Resets: der Cerbo ist abgestürzt

**Gesichert (per SSH auf `einstein` = Cerbo `192.168.2.181`, Key hinterlegt):**
- **Uptime 5:48 h um 21:38 → Boot um 15:50:20.** `localsettings`-Log: `*** Venus OS v3.71 booted ***` um 13:50:44 UTC. Die HA-Datenlücke lief 15:49:30–15:51:04 — deckungsgleich.
- **Kein Firmware-Update:** v3.71, Build 20260315 (März), nicht heute installiert.
- **Schlagartiger Stopp ohne jede Spur:** kein Shutdown-Eintrag, kein Kernel-Panic, kein OOM, keine Thermal- oder Undervoltage-Warnung. Sämtliche Service-Logs (`dbus-systemcalc-py`, `vrmlogger`, `localsettings`, `dbus-modbus-client`) brechen ab und beginnen erst wieder mit dem Boot. ⚠️ `dmesg.old` ist NICHT der Absturz-Puffer, sondern ein Boot-Snapshot der ersten ~15 s — für Absturzanalysen unbrauchbar.
- **Hardware-Watchdog scharf:** `sunxi-wdt 1c20c90.watchdog: Watchdog enabled (timeout=16 sec, nowayout=1)`.
- **Der Cerbo ist chronisch überlastet: nur 2 CPU-Kerne, Load 3,3–4,9, 8 % idle, 72 % usr.** Temperatur 59 °C (abends, ohne Throttling — mittags entsprechend höher).

**Schlussfolgerung:** Ein **Watchdog-Reset durch Überlast** erklärt alles — und zwar gerade, weil nichts im Log steht: Der Hardware-Watchdog resettet den SoC, ohne dass das OS etwas mitbekommt. Das erklärt zugleich den Zähler-Reset: Bei einem harten Reset schreibt Venus die VE.Bus-`/Energy/`-Werte nicht mehr in den Flash. Ein sauberer Reboot hätte sie erhalten. Alternative wäre ein harter Spannungsverlust am Cerbo — weniger wahrscheinlich, da kein Hausstromausfall (Z2M/HA liefen durch) und die Anlage selbst weiterlief. **Beweisbar ist ein Watchdog-Reset nicht** (er hinterlässt per Definition keine Spur), aber er ist die einzige Hypothese, die zu allen Beobachtungen passt.

**Lastquellen (`top`):** `dbus-modbus-client` bis 17 % (scannt periodisch das ganze `192.168.2.0/24` — „Scan completed in 23 seconds"; Gridmeter sitzt auf `192.168.2.182:502`), `dbus_systemcalc` 11 %, `dbus_shelly` 8 %, `venus-gui-v2` (214 MB VSZ), `dbus-daemon` mit `system-insecure.conf` 6 % (= der D-Bus-über-TCP-Zugriff aus dem LXC).

> [!WARNUNG] Die Auslagerung nach Node-RED-LXC ist nur halb passiert
> **Auf dem Cerbo läuft weiterhin ein eigenes Node-RED** (`/service/node-red-venus`, PID 1724, 191 MB RAM, seit dem Boot). Die `flows.json` stammt vom **05.07.** und enthält 2 Tabs mit lediglich 2 `victron-input-ess`-Nodes — **keine MQTT-Ausgabe, keine Steuer-Nodes**, also kein Konflikt mit den LXC-Flows und offenbar ein Überbleibsel der Anfangsexperimente. Abschalten (Remote Console → Einstellungen → Venus OS Large → Node-RED) würde RAM und CPU freigeben, ohne etwas zu verlieren — vorher einmal reinschauen.

**Weitere Entlastungsmöglichkeiten:** Modbus-Gerät fest eintragen statt das Subnetz scannen zu lassen; `dbus_shelly` deaktivieren, falls keine Shellys über Victron eingebunden sind.

> [!KORREKTUR] Die Überlast-These oben ist WIDERLEGT (Messfehler)
> Die Aussage „chronisch überlastet, 8 % idle, Load 3,3–4,9" beruhte auf `top -bn1` — **das ist unbrauchbar**: Beim einzigen Durchlauf zeigt top die CPU-Zeit *seit Prozessstart*, nicht die momentane Last. Exakte Messung aus `/proc/stat` über 6 h Laufzeit: **user 40,8 % · system 9,6 % · IDLE 46,5 % · iowait 0,0 %**, momentan 56 %. Die eMMC war in 6 h ganze **1,16 s** aktiv (`io_ticks`), keine mmc-Fehler, 424 MB RAM frei. **Der Cerbo hat kein Ressourcenproblem.**
> Der Load-Average von 3,3 ist hier irreführend: Bei ~1750 Kontextwechseln/s zählt er viele kurzlebige lauffähige Tasks, ohne dass die CPU gesättigt wäre. **Load-Average nie ohne `/proc/stat` interpretieren.**
>
> **Folge: „Watchdog-Reset wegen Überlast" ist nicht haltbar.** Ein Watchdog-Reset bleibt die plausibelste Erklärung für den spurlosen Stopp (er hinterlässt per Definition nichts), aber als Auslöser kommt dann eher ein einzelner hängender Prozess, ein Kernel-Deadlock oder ein kurzer Spannungsverlust in Frage — nicht Ressourcenmangel. **Mit den verfügbaren Daten ist die Ursache nicht bestimmbar.** Bei Wiederholung: Uptime prüfen und schauen, ob die Abstände ein Muster zeigen.

**Echte CPU-Verteilung (CPU-Sekunden / % eines Kerns, 6 h):** `dbus-modbus-client` 4738 s / 21 %, `venus-gui-v2` 4495 s / 20 %, `dbus_systemcalc` 1922 s / 8 %, `vrmlogger` 1459 s / 6 %, `dbus_shelly` 1287 s / 5 %, `dbus-daemon` (D-Bus-über-TCP aus dem LXC) 1166 s / 5 %, `node-red` (auf dem Cerbo) 1143 s / 5 %. Summe aller Prozesse 21165 s von 43450 s Budget.

**Zur Auslagerungsfrage:** Node-RED kostet auf dem Cerbo **5 % eines Kerns im Leerlauf** (nur 2 ESS-Input-Nodes, keine Flows) — das ist der Grundverbrauch der Node.js-Runtime plus 191 MB RAM. Mit den ~15 produktiven Flows käme Flow-Last dazu, dafür entfielen die 5 % des `dbus-daemon` für den TCP-Zugriff. **Netto hätte der Cerbo es bei 46 % Idle auch selbst getragen.** Der Gewinn der Auslagerung liegt nicht in der Performance, sondern in der **Isolation**: Deploys, Node-RED-Abstürze und Paletten-Updates berühren den GX nicht mehr.

> [!KORREKTUR 2] Das Cerbo-Node-RED ist NICHT abschaltbar ohne Funktionsverlust
> Die Aussage oben („2 Tabs, nur `victron-input-ess`, keine MQTT-Ausgabe, keine Steuer-Nodes, Überbleibsel") war falsch — sie beruhte auf einem grep nach `victron-input-[a-z]+`, das die **schreibenden** `victron-output-ess`-Nodes übersah. Tatsächlich laufen dort **zwei produktive Funktionen**:
> 1. **Tägliche ESS-Grid-Setpoint-Automatik** (Tab „Flow 1"): zwei cron-Injects — `00 07 * * *` schreibt **−50**, `00 20 * * *` schreibt **−30** auf `/Settings/CGwacs/AcPowerSetPoint`. Kommentar im Flow: „Morgens im Sommer-100" (frühere Einstellung).
> 2. **SoC-Bedien-Dashboard** (Tab „SOC einstellen"): zwei `ui-slider` („SOC min %", „SOC active %") schreiben `/Settings/CGwacs/BatteryLife/MinimumSocLimit` bzw. via `exec`-Node lokal `dbus -y com.victronenergy.settings /Settings/CGwacs/BatteryLife/SocLimit SetValue`.
>
> **Backup gesichert:** `flows/cerbo-lokal-ess-setpoint-soc.json` (diese Flows waren bisher NICHT im Repo versioniert). Die Datei selbst liegt auf dem Cerbo unter `/data/home/nodered/.node-red/flows.json` auf der eigenen `/data`-Partition (`mmcblk1p5`) — Abschalten des Dienstes löscht nichts, Node-RED hält dort zusätzlich ein `.flows.json.backup`.
>
> **Vor dem Abschalten migrieren.** Zwei Fallstricke: (a) Der `exec`-Node ruft `dbus` **lokal auf dem Cerbo** auf — im LXC existiert das Kommando nicht, er muss durch einen `victron-output-ess`-Node ersetzt werden. (b) Die `ui-slider` brauchen die Dashboard-Palette im LXC. Der Setpoint bleibt nach dem letzten Schreiben stehen (persistente Einstellung), ein Abschalten ohne Migration richtet also keinen Sofortschaden an — es entfällt nur der tägliche Wechsel −50/−30.

## 2026-08-12 (Teil 3) — Venus OS v3.75 + Cerbo-ESS-Flow versioniert, Wächter live bestätigt

**Update v3.71 → v3.75** über die Remote Console eingespielt (22:22–22:26). A/B-Partitionswechsel bestätigt: aktives root von `mmcblk1p2` auf `mmcblk1p3`. Der zu 92 % volle `/` war dadurch nie ein Problem — das Update landet in der inaktiven Rootfs. `ImageType 1` (Large) bleibt erhalten, Node-RED überlebte. Settings vorher gesichert: `backups/cerbo-settings-20260812.xml` (⚠️ enthält die komplette Anlagenkonfiguration → bei `git init` in `.gitignore`).

**Nebenbefund, der die Absturzursache endgültig klärt:** `/Settings/System/AutoUpdate = 2` = „prüfen und herunterladen, **nicht** installieren". Venus konnte also gar nichts von allein aufspielen — der 15:50-Neustart war definitiv kein Update.

> [!ERFOLG] Der Wächter hat den Praxistest live bestanden
> **Das Update hat die VE.Bus-Zähler erneut auf 0 gesetzt** (`InverterToAcOut` = 0.0) — der Mechanismus „harter GX-Neustart ⇒ `/Energy/*` verloren" ist damit reproduziert und bestätigt. Der neue Monotonie-Wächter hat den zweiten Reset selbstständig überbrückt: Offset **806,57 → 815,73**, `ac_out_energy` lief **nahtlos von 815,73 auf 815,77 weiter**, ohne Sprung in HA.
> **Entscheidender Beleg:** Die Wandlungsverluste buchten in der 22:00-Stunde **0,23 kWh** — exakt der Normalwert. Vor dem Fix waren es 1,5–2,0 kWh/h. Ohne den heutigen Umbau hätte dieses Update die Verlustrechnung erneut entgleisen lassen.

**Cerbo-ESS-Flow ist jetzt generiert und getestet:** `tools/gen_cerbo_ess_flow.py` → `flows/cerbo-lokal-ess-setpoint-soc.json`, Tests in `tools/test_cerbo_ess_flow.js` (24 Stunden durchgeprüft, Umschaltgrenzen, Konsistenz cron ↔ Startlogik). Deploy per SSH (`cat >` nach `/data/home/nodered/.node-red/flows.json`, dann `svc -t /service/node-red-venus`); Backup liegt als `flows.json.bak-20260812-2215` auf dem Cerbo. Vergleich gegen den Live-Stand ergab: kein Node verloren, nur die zwei neuen dazu.

**Neu darin — die Neustart-Lücke ist geschlossen:** Ein Start-Inject setzt den Grid-Setpoint nach jedem Neustart anhand der Uhrzeit. Live bestätigt um 22:26:27: `Grid-Setpoint nach Neustart gesetzt: -30 W (22:xx Uhr, Nachtfenster)`. Setpoint-Werte (−50 Tag / −30 Nacht, Umschaltung 7/20 Uhr) stehen jetzt an **einer** Stelle im Generator und speisen cron **und** Startlogik — sie können nicht mehr auseinanderlaufen.

**Offen/beobachten:** `sensor.victron_multiplus_ac_aufnahme` steht bei 0 und wird nicht publiziert, weil der Rohwert konstant 0 ist (kein Netzbezug in den Multi seit dem Reset) — die vebus-Node feuert nur bei Änderung. Der Zähler beginnt bewusst neu bei 0; ein Anker wurde hier absichtlich NICHT gesetzt, um keine Phantom-Buchung von 22,94 kWh zu erzeugen. Sobald wieder aus dem Netz geladen wird, wächst er normal.

---

## 📌 Für den Winter: Entscheidungsgrundlage Multi-Standby (Stand 2026-08-12)

> [!ZUSAMMENFASSUNG] Kurzfassung
> Der Leerlauf der drei Multis ist **gemessen 174–200 W** (≈ 4,9 kWh/Tag) und **saisonunabhängig**. Im Sommer ist Abschalten **funktional ausgeschlossen**, im Winter wird es **wirtschaftlich interessant** — aber die Datengrundlage dafür fehlt noch. Vor einer Entscheidung: die unten beschriebene Kennzahl über den Winter mitzählen lassen.

### Gemessene Basis (Regression über 72 Stunden, 08.–10.08.2026)

`Verlust [kWh/h] = 0,0332 × AC-Abgabe + 0,1736`

| Kennzahl | Wert | Methode |
|---|---|---|
| **Fixverlust (Leerlauf)** | **174 W** | Achsenabschnitt der Regression |
| Gegenprobe | **200 W** (151–267) | 12 Stunden mit AC-Abgabe ≈ 0 |
| Marginaler Wandlungsverlust | 3,3 % → **η ≈ 96,8 %** | Steigung |
| Aufteilung Tagesverlust (5,19 kWh) | **80 % Leerlauf / 20 % Wandlung** | |
| Durchschnittliche AC-Leistung | 890 W | 21,35 kWh / 24 h |
| **Auslastung** | **3,0 %** von 3 × 10 kVA | |

Pro Gerät sind das ~60 W statt 35 W laut Datenblatt. Die Differenz ist plausibel (aktiver ESS-Regelbetrieb, mitgezählte DC-Verbraucher wie Cerbo und BMS, Verkabelung). ⚠️ Es ist eine **Bilanzgröße** aus der Differenz zweier Messwelten — ein systematischer Kalibrierungs-Offset zwischen Batrium/MPPT- und vebus-Zählern würde hier ebenfalls als „Leerlauf" erscheinen. Auf die letzten ~20 W nicht festlegen.

### Warum der Sommer entschieden ist

- **Die MPPTs sind DC-gekoppelt** (~26 kWh/Tag = 22 % der PV). Ihr einziger Weg ins Hausnetz führt durch die Multis. Ohne sie könnten sie nur die Batterie laden (7,5 kWh) und würden danach abregeln → **~19 kWh/Tag Ertragsverlust**. Der Fronius (~94 kWh/Tag, 78 %) läuft dagegen AC-seitig an den Multis vorbei.
- Die Multis liefern **20,0 h/Tag** AC (83 % der Zeit), Durchsatz 20,2 kWh/Tag. Untätig sind sie nur **4,0 h/Tag — und zwar morgens 7–10 Uhr**, nicht nachts: Dann deckt der Fronius den Verbrauch und die MPPTs laden die Batterie direkt über DC.
- Abschalten in genau diesen 4 Stunden spart 0,82 kWh/Tag = 18 % des Leerlaufs — im Sommer bewertet mit 7,38 ct sind das **6 Cent pro Tag**. Lohnt nicht.

### Was im Winter anders ist

Batterie: **280 Ah ≈ 14,3 kWh brutto, bei MinimumSocLimit 20 % rund 11,5 kWh nutzbar.**

- Der MPPT-Ertrag bricht bei Ost/West und tiefem Sonnenstand stark ein → kaum noch DC-Durchleitung.
- Die Batterie wird oft nicht mehr voll und erreicht abends die 20 %-Grenze. **Ab da stehen die Multis wirklich untätig**, während das Haus am Netz hängt — genau die Stunden, die im Sommer fehlen.
- Der Leerlauf bleibt bei 4,9 kWh/Tag, **aber der Preis kehrt sich um: 7,38 ct → 34,94 ct (Faktor 4,7)** — 1,71 €/Tag statt 0,36 €.

**Modellrechnung (NICHT gemessen, Annahme: halber Tag „Batterie leer, keine PV"):**
`12 h × 205 W = 2,5 kWh/Tag → per Mode 2 hebbar ~60–70 % → ~0,56 €/Tag → ~65 €/Jahr`
Deckt sich mit der früheren Schätzung von ~70 €. Bestätigt zugleich, warum die 350 € aus der reinen Fixverlustrechnung zu hoch gegriffen waren: Der Großteil des Leerlaufs fällt an, **während die Multis gebraucht werden**.

### ⬜ Offen: die Kennzahl, die im Winter zu messen ist

Datenlage: **Es gibt keine Winterdaten** — die HA-Langzeitstatistik beginnt im Juli 2026 (auch `sensor.fronius_pv_energie` hat nur Juli/August). Die obige Rechnung ist deshalb ein Modell, keine Messung.

Zu erheben ist **nicht** der Gesamtverlust, sondern der **vermeidbare Anteil**: Verlust, der anfällt, während die Anlage nichts leisten kann — also **SoC ≤ 25 % UND PV ≈ 0**. Ein kleiner Zusatz-Flow im LXC (Generator-Stil wie die übrigen), der genau diese Stunden akkumuliert, würde bis Januar aus der Schätzung eine Zahl machen. *Noch nicht gebaut — bewusst zurückgestellt.*

### Randbedingungen für eine spätere Standby-Automatik

1. **Ausschließlich Mode 2 (Charger Only) schreiben.** Mode 4 (Off) öffnet das Transferrelais → AC-Out spannungslos. Relevant spätestens nach einem Whole-home-Umbau.
2. **Rechtzeitig zurückschalten, bevor morgens PV kommt** — sonst kostet die Automatik DC-Ertrag statt Geld zu sparen.
3. Bedingung saisonal bzw. am Ertrag der Vortage aufhängen, **nicht** an Tag/Nacht: Nachts arbeiten die Multis (Batterieentladung), sie stehen nur, wenn die Batterie leer ist.

### ⚠️ Ergänzung zum Winter-Abschnitt: Batterie-Ausbau auf ~30 kWh ändert die Rechnung

Der oben stehende Abschnitt rechnet mit **11,5 kWh nutzbar** (280 Ah, MinSoC 20 %). Mit dem beschlossenen Ausbau auf ~30 kWh brutto (2. Pack à 15 kWh parallel, weitere CellMate K9 am selben WatchMon Core) sind es **~24 kWh nutzbar** — die dortigen Zahlen sind damit überholt.

**1. Das Sparpotenzial sinkt, statt zu steigen.** Die Modellrechnung von ~65 €/Jahr unterstellt, dass die Batterie im Winter etwa den halben Tag leer ist und die Multis untätig stehen. Mit doppeltem Speicher überbrückt sie längere Dunkelphasen und erreicht die 20 %-Grenze **seltener** — genau die Stunden, um die es beim Standby geht, werden weniger. **Die 65 €/Jahr sind damit eine obere Grenze, die nach dem Ausbau weiter fällt.** Eine Standby-Automatik lohnt sich danach noch weniger als ohnehin schon.

**2. Der Wirkungsgrad steigt von allein.** Der Leerlauf (4,2 kWh/Tag) bleibt konstant und verteilt sich auf mehr Durchsatz. Mit der gemessenen Verlustgeraden `Verlust = 0,0332 × AC + 4,17`:

| Szenario | AC-Durchsatz | Verlust | Wirkungsgrad |
|---|---|---|---|
| heute (Sommer) | 21,4 kWh/Tag | 4,9 kWh | **80,4 %** |
| E-Auto nachts aus dem Speicher | ~41 kWh/Tag | 5,5 kWh | **88,2 %** |

Der Verlust steigt nur um 0,6 kWh, obwohl fast doppelt so viel Energie durchgeht — es wächst ja nur der variable Anteil von 3,3 %. ⚠️ **Wichtig: Der Durchsatz steigt NICHT automatisch mit der Kapazität.** Im Sommer werden heute nur 7,6 von 11,5 nutzbaren kWh entladen — der Nachtbedarf des Hauses ist der Engpass, nicht der Speicher. Zusätzlichen Durchsatz bringt vor allem **die Nachtladung des E-Autos aus der Batterie** (Auto tagsüber unterwegs, lädt abends aus dem Speicher statt aus dem Netz). Ohne dieses Nutzungsmuster bleibt die zweite Batteriehälfte im Sommer weitgehend ungenutzt.

**3. Korrektur zur früheren Modellannahme (Abschnitt „Fix/Variabel-Verlustmodell"):** Dort war `P_fix` mit **~2 kWh/Tag** angesetzt. Gemessen sind es **4,17–4,8 kWh/Tag**, also mehr als das Doppelte. Die dortige Wirkungsgrad-Prognose (85–88 % bei hohem Durchsatz) trifft trotzdem ungefähr zu — aber nur, weil der angenommene Durchsatz hoch war, nicht weil P_fix stimmte.

**4. Die Multi-Dimensionierung ist damit gerechtfertigt.** Mit 24 kWh nutzbar wird die 11-kW-Nachtladung praktikabel: 2,2 Stunden Volllast statt 1,0, das sind ~24 kWh ins Auto (140–160 km), bei entspannten 0,41 C statt 0,82 C. Drei 5000er wären bei 3,67 kW/Phase plus Hauslast am Limit gewesen — die 10000er sind für dieses Szenario die richtige Wahl. Der Preis der Bereitschaft (110–190 €/Jahr Mehr-Leerlauf gegenüber 5000ern) kauft damit eine real nutzbare Funktion, nicht nur Reserve.

**To-dos, die mit dem Ausbau akut werden (aus den Abschnitten oben):**
- [ ] **Gesamtkapazität im WatchMon Core neu einstellen** (SoC/Coulomb) — sonst rechnet der SoC auf der alten Kapazität weiter und **alle SoC-basierten Automatiken laufen falsch**.
- [ ] **Ladedeckel-Automatik anpassen**: Schwelle „SoC < 60 %" ist für 15 kWh gerechnet (= 6 kWh Restbedarf); bei 30 kWh entspricht dasselbe Ziel nur noch 30 %. Besser gleich **in kWh-Restbedarf statt SoC-Prozent** umstellen.
- [ ] Nach dem Ausbau die Verlustgerade **neu regressieren** (`Verlust vs. AC-Abgabe`, Stundenwerte) — P_fix sollte unverändert bleiben; tut es das nicht, war ein Teil des „Leerlaufs" in Wahrheit ein Messoffset.

### 🔥 Ergänzung 2: Heizen mit den Split-Klimas verändert das Winterbild grundlegend

Erfasst sind aktuell zwei Einheiten mit eigenem Zigbee-Zähler: `sensor.0xa085e3fffebc4574_energy` (**SK_Arbeitszimmer_Küche**, 298 kWh) und `sensor.0xa085e3fffebd16b8_energy` (**SK_Wohnzimmer**, 144 kWh). Geplant ist, im Winter **vermehrt damit zu heizen**.

**Damit ist die Standby-Überlegung endgültig vom Tisch.** Der Winterabend- und Nachtverbrauch steigt genau in den Stunden, in denen die Multis nach der Modellrechnung untätig herumstehen sollten. Sie arbeiten dann — der „vermeidbare Leerlauf", um den es ging, verschwindet weitgehend von selbst. **Die 65 €/Jahr aus dem Modell oben sind damit endgültig eine theoretische Obergrenze.**

**Der Wirkungsgrad wandert in den prognostizierten Bereich.** Bei 15–20 kWh zusätzlichem Tagesbedarf im Winter liegt der Durchsatz genau dort, wo die frühere Notiz 85–88 % erwartet hat — diesmal aus einem realen Nutzungsmuster statt aus einer Annahme. Zusammen mit dem 30-kWh-Speicher ist das die Konstellation, für die die Anlage ausgelegt wurde.

**Wirtschaftlichkeit gegen die Ölheizung** (Referenz aus dem Abschnitt oben: **13,5 ct/kWh Wärme**, 3.500 l/Jahr bei Kessel 85 %), gerechnet mit Netzstrom 34,94 ct:

| COP | Wärmekosten | vs. Öl (13,5 ct) |
|---|---|---|
| 3,5 (mild, ~10 °C) | 10,0 ct | −26 % |
| 3,0 (~0 °C) | 11,6 ct | −14 % |
| **2,6** | **13,5 ct** | **Break-even** |
| 2,0 (~−10 °C) | 17,5 ct | +30 % |

**Break-even bei COP 2,6** — den unterschreiten moderne Split-Geräte erst bei deutlichem Frost. Mit PV- oder Batteriestrom (Opportunitätskosten 7,38 ct statt 34,94 ct) ist die Klima praktisch immer im Vorteil. ⚠️ Nicht vergessen: Im Heizbetrieb ziehen Split-Geräte **deutlich mehr als im Kühlbetrieb** (Abtauzyklen, größere Temperaturspreizung) — die heute gemessenen 442 W (Kühlen, beide Räume) sind keine Referenz für den Winter.

**To-do: den realen Jahres-COP messen statt schätzen.** Beide Geräte haben Stromzähler. Über den Winter den SK-Stromverbrauch gegen den **eingesparten** Ölverbrauch stellen (Liter × 10 kWh × 0,85 Kesselwirkungsgrad = ersetzte Nutzwärme) → daraus ergibt sich der tatsächliche Arbeitsjahres-COP der Anlage. Das ist belastbarer als jedes Datenblatt und beantwortet endgültig, wie weit sich der Umstieg trägt. Sinnvollerweise **vor** der nächsten Ölbestellung auswerten.

---

## 📐 Leerlaufverlust: Stand der Messungen (konsolidiert 2026-08-12)

Dieser Abschnitt löst widersprüchliche Angaben an mehreren Stellen der Notiz auf. **Was hier steht, gilt.**

### Die drei Datenpunkte

| Quelle | Wert | Methode | Belastbarkeit |
|---|---|---|---|
| Victron-Datenblatt | **114 W** (3 × 38 W) | Laborwert „zero load power", 25 °C, ohne Last am AC-Out | untere Grenze — im netzparallelen ESS regelt der Multi permanent nach, Search/AES sind wirkungslos |
| Referenznacht 25.07. | **173 W** | Einzelmessung, 7,4 h, 662 W Durchsatz | Größenordnung richtig, aber Sockel und Teillast-Wandlung nicht trennbar |
| **Regression 08.–10.08.** | **174 W** | 72 Stundenpaare, `Verlust = 0,0332 × AC-Abgabe + 0,1736` — der Achsenabschnitt ist per Konstruktion der Wert bei Durchsatz **null** | **maßgeblich** |
| Gegenprobe | **200 W** (151–267) | 12 Stunden mit AC-Abgabe ≈ 0 | bestätigt unabhängig |

### Auflösung

**Der Leerlaufsockel beträgt 174–200 W**, nicht 114 W. Die 173 W der Referenznacht waren bereits der Sockel — die damalige Deutung als „Sockel + Teillast-Wandlung" (Abschnitt „Idee nachts kleiner MultiPlus", 2026-07-26) ist widerlegt. Regression und Gegenprobe messen unabhängig voneinander dasselbe; das Datenblatt ist der zu optimistische Laborwert. Pro Gerät sind das ~60 W statt 38 W.

⚠️ Es bleibt eine **Bilanzgröße** aus der Differenz zweier Messwelten (Batrium/MPPT-Zähler gegen vebus-Zähler). Enthalten sind neben dem Multi-Eigenverbrauch auch Cerbo, BMS, DC-Verkabelung und ein möglicher Kalibrierungsoffset. Auf die letzten ~20 W nicht festlegen.

### Was daraus folgt — korrigierte Werte

| Größe | alt (Notiz 25.07.) | **gültig** |
|---|---|---|
| Standby-Anteil am Tagesverlust | 60 % | **80–92 %** |
| Tageszerlegung (bei 21,4 kWh AC) | 2,74 fix + 1,9 Wandlung | **4,17 fix + 0,7–1,0 Wandlung** |
| Wandlungswirkungsgrad | ~93 % (Durchschnitt) | **96,7 % marginal** (3,3 % je zusätzlicher kWh) |
| Leerlauf pro Jahr | 1.000 kWh ≈ 350 € | **1.520–1.750 kWh** |

Die alte Zerlegung ging rechnerisch auf (2,74 + 1,9 = 4,6 ✓), verteilte die Anteile aber falsch. **Kernaussage bleibt und verstärkt sich sogar:** Der Löwenanteil des „Wandlungsverlusts" ist Bereitschaftsverbrauch, nicht Wandlung — die Wandlung selbst ist mit ~96,7 % ausgezeichnet.

### Weitere vereinheitlichte Angaben

- **Batteriekapazität:** 280 Ah bei 16S-LiFePO4 (51,2 V nominal) = **~14,3 kWh brutto**, bei realer Betriebsspannung ~52,5 V rund 14,7 kWh. Die Angabe „≈ 15 kWh" (Abschnitt Batrium-Zähler) ist die gerundete Obergrenze — beide meinen dasselbe Pack. **Nutzbar bei MinSoC 20 %: ~11,5 kWh**, nach dem Ausbau auf 2 Packs **~23–24 kWh**.
- **Standby-Sparpotenzial im Winter:** Die Angaben „~70 €/Jahr" (2026-07-26) und „~65 €/Jahr" (Winter-Abschnitt) sind dieselbe Größenordnung aus leicht verschiedenen Annahmen. **Verbindlich: 65–70 €/Jahr als obere Grenze**, die durch Batterie-Ausbau und Heizbetrieb weiter sinkt.
- **Die „350 €"** aus der Kostenbezifferung sind der *physikalische* Verlust bei durchgängigem Winterstrompreis, nicht der hebbare Anteil. Mit dem korrigierten Sockel wären es rechnerisch 530–610 €/Jahr — **aber nur ein Bruchteil davon ist überhaupt vermeidbar**, weil die Multis 20 h/Tag arbeiten. Für Entscheidungen ist ausschließlich der hebbare Anteil relevant.

### ⬜ Offene Gegenprobe

Nach dem Batterie-Ausbau die Regression wiederholen. **`P_fix` muss unverändert bei 174–200 W liegen** — am Leerlauf der Multis ändert ein zweites Batteriepack nichts. Kommt ein anderer Wert heraus, war ein Teil des gemessenen „Leerlaufs" in Wahrheit ein Kalibrierungsoffset zwischen den Messwelten.

---

## 🛢️ Öl vs. Wärmepumpe: Stand 08/2026 (konsolidiert, ersetzt die Rechnung von 07/2026)

**Anlass:** Ölpreis deutlich gestiegen + Rechenfehler in der Juli-Analyse gefunden. **Was hier steht, gilt.**

### Neue Preisbasis

| | 07/2026 | **08/2026** | Δ |
|---|---|---|---|
| Heizöl | 114,56 €/100 l | **139,90 €/100 l** | **+22,1 %** |
| Wärmepreis (10 kWh/l, Kessel 85 %) | 13,48 ct/kWh | **16,46 ct/kWh** | |
| 3.500 l/Jahr | 4.009,65 € | **4.896,50 €** | **+887 €** |
| **Break-even-JAZ** (Strom 34,94 ct) | 2,59 | **2,12** | |

**Die Break-even-JAZ ist die entscheidende Zahl.** Sie sagt, ab welcher Arbeitszahl die Split-WP billiger heizt als der Ölkessel. Bei 2,12 liegt sie so tief, dass moderne Inverter-Splits sie **an praktisch jedem Tag des Jahres** überbieten — auch bei −15 °C (dort noch COP ~1,8–2,2). **Damit ist das Argument „die letzten 20 % Wärmearbeit sind unwirtschaftlich" hinfällig**; es galt nur beim alten Ölpreis (Break-even 2,59).

### Szenarien bei 139,90 €/100 l (29.750 kWh Nutzwärme/Jahr)

| Szenario | Kosten/Jahr | Ersparnis |
|---|---|---|
| **A** — weiter nur Öl | 4.897 € | — |
| **B** — Vollverzicht, JAZ 2,5 | 4.158 € (11.900 kWh) | **+739 €** |
| **B** — Vollverzicht, JAZ 3,0 | 3.465 € (9.917 kWh) | **+1.432 €** |
| **B** — Vollverzicht, JAZ 3,5 | 2.970 € (8.500 kWh) | **+1.927 €** |
| **C** — bivalent 70 % (JAZ 3,5) | 3.548 € (2.079 € Strom + 1.469 € Öl) | +1.349 € |
| **C** — bivalent 80 % (JAZ 3,5) | 3.355 € (2.376 € Strom + 979 € Öl) | +1.541 € |

**Wirtschaftlich liegen Vollverzicht und Bivalenz nah beieinander** (1.350–1.930 €) — die Entscheidung fällt daher nicht am Geld, sondern an Technik und Komfort. ⚠️ **Nicht in der Tabelle, aber relevant: Bei echtem Vollverzicht entfallen die Kesselfixkosten** (Wartung, Schornsteinfeger, Tankprüfung) — grob **200–350 €/Jahr**. Damit zieht Szenario B an B/C vorbei.

**Strategisch entscheidend:** Die Stromkosten (2.970–4.158 €) sind **unabhängig vom Ölpreis**. Jede weitere Ölpreissteigerung verbessert die WP-Bilanz automatisch, ohne dass sich an der Anlage etwas ändert — der PV-Eigenverbrauch drückt sie zusätzlich.

### Was Vollverzicht technisch noch fehlt (unverändert gegenüber 07/2026)

Der Ölpreis ändert die **Wirtschaftlichkeit**, nicht die **Physik**:

1. **Leistung an Frosttagen.** Normheizlast grob 12–15 kW (aus 29.750 kWh Jahreswärme); die zwei vorhandenen Splits liefern bei Kälte ~4–5 kW thermisch. Lücke bleibt.
2. **Verteilung.** Splits heizen nur ihren Raum — Schlafzimmer, Bad, Flur haben keine Einheit.

**→ Der Weg zu null Öl führt über weitere Split-Einheiten**, die beide Probleme gleichzeitig lösen und die JAZ heben (jedes Gerät arbeitet entspannter). Bei 1.350–1.930 € Jahresersparnis amortisiert sich eine zusätzliche Einheit (~1.500–2.500 € installiert) **binnen ein bis zwei Jahren**.

### Vorgehen

- **Öl nur noch in Teilmengen kaufen**, nicht 3.500 l am Stück — kein Kapital in einem Brennstoff binden, der eventuell nicht mehr gebraucht wird; Reserve für Extremtage bleibt.
- **Diesen Winter die reale JAZ messen** (Zigbee-Zähler an beiden Splits vorhanden, Wärme über Öl-Minderverbrauch gegenrechnen). Sie entscheidet zwischen +739 € und +1.927 € — die Spanne ist größer als jeder andere Unsicherheitsfaktor.
- **Parallel den Deckungsgrad protokollieren**: An wie vielen Tagen bleibt der Kessel aus? Das beantwortet die Leistungsfrage praktisch statt rechnerisch und zeigt, wie viele Zusatzgeräte wirklich nötig sind.

---

## BW-WP-Stufentest: Auswertung und verkehrte Pumpenrichtung (13.08.2026)

→ Verschoben nach [[Brauchwasser-Wärmepumpe]] (23.08.2026).

## Cerbo-Wächter: Reboot- und Ausfallmeldung per Pushover (13.08.2026) ✅

**Anlass:** Der Cerbo hatte am 12.08. neu gestartet, der Grund war nicht mehr feststellbar. Norbert wollte künftig eine Push-Nachricht.

### Der Reboot vom 12.08. ist geklärt

Aus `/data/log/messages` (überlebt den Neustart):

```
Aug 12 20:23:18 UTC  shutdown[27489]: shutting down for system reboot
Aug 12 20:23:18 UTC  init: Switching to runlevel: 6
Aug 12 20:24:04 UTC  Booting Linux on physical CPU 0x0
```

**Kein Absturz, sondern ein sauber angeforderter Reboot** — Runlevel 6, alle Dienste ordentlich beendet, nach 46 s wieder oben. Vorlaufzeit 23.581 s (6,5 h), der Boot davor war am 12.08. um 13:50 UTC. Wer den Reboot ausgelöst hat, verrät das Log nicht (nur die PID des `shutdown`-Aufrufs). ⚠️ **Der Cerbo läuft auf UTC** — Logzeitstempel sind nicht Lokalzeit (22:23 lokal = 20:23 im Log).

Ein Watchdog-Reset sähe anders aus: Das Log bricht mitten im Betrieb ab, ohne Shutdown-Vermerk. Genau diese Unterscheidung wertet der Wächter jetzt automatisch aus. (Hardware-Watchdog ist aktiv: `sunxi-wdt`, 16 s, `nowayout=1`.)

### Warum ein Reboot bisher unsichtbar war

In HA war vom Neustart **nichts** zu sehen — der SoC-Verlauf läuft über die 48 h lückenlos durch. Ursache: Die MQTT-Sensoren sind retained, HA hält also den letzten Wert stur weiter. Es gab keine Stelle in der Kette, an der ein Ausfall sichtbar wurde.

### Architektur: zwei Ereignisquellen, kein Polling

Der Glücksfall des Setups: Node-RED läuft im LXC, überlebt den beobachteten Ausfall also. Bewusst **ereignisbasiert** statt gepollt:

| Ereignis | Quelle | Aussage |
|---|---|---|
| `victron/cerbo/boot` (retained, JSON) | `/data/rc.local` auf dem Cerbo | „Ich wurde neu gestartet" — inkl. Startart, Vorlaufzeit, Firmware, Logauszug |
| `victron/cerbo/status` = `offline` | **Last Will**, publiziert vom Broker | „Er antwortet nicht mehr" (~45 s Erkennungszeit) |

**Merksatz:** Ein Ausfall kann nie ein Ereignis der ausgefallenen Komponente sein — tote Systeme senden nichts. Deshalb braucht „ist weg" einen Stellvertreter (LWT) oder eine Zeitüberwachung. Das LWT delegiert das Polling an den Broker, der das Keepalive-Timing ohnehin macht.

### Bausteine

**Auf dem Cerbo** (Quellen im Repo unter `cerbo/`, Ziel `/data/cerbo-online-watch/`):
- `service/run` + `service/log/run` — daemontools-Dienst, eingehängt als `/service/cerbo-online-watch`. Hält per `mosquitto_sub` die MQTT-Session offen, Last Will `offline` retained.
- `boot-notify.sh` — Boot-Meldung inkl. Klassifikation `geplant` / `unerwartet` / `ausgeschaltet` / `unbekannt`.
- `/data/rc.local` — von Venus OS bei jedem Start aufgerufen (`/etc/rc5.d/S99custom-rc-late.sh` → `/etc/init.d/custom-rc-late.sh`). Setzt den Service-Symlink neu und startet die Boot-Meldung im Hintergrund. `/data` ist persistent → überlebt Firmware-Updates.

**In Node-RED:** Tab „Cerbo-Wächter (Reboot & Ausfall)", Live-ID `9adad6a1f416f4d4`, Repo `flows/cerbo-waechter-pushover.json` (mit Platzhaltern statt Credentials, wie beim Batrium-Flow). Generator: `tools/`-Skript im Scratchpad, Deploy per `PUT /flow/9adad6a1f416f4d4`.

**In HA** (MQTT-Discovery, Device „Cerbo GX"): `binary_sensor.cerbo_gx_online` (connectivity), `sensor.cerbo_gx_letzter_start` (timestamp), `sensor.cerbo_gx_startart` (diagnostic). Die beiden Boot-Sensoren haben `availability_topic` auf `victron/cerbo/status` → werden bei Ausfall echt `unavailable`.

**Push-Prioritäten:** unerwarteter Neustart und Ausfall = Prio 1 (Ton), geplanter Neustart und Entwarnung = Prio 0.

### Zwei Fallen, die erst der Test zeigte

1. **`mosquitto_sub` reconnectet still.** Nach dem Keepalive-Timeout feuert der Broker das Last Will und trennt — aber `libmosquitto` (`loop_forever`) verbindet von selbst neu, ohne dass jemand den Status korrigiert. Der Prozess stirbt also nicht, supervise startet nichts neu, und das retained `offline` bliebe **dauerhaft** stehen, obwohl der Cerbo längst wieder da ist. Live reproduziert (Client per `SIGSTOP` eingefroren). **Fix:** Der Dienst abonniert sein eigenes Status-Topic und republisht `online`, wenn er ein `offline` empfängt, obwohl er läuft — selbstheilend.
2. **Deduplizierung über den Startzeitpunkt, nicht den Meldezeitpunkt.** Sonst meldet jeder erneute Aufruf von `boot-notify.sh` einen Neustart. Toleranz bewusst nur 30 s: Der Cerbo bootet in ~46 s, eine **Boot-Schleife** erzeugt Neustarts im Minutentakt — die sollen einzeln gemeldet werden. Eine zunächst gesetzte 120-s-Toleranz hätte sie verschluckt (vom Unit-Test gefunden).

Dazu ein Sicherheitsdetail: `mosquitto_pub -P <pw>` legt das Passwort in der **Prozessliste** offen (`ps`). Deshalb Zugangsdaten in `mosquitto/mosquitto_{pub,sub}` (chmod 600) und `XDG_CONFIG_HOME` explizit setzen — beim Boot ist `HOME` nicht garantiert.

### Verifikation (13.08.2026)

- Last Will feuert bei unsauberem Abriss ✓ (Client eingefroren, Broker publizierte `offline`)
- Selbstheilung stellt `online` wieder her ✓ (Log + `binary_sensor` in HA)
- Sauberer Stopp meldet `offline` über den `trap` ✓ (LWT feuert dabei nicht — MQTT-Spec)
- Boot-Kette komplett ✓ (`custom-rc-late.sh start` → Symlink + Meldung, korrekt als `geplant`)
- Auswertelogik: 11 Fälle grün, `tools/test_cerbo_waechter.js` (liest den Code direkt aus dem Flow-Export, läuft mit `node`, braucht weder Broker noch Push)
- HA-Entitäten liefern ✓

**Noch offen:** Ein echter Reboot als Endabnahme (verifiziert `rc.local` im Kaltstart) — bewusst nicht ungefragt ausgelöst, weil der Cerbo ~46 s die ESS-Regelung aussetzt. Optional außerdem: `availability_topic` auch an die bestehenden Victron-Sensoren hängen, damit bei einem Ausfall nicht nur der neue Sensor, sondern das ganze Bild ehrlich `unavailable` zeigt.

### Betriebshinweise

```sh
# Dienststatus / Log
ssh root@192.168.2.181 'svstat /service/cerbo-online-watch'
ssh root@192.168.2.181 'tail -n 20 /var/log/cerbo-online-watch/current | tai64nlocal'

# Boot-Meldung von Hand auslösen (löst bewusst KEINEN Push aus - gleicher Start)
ssh root@192.168.2.181 '/data/cerbo-online-watch/boot-notify.sh'

# Push-Kette testen
curl -X POST http://192.168.2.80:1880/inject/cw_inject_test

# Passwortdateien neu erzeugen (nach Broker-Passwortwechsel)
ssh root@192.168.2.181 'PW=$(cat /data/cerbo-online-watch/mqtt.pw); for c in mosquitto_pub mosquitto_sub; do printf -- "-u mqttuser\n-P %s\n" "$PW" > /data/cerbo-online-watch/mosquitto/$c; chmod 600 /data/cerbo-online-watch/mosquitto/$c; done; svc -t /service/cerbo-online-watch'
```

### Der eigentliche Befund: Watchdog-Reset wegen Überlast (13.08.2026, abends)

Noch während der Wächter fertig wurde, **startete der Cerbo spontan neu** (13.08. 23:04 lokal) — der Wächter hat es live erfasst. Die anschließende Forensik ergab ein anderes Bild als gedacht.

**Reboot-Historie** (aus `/data/log/messages.{,0-5}`, Zeiten UTC):

| Zeitpunkt | Signatur | Ursache |
|---|---|---|
| 20.03. · 09.05. · 31.05. · 23.06. · 03.07. | je 1 Boot | Normalbetrieb, ~alle 3–6 Wochen |
| **12.08. 13:50** | **kein** Shutdown-Eintrag, Forensikdatei geschrieben | **Watchdog-Reset, Load 9.03/7.67/7.00** |
| 12.08. 20:24 | `shutdown[27489]`, Runlevel 6 | angefordert — **von Norbert bestätigt** (nach dem Deploy des lokalen ESS-Flows um 22:13 lokal) |
| 13.08. 21:04 | `shutdown[19471]`, Runlevel 6 | angefordert — **von Norbert bestätigt** |

**Fazit: Genau ein Reboot war ein echtes Ereignis** — der Watchdog-Reset am 12.08. 13:50 UTC (15:50 lokal). Er ist damit auch der Auslöser der ursprünglichen Frage („gestern hat der Cerbo neu gebootet, Grund unklar"): mitten am Nachmittag, ohne Zutun, und im Log genau an der Stelle unsichtbar, an der man zuerst nachsieht.

**Der Schlüssel ist `/etc/watchdog.conf`:** Venus OS fährt einen Software-Watchdog mit `max-load-15 = 6`, `max-load-5 = 10`, `min-memory = 2500`. Sein `repair-binary` (`/usr/sbin/store_watchdog_error.sh`) schreibt unmittelbar vor dem Reset `top -b -n1` nach **`/data/log/watchdog_processlist.txt`** und den Fehlercode nach `/data/watchdog.reset` (wird nach dem Boot vom System abgeräumt). Die Datei vom 12.08. 13:48 belegt: **15-min-Load 7.00 > Grenze 6 → Reset.**

⚠️ **Damit ist eine frühere Annahme widerlegt:** „Ein Watchdog-Reset hinterlässt nirgends eine Spur" stimmt nicht — die Prozessliste ist genau diese Spur, inklusive Load-Werten und Verursachern. (Memory entsprechend korrigiert.)

**Lastbild zum Reset-Zeitpunkt:** `dbus-modbus-client` 17 %, `venus-gui-v2` 9 % CPU / 21 % Speicher, `dbus_shelly` 4 % — und **`node-red` (187 MB)**: Auf dem Cerbo läuft das **lokale Node-RED des Large-Images** (`/service/node-red-venus`, für `flows/cerbo-lokal-ess-setpoint-soc.json`), zusätzlich zum Node-RED im LXC. Bei 2 Kernen und einem Normal-Load um 2,5–3,3 ist die Grenze 6 nicht weit weg. Vor dem 12.08. rebootete der Cerbo alle paar Wochen, seither dreimal in zwei Tagen — zeitlich zusammenfallend mit der Inbetriebnahme des lokalen ESS-Flows.

**Zu prüfen:** Ob der lokale Node-RED-Flow der Auslöser ist. Ansätze: Last des `node-red-venus`-Dienstes über `/proc/<pid>/stat` beobachten (⚠️ nicht `top -bn1`, das lügt hier — s. Memory), Poll-Intervalle im lokalen Flow entzerren, oder den Flow zurück in den LXC holen, falls er dort laufen kann. Die Grenzwerte in `/etc/watchdog.conf` anzuheben wäre Symptombekämpfung — der Watchdog schützt vor einem hängenden GX.

### Zwei Korrekturen aus dem Vorfall

1. **Log-Rotation berücksichtigt.** `syslogd` rotiert `/data/log/messages` bei ~100 kB — und das passiert gerne beim Boot, weil dann viel auf einmal geschrieben wird. Beim Reboot am 13.08. landete die Vorgeschichte in `messages.0`, die Startart wurde deshalb als `unbekannt` gemeldet. `boot-notify.sh` wertet jetzt `messages.0` + `messages` gemeinsam aus (Zwischendatei in `/run`). Rückwirkend geprüft: liefert für denselben Boot korrekt `geplant`, Vorlauf 88.843 s.
2. **Watchdog-Reset wird erkannt und benannt.** `boot-notify.sh` merkt sich einen Fingerabdruck (Größe + Zeitstempel) von `watchdog_processlist.txt` in `.watchdog_stempel`. Ändert er sich gegenüber dem letzten Boot, hat der Watchdog unmittelbar vorher geschrieben → `art=watchdog`, Prio 1, **mit Load-Werten und den größten Verbrauchern in der Push-Meldung**. Diese Prüfung hat **Vorrang** vor der Log-Einordnung: Der Watchdog kann über ein reguläres `shutdown -r` neu starten und sähe sonst wie ein harmloser geplanter Reboot aus.

### Endabnahme mit echtem Reboot (13.08., 23:14 lokal) ✅

Kontrollierter `reboot` mit der korrigierten Fassung:

```
23:14:43  reboot ausgelöst
23:14:47  binary_sensor.cerbo_gx_online → off      (trap meldete offline)
23:15:44  → on, Dienst von rc.local eingehängt (pid 1901)
          Boot-Meldung: art=geplant, uptime=45 s, vorlauf=618 s
          Watchdog-Stempel unverändert → kein Fehlalarm
```

Ausfalldauer ~57 s. **Energiezähler sauber weitergelaufen** (Netzbezug 2663,04 → 2663,07 kWh; Batterie entladen 632,3 → 632,5) — kein Rücksprung, kein falscher HA-Reset über zwei Reboots hinweg. Die in der Wartungs-Checkliste empfohlene Node-RED-Pause ist für einen kurzen Reboot demnach nicht nötig.

Der Unit-Test deckt jetzt 12 Fälle ab, inkl. Watchdog-Meldung: `node tools/test_cerbo_waechter.js`.

### Lastanalyse: Warum Dienste abschalten hier nicht hilft (13.08.2026) — Sammler zurückgestellt

**Anlass:** Frage von Norbert, ob sich `dbus-modbus-client` auf dem Cerbo deaktivieren lässt, um Last zu sparen.

**Antwort: Nein.** Der Dienst hat genau einen D-Bus-Service registriert — `com.victronenergy.grid.ve_HQ2338HC9FP`, also den **Gridmeter VM-3P75CT** (`udp:192.168.2.182:502`). Er ist dessen Treiber. Ohne ihn: kein Netzzähler → keine ESS-Nullausregelung, kein Netzbezug/Einspeisung in HA, Grid-Flows laufen leer.

⚠️ **Nicht verwechseln:** `dbus-modbus-client` (Client, liest externe Modbus-Geräte → unverzichtbar) vs. `dbus-modbustcp` (Server, stellt Cerbo-Daten bereit). Der Server hatte zwei aktive Verbindungen (192.168.2.48, 192.168.2.208) und kostet kaum Last.
**Zugeordnet am 2026-08-17:** Beides sind die **zwei evcc-LXCs** (105 auf `.48`, 119 auf `.208`,
s. [[homelab-infrastruktur]]). Der verwaist geglaubte 105 spricht also aktiv Modbus mit dem
Cerbo — er ist konfiguriert und läuft, nicht bloß eine Karteileiche.

**Dauerlast, sauber über 60 s aus `/proc/<pid>/stat` gemessen** (nicht `top -bn1`, das lügt hier):

| Prozess | % eines Kerns |
|---|---|
| `dbus-modbus-client` | 21 % |
| `venus-gui-v2` | 16 % |
| `node-red` (lokal, Large-Image) | 5 % |
| `dbus_shelly` | 5 % |

Summe ~47 % eines Kerns = ~24 % der zwei Kerne — bei Load 3,0.

**Der entscheidende Befund steht in der Watchdog-Forensikdatei selbst:**

```
Load average: 9.03  7.67  7.00
CPU:  34% usr  21% sys  0% nic  43% idle  0% io
```

**43 % idle bei Load 7.** Die CPU war beim Reset nicht ausgelastet. Der Load-Average misst auf diesem Gerät keine CPU-Not, sondern den Andrang lauffähiger und wartender Prozesse (~30 Python-Dienste im D-Bus-Takt, plus alles im D-State). Deckt sich mit der älteren Beobachtung „Load 3,3 trotz 46 % idle, ~1750 Kontextwechsel/s".

**Konsequenz:** Dienste abzuschalten adressiert die falsche Größe — CPU war nie der Engpass. Der Verdacht geht Richtung **I/O-Wartezeit** (D-State-Prozesse zählen voll in den Load, ohne CPU zu verbrauchen): eMMC-Schreiblast oder hängende Modbus-/Netzwerkzugriffe mit Timeouts. Auch das Anheben von `max-load-15` wäre Symptombekämpfung.

**Entscheidung (Norbert, 13.08.2026): Zurückgestellt.** Ein Sammler für Load, `iowait` und die Zahl der Prozesse in R/D-State (alle paar Minuten per MQTT nach HA, würde in den bestehenden `cerbo-online-watch`-Dienst passen) wird **erst gebaut, wenn der Cerbo erneut aussteigt**. Bis dahin übernimmt der Wächter die Beobachtung: Er meldet einen Watchdog-Reset jetzt eindeutig als solchen, samt Load-Werten und den größten Verbrauchern zum Reset-Zeitpunkt — das ist die Datenbasis, die beim letzten Mal gefehlt hat.

**Falls doch an Diensten gespart werden soll:** `venus-gui-v2` (16 %) ist der lohnendere Kandidat als der Modbus-Client — läuft dauerhaft, auch ohne offene Remote Console. Ganz verzichtbar ist sie aber nicht (Bedienoberfläche des Cerbo).

## Ersparnis durch Eigenverbrauch: Euro-Kennzahl auf dem Energie-Dashboard (15.08.2026) ✅

**Ausgangspunkt:** Die CO₂-Gauge auf „Energie (Sys)" sollte durch etwas Sinnvolleres ersetzt werden. Zwei Vorüberlegungen wurden dabei verworfen:

- Eine **CO₂-Kennzahl** (vermiedene Emissionen durch Einspeisung) wäre zwar rechenbar — 14.08. rund 20,3 kg bei einspeisungsgewichteten 212 g/kWh —, aber **folgenlos**: Das ESS speist ein, wenn die Sonne scheint; die Netz-CO₂-Intensität ist nicht beeinflussbar. Nebenbefund: Die Abendeinspeisung ist pro kWh **drei Mal so wertvoll** wie die Mittagseinspeisung (158 g/kWh um 13 Uhr vs. 496 g/kWh um 19 Uhr), weil mittags nur andere PV verdrängt wird.
- Eine **Eigenverbrauchs-Gauge** war überflüssig — die steht bereits im eingebauten `/energy`-Panel. Wichtig dabei: Deren 18 % und die naive Rechnung `(PV − Einspeisung)/PV` = 21,6 % sind **verschiedene Kennzahlen**. Die HA-Karte verfolgt Solarenergie bei vorhandener Batterie stundenweise per LIFO-Stapel durch den Speicher und **ignoriert Energie unbekannter Herkunft** (nachts entladene Vortagsenergie zählt weder im Zähler noch im Nenner). Bei Tag-Nacht-Zyklen liegt sie deshalb systematisch niedriger — kein Fehler, die strengere Definition.

**Gebaut wurde stattdessen** die Kennzahl, die HA strukturell fehlt: der Wert des **vermiedenen Einkaufs**. HA kennt nur Zahlungsvorgänge (Netzbezugskosten, Einspeisevergütung); dass eine selbst genutzte kWh 34,94 ct spart statt 7,382 ct einzubringen, taucht nirgends auf. Am 14.08.: **8,64 € aus 24,7 kWh Eigenverbrauch gegen 7,04 € aus 95,4 kWh Einspeisung — 20 % der Energie, 55 % des Geldes**, Faktor 4,7 je kWh. Das ist der Maßstab für jede Investitionsfrage (Ladefenster, Wärmepumpe, Speicher): nicht mehr erzeugen, sondern mehr behalten.

**Umsetzung:** Flow `victron-eigenverbrauch-wert-mqtt` (Tab `577ab5addc422ce0`), Generator `tools/gen_savings_flow.py`, Tests `tools/test_savings_flow.js` (8 Szenarien, alle grün).

- `Eigenverbrauch = PV + Batt_entladen − Einspeisung − Batt_geladen` — der Netzbezug kürzt sich heraus, also eine Quelle weniger, die ausfallen kann.
- Publiziert `victron/savings/self_consumption_value` (retained) → HA-Sensor `sensor.victron_ersparnis_eigenverbrauch`. **`device_class: monetary` erlaubt ausschließlich `state_class: total`** (im HA-Quellcode `DEVICE_CLASS_STATE_CLASSES` verifiziert) — `total_increasing` hätte dieselbe Log-Warnung erzeugt wie die PV-Prognose-Helfer.
- Zwei Wächter, der zweite erst nach einem fehlgeschlagenen Test ergänzt: (1) steht der Export-Zähler während die PV läuft und nimmt weder Batterie noch Wallbox die Energie auf → Fenster verwerfen; (2) Grenze auf den Eigenverbrauch **ohne Wallbox** (6 kW). Nur Wächter 1 reichte nicht — solange die Batterie lädt, fließt sichtbar Energie, nur viel zu wenig: Im Test (Export ab 10 Uhr eingefroren) buchte der Flow dadurch 20,79 € statt 8,65 €. Mit Wächter 2 bleiben 4,66 €, also eine Lücke statt eines Fehlwerts.

**Dashboard:** CO₂-Gauge ersetzt durch „Ersparnis durch Eigenverbrauch", daneben neu „Einspeisevergütung" (`…_netzeinspeisung_compensation`, die Statistik hatte HA schon). Beide als `statistic`-Karte mit `period: energy_date_selection`, folgen also der Datumsauswahl über Tag/Woche/Monat/Jahr. Wandlungsverluste rutschen eine Zeile tiefer.

**Offen:** Der Zähler startet bei 0, es gibt keine Historie. Rückwirkendes Füllen wäre über `recorder/import_statistics` aus den vorhandenen Stundenstatistiken möglich — bewusst nicht ohne Rücksprache gemacht.

**Electricity Maps** bleibt vorerst installiert, wird aber von nichts ausgewertet: keine Automation, kein Skript, kein Helfer, keine der 29 HA-Dashboards, keines der drei Grafana-Dashboards. Die Integration speist ausschließlich die drei automatischen Anzeigen des Energie-Systems (Low-Carbon-Kreis in der Energieverteilung, CO₂-Gauge im `/energy`-Panel, und die inzwischen ersetzte Gauge auf „Energie (Sys)"). Ersatzloses Entfernen wäre folgenlos.

### Nachtrag: Ertrag gesamt + Restore-Härtung (15.08.2026)

**Dritte Karte „Ertrag gesamt"** (Norberts Vorschlag) — die Summe aus vermiedenem Einkauf und Einspeisevergütung, also die Zahl für Amortisationsfragen. Sie steht in voller Breite über den beiden Einzelposten, die sie aufschlüsseln.

Die naheliegende Umsetzung wäre falsch gewesen: Ein Template-Sensor, der die beiden HA-Zählerstände addiert, bricht bei jedem HA-Neustart ein, weil `…_netzeinspeisung_compensation` **kein Lebenszeitzähler** ist (`last_reset` bei jedem Start, aktuell 0,003 €) — nur die Statistik dahinter läuft durch. Die Summe wird deshalb im Flow gebildet, wo die Einspeisung ohnehin als Delta vorliegt: `accTot += dSelf × 0,3494 + dExport × 0,07382`, publiziert auf `victron/savings/total_yield_value` → `sensor.victron_ertrag_gesamt`. Dass die Vergütung damit doppelt gerechnet wird (HA-intern und in Node-RED), ist gewollt — die beiden Werte sind eine brauchbare Gegenprobe.

**Restore-Härtung** — behebt die Deploy-Falle dauerhaft: Der Restore-Zweig belegt jetzt `now` aus dem retained `prev` vor, wenn der `_acc`-Satz frisch ist. Grund war die beim Erst-Deploy erlebte Falle, dass ein neuer `mqtt in` die retained Werte nicht zugestellt bekommt, solange ein anderer Flow dasselbe Topic über denselben Broker abonniert (Node-RED bündelt Subscriptions je Verbindung) — nachts fällt das voll durch, weil MPPT/Fronius/Batterie sich nicht ändern. Der zweite Deploy lief damit **ohne jeden Handgriff** durch. Das Muster gehört in jeden akkumulierenden Flow; Tests 10 und 11 decken beide Richtungen ab (frischer Restore rechnet weiter, alter wartet auf frische Quellen).

Tests jetzt 11 Szenarien, alle grün — u. a. Tagesverlauf 15,70 € gegen erwartete 15,71 €.

### Historie nachtragen: gescheitert und zurückgebaut (15.08.2026, 01:00–02:00)

**Ausgangsproblem:** Die beiden neuen Zähler starteten bei 0, während die daneben liegende Einspeisevergütung HAs volle Historie hat — die Summe wäre für Woche/Monat kleiner gewesen als einer ihrer Summanden. Der Versuch, das per `recorder/import_statistics` zu heilen, ist **fehlgeschlagen**.

**Die Rechnung war richtig, der Weg nicht.** Aus den Stundenstatistiken seit 1. Juli ergaben sich 309,04 € Ersparnis + 283,80 € Vergütung = 592,84 €; der Vergütungsanteil traf HAs eigene Statistik (283,79 €) auf den Cent — die Nachrechnung ist also belastbar. Nur ankommen tat sie nicht: `import_statistics` ist für **externe** Statistiken gedacht. Bei einer normalen Sensor-Entität führt der Recorder seine Summe unabhängig weiter und leitet sie aus dem State ab; eingefügte Zeilen ignoriert er. Gemessen: `sum=0.306` bei `state=0.306`, also `change=-592.5` gegen die importierte Historie. **Auch ein HA-Neustart half nicht** (getestet, 01:04) — der Recorder liest seinen Ausgangswert nicht aus der hourly-Tabelle.

**Zurückgebaut** per `recorder/clear_statistics` auf beide Zähler: Historie und Einbruch sind zusammen verschwunden, die Zähler laufen unangetastet weiter (0,306 € / 0,304 € um 01:55), HA baut ab 02:00 saubere Statistik auf. Die Karten zeigen ab jetzt korrekte Werte, Juli und die erste Augusthälfte fehlen dauerhaft.

**Wer es doch braucht:** eigene externe Statistik (`statistic_id` mit Doppelpunkt, `source != "recorder"`) plus zweite Dashboard-Karte. Eine Sensor-Entität lässt sich nicht rückwirkend füllen. Rechenlogik dafür liegt in `tools/import_savings_history.py` (mit Warnhinweis im Kopf).

**Zweiter Befund derselben Nacht — der Flow stand nach jedem Redeploy still.** `_diag` lieferte den Beweis für die MQTT-Bündelung: `seen: [exp, ev]` — exakt die beiden Topics, die nur dieser Flow abonniert; die fünf fehlenden (mppt1/2, fronius, chg, dis) liest auch der Wandlungsverluste-Flow. Node-RED bündelt Subscriptions je Broker-Verbindung, der Broker stellt retained dann nicht erneut zu. Behoben im Tick: fehlende Werte werden aus `prev` aufgefüllt (Delta 0 bis frische Werte kommen). Der zwischenzeitlich eingebaute `reload`-Inject wurde wieder **entfernt** — er setzte `restored=false`, das erwartete `_acc` kam nie an, und weil der Tick mit `if (!st.restored) return null` beginnt, legte er den Flow lautlos komplett still.

### Gegenprobe „Ertrag gesamt" vs. Summe der Einzelposten (15.08.2026, mittags)

**Beobachtung (Norbert):** Karte „Ertrag gesamt" 3,79 €, Ersparnis + Einspeisevergütung von Hand addiert 3,83 €. Die Differenz ist echt, aber **kein Rechenfehler — sie ist ein Abtastversatz.**

Gemessen an den Stundenstatistiken 02:00–11:00: Ertrag gesamt **3,117 €** gegen Ersparnis 1,692 € + Vergütung 1,458 € = **3,150 €**, also −0,033 €. Die Nachtstunden sind deckungsgleich, die Lücke entsteht erst ab 07:00 mit der Einspeisung und wächst mit deren Leistung.

**Ursache:** Der Flow tickt alle 300 s (Inject `repeat: 300`) und publiziert um `hh:mm:53`. Der letzte Wert vor einem Statistik-Stundenende stammt damit von `hh:55:53` — **die letzten gut vier Minuten jeder Stunde fehlen dem Node-RED-Zähler**, während HAs `…_netzeinspeisung_compensation` dem Gridmeter sekundennah folgt. Bei 0,5–0,6 €/h Einspeisung sind genau das die 0,03–0,05 €. Der Rückstand ist reiner Nachlauf, kein Verlust: Der nächste Tick holt ihn ein.

**Beweis über das 5-Minuten-Raster** (dort beträgt der Versatz auf beiden Seiten gleich ~4 min und kürzt sich im Delta heraus): 09:00 → 11:40 Node-RED-Exportanteil (Ertrag − Ersparnis) **1,314 €** gegen HA **1,323 €** — Restabweichung 0,009 €, exakt der Betrag, den die *Änderung* der Einspeiserate über den Versatz erklärt (0,145 €/h × 4,1 min). **Beide Rechenwege stimmen also auf unter 1 % überein**; die bewusst doppelte Vergütungsrechnung hat ihre Gegenprobe bestanden.

Dazu kommen 0,004 € aus den Stunden 00:00–02:00, für die es nach dem `clear_statistics` der Vornacht keine Statistik gibt.

**Konsequenz:** Solange die Einspeisung zum Periodenende noch läuft (Tagesansicht am Nachmittag), liegt „Ertrag gesamt" strukturell ein paar Cent unter der Summe seiner beiden Karten. Bei Woche/Monat endet die Periode nachts bei Einspeisung 0 → Effekt praktisch weg. **Gegenprobe abends fahren:** Geht die Tagesdifferenz nicht auf ~0,004 € zurück, wären doch verworfene Fenster im Spiel.

**Möglicher Fix (nicht umgesetzt):** zusätzlicher Cron-Inject kurz vor der vollen Stunde (`hh:59:30`) statt kürzerem Tick-Intervall — 60-s-Ticks würden die Deltas gegen die grobe Quantisierung der Quellzähler (Batrium 0,1 kWh) drücken und alle Wächterschwellen (`MAXHOUSE` 0,5 kWh/Tick usw.) entwerten.

**Warum die Summe nicht einfach in HA gebildet wird** (Rückfrage Norbert, 15.08.): Weil `…_netzeinspeisung_compensation` kein Lebenszeitzähler ist — sein `state` zählt nur seit dem letzten `last_reset` (aktuell `2026-08-14T23:04:42Z`, der dokumentierte Neustart um 01:04). Tages-`state` der letzten 10 Tage: 78,53 → 0,005 → 8,15 → 16,83 → 5,39 → 0,018 → 7,59 → 15,09 → 22,22 → 0,003 €, während `sum` sauber auf 285 € durchlief. Ein Template- oder Gruppen-Sensor „Ersparnis + Vergütung" bräche also bei jedem HA-Neustart um bis zu 78 € ein — und HA läse den Absturz eines `total`-Sensors als Zählerreset. **Nicht die Addition ist der Grund für den Flow**, sondern der Eigenverbrauchswert, den HA strukturell nicht kennt; `accTot` ist darin eine einzige zusätzliche Zeile in einem Tick, der ohnehin läuft.

**Umgesetzt (15.08.2026, 12:20):** Zusatz-Tick `crontab: "30 59 * * * *"` als zweite Inject-Node (`clsav0tick59901`) auf denselben Function-Eingang — dasselbe `topic: "tick"`, also keine Zeile im Rechenkern. Vorher verifiziert statt angenommen: `20-inject.js` reicht den String unverändert an `cronosjs` weiter (`scheduleTask(this.crontab, …)`, keine Validierung), und laut cronosjs-README gilt bei sechs Feldern das erste als Sekundenfeld. **Der Editor-Dialog kann 6-Feld-Ausdrücke aber nicht darstellen** (`oneditprepare` liest `cronparts[0]` als Minuten) — wer die Node im Editor öffnet und speichert, zerstört den Ausdruck. Warnung steht im Node-Namen, im Generator-Kommentar und in der Tab-Info. Deploy per neuem `tools/deploy_savings_flow.py` (PUT auf `/flow/577ab5addc422ce0`), Restore lief sauber durch (Zählerstand 5,009 € statt Neustart bei 0), alle 11 Tests grün.

**Verifiziert (15.08.2026, 13:01, erster Stundenrand-Tick um 12:59:30):** Tageskumulation nach dem 12:00-Bucket — Ersparnis 2,844 € + Vergütung 2,826 € = 5,670 € gegen Ertrag gesamt 5,658 €. **Differenz von 0,046 € auf 0,012 € gefallen** (1,1 % → 0,21 %), obwohl die Einspeiserate in der Zwischenzeit von 0,565 auf 0,798 €/h gestiegen ist — der Effekt ist also größer, als die nackte Zahl zeigt. Aufschlüsselung des Rests: 0,0044 € sind die nicht mehr herstellbaren Nachtstunden 00:00–02:00 aus dem `clear_statistics`, die verbleibenden 0,0074 € entsprechen bei 0,798 €/h genau **33 Sekunden Versatz** — erwartet waren 30. Die Rechnung stimmt damit auf drei Sekunden. Auf den Karten sind es jetzt 2,84 + 2,83 gegen 5,66, also ein Cent Rundungsdifferenz statt vier.

Im 12:00-Bucket selbst steht „Ertrag gesamt" mit 1,492 € **über** der Summe der Einzelposten (1,458 €) — das ist der Nachholeffekt: Der Tick um 12:59:30 hat die vier Minuten eingesammelt, die am Ende der 11:00-Stunde fehlten, und bucht sie in die 12:00-Stunde. Einmaliger Übergang, danach selbstkorrigierend.

**Was die Karte wirklich anzeigt** (Fehlannahme unterwegs, korrigiert): Die `statistic`-Karte mit `period: energy_date_selection` zeigt den Stand **bis zur letzten abgeschlossenen Stunde**, nicht bis zur laufenden Minute. Beleg aus dem Betrieb: Norbert las um 12:04 die Werte 2,18 / 2,03 ab — exakt die Summe der `change`-Buckets bis 12:00:00. Hätte die Karte die laufenden vier Minuten mitgerechnet, stünden dort rund 0,04 € mehr. Wichtig für jede künftige Nachrechnung: **immer gegen Bucket-Grenzen prüfen, nie gegen `state` zur Ablesezeit.** (Zwischenzeitlich hatte ich die Buckets um eine Stunde verschoben zugeordnet und daraus das Gegenteil geschlossen — `1786784400` ist 11:00, nicht 12:00.)

### Wo darf gerechnet werden? Frontend, Backend, Statistik (15.08.2026)

Aufgeworfen von Norbert aus der Unterrichtsperspektive: *„Meinen Schülern predige ich, sowas nicht im Frontend zu tun — wird das Frontend ausgetauscht, ist die Summenberechnung dahin."* Die Frage zwang zu einer Korrektur und lieferte nebenbei ein Lehrbeispiel.

**Korrektur:** Der Kartenwert wird **nicht** im Browser berechnet. Belegt in `hui-statistic-card.ts`:

```ts
const stats = await fetchStatistic(hass, this._config.entity,
  { fixed_period: { start: this._energyStart, end: this._energyEnd } });
this._value = stats[this._config.stat_type];
```

und `fetchStatistic` in `data/recorder.ts` ist nur ein WS-Aufruf: `{ type: "recorder/statistic_during_period", statistic_id, fixed_period }`. Das Frontend schickt zwei Parameter — welche Statistik, welcher Zeitraum — und liest ein Feld aus der Antwort. Die Aggregation läuft im Recorder, also in Python.

**Die Abgrenzung, die daraus folgt:** Was im Frontend liegt, ist der *Zeitraum* — ein Query-Parameter. Strukturell dasselbe wie `SELECT SUM(betrag) … WHERE datum BETWEEN ? AND ?`: Filter aus dem UI, Summe aus der DB. Wirf das Frontend weg und schreib es neu — keine Zahl geht verloren, nur eine Ansicht.

**Die frühere Aussage „im Backend gibt es diesen Wert nicht" bleibt trotzdem richtig, aber aus anderem Grund:** nicht falscher Ort, sondern **kein materialisierter Wert**. Der Kartenwert ist ein Query-Ergebnis ohne Entität. Genau deshalb lässt er sich nicht addieren — und genau deshalb muss die Summe irgendwo materialisiert werden (Node-RED-Zähler oder externe Statistik). Norberts Fall ist damit die praktische Bestätigung seines eigenen Prinzips.

**Das Gegenbeispiel im selben System** — hier verletzt HA das Prinzip wirklich: `calculateSolarConsumedGauge()` in `frontend/src/data/energy/index.ts`. Die stundenweise LIFO-Verfolgung der Solarenergie durch die Batterie ist eine *fachliche Definition* von Eigenverbrauch und existiert ausschließlich im Frontend-TypeScript. Folgen im eigenen Betrieb erlebt: Die 18 % waren ohne Lesen des Quellcodes nicht nachrechenbar, und keine Automation, kein Node-RED, kein Grafana kann die Zahl je verwenden.

**Als Unterrichtsmaterial** — gleiches System, zwei Karten, eine sauber, eine nicht. Prüffragen:
1. *Wenn ich das Frontend lösche und neu schreibe — verliere ich Wissen oder nur Ansicht?* Statistik-Karte: nur Ansicht. Eigenverbrauchs-Gauge: Wissen.
2. *Kann etwas anderes als ein Bildschirm diesen Wert brauchen?* Wenn ja → Backend, materialisiert.
3. *Aggregieren zur Anzeigezeit ist erlaubt; Fachbegriffe definieren nicht.* Summe über einen gewählten Zeitraum = Präsentation. „Was zählt als Eigenverbrauch" = Domäne.

Fair bleiben: `energy_date_selection` ist ein impliziter globaler Frontend-Zustand, über den mehrere Karten gekoppelt sind — Kopplung über globalen Zustand ist eine Designschwäche, aber Präsentationszustand, kein Fachwissen.

### Offene Punkte aus diesem Strang

- ~~**Abend-Gegenprobe (21:30 am 15.08.2026, Timer läuft)**~~ → **erledigt, siehe unten.**
- **Externe Statistik `victron:ertrag_gesamt` — zurückgestellt, aber jetzt belegt machbar.** Die `statistic`-Karte akzeptiert externe statistic_ids ausdrücklich (`hui-statistic-card.ts`, Zeile 156: `isExternalStatistic(config.entity)` schlägt die Gültigkeitsprüfung nicht durch; nur `interactive` wird false, es gibt ja keine Entität für den More-Info-Dialog). Damit wäre die Summe per Konstruktion exakt — sie käme aus denselben Stundenwerten wie die beiden Einzelkarten — und die Historie ab 1. Juli ließe sich nachladen, weil `import_statistics` für **externe** Statistiken der vorgesehene Weg ist (an einer Sensor-Entität war genau das gescheitert). Preis: ein stündlicher Job gegen die WS-API mit eigenem Long-Lived-Token, und die unabhängige Gegenprobe zweier Rechenwege entfiele. **Entscheidung: erst laufen lassen, bis der Zähler eigene Historie hat.**

### Abend-Gegenprobe bestanden (15.08.2026, 21:15) ✅

Gefahren mit den Stundenbuckets **02:00–20:00** (die Nachtstunden 00:00/01:00 haben nach dem `clear_statistics` der Vornacht keine Node-RED-Statistik). Einspeisung war zum Periodenende faktisch aus: `change` der Vergütung im 20:00-Bucket 0,0030 €, im 19:00-Bucket 0,0125 € — die Bedingung „Einspeisung 0" war damit schon vor 21:30 erfüllt, ein Warten hätte nichts hinzugefügt (die 21:00-Stunde wäre um 21:30 ohnehin noch offen gewesen).

| | 02:00–20:00 |
|---|---|
| Ertrag gesamt (Node-RED) | **13,214 €** |
| Ersparnis Eigenverbrauch (Node-RED) | 7,571 € |
| Einspeisevergütung (HA) | 5,642 € |
| Summe der Einzelposten | **13,213 €** |
| Differenz | **+0,001 €** |

**Damit ist die Frage beantwortet, die den Termin ausgelöst hat:** Der Rückstand ist restlos verschwunden — mittags waren es noch 0,046 €, nach dem `59:30`-Tick 0,012 €, jetzt 0,001 €. **Also kein einziges verworfenes Fenster über den ganzen Tag.** Ein von den Wächtern verworfenes Fenster hätte den Exportanteil im `accTot` gekürzt, ohne HAs Vergütungsstatistik zu berühren; genau dieser Anteil stimmt aber: Node-RED-Export (13,214 − 7,571 = 5,643 €) gegen HA 5,642 €. `_diag` und die `node.warn`-Ausgaben mussten nicht bemüht werden.

Auf der Tageskarte bleibt die bekannte Nachtlücke stehen: Die Vergütungskarte zählt die Buckets 00:00 und 01:00 mit (2× 0,0022 €), die Node-RED-Zähler nicht → Kartensumme 13,217 € gegen 13,214 €, **0,003 € — exakt der vorhergesagte Rest**. Der 33-Sekunden-Versatz hat sich über neun Stundenränder (12:59:30 bis 20:59:30) gehalten, statt nur über den einen von mittags.

**Betriebsstand:** Zähler laufen sauber durch (21:14: Ertrag gesamt 13,59 €, Ersparnis 7,945 €), Tick alle 300 s plus Stundenrand. Der Strang ist damit abgeschlossen; offen bleibt nur die zurückgestellte externe Statistik.

#### Zweite, unabhängige Gegenprobe — verworfene Fenster jetzt wirklich ausgeschlossen (21:25)

**Die erste Gegenprobe hat eine strukturelle blinde Stelle**, die beim Aufschreiben auffiel: Sie vergleicht Ertrag gesamt gegen Ersparnis + HA-Vergütung. Ein verworfenes Fenster kürzt aber **beide** Node-RED-Zähler gleichzeitig — die Differenz bleibt 0. Sichtbar werden dort nur Verwerfungen **mit Einspeisung im Fenster**; ein nachts von Wächter 2 (`MAXHOUSE`) verworfenes Fenster wäre unsichtbar geblieben.

Geschlossen über eine Energiebilanz aus **HAs eigenen Zählern**, gerechnet nach derselben Formel wie im Flow (02:00–20:00):

| | kWh |
|---|---|
| PV gesamt | 102,00 |
| Batterie entladen | +4,50 |
| Netzeinspeisung | −76,43 |
| Batterie geladen | −8,40 |
| **Eigenverbrauch** | **21,670** |

21,670 kWh × 0,3494 €/kWh = **7,571 €** gegen die 7,571 € des Node-RED-Zählers — **Abweichung 0,001 kWh**. Damit ist belegt: **über den ganzen Tag wurde kein einziges Fenster verworfen**, auch nicht nachts. `_diag` und `node.warn` sind nicht nötig (und wären mangels MQTT-Client auf dem Arbeitsrechner und ohne SSH auf den LXC gerade auch nicht lesbar).

**Merksatz für künftige Prüfungen:** Zwei Rechenwege gegeneinander zu stellen beweist nur so viel, wie sie sich *unterscheiden*. Ertrag vs. Ersparnis+Vergütung teilt sich den `dSelf`-Term und kann Fehler darin nicht sehen. Erst die Bilanz aus fremden Zählern ist unabhängig.

#### Fehlspur unterwegs: „Der Flow steht" (war er nicht)

Der Sensor meldete um 21:22 als letzten Wert 21:14:07, das 5-Minuten-Raster schien Lücken zu haben (21:09, 21:19 fehlten). **Fehlalarm — und der Kontrolltest, der ihn aufgelöst hat, gehört ins Werkzeug:** In dieser HA-Instanz ist `last_reported` **immer identisch mit** `last_changed` (geprüft an fünf Victron-Sensoren, u. a. `…_fronius_leistung`, seit 20:42 unverändert 0 trotz laufender Quelle). Ein unveränderter Wert hinterlässt also **keine Spur** — aus einem stehenden Zeitstempel folgt hier *nicht*, dass die Quelle stumm ist.

Die „fehlenden" Ticks sind die **0,1-kWh-Quantisierung des Batrium-Zählers**: 0,1 kWh × 0,3494 €/kWh = 0,035 € — exakt die beobachtete Schrittweite. Nachts bei ~750 W dauert ein Quant 8 Minuten, also ändert sich der Wert nur bei jedem zweiten Tick.

Belegt wurde der laufende Betrieb stattdessen über zwei andere Wege: parallele Node-RED-Sensoren aus anderen Flows meldeten frisch (Wandlungsverluste 21:19:35, Roundtrip 21:20:18), und der Live-Abruf `GET /flow/577ab5addc422ce0` von der Admin-API zeigt den Tab aktiv mit **intaktem `crontab: "30 59 * * * *"`** — die dokumentierte Editor-Falle hat nicht zugeschlagen.

#### Sicherungsstand (15.08.2026, 21:35)

**vzdump des Node-RED-LXC um 21:35 gezogen (Norbert).** Es enthält damit den um 21:23 per Admin-API verifizierten Live-Stand: 19 Tabs / 308 Objekte, inklusive „Victron Eigenverbrauch-Wert (MQTT)" mit dem Stundenrand-Tick. Die lokalen Exporte in `backups/` (17 bzw. 18 Tabs, letzter 14.08. 23:51) sind damit überholt — sie waren ohnehin Handarbeit, es gibt kein Skript und keinen Cron dafür.

### Sicherungslücken-Inventur: was ist wo gesichert? (15.08.2026, 22:30)

Auf Norberts Frage „sind alle Flows gesichert?" per Node-ID-Abgleich Live gegen `flows/` geprüft — nicht geschätzt.

**Blinder Fleck, der dabei auffiel: Es gibt ZWEI Node-RED-Instanzen.** Neben dem LXC (`192.168.2.80:1880`, Node-RED 5.0.1, 19 Tabs) läuft auf dem **Cerbo selbst** eine zweite (`https://192.168.2.181:1881`, Node-RED 4.1.1, 2 Tabs / 17 Nodes: „Flow 1" und „SOC einstellen") — die ESS-Setpoint- und SoC-Steuerung, bewusst ans Gerät gelegt (Messen in den LXC, Regeln ans Gerät). **Das vzdump des LXC erfasst sie nicht.** Entwarnung: Ihre 17 Node-IDs sind mit `flows/cerbo-lokal-ess-setpoint-soc.json` **exakt deckungsgleich** (17/17, keine Drift) — sie hängt also an genau einer Kopie im Repo, und die stimmt.

**LXC-Tabs, vollständig als Einzelexport im Repo (12 von 19):** Grid (aus `victron-grid-energie-mqtt` + `victron-grid-leistung-glatt`), Batterie, openWB EV, Leistung, PV (aus `victron-pv-energie-mqtt` + `victron-pv-summe-mqtt`), Speicher-Wirkungsgrad, Zigbee BW-WP, MultiPlus, Wandlungsverluste, Cerbo-Wächter, Eigenverbrauch-Wert — sowie **Fronius-Ladedeckel**: dessen Node-IDs weichen ab (per Editor-Import deployt, der vergibt neue IDs), Typ+Name sind aber Knoten für Knoten identisch mit `victron-ladedeckel-auto.json`. Merkposten: **Ein ID-Vergleich allein hätte ihn fälschlich als ungesichert gemeldet.**

**Nur im vzdump, kein Repo-Export (7 Tabs, ~91 Nodes):** „Flow 1" (4), „Flow 2" (2), „MQTT Keepalive Control" (5), „UDP Akku" (2), „Flow 3" (3), „Flow 4" (2) und der Hauptteil von **„WatchMon" (73 von 90 Nodes** — abgedeckt sind nur die 17 aus `batrium-schuetz-pushover` + `wassersensor-leck-pushover`). Das ist der handgebaute Batrium-Altbestand und damit der wertvollste Teil ohne Einzelsicherung.

**Nachgeprüft: die globalen Config-Nodes** (liegen ohne `z` außerhalb aller Tabs und fallen bei einer Tab-Inventur durchs Raster). LXC: 4 Stück, davon **2 nicht im Repo** — die MQTT-Broker „Proxmox Mqtt" und „Victron Mqtt". Ein Restore allein aus `flows/` ergäbe also mqtt-Nodes, die auf nicht existierende Broker-IDs zeigen; Host, Port, Keepalive und LWT müssten von Hand nachgebaut werden, Passwörter ohnehin (die stehen nur in `flows_cred.json` im vzdump). Cerbo: **5/5 im Repo**, inklusive der Dashboard-Nodes (`ui-base`, `ui-theme`, `ui-page`, `ui-group`) — die Instanz ohne eigenes Backup ist damit als einzige vollständig aus dem Repo rekonstruierbar.

**Lücke geschlossen (15.08.2026, 22:45): `flows/batrium-watchmon.json`** — 92 Objekte, 140 KB: 1 Tab + 90 Nodes + die Config-Node „Proxmox Mqtt" (damit ist auch eine der beiden fehlenden Broker-Configs im Repo). Inhalt: 44 `function`, 34 `debug`, 3 `inject`, 3 `comment`, 1 `udp in`, 1 `switch`, 1 `http request`, 3 mqtt.

**Beim Export wären beinahe echte Zugangsdaten ins Repo gewandert.** Die Function-Node „Pushover-Request bauen" enthält im **laufenden** Flow das echte Application-Token und den User-Key im Klartext; die Repo-Fassungen tragen bewusst Platzhalter (`PUSHOVER_APP_TOKEN` / `PUSHOVER_USER_KEY`, Konvention aus `batrium-schuetz-pushover.json`, vgl. Kopfkommentar von `deploy_cerbo_flow.py`). Beide Zeilen wurden vor dem Schreiben ersetzt. Gegenprobe per Diff Live↔Datei: **genau eine Node weicht ab, genau in diesen zwei Zeilen** — alles andere ist bitgleich. Angenehmer Nebeneffekt der bestehenden Code-Logik: Ein versehentlicher Re-Import würde wegen `TOKEN.startsWith('PUSHOVER_')` nur warnen und die Meldung verwerfen, statt kaputte Requests zu senden.

Die MQTT-Broker-Config enthält keine Zugangsdaten (`user`/`password` liefert die Admin-API nicht mit, die stehen in `flows_cred.json`) — der Export ist damit repo-tauglich.

**Bewusste Redundanz:** Die 14 Nodes aus `batrium-schuetz-pushover.json` und die 3 aus `wassersensor-leck-pushover.json` sind vollständig in der neuen Datei enthalten (14/14 bzw. 3/3) — sie sind Teilmengen des WatchMon-Tabs, weil beide Erweiterungen dort eingebaut wurden. Beim Wiederherstellen also **nur `batrium-watchmon.json` importieren**, sonst entstehen Dubletten.

**Restliche sechs Tabs exportiert (15.08.2026, 23:00)** — in drei Dateien statt sechs, weil vier davon Experimentierreste mit 2–4 Nodes sind:

| Datei | Inhalt | Bewertung |
|---|---|---|
| `flows/mqtt-keepalive-control.json` | 5 Nodes | **produktiv** — zwei `exec`-Nodes starten/stoppen eine `mosquitto_pub`-Schleife auf `R/c0619ab31b3b/keepalive` (alle 30 s, gegen 192.168.2.181), PID in `/tmp/keepalive.pid`. Ohne sie stellt der Venus-Broker das Senden ein |
| `flows/udp-akku-sniffer.json` | 2 Nodes | `udp in` 18542 + debug |
| `flows/altbestand-flow-1-4.json` | 4 Tabs, 11 Nodes | „Flow 1" MQTT-Test-Publisher (inject→random→mqtt out), „Flow 2" Venus-MQTT-Sniffer auf `N/c0619…/battery/512/Dc/0/Voltage`, „Flow 3" Spielwiese, „Flow 4" **disabled** (victron-input-system + debug) |

**Nebenbefund mit Substanz — der dokumentierte Port-18542-Konflikt hat einen konkreten Verursacher:** „UDP Akku" und „WatchMon" haben **beide einen aktiven `udp in` auf Port 18542**, keiner davon deaktiviert. Der WatchMon-Listener ist der produktive; „UDP Akku" ist ein Debug-Rest, der ihm die Batrium-Pakete streitig macht. Kandidat zum Löschen (nicht angefasst, kein Auftrag).

Die `exec`-Kommandos enthalten **keine Zugangsdaten** (Venus-MQTT läuft lokal ohne Auth) — nur die VRM-Portal-ID `c0619ab31b3b`, relevant nur, falls das Repo je öffentlich wird.

**Damit ist die Inventur geschlossen:** alle **19 LXC-Tabs** und die **Cerbo-Instanz (17/17)** im Repo, **Config-Nodes 4/4** — die beiden fehlenden Broker („Proxmox Mqtt", „Victron Mqtt") kamen mit `altbestand-flow-1-4.json` mit.

**Einzige Restunschärfe: „Fronius-Ladedeckel (Auto)".** Seine Node-IDs weichen ab (Editor-Import), eine ID-Inventur meldet dort weiterhin 0/16. Inhaltlich geprüft: von 16 Nodes weichen **2 in je einem kosmetischen Feld** ab — ein `debug` steht live auf `active: false` (im Repo `true`), und `initialValue` ist live `null` statt `""`. Die Logik ist identisch, die Repo-Datei also aktuell. Ein ID-genauer Neu-Export würde nur die Inventur glätten.

**Fazit:** Es ist nichts ungesichert — aber die Ebenen sind ungleich verteilt. Die 7 Alt-Tabs hängen allein am vzdump (grob, aber vollständig), die Cerbo-Instanz allein am Repo (fein, aber ohne eigenes Geräte-Backup). Kein akuter Handlungsbedarf, beide Kopien sind aktuell und geprüft.

### Schutz vor versehentlichen Änderungen: Flow-Sperre verifiziert (15.08.2026, 22:15)

**Anlass (Norbert):** *„Was kann man tun, damit nichts versehentlich am Flow geändert wird?"* — dahinter die berechtigte KISS-Kritik, dass die Cron-Node aktuell nur durch einen **Warnhinweis im Node-Namen** geschützt ist, also durch die schwächste denkbare Maßnahme. Übertragen auf das STOP-Prinzip aus dem Arbeitsschutz (Substitution → Technisch → Organisatorisch → Persönlich) sichern wir eine Falle bislang mit einem Warnschild.

**Node-RED 5.0.1 auf dem LXC kann Tabs sperren** (`locked: true`, Feature seit 3.1). Geprüft wurde nicht die Doku, sondern das **tatsächlich ausgelieferte Editor-Bundle** des laufenden Servers (`GET http://192.168.2.80:1880/red/red.js`, 2,8 MB, unminifiziert):

| Fundstelle | Wirkung |
|---|---|
| `showEditDialog`, Z. 41356: `if (node.z && RED.workspaces.isLocked(node.z)) { return }` | Der Node-Dialog **öffnet gar nicht** — stärker als read-only |
| `showEditWorkspaceDialog`, Z. 22488 | Flow-Properties per Tab-Doppelklick blockiert |
| `deleteWorkspace`, Z. 22465 | Tab löschen blockiert |
| ~20× `if (RED.workspaces.isLocked()) { return }` in der View-Logik | Verschieben, Einfügen, Löschen, Gruppieren |
| `ensureUnlocked` nach dem Deploy, Z. 18950 | Node-RED **entsperrt selbst temporär**, um interne Flags zu setzen — Kommentar im Quelltext: *„Node's properties cannot be modified if its workspace is locked."* |

**Damit ist die Cron-Falle technisch ausschließbar:** Der `oneditprepare`-Code, der den Sechs-Feld-Ausdruck zerstört, läuft nie, weil der Dialog nicht aufgeht.

**Die Falle bei der Umsetzung** — belegt in `tools/deploy_savings_flow.py`: Das Skript baut sein Payload explizit aus `id, label, info, disabled, nodes, configs`. `PUT /flow/<id>` ersetzt das komplette Tab-Objekt, also **entfernt jeder Deploy eine im Editor gesetzte Sperre still wieder**. Die Sperre muss in `gen_savings_flow.py` (Tab-Definition, Z. 328) *und* ins Deploy-Payload — sonst baut man eine Schutzmaßnahme, die sich bei jeder Wartung selbst abschaltet.

**Grenze der Prüfung:** Quellcode-Lektüre am ausgelieferten Bundle, kein Klicktest im Browser.

**Nicht empfohlen:** Node-RED-Projects (löst dasselbe, migriert aber das Arbeitsverzeichnis), `adminAuth` als Versehensschutz (man meldet sich ohnehin als Admin an), Selbstüberwachung des Ticks im Flow (Komplexität, die Komplexität absichert). *Getrennt davon offen:* Die Admin-API auf `192.168.2.80:1880` läuft ohne Authentifizierung — Sicherheitsthema, kein Versehensschutz.

**Die beiden Sicherungsebenen ersetzen einander nicht:** Das vzdump enthält als einziges `flows_cred.json` **samt** `credentialSecret` aus `settings.js`, dazu Palette und Umgebung (`NODE_RED_DBUS_ADDRESS`) — ein reiner `flows.json`-Export käme ohne MQTT-Passwörter zurück. Umgekehrt kann nur der Export einen **einzelnen Tab** zurückholen, ohne den halben Container auf einen alten Stand zu drehen. Dritte Ebene und für die Victron-Flows die eigentliche: die Generatoren in `tools/`, mit denen die Flows *reproduzierbar* statt nur gesichert sind. Nicht generiert sind die handgebauten Alt-Tabs („Flow 1–4", „UDP Akku", „MQTT Keepalive Control", „WatchMon") — sie hängen an Backup und Export, seit 15.08. 23:00 aber immerhin an beidem.

### Wie geht der Akku in den Ertrag ein? — und der Befund dahinter (15.08.2026, 23:30)

**Norberts Frage:** *„Wie berücksichtigst du den Akku bei der Berechnung des Ertrags?"* Formal über zwei Terme in `dSelf = dPv + d.dis − d.exp − d.chg`: **Entladung positiv, Ladung negativ.**

**Was daran richtig ist:**
- **Der Speicher ist ein Puffer, kein Ertrag.** Geladene Energie wird abgezogen und erst gutgeschrieben, wenn sie wieder herauskommt. Über die Lebensdauer korrekt; nur die *Tages*kennzahl verschiebt sich (abends volladen drückt den Tageswert, der Folgetag holt ihn).
- **Die DC-Roundtrip-Verluste mindern den Eigenverbrauch von selbst.** Was hineingeht, wird voll abgezogen; was nie herauskommt, wird nie gutgeschrieben. Es braucht dafür **keinen Wirkungsgradfaktor im Code** — ein Fall, in dem eine schlichte Bilanz das Richtige tut.

#### Und die Verluste? Zwei Arten, zwei völlig verschiedene Behandlungen

Norberts Nachfrage (*„Verluste sind ja kein Ertrag"*) trifft eine **Asymmetrie**, die nicht in der Formel steckt, sondern in der **Lage der Messstellen**:

| Verlustart | Größe | Behandlung | Warum |
|---|---|---|---|
| **Batterie-Roundtrip (DC)** | nicht messbar (s. u.) | **korrekt ausgeschlossen** | `chg` und `dis` messen **beide auf der DC-Seite** — die Differenz ist echt |
| **Wandlung DC→AC im MultiPlus** | ~3,6 kWh/Tag | **fälschlich als Ertrag gebucht** | `mppt`/`dis` messen DC, `exp` misst AC — der Verlust fällt *zwischen* die Zähler |

**Der elegante Teil:** Für die Batterie muss die Formel gar nicht unterscheiden, ob Energie *noch drin* oder *verloren* ist — sie schreibt schlicht gut, was herauskommt. Verlorenes kommt nie heraus und wird nie gutgeschrieben; Gespeichertes kommt später heraus und wird dann gutgeschrieben. Ein Wirkungsgradfaktor wäre nicht nur überflüssig, sondern falsch.

**Die Größe des DC-Roundtrip-Verlusts ist aus diesen Zählern nicht bestimmbar** — geprüft über 08.–14.08. (7 volle Tage, jeder erreicht SoC 100 %, die gespeicherte Energie hebt sich also weitgehend auf): geladen 56,7 kWh, entladen **57,3** kWh. Es kam **0,6 kWh mehr heraus als hinein** — physikalisch unmöglich, also liegt das Messrauschen (Batrium-Quantisierung 0,1 kWh, SoC-Versatz an den Tagesgrenzen, evtl. Offset zwischen Lade- und Entladezählung) **über** dem Effekt. Für LiFePO4 wären 2–4 % zu erwarten, hier ~1,5–2 kWh auf die Woche — nicht auflösbar.

**Die Pointe:** Der kleine, nicht einmal messbare Verlust wird sauber behandelt; der große, gut messbare fällt durch. **Nicht die Formel entscheidet darüber, sondern wo die Zähler sitzen.** Eine Energiebilanz ist nur so sauber wie die Lage ihrer Messstellen — brauchbarer Merksatz auch für den Unterricht.

**Was daran nicht stimmt — die DC/AC-Grenze.** Die Formel mischt Messebenen: `mppt1/2` und beide Batteriezähler sind **DC**, `exp` (Gridmeter) und Fronius sind **AC**. Alles, was auf dem Weg DC→AC im MultiPlus verlorengeht, verschwindet damit weder in den Export noch in die Batterie — und landet rechnerisch im **Eigenverbrauch**, obwohl es nie im Haus ankommt.

Bilanz für den 15.08. (Tageswerte), unabhängig vom `Wandlungsverluste`-Sensor gerechnet:

| | kWh |
|---|---|
| MPPT 1 + 2 (DC) | 21,21 |
| Fronius (AC, = PV gesamt 102,01 − MPPT) | 80,80 |
| Batterie geladen / entladen (DC) | 8,40 / 7,30 |
| **DC durch den Wechselrichter** (21,21 − 8,40 + 7,30) | **20,11** |
| gemessene MultiPlus-AC-Abgabe | 16,22 |
| **Differenz = Wandlungsverlust** | **3,89** (η = 80,7 %) |

Der separate Sensor `…wandlungsverluste_dc_ac` nennt für denselben Tag **3,60 kWh** — die Bilanz schließt also auf 0,29 kWh (1,8 %).

**Konsequenz:** Von den heute gebuchten 22,74 kWh Eigenverbrauch (7,95 €) sind rund **3,6 kWh nie im Haus angekommen — etwa 15,8 %, in Geld ~1,26 €.** AC-seitig korrigiert wären es 19,1 kWh bzw. 6,69 €. Der Effekt skaliert mit dem DC-Anteil: Hier läuft der Löwenanteil der PV (80,8 von 102 kWh) AC-gekoppelt über den Fronius und ist nicht betroffen; träfe es die ganze Erzeugung, wäre der Fehler ein Vielfaches.

**Selbstkorrektur:** Die „zweite, unabhängige Gegenprobe" von 21:25 hat das **nicht** aufgedeckt und konnte es nicht — sie hat mit denselben Zählern nachgerechnet und damit nur bewiesen, dass der Flow *seine Formel* korrekt ausführt, nicht dass die Formel den vermiedenen Einkauf richtig definiert. Exakt der Merksatz, den ich dort selbst notiert habe („zwei Rechenwege beweisen nur, worin sie sich unterscheiden"), trifft die Prüfung also selbst. Eine echte Validierung müsste **AC-seitig** gegenrechnen: `Fronius_AC + MultiPlus_AC_Abgabe − Export − MultiPlus_AC_Aufnahme`.

**Nicht geändert** (Konsistenz mit der Abendentscheidung). Das ist kein Bug, sondern eine Definitionsfrage an der Messgrenze — aber eine mit ~1,2 €/Tag Wirkung, also deutlich gewichtiger als der Drei-Cent-Abtastversatz, für den der Stundenrand-Tick samt Editor-Falle gebaut wurde. **Die unangenehme Erkenntnis des Abends: an der dritten Nachkommastelle poliert, während an der Messgrenze ~16 % danebenliegen.**

#### Umgesetzt: Wandlungsverluste werden abgezogen, Zähler zurückgesetzt (15.08.2026, 23:13) ✅

Norberts Entscheidung, nachdem sich zeigte, dass der Gegenwert bereits im System liegt. **Ein Subtraktionsterm statt neuer Formel:**

```js
const dSelf = dPv + d.dis - d.exp - d.chg - d.loss;
```

Quelle ist `victron/losses/conversion_energy` aus dem Wandlungsverluste-Flow — retained, derselbe Broker, von keinem anderen Flow abonniert (die MQTT-Bündelungsfalle greift hier also nicht). Der Losses-Flow führt exakt `(mppt1+mppt2+dis−chg) − (acOut−acIn)`, also denselben DC-Term wie der Savings-Flow gegen die AC-Seite, inklusive abgezogener Netzladung.

**Bemerkenswert:** Der Kopfkommentar von `gen_losses_flow.py` beschreibt genau diese Falle — *„Verluste dazwischen landen dadurch im ‚Hausverbrauch'"* — sie war für HAs Hausverbrauch also längst erkannt und wurde beim Bau des Savings-Flows unbemerkt wieder eingebaut.

Geändert in `gen_savings_flow.py`: `KEYS` um `loss`, neuer `mqtt in` (`clsav0in_loss1`), Messwert-Zuordnung, der Subtraktionsterm, TAB_INFO und Comment-Node. **Tests jetzt 12 Szenarien, alle grün** — neu ist Nr. 12: derselbe Tag einmal mit laufendem, einmal mit eingefrorenem Verlustzähler. Ergebnis **7,06 € gegen 8,65 €** — die 8,65 € sind exakt der alte dokumentierte Wert für den 14.08., die Differenz von 1,59 € (4,58 kWh) also der bisherige systematische Fehler.

**Deploy 23:12** (20 Nodes, Cron-Ausdruck `30 59 * * * *` unbeschädigt), **Reset 23:12:59** per `POST /inject/clsav0reset0001` — beide HA-Sensoren stehen auf 0. Der Zeitpunkt war bewusst gewählt: Die Historie war erst anderthalb Tage alt, ein Definitionswechsel damit so billig wie nie wieder.

**Restrisiko, in Test 12 mitgeprüft:** Fällt der Verlustzähler aus oder verwirft er ein Fenster, ist die Korrektur zu klein — der Fehler geht dann in die alte, bekannte Richtung, nur kleiner. **Eine Warnung gibt es dabei nicht**, der Ausfall fällt nur numerisch auf.

**Erledigt (23:18): `recorder/clear_statistics` ausgeführt** — Trockenlauf zeigte 8,078 € / 13,725 € im Tagesbucket, danach meldet die Statistik-Abfrage für beide `statistic_ids` *„No statistics found"*. Der Einbruch ist damit weg, HA baut ab der nächsten vollen Stunde neu auf.

**Nachkontrolle (wichtig, weil hier ein Blindfleck lauert):** Dass beide Sensoren um 23:18 noch auf `0` mit `last_changed 23:12:59` stehen, ist **kein** Hinweis auf einen hängenden Flow — nach dem Reset ist `st.prev = {}`, der erste Tick setzt nur `prev = now` ohne Delta, und ein unveränderter Wert hinterlässt in dieser HA-Instanz keine Spur (`last_reported` == `last_changed`, siehe Fehlspur um 21:22). Der Beweis kann erst der erste Wert > 0 sein; nachts dauert das wegen der 0,1-kWh-Quantisierung der Batterie und des neuen Verlustabzugs entsprechend länger.

**Ursprünglich offen war:** `recorder/clear_statistics` für beide Sensoren. Der Sprung 13,73 € → 0 wird bei `state_class: total` als negativer Zuwachs gebucht; ohne Bereinigung zeigen Tag/Woche/Monat Unsinn. Werkzeug liegt bereit: `tools/clear_savings_statistics.py` (Trockenlauf ohne Argument, Ausführung mit `--yes`, WS-API mit Token aus `~/.claude.json`). Bewusst **nicht** `adjust_sum_statistics`: Das korrigiert eine Stunde und setzt voraus, dass der Rest der Reihe stimmt — nach einem Definitionswechsel stimmt er gerade nicht.

#### Ist ein Umbau nötig? Vorgehen in drei Stufen (Stand 15.08.2026, 23:45 — durch die Umsetzung überholt)

**Antwort: nicht jetzt — aber vor der nächsten Investitionsrechnung.** Zwei Gründe gegen sofortiges Handeln:

1. **Die Korrekturgröße ist noch nicht belastbar.** η = 80,7 % ist für einen MultiPlus auffällig niedrig (Nenn 93–95 %). Plausibel wären Teillastbetrieb bei 300–700 W und Standby-Eigenverbrauch — möglich ist aber auch, dass die Annahme „alle Batterieladung stammt aus den MPPTs" für diesen Tag nicht trägt. **Ein Tag ist keine Basis für einen Umbau.** (Richtungssicher ist der Befund trotzdem: Käme ein Teil der Ladung aus dem Netz, wäre der DC-Durchsatz höher und η noch schlechter.)
2. **Ein Definitionswechsel bricht den Lebenszeitzähler.** Die aufgelaufenen 13,59 € sind nach der alten Formel gerechnet; ein Schnitt mitten im Zähler erzeugt eine Zahl, die vorher und nachher Verschiedenes bedeutet — dasselbe Muster, an dem in der Nacht zum 15.08. der Historien-Import gescheitert ist. Das ist **die** Entscheidung, nicht ein Detail des Umbaus.

**Stufe 1 — beobachten, ohne Eingriff.** Die Gegenrechnung läuft vollständig auf vorhandenen HA-Statistiken (MPPT 1+2, PV gesamt, Batterie geladen/entladen, MultiPlus-AC-Abgabe, Wandlungsverluste) und ist für beliebige Tage wiederholbar. Ist der Faktor über eine Woche stabil bei ~15 %, ist er real; schwankt er stark, war der 15.08. ein Sonderfall. Kosten: null.

**Stufe 2 — dann bewusst wählen:** (a) umbauen und Zähler neu starten (Historie erst seit 02:00 des 15.08., also billig), (b) einen **zweiten**, AC-seitigen Zähler parallel führen und beide vergleichen, (c) wissentlich so lassen und die Kennzahl korrekt etikettieren.

**Stufe 3 — Umbau**, falls Stufe 2 dafür spricht: AC-seitige Formel `Fronius_AC + MultiPlus_AC_Abgabe − Export − MultiPlus_AC_Aufnahme`; die Quellen liegen als HA-Sensoren bereits vor.

**Einordnung für die Praxis:** Die Kernaussage der Kennzahl — eine selbst genutzte kWh ist ~4,7× so wertvoll wie eine eingespeiste — bleibt unberührt, Faktor und Rangfolge stehen. Relevant wird der Fehler nur bei absoluten Beträgen (Amortisation, Wärmepumpe, Speichererweiterung): Dort liegt „Ertrag gesamt" derzeit rund **10 % zu hoch** (13,59 € statt ~12,33 €), und zwar ausschließlich im Eigenverbrauchs-Term — die Einspeisevergütung ist AC-gemessen und sauber.

### Was an diesem Abend NICHT gebaut wurde — und warum (15.08.2026, 23:15)

Auf dem Tisch lagen `git init`, ein `tools/backup_flows.py` mit Diff-Wächter, ein systemd-Timer und die Flow-Sperre. **Entscheidung (Norbert): alles so lassen.** Die Begründung ist es wert, festgehalten zu werden, weil sie beim nächsten Mal wieder trägt.

**Auslöser war Norberts KISS-Einwand:** *„Das scheint mir ein ziemlich komplizierter Flow zu sein, der sehr fehleranfällig ist."* Die Prüfung daraufhin, Konstrukt für Konstrukt, mit der Leitfrage **„welche Annahme über die Wirklichkeit macht diese Zeile — und stimmt sie?"**:

- **Essentiell** (je eine Narbe von einem real passierten Fehler): `sane`-Check gegen Zählerrücksprünge, Wächter 1 (toter Export-Zähler liefert Delta 0), Wächter 2 (`MAXHOUSE`, ohne ihn im Test 20,79 € statt 8,65 €), rohes Akkumulieren gegen Rectification Bias, Auffüllen aus `prev` gegen die MQTT-Subscription-Bündelung, Monotonie vor Publish für HAs `sum`. Sechs von sieben Konstrukten — die Komplexität sitzt in der **Anlage**, nicht im Code (Brooks: essentiell vs. akzidentell; Tesler: was man dem System nimmt, landet im Menschen).
- **Akzidentell — hier hatte Norbert recht:** der Zusatz-Tick `30 59 * * * *`. Er löst ein rein kosmetisches Problem (drei Cent Anzeigeversatz innerhalb der laufenden Stunde, selbstkorrigierend) und bezahlt dafür mit einer Node, die **beim Ansehen kaputtgeht**. Eine Konstruktion, deren einzige Sicherung ein Warnhinweis im eigenen Namen ist, ist ein Code Smell.

**Der eigentliche Lehrsatz aus dem Abend** (und die Selbstkorrektur): Der Vorschlag, die fragile Cron-Node durch einen Diff-Wächter im Backup-Skript zu *überwachen*, war das Antipattern — **Komplexität mit Komplexität absichern**, statt die Fragilität zu beseitigen. Ebenso verworfen: den Export in `deploy_savings_flow.py` einzuhängen. Ein Deploy-Hook sichert nur die Änderungen, die ohnehin aus dem Generator reproduzierbar sind; gefährlich sind die **am Deploy vorbei** (Handarbeit im Editor). Wer sich nur sichert, wenn er sich ordentlich verhält, sichert die falschen Fälle → Zeitsteuerung, nicht Ereignissteuerung.

**Warum trotzdem nichts gebaut wurde:** Der Schaden der Falle ist eine Bagatelle (drei Cent Anzeige, kein Fehlwert, kein Datenverlust), jeder Eingriff kostet dagegen sofort Deploy, Restore-Fenster und eine neue Verifikationsrunde an einem System, das gerade auf 0,001 € genau rechnet. Getragen wird der Zustand dreifach: vzdump 21:35, Generatoren in `tools/`, und diese Dokumentation.

**Mitnehmen, sobald der Flow aus anderem Grund ohnehin angefasst wird** (Grenzkosten dann ≈ 0):
1. `"locked": True` in die Tab-Definition (`gen_savings_flow.py` Z. 328) **und** ins Deploy-Payload.
2. Sechs-Feld-Cron ersetzen oder streichen. Kandidat: Fünf-Feld-Ausdruck mit Minutenliste (`0,5,…,55,59 * * * *`, tickt `hh:59:00` statt `hh:59:30`, kostet ~0,007 €) — **Editor-Darstellbarkeit vorher verifizieren, nicht annehmen.**
3. Löschkandidat „UDP Akku" (streitet WatchMon die Batrium-Pakete auf Port 18542 ab).

**Woran man merkt, dass es doch Zeit wird:** „Ertrag gesamt" bleibt dauerhaft ein paar Cent hinter der Summe seiner Einzelposten und fällt nachts nicht auf ~0,004 € zurück → der Stundenrand-Tick ist weg, jemand hat die Node geöffnet.

**Als Unterrichtsmaterial** taugt der Flow gerade wegen der Mischung: sechs Konstrukte, die man nach KISS fälschlich wegkürzen würde (und sich 12 € Fehlbuchung einhandelt), und eines, das man zu Recht rauswirft, obwohl es nachweislich funktioniert. Prüffragen: (1) Welche Annahme über die Wirklichkeit macht diese Zeile? (2) Wird das System einfacher — oder nur der Mensch belastet? (3) Womit sichere ich die Konstruktion ab? Lautet die Antwort „mit einem Kommentar" oder „mit Aufpassen", ist die Konstruktion falsch, nicht die Absicherung zu schwach. Ergänzend die STOP-Hierarchie aus dem Arbeitsschutz: Substitution → Technisch → Organisatorisch → Persönlich; ein Warnschild ist die schwächste Stufe.

---

## Statistik nachgeprüft — bestanden (16.08.2026, 21:30) ✅
> Erledigt. Die ursprüngliche Aufgabenstellung steht unverändert darunter, das Ergebnis am Ende des Abschnitts.

### ▶ Aufgabenstellung (offen gewesen seit 15.08.2026, 23:20)

**Zuständig: Norbert, morgen früh (16.08.2026).** Erste Sichtprüfung genügt: Zeigen die Karten „Ersparnis durch Eigenverbrauch" und „Ertrag gesamt" auf „Energie (Sys)" wieder Werte über 0, ist der Umbau durch. Die Schritte 2–6 nur, wenn dort etwas nicht stimmt.

**Faustregel für den Morgen:** Bis Sonnenaufgang gibt es keine Einspeisung, also müssen **beide Karten denselben Betrag** zeigen (`Ertrag gesamt − Ersparnis ≈ 0`). Und der Nachtzuwachs fällt jetzt **kleiner** aus als in den Nächten davor — das ist genau der neue Verlustabzug und der eigentliche Beleg, dass er greift.

**Warum:** Nach dem Definitionswechsel (Wandlungsverluste werden abgezogen), dem Reset um 23:12:59 und `clear_statistics` um 23:18 steht der Zähler bei 0 und HA hat keine Statistik mehr. **Ob der Flow tatsächlich wieder rechnet, ist bislang unbelegt** — die letzte Kontrolle um 23:21 sah weiterhin `0` bei `last_changed 23:12:59`, was aber nichts beweist: Nach dem Reset ist `st.prev = {}`, der erste Tick setzt nur `prev = now`, und ein unveränderter Wert hinterlässt in dieser HA-Instanz keine Spur (`last_reported` == `last_changed`). Der erste Wert **> 0** ist der einzige belastbare Beleg.

### Prüfschritte

1. **Läuft er überhaupt?** `sensor.victron_ersparnis_eigenverbrauch` und `sensor.victron_ertrag_gesamt` abfragen. Wert > 0 → Entwarnung, Schritt 2. Wert klebt bei 0 → Schritt 5.
2. **Nacht-Plausibilität** (die schärfste Probe, weil PV = 0 und Export ≈ 0): Es muss ungefähr gelten
   `Ersparnis-Zuwachs ≈ (Batterie_entladen − Wandlungsverluste) × 0,3494 EUR/kWh`
   Quellen: `sensor.victron_batterie_entladen`, `sensor.victron_speicher_wandlungsverluste_dc_ac`, Stundenstatistik `change` von 00:00 bis Sonnenaufgang.
3. **Alte Gegenprobe wiederholen** (wie 15.08. 21:15, jetzt mit sauberer Reihe): Ertrag gesamt gegen Ersparnis + `sensor.victron_vm_3p75ct_victron_netzeinspeisung_compensation`, über abgeschlossene Stundenbuckets, bei Einspeisung ≈ 0. Erwartung: Differenz nahe 0.
4. **Größenordnung gegen früher:** Der Tageswert muss jetzt spürbar unter dem alten Niveau liegen — 15.08. wären es 12,40 € statt 13,73 € Ertrag gewesen (−10 %), bei der Ersparnis −16 %. Liegt er auf altem Niveau, greift der Verlustabzug nicht.
5. **Falls der Zähler bei 0 klebt:** `victron/savings/_diag` lesen (nennt fehlende Keys) — der Verdacht wäre, dass `victron/losses/conversion_energy` nicht ankommt. Gegen die Admin-API prüfen, ob der Eingang `clsav0in_loss1` existiert: `curl -s http://192.168.2.80:1880/flow/577ab5addc422ce0`. Notnagel: Quellwert einmal identisch retained neu publizieren (Rezept im Kopf von `tools/gen_savings_flow.py`).
6. **Statistik-Neuaufbau kontrollieren:** Ab der ersten vollen Stunde nach 23:18 müssen wieder Buckets entstehen. Fehlen sie dauerhaft, war nicht der Flow das Problem, sondern der Recorder.

### Erwartetes Ergebnis
Ein sauber ab Mitternacht wachsender Zähler, dessen Nachtzuwachs der Batterieentladung **abzüglich** Wandlungsverluste entspricht — genau die Korrektur, die heute eingebaut wurde.

### Ergebnis der Prüfung (16.08.2026, 21:30) ✅

**Alle sechs Prüfschritte bestanden. Der Verlustabzug greift nachweisbar, die Statistik ist lückenlos neu aufgebaut, der Stundenrand-Tick lebt.** Stand 21:22: Ersparnis **16,083 €**, Ertrag gesamt **19,241 €** (Lebenszeit seit Reset 15.08. 23:12:59).

**1. Läuft er?** Ja. Beide Sensoren > 0, `last_changed` 21:22:53 — der Blindfleck von 23:21 ist damit aufgelöst, der erste Wert > 0 ist da.

**2. Nacht-Plausibilität (die schärfste Probe), Stundenbuckets 00:00–07:00:**

| | kWh bzw. € |
|---|---|
| Batterie entladen | 4,20 |
| Wandlungsverluste | 0,96 |
| Erwartung `(4,20 − 0,96) × 0,3494` | **1,132 €** |
| Ist (Summe `change` Ersparnis) | **1,107 €** |
| Erwartung **ohne** Verlustabzug | 1,468 € |

Abweichung 0,025 € (2,2 %) — im Rahmen der 0,1-kWh-Quantisierung der Batrium-Zählung. Entscheidend ist die Lage zwischen den beiden Erwartungswerten: Der Ist-Wert liegt eindeutig auf der korrigierten Seite, **32 % unter der alten Formel**. Damit ist der Verlustabzug nicht nur eingebaut, sondern wirksam.

**3. Gegenprobe Ertrag vs. Ersparnis + HA-Vergütung (Tagesbucket 16.08.):** Ertrag 19,025 − Ersparnis 15,870 = **3,155 €** gegen HA-`…_netzeinspeisung_compensation` **3,15507 €**. Differenz **0,00007 €**. Die 33-Sekunden-Nachlaufkorrektur vom 15.08. hält — und damit ist zugleich das Warnsignal aus „Woran man merkt, dass es doch Zeit wird" negativ: Der Sechs-Feld-Cron `30 59 * * * *` ist unbeschädigt, niemand hat die Node geöffnet.

**4. Größenordnung — hier war eine Erwartung falsch, nicht der Zähler.** Erwartet war ein Tageswert *unter* dem alten Niveau; tatsächlich steht der 16.08. mit 19,03 € **deutlich über** dem 15.08. (13,73 €). Die Erklärung liegt nicht im Flow, sondern in der Anlage:

| | 15.08. | 16.08. |
|---|---|---|
| PV gesamt | 102,01 kWh | 95,17 kWh |
| **Einspeisung** | **76,58 kWh** | **42,74 kWh** |
| **openWB geladen** | **0,42 kWh** | **32,19 kWh** |
| Eigenverbrauch (aus Ersparnis) | ~23 kWh | **45,4 kWh** |

Der Einspeisungsrückgang von 33,8 kWh entspricht fast exakt den 31,8 kWh Mehr-Autoladung. **Bei weniger Sonne wurde mehr Geld verdient** — genau das, was die Kennzahl zeigen soll (Faktor 4,7 zwischen behaltener und eingespeister kWh). Der 16.08. ist damit unfreiwillig das beste Lehrbeispiel für den Sinn der ganzen Konstruktion: nicht mehr erzeugen, sondern mehr behalten.

Der Verlustabzug ist in diesen Zahlen trotzdem enthalten: Ohne ihn hätte der Tag 17,10 € statt 15,87 € Ersparnis gebucht (**−1,24 €, −7,2 %**). Prüfschritt 4 in seiner ursprünglichen Formulierung war unbrauchbar, weil er Tage mit verschiedenem Lastprofil vergleicht — **die Bilanzprobe (Schritt 2) und die Formelnachrechnung sind die belastbaren Wege.** Gegenrechnung der Formel für den ganzen Tag: `95,17 + 5,80 − 42,74 − 9,30 − 3,561 = 45,37 kWh` gegen 45,42 kWh aus dem Zählerstand — 0,1 % Abweichung.

**5. Entfiel** (Zähler klebt nicht bei 0), `_diag` musste nicht gelesen werden.

**6. Statistik-Neuaufbau:** Erster Bucket ist die 23:00-Stunde des 15.08. mit `change` 0,133 € — also unmittelbar nach `clear_statistics` um 23:18, ohne Lücke und ohne negativen Sprung. Der 15.08. steht im Tagesbucket folgerichtig nur noch mit 0,133 € / 0,134 € (Rest ab 23:18), die alte Reihe ist weg.

#### Merksatz aus dieser Prüfung: ein Prüfkriterium ist so gut wie das, was es konstant hält

Schritt 4 („der Tageswert muss unter dem alten Niveau liegen") war beim Aufschreiben plausibel und ist beim Ausführen wertlos gewesen — er vergleicht zwei Tage, zwischen denen sich **nicht nur die geänderte Formel, sondern auch das Lastprofil** geändert hat (32 kWh Autoladung). Ein Kriterium, dessen Kontrollgröße mitwandert, kann den Effekt weder bestätigen noch widerlegen; hier hätte es den korrekt arbeitenden Flow sogar **fälschlich als defekt gemeldet** („liegt auf altem Niveau → Verlustabzug greift nicht" — tatsächlich lag er darüber).

Getragen hat die Prüfung stattdessen das, was **innerhalb desselben Zeitraums** rechnet: die Nachtbilanz (PV = 0, Export ≈ 0 → nur ein Term übrig) und die Nachrechnung der Formel gegen den Zählerstand. **Regel: gegen eine Bilanz im selben Fenster prüfen, nicht gegen einen Erfahrungswert von gestern.**

Damit reiht sich der Tag in die beiden Vorgänger ein — dreimal dieselbe Familie von Fehlern, jedes Mal eine Stufe früher erwischt:
1. *15.08. 21:25* — zwei Rechenwege beweisen nur, worin sie sich **unterscheiden** (gemeinsamer `dSelf`-Term blieb unsichtbar).
2. *15.08. 23:30* — eine Bilanz ist nur so sauber wie die **Lage ihrer Messstellen** (DC/AC-Grenze).
3. *16.08.* — ein Prüfkriterium ist nur so gut wie das, was es **konstant hält** (Lastprofil).

Alle drei taugen als Unterrichtsmaterial zum selben Kern: **Validierung scheitert nicht am Rechnen, sondern an unausgesprochenen Annahmen über den Vergleichsmaßstab.**

**Eine Beobachtung fürs Auge behalten:** Der Wandlungsverluste-Zähler meldet für die 06:00-Stunde `change = 0` bei gleichzeitig 0,4 kWh Batterieentladung. Genau das ist das am 15.08. notierte **Restrisiko ohne Warnung** (verworfenes Fenster im Losses-Flow → Korrektur zu klein, Fehler in die alte Richtung). Einmalig und im Centbereich, aber die einzige Stelle, an der die Prüfung nicht sauber aufging. Häuft sich das an den Dämmerungsstunden, lohnt ein Blick in den Losses-Flow — der Verdacht wäre eine Bilanz, die beim Umschalten kurz negativ wird und auf 0 geklemmt wird.

## „Cerbo-Neustart" am 18.08.2026, 03:52 — war keiner: 80 s Netzwerkausfall

**Anlass:** Norbert meldete einen Neustart des Cerbo um 3:52 Uhr. **Es gab keinen.**

**Gegenbeweis, dreifach:**
- `uptime` am 18.08. um 09:33 lokal: **4 Tage, 10:18 h** → letzter Boot 13.08. 23:15 lokal, also der dokumentierte Endabnahme-Reboot.
- `sensor.cerbo_gx_letzter_start` steht unverändert auf `2026-08-13T21:15:00+00:00`.
- Kein Boot-Eintrag in `/data/log/messages`, `cerbo-online-watch-boot.log` unverändert vom 13.08.

**Was wirklich passierte — der Ethernet-Link fiel aus** (`/data/log/messages`, UTC):

```
Aug 18 01:52:24  sun4i-emac eth0: Link is Down
Aug 18 01:53:44  sun4i-emac eth0: Link is Up - 100Mbps/Full
```

**80 Sekunden**, physischer Carrier-Verlust (`operstate DOWN`, Adresse und Routen abgeräumt), kein Uptime-Rücksprung (Kernel-Ticks laufen von 362238 auf 362318 durch). Lokalzeit **03:52:24 – 03:53:44** — genau Norberts Uhrzeit.

**Der Wächter hat korrekt gearbeitet.** Er meldete per Last Will einen **Ausfall**, keinen Neustart: `binary_sensor.cerbo_gx_online` off um 03:52:57, on um 03:53:53; im Dienst-Log der bekannte Selbstheilungspfad (`offline` → „Broker fuehrt uns als offline, obwohl der Dienst laeuft - korrigiere" → `online`). Die Push-Meldung sagte also die Wahrheit — die Fehldeutung entstand beim Lesen. **Lehre für die Meldungstexte:** „Ausfall" und „Neustart" müssen im Push sprachlich weiter auseinanderliegen; der Startzeit-Sensor ist das entscheidende Unterscheidungsmerkmal und sollte in der Ausfall-Meldung mitgeschickt werden.

### Der Ausfall betraf mehr als den Cerbo — und nicht das ganze Netz

Aus dem HA-Logbuch, sekundengenau:

| Zeit (lokal) | Gerät | Anbindung |
|---|---|---|
| 03:52:24 | **Cerbo GX** Link down | Ethernet |
| 03:52:51 | **Sonos Wohnzimmer** (`Subscription renewal failed: Request timed out`) | LAN/WLAN |
| 03:52:53 | **Reolink Doorbell PoE + Kamera TW + Chime** | PoE |
| 03:52:57 | Cerbo-Last-Will greift (MQTT-Keepalive) | — |

Zurück: Reolink 03:53:52, Cerbo 03:53:53, Sonos 03:54:39, Kamera TW 03:54:53.

**Nicht betroffen:** alles, was innerhalb von Proxmox bleibt. Die Zigbee-Präsenzmelder (AZ/WZ) melden über das gesamte Fenster **lückenlos** weiter (03:52:56, 03:53:31, 03:53:38 …) — Z2M-LXC, MQTT-LXC und HA-VM reden über die interne Bridge `vmbr0` und sehen den physischen Switch gar nicht. Der Proxmox-Host selbst verzeichnete **keinen** Link-Down (letzter am 29.07.), Uptime 91 Tage.

**Diagnostische Trennschärfe:** Genau diese Aufteilung — externe IP-Geräte weg, interne Gäste unbeeinträchtigt — grenzt die Ursache auf ein **Netzsegment außerhalb des Proxmox-Uplinks** ein. Es ist die Ergänzung zum bekannten Muster „sekundengleiche `unavailable` über mehrere Geräte = zentraler Ausfall, nie Geräte-Crash": Die Liste der **nicht** betroffenen Geräte lokalisiert den Ausfall.

### Verdacht: nächtliches UniFi-Wartungsfenster (am 18.08. bestätigt, Beweis in [[homelab-infrastruktur#Netzwerk-Vorfall 18.08.2026 und Fernzugriff]])

Die drei bekannten Link-Aussetzer ohne Reboot liegen alle im selben Nachtfenster:

| Datum | Gerät | Zeit (lokal) | Dauer |
|---|---|---|---|
| 08.04.2026 | Cerbo `eth0` | 03:27:42 | 94 s |
| 29.07.2026 | Proxmox `enp2s0f1` | 03:44:49 / 03:45:11 / 03:45:45 | 3× 3–7 s |
| 18.08.2026 | Cerbo `eth0` | 03:52:24 | 80 s |

Drei Ereignisse zwischen 3:27 und 3:53 Uhr sind kein Beweis, aber ein auffälliges Muster — typisch für automatische Firmware-Updates bzw. Provisioning-Läufe der UniFi-Geräte (Netz läuft auf einer **UDM Pro**, s. [[homelab-infrastruktur]]).

**Nicht weiter verifizierbar mit den vorhandenen Zugängen:** SSH auf die UDM (`192.168.2.1`) wird abgewiesen (in UniFi standardmäßig aus), der `UnifiController`-LXC 110 (`192.168.2.205`) ist **gestoppt** — die Switches verwaltet also der integrierte Controller der UDM.

**Nächster Schritt für Norbert (UniFi-UI):**
1. *Settings → System → Updates*: „Auto-Update" und dessen Zeitfenster prüfen — steht es auf ~3–4 Uhr, ist die Frage beantwortet.
2. *Insights → Events* bzw. das Systemprotokoll auf den 18.08. ~03:52 filtern: Switch-Restart, Port-Flap oder Provisioning?
3. Am selben Switch identifizieren, an welchem Port Cerbo, Reolink-PoE und Sonos hängen — das benennt die Komponente eindeutig.
4. Optional dauerhaft: SSH auf der UDM aktivieren (*Settings → Control Plane → Console → SSH*), dann sind solche Fragen ohne UI beantwortbar.

### Kein Datenschaden

Der Cerbo lief durch, nur seine Netzanbindung fehlte — die VE.Bus-`/Energy/`-Werte blieben also unangetastet (anders als bei einem harten Reset). `sensor.victron_multiplus_ac_abgabe`: 903,70 kWh um 03:48:41 → 903,71 kWh um 03:57:33. **Meldelücke ~9 min, kein Rücksprung, kein Reset, kein Eingriff des Monotonie-Wächters nötig.** Die ESS-Regelung setzte für 80 s aus (Node-RED im LXC ohne D-Bus-Verbindung) — bei nächtlicher Grundlast folgenlos.

### Umgesetzt: Meldungstexte trennen Ausfall und Neustart (18.08.2026)

Die Fehldeutung entstand nicht am Gerät, sondern am Text. Die alte Ausfall-Meldung endete mit „Kommt er zurueck, folgt eine Meldung mit der Startart" — das klingt nach einem angekündigten Neustart, obwohl bei einem reinen Netzausfall nie eine Boot-Meldung folgt.

**Drei Änderungen in `tools/gen_cerbo_flow.py`, deployt per `tools/deploy_cerbo_flow.py`:**

1. **Zwei getrennte Wortfamilien im Titel.** Verbindungsereignisse heißen jetzt `Cerbo GX: VERBINDUNG WEG` / `Cerbo GX: Verbindung wieder da`, alle fünf Startarten `Cerbo GX: NEUSTART (…)`. Der Titel ist alles, was auf dem Sperrbildschirm ankommt — dort muss die Unterscheidung schon fallen.
2. **Der letzte bekannte Startzeitpunkt steht in beiden Verbindungsmeldungen** (aus `flow.get('cerbo_boot_zeit')`, samt „läuft seit"). Bleibt er über einen Ausfall hinweg stehen, war es kein Neustart — das ist die Angabe, die am 18.08. gefehlt hat.
3. **Beide Texte benennen die Abgrenzung ausdrücklich:** die Ausfall-Meldung mit „Das ist ein Ausfall, KEIN Neustart" plus dem Hinweis, dass bei einem echten Reboot gleich eine eigene NEUSTART-Meldung folgt; die Entwarnung mit „Folgt jetzt KEINE NEUSTART-Meldung, ist der Cerbo durchgelaufen — dann liegt die Ursache im Netz, nicht im Gerät".

**Warum die Entwarnung den Neustart nicht selbst ausschließt:** Sie feuert, sobald `status` wieder `online` ist — die Boot-Meldung braucht ein paar Sekunden länger. Der Flow *kann* an dieser Stelle also nicht wissen, ob gleich noch ein Reboot gemeldet wird. Statt einer künstlichen Verzögerung sagt der Text, woran es zu erkennen ist. Ehrliche Aussage schlägt scheinbare Sicherheit.

**Nebenkorrektur `dauer()`:** Im Minutenbereich laufen die Sekunden jetzt mit (`1 min 20 s` statt `1 min`). Bei Ausfällen ist gerade die genaue Dauer das Indiz — der Cerbo braucht rund 50 s zum Booten, ein kürzerer Ausfall kann gar kein Neustart gewesen sein.

**Tests: 15 Fälle, alle grün** (`node tools/test_cerbo_waechter.js`). Neu darunter der reale Fall „Ausfall ohne Neustart" und eine **Regressionssicherung auf Titel-Trennschärfe**: kein Neustart-Titel darf das Wort „Verbindung" tragen, kein Verbindungs-Titel das Wort „NEUSTART". Damit kann die Verwechslung nicht stillschweigend zurückkehren.

Deploy verifiziert (HTTP 200, `PUT /flow/9adad6a1f416f4d4`): neue Texte live, Pushover-Credentials unverändert erhalten (das Deploy-Skript zieht sie aus dem laufenden Flow, die Repo-Kopie behält ihre Platzhalter).

**Endabnahme mit echtem Ausfall (18.08.2026, 10:01 lokal):** Dienst per `svc -d` gestoppt, nach 28 s per `svc -u` wieder hoch.

```
10:01:12  trap meldet offline  → binary_sensor.cerbo_gx_online = off,
                                 sensor.cerbo_gx_letzter_start = unavailable
                                 → Push Prio 1 "Cerbo GX: VERBINDUNG WEG"
10:01:40  Dienst oben, online  → Sensor wieder on, letzter Start UNVERAENDERT
                                 2026-08-13T21:15:00Z
                                 → Push Prio 0 "Cerbo GX: Verbindung wieder da",
                                   Ausfalldauer 28 s
                               → KEINE Neustart-Meldung (korrekt)
```

Genau das Verhalten, das am 18.08. um 3:52 zur Fehldeutung führte — nur ist es jetzt aus der Meldung selbst ablesbar: Der Startzeitpunkt steht in beiden Pushes und bleibt über den Ausfall hinweg stehen. `cerbo_weg_seit` wurde sauber auf `null` zurückgesetzt, der Flow-Kontext ist wieder im Ausgangszustand.

⚠️ **Für künftige Tests:** `svc -d` ist ein *sauberes* Beenden — es feuert kein Last Will, sondern der `trap` meldet selbst. Der Weg testet damit die Trap-Strecke, nicht die Broker-Strecke. Einen echten Keepalive-Ausfall (Last Will durch den Broker) erzwingt nur `kill -STOP` auf den Dienstprozess; das dauert bis zum Keepalive-Timeout (~45 s) und ist der härtere Test.

## Ursache bewiesen: Firmware-Auto-Update des USW-24-PoE (18.08.2026)

→ Verschoben nach [[homelab-infrastruktur#Netzwerk-Vorfall 18.08.2026 und Fernzugriff]] (23.08.2026).

## E-Auto-Bilanz 6,96 MWh ohne Ladevorgang (18.08.2026) ✅ behoben

**Folgeschaden des 80-s-Netzausfalls von 03:52** (Ausfallanalyse im Abschnitt oben, Netzwerk-Ursache in [[homelab-infrastruktur#Netzwerk-Vorfall 18.08.2026 und Fernzugriff]]) — diesmal traf es das Energie-Dashboard.

**Symptom:** Energie-Dashboard zeigte für heute 6,96 MWh unter „E-Auto", obwohl das Auto nicht geladen wurde.

**Befund:** `sensor.openwb_geladen` hatte einen 8-Sekunden-Aussetzer:

| Zeit | Wert |
|---|---|
| 17.08. 20:00 | 6960,67 |
| 18.08. 03:53:47 | **0** |
| 18.08. 03:53:55 | 6960,67 |

Die Statistik-Summe sprang im 5-Minuten-Bucket `03:50` von 162,75 auf 7123,42 — Differenz exakt 6960,67, also der komplette Zählerstand als Tagesverbrauch.

**Mechanik:** `state_class: total_increasing` heißt für den Recorder „ich zähle nur aufwärts, ein kleinerer Wert ist ein Reset". HA sah 6960,67 → 0 → 6960,67 und buchte folgerichtig 6960,67 kWh Zuwachs seit dem vermeintlichen Reset. Die Logik ist korrekt, sie war nur mit einer Lüge gefüttert. **Der Zählerstand selbst war nie beschädigt** — kaputt war ausschließlich die abgeleitete Summe.

Dieselbe Fehlerklasse wie beim Fronius-Zähler und beim VE.Bus-Reset am 12.08.: Quelle fällt kurz weg, die Summe dippt, HA liest den Wiederanstieg als Zuwachs.

**Warum die Null durchkam:** Der Flow filterte mit `if (!Number.isFinite(kwh)) return null;` — das fängt `NaN` und `Infinity`, aber `Number("")` ist `0` und damit endlich.

### Korrektur der Statistik

Kein Service, sondern WebSocket-Kommando — über den `ws_command`-Escape-Hatch des HA-MCP-Servers direkt aufrufbar:

```
recorder/adjust_sum_statistics
  statistic_id: sensor.openwb_geladen
  start_time:   2026-08-18T03:00:00+02:00     (Stundenraster!)
  adjustment:   -6960.67
  adjustment_unit_of_measurement: kWh
```

Danach `sum` durchgehend 162,75, alle `change` = 0. HA zieht die Folgestunden automatisch mit.

### Flow gehärtet (`flows/openwb-ev-energie-mqtt.json`)

Die Function `Wh → kWh` wurde zu `Wh → kWh + Wächter` mit drei Regeln:

1. Nicht-numerische Payloads verwerfen.
2. **Werte ≤ 0 immer verwerfen** — ein Wallbox-Energiezähler fällt nicht auf 0. Fängt auch den leeren String.
3. Rückwärtssprünge verwerfen, aber als möglichen echten Zähler-Reset vormerken. Erst wenn der niedrigere Stand **15 Minuten und 3 Meldungen** lang plausibel bleibt, wird er per Offset übernommen: `offset = lastPublished − raw`. Der publizierte Wert bleibt dabei stetig, HA sieht also gar keinen Reset.

Regel 3 ist die Lehre aus dem 12.08.: Ein Monotonie-Wächter **ohne** Reset-Erkennung friert nach einem echten Zählerwechsel dauerhaft ein.

Zweiter Function-Ausgang → `victron/ev/energy/_diag` (retained, JSON mit `action`/`reason`/`raw`/`offset`), damit Verwerfen sichtbar ist statt still zu passieren.

**Lokal durchgetestet** vor dem Deploy: echter Störfall, leerer Payload, echter Zählerwechsel (wird nach der Frist sauber übernommen und zählt monoton weiter), kurzer Rücksprung mit Rückkehr. Deploy per `PUT /flow/clev0tab00000001`; live verifiziert über den Node-RED-Statuskanal — Function meldete `Start 6960.67 kWh`, alle vier MQTT-Nodes verbunden.

> [!TIP] Node-RED live beobachten ohne Editor
> `ws://192.168.2.80:1880/comms`, nach dem Handshake `{"subscribe":"status/#"}` und `{"subscribe":"debug"}` senden. Liefert Node-Status und Debug-Ausgaben im Klartext — der schnellste Weg zu prüfen, ob ein frisch deployter Flow wirklich Daten verarbeitet. Ein roher WebSocket-Client in reinem Python genügt (kein `websockets`-Paket nötig).

## Zähler-Flows systematisch durchgegangen (18.08.2026) ✅

Anlass: Nach dem openWB-Phantomzähler die Frage, welche anderen `total_increasing`-Zähler beim nächsten nächtlichen Switch-Update verwundbar sind. Alle 20 Tabs, 9 Zähler-Sensoren geprüft.

### Warum es nur openWB traf — der strukturelle Unterschied

Gegenprobe an den beiden härtesten Ereignissen: **Cerbo-Absturz 12.08.** und **Switch-Ausfall 18.08. 03:52**. Bei Grid und Batterie in beiden Fällen **kein einziger Sprung**, alle Stundenzuwächse plausibel.

Der Grund ist die Bezugsart: Die Victron-Werte kommen über `victron-input-*`-Nodes per D-Bus — fällt die Verbindung weg, sendet der Node **gar nichts**. Ein fremder MQTT-Publisher wie openWB schickt beim Hochfahren dagegen aktiv eine 0.

> [!IMPORTANT] Lücke vs. Lüge
> `total_increasing` reagiert nur auf Werte, die es auch sieht. Eine Meldelücke ist harmlos, ein falscher Wert ist es nicht. Deshalb brauchen D-Bus-Quellen keinen Nullwert-Schutz, fremde MQTT-Publisher aber sehr wohl.

### Stand der Flows

| Flow / Sensor | Schutz | Status |
|---|---|---|
| MultiPlus AC-Abgabe + AC-Aufnahme | Offset-Reset-Erkennung, gestaffelte Fristen, State retained | ✅ Referenzmuster |
| Fronius-Guard | `!isFinite \|\| v <= 0` | ✅ |
| Wandlungsverluste | akkumuliert selbst, `MAXDELTA` | ✅ |
| Eigenverbrauch (EUR) | Delta-Rechnung, `MAXAGE`/`MAXSELF`/`MAXHOUSE` | ✅ |
| openWB EV | Wächter mit Reset-Erkennung | ✅ (heute gebaut) |
| PV Energie gesamt | war Monotonie **ohne** Reset-Erkennung | ✅ **heute umgebaut** |
| Grid Bezug/Einspeisung | keiner — durch D-Bus-Bezug gedeckt | ✅ bewusst so |
| Batterie geladen/entladen | keiner — durch D-Bus-Bezug gedeckt | ✅ bewusst so |

### PV-Summe auf das MultiPlus-Muster umgebaut

Der alte Wächter war `if (sum < last - 0.001) return null;` — die bekannte Falle vom 12.08. Beide Ausgänge waren schlecht:

- **ohne Neustart:** nach echtem Zähler-Reset friert der Sensor dauerhaft ein;
- **mit Neustart:** `lastSum` lag nur im Node-Kontext, war also weg → der niedrige Wert wird übernommen → jetzt bucht HA den Reset als Phantom-Zuwachs.

Neu (identisch zum MultiPlus, `flows/victron-pv-summe-mqtt.json`): Offset-Überbrückung statt Blockade, Bestätigungsfristen 2 min bei Sprung > 1 kWh / 30 min bei kleinem Dip, plus State-Persistenz über das retained Topic `victron/pv/_total_state` mit `arm`-Inject nach 15 s. Dazu die fachliche Untergrenze `v <= 0` aus dem openWB-Fall.

Lokal durchgetestet: Normalbetrieb, 0-Glitch einer Quelle, echter Zählerwechsel (nach Frist per Offset überbrückt, Ausgabe stetig, zählt danach korrekt weiter), Neustart mit Restore (rettet den Anker — ohne ihn wären ~7.900 statt ~40.900 kWh publiziert worden), Erststart ohne State.

Live verifiziert: Node grün (Offset 0), monoton 40895,72 → 40895,73 → 40895,74 kWh.

**Restrisiko (gilt auch für den MultiPlus):** Geht das retained State-Topic verloren (Broker-Neuaufsetzung, manuelles Löschen), startet der Wächter ohne Anker und übernimmt den ersten Wert ungefiltert. Bewusst akzeptiert — die Alternative wäre ein Sensor, der nach jedem Broker-Umzug manuell angestoßen werden muss.

### Nebenbefund, nicht kritisch

`PV-Summe` im Tab *Victron Leistung* nutzt `Number(msg.payload) || 0`, macht aus einem fehlenden Wert also eine 0. Betrifft nur Momentanleistung (`measurement`), keine Statistik — kosmetisch.

## Wandlungsverluste 18.08.2026: Wetter erklärt nur einen kleinen Teil — ~0,5 kWh fehlen

18.08. nur **1,88 kWh** statt der üblichen 4–5 kWh. Kein Messfehler: Node grün, kein `stalled`, Akkumulator läuft (119,37 → 119,39 kWh).

| | 18.08. (bis 17:30) | 17.08. | 13.–15.08. |
|---|---|---|---|
| Wandlungsverluste | **1,88** | 3,97 | 3,84–4,30 |
| PV gesamt | 27,89 | 79,47 | 102–122 |
| davon MPPT (**DC**-gekoppelt) | 5,69 | 16,57 | 21–26 |
| davon Fronius (**AC**-gekoppelt) | 22,20 | 62,90 | 81–98 |
| MultiPlus AC-Abgabe | 3,61 | 12,85 | 17–21 |
| MultiPlus AC-Aufnahme | **3,98** | 0,84 | 0,55–0,62 |
| Batterie geladen / entladen | 8,00 / 4,70 | 8,20 / 8,30 | ~9 / ~8 |
| Netzbezug | 0,13 | 0,09 | 0,09–0,16 |

**Der Hebel ist nicht die PV-Menge, sondern der Anteil, der durch den Wechselrichter muss.** Der Fronius ist AC-gekoppelt und läuft am MultiPlus vorbei ins Hausnetz — verlustfrei, was diese Kennzahl angeht. Nur die MPPT-Energie (DC) und die Batterie gehen durch die Wandlung. Und genau die DC-Seite ist heute auf ein Drittel eingebrochen (5,69 statt 16,57 kWh), die AC-Abgabe entsprechend auf 3,61 statt 12,85 kWh.

**Betriebsart-Wechsel:** Der Multi lief heute netto als *Lader* statt als Wechselrichter — AC-Aufnahme 3,98 kWh gegen sonst 0,6. Bei Netzbezug von nur 0,13 kWh kam diese Energie vom Fronius: Die DC-PV reichte nicht zum Laden, also ging Fronius-AC-Strom per AC→DC in die Batterie. Bilanz passt: 5,69 (DC-PV) + 3,98 (AC→DC) = 9,67 kWh Zufluss für 8,00 kWh Batterieladung, Differenz = Ladeverlust.

> [!NOTE] Der Sensorname ist an solchen Tagen irreführend
> Die Formel ist eine geschlossene Bilanz — `Verlust = (MPPT + entladen + AC-Aufnahme) − (geladen + AC-Abgabe)`. Sie erfasst damit **beide** Richtungen. An normalen Tagen dominiert DC→AC, an ladelastigen Tagen wie heute steckt auch der AC→DC-Ladeverlust drin. „DC→AC" im Namen beschreibt den Normalfall, nicht die Rechnung.

Überschlag mit den Tageswerten: `5,69 + 4,70 − 8,00 − 3,61 + 3,98 = 2,76` gegen publizierte 1,88. Die Überschlagsrechnung liegt systematisch ~0,7–0,9 kWh höher (17.08.: 4,66 gegen 3,97) — Folge der Monotonie-Klammer `if (loss > st.pub)`, die Rücksetzer nicht publiziert. Kein Sondereffekt von heute.

### ⚠️ Korrektur (nach Rückfrage): Der Wetter-Effekt ist zu klein für den Einbruch

Einwand Norbert: Wenn der Multi beim Laden mithalf, müssten die Verluste *höher* sein. Der Einwand trifft — und die Regression aus dem Leerlauf-Abschnitt bestätigt ihn: Bei `Verlust = 0,0332 × AC-Abgabe + 0,1736/h` ist der Wandlungsanteil **marginal**. Der Durchsatz-Rückgang von 12,85 auf 3,61 kWh erklärt gerade einmal `0,0332 × 9,24 = 0,31 kWh`. Der Leerlaufsockel läuft wetterunabhängig weiter. Der Tageswert dürfte also kaum sinken — er ist aber um über 1 kWh gefallen.

**Der Nachtvergleich isoliert die Ursache** (00:00–08:00, kein PV, kein Ladebetrieb — dort gilt exakt `Verlust = entladen − AC-Abgabe`):

| 00:00–08:00 | 17.08. | 18.08. |
|---|---|---|
| Batterie entladen | 4,40 | 4,30 |
| AC-Abgabe | 3,28 | 3,17 |
| Verlust **rechnerisch** | 1,12 | 1,13 |
| Verlust **gebucht** | 1,24 | **0,63** |

Nahezu identischer Durchsatz, halber gebuchter Verlust. Die Lücke sitzt in den Stunden **04:00, 05:00 und 07:00**, die exakt 0,00 kWh zeigen, obwohl die Batterie dort 1,40 kWh entladen und der Multi 1,04 kWh abgegeben hat. Aufsummiert fehlen **~0,50 kWh** — exakt die Differenz.

**Das ist der Netzausfall von 03:52, Folgeschaden Nummer zwei** (nach dem openWB-Phantomzähler). Und es ist **kein Bug, sondern der eingebaute Schutz**: Der Flow verwirft Fenster, wenn die AC-Seite steht, während DC läuft — „lieber eine Lücke als ein Fehlwert", genau um die Explosion vom 12.08. zu verhindern. Wahrscheinlichster Auslöser: Der MultiPlus-Monotonie-Wächter hielt nach dem Ausfall seine Ausgabe zurück (Dip-Bestätigung bis 30 min), womit `acOut` aus Sicht des Verlust-Flows stillstand.

> [!IMPORTANT] Konsequenz für die Interpretation
> Nach jedem Netz-/Cerbo-Ausfall ist der Tagesverlust **zu niedrig**, nicht zu hoch. Die Kennzahl unterschätzt still — die verworfenen Fenster hinterlassen keine Spur im Sensor, nur Null-Stunden bei laufendem Umsatz. Beim Bewerten von Verlusttagen also immer erst gegen `Batterie entladen − AC-Abgabe` der Nachtstunden gegenrechnen.

**Offen:** Die genaue Auslöse-Ursache steht nur in den `node.warn`-Meldungen des Flows (Node-RED-Log), nicht in HA. Zum Festnageln beim nächsten Ereignis den Debug-Kanal mitlesen.

### Nachtrag 17:46 — der Rest des Tages misst korrekt

Stand 1,95 kWh. Gegenprobe der Nachmittagsstunden (Formel gegen gebucht): 14:00 → 0,22/0,18, **15:00 → 0,24/0,24**, 13:00 → 0,10/0,05. Der Flow rechnet sauber, die Abweichungen sind das Quantisierungsrauschen der Batrium-Zähler (0,1-kWh-Stufen). Keine verworfenen Fenster mehr nach dem Vormittag.

Der niedrige Nachmittagswert hat einen simplen Grund: **AC-Abgabe 0,22 kWh in fünf Stunden.** Der Multi hat kaum wechselgerichtet — der Fronius versorgte das Haus direkt und lud den Überschuss über den Lader in die Batterie (1–2 kWh/h). Der variable DC→AC-Anteil fällt damit weg, übrig bleiben Leerlauf + ~3 % Ladeverlust. Modellabgleich 12–17 Uhr: 0,71 kWh gebucht gegen 0,87 kWh reinen Leerlaufsockel — richtige Größenordnung.

**Tagesprognose:** ~3,1 kWh gebucht bzw. ~3,6 kWh inklusive der fehlenden 0,5 kWh, gegen 3,97 (17.08.) und 4,05 (16.08.). Die Restdifferenz von ~0,3 kWh ist der marginale Wandlungsanteil — der einzige echte Wettereffekt. **Fazit: Nur die drei Nachtstunden sind defekt, nicht der Tag.**

### Verifikation der Verlust-Bilanz: alle 12 vebus-Energiepfade geprüft (18.08.2026)

Anlass: Rückfrage, ob die Kennzahl womöglich Fronius-Verluste einsammelt und warum sie sinkt, obwohl der Multi als Lader zugeschaltet war (Flow *Fronius-Ladedeckel*, setzt `/Settings/SystemSetup/MaxChargeCurrent` von 5 auf 50 A).

Gemessen per temporärem Node-RED-Diagnose-Tab (danach wieder gelöscht), Rohwerte seit dem Zähler-Reset vom 12.08.:

| Pfad | kWh | im Flow |
|---|---|---|
| InverterToAcIn1 | 68,34 | ✅ AC-Abgabe |
| InverterToAcOut | 1,04 | ✅ AC-Abgabe |
| AcIn1ToInverter | 6,43 | ✅ AC-Aufnahme |
| AcIn1ToAcOut | 1,71 | Durchleitung, nicht gewandelt |
| AcOutToAcIn1 | 5,44 | Durchleitung, nicht gewandelt |
| **OutToInverter** | **0,35** | ⚠️ **fehlt in der Bilanz** |
| AcIn2* / InverterToAcIn2 | 0 | kein zweiter AC-Eingang |

**Ergebnis: Die Bilanz ist korrekt konstruiert.** Sie umschließt DC-Bus + MultiPlus; der Fronius hängt AC-seitig außerhalb und taucht nicht auf — seine Wandlungsverluste stecken bereits im gemessenen Fronius-Ertrag (AC-seitiger Zähler). Einzige echte Lücke: `OutToInverter` (Laden aus einer Quelle am AC-Out), mit ~0,06 kWh/Tag ohne praktische Bedeutung. ⬜ Bei Gelegenheit als 7. Quelle ergänzen.

### Warum die Verluste bei Lader-Zuschaltung trotzdem sinken

Der intuitive Denkfehler: An einem sonnigen Tag ist der Multi **nicht** im Leerlauf, während die MPPTs laden — dann arbeitet er am härtesten.

| | 17.08. (sonnig) | 18.08. (trüb) |
|---|---|---|
| MPPT-Ertrag (DC) | 16,57 | 5,69 |
| Batterieladung | 8,20 | 8,00 |
| **DC-Überschuss, der gewandelt werden MUSS** | **+8,37** | **−2,31** |
| Durchsatz durch den Wandler | **13,69** | **7,59** |

DC-gekoppelte PV hat nur einen Weg ins Haus: durch den Wechselrichter. Gestern mussten 8,37 kWh Überschuss plus 8,30 kWh nächtliche Batterieentladung gewandelt werden. Heute reichten die MPPTs nicht einmal für die Batterie — **die Lader-Zuschaltung kam also nicht zusätzlich zum Normalbetrieb, sondern anstelle des weggefallenen Wechselrichterbetriebs.** 45 % weniger Durchsatz, entsprechend weniger Abwärme; der Leerlaufsockel bleibt konstant.

**Zerlegung 17.→18.08.:** 3,97 − 0,31 (weniger DC→AC) **+ 0,10 (Lader-Zuschaltung)** − 0,50 (fehlende Nachtstunden) = **3,26 kWh** erwartet, ~3,1 prognostiziert gebucht. Die Lader-Zuschaltung ist der einzige Posten mit positivem Vorzeichen — sie erhöht die Verluste tatsächlich, nur mit ~3 % Ladeverlust auf 3,14 kWh Mehrmenge zu wenig, um sichtbar zu werden.

## ⚠️ Zweite Regression widerspricht dem Leerlaufsockel (18.08.2026) — zu klären

Anlass: Rückfrage, ob die Verluste hauptsächlich beim DC→AC-Wandeln entstehen, also nachts bzw. wenn die MPPT-Energie bei voller Batterie durchgereicht wird.

Eigene Regression über **120 Stunden (13.–17.08.)**, Datenbasis gegen die Tagessummen validiert (alle 5 Tage exakt):

```
Verlust = 0,0710 × AC-Abgabe + 0,1176      R² = 0,47
```

| Kennwert | diese Rechnung (13.–17.08.) | Vault-Regression (08.–10.08.) |
|---|---|---|
| Sockel | **118 W** | 174 W |
| marginal je kWh | **7,1 %** | 3,3 % |
| Direktmessung Ruhestunden | **107 W** (n=22) | 200 W (n=12) |

**Beide Methoden dieses Zeitraums stimmen überein** (Regression 118 W, Direktmessung 107 W) und liegen deutlich unter den bisher als maßgeblich notierten 174–200 W.

**Wahrscheinlichste Erklärung: der Zusammenhang ist nicht linear.** Beide Geraden laufen durch ihren jeweiligen Datenschwerpunkt; eine steilere Steigung drückt zwangsläufig den Achsenabschnitt. Bei 3 parallelen Multis mit Zuschaltschwellen ist Nichtlinearität zu erwarten. Der Achsenabschnitt ist eine **Extrapolation** auf einen Betriebspunkt, der in den Daten kaum vorkommt — real gemessene Ruhestunden sind belastbarer als jede Extrapolation.

⬜ **Zu klären, bevor die Winter-Entscheidung darauf aufbaut:** Liegt der Bereitschaftsverbrauch näher an 110 oder an 175 W? Der Unterschied verschiebt das Standby-Sparpotenzial um gut ein Drittel. Sauberer Weg: gezielt Stunden mit Durchsatz ≈ 0 über mehrere Wochen sammeln, statt aus Lastdaten zu extrapolieren. Zweite Hypothese, die zu prüfen wäre: Der seit 12.08. eingebaute `stalled`-Check verwirft Fenster und könnte systematisch untererfassen (am 18.08. nachweislich 0,5 kWh).

### Wo die Verluste zeitlich anfallen

| Zeitfenster | mittlere Verlustleistung |
|---|---|
| 00–07 Uhr (Batterie versorgt Haus) | 140 W |
| 09–16 Uhr (PV-Betrieb) | **188 W** |
| Stunden fast ohne Durchsatz (n=22) | 107 W |
| Stunden mit ≥ 1 kWh Durchsatz (n=25) | **244 W** |

**Nicht die Nacht ist der Haupttreiber, sondern der sonnige Nachmittag** — nachts liefert der Multi 0,4–0,6 kWh/h, mittags bis 2,7 kWh/h. Aufteilung über die 120 h: 68 % Sockel, 32 % durchsatzabhängig.

### Betriebsfrage: Multi-Durchsatz drosseln, um Verluste zu sparen? → Nein

Geprüft am 18.08.2026. **Verlustminimierung ist hier die falsche Zielgröße** — der Durchsatz skaliert mit dem Nutzen.

Bei voller Batterie hat MPPT-Energie nur zwei Wege: durch den Wechselrichter oder Abregelung. Ein dritter existiert nicht.

| 1 kWh MPPT-Überschuss, Batterie voll | Wert |
|---|---|
| wandeln → Eigenverbrauch (0,93 kWh AC) | **0,32 €** |
| wandeln → Einspeisung | 0,07 € |
| abregeln | **0,00 €** |

Die 7,1 % Wandlungsverlust kosten 7 % des Werts — sie zu vermeiden kostet 100 %.

| 1 kWh Fronius-Überschuss (AC) | Rechenweg | Wert |
|---|---|---|
| einspeisen | 1,0 × 0,07382 | 0,07 € |
| **Batterie laden** | 0,94 × 0,93 × 0,3494 | **0,31 €** |

Faktor 4,4 zugunsten des Ladens → die Lader-Zuschaltung bei schlechtem Wetter ist wirtschaftlich richtig, auch wenn sie die Verlustkennzahl erhöht. Eingespeist wird nur, was die 11,5 kWh nutzbare Kapazität nicht mehr aufnimmt.

**Einziger echter Hebel bleibt der Bereitschaftsverbrauch**, weil er unabhängig vom Durchsatz läuft: 118 W × 24 h = 2,83 kWh/Tag → im Sommer ~76 €/Jahr (frisst nur Einspeisevergütung), im Winter ~361 €/Jahr (frisst Netzstrom). Deshalb ist die offene 110-vs-175-W-Frage praktisch relevant — sie verschiebt diese Zahlen um ein Drittel. Der bekannte Haken bleibt: Die Multis arbeiten ~20 h/Tag, nur ein Bruchteil des Sockels ist abschaltbar.

## 💶 Leerlaufsockel in Euro: Saisonrechnung (18.08.2026)

Ergänzt den Abschnitt *„📌 Für den Winter: Entscheidungsgrundlage Multi-Standby"*. **Saisonaufteilung 8 Monate Sommer / 4 Monate Winter — Setzung Norbert**, plausibel für diese Anlagengröße.

> [!WARNING] Frühere Zahlen richtig lesen
> Die zuvor genannten „76 €" und „361 €" waren **keine** Sommer-/Winteranteile, sondern zwei Jahres-**Extremfälle**: der Sockel durchgängig zu Einspeisevergütung bzw. durchgängig zu Bezugspreis bewertet. Die Realität liegt dazwischen — erst die Saisonaufteilung macht daraus eine Jahreszahl.

### Belegt: Im Sommer kostet der Sockel nur Einspeisevergütung

| Monat | Netzbezug | Einspeisung |
|---|---|---|
| Juli 2026 | **3,89 kWh** | 2.447 kWh |
| August 2026 (bis 18.) | **2,36 kWh** | 1.596 kWh |

Netzbezug praktisch null → der Leerlauf frisst ausschließlich PV, die sonst eingespeist worden wäre. Bewertung mit 7,382 ct ist für diese Monate **gemessen, nicht geschätzt**. ⚠️ Der Gridmeter-Sensor existiert erst seit Juli 2026, für den Winter fehlen Messdaten.

### Jahreskosten (243 d × 7,382 ct + 122 d × 34,94 ct)

| Sockel | kWh/Jahr | Sommer | Winter | **Jahr** |
|---|---|---|---|---|
| **118 W** (Regression 13.–17.08.) | 1.034 | 50,80 € | 120,73 € | **171 €** |
| **174 W** (Vault-Regression 08.–10.08.) | 1.524 | 74,91 € | 178,04 € | **253 €** |

Die Spanne von 82 € ist exakt die offene Sockelfrage (110 vs. 175 W, s. Abschnitt oben). **Über 70 % der Kosten fallen in einem Drittel des Jahres an** — der Winter dominiert, weil der Bezugspreis das 4,7-fache der Einspeisevergütung ist.

### Hebbar ist nur der Winteranteil — und dort nur die echten Standby-Stunden

| Standby-Stunden/Tag (Winter) | bei 118 W | bei 174 W |
|---|---|---|
| 4 h | 20 € | 30 € |
| 8 h | 40 € | 59 € |
| **12 h** | 60 € | **89 €** |

Die bisher notierten **65–70 €/Jahr** entsprechen grob **12 Standby-Stunden/Tag beim hohen Sockelwert** — also der optimistischen Ecke beider Unsicherheiten. Realistisch ist eher die Mitte der Tabelle.

### ⬜ Messplan für den kommenden Winter

Beide offenen Fragen brauchen dieselben Daten und lassen sich in einem Aufwasch klären:

1. **Sockelhöhe:** Stunden mit Durchsatz ≈ 0 sammeln und direkt mitteln, statt aus Lastdaten zu extrapolieren.
2. **Hebbare Stunden:** Stunden zählen, in denen AC-Abgabe ≈ 0 **und gleichzeitig Netzbezug** läuft — das ist die Zeit, in der die Batterie leer ist, das Haus am Netz hängt und der Multi nutzlos im Leerlauf steht. Genau diese Stunden sind das Sparpotenzial.

Erst danach ist die Standby-Entscheidung belastbar. Vorher gilt: Im Sommer lohnt Abschalten ohnehin nicht (kostet nur Einspeisevergütung).

---

## Runbook: Home Assistant von außen (NPM-Proxy, Alexa-Skill)

→ Verschoben nach [[homelab-infrastruktur#Netzwerk-Vorfall 18.08.2026 und Fernzugriff]] (23.08.2026).

