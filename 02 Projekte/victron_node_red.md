---
tags: [projekt/aktiv, victron, node-red, energie, homeassistant, mqtt, pv]
status: aktiv
date: 2026-07-05
updated: 2026-08-23
---

# Victron Node-RED

Node-RED-Flows für die Victron-Anlage, Repo: `~/Code/victron_node_red`.

> [!info] Umbau läuft (Etappe 2 von 4, Stand 23.08.2026)
> Diese Notiz wird in einen Ordner mit Themennotizen und einem Journal aufgeteilt.
> **Erledigt:** Frontmatter ergänzt, projektfremde Themen ausgelagert, Journal abgetrennt.
> **Offen:** Themennotizen destillieren (Etappe 3), MOC anlegen und umbenennen (Etappe 4).

**Verlauf (Journal — was passiert ist):**
- [[2026-07 Journal]] — Aufbau der Flows, Dashboards, erste Auswertungen (34 Einträge)
- [[2026-08 Journal]] — Cerbo-Absturz und Wächter, Ersparnis-Kennzahl, Zähler-Härtung (14 Einträge)

Was hier steht, ist der **aktuell gültige Stand**. Wie es dazu kam, steht im Journal.

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

## Ziel
Gridmeter **VM-3P75CT** in Home Assistant einbinden, insbesondere fürs Energie-Dashboard.

## MQTT-Broker-Landkarte (2026-07-06)
- **192.168.2.233** — Proxmox-Mosquitto (HA + Node-RED), **Auth erforderlich** (anonym: Not authorized)
- **192.168.2.29** — openWB 2.x, anonym lesbar. EV-Zähler: `openWB/chargepoint/1/get/imported` in **Wh** (÷1000!). openWB sieht auch Netzzähler (counter/14), Victron-Batterie (bat/16), PV (pv/21) → macht eigenes PV-Überschussladen. Wallbox-Hardware: 192.168.2.145.
- **192.168.2.181** — Cerbo GX Venus-MQTT (`N/c0619ab31b3b/...`), anonym lesbar, Keepalive-gebunden

## Nächster Schritt
- Verifizieren (tagsüber): PV-Sensoren (Sonnenaufgang), Batterie „Geladen" (erste Ladephase), E-Auto-Balken (erste Ladung), Validierungswarnungen im Energie-Dashboard sollten alle weg sein; Sankey-Karte zeigt dann Flüsse inkl. E-Auto-Ast.

## Anlagen-Stammdaten (2026-07-06, von Norbert bestätigt + OSM-verifiziert)
- **Gesamt: 25,2 kWp**, Satteldach **30°**, Ost/West. Firstachse 13,8° NNO (aus OSM-Gebäudeumriss 15,7×9 m, Friedrich-Ebert-Str. 17, 69493 Hirschberg-Großsachsen, way 151647625; Standort 49.51404, 8.65993).
- Dachflächen-Azimut (OSM-berechnet, ersetzt Norberts Schätzung −80/+110): **Ost −76°** zu Süd (Kompass 104°), **West +104°** zu Süd (Kompass 284°).
- **Fronius Symo 17.5-3-M** (AC 17,5 kW): 10,08 kWp Ost + 10,08 kWp West (2 MPP-Tracker).
- **2× SmartSolar MPPT 250/70** (DC, an Batterie): „MPPT Ost" (Instanz 278) 2,52 kWp, „MPPT West" (277) 2,52 kWp. 31-Tage-Peaks: Ost 3.098 W, West 2.828 W (per Venus-MQTT `/History/Daily/n/MaxPower` ausgelesen — Keepalive an `R/c0619ab31b3b/keepalive`, dann `N/.../solarcharger/#`).
- Pro Seite gesamt: **12,6 kWp Ost, 12,6 kWp West**.

## Begriffe / Hausgeräte
- **„SK" = Split-Klimaanlage** (nicht Stromkreis!). Zwei Stück, per Zigbee-Messsteckdose erfasst: `sensor.0xa085e3fffebc4574_power` (**SK AZ** = Arbeitszimmer/Küche) und `sensor.0xa085e3fffebd16b8_power` (**SK WZ** = Wohnzimmer). Beide als individual-Kreise in der PFCP-Karte (`energie-sys`), Icon mdi:air-conditioner. Ihre kWh-Zähler (`..._energy`) sind Kandidaten für device_consumption im Energie-Dashboard (noch nicht eingetragen).

