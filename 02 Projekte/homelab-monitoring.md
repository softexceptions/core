---
tags: [projekt/aktiv, homelab, monitoring, grafana, influxdb, homeassistant]
status: aktiv
date: 2026-07-01
updated: 2026-07-01
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
                                   Grafana  (LXC 128, 192.168.2.92)
```

## Verbindungen

| Dienst | URL | Details |
|---|---|---|
| InfluxDB | `http://192.168.2.119:8086` | Org: `ng`, Bucket: `homeassistant` |
| Grafana | `http://192.168.2.92:3000` | Login: admin |

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
```

Nach Änderung: HA → Integrationen → InfluxDB → `...` → **Neu laden** (kein HA-Neustart nötig).
HA schreibt nur bei State Changes — zum Testen Steckdose kurz ein-/ausschalten.

---

## Überwachte Steckdosen

| Raum | Gerät | entity_id Prefix | Grafana Dashboard |
|---|---|---|---|
| EG Arbeitszimmer/Küche | Klimaanlage | `0xa085e3fffebc4574` | `eg-arbeitszimmer` |
| EG Wohnzimmer | Klimaanlage | `0xa085e3fffebd16b8` | `eg-wohnzimmer` |

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

Jedes Raum-Dashboard hat 5 Panels:

| Panel | Typ | Einheit |
|---|---|---|
| Momentanleistung | Gauge | W |
| Tagesverbrauch | Bar chart | kWh |
| Wochenverbrauch | Bar chart | kWh |
| Monatsverbrauch | Bar chart | kWh |
| Jahresverbrauch | Bar chart | kWh |

**Einheit setzen:** Standard options → Unit → `watt` bzw. `kwh`

### X-Achsen-Labels (Grafana Transformation)

Transform tab → Add transformation → Convert field type → Time → String → Timezone: Europe/Berlin

| Panel | Date format | Ergebnis |
|---|---|---|
| Wochenverbrauch | `[KW]WW` | KW26, KW27 |
| Monatsverbrauch | `MMM YYYY` | Jul 2026 |
| Jahresverbrauch | `YYYY` | 2026 |

> [!note] Moment.js Format
> Grafana verwendet moment.js — Literale mit eckigen Klammern escapen: `[KW]` → „KW".
> Single quotes funktionieren nicht (das ist date-fns Syntax).

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

---

## Neue Steckdose hinzufügen

1. `configuration.yaml` um energy + power entity ergänzen
2. HA → Integrationen → InfluxDB → Neu laden
3. Steckdose kurz ein-/ausschalten → ersten InfluxDB-Eintrag triggern
4. Grafana-Dashboard duplizieren (Settings → Save As)
5. In allen 5 Panels den entity_id-Prefix ersetzen
