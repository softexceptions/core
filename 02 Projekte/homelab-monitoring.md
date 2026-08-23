---
tags: [projekt/aktiv, homelab, monitoring, grafana, influxdb, homeassistant]
status: aktiv
date: 2026-07-01
updated: 2026-08-10
---

# homelab-monitoring

Betriebsdokumentation für den Monitoring-Stack: Home Assistant → InfluxDB → Grafana.
Die Ansible-Deployment-Notizen (LXC-Erstellung, Playbooks) liegen in [[homelab-ansible]].

## Stack-Übersicht

```
Zigbee-Steckdose → Zigbee2MQTT → Home Assistant
                                        ↓
                                   InfluxDB (LXC 112, 192.168.2.119)
                                        ↓
                                   Grafana  (LXC 128, 192.168.2.214)
```

## Verbindungen

| Dienst | URL | Details |
|---|---|---|
| InfluxDB | `http://192.168.2.119:8086` | Org: `ng`, Bucket: `homeassistant` |
| Grafana | `http://192.168.2.214:3000` | Login: admin (Passwort im Passwort-Manager, siehe Session 2026-08-10) |
| Grafana (in HA eingebettet) | http://192.168.2.186:8123/dashboard-grafana/0 | Home-Assistant-Dashboard mit iFrame-Einbettung, siehe „Grafana in die HA-Sidebar eingebettet" im [[victron_node_red|Victron-Node-RED-Projekt]] |

**Grafana Datasource:** InfluxDB v2, Query Language: Flux, URL `http://192.168.2.119:8086`

---

## HA InfluxDB-Integration

**Verbindung:** UI-basiert (ab HA 2026.x sind Verbindungskeys aus YAML entfernt)
→ HA → Einstellungen → Integrationen → InfluxDB

**Entity-Filter** liegt weiterhin in `configuration.yaml`:

```yaml
influxdb:
  include:
    entities:
      - sensor.0xa085e3fffebc4574_energy    # AZ + Küche
      - sensor.0xa085e3fffebc4574_power     # AZ + Küche
      - sensor.0xa085e3fffebd16b8_energy    # Wohnzimmer
      - sensor.0xa085e3fffebd16b8_power     # Wohnzimmer
      - sensor.0xa085e3fffeb7c870_energy    # BW-WP Heizungskeller (seit 2026-07-09)
      - sensor.0xa085e3fffeb7c870_power     # BW-WP Heizungskeller (seit 2026-07-09)
```

Nach Änderung: HA → Integrationen → InfluxDB → `...` → **Neu laden** (kein HA-Neustart nötig).
HA schreibt nur bei State Changes — zum Testen Steckdose kurz ein-/ausschalten.

---

## Überwachte Steckdosen

