---
tags: [bereich, elektrotechnik, erneuerbare-energien, pv, victron]
---

# Elektrotechnik & Erneuerbare Energien

Langfristiger Bereich für meine eigene Photovoltaik-/Batterie-Anlage und Hausautomation — zweites Fachgebiet neben der Softwareentwicklung, mit direkter Schnittstelle zu meiner Unterrichtstätigkeit an der [[04 Ressourcen/Heinrich-Emanuel-Merck-Schule|HEMS]] (Zentrum für Erneuerbare Energien).

## Beschreibung

Ich betreibe, überwache und erweitere eine eigene PV-/Speicher-Anlage und verbinde dabei Elektrotechnik mit Softwareentwicklung: Node-RED-Flows, MQTT und Home Assistant machen die Anlage mess-, steuer- und auswertbar. Erkenntnisse daraus fließen auch in den Unterricht (z. B. MQTT-Bridge- und openWB-Diagramme).

## Anlagen-Steckbrief (Stand 2026-07-08)

| Komponente | Daten |
|---|---|
| **PV-Wechselrichter** | Fronius Symo 17.5-3-M, AC-gekoppelt am AC-IN der MultiPlus (netzabhängig) |
| **PV-Leistung** | 25,2 kWp gesamt — Ost 12,6 kWp / West 12,6 kWp, Satteldach 30° |
| **MPPT-Laderegler** | 2× Victron SmartSolar MPPT 250/70, DC-gekoppelt an der Batterie (~5 kWp) |
| **Wechselrichter/Speicher** | 3× Victron MultiPlus-II 48/10000 (ein VE.Bus-Verbund, ~24 kW Insel gesamt) |
| **Batterie** | ~15 kWh (280 Ah/48 V), Ausbau auf ~30 kWh geplant |
| **BMS** | Batrium — 1× WatchMon Core + CellMate K9, CAN-Anbindung an Cerbo GX (DVCC) |
| **Systemmonitor** | Victron Cerbo GX (D-Bus über TCP) |
| **Automatisierung** | Node-RED im Proxmox-LXC (D-Bus-TCP zum Cerbo, unabhängig von der Anlage stromversorgt) |
| **Smart Home** | Home Assistant — Anbindung durchgängig per MQTT Discovery |
| **E-Auto-Laden** | openWB 2.x (PV-Überschussladen, eigenes MQTT) |
| **Gridmeter** | VM-3P75CT |

## Aktuelle Themen

- **Whole-Home-Backup** — Notstromfähigkeit über AC-Out der MultiPlus, Batterie-Ausbau als Voraussetzung
- **Speicher-Wirkungsgrad korrekt messen** — DC-Roundtrip vs. AC-Roundtrip sauber trennen
- **Wärmewende & Speicher-Arbitrage** — Öl vs. Wärmepumpe, dynamischer Tarif, thermische vs. batterieelektrische Speicherung
- **Fronius-Ladedeckel-Automatik** — SoC-gesteuerte AC-Ladehilfe über MaxChargeCurrent
- **Energie-Dashboards in Home Assistant** — Sankey, Power Flow Card Plus, Forecast.Solar/Solcast-Prognosen, CO2-Anteil

## Verbundene Projekte

- [[02 Projekte/victron_node_red|Victron Node-RED]] — Node-RED-Flows, HA-Integration, Anlagen-Topologie, Wartungsanleitung, alle laufenden Auswertungen
- [[02 Projekte/Brauchwasser-Wärmepumpe|Brauchwasser-Wärmepumpe]] — Stillstandsverlust der BW-WP, Stufentest und Zirkulations-Umbau (aus dem Victron-Projekt ausgelagert)
- [[02 Projekte/homelab-monitoring|homelab-monitoring]] — Grafana/InfluxDB-Monitoring für Energieverbrauch (aktuell zwei Klimaanlagen), Ergänzung zum HA-Energie-Dashboard aus dem Victron-Projekt
- [[02 Projekte/bwt|bwt]] — Flutter-App als digitale Bedienungsanleitung für Brauchwasser-Wärmepumpen (ecodesign/Wolf)
- [[02 Projekte/Energielandingpage|Energielandingpage]] — veröffentlichter Artikel „Die Energierevolution die wir brauchen", live unter [energie.softexceptions.com](https://energie.softexceptions.com)

## Verbundene Unterrichtsmaterialien

- [[03 Bereiche/Unterricht/MQTT-Bridge-Diagramm|MQTT-Bridge-Diagramm]] — Cerbo GX ↔ Proxmox MQTT-Bridge, aus der eigenen Anlage abgeleitet
- [[03 Bereiche/Unterricht/openWB-Abschaltvorgang|openWB-Abschaltvorgang]] — Abschaltverhalten der eigenen Wallbox als Lehrbeispiel
