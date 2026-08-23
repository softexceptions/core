---
tags: [journal, victron, node-red, energie]
status: abgeschlossen
date: 2026-07-31
---

# Victron-Journal Juli 2026

Verlauf der Arbeiten an der Victron-Anlage im Juli 2026 — Aufbau der Flows,
Energie-Dashboards, erste Auswertungen.

> [!note] Was hier steht
> Ein Journal wird **nicht überschrieben**. Es hält fest, was passiert ist,
> einschließlich verworfener Hypothesen und später korrigierter Zahlen.
> Was **aktuell gilt**, steht in den Themennotizen — nicht hier.
>
> Reihenfolge wie in der Ursprungsnotiz (grob, nicht streng chronologisch);
> das Datum steht in jeder Überschrift.

Übersicht: [[victron_node_red]] · Folgemonat: [[2026-08 Journal]]

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

