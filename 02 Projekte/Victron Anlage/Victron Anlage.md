---
tags: [projekt/aktiv, victron, node-red, energie, homeassistant, mqtt, pv, moc]
status: aktiv
date: 2026-07-05
updated: 2026-08-23
---

# Victron Anlage

Node-RED-Flows für die Victron-Anlage (Cerbo GX, Gridmeter VM-3P75CT) → Home Assistant.
Repo: `~/Code/victron_node_red`

**Kurzstand (23.08.2026):** Alle Zähler-Flows gehärtet, Ersparnis-Kennzahl in Euro
auf dem Energie-Dashboard live, Cerbo-Wächter meldet Reboots und Ausfälle getrennt.
Offen ist im Wesentlichen die Winter-Messkampagne.

> [!tip] Wie diese Notizen zu lesen sind
> **Themennotizen** sagen, was *jetzt gilt* — sie werden überschrieben.
> Das **Journal** sagt, *was passiert ist* — es wird nie geändert.
> Bei Widersprüchen gewinnt die Themennotiz.

## Grundlagen

- [[Anlage und Topologie]] — Aufbau, D-Bus über TCP, MQTT-Broker-Landkarte, Begriffe
- [[Wartung und Betrieb]] — Anlage sicher ab- und zuschalten (5 Sicherheitsregeln)

## Datenwege

- [[Node-RED Flows]] — Register aller Flows, Topic-Konventionen, Generator-Regel
- [[Zähler und Statistik]] — Härtung gegen `total_increasing`-Phantomzuwächse
- [[Cerbo Wächter]] — Reboot- und Ausfallmeldung, Reboot-Forensik

## Analyse und Entscheidungen

- [[Energiebilanz und Kennzahlen]] — Wirkungsgrade, Leerlaufsockel, was welche Zahl misst
- [[Wirtschaftlichkeit]] — Öl vs. Wärmepumpe, Arbitrage, was der Leerlauf kostet
- [[Ausbauplanung]] — Whole-home-Backup, Batterie 30 kWh, Winter-Messplan

## Verlauf

- [[2026-07 Journal]] — Aufbau der Flows, Dashboards, erste Auswertungen
- [[2026-08 Journal]] — Cerbo-Absturz und Wächter, Ersparnis-Kennzahl, Zähler-Härtung

## Offene Punkte

**Messungen, die den Winter brauchen**
- ⬜ **Leerlaufsockel: 118 W oder 174 W?** Zwei Regressionen widersprechen sich; die
  Differenz verschiebt die Jahreskosten um 82 €. → [[Energiebilanz und Kennzahlen]]
- ⬜ **Hebbare Standby-Stunden zählen** (AC-Abgabe ≈ 0 **und** Netzbezug läuft) —
  erst danach ist die Standby-Entscheidung belastbar. → [[Ausbauplanung]]
- ⬜ **Reale JAZ der Split-Klimas im Heizbetrieb** — entscheidet zwischen +739 € und
  +1.927 € Jahresersparnis, größte Unsicherheit der ganzen Rechnung. → [[Wirtschaftlichkeit]]
- ⬜ **Gegenprobe nach dem Batterie-Ausbau:** Regression wiederholen, `P_fix` muss
  gleich bleiben. → [[Ausbauplanung]]

**Kleinere offene Punkte**
- *(Prognose-Vergleich und Odenwald-Verdacht am 23.08.2026 geklärt → [[Anlage und Topologie#PV-Prognose]])*
- **Grafana „BW-WP Heizungskeller": Reset-Tag 12.07. bleibt ohne Tagesbalken** — der Zähler-Reset durchs Firmware-Update (7,41 → 0, 13:13 Uhr) lässt `difference(nonNegative: true)` den Tag verwerfen; Datenpfad HA → InfluxDB → Grafana nachweislich intakt, ab 13.07. Selbstheilung. Entscheidung offen: Panels auf reset-festes Muster umbauen (erst Punkt-Differenz, dann Tages-Summe) — Diagnose, Nachrechnung und Query-Muster in [[homelab-monitoring]] (Session 2026-07-12 + Stolperfalle „Zähler-Reset reißt ein Tagesloch").
- Weitere **Geräte-Inventur** fürs große Energie-Board — die beiden anderen Punkte
  dieses alten Wunschzettels (Electricity-Maps-CO₂-Gauge, Forecast.Solar) sind seit
  dem 06.07.2026 umgesetzt.
- Kosmetik: Anzeigename doppelt „Victron" („Victron VM-3P75CT Victron Netzbezug") — bei Gelegenheit Discovery-Namen kürzen (Grid-Flow).
- **rateLimit der Victron-Nodes greift nicht** (Spannung/Strom/Leistung updaten 1×/s trotz rateLimit 5000): Semantik prüfen (evtl. nur mit onlyChanges=false wirksam?) oder Delay-Nodes (rate 1/5s, drop) nachrüsten — Recorder-Volumen. Anzeige-Präzision im Registry gesetzt: Spannung 2, Strom 1 Nachkommastellen.


## Verwandt

[[Brauchwasser-Wärmepumpe]] — Stillstandsverlust und Zirkulations-Umbau
· [[homelab-monitoring]] — Grafana/InfluxDB, Zigbee-Messstellen
· [[homelab-infrastruktur]] — Netzwerk, Fernzugriff
· [[Elektrotechnik & Erneuerbare Energien]] — Bereichsnotiz mit Anlagen-Steckbrief
