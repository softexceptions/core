---
tags: [journal, victron, node-red, energie]
status: aktiv
date: 2026-08-31
---

# Victron-Journal August 2026

Verlauf der Arbeiten an der Victron-Anlage im August 2026 — Cerbo-Absturz und
Wächter, Ersparnis-Kennzahl, Härtung der Zähler-Flows, Verlust-Analysen.

> [!note] Was hier steht
> Ein Journal wird **nicht überschrieben**. Es hält fest, was passiert ist,
> einschließlich verworfener Hypothesen und später korrigierter Zahlen.
> Was **aktuell gilt**, steht in den Themennotizen — nicht hier.

Übersicht: [[Victron Anlage]] · Vormonat: [[2026-07 Journal]]

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

```
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

```
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

```
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

## Runbook: Home Assistant von außen (NPM-Proxy, Alexa-Skill)

→ Verschoben nach [[homelab-infrastruktur#Netzwerk-Vorfall 18.08.2026 und Fernzugriff]] (23.08.2026).


## Prognose-Frage geklärt: beide Dienste bleiben (23.08.2026)

Zwei seit Juli offene Punkte von Norbert beantwortet:

- **Forecast.Solar wird behalten** — nicht als unterlegener Vergleichskandidat,
  sondern gezielt, um den Ertrag des nächsten Tages zu sehen. Solcast bleibt die
  primäre Prognose im Energie-Dashboard. Der Plan „nach dem Sonnentage-Test den
  Verlierer entfernen" ist damit hinfällig.
- **Odenwald-Verdacht nicht bestätigt.** Der vermutete Ost-Horizont-Effekt
  (4–6°, geschätzt 2–5 % Ost-Tagesertrag durch späteren Sonnenaufgang) zeigte
  sich in der Beobachtung nicht. Keine Erwartungskorrektur nötig.

Gültiger Stand jetzt in [[Anlage und Topologie#PV-Prognose]].
