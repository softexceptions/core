---
tags: [victron, cerbo, monitoring, pushover, runbook]
status: aktiv
date: 2026-08-13
updated: 2026-08-23
---

# Cerbo Wächter

Meldet Reboots und Ausfälle des Cerbo GX per Pushover. **Was hier steht, gilt** —
Bau, Fallen und die Forensik der Vorfälle stehen im [[2026-08 Journal]].

> [!abstract] Warum es das braucht
> Ein Cerbo-Reboot war in HA vorher **unsichtbar**: retained Werte laufen lückenlos
> weiter, die Statistik zeigt keinen Bruch. Erst der Wächter macht ihn sichtbar.

> [!important] Der Merksatz dahinter
> **Ein Ausfall kann nie ein Ereignis der ausgefallenen Komponente sein** — tote
> Systeme senden nichts. „Ist weg" braucht deshalb einen Stellvertreter (MQTT
> Last Will) oder eine Zeitüberwachung. Das Last Will delegiert das Polling an
> den Broker.

## Architektur

Zwei Ereignisquellen, kein Polling:

| Quelle | Mechanismus | Meldet |
|---|---|---|
| **Ausfall** | MQTT Last Will, retained | Cerbo weg / wieder da |
| **Neustart** | `boot-notify.sh` beim Hochfahren | Startzeit + Startart |

**Startart-Klassifikation:** `geplant` · `unerwartet` · `ausgeschaltet` · `unbekannt`.
**Push-Prioritäten:** unerwarteter Neustart und Ausfall = Prio 1 (Ton),
geplanter Neustart und Entwarnung = Prio 0.

## Bausteine

- **Auf dem Cerbo** (Repo `cerbo/`, Ziel `/data/cerbo-online-watch/`):
  daemontools-Dienst `service/run`, `boot-notify.sh`, Hook in `/data/rc.local`
- **In Node-RED:** Tab „Cerbo-Wächter (Reboot & Ausfall)", `flows/cerbo-waechter-pushover.json`,
  Generator `tools/gen_cerbo_flow.py`, Test `tools/test_cerbo_waechter.js` (11 Fälle,
  läuft ohne Broker und ohne Push)
- **In HA** (MQTT-Discovery, Device „Cerbo GX"): `binary_sensor.cerbo_gx_online`,
  `sensor.cerbo_gx_letzter_start`, `sensor.cerbo_gx_startart`

> [!warning] Venus-OS-Besonderheiten
> - `/service` wird bei **jedem Boot neu aufgebaut** → der Symlink muss in
>   `rc.local` neu gesetzt werden
> - `/data/rc.local` ist persistent und überlebt Firmware-Updates
> - **Der Cerbo läuft auf UTC**, Node-RED und HA nicht — bei Zeitvergleichen beachten
> - `mosquitto_pub -P <pw>` legt das Passwort in `ps` offen → Zugangsdaten in
>   `mosquitto/mosquitto_{pub,sub}` (chmod 600), `XDG_CONFIG_HOME` explizit setzen
>   (beim Boot fehlt `HOME`)

## Zwei Fallen, die erst der Test zeigte

1. **`mosquitto_sub` reconnectet still.** Nach dem Keepalive-Timeout feuert der
   Broker das Last Will und trennt — `libmosquitto` verbindet aber von selbst neu,
   ohne den Status zu korrigieren. Das retained `offline` bliebe **für immer** stehen.
   → Der Client muss sein eigenes Status-Topic mitlesen und `online` republishen.
2. **Deduplizierung über den Startzeitpunkt, nicht den Meldezeitpunkt** — sonst
   meldet jeder Aufruf von `boot-notify.sh` einen Neustart. Toleranz 30 s, damit
   eine echte Boot-Schleife noch erkannt wird (Cerbo bootet in ~46 s).

> [!tip] Ausfälle richtig testen
> Immer per `SIGSTOP` (unsauberer Abriss), **nie** per sauberem Stopp — ein
> sauberes DISCONNECT feuert laut MQTT-Spec gar kein Last Will. Dafür meldet
> der `trap` selbst.

## Deutung einer Meldung

> [!caution] „Der Cerbo hat neu gestartet" — erst prüfen, dann glauben
> Der Wächter meldet auch **reine Netzausfälle** (Last Will), ohne dass der Cerbo
> rebootet hätte. Ein 80-Sekunden-Link-Down sieht in der Meldung genauso aus.
> **Immer zuerst `uptime` bzw. `sensor.cerbo_gx_letzter_start` prüfen.**
> Beispiel: der „Neustart" vom 18.08. 03:52 war ein Firmware-Update des Switches
> — siehe [[homelab-infrastruktur#Netzwerk-Vorfall 18.08.2026 und Fernzugriff]].

**Reboot-Ursache forensisch bestimmen** (auf dem Cerbo, Zeiten UTC):

| Fund | Bedeutung |
|---|---|
| `shutting down` + `runlevel: 6` in `/data/log/messages` | geplanter Reboot |
| beides fehlt, aber `/data/log/watchdog_processlist.txt` geschrieben | **Watchdog-Reset** |

⚠️ Der Watchdog rebootet per regulärem `shutdown -r` — das **sieht aus wie geplant**.
Die Forensikdatei hat deshalb Vorrang vor der Logzeile. Auslöser war bisher
Überlast (`/etc/watchdog.conf`, `max-load-15=6`; Normal-Load 2,5–3,3 liegt nah dran,
am 12.08. mit 7,00 ausgelöst). `messages` rotiert beim Boot → `messages.0` mitlesen.

## Betriebshinweise

```
# Dienststatus / Log
ssh root@192.168.2.181 'svstat /service/cerbo-online-watch'
ssh root@192.168.2.181 'tail -n 20 /var/log/cerbo-online-watch/current | tai64nlocal'

# Boot-Meldung von Hand auslösen (löst bewusst KEINEN Push aus - gleicher Start)
ssh root@192.168.2.181 '/data/cerbo-online-watch/boot-notify.sh'

# Push-Kette testen
curl -X POST http://192.168.2.80:1880/inject/cw_inject_test
```

Verwandt: [[Node-RED Flows]] · [[Wartung und Betrieb]] · [[Zähler und Statistik]]
