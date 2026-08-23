---
tags: [victron, wartung, betrieb, sicherheit, runbook]
status: aktiv
date: 2026-07-07
updated: 2026-08-23
---

# Wartung und Betrieb

Anleitungen für Eingriffe an der laufenden Anlage. **Was hier steht, gilt.**

Verwandt: [[Anlage und Topologie]] · [[Cerbo Wächter]] · [[Node-RED Flows]]

## Wartung: Anlage sicher ab- und zuschalten

> ⚠️ **Sicherheit:** Arbeiten an der festen Elektroinstallation gehören in die Hand einer Elektrofachkraft. Die **5 Sicherheitsregeln** beachten: freischalten → gegen Wiedereinschalten sichern → **Spannungsfreiheit messen** → erden/kurzschließen → benachbarte Teile abdecken. **PV-Module/-Strings führen bei Tageslicht IMMER Spannung** — ein „ausgeschalteter" Wechselrichter/MPPT ist DC-seitig NICHT spannungsfrei. Ein Schalter erzeugt nur eine **Trennstelle**; spannungsfrei ist ausschließlich der allseitig (von PV, Batterie, Netz) getrennte Abschnitt. Am Arbeitsort immer **messen**, nie aus der Schalterstellung schließen.

**Anlagen-Topologie (2026-07-07 bestätigt):** Fronius Symo 17.5-3-M am **AC-IN** der 3× MultiPlus-II (→ netzabhängig, kein Insel-/Frequenzregelbetrieb, kein Backup-PV); 2× SmartSolar MPPT DC-gekoppelt an der Batterie; Batterie mit Batrium-BMS/Schütz; Cerbo GX über **separates 24-V-Netzteil an 230 V AC** (geht NICHT mit dem Batterie-Trennschalter aus, muss separat stromlos); Node-RED im Proxmox-LXC (stromunabhängig von der Anlage, hängt per D-Bus-TCP am Cerbo).

**Grundprinzip:** Abschalten = Erzeuger zuerst, Speicher (Batterie) zuletzt. Einschalten = umgekehrt. Vor jeder mechanischen DC-Trennung erst **lastfrei** machen (Software-Aus bzw. AC-Trennung) → kein DC-Lichtbogen (DC hat keinen Nulldurchgang!). **Fronius zuerst aus, zuletzt an.**

### Abschalten (für Wartung)
1. **Node-RED stoppen** im LXC (`systemctl stop nodered`) — verhindert false-reset-Statistiken beim späteren Reconnect.
2. **Haus-AC-Lasten** wegnehmen (Sicherungsautomaten einzeln aus).
3. **Fronius AC freischalten** (RCD/LS des Fronius-Abgangs, **allpolig**) → Anti-Islanding, Fronius stellt ab, DC-Seite wird lastfrei. Gegen Wiedereinschalten sichern.
4. **Fronius DC-Trennschalter** öffnen (jetzt lastfrei → lichtbogenfrei).
5. **PV-Strings im GAK** Richtung Fronius trennen (lastfrei; MC4-Stecker nie unter Last!).
6. **MultiPlus (3er-Verbund) ausschalten** — zentral über den **Cerbo GX** (Remote-Schalter im Gerätemenü). Die 3 Multi sind EIN VE.Bus-System; **nicht** einzeln oder „nur den Master" am Frontschalter schalten → VE.Bus-Fehler (Error 17/3). **Frontschalter aller 3 bleiben auf Stellung `I` (On)** — nicht `II` (Charger only), nicht `0` (Off). **Vor** dem Netz, sonst unnötiger Inselbetrieb-Wechsel. (Remote-Off = nur Standby; spannungsfrei erst durch Netz-/Batterie-Trennung.)
7. **Netz-Hauptschalter (AC-In)** freischalten — danach ist die AC-In-Schiene tot.
8. **MPPT 1 + 2 in VictronConnect auf „Aus"** (kein Lade-/Entladestrom). Nur falls an der PV-DC-Seite gearbeitet wird: **PV-Trenner der MPPT** öffnen, danach **Batterieseite** — Reihenfolge **PV → Batterie** einhalten (SmartSolar nie PV-only).
9. **Batterie** trennen (DC-Hauptschalter / Batrium-Schütz).
10. **Cerbo GX** separat stromlos machen (LS des 230-V-Netzteil-Kreises).
11. Vor Arbeiten: an jeder Arbeitsstelle **Spannungsfreiheit messen**. Fronius-Gehäuse erst nach **5 min Kondensator-Entladezeit** öffnen (klassische Symo-Serie; genaue Zeit steht auf dem Gerät). Modulseite bleibt bei Licht heiß.

### Einschalten (nach Wartung) — umgekehrt
1. **Batterie** zuschalten (DC-Hauptschalter / Batrium-Schütz).
2. **Cerbo GX** einschalten, sobald sein 230-V-Kreis Spannung hat, warten bis vollständig hochgefahren (⚠️ je nachdem, woher die 230 V kommen, ggf. erst nach Netz/MultiPlus verfügbar — vor Ort prüfen).
3. **MPPT** DC verbinden (**Batterie → dann PV**), in VictronConnect wieder auf „Ein".
4. **Netz-Hauptschalter (AC-In)** zuschalten.
5. **MultiPlus** einschalten — zentral über den **Cerbo GX** (Remote On), Frontschalter aller 3 auf Stellung `I` (On). Synchronisieren direkt aufs anliegende Netz (kein Insel-Zwischenschritt), warten bis stabil.
6. **Fronius** DC zuschalten (Module → DC-Trennschalter), dann **AC (RCD/LS)** → er synchronisiert auf die anliegende AC-In-Spannung. Fronius kommt zuletzt.
7. **Haus-AC-Lasten** wieder zuschalten.
8. **Warten**, bis Cerbo/VRM alle Geräte + Werte zeigt und das System stabil ist.
9. **Node-RED starten** (`systemctl start nodered`) — erst jetzt, damit keine Teilwerte in die retained-Topics/Statistik laufen.

**Vor Ort noch zu verifizieren:** exakte Bezeichnung/Reihenfolge der physischen Trennschalter; woher die 230 V für das Cerbo-Netzteil kommen (eigener Hauskreis oder Anlagen-AC?); SmartSolar-Anschlussreihenfolge laut Handbuch. Quelle Entladezeit: Fronius Symo/Eco Bedienungsanleitung (5 min).

