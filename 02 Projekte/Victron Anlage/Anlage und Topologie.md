---
tags: [victron, anlage, topologie, hardware, mqtt]
status: aktiv
date: 2026-07-05
updated: 2026-08-23
---

# Anlage und Topologie

Wie die Anlage aufgebaut ist und wo die Daten herkommen. **Was hier steht, gilt.**
Wie es dazu kam, steht im [[2026-07 Journal]] / [[2026-08 Journal]].

> [!info] Anlagen-Steckbrief
> Der ausführliche Steckbrief (kWp, Gerätetypen, BMS) steht in der Bereichsnotiz
> [[Elektrotechnik & Erneuerbare Energien]] und wird dort gepflegt — hier stehen
> die Daten, die für Node-RED und die Datenerfassung gebraucht werden.

## Ziel

Gridmeter **VM-3P75CT** in Home Assistant einbinden, insbesondere fürs Energie-Dashboard.
Inzwischen deutlich darüber hinausgewachsen: Energiebilanz, Verlustrechnung,
Ersparnis-Kennzahl und Anlagenüberwachung laufen über dieselbe Node-RED-Instanz.

## Setup
- Node-RED läuft in einem **Proxmox-LXC-Container** (nicht auf dem Cerbo GX, um ihn zu entlasten).
- Victron-Nodes sprechen den Cerbo per **D-Bus über TCP** an: auf dem GX `InsecureDbusOverTcp=1`, im Container `NODE_RED_DBUS_ADDRESS=<cerbo-ip>:78`.
- Hardware: Cerbo GX, Gridmeter VM-3P75CT.
- Daneben existiert ein **Venus-MQTT-Keepalive-Setup in HA** (Switch + Automationen), das **nur bei Dashboard-Aufruf** aktiviert wird und danach automatisch stoppt — der Venus-MQTT-Pfad liefert also absichtlich nicht dauerhaft. Für kontinuierliche Daten (z.B. Energie-Dashboard) daher immer den D-Bus-über-TCP-Pfad via Node-RED nutzen, nie Venus-MQTT.

## MQTT-Broker-Landkarte (2026-07-06)
- **192.168.2.233** — Proxmox-Mosquitto (HA + Node-RED), **Auth erforderlich** (anonym: Not authorized)
- **192.168.2.29** — openWB 2.x, anonym lesbar. EV-Zähler: `openWB/chargepoint/1/get/imported` in **Wh** (÷1000!). openWB sieht auch Netzzähler (counter/14), Victron-Batterie (bat/16), PV (pv/21) → macht eigenes PV-Überschussladen. Wallbox-Hardware: 192.168.2.145.
- **192.168.2.181** — Cerbo GX Venus-MQTT (`N/c0619ab31b3b/...`), anonym lesbar, Keepalive-gebunden

## Anlagen-Stammdaten (2026-07-06, von Norbert bestätigt + OSM-verifiziert)
- **Gesamt: 25,2 kWp**, Satteldach **30°**, Ost/West. Firstachse 13,8° NNO (aus OSM-Gebäudeumriss 15,7×9 m, Friedrich-Ebert-Str. 17, 69493 Hirschberg-Großsachsen, way 151647625; Standort 49.51404, 8.65993).
- Dachflächen-Azimut (OSM-berechnet, ersetzt Norberts Schätzung −80/+110): **Ost −76°** zu Süd (Kompass 104°), **West +104°** zu Süd (Kompass 284°).
- **Fronius Symo 17.5-3-M** (AC 17,5 kW): 10,08 kWp Ost + 10,08 kWp West (2 MPP-Tracker).
- **2× SmartSolar MPPT 250/70** (DC, an Batterie): „MPPT Ost" (Instanz 278) 2,52 kWp, „MPPT West" (277) 2,52 kWp. 31-Tage-Peaks: Ost 3.098 W, West 2.828 W (per Venus-MQTT `/History/Daily/n/MaxPower` ausgelesen — Keepalive an `R/c0619ab31b3b/keepalive`, dann `N/.../solarcharger/#`).
- Pro Seite gesamt: **12,6 kWp Ost, 12,6 kWp West**.

## PV-Prognose

**Beide Dienste laufen dauerhaft parallel** — bestätigt von Norbert am 23.08.2026.

| Dienst | Rolle | Entitäten |
|---|---|---|
| **Solcast** (HACS `BJReplay/ha-solcast-solar`) | **primär** — Prognoselinie im Energie-Dashboard, deckungsgleich mit VRM | `sensor.solcast_pv_forecast_prognose_heute` / `_morgen` |
| **Forecast.Solar** (Core-Integration, 2 Einträge Ost/West) | Zweitmeinung für den **Ertrag des nächsten Tages** | min_max-Helfer `sensor.pv_prognose_heute` / `_morgen` |

Der frühere Plan, nach dem Vergleichszeitraum den „Verlierer zu entfernen", ist
damit **hinfällig** — Forecast.Solar bleibt bewusst als zweite Quelle bestehen.

⚠️ **Solcast-Azimut-Konvention: gegen den Uhrzeigersinn positiv** (0 = Nord,
Ost = −90). Hobbyist-Account: max. 2 Sites, 10 API-Abrufe/Tag.

> [!note] Ost-Horizont („Odenwald-Verdacht") — nicht bestätigt
> Die Vermutung, ein Ost-Horizont von 4–6° koste 2–5 % Ost-Tagesertrag durch
> späteren Sonnenaufgang, hat sich in der Beobachtung **nicht bestätigt**
> (Norbert, 23.08.2026). Es ist keine Erwartungskorrektur nötig.

## Begriffe / Hausgeräte
- **„SK" = Split-Klimaanlage** (nicht Stromkreis!). Zwei Stück, per Zigbee-Messsteckdose erfasst: `sensor.0xa085e3fffebc4574_power` (**SK AZ** = Arbeitszimmer/Küche) und `sensor.0xa085e3fffebd16b8_power` (**SK WZ** = Wohnzimmer). Beide als individual-Kreise in der PFCP-Karte (`energie-sys`), Icon mdi:air-conditioner. Ihre kWh-Zähler (`..._energy`) sind Kandidaten für device_consumption im Energie-Dashboard (noch nicht eingetragen).