## Geplant (zurückgestellt bis Herbst): Hochlast-Gauge + Fix/Variabel-Verlustmodell (2026-07-10)

**Anlass:** E3DC-Vergleich („>95 %" = Datenblatt-Peak vs. unsere gemessenen 73,6 % Nacht-Roundtrip). Befund: kein Technik-Nachteil, sondern andere Kennzahl — realer E3DC-Nachtbetrieb liegt laut PV-Forum ebenfalls bei ~80 % und schlechter. Einziger struktureller Nachteil unserer Anlage: Überdimensionierung (3× 10 kVA fürs Whole-home-Backup → ~173 W Fixverlust nachts, davon Eigenverbrauch ~2 kWh/Tag ≈ 83 W).

**Beschlossenes Design (noch NICHT umgesetzt — Hochsommer, Hochlast-Fenster zu selten):**
1. **Hochlast-Gauge:** zweites Akkumulator-Paar (`accAcHi`/`accDcHi`) im Wirkungsgrad-Flow, Gate wie gehabt (PV < 30 W, entlädt) **plus Entladeleistung > 1,5 kW**. Dritter Discovery-Sensor „Entlade-Wirkungsgrad Hochlast" + Gauge in `victron-sys`. Erwartung ~88–91 %. Publikation erst ab > 1 kWh (sonst unknown). **Umsetzung im Generator `tools/gen_eff_flow.py`, nie direkt im Flow-JSON!**
2. **Datenlage:** Sommer dünn (dunkel erst ~22 Uhr; nur späte Spülmaschine/Trockner). Ab Herbst füllt es sich täglich (Kochen bei Dunkelheit + **Split-Klimas im Heizbetrieb 1–3 kW aus dem Akku**) — liefert genau den η am Heizlast-Betriebspunkt für die Wärmepreis-Rechnung. Schwelle bewusst 1,5 kW (datenblatt-vergleichbar); Alternative 1,2 kW (BW-WP+Grundlast fiele rein, aber näher an Mittellast) verworfen. E-Auto aus dem Akku wäre zwar Messfutter, ist aber unerwünschtes Szenario (doppelter Roundtrip, 15 kWh in ~3 h leer).
3. **Fix/Variabel-Verlustmodell statt „Eigenverbrauch herausrechnen":** Bereinigen um angenommene 2 kWh/Tag ergäbe nachts ~84–86 % Wandlungs-η, wäre aber Modell statt Messung (Annahme-Konstante!) und für Wirtschaftlichkeit falsch (Eigenverbrauch ist real bezahlt). Stattdessen: Verlust(P) = P_fix + k·P. **Die zwei Bins liefern zwei Punkte der Verlustgeraden → P_fix und k empirisch lösbar** — dann ist der Eigenverbrauch gemessen statt geschätzt, und der Roundtrip für jedes Durchsatz-Szenario (30-kWh-Ausbau, Winterheizung, Arbitrage) berechenbar. Kern-Einsicht: η ist Funktion des Durchsatzes; 2 kWh Fix bei 5 kWh Nacht-Durchsatz = 29 % Overhead → 73 %; bei 15–20 kWh Winter-Durchsatz ~9 % → 85–88 % von allein.
4. Referenz-Nacht zum Nachrechnen: 4,90 kWh DC-Entladung, 3,62 kWh AC über 7,4 h → 73,9 %; bereinigt 3,62/(4,90−0,61) = 84,4 %.

**Trigger für Umsetzung:** Herbstbeginn / erste Heiztage, oder wenn Norbert es früher will. Aufwand klein (Generator + Deploy + 1 Gauge). Damit erledigt sich auch die offene **Gauge-Schwellen-Frage** (grün ≥ 88): Das bestehende Gauge bleibt der ehrliche Grundlast-Wert, das neue zeigt den vergleichbaren Hochlast-Wert.

## Offen
- **Grafana „BW-WP Heizungskeller": Reset-Tag 12.07. bleibt ohne Tagesbalken** — der Zähler-Reset durchs Firmware-Update (7,41 → 0, 13:13 Uhr) lässt `difference(nonNegative: true)` den Tag verwerfen; Datenpfad HA → InfluxDB → Grafana nachweislich intakt, ab 13.07. Selbstheilung. Entscheidung offen: Panels auf reset-festes Muster umbauen (erst Punkt-Differenz, dann Tages-Summe) — Diagnose, Nachrechnung und Query-Muster in [[homelab-monitoring]] (Session 2026-07-12 + Stolperfalle „Zähler-Reset reißt ein Tagesloch").
- ~~**BW WP in der EFCP-Energiebilanz zeigt 840 Wh statt 1,26 kWh**~~ ✅ ERLEDIGT 2026-07-18: Nach Norberts Update zeigt die EFCP-Karte wieder exakt die Statistik-Werte (siehe [[2026-07 Journal#EFCP-Anzeige-Bug behoben + SK-WZ-„20-Wh"-Fehlalarm (2026-07-18) ✅]]).
- **Prognose-Vergleich bis ~11.07.** (Solcast ist seit 06.07. primär): Tages-Ist (`victron_system_pv_energie_gesamt`) vs. Solcast (`solcast_pv_forecast_prognose_heute`) vs. Forecast.Solar (min_max-Helfer `pv_prognose_heute`) — heute Abend erster Datenpunkt (Solcast 104,4 / FS 52,3 / VRM 89–109). Nach den Sonnentagen 9.–11.07. (Met.no: sonnig, Klartag ≈ 130–150) Verlierer entfernen: FS-Integrationseinträge + ggf. Helfer auf Solcast umziehen oder löschen.
- **Odenwald-Verdacht** (Horizont Ost 4–6°, ~30–45 min späte Morgensonne, geschätzt 2–5 % Ost-Tagesertrag) gilt auch für Solcast: nach 1–2 Wochen prüfen, ob Ist morgens systematisch unter Prognose startet. Solcast Hobbyist hat allerdings keine Damping-Option — wäre dann nur dokumentierte Erwartungskorrektur.
- **Tages-Verifikation** (Rest): E-Auto-Balken nach erster Ladung; PFCP Netz-Richtung visuell bestätigen (Konvention stimmt laut Doku überein: Victron positiv=Bezug = PFCP-Erwartung; aktuell Einspeisung, Punkte müssen Haus→Netz laufen). ✅ Erledigt am 2026-07-06: PV-Zähler + PV-Summe liefern, Batterie „Geladen" liefert, PFCP-Batterie-Richtung in beiden Richtungen verifiziert (`invert_state: true` bleibt, hergeleitet in [[2026-07 Journal#PFCP-Batterie-Vorzeichen: Verifikation mit Doppel-Irrtum (2026-07-06) ✅]]), EFCP-Batterie-Zuordnung (consumption=entladen) per Grid-Analogie bestätigt.
- Optional: CO2 Signal (Grün/Fossil + Carbon-Gauge), Forecast.Solar (PV-Prognose), weitere Geräte-Inventur — für das große Energie-Board nach Vorbild-Screenshot.
- Kosmetik: Anzeigename doppelt „Victron" („Victron VM-3P75CT Victron Netzbezug") — bei Gelegenheit Discovery-Namen kürzen (Grid-Flow).
- **rateLimit der Victron-Nodes greift nicht** (Spannung/Strom/Leistung updaten 1×/s trotz rateLimit 5000): Semantik prüfen (evtl. nur mit onlyChanges=false wirksam?) oder Delay-Nodes (rate 1/5s, drop) nachrüsten — Recorder-Volumen. Anzeige-Präzision im Registry gesetzt: Spannung 2, Strom 1 Nachkommastellen.

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

Die Spanne von 82 € ist exakt die offene Sockelfrage (110 vs. 175 W, siehe Abschnitt „Leerlaufverlust: Stand der Messungen" oben). **Über 70 % der Kosten fallen in einem Drittel des Jahres an** — der Winter dominiert, weil der Bezugspreis das 4,7-fache der Einspeisevergütung ist.

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

