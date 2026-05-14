---
tags: [projekt, unterricht, mqtt, excalidraw, diagramm]
status: abgeschlossen
date: 2026-05-10
---

# MQTT Bridge Diagramm

Excalidraw-Diagramm zur Visualisierung des MQTT-Bridge-Setups zwischen Cerbo GX und Proxmox — erstellt für den Unterricht.

## Ziel

Das Diagramm macht das Konzept der MQTT-Bridge-Konfiguration (`mosquitto.conf`) visuell verständlich: warum die Bridge standardmäßig eine Einbahnstraße ist und wie der Rückkanal via `topic R/# out 0` explizit geöffnet werden muss.

## Diagramm

![[Excalidraw/mqtt-bridge-setup.excalidraw]]

## Inhalt

- **Cerbo GX** (192.168.2.XXX:1883) mit MQTT Broker
- **Proxmox Server** mit Mosquitto MQTT Broker + `mosquitto.conf`-Konfigurationsblock
- **Node-RED** als Keep-Alive Publisher
- **IN-Pfeil** (grün): gefilterte Live-Werte `N/+/battery/+/+/+`, `N/+/solarcharger/+/+/+`
- **OUT-Pfeil** (orange): Keep-Alive Signal `R/+/keepalive`
- **Warnung**: "Default: Einbahnstraße — ohne `topic R/# out 0` kein Rückkanal"

## Unterrichtseinsatz

Direkt auf [excalidraw.com](https://excalidraw.com) importierbar — Schüler:innen können Elemente anklicken, verschieben und eigene Notizen ergänzen.
