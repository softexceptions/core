---
tags: [victron, node-red, flows, mqtt, register]
status: aktiv
date: 2026-08-23
updated: 2026-08-23
---

# Node-RED Flows

Register aller produktiven Flows: was sie tun, welche Topics sie bedienen und
woher sie kommen. **Was hier steht, gilt** — Baugeschichte im Journal.

> [!important] Generator-Regel
> Mehrere Flows werden aus **Python-Generatoren** erzeugt, nicht von Hand gepflegt.
> Bei diesen Flows **immer den Generator ändern, neu ausführen, dann deployen** —
> nie die JSON direkt editieren, sonst ist die Änderung beim nächsten Lauf weg.

## Register

| Flow (Repo `flows/`) | Live-ID | Zweck | Generator |
|---|---|---|---|
| `victron-grid-energie-mqtt.json` | `f7e2b3c4…` | Netzbezug/-einspeisung → HA | — |
| `victron-pv-energie-mqtt.json` | `f8e3c4d5…` | Fronius + MPPT-Erträge einzeln | — |
| `victron-pv-summe-mqtt.json` | (Tab in Live) | PV-Summe, gehärtet | — |
| `victron-batterie-energie-mqtt.json` | `f9e4d5e6…` | geladen/entladen/SoC/Strom | — |
| `victron-multiplus-energie-mqtt.json` | `clmpx0tab…` | AC-Abgabe/-Aufnahme — **Referenzmuster Härtung** | `gen_multiplus_flow.py` |
| `victron-wandlungsverluste-mqtt.json` | `b8e14530…` | Verlustbilanz DC-Bus + MultiPlus | `gen_losses_flow.py` |
| `victron-speicher-wirkungsgrad-mqtt.json` | `fa26bca6…` | Entlade-η und AC-Roundtrip | `gen_eff_flow.py` |
| `victron-eigenverbrauch-wert-mqtt.json` | `577ab5ad…` | Ersparnis in Euro | `gen_savings_flow.py` |
| `victron-leistung-mqtt.json` | `clpow0tab…` | Momentanleistungen (kein Zähler) | — |
| `victron-grid-leistung-glatt.json` | (Tab in Live) | geglättete Netzleistung | — |
| `openwb-ev-energie-mqtt.json` | `clev0tab…` | E-Auto-Energie aus openWB | — |
| `victron-ladedeckel-auto.json` | `ldeck0tab…` | SoC-gesteuerte Fronius-Ladehilfe | — |
| `cerbo-waechter-pushover.json` | `cerbowatch…` | Reboot-/Ausfallmeldung → [[Cerbo Wächter]] | `gen_cerbo_flow.py` |
| `cerbo-lokal-ess-setpoint-soc.json` | `739faa96…` | ESS-Setpoint **auf dem Cerbo selbst** | `gen_cerbo_ess_flow.py` |
| `batrium-watchmon.json` | `703f4458…` | BMS-Daten per UDP | — |
| `batrium-schuetz-pushover.json` | (kein Tab) | Schütz-Alarm → Pushover | — |
| `wassersensor-leck-pushover.json` | (kein Tab) | Leckalarm | — |
| `zigbee-bwwp-energie-poll.json` | `c0cf881d…` | **deaktiviert** — Poll wurde nach Firmware-Update überflüssig | — |
| `mqtt-keepalive-control.json` | `1cdee4f3…` | Venus-MQTT-Keepalive (nur bei Dashboard-Aufruf) | — |
| `udp-akku-sniffer.json` | `92509369…` | Diagnose-Sniffer | — |
| `altbestand-flow-1-4.json` | div. | Altbestand, Flow 4 deaktiviert | — |

## Topic-Konventionen

- **Nutzdaten:** `victron/<bereich>/<größe>` — z. B. `victron/grid/energy_forward`,
  `victron/pv/mppt1/yield`, `victron/battery/soc`, `victron/multiplus/ac_out_energy`
- **Interner State (retained, mit `_`):** `victron/pv/_total_state`,
  `victron/multiplus/_acout_state`, `victron/losses/_acc`, `victron/battery/_eff_acc`
  → Diese Topics tragen den Anker der Monotonie-Wächter, siehe [[Zähler und Statistik]].
  **Nicht löschen** — ohne Anker übernimmt der Wächter den nächsten Wert ungefiltert.
- **HA-Anbindung** ausschließlich über MQTT Discovery (`homeassistant/.../config`, retained),
  nie über die Companion-/WebSocket-Integration.

## Betrieb

```
systemctl restart nodered          # im LXC, nicht auf dem Cerbo
curl -X POST http://192.168.2.80:1880/inject/<inject-id>
curl -X PUT  http://192.168.2.80:1880/flow/<tab-id> --data-binary @deploy.json
```

Die Node-RED-Admin-API auf `192.168.2.80:1880` ist **ohne Auth** erreichbar und
deployt selbst — praktisch für Automatisierung, aber auch die Stelle, an der ein
versehentlicher PUT einen Flow überschreibt.

> [!warning] Neu angelegte `mqtt-out`-Nodes
> Nach einem Deploy per Admin-API publiziert ein frisch erzeugter `mqtt-out`-Node
> beim ersten Mal manchmal nicht (Inject meldet HTTP 200, am Broker kommt nichts).
> Erst ein weiterer Deploy hängt ihn an die Broker-Verbindung. **Erstes Publish
> immer per Mitlese-Probe verifizieren.**

Verwandt: [[Zähler und Statistik]] · [[Anlage und Topologie]] · [[Wartung und Betrieb]]
