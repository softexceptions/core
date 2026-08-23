---
tags: [victron, zaehler, statistik, homeassistant, haertung]
status: aktiv
date: 2026-08-18
updated: 2026-08-23
---

# Zähler und Statistik

Wie die Energiezähler gegen Ausfälle gehärtet sind und was in der
HA-Langzeitstatistik geht und was nicht. **Was hier steht, gilt.**

## Das Grundproblem: `total_increasing`

Home Assistant deutet bei `state_class: total_increasing` **jeden Rückwärtssprung
als Zählerwechsel** und bucht die Differenz als neuen Zuwachs. Ein einziger
falscher Wert erzeugt damit einen Phantom-Zuwachs in Höhe des Zählerstands —
so entstanden die 6,96 MWh E-Auto-Energie ohne einen einzigen Ladevorgang.

> [!important] Lücke vs. Lüge
> `total_increasing` reagiert nur auf Werte, die es **sieht**.
> - **Meldelücke** (Quelle schweigt) → harmlos
> - **Falscher Wert** (Quelle sendet aktiv eine 0) → Phantom-Zuwachs
>
> Daraus folgt die Schutzbedürftigkeit — sie hängt an der **Bezugsart**, nicht
> am Sensor.

| Bezugsart | Verhalten bei Ausfall | Schutz nötig? |
|---|---|---|
| `victron-input-*` (D-Bus) | Node sendet **gar nichts** | nein |
| Fremder MQTT-Publisher (openWB, Zigbee) | sendet beim Hochfahren aktiv **0** | **ja** |

Belegt an den zwei härtesten Ereignissen — Cerbo-Absturz 12.08. und
Switch-Ausfall 18.08.: bei Grid und Batterie **kein einziger Sprung**.

## Das Härtungsmuster (Referenz: MultiPlus-Flow)

Ein naiver Monotonie-Wächter (`if (v < last) return null;`) ist **beides falsch**:

- **ohne Neustart:** nach einem echten Zähler-Reset friert der Sensor **dauerhaft** ein
- **mit Neustart:** der Anker lag nur im Node-Kontext → ist weg → der niedrige Wert
  wird übernommen → jetzt bucht HA den Reset als Phantom-Zuwachs

Das korrekte Muster hat vier Teile:

1. **Offset-Überbrückung statt Blockade** — bei einem echten Reset wird ein Offset
   gegen den zuletzt publizierten Wert gebildet, die Ausgabe läuft stetig weiter
2. **Bestätigungsfristen** — 2 min bei Sprung > 1 kWh, 30 min bei kleinem Dip;
   verhindert, dass ein einzelner Glitch schon als Reset gilt
3. **State-Persistenz über ein retained Topic** (`victron/…/_…_state`) plus
   `arm`-Inject nach 15 s — überlebt den Node-RED-Neustart
4. **Fachliche Untergrenze `v <= 0`** — nicht `Number.isFinite`, denn
   `Number("") === 0`: ein leerer MQTT-Payload ist „endlich" und rutscht durch

> [!warning] Restrisiko
> Geht das retained State-Topic verloren (Broker neu aufgesetzt, manuell gelöscht),
> startet der Wächter **ohne Anker** und übernimmt den ersten Wert ungefiltert.
> Bewusst akzeptiert — die Alternative wäre ein Sensor, der nach jedem Broker-Umzug
> von Hand angestoßen werden muss.

## Stand der Zähler-Flows

| Flow / Sensor | Schutz | Status |
|---|---|---|
| MultiPlus AC-Abgabe + AC-Aufnahme | Offset-Reset-Erkennung, Fristen, State retained | ✅ **Referenzmuster** |
| PV Energie gesamt | dito (18.08. umgebaut) | ✅ |
| openWB EV | Wächter mit Reset-Erkennung | ✅ |
| Fronius-Guard | `!isFinite \|\| v <= 0` | ✅ |
| Wandlungsverluste | akkumuliert selbst, `MAXDELTA` | ✅ |
| Eigenverbrauch (EUR) | Delta-Rechnung, `MAXAGE`/`MAXSELF`/`MAXHOUSE` | ✅ |
| Grid Bezug/Einspeisung | keiner | ✅ **bewusst** — D-Bus-Bezug |
| Batterie geladen/entladen | keiner | ✅ **bewusst** — D-Bus-Bezug |

## Fallen in der HA-Statistik

- **`import_statistics` kann eine Sensor-Entität nicht rückwirkend füllen.**
  Der Recorder leitet `sum` aus dem State ab; ein Neustart hilft nicht, der Bruch
  zeigt sich erst zur nächsten vollen Stunde. Zum Aufräumen `clear_statistics`.
- **`adjust_sum_statistics` ist kein Service** — nur über WebSocket/UI erreichbar.
- **`device_class: monetary` erlaubt nur `state_class: total`**, nicht `total_increasing`.
- **HA-Energy-Kostensensoren (`…_cost`) sind keine Lebenszeitzähler** — der State
  resettet bei jedem HA-Neustart, nur `sum` läuft durch. Nie per Template addieren.
- **Abtastversatz:** Ein Node-RED-Zähler mit 300-s-Tick liefert der Stundenstatistik
  nur den Stand von `hh:55:53` → konstanter Nachlauf gegen sekundennah abgetastete
  HA-Sensoren. Nachts null, wächst mit der Leistung. Gegenprobe auf `period: "5minute"`.
- **Summen aus `total_increasing`-Quellen bilden** ist gefährlich: Fällt eine Quelle
  nachts weg, dippt die Summe → HA bucht einen Reset. Deshalb der Monotonie-Wächter
  **auf der Summe**, nicht nur auf den Einzelzählern.

Verwandt: [[Node-RED Flows]] · [[Energiebilanz und Kennzahlen]] · [[Cerbo Wächter]]