| Raum | Gerät | entity_id Prefix | Grafana Dashboard |
|---|---|---|---|
| EG Arbeitszimmer/Küche | Klimaanlage | `0xa085e3fffebc4574` | `eg-arbeitszimmer` |
| EG Wohnzimmer | Klimaanlage | `0xa085e3fffebd16b8` | `eg-wohnzimmer` |
| KG Heizungsraum | Brauchwasser-Wärmepumpe („BrauchwasserWP") | `0xa085e3fffeb7c870` | `bwwp-heizung` (provisioniert, siehe [[victron_node_red]]) |

Pro Steckdose:
- `sensor.<prefix>_power` → Momentanleistung in W
- `sensor.<prefix>_energy` → Kumulativer Zählerstand in kWh

---

## InfluxDB Datenstruktur

HA schreibt `entity_id` **nicht** als indizierten Tag, sondern als reguläre Spalte:

| Feld | Wert |
|---|---|
| `_measurement` | `W` (Leistung) oder `kWh` (Energie) |
| `entity_id` | z.B. `0xa085e3fffebc4574_power` |
| `domain` | `sensor` |
| `_field` | `value` |

> [!important] Query Builder nicht verwenden
> Grafanas Query Builder ruft intern `schema.tagValues()` auf → schlägt fehl mit „no tag keys found".
> Immer **Script Editor** (rohe Flux-Query) verwenden.

---

## Grafana Dashboard-Struktur

Jedes Raum-Dashboard hat 5 Panels (Momentanleistungs-Gauge am 2026-07-05 entfernt,
siehe Design-Standard unten — Echtzeit übernimmt die native HA-Gauge-Karte):

| Panel | Typ | Einheit | Breite |
|---|---|---|---|
| Tagesverbrauch | Bar chart | kWh | w:8 |
| Wochenverbrauch | Bar chart | kWh | w:8 |
| Jahresverbrauch | Bar chart | kWh | w:8 |
| Monatsverbrauch | Bar chart | kWh | w:16 |
| Jahreskosten | Bar chart | EUR | w:8 |

**Einheit setzen:** Standard options → Unit → `watt` bzw. `kwh`

> [!important] Design-Standard: max. 8 Balken pro Drittel-Breite-Panel (seit 2026-07-05)
> Grafanas Bar-Chart unterdrückt Wert-Labels, wenn der Balken-Slot schmaler als der
> Label-Text wird (siehe Stolperfallen-Warnung unten). Getestet stabil: **8 Balken bei
> w:8-Panel** mit `Decimals: 1` + `Text size → Value: 12` (bis unter 380 px Panel-Breite
> getestet; bei 380 px stoßen die Labels fast aneinander — kein Platz mehr für größere Schrift).
> Panels mit mehr Balken müssen breiter sein: das **Monats-Panel (bis 13 Balken) hat
> deshalb w:16** — bewusste Ausnahme, um den Saisonvergleich übers volle Jahr zu behalten.
> Layout seither (5 Panels, 2 Zeilen): Zeile 1 = Tag/Woche/Jahr (je w:8, Verbrauch in
> aufsteigenden Zeiträumen), Zeile 2 = Monat (w:16) + Kosten (w:8). Einheitlich auf
> allen Balken-Panels:
> `showValue: always`, `Text size → Value: 12`, `Decimals: 1` (Ausnahme Kosten:
> 2 Nachkommastellen, Euro-Konvention). Legende: `Standard options → Display name` =
> `SK` statt des Flux-Seriennamens (`value {domain=…, entity_id=…}`) —
> **nicht beim Kosten-Panel** setzen, dort hängen die Rot/Grün-Overrides an den
> Seriennamen „ohne PV"/„mit PV". Die **Legende bleibt sichtbar** (Entscheidung Norbert,
> 2026-07-05): wird auf dem Smartphone in der verkleinerten iFrame-Ansicht zur
> Orientierung gebraucht — nicht ausblenden.
>
> Die **Momentanleistungs-Gauge wurde am 2026-07-05 entfernt** — redundant zur nativen
> HA-Gauge-Karte (reagiert sofort statt 15m-Polling) und bei 15m-Refresh irreführend
> („Momentanleistung" mit bis zu 15 min altem Wert). Arbeitsteilung seither: HA = Echtzeit,
> Grafana = historische Analyse. Leistungsdaten fließen unverändert nach InfluxDB; die
> Gauge ist mit dem Flux-Muster unten jederzeit wiederherstellbar.
>
> **Norberts Präferenz (2026-07-05): keine Mobile-Optimierung für Grafana.** Auf dem
> Smartphone bewusst die verkleinerte, layouttreue Desktop-Ansicht im HA-iFrame nutzen
> (Zoom per Geste) — Grafanas responsive Einspalten-Ansicht verändert Panel-Proportionen
> und Label-Positionen und ist unerwünscht. Nicht erneut Homescreen-Icon/Mobile-Dashboard
> vorschlagen.

### X-Achsen-Labels (Grafana Transformation)

Transform tab → Add transformation → Convert field type → Time → String → Timezone: Europe/Berlin

| Panel | Date format | Ergebnis |
|---|---|---|
| Tagesverbrauch | `DD.MM.` | 27.06. |
| Wochenverbrauch | `[KW]WW` | KW26, KW27 |
| Monatsverbrauch | `MMM YYYY` | Jul 2026 |
| Jahresverbrauch | `[Jahr ]YYYY` | Jahr 2026 |

> [!note] Moment.js Format
> Grafana verwendet moment.js — Literale mit eckigen Klammern escapen: `[KW]` → „KW".
> Single quotes funktionieren nicht (das ist date-fns Syntax).

> [!warning] Achsen-Label darf nicht rein numerisch aussehen
> `DD.MM` erzeugt für den 27. Juni die Zeichenkette `27.06` — sieht aus wie eine Dezimalzahl.
> Grafanas Bar-Chart-Panel erkennt das und formatiert das Achsen-Label dann mit der im Panel
> hinterlegten Einheit (`kwatth` = kWh) statt es als reine Text-Kategorie zu behandeln.
> Ergebnis: `27.1 kWh` statt `27.06` auf der X-Achse. Betrifft **auch reine Ganzzahlen ohne
> Dezimalpunkt** — `YYYY` (`2026`) wurde ebenfalls als Zahl erkannt und mit SI-Skalierung
> formatiert (`2.03 MWh` statt `2026`). Die ursprüngliche Annahme, Ganzzahlen seien sicher,
> war falsch.
>
> **Fix — zwei Varianten, je nach Format:**
> - Enthält das Format bereits einen Dezimalpunkt (`DD.MM`): einen **zweiten** Punkt anhängen
>   (`DD.MM.`) — macht die Zeichenkette syntaktisch ungültig als Zahl (`Number("27.06.")` → `NaN`).
> - Enthält das Format **keinen** Punkt (`YYYY`): ein Punkt am Ende reicht **nicht**
>   (`Number("2026.")` → `2026`, bleibt gültig!). Stattdessen ein Text-Präfix per
>   Klammer-Escape ergänzen: `[Jahr ]YYYY` → `Jahr 2026`.
>
> Grundsätzlich sicher: jedes Format mit Buchstaben (`[KW]WW`, `MMM YYYY`) — nie rein numerisch.

> [!important] Panel-Konfiguration per Grafana-API vergleichen
> Anonymer Lesezugriff ist aktiviert (für iFrame-Embedding, siehe Session 2026-06-27 (3)) —
> Dashboard-JSON lässt sich ohne Login abrufen und diffen:
> `curl -s http://192.168.2.214:3000/api/dashboards/uid/<uid>` (UID über `/api/search?query=...`)
> Damit lassen sich Panel-Configs 1:1 vergleichen, wenn ein Panel nicht wie erwartet funktioniert.

> [!warning] „Format time" ≠ „Convert field type"
> Grafana bietet zwei ähnlich klingende Transformationen. **„Format time"** hat in der Praxis
> nicht funktioniert — das „Time field"-Dropdown blieb leer (`"timeField": ""`) und die
> Transformation war dadurch ein No-Op (Achse blieb im US-Format). Immer **„Convert field type"**
> verwenden (Field: `Time` → Type: `String` → Date format → Timezone), wie in der Tabelle oben.

---

## Flux-Query-Muster

### Momentanleistung (Gauge)

```flux
from(bucket: "homeassistant")
  |> range(start: -12h)
  |> filter(fn: (r) => r["_measurement"] == "W")
  |> filter(fn: (r) => r["entity_id"] == "0xa085e3fffebc4574_power")
  |> filter(fn: (r) => r["_field"] == "value")
  |> last()
```

### Verbrauch — Zwei-Stufen-Aggregation

Das kWh-Measurement ist ein kumulativer Zählerstand. Korrektes Muster:
**Stufe 1** — Tägliche Differenzen | **Stufe 2** — Pro Zeitraum summieren

> [!important] Warum Zwei-Stufen?
> `aggregateWindow(fn: last) |> difference()` direkt auf Wochenbasis verliert die erste
> (unvollständige) Woche, da kein Vorgänger-Bucket existiert. Die Zwei-Stufen-Methode
> summiert fertige Tageswerte — korrekt auch wenn Daten erst mitten in einer Woche beginnen.

#### Tagesverbrauch (`range: -8d`)

```flux
import "timezone"

from(bucket: "homeassistant")
  |> range(start: -8d)
  |> filter(fn: (r) => r["_measurement"] == "kWh")
  |> filter(fn: (r) => r["entity_id"] == "0xa085e3fffebc4574_energy")
  |> filter(fn: (r) => r["_field"] == "value")
  |> aggregateWindow(
       every: 1d,
       fn: last,
       createEmpty: false,
       location: timezone.location(name: "Europe/Berlin"),
       timeSrc: "_start"
     )
  |> difference(nonNegative: true)
```

#### Wochenverbrauch (`range: -9w`, ISO-KW, Montag-Start)

```flux
import "timezone"

from(bucket: "homeassistant")
  |> range(start: -9w)
  |> filter(fn: (r) => r["_measurement"] == "kWh")
  |> filter(fn: (r) => r["entity_id"] == "0xa085e3fffebc4574_energy")
  |> filter(fn: (r) => r["_field"] == "value")
  |> aggregateWindow(
       every: 1d,
       fn: last,
       createEmpty: false,
       location: timezone.location(name: "Europe/Berlin"),
       timeSrc: "_start"
     )
  |> difference(nonNegative: true)
  |> aggregateWindow(
       every: 1w,
       fn: sum,
       createEmpty: false,
       location: timezone.location(name: "Europe/Berlin"),
       offset: -3d,
       timeSrc: "_start"
     )
```

#### Monatsverbrauch (`range: -13mo`)

```flux
import "timezone"

from(bucket: "homeassistant")
  |> range(start: -13mo)
  |> filter(fn: (r) => r["_measurement"] == "kWh")
  |> filter(fn: (r) => r["entity_id"] == "0xa085e3fffebc4574_energy")
  |> filter(fn: (r) => r["_field"] == "value")
  |> aggregateWindow(
       every: 1d,
       fn: last,
       createEmpty: false,
       location: timezone.location(name: "Europe/Berlin"),
       timeSrc: "_start"
     )
  |> difference(nonNegative: true)
  |> aggregateWindow(
       every: 1mo,
       fn: sum,
       createEmpty: false,
       location: timezone.location(name: "Europe/Berlin"),
       timeSrc: "_start"
     )
```

#### Jahresverbrauch (`range: -3y`)

```flux
import "timezone"

from(bucket: "homeassistant")
  |> range(start: -3y)
  |> filter(fn: (r) => r["_measurement"] == "kWh")
  |> filter(fn: (r) => r["entity_id"] == "0xa085e3fffebc4574_energy")
  |> filter(fn: (r) => r["_field"] == "value")
  |> aggregateWindow(
       every: 1d,
       fn: last,
       createEmpty: false,
       location: timezone.location(name: "Europe/Berlin"),
       timeSrc: "_start"
     )
  |> difference(nonNegative: true)
  |> aggregateWindow(
       every: 1y,
       fn: sum,
       createEmpty: false,
       location: timezone.location(name: "Europe/Berlin"),
       timeSrc: "_start"
     )
```

---

## Bekannte Stolperfallen

**`timeSrc: "_start"` Pflicht** — Standard `_stop` setzt den Zeitstempel ans Fenster-Ende.
Der heutige Balken erscheint erst morgen. Mit `_start` erscheint der laufende Tag sofort.

**`offset: -3d` für Wochen** — Unix-Epoch (1970-01-01) war ein Donnerstag → Fenster laufen
Do→Do ohne Offset. Mit `-3d` verschiebt sich der Start auf Montag = ISO-Kalenderwochen.

**`nonNegative: true`** — Zigbee-Chips können durch Schaltimpulse des Klimakompressors
kurz gestört werden → kWh-Wert springt zurück. `nonNegative: true` filtert das heraus.

**`+1 Puffer` im range()** — `difference()` braucht einen Datenpunkt vor dem ersten
angezeigten Zeitraum: `-8d` für 7 Tage, `-9w` für 8 Wochen, `-13mo` für 12 Monate.

**Erster Tag nach Einrichtung bleibt dauerhaft ohne Balken** — `difference()` vergleicht
den Tages-Endwert gegen den Endwert des Vortags. Für den allerersten Erfassungstag fehlt
diese Vortags-Baseline (keine Daten vor dem Einrichtungszeitpunkt), daher kann für diesen
Tag **nie** ein Balken berechnet werden — auch nicht nachträglich durch Abwarten. Der erste
sichtbare Balken erscheint erst für den **zweiten** vollen Tag nach Einrichtung. Kein Fehler,
sondern strukturell bedingt. Bestätigt an SK_Wohnzimmer (Einrichtung 2026-07-01, 14:35 Uhr —
01.07. bleibt ohne Balken, erster echter Balken ab 02.07.).

> [!warning] Zähler-Reset reißt ein Tagesloch — Dashboard wirkt „eingefroren"
> Wird der kWh-Zähler einer Steckdose zurückgesetzt (Firmware-Update, Neuanlernen,
> Werksreset), zeigt das Tages-Panel für den Reset-Tag **keinen Balken** — und Wochen-/
> Monats-/Jahres-Panels übergehen den Tag ebenfalls. Wirkt wie „Grafana/InfluxDB wird
> nicht aktualisiert", obwohl der Datenpfad HA → InfluxDB einwandfrei läuft.
>
> **Mechanismus:** Die Stufe-1-Query bildet `aggregateWindow(1d, fn: last)` und dann
> `difference(nonNegative: true)`. Am Reset-Tag ist `last(heute) − last(gestern)` negativ
> (z. B. 0,5 − 5,88) → `nonNegative` verwirft den Wert → kein Balken. Zusätzlich ist der
> vor dem Reset aufgelaufene Tagesverbrauch aus Panel-Sicht verloren (steckt zwar in
> InfluxDB, ist aber über den Reset hinweg nicht zuzuordnen).
>
> **Selbstheilung:** Ab dem Folgetag stimmen alle Balken wieder (Differenz gegen den
> neuen, niedrigen Zählerstand ist positiv). Nur der Reset-Tag bleibt dauerhaft leer.
>
> **Reset-festes Muster (Umbau-Option, Stand 2026-07-12 nicht umgesetzt):** Reihenfolge
> umdrehen — **erst** Punkt-zu-Punkt-`difference(nonNegative: true)` auf den Rohdaten,
> **dann** täglich summieren. Beim Reset fällt nur der eine negative Sprung raus, alle
> Zuwächse davor und danach werden gezählt:
> ```flux
> import "timezone"
>
> from(bucket: "homeassistant")
>   |> range(start: -8d)
>   |> filter(fn: (r) => r["_measurement"] == "kWh")
>   |> filter(fn: (r) => r["entity_id"] == "0xa085e3fffeb7c870_energy")
>   |> filter(fn: (r) => r["_field"] == "value")
>   |> difference(nonNegative: true)
>   |> aggregateWindow(every: 1d, fn: sum, createEmpty: false,
>        location: timezone.location(name: "Europe/Berlin"), timeSrc: "_start")
> ```
> Betrifft bei Umbau **alle** Verbrauchs-/Kosten-Panels (Stufe 1 der Zwei-Stufen-
> Aggregation ersetzen; Stufe 2 bleibt). Beim BW-WP-Dashboard liegt die Quelle in der
> provisionierten Datei `/var/lib/grafana/dashboards/bwwp-heizungskeller.json` auf dem
> Grafana-LXC — UI-Edits würden beim nächsten Datei-Re-Import überschrieben. Die
> SK-Dashboards sind UI-verwaltet (Fix per UI/API). Erstmals aufgetreten am 2026-07-12
> (BW-WP, Shelly-Firmware-Update, Zähler 7,41 → 0 um 13:13 Uhr).

> [!warning] Balken-Wert-Label fehlt trotz `showValue: always` — Y-Achsen-Max zu knapp
> Bei „Wöchentlicher Energiebedarf SK AZ und Küche" (2026-07-05) fehlte der Wert-Text über
> dem höchsten Balken, obwohl `showValue: always` gesetzt war. Root Cause per Screenshot-
> Vergleich (Playwright/Chromium gegen `/d-solo/`-Endpunkt) gefunden: Grafanas automatisches
> Achsen-Maximum wählte eine „runde" Gitterlinie (18 kWh), die **unter** dem tatsächlichen
> Datenwert (19,03 kWh) lag — der Balken lief über den sichtbaren Panel-Bereich hinaus,
> das Label hatte keinen Platz zum Rendern. Tritt nicht bei jedem knappen Fall auf (das
> Monatspanel zeigte „26.8 kWh" bei Achsen-Max 27,5 noch korrekt an — dort lag Auto-Max
> knapp **über** dem Datenwert, kein Overflow).
>
> **Fix:** Feld-Tab → **Standard options → Max** einen expliziten Wert mit Puffer über dem
> erwarteten Höchstwert setzen (hier: `25` bei einem Wochenmaximum von ~19 kWh). Verhindert
> sowohl den Balken-Overflow als auch das fehlende Label.
>
> **Diagnose-Muster für „Wert wird nicht angezeigt"-Fälle:** Grafana-Panel-Konfiguration
> per API vergleichen reicht oft nicht (Konfiguration kann identisch und trotzdem das
> Render-Ergebnis unterschiedlich sein, je nach tatsächlichen Datenwerten). Screenshot per
> Playwright gegen `http://<grafana>/d-solo/<uid>/<slug>?panelId=<id>&width=…&height=…&theme=light`
> zeigt das tatsächliche Render-Ergebnis und macht Clipping-Probleme sofort sichtbar.

> [!warning] Balken-Wert-Label fehlt trotz `showValue: always` — Balken-Slot schmaler als Label-Text
> Zweite Ursache neben dem Y-Achsen-Clipping (2026-07-05, „Täglicher Energiebedarf SK AZ und
> Küche"): Grafanas Bar-Chart unterdrückt **alle** Wert-Labels, sobald der horizontale Platz
> pro Balken schmaler ist als der Label-Text — auch bei `showValue: always`. Per Screenshot-
> Serie verifiziert: identisches Panel zeigt bei 620 px Breite alle Labels, bei 450 px und
> 380 px **keine**. Deshalb trifft es Panels mit vielen Balken (Tagespanel: 8 Balken) bei
> schmalem Browserfenster, während das Wochenpanel (1–2 Balken) nie betroffen ist. Erklärt
> auch, warum „im Panel-Editor sichtbar, im Dashboard nicht" auftreten kann — der Editor-
> Preview ist breiter als die Dashboard-Kachel.
>
> **Fix:** Label-Text schmaler machen — `Standard options → Decimals: 1` und
> `Bar chart → Text size → Value: 10`. Damit erscheinen die Labels bis unter 380 px
> Panel-Breite (getestet). Bei noch mehr Balken zusätzlich Panel breiter ziehen oder
> Balkenzahl reduzieren (`range()` verkürzen).

---

## Jahreskosten-Panel (editierbarer Preis pro kWh)

Zusätzliches Panel „Jahreskosten" pro Dashboard — zeigt Kosten ohne PV vs. mit PV als
gruppiertes Balkendiagramm, Preise live über Dashboard-Textfelder editierbar.

### Aufbau

1. **Zwei Dashboard-Variablen** (Settings → Variables → Textbox):
   `preis_ohne_pv` (Default `0.35`), `preis_mit_pv` (Default `0.075`)
2. Jahresverbrauch-Panel duplizieren, umbenennen zu „Jährliche Kosten ..."
3. **Zwei Flux-Queries** (A + B) — je die normale Jahresverbrauch-Query, aber mit Preis
   direkt in Flux eingerechnet (siehe Muster unten)
4. **Transformationen:** `Outer join` (Mode: Time series, Field: Time) **vor** `Convert field type`
5. **Overrides:** je ein Fixed-Color-Override pro Feldname (`Kosten ohne PV` → Rot,
   `Kosten mit PV` → Grün)
6. **Unit:** `Euro (€)`

### Flux-Muster (Preis direkt einrechnen)

```flux
import "timezone"

from(bucket: "homeassistant")
  |> range(start: -3y)
  |> filter(fn: (r) => r["_measurement"] == "kWh")
  |> filter(fn: (r) => r["entity_id"] == "0xa085e3fffebc4574_energy")
  |> filter(fn: (r) => r["_field"] == "value")
  |> aggregateWindow(every: 1d, fn: last, createEmpty: false, location: timezone.location(name: "Europe/Berlin"), timeSrc: "_start")
  |> difference(nonNegative: true)
  |> aggregateWindow(every: 1y, fn: sum, createEmpty: false, location: timezone.location(name: "Europe/Berlin"), timeSrc: "_start")
  |> drop(columns: ["_start", "_stop", "_field", "_measurement", "domain", "entity_id"])
  |> map(fn: (r) => ({ r with _value: r._value * $preis_ohne_pv }))
  |> rename(columns: {_value: "Kosten ohne PV"})
```

Query B identisch, nur `$preis_mit_pv` / `"Kosten mit PV"`.

> [!warning] „Add field from calculation" (Binary operation) funktioniert NICHT mit Variablen
> Das „By value"-Feld bei der Transformation „Add field from calculation" (Mode: Binary
> operation) ist ein reines Zahlenfeld — `${variable}`-Text wird beim Eintippen sofort wieder
> gelöscht. Zwar unterstützen Grafana-Transformationen laut Doku generell Variablen in
> Text-Eingabefeldern, aber dieses spezielle Feld ist keins. **Fix:** Preis stattdessen direkt
> in der Flux-Query verrechnen (siehe Muster oben), keine Transformation nötig.

> [!warning] Flux-eigene `${}`-String-Interpolation kollidiert mit Grafana-Variablen
> Flux nutzt selbst `"${ausdruck}"` für String-Interpolation — exakt dieselbe Syntax wie
> Grafana-Dashboard-Variablen. Schreibt man `float(v: "${preis_ohne_pv}")`, versucht **Flux**,
> `preis_ohne_pv` als eigenen Bezeichner auszuwerten (Fehler „undefined identifier" oder
> „expected RPAREN"), **bevor** Grafana überhaupt zum Interpolieren kommt.
> **Fix:** Die einfache Schreibweise **`$preis_ohne_pv`** (ohne geschweifte Klammern, ohne
> Anführungszeichen) direkt in der Rechnung verwenden — kollidiert nicht mit Flux-Syntax,
> Grafana ersetzt den Text trotzdem. Macht `float(v: ...)` überflüssig, da direkt eine nackte
> Zahl eingesetzt wird (`r._value * $preis_ohne_pv`).

> [!warning] Feld-Labels landen im Anzeigenamen — Override-Matcher werden instabil
> Das `value`-Feld trägt InfluxDB-Tags (`domain`, `entity_id`, `_field`, `_measurement`,
> `_start`, `_stop`) als Labels. `rename()` ändert nur den Spaltennamen, nicht die Labels —
> Grafana hängt sie weiterhin an den Anzeigenamen an: `Kosten ohne PV {_start="...", _stop="...", ...}`.
> Da `_start`/`_stop` bei **jedem** Refresh neue Zeitstempel enthalten, bricht ein per „byName"
> gesetzter Field-Override (z. B. für feste Farben) beim nächsten Laden wieder — die gespeicherte
> Farbe bleibt zwar korrekt, der Matcher trifft aber nicht mehr, Grafana fällt auf die
> automatische Palettenfarbe zurück.
> **Fix:** Vor `rename()` die überflüssigen Tag-Spalten droppen:
> `|> drop(columns: ["_start", "_stop", "_field", "_measurement", "domain", "entity_id"])`
> Danach heißt das Feld sauber nur noch `Kosten ohne PV`, ohne Anhängsel — Override bleibt stabil.

> [!note] Gruppierte Balken einfärben — Duplikat erbt „Fixed color"
> Ein von Wochen-/Jahresverbrauch dupliziertes Panel hat oft `color.mode: "fixed"` (eine feste
> Farbe für die einzige Serie). Bei zwei Serien im selben Panel bekommen dann **beide** Balken
> dieselbe Farbe. Fix: Standard options → Color scheme → **„Classic palette"** (automatisch
> unterschiedliche Farben), oder gezielt per **Field Override** (Fields with name → Color →
> Single color) je Serie eine feste Farbe setzen.

> [!note] Zwei Queries zu einem Balkendiagramm zusammenführen
> Zwei separate Flux-Queries (A + B) liefern zwei getrennte Frames. Damit sie als gruppierte
> Balken im selben Zeit-Raster erscheinen: Transformation **„Outer join"** (Mode: Time series,
> Join field: `Time`) **vor** „Convert field type" einfügen.

> [!note] Grafana Bar Chart: keine Labels direkt unter einzelnen Balken
> Bei gruppierten Balken (mehrere Serien pro Kategorie) gibt es nur **eine** gemeinsame
> Kategorie-Beschriftung (hier: das Jahr) plus eine **Legende** für die Serien-Zuordnung.
> Individuelle Text-Labels direkt unter jedem einzelnen Balken unterstützt der Bar-Chart-Typ
> nicht — die Legende (Standard options → Legend → Show legend, Bottom, List) ist der
> Grafana-native Weg, um „welcher Balken ist was" sichtbar zu machen.

---

## Neue Steckdose hinzufügen

1. `configuration.yaml` um energy + power entity ergänzen
2. HA → Integrationen → InfluxDB → Neu laden
3. Steckdose kurz ein-/ausschalten → ersten InfluxDB-Eintrag triggern
4. Grafana-Dashboard duplizieren (Settings → Save As)
5. In allen 5 Panels den entity_id-Prefix ersetzen

---

## Session 2026-07-04 — Tagesverbrauch-Achse auf deutsches Datumsformat umgestellt

### Was getan wurde

- [x] Tagesverbrauch-Panel bei **SK AZ und Küche** und **SK Wohnzimmer** von US-Format (`06/27`)
      auf deutsches Format (`27.06.`) umgestellt
- [x] Beide Panels nutzten fälschlich die Transformation „Format time" (No-Op, leeres
      `timeField`) — auf „Convert field type" umgestellt (siehe Warnung oben)
- [x] Numerisch-Label-Falle entdeckt und gelöst: `DD.MM` wurde von Grafana als Dezimalzahl
      interpretiert und mit der Panel-Einheit formatiert (`27.1 kWh`) — Fix: `DD.MM.` (siehe
      Warnung oben)
- [x] Beide Dashboards per Grafana-API gegengecheckt — Konfiguration jetzt identisch zu den
      funktionierenden Wochen-/Monats-/Jahres-Panels
- [x] Gleiche Numerisch-Label-Falle auch bei Jahresverbrauch entdeckt: `YYYY` (`2026`) wurde
      als Zahl erkannt und zu `2.03 MWh` skaliert. Trailing-Dot-Fix greift hier **nicht**
      (`2026.` bleibt gültige Zahl) — stattdessen Text-Präfix `[Jahr ]YYYY` → `Jahr 2026`
- [x] Fix in beiden Dashboards angewendet und per API gegengecheckt

### Ergebnis

Alle Tagesverbrauch-Panels zeigen jetzt konsistent `DD.MM.` (z. B. `27.06.`) statt `MM/DD`.
Alle Jahresverbrauch-Panels zeigen `Jahr 2026` statt `2.03 MWh`.

---

## Session 2026-07-05 — Jahreskosten-Panel mit editierbarem kWh-Preis

### Was getan wurde

- [x] Panel „Jährliche Kosten SK AZ und Küche" gebaut — Preis ohne/mit PV live über
      Dashboard-Textfelder editierbar (siehe Abschnitt „Jahreskosten-Panel" oben)
- [x] Mehrere Sackgassen durchlaufen und dokumentiert: „Add field from calculation" akzeptiert
      keine Variablen, Flux-`${}`-Syntax kollidiert mit Grafana-Variablen, Tag-Labels im
      Feldnamen destabilisieren Color-Overrides
- [x] Finales Muster: Preis direkt in Flux via `$preis_ohne_pv` (ohne Klammern/Anführungszeichen)
      einrechnen, `drop()` vor `rename()` für stabile Feldnamen, `Outer join` zum Zusammenführen
      der zwei Queries, Fixed-Color-Overrides für Rot/Grün
- [x] Per API gegengecheckt: Queries, Transformationen, Unit (`currencyEUR`) und Overrides
      korrekt und stabil (Override-Matcher ohne Zeitstempel-Anhängsel)

### Ergebnis

„Jahreskosten"-Panel bei **beiden** Dashboards (SK AZ und Küche + SK Wohnzimmer) vollständig
funktionsfähig: zwei editierbare Preisfelder oben im Dashboard, Balken in Rot (ohne PV) /
Grün (mit PV), Einheit Euro. Beim Übertragen aufs Wohnzimmer-Dashboard nur ein Stolperer:
Panel-Titel nach dem Duplizieren nicht direkt umbenannt (Leerzeichen-Tippfehler beim zweiten
Versuch) — sonst lief die Übertragung mit dem dokumentierten Muster beim ersten Versuch durch.

---

## Session 2026-07-05 (2) — Wert-Label „Wöchentlicher Energiebedarf" fehlt weiterhin *(gelöst in Session 2026-07-05 (3))*

### Ausgangspunkt

Bei „Wöchentlicher Energiebedarf SK AZ und Küche" fehlte der Wert-Text über dem Balken.

### Diagnose-Verlauf

1. **Config-Vergleich per API** (AZ vs. Wohnzimmer) — identisch bis auf `showValue`. Kein
   Config-Fehler.
2. **Screenshot-Vergleich per Playwright** (`/d-solo/<uid>/<slug>?panelId=<id>`) — zeigte den
   echten Root Cause: Balken (19,03 kWh) überschritt das automatische Achsen-Max (18 kWh) —
   Balken lief über den Panel-Rand, kein Platz fürs Label. Fix 1: `Standard options → Max`
   explizit auf `25` — funktionierte (bestätigt vom User).
3. **Nutzer-Einwand:** Hartes `Max` ist unkomfortabel (wächst der Verbrauch, muss man
   nachjustieren, sonst kappt es künftige Balken). Bessere Lösung recherchiert: **`Axis → Soft
   max`** (nicht `Standard options → Max`) — wirkt nur als Mindest-Ausschlag, Grafana erweitert
   die Achse automatisch weiter, wenn Daten den Soft-Max-Wert übersteigen. Kein Clipping, keine
   manuelle Pflege. Fix 2: `axisSoftMax: 20` gesetzt, hartes `Max` entfernt.
4. **Nutzer-Rückmeldung:** „im Panel wird es angezeigt, im Dashboard nicht" — Label erscheint
   im Panel-Editor (großer Preview), aber nicht in der eingebetteten Dashboard-Kachel.
5. **Screenshot-Vergleich isoliert vs. eingebettet:** Bei identischer Panel-Größe (500×270px)
   zeigt die isolierte `/d-solo/`-Ansicht den Wert korrekt an — im vollen Dashboard (6 Panels,
   „10s"-Auto-Refresh) fehlt er weiterhin, reproduzierbar auch nach Seiten-Reload und nach
   einem vollen Refresh-Zyklus. Größen-Theorie damit widerlegt (auch bei 2-Spalten-Layout mit
   mehr Panel-Breite als sonst trat das Problem weiterhin auf).

### Offene Hypothese

Einziger verbleibender struktureller Unterschied: aktives **„10s"-Auto-Refresh** im vollen
Dashboard (isolierte Testansicht hatte keinen Auto-Refresh). Vermutung: Bei jedem Reload wird
die Wochen-Query neu ausgeführt: an Wochengrenzen kann sich die Balkenanzahl kurzzeitig ändern
— das könnte einen Redraw-Bug in der Chart-Bibliothek auslösen, der ausgerechnet dieses Panel
(im Gegensatz zum Monatspanel) trifft. Nicht abschließend verifiziert.

**Nächster Schritt (nächste Session):** Dashboard-Refresh-Intervall von `10s` auf z. B. `15m`
hochsetzen (Wochenwerte brauchen ohnehin keine Sekundenaktualität) und prüfen, ob das Label
danach erscheint.

> [!warning] Vorfall: Grafana kurzzeitig nicht erreichbar durch Diagnose-Automatisierung
> Während der Diagnose wurden mehrere Playwright/Chromium-Sessions gegen das Live-Dashboard
> gestartet, teils mit `wait_until="networkidle"` gegen ein Dashboard mit aktivem 10s-Refresh
> (verhindert das Idle-Signal, führt zu langen/hängenden Requests) und ein Aufruf des
> `/render/d-solo/...`-Endpunkts (kein Image-Renderer-Plugin installiert → 500er, evtl.
> hängender Prozess). Kurz danach waren sowohl Grafana (Port 3000) als auch SSH (Port 22) auf
> dem LXC nicht mehr erreichbar (nur ICMP-Ping ging noch durch). Fix: `systemctl restart
> grafana-server` auf dem LXC durch den User — danach sofort wieder normal.
>
> **Lehre:** Gegen dieses Grafana-Dashboard künftig keine Playwright-Sessions mit
> `networkidle` fahren (Dashboard hat Dauer-Refresh, Idle-Signal kommt nie) — stattdessen
> `domcontentloaded` + festes `wait_for_timeout` verwenden, Browser in `finally` schließen,
> und den `/render/`-Endpunkt meiden, da kein Renderer-Plugin installiert ist.

---

## Session 2026-07-05 (3) — Wert-Labels stabil gefixt (Tag + Woche, beide Dashboards)

### Ausgangspunkt

Wert-Labels über den Balken fehlten erneut — diesmal bei **Täglicher** und **Wöchentlicher
Energiebedarf** (SK AZ und Küche). Anforderung: Fix, der dauerhaft stabil bleibt.

### Diagnose (per API, read-only — kein Playwright nach Vorfall von gestern)

- Tages-Panel stand noch auf `showValue: auto` (Grafana blendet Labels bei `auto` situativ
  aus) und hatte kein Soft max
- Wochen-Panel hatte den Fix von gestern (`always` + Soft max 20), aber Soft max 20 lag nur
  5 % über dem bisherigen Maximum (19,03 kWh) — kaum Puffer
- Dashboard-Refresh stand weiterhin auf `10s` (der dokumentierte Verdacht aus Session (2))
- SK Wohnzimmer hatte noch **gar keine** der Korrekturen (alles `auto`, kein Soft max, 10s)
- Tageswerte per `/api/ds/query` geprüft: Max. 8,52 kWh (28.06.) → Soft max 10 passt

### Angewendeter Fix (per API, `POST /api/dashboards/db`)

| Einstellung | SK AZ und Küche (v75) | SK Wohnzimmer (v28) |
|---|---|---|
| Dashboard-Refresh | 10s → **15m** | 10s → **15m** |
| Täglich: showValue / Soft max | **always** / **10** | **always** / **2** |
| Wöchentlich: showValue / Soft max | always / 20 → **25** | **always** / **5** |

Monats-/Jahres-/Kosten-Panels unverändert (dort trat das Problem nie auf).

### Warum das stabil ist

Drei Ursachen gleichzeitig adressiert: `showValue: always` erzwingt das Label (kein
situatives Ausblenden), Soft max mit ~30 % Puffer verhindert Balken-Overflow über den
Panelrand (Clipping-Falle, siehe Stolperfallen-Warnung oben), und Refresh 15m eliminiert
den 10s-Redraw-Verdacht aus Session (2). Momentanleistung bleibt live über die native
HA-Gauge-Karte sichtbar — das Grafana-Dashboard braucht kein Sekunden-Refresh.

### Verifikation

Per API-Read-Back bestätigt (beide Dashboards, alle Werte korrekt). Sichtprüfung im
Browser durch Norbert: **Woche ok, Tag weiterhin ohne Labels** → Nachtrag unten.

### Nachtrag — Tages-Panel brauchte einen zweiten Fix (Balken-Slot-Breite)

Sichtprüfung ergab: Wochen-Labels da, Tages-Labels weiterhin nicht. Kontrollierte
Screenshot-Serie (Playwright gegen `/d-solo/`, `domcontentloaded` + festes Timeout,
**kein** `networkidle`/`/render/`, Health-Check nach jedem Lauf — Regeln aus Session (2)
eingehalten, kein Vorfall):

- 620 px Panel-Breite: alle 8 Tages-Labels sichtbar
- 450 px / 380 px: **keine** Labels — trotz `showValue: always`

Root Cause: Balken-Slot schmaler als Label-Text (8 Balken vs. 1–2 beim Wochenpanel), siehe
neue Stolperfallen-Warnung oben. Fix auf beiden Tages-Panels: `decimals: 1` +
`text.valueSize: 10` (AZ v76, Wohnzimmer v29). Per Screenshot bei 380 px verifiziert:
alle Labels sichtbar („8.5 kWh" statt „8.52 kWh"). Sichtprüfung durch Norbert: **bestätigt**.

Auf Norberts Wunsch anschließend `text.valueSize: 10` einheitlich auf **alle** Balken-Panels
beider Dashboards gesetzt (vorher nur Tages-Panel; Rest lief auf Auto-Größe, die je nach
Platz variiert) — AZ v77, Wohnzimmer v30. Einheitliches Schriftbild = Standard für künftige
Panels.

Abschließend die **8-Balken-Analyse** über alle Panels: Tag (-8d) und Woche (-9w) halten die
getestete 8er-Grenze ein, Jahr/Kosten (-3y) sind auf Jahre unkritisch — nur das Monats-Panel
(-13mo, bis 13 Balken) hätte ab ca. Frühjahr 2027 schleichend die Labels verloren.
Entscheidung Norbert: **Variante B** — Monats-Panel auf w:16 verbreitert statt Range zu
kürzen (Saisonvergleich bleibt), Jahr rückt daneben, Kosten in Zeile 3. Zusätzlich
`Decimals: 1` auf Woche/Monat/Jahr vereinheitlicht (AZ v78, Wohnzimmer v31). Design-Standard
oben unter „Dashboard-Struktur" dokumentiert.

Zum Abschluss die **Momentanleistungs-Gauge entfernt** (Entscheidung Norbert): redundant
zur nativen HA-Gauge, bei 15m-Refresh irreführend. Layout auf 2 volle Zeilen gestrafft —
Zeile 1: Tag/Woche/Kosten, Zeile 2: Monat (w:16) + Jahr (AZ v79, Wohnzimmer v32).

Feinschliff danach: Wert-Schriftgröße einheitlich auf **12** erhöht (380-px-Test bestanden,
getestetes Maximum), Legenden-Alias **„SK"** statt des langen Flux-Seriennamens gesetzt
(Kosten-Panel ausgenommen, siehe Design-Standard), und **Jahresverbrauch mit Jahreskosten
getauscht** — Zeile 1 zeigt jetzt Verbrauch in aufsteigenden Zeiträumen (Tag/Woche/Jahr),
Kosten stehen unten neben dem Monat (AZ v84, Wohnzimmer v36).

Außerdem **Grafana in die HA-Sidebar eingebettet** (Muster wie Fhem/Node-RED): neues
HA-Dashboard `dashboard-grafana` mit iframe-Strategie auf
`http://192.168.2.214:3000/dashboards`, Icon `mdi:chart-bar`. Voraussetzungen waren schon
da (`allow_embedding` + Anonymous Access aus Session 2026-06-27 (3)). Das „Powered by
Grafana"-Branding betrifft nur `/d-solo/`-Panel-Embeds, nicht die Vollansicht. Achtung:
HTTP-iFrame lädt nur bei lokalem HTTP-Zugriff auf HA (Mixed Content bei HTTPS/Nabu Casa) —
gilt für Fhem/Node-RED-Einträge genauso.

---

## Session 2026-07-12 — „BW-WP Heizungskeller wird nicht aktualisiert" — Diagnose: Zähler-Reset, kein Datenpfad-Problem

### Symptom (Norbert)

Eindruck, dass das Grafana-Dashboard „BW-WP Heizungskeller" (`bwwp-heizung`) bzw. InfluxDB
nicht mehr aktualisiert wird.

### Diagnose (read-only: HA-History via ha-mcp + Grafana `/api/ds/query`)

1. **HA-Sensoren laufen:** `sensor.0xa085e3fffeb7c870_power` liefert live (~640 W),
   `_energy` tickt alle ~10 Min. selbstständig (Self-Reporting, Poll-Flow bleibt deaktiviert).
2. **InfluxDB bekommt frische Punkte** für beide Entitäten (per Grafana-Datasource-Query
   verifiziert) — Datenpfad HA → InfluxDB → Grafana vollständig intakt.
3. **Root Cause:** Das Shelly-Firmware-Update am 2026-07-12 (siehe [[victron_node_red]],
   „Fix umgesetzt 2026-07-12") hat den kWh-Zähler zurückgesetzt: 11:03 Uhr 7,41 kWh →
   `unavailable` → 13:13 Uhr 0, zählt seither neu. Die Panel-Query verwirft dadurch den
   heutigen Tag (`difference(nonNegative: true)` auf Tages-Endstände: 0,5 − 5,88 < 0 → None).
   Nachgerechnet per exakter Panel-Query: 10.07. = 2,61 | 11.07. = 2,82 | 12.07. = **kein Wert**.

### Konsequenzen

- **Ab 13.07. heilt sich das Dashboard von selbst** — nur der 12.07. bleibt ohne Balken.
- ~1,53 kWh vom 12.07.-Vormittag (5,88 → 7,41) sind aus Panel-Sicht verloren.
- Stolperfalle inkl. reset-festem Query-Muster oben dokumentiert („Zähler-Reset reißt ein
  Tagesloch").

### Offen (Entscheidung Norbert)

- [ ] Panels auf das reset-feste Muster umbauen (BW-WP: provisionierte JSON auf dem LXC;
      SK-Dashboards per UI/API) — **oder** bewusst bei Selbstheilung belassen und nur die
      dokumentierte Stolperfalle kennen.

---

## Session 2026-08-10 — „Tageswerte verloren" bei SK Wohnzimmer: kein Datenverlust, Panel-Darstellung gefixt

### Symptom (Norbert)

Im Panel „Täglicher Energiebedarf SK Wohnzimmer" fehlten Tagesbalken — Verdacht auf verlorene Daten.

### Diagnose (read-only, zwei unabhängige Quellen gegeneinander)

HA-Langzeitstatistik (`ha_get_history` source=statistics, period=day, `change`) gegen die echten
InfluxDB-Tageswerte (Grafana `/api/ds/query`, `aggregateWindow(1d, last, createEmpty: true)`;
zusätzlich `fn: count` je Tag, um Punkte- von Wertproblemen zu trennen):

1. **Jeder Tag mit Verbrauch existiert in beiden Quellen und ist wertgleich** (bis 2. Nachkommastelle).
2. Die fehlenden Balken sind **echte Null-Tage**: 04.–05.07., 20.–24.07., 31.07.–08.08.
   Zählerstand stand vom 30.07. bis 09.08. 01:37 unverändert auf 131,45 kWh,
   `switch.0xa085e3fffebd16b8` war durchgehend `on` → Steckdose lief, Last zog nichts.
3. **Root Cause der Darstellung:** `createEmpty: false` — Zigbee-Steckdosen schreiben nur bei
   Wertänderung nach InfluxDB, an Null-Tagen existiert **kein Datenpunkt**, das Tagesfenster
   entfällt komplett statt einen 0-Balken zu liefern. Verstärkt durch hart kodiertes
   `range(start: -8d)` (ignoriert den Zeit-Picker; Dashboard stand auf `now-6h`).
4. **Kein Bug, aber merken:** Vor dem **01.07.2026** liegen für `0xa085e3fffebd16b8_energy` gar
   keine Influx-Daten (Aufzeichnungsbeginn — davor nicht im `include.entities`-Filter).
   Juni-Balken in den Monats-/Jahres-Panels sind deshalb unvollständig.

### Grundsatzentscheidung: Grafana zeigt ab der ersten InfluxDB-Messung — Backfill verworfen

Norbert, 2026-08-10: **„Grafana soll nur das anzeigen ab dem Tag, wo die erste Messung in InfluxDB
erfolgte. Alles andere ist Vorgeschichte."** Damit ist der zwischenzeitlich erwogene Backfill aus
der HA-Langzeitstatistik **vom Tisch** — nicht vergessen, sondern bewusst verworfen.

Erste Influx-Punkte (per `|> first()` ermittelt):
`0xa085e3fffebc4574_energy` (AZ+Küche) **26.06.2026 23:35** = 179,86 kWh ·
`0xa085e3fffebd16b8_energy` (Wohnzimmer) **01.07.2026 14:35** = 95,85 kWh

Unter dieser Definition **stimmen die Panels** — geprüft gegen die Zählerstandsdifferenz seit dem
ersten Punkt:

| | seit erster Messung real | Jahres-Panel | Rest |
|---|---|---|---|
| Wohnzimmer | 42,60 kWh | 40,90 kWh | 1,70 |
| AZ und Küche | 106,90 kWh | 106,70 kWh | 0,20 |

Der Rest ist der **erste erfasste Tag**: Die erste Messung ist Bezugspunkt, kein Verbrauchswert,
`difference()` verwirft sie zwangsläufig. Die Größe hängt nur an der Uhrzeit der Ersterfassung
(WZ 14:35 → 1,70 kWh verloren; AZ 23:35 → 0,20 kWh). Rettbar nur über eine `union`/`join`-Konstruktion,
die für den ersten Tag zusätzlich den Tagesanfangswert heranzieht — **bewusst nicht gemacht**,
unverhältnismäßig für einen einmaligen Betrag.

> [!note] Erster Balken jeder Reihe ist ein Teilzeitraum
> Wochen-Panel WZ: KW27 = nur 02.–05.07. (0,12 kWh) · Monats-Panel AZ: Juni = nur ab 26.06.
> (26,84 kWh) · Jahres-Panel: 2026 beginnt mit der Inbetriebnahme, nicht am 01.01.
> Beim Vergleichen beachten; ab 2027 erledigt sich das für das Jahres-Panel von selbst.
> Zur Einordnung, falls die Frage „warum zeigt HA mehr?" wieder aufkommt: HA-Statistik und
> InfluxDB sind **zwei getrennte Pipelines mit verschiedenem Startdatum** — HA ab Sensor-Anlage
> (AZ April 2026, WZ Juni 2026), InfluxDB ab `include.entities`-Eintrag. HA-Jahreswerte 2026 zum
> Vergleich: WZ 83,96 kWh · AZ 261,01 kWh (enthalten die Vorgeschichte, Grafana bewusst nicht).

### Fix (umgesetzt, Tages-Panels beider SK-Dashboards)

```flux
  |> range(start: -31d)           // RECHENfenster, nicht Anzeigefenster (Begründung unten)
  ...
  |> aggregateWindow(every: 1d, fn: last, createEmpty: true,
       location: timezone.location(name: "Europe/Berlin"), timeSrc: "_start")
  |> fill(usePrevious: true)      // ohne diese Zeile steigt difference() an null-Fenstern aus
  |> difference(nonNegative: true)
  |> tail(n: 7)                   // ANZEIGEfenster: die letzten 7 Tage = eine Woche
```

`createEmpty: true` allein reicht **nicht** — es erzeugt `null`-Fenster, an denen `difference()`
wieder aussteigt. Erst `fill(usePrevious: true)` schreibt den Vortagesstand fort → Differenz 0.

> [!important] Anzeigefenster ≠ Rechenfenster — `range()` NICHT zum Begrenzen der Balken benutzen
> Naheliegend wäre, die 7 Tage über `range(start: -8d)` einzustellen (altes Muster, „+1 Puffer
> für `difference()`" aus [[homelab-ansible]]). Zusammen mit `fill(usePrevious: true)` ist das
> **falsch**: `fill()` kann nur auf Werte *innerhalb* des Ranges zurückgreifen. Fällt der
> Fensteranfang in eine längere Null-Phase (kein einziger Datenpunkt), gibt es nichts
> fortzuschreiben → die Tage bleiben leer. Live gemessen am 10.08.: `-8d` lieferte nur
> **4 statt 7 Balken**. Deshalb großzügiges Rechenfenster (`-31d`) + `tail(n: 7)` für die Anzeige.
> Norberts Vorgabe (2026-08-10): **Tages-Panel zeigt immer 7 Tage — eine Woche hat 7 Tage.**
> Deckt sich mit dem Design-Standard „max. 8 Balken bei w:8" (ein Zwischenstand mit 30 Balken
> war unbrauchbar, Wert-Labels verschwinden).

Dank `showValue: "always"` + `tooltip.hideZeros: false` (waren schon gesetzt) steht an Null-Tagen
lesbar `0.0 kWh` auf der Grundlinie. Damit unterscheidet das Panel jetzt zwei Zustände, die vorher
gleich aussahen: **Balken auf 0** = gemessen, nichts verbraucht · **kein Balken** = keine Daten.

**Wochen-Panel (Panel 3) nachgezogen** — gleiches Muster, andere Zahlen: `createEmpty: true`
**nur in der ersten Stufe** (Tages-`aggregateWindow`) + `fill(usePrevious: true)`, Rechenfenster
`-9w` → `-16w`, `tail(n: 8)` für die Anzeige. Die zweite Stufe (`every: 1w, fn: sum`) bleibt auf
`createEmpty: false` — sie bekommt durch die gefüllte erste Stufe ohnehin für jeden Tag einen Wert.
Die angezeigten Werte ändern sich dadurch **nicht** (vorher/nachher identisch geprüft); der Effekt
greift erst, wenn eine Woche *durchgehend* null Verbrauch hat — die würde vorher komplett fehlen.
Versionen: `adthnnt` → 39 (7 Balken, KW27–KW33), `adxk68k` → 87 (8 Balken, KW26–KW33).
Wohnzimmer zeigt nur 7 Wochen, weil die Aufzeichnung erst am 01.07. beginnt (wächst von selbst auf 8).

- `SK Wohnzimmer` (`/d/adthnnt`) Panel 2 → **Version 38** (37 = Zwischenstand mit 30 Balken)
  Endstand: 04.–08.08. je `0.00` · 09.08. `3.40` · 10.08. `3.60`
- `SK AZ und Küche` (`/d/adxk68k`) Panel 2 → **Version 86** (85 = Zwischenstand)
  Endstand: 04.–08.08. 0,10–0,40 · 09.08. `8.11` · 10.08. `6.52` — dort gab es nie Lücken,
  das Panel ist jetzt aber gegen denselben Effekt abgesichert
- Panels 3–6 beider Dashboards **unverändert** (per JSON-Vergleich verifiziert); Rollback über
  Grafana → Dashboard versions oder die Backups im Session-Scratchpad.

### Zugang

Grafana-Admin-Login existiert (User `admin`, angelegt 25.06.2026, einziges Konto) — **Passwort
liegt in Norberts Passwort-Manager** unter `192.168.2.214:3000`, absichtlich nicht hier notiert.
Der Grafana-Default `admin:admin` ist geändert. Damit ist der Schreibweg per
`POST /api/dashboards/db` (mit `folderUid` aus `meta`, sonst wandert das Dashboard nach „General")
verfügbar — Provisioning wie bei BW-WP ist für UI-Dashboards **nicht** nötig und hätte sie in die
Dateiverwaltung gezwungen.

### kWh-Preise auf die exakten HA-Werte umgestellt (alle drei Dashboards)

Die Preis-Variablen standen auf gerundet `0.35` / `0.075`. Maßgeblich sind die im **HA-Energie-Dashboard**
hinterlegten Werte (`.storage/energy` → `energy_sources[grid]`, per `ha_manage_energy_prefs` mode=get):
**`number_energy_price: 0.3494`** (Bezug) und **`number_energy_price_export: 0.07382`**
(Einspeisevergütung = Opportunitätskosten für „mit PV"). Beide Variablen jetzt exakt gesetzt:

| Dashboard | Weg | Version | Kosten 2026 ohne PV / mit PV |
|---|---|---|---|
| SK Wohnzimmer `adthnnt` | API | 40 | 14,29 € / 3,02 € |
| SK AZ und Küche `adxk68k` | API | 88 | 37,28 € / 7,88 € |
| BW-WP Heizungskeller `bwwp-heizung` | **Datei** (provisioniert!) | — | 20,78 € / 4,39 € |

> [!important] Zwei Fallen beim Ändern von Preis-Variablen
> 1. **Textbox-Variablen brauchen beides**: `query` (= Default) UND `current.text`/`current.value`.
>    Nur `current` zu setzen wirkt sofort, springt aber beim nächsten Laden auf den alten Default zurück.
> 2. **BW-WP ist provisioniert** — Änderung MUSS in `/var/lib/grafana/dashboards/bwwp-heizungskeller.json`
>    auf dem LXC erfolgen, nicht per API, sonst überschreibt der Provisioner sie beim nächsten Scan.
>    Re-Import dauerte gemessen ~20 s (Provider-Intervall 60 s). Backup liegt daneben als `.bak-<stamp>`.

Interpretation der beiden Kostenbalken (gilt unverändert): Sie sind **Ober- und Untergrenze**, nicht der
tatsächlich angefallene Betrag — „ohne PV" unterstellt 100 % Netzbezug, „mit PV" 100 % Eigenstrom.
Der reale Wert liegt dazwischen; bei den Klimaanlagen im Sommer nahe am unteren Balken.

### Bewusst NICHT geändert: Monats-, Jahres- und Kosten-Panel (4, 5, 6)

Entscheidung Norbert (2026-08-10): vorerst so lassen. Sie tragen dasselbe `createEmpty: false`-Muster,
aber das Risiko ist gering — es müsste ein **ganzer Monat bzw. ganzes Jahr ohne jeden Verbrauch**
vergehen, damit ein Balken verschwindet. Falls es doch mal auftritt (z. B. Klimaanlage einen
kompletten Winter aus), ist das Rezept identisch zum Wochen-Panel:

| Panel | Rechenfenster (`range`) | Anzeige (`tail`) |
|---|---|---|
| 4 Monat | `-13mo` → großzügiger, z. B. `-24mo` | `tail(n: 13)` |
| 5 Jahr | `-3y` → z. B. `-6y` | `tail(n: 3)` |
| 6 Kosten | wie Panel 5, **beide** Targets (ohne PV / mit PV) einzeln | `tail(n: 3)` |

Jeweils: `createEmpty: true` **nur in der ersten Stufe** (Tages-`aggregateWindow`) +
`fill(usePrevious: true)` vor `difference()`, zweite Stufe unverändert lassen.
Bei Panel 6 daran denken, dass es **zwei** Targets hat und der `map()`/`rename()`-Teil hinter der
Aggregation steht — `tail()` gehört davor, sonst greifen die Rot/Grün-Overrides an den
Seriennamen nicht mehr. Vorher immer mit dem Vorher/Nachher-Vergleich testen
(Balkenzahl darf nicht sinken), Muster siehe `patch3.py` im Session-Scratchpad.

> [!warning] Design-Standard beachten (max. 8 Balken bei w:8)
> Bei jeder Änderung an `range`/`tail` gegenprüfen, wie viele Balken herauskommen — beim ersten
> Anlauf am 10.08. erzeugte ein `range(start: -31d)` **ohne** `tail()` 30 Balken im Tages-Panel,
> wodurch Grafana die Wert-Labels unterdrückte und das Panel unbrauchbar wurde.
> Ausnahme bleibt das Monats-Panel mit w:16 (bis 13 Balken).

### Kontrollfrage am Ende: „Im Monats-Panel SK Wohnzimmer fehlt Juni — korrekt?"

**Ja, korrekt.** Erster Influx-Punkt WZ = 01.07. 14:35, für Juni existiert kein einziger Wert →
kein Balken. Auch der Tages-Fix würde daran nichts ändern (`fill()` hat vor dem ersten Punkt nichts
fortzuschreiben; es entstünde ein leeres Fenster, kein Null-Balken). Ein `0,0`-Juni-Balken wäre sogar
eine **Falschaussage** — real waren es 41,26 kWh, sie liegen nur in der bewusst ausgeblendeten
Vorgeschichte. Zum Vergleich AZ+Küche: dort **gibt** es einen Juni-Balken (26,84 kWh), aber nur für
die letzten viereinhalb Junitage ab dem 26.06. — ebenfalls kein Monatswert.

Monats-Panel WZ gegen HA-Statistik geprüft: Juli 33,90 (HA ab erster Messung 35,60 → **−1,70**,
der Starttag-Effekt) · August 7,00 = **exakt deckungsgleich** mit HA. Sobald der Juli aus dem
13-Monats-Fenster gewandert ist, stimmt das Panel durchgehend.

### Endstand 2026-08-10

| Dashboard | Panel 2 Tag | Panel 3 Woche | Preise | Panels 4–6 |
|---|---|---|---|---|
| SK Wohnzimmer `adthnnt` | v38 · 7 Balken | v39 · 7 Balken | v40 | Query unverändert |
| SK AZ und Küche `adxk68k` | v86 · 7 Balken | v87 · 8 Balken | v88 | Query unverändert |
| BW-WP `bwwp-heizung` | — | — | Datei | — |

Alle Änderungen wurden nach dem Schreiben gegen InfluxDB **nachgerechnet** (gespeicherte Query per
`/api/ds/query` ausgeführt) und die nicht angefassten Panels per JSON-Vergleich als unverändert
belegt. Rollback: Grafana → Dashboard settings → Versions (die Vorgängerversionen der Tabelle),
beim BW-WP über `.bak-<stamp>` neben der provisionierten Datei. Die JSON-Backups aus der Session
lagen nur im temporären Scratchpad — **Grafanas Versionshistorie ist der dauerhafte Rollback-Weg**.

**Methodik, die sich bewährt hat** (für künftige „Grafana zeigt falsche Werte"-Fragen):
HA-Langzeitstatistik (`ha_get_history` source=statistics) und InfluxDB (Grafana `/api/ds/query`,
anonym lesbar) sind zwei **unabhängige** Quellen über denselben Sensor — sie gegeneinander zu stellen
trennt sofort „Daten fehlen" von „Darstellung unterschlägt sie". Zusatzgriffe: `fn: count` je Fenster
zeigt, ob überhaupt Punkte da sind; `|> first()` liefert den Aufzeichnungsbeginn einer Entität.

---

## Brauchwasser-WP: Messstelle, Datenpfad, Zigbee-Störungen

Übernommen aus [[victron_node_red]] am 23.08.2026 — die Blöcke sind dort über das
Energie-Dashboard entstanden, gehören fachlich aber zu Monitoring und Zigbee.
Die Wärme-Auswertung (Stillstandsverlust, Zirkulation) liegt in [[Brauchwasser-Wärmepumpe]].

## BW-WP → InfluxDB + Grafana (2026-07-09) ✅
- **Ziel:** BW-WP-Monitoring analog „SK AZ und Küche" (Grafana-Dashboard mit Tages-/Wochen-/Monats-/Jahres-Energie + Jahreskosten).
- **Architektur-Aufklärung (wichtig!):** Die HA-InfluxDB-Anbindung ist zweigeteilt — **Verbindung als UI-Config-Entry** (Settings → Integrationen, Entry `01KW2XFJ2JVCFF0178139DZRYY`: InfluxDB2 `http://192.168.2.119:8086`, org `ng`, bucket `homeassistant`, Token) und **Include-Filter in der YAML** (`influxdb:`-Block, nur `include.entities`). Die YAML sieht dadurch „unvollständig" aus — ist sie nicht. (Der `influxdb_token` in secrets.yaml ist ein unreferenziertes Relikt; Debug-Forensik-Sackgassen: `ha core logs` ist kopfrotiert, SSH-Add-on ist NICHT host_network → localhost-Tests dort wertlos.)
- **HA:** `sensor.0xa085e3fffeb7c870_energy|_power` in die Include-Liste ergänzt (Backup `/config/configuration.yaml.claude-bak-20260709`), `ha core check` ok, Config-Entry-Reload statt Neustart — **verifiziert**: Energy-Punkt kam nach `zigbee2mqtt/.../get`-Anstoß in InfluxDB an. Power-Punkte folgen bei nächster Laständerung (WP war gerade auf 0 W gefallen).
- **Grafana** (`192.168.2.214:3000`, v13.1): Dashboard **„BW-WP Heizungskeller"** (`/d/bwwp-heizung`, Ordner „Home") = 1:1-Klon von „SK AZ und Küche" (uid `adxk68k`), nur Entität `bc4574→b7c870`, displayName „BW-WP", inkl. Preis-Variablen (0,35/0,075 €/kWh). Anonym-Zugang ist nur Viewer → Deploy **per Provisioning** (root-SSH): Provider `/etc/grafana/provisioning/dashboards/home-dashboards.yaml` → `/var/lib/grafana/dashboards/bwwp-heizungskeller.json`, `allowUiUpdates: true` (UI-Edits möglich; bei Datei-Änderung re-importiert Grafana). Grafana-Neustart durchgeführt.
- **Caveat:** Panels zeigen sinnvolle Balken erst, wenn Tageshistorie aufläuft (Steckdose misst erst seit kurzem, Zähler 0,08 kWh).

## BrauchwasserWP-Dose schaltet sich selbst aus (2026-07-12) — Diagnose abgeschlossen, Fix offen

**Symptom (Norbert):** Shelly 1PM Gen 4 „BrauchwasserWP" (S4SW-001P16EU, Zigbee via Z2M) schaltet sich von alleine aus.

**Befunde (HA-Historie 7 Tage + Z2M-Probe via Node-RED-Admin-API):**
- Spontane OFFs: 11.07. 04:07 und 11.07. 22:10 — **beide im Standby (2 W)**, nie unter Last → Überstrom-/Übertemperatur-Schutz ausgeschlossen.
- 22:10-Off gefolgt von **Reboot 22:15** (unavailable → unknown → meldet OFF). Weitere Instabilität: 10.07. 06:24 5-min-Ausfall; 08.–09.07. >1 Tag offline mit Energie-Zähler-Reset (dokumentiert). Das Gerät crasht/rebootet also wiederholt.
- Keine Fremdsteuerung: keine Automation/Skript/Szene referenziert das Gerät, keine Shelly-WiFi-Integration in HA, Node-RED pollt nur lesend (`/get energy`). LQI 204 = Funk gut.
- **Mechanismus:** Zigbee-Attribut `startUpOnOff` = UNSUPPORTED (raw-read verifiziert) → Power-on-Verhalten steckt in der Shelly-RPC-Config (`initial_state`, Default **`match_input`**). SW1 ist unbeschaltet → nach jedem Firmware-Reboot initialisiert das Relais **AUS**. „Schaltet sich aus" = „ist gecrasht und neu gestartet".
- **Verdächtiger Crash-Treiber:** WLAN auf der Dose **aktiviert** (SSID „nob"), aber dauerhaft `disconnected` → Dauer-Reconnect auf dem geteilten Funk-SoC (bekannte Gen4-Instabilitätsquelle). Firmware-Stand unbekannt (`installed_version: -1`); **Zigbee-OTA unmöglich** — Gerät hat keinen OTA-Cluster, Update geht NUR über WLAN (Web-UI/App).

**Empfohlener Fix (offen):** (1) Dose ins WLAN bringen → Firmware-Update (Gen4-Zigbee-Stabilitätsfixes) + in der Web-UI `initial_state` auf `restore_last`/`on` stellen (eigentlicher Fix: Relais bleibt nach Reboot AN); (2) danach WLAN auf der Dose deaktivieren (Z2M `wifi_config.enabled=false`); (3) optional Sofort-Workaround: HA-Automation „switch off → wieder einschalten" (Dose ist reine Messdose); (4) falls weiter instabil: Gerät tauschen (SK-Dosen gleichen Modells laufen stabil).

**Tripwire aktiv:** Temporärer Probe-Flow „TEMP Shelly Probe" (Live-ID `b40b73029cf7915c`, kein Repo-File) sammelt Z2M-States/Logs/Actions der Dose in Node-RED-Globals `probe_bwwp_*` — fängt beim nächsten Off-Ereignis, ob ein `action`-Event (SW1-Eingang) oder ein Reboot vorausgeht. Entfernen: `curl -X DELETE http://192.168.2.80:1880/flow/b40b73029cf7915c`.

### Fix umgesetzt (2026-07-12 nachmittags) ✅
- **Firmware-Update durch Norbert** (via WLAN): jetzt `2.0.0-beta3` (Build 20260701, App `S1PMG4ZB` = **Zigbee-Track**; das von `Shelly.CheckForUpdate` angebotene „stable 1.7.5" ist der WiFi/Matter-Track — **kein Downgrade!**). Dose hat IP `192.168.2.223` (SSID „nob"). ⚠️ Update hat den **Energie-Zähler auf 0 zurückgesetzt** (war 6,6 kWh) — HA verbucht es als sauberen total_increasing-Reset, kein Phantom.
- **`initial_state` stand auch nach dem Update noch auf `match_input`** → per RPC (`Switch.SetConfig`) auf **`restore_last`** gesetzt + **`in_mode: detached`** (Relais vom unbeschalteten SW1 entkoppelt). Verifiziert per GetConfig; kein Neustart nötig. Damit bleibt das Relais bei künftigen Reboots AN — Symptom behoben unabhängig von der Crash-Frage.
- **Energie-Self-Reporting funktioniert jetzt** (Beweis: Poll-Flow deaktiviert, 15-min-Fenster, Zähler sprang 12:58 UTC selbstständig 0→0,1 kWh bei ~595 W) → **Poll-Flow „Zigbee BW-WP Energie-Poll" (`c0cf881d9a69a14b`) dauerhaft DEAKTIVIERT** (nicht gelöscht — Rollback: `disabled:false` via Admin-API; Repo-File `flows/zigbee-bwwp-energie-poll.json` bleibt als Referenz).
- Reporting-Konfiguration (inkl. seMetering) hat das Update überlebt. Interne Gerätetemperatur bei 595 W Last: 49,3 °C (unkritisch).
- ~~**Beobachtungswoche bis ~19.07.**~~ ✅ **ABGESCHLOSSEN 2026-07-18** (siehe unten „Beobachtungswoche abgeschlossen").
### Beobachtungswoche abgeschlossen (2026-07-18) ✅ — Fall BW-WP-Dose ZU
- **Tripwire-Endauswertung (12.–18.07.): Dose blieb ruhig.** Keine Reboots, keine spontanen OFFs, kein Solo-unavailable. Alle Fehler-Logs fremdverursacht: 15.07. = Elektroarbeiten/Coordinator (eigene Sektion), 16.07. ~12:24 kurzer Z2M-Reconnect (`ECONNRESET`, Bridge-seitig, alle Geräte), dazu der bekannte harmlose get-power-Timeout 15.07. Energie-Self-Reporting lief die ganze Woche (Zähler am 18.07. bei ~20 kWh).
- **WLAN deaktiviert (2026-07-18 ~15:58):** per Z2M-Set `{"wifi_config":{"enabled":false}}` auf `zigbee2mqtt/BrauchwasserWP/set` (temporärer Admin-API-Flow, danach gelöscht). Verifiziert dreifach: Dose bestätigt `enabled: false` im State, IP 192.168.2.223 nicht mehr pingbar, Zigbee meldet danach frisch weiter (kein Reboot, Relais AN bei ~660 W Last, Zähler lief durch). ⚠️ RPC/Web-UI ab jetzt nicht mehr erreichbar — für künftige Config-Änderungen WLAN erst wieder per Z2M-Set aktivieren (`enabled:true` + ssid/Passwort). Hinweis: `wifi_status` im Z2M-State zeigt noch stale „got ip" (Attribut wird nur bei RPC-Poll aktualisiert) — nicht verwirren lassen.
- **Tripwire-Flow `b40b73029cf7915c` gelöscht** (DELETE verifiziert 404) + alle Probe-/Debug-Globals aufgeräumt (`probe_bwwp_*`, `z2m_*`, `poll_debug`, `probe_disc`; `probe_wzlampe*` bewusst belassen — gehört zur T1M-Lampe).
- Damit ist der Fall „BrauchwasserWP schaltet sich selbst aus" vollständig geschlossen: Ursache (Firmware-Crash + `match_input`) behoben, Woche stabil, Aufräumen erledigt.

- **Zwischenstand Beobachtungswoche (2026-07-15, Tag 3): Dose blieb ruhig ✅** — HA-Historie 12.–15.07.: keine spontanen OFFs, keine Solo-unavailable, kein Reboot; Tripwire-Logs seit dem Fix ohne Geräte-Auffälligkeit (nur ein einzelner get-power-Timeout 15.07. 05:56 — harmlos). Die unavailable-Fenster am 15.07. vormittags waren der **Z2M-Coordinator-Ausfall** (eigene Sektion 2026-07-15), betrafen alle Zigbee-Geräte und zählen nicht gegen die Dose. Plan unverändert: nach ~19.07. WLAN aus + Tripwire löschen.


## Zigbee-Vormittags-Ausfall 15.07. — Coordinator-Socket-Timeouts, NICHT die Dosen (2026-07-15, Diagnose korrigiert)

**Symptom (Norbert):** Keine Energiewerte von der Klimaanlage EG Arbeitszimmer (SK AZ, `sensor.0xa085e3fffebc4574_energy`).

**Erste (falsche) Diagnose:** Gen4-Firmware-Crash der SK-AZ-Dose (Muster wie BW-WP am 12.07.) → Firmware-Update empfohlen. **Widerlegt** beim Gegencheck für die BW-WP-Beobachtungswoche: **Alle drei Shelly-Dosen (SK AZ, SK WZ, BW-WP) gingen sekundengleich offline** (08:14:56, 11:30:56, 11:43:01, 11:57:32, 13:18:02) — zeitgleiche Ausfälle über mehrere Geräte = Infrastruktur, nie Geräte-Crash.

**Tatsächliche Ursache (dreifach verifiziert):**
- `binary_sensor.zigbee2mqtt_bridge_connection_state`: **Z2M-Bridge offline 08:15–11:30, 11:43–11:44, 11:57–13:18** — deckungsgleich mit allen Geräte-unavailable-Fenstern.
- Z2M-Log (via Tripwire-Probe `probe_bwwp_logs`): **`zh:zstack:znp: Socket error Error: read ETIMEDOUT`** um 06:14:56/09:43:01/09:57:32 UTC (= 08:14:56/11:43:01/11:57:32 lokal) → die Verbindung **Z2M → Zigbee-Coordinator** (zstack/ZNP, Socket = vermutlich netzwerk-angebundener Coordinator) brach ab.
- Victron-MQTT-Sensoren (gleicher Broker, via Node-RED) liefen lückenlos durch → Broker + HA gesund, **nur Z2M↔Coordinator** betroffen.

**Auswirkung auf SK AZ (erklärt das Symptom vollständig):** Energie-Zähler meldet nur bei 0,1-kWh-Schritten; alle Reports während der Z2M-Ausfälle gingen verloren (kein Queueing). Zähler „fror" bei 246,72 kWh ein (letzter Report 14.07. 22:29), sprang nach Bridge-Rückkehr 13:35 auf 248,09 (+1,37 — die Dose zählt intern korrekt weiter). 13:57 kam der reguläre Selbst-Report pünktlich (248,19 bei 277 W ≈ alle 22 min) → **Self-Reporting intakt, Dose fehlerfrei**. Tagesverbrauch klumpt statistisch in der 13:35-Zelle (belassen, kein Verlust), kein Zähler-Reset, keine Reparatur nötig.

**Lehre (Diagnose-Regel):** Bei Zigbee-„Gerät spinnt"-Symptomen ZUERST prüfen, ob andere Zigbee-Geräte zeitgleich unavailable waren (`bridge_connection_state`-Historie) — erst wenn das Gerät ALLEIN ausfällt, geräteseitig suchen (wie BW-WP 08.–12.07., dort war es real die Firmware).

**AUFGELÖST (gleicher Tag, Norbert):** Am 15.07. vormittags liefen **Elektroarbeiten im Haus** — der Stromkreis des Coordinators (bzw. seines Netzwerkpfads) war stromlos. NICHT das öffentliche Netz: Gridmeter zeigte durchgehend Einspeisung (bis −18,4 kW Spitze), Proxmox/HA/PV liefen ununterbrochen — nur einzelne Hauskreise waren freigeschaltet. Die Ausfallfenster (08:15–11:30, 11:43, 11:57–13:18) = Arbeitsphasen. Kein Infrastruktur-Problem, keine weitere Beobachtung nötig.

