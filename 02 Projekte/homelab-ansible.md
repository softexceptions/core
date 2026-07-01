---
tags: [projekt/aktiv, devops, ansible, proxmox, homelab]
status: aktiv
date: 2026-06-23
updated: 2026-07-01
---

# homelab-ansible

Ansible-Repo zur Verwaltung des Homelab-Proxmox-Servers und aller LXC-Container. Infrastructure as Code — neue Dienste werden nicht manuell eingerichtet, sondern als Playbook + Rollen beschrieben.

## Projektpfad

```
/home/norbert/homelab-ansible/
```

GitHub: `https://github.com/softexceptions/homelab-ansible`

## Infrastruktur

### Proxmox-Host

| Eigenschaft | Wert |
|---|---|
| IP | 192.168.2.241 |
| Node | `pve` |
| API-User | `root@pam` |
| Python-Interpreter | `/opt/ansible-venv/bin/python` |

### LXC-Container

| Name | IP | Dienst |
|---|---|---|
| `node_red_lxc` | 192.168.2.80 | Node-RED |
| `zigbee2mqtt_lxc` | 192.168.2.147 | Zigbee2MQTT |
| `vaultwarden_lxc` | 192.168.2.17 | Vaultwarden |
| `noten_it_lxc` | 192.168.2.6 | Notenerfassung IT |
| `influxdb_lxc` | 192.168.2.119 | InfluxDB (VMID 112, DHCP) |
| `grafana_lxc` | 192.168.2.92 | Grafana (VMID 128, DHCP) |

### ansible-control

Deployments laufen von einem dedizierten LXC-Container (`ansible-control`, 192.168.2.101) — nicht vom Desktop. Der Desktop entwickelt und pusht; ansible-control pullt und deployt.

```
Desktop: git push → GitHub
ansible-control: git pull → ansible-playbook
```

## Nächste Aufgabe

— keine offenen Punkte —

## Wartungsprinzip

`wartung/system_updates.yml` läuft auf `hosts: all` — jeder neue Host der in `hosts.ini` eingetragen wird, wird automatisch mitgepflegt. Kein manuelles Ergänzen im Wartungs-Playbook nötig.

## Aktive Dienste

| Dienst | Playbook | Status |
|---|---|---|
| Notenerfassung IT | `playbooks/noten_it.yml` | deployed, HTTPS aktiv |
| Zigbee2MQTT | `wartung/run_lxc_zigbee2mqtt_updates.yml` | läuft, Update-Playbook vorhanden |
| InfluxDB | — | via Proxmox Community Script, im Inventar |
| Grafana | `playbooks/grafana.yml` | deployed, läuft |
| Node-RED | — | manuell eingerichtet (noch kein Playbook) |
| Vaultwarden | — | manuell eingerichtet (noch kein Playbook) |

---

## Session 2026-06-24 — Zigbee2MQTT aufgeräumt

### Entscheidungen & Erkenntnisse

**`wartung/` bleibt eigenständig** — konzeptionelle Trennung zwischen Deployment (Infrastruktur aufbauen) und Wartung (Betrieb pflegen) ist sinnvoll. Keine Rolle für Wartungs-Tasks die nicht wiederverwendet werden.

**Zigbee2MQTT war via Proxmox Community Script installiert** — git clone + pnpm, kein Docker. Deshalb kein 3-Phasen-Muster möglich (kein `docker_install`). Update läuft über offizielles `update.sh` im Container.

**Deployment läuft ausschließlich von `ansible-control`** — nicht vom Desktop. Befehle werden Norbert als Code-Block gegeben, er führt sie selbst aus.

### Was getan wurde

- [x] `core` zu `.gitignore` hinzugefügt (Obsidian-Vault-Symlink)
- [x] Alten Container 102 (`zigbee2mqtt`, gestoppt) gelöscht — war ersetzt durch 118
- [x] Container 118 von `zigbee2mqttSicherung` → `zigbee2mqtt` umbenannt (Proxmox + Hostname im Container)
- [x] Zigbee2MQTT-Secrets in Ansible Vault aufgenommen:
  - `zigbee2mqtt_mqtt_password`
  - `zigbee2mqtt_network_key`
  - `zigbee2mqtt_pan_id`
  - `zigbee2mqtt_ext_pan_id`
- [x] `wartung/run_lxc_zigbee2mqtt_updates.yml` korrigiert — `npm install -g` war falsch, jetzt `bash /opt/zigbee2mqtt/update.sh`
- [x] Update-Playbook getestet: Zigbee2MQTT `2.6.0 → 2.12.0` erfolgreich aktualisiert

### Offen

- [x] Bootdisk Container 118 auf 8 GB vergrößert (war 4 GB, 87% voll)

> [!note] Zigbee2MQTT Container
> VMID 118, IP 192.168.2.147 (DHCP, Lease auf MAC), Debian 12, Node.js v22, pnpm
> Zigbee-Stick: TCP `192.168.2.59:6638` (nicht USB direkt)
> MQTT-Broker: `192.168.2.233:1883`

---

## Session 2026-06-25 — InfluxDB-LXC ins Ansible-Inventar aufgenommen

### Was getan wurde

- [x] `influxdb_lxc` (VMID 112, IP 192.168.2.119) in `hosts.ini` eingetragen
- [x] SSH-Key von ansible-control via `pct push` + `pct exec` in den Container injiziert (Passwort-Auth war deaktiviert, kein Root-Passwort bekannt)
- [x] InfluxDB-GPG-Key erneuert — alter Key `9D539D90...` war abgelaufen (expired 2026-01-17), neuer Key `DA61C26A0585BD3B` vom Ubuntu-Keyserver geholt
- [x] Erstes `ansible-playbook wartung/system_updates.yml` erfolgreich — 90 Pakete aktualisiert, InfluxDB auf aktuellem Stand

### Erkenntnisse

**VMID ≠ IP** — VMID 112 hat IP `192.168.2.119` (DHCP-Lease fest auf MAC). VMID 119 ist `evcc`, nicht InfluxDB — Verwechslungsgefahr.

**Initialer SSH-Key-Inject via `pct push`** — Standardweg wenn Passwort-Auth deaktiviert und kein Key vorhanden:
```bash
scp ~/.ssh/id_rsa.pub root@192.168.2.241:/tmp/ak.pub
ssh root@192.168.2.241 "pct push 112 /tmp/ak.pub /tmp/ak.pub"
ssh root@192.168.2.241 "pct exec 112 -- bash -c 'mkdir -p /root/.ssh; cat /tmp/ak.pub >> /root/.ssh/authorized_keys; chmod 700 /root/.ssh; chmod 600 /root/.ssh/authorized_keys'"
```

**GPG-Key ohne `/dev/tty`** — `pct exec` hat kein Terminal. `gpg --batch --yes --dearmor` + vorher `rm` der alten Datei. Key-Quelle: Ubuntu-Keyserver per HTTPS (`keyserver.ubuntu.com/pks/lookup?op=get&search=0x<KEY-ID>`).

> [!note] InfluxDB Container
> VMID 112, IP 192.168.2.119 (DHCP, Lease auf MAC), Debian 12 (bookworm), via Proxmox Community Script installiert
> Repo: `https://repos.influxdata.com/debian stable`, Key: `DA61C26A0585BD3B`

---

## Session 2026-06-25 (2) — Grafana-LXC via Ansible deployed

### Was getan wurde

- [x] `roles/grafana_install/` erstellt — Grafana via apt, offizielles Grafana-Repo (`apt.grafana.com`)
- [x] `playbooks/grafana.yml` — 2-Phasen-Muster: `lxc` + `grafana` (kein Docker nötig)
- [x] Bug in `proxmox_lxc`-Rolle gefixt: `playbook_dir` → `inventory_dir` für `geheimnisse.yml` (Pfad war falsch wenn Playbook in `playbooks/`-Unterordner liegt)
- [x] `grafana_lxc` (VMID 128, IP 192.168.2.92) in `hosts.ini` eingetragen
- [x] Grafana 13.1.0 installiert und gestartet — erreichbar auf `http://192.168.2.92:3000`

### Erkenntnisse

**Grafana ohne Docker** — nativer apt-Dienst, kein `docker_install` nötig. Schlanker als Docker für ein einzelnes Binary. `nesting=1` wird von `proxmox_lxc` trotzdem gesetzt (hardcoded), schadet aber nicht.

**`playbook_dir` vs `inventory_dir`** — `playbook_dir` zeigt auf das Verzeichnis des Playbooks, nicht den Repo-Root. Bei Playbooks in `playbooks/`-Unterordner zeigt `inventory_dir` korrekt auf den Repo-Root wo `geheimnisse.yml` liegt.

**SSH-Key bei neuen LXC automatisch injiziert** — `proxmox_lxc`-Rolle trägt `id_rsa.pub` von ansible-control direkt ein. Nur `ssh-keyscan` für `known_hosts` nötig, kein `pct push`.

**InfluxDB + Grafana Kompatibilität:** InfluxDB v2.9.1 + Grafana 13.1.0 — Datasource-Typ: InfluxDB mit Flux Query Language (`http://192.168.2.119:8086`)

> [!note] Grafana Container
> VMID 128, IP 192.168.2.92 (DHCP), Debian 13, Grafana 13.1.0
> Login: `admin` / `admin` (beim ersten Login ändern)
> Noch zu tun: InfluxDB-Datasource einrichten (Connections → Data sources → InfluxDB, Flux-Modus)

---

## Session 2026-06-27 — ha-mcp + SK_Arbeitszimmer_Küche Energiemonitoring

### Was getan wurde

- [x] ha-mcp lokal eingerichtet — kein LXC nötig, läuft als stdio-MCP-Prozess direkt auf dem Desktop
  - `claude mcp add home-assistant --scope user --env HOMEASSISTANT_URL="http://192.168.2.186:8123" --env HOMEASSISTANT_TOKEN="..." -- uvx --python 3.13 --refresh ha-mcp@latest`
  - 77 Home Assistant Tools in Claude Code verfügbar
- [x] Grafana Datasource für InfluxDB eingerichtet: InfluxDB v2, Flux, `http://192.168.2.119:8086`, Org `ng`, Bucket `homeassistant`
- [x] HA InfluxDB v2 Integration eingerichtet (UI-basiert, da YAML-Config in 2026.x deprecated)
  - Schreibt `sensor.0xa085e3fffebc4574_energy` (kWh) + `sensor.0xa085e3fffebc4574_power` (W) — Zigbee-Steckdose „SK_Arbeitszimmer_Küche" (Klimaanlage)
- [x] Grafana Dashboard „SK_Arbeitszimmer_Küche" erstellt:
  - Momentanleistung (Gauge, W)
  - Tagesverbrauch (Bar chart, kWh) — füllt sich ab morgen
  - Wochenverbrauch (Bar chart, kWh)
  - Monatsverbrauch (Bar chart, kWh)

### Erkenntnisse

**ha-mcp ist stdio-basiert** — kein Daemon, kein LXC, kein Docker. Claude Code startet `uvx ha-mcp` on-demand. Installation per `claude mcp add`, nicht per Ansible.

**HA InfluxDB YAML deprecated** — ab HA 2026.9 werden Verbindungskeys (`host`, `token`, `organization`, `bucket` etc.) aus YAML entfernt. Verbindung muss über UI eingerichtet werden. `include.entities` kann weiterhin in YAML stehen.

**difference() braucht 2 Tage** — Tages-/Wochen-/Monatsverbrauchs-Panels zeigen erst Daten wenn Datenpunkte aus mindestens 2 verschiedenen Aggregationsfenstern vorhanden sind.

### Offen

- [x] Grafana-Panels auf rohe Flux-Queries umgestellt (Script Editor, siehe Session 2026-06-27 (2))
- [x] Grafana-Panel in HA-Dashboard integriert (siehe Session 2026-06-27 (3))

> [!note] SK_Arbeitszimmer_Küche
> Zigbee-Steckdose mit Energiemessung, hängt an der Klimaanlage (entity_id-Prefix: `0xa085e3fffebc4574`)
> InfluxDB-Measurement: `W` (power) + `kWh` (energy), entity_id-Tag: `0xa085e3fffebc4574_power` / `0xa085e3fffebc4574_energy`

### Erkenntnisse (Nachtrag 2026-06-27) — Grafana „no tag keys found"

**Root Cause:** HA InfluxDB v2 schreibt `entity_id` nicht als indizierten InfluxDB-Tag, sondern als reguläre Spalte. Grafanas Query Builder ruft intern `schema.tagValues()` auf — das schlägt fehl → Fehler „no tag keys found in the current time range".

**Fix:** In jedem Grafana-Panel auf **Script Editor** (rohe Flux-Query) umstellen, Query Builder nicht verwenden.

**Bestätigte Datenstruktur in InfluxDB:**
- `_measurement`: `W` (Leistung) / `kWh` (Energie)
- `entity_id`: `0xa085e3fffebc4574_power` / `0xa085e3fffebc4574_energy`
- `domain`: `sensor`
- `_field`: `value` (numerischer Wert)

**Finale Flux-Queries (mit Timezone-Fix):**

Gauge:
```flux
from(bucket: "homeassistant")
  |> range(start: -12h)
  |> filter(fn: (r) => r["_measurement"] == "W")
  |> filter(fn: (r) => r["entity_id"] == "0xa085e3fffebc4574_power")
  |> filter(fn: (r) => r["_field"] == "value")
  |> last()
```

Tagesverbrauch:
```flux
import "timezone"

from(bucket: "homeassistant")
  |> range(start: -8d)
  |> filter(fn: (r) => r["_measurement"] == "kWh")
  |> filter(fn: (r) => r["entity_id"] == "0xa085e3fffebc4574_energy")
  |> filter(fn: (r) => r["_field"] == "value")
  |> aggregateWindow(every: 1d, fn: last, createEmpty: false, location: timezone.location(name: "Europe/Berlin"))
  |> difference(nonNegative: true)
```

Wochenverbrauch:
```flux
import "timezone"

from(bucket: "homeassistant")
  |> range(start: -9w)
  |> filter(fn: (r) => r["_measurement"] == "kWh")
  |> filter(fn: (r) => r["entity_id"] == "0xa085e3fffebc4574_energy")
  |> filter(fn: (r) => r["_field"] == "value")
  |> aggregateWindow(every: 1w, fn: last, createEmpty: false, location: timezone.location(name: "Europe/Berlin"))
  |> difference(nonNegative: true)
```

Monatsverbrauch:
```flux
import "timezone"

from(bucket: "homeassistant")
  |> range(start: -13mo)
  |> filter(fn: (r) => r["_measurement"] == "kWh")
  |> filter(fn: (r) => r["entity_id"] == "0xa085e3fffebc4574_energy")
  |> filter(fn: (r) => r["_field"] == "value")
  |> aggregateWindow(every: 1mo, fn: last, createEmpty: false, location: timezone.location(name: "Europe/Berlin"))
  |> difference(nonNegative: true)
```

---

---

## Session 2026-06-27 (2) — Grafana Flux-Queries umgestellt *(durch Session 2026-07-01 überholt)*

### Was getan wurde

- [x] Alle 4 Grafana-Panels von Query Builder auf Script Editor (rohe Flux-Queries) umgestellt
- [x] Gauge: `range(start: -12h)` + `last()` — robuster gegen 6h Stille bei konstantem Shelly-Verbrauch
- [x] Tagesverbrauch: `range(start: -8d)`, `aggregateWindow(every: 1d)`, `difference(nonNegative: true)`
- [x] Wochenverbrauch: `range(start: -9w)`, `aggregateWindow(every: 1w)`, `difference(nonNegative: true)`
- [x] Monatsverbrauch: `range(start: -13mo)`, `aggregateWindow(every: 1mo)`, `difference(nonNegative: true)`

### Erkenntnisse

**Shelly sendet ausschließlich bei Wertänderung** — bei konstantem 7W-Standby war der Sensor 5h49min still (01:10–06:59 Uhr). `range(start: -12h)` fängt das zuverlässig ab.

**Reconnects bei Lastübergängen** — Zigbee-Chip im Sensor wird durch Schaltimpulse des Inverter-Klimakompressors kurz gestört. LQI 207 (gut) → kein Signal-/Routing-Problem. `nonNegative: true` filtert die daraus resultierenden kWh-Sprünge heraus.

**`+1 Puffer` für difference()** — `aggregateWindow` + `difference()` braucht einen Datenpunkt vor dem eigentlichen Zeitraum. Deshalb immer 1 Einheit mehr im `range()` als angezeigt werden soll (z.B. `-8d` für 7 Tagesbalken).

**Timezone-Fix für aggregateWindow** — Flux rechnet standardmäßig in UTC. Für korrekte Tagesgrenzen (00:00 Berlin-Zeit) `import "timezone"` + `location: timezone.location(name: "Europe/Berlin")` verwenden. Ohne Fix werden Tagesgrenzen 2h versetzt berechnet.

### Offen

- [x] Grafana-Panel in HA-Dashboard integriert (siehe Session 2026-06-27 (3))

---

## Session 2026-06-27 (3) — Grafana-Panel in HA-Dashboard integriert

### Was getan wurde

- [x] Desktop-SSH-Key (`id_ed25519`) in Grafana-LXC hinterlegt via `ssh-copy-id`
- [x] `allow_embedding = true` in `/etc/grafana/grafana.ini` unter `[security]` gesetzt
- [x] `enabled = true` unter `[auth.anonymous]` gesetzt — nötig damit iFrame ohne Cookie funktioniert
- [x] `iframe`-Karte im EG Arbeitszimmer-Dashboard eingefügt (Section „Klimaanlage")
- [x] Native HA Gauge-Karte für Momentanleistung (statt Grafana iFrame) — kein Branding-Problem
- [x] Native HA statistics-graph-Karte für Tagesverbrauch (letzte 7 Tage, Balkendiagramm)
- [x] Grafana iFrame für Wochen-/Monatsverbrauch optional weiterhin verfügbar

### Erkenntnisse

**`webpage` vs `iframe`** — HA-Kartentyp für externe URLs ist `iframe`, nicht `webpage`. `webpage` wirft einen Konfigurationsfehler.

**SameSite-Cookie-Problem** — Browser sendet Grafana-Session-Cookie nicht aus einem iFrame eines anderen Hosts (HA). Lösung: Anonymous Access in Grafana aktivieren (`[auth.anonymous] enabled = true`), dann kein Cookie nötig.

**`allow_embedding` allein reicht nicht** — Zusätzlich muss Anonymous Access aktiv sein, sonst kommt Grafana-Startseite statt Panel.

**„Powered by Grafana" nicht entfernbar** — OSS-Pflichtbranding, nur in Grafana Enterprise entfernbar. Lösung: Native HA-Karte statt Grafana-iFrame für Einzelwerte.

**Desktop-SSH-Key** — Desktop nutzt `id_ed25519`, nicht `id_rsa`. Bei neuem LXC immer beide Keys via `ssh-copy-id` eintragen oder direkt `id_ed25519.pub` per `pct push` injizieren.

**`state_class: total_increasing`** — Voraussetzung für `statistics-graph`. Sensor `0xa085e3fffebc4574_energy` hat diese Klasse — HA-Langzeitstatistiken funktionieren direkt.

**HA Gauge-Karte (Momentanleistung):**
```yaml
type: gauge
entity: sensor.0xa085e3fffebc4574_power
name: Momentanleistung
unit: W
min: 0
max: 3000
severity:
  green: 0
  yellow: 1000
  red: 2000
```

**HA statistics-graph-Karte (Tagesverbrauch):**
```yaml
type: statistics-graph
entities:
  - entity: sensor.0xa085e3fffebc4574_energy
    name: Klimaanlage
stat_types:
  - change
period: day
days_to_show: 7
title: Tagesverbrauch
chart_type: bar
```

**HA statistics-graph-Karte (Tagesverbrauch) — wiederverwendbares Muster:**
```yaml
type: statistics-graph
entities:
  - entity: sensor.<entity_id>_energy
    name: <Name>
stat_types:
  - change
period: day
days_to_show: 7
title: Tagesverbrauch
chart_type: bar
```
Voraussetzung: `state_class: total_increasing` am Energie-Sensor.

**Eingerichtete Klimaanlagen-Dashboards:**

| Raum | Power-Entity | Energy-Entity | Dashboard |
|---|---|---|---|
| EG Arbeitszimmer/Küche | `0xa085e3fffebc4574_power` | `0xa085e3fffebc4574_energy` | `eg-arbeitszimmer` |
| EG Wohnzimmer | `0xa085e3fffebd16b8_power` | `0xa085e3fffebd16b8_energy` | `eg-wohnzimmer` |

### Offen

— keine offenen Punkte —

---

## Session 2026-07-01 — Grafana-Queries korrigiert, Wohnzimmer-Dashboard eingerichtet

### Was getan wurde

- [x] Flux-Queries auf **Zwei-Stufen-Aggregation** umgestellt (tägliche Differenzen → wöchentlich/monatlich/jährlich summieren) — korrekte Werte auch für unvollständige erste Wochen
- [x] `timeSrc: "_start"` in allen Queries ergänzt — heutiger Balken erscheint sofort statt erst morgen
- [x] `offset: -3d` im Wochen-Query — ISO-Kalenderwochen (Mo–So) statt Do–Do (Unix-Epoch-Default)
- [x] Jahresverbrauch-Panel neu angelegt
- [x] Grafana-Transformation für lesbare X-Achsen-Labels:
  - Woche: `[KW]WW` → KW26, KW27 (moment.js Format, eckige Klammern für Literale)
  - Monat: `MMM YYYY` → Jul 2026
  - Jahr: `YYYY` → 2026
- [x] Wohnzimmer-Entitäten in `configuration.yaml` ergänzt + InfluxDB-Integration neu geladen
- [x] Grafana Dashboard „EG Wohnzimmer" dupliziert und entity_id-Prefix ersetzt
- [x] `homelab-monitoring.md` im Vault angelegt — alle Betriebsdetails dort

### Erkenntnisse

**Zwei-Stufen statt direkte Wochen-Aggregation** — `aggregateWindow(every: 1w, fn: last) |> difference()` verliert die erste Woche wenn kein Vorgänger-Bucket existiert. Fix: erst täglich aggregieren + difference(), dann wöchentlich summieren.

**moment.js Escape-Syntax** — Grafanas „Convert field type"-Transformation verwendet moment.js. Literale mit `[...]` escapen, nicht mit `'...'` (das ist date-fns Syntax → zeigt die Anführungszeichen literal an).

**HA schreibt nur bei State Changes** — nach `influxdb`-Reload in HA und Ergänzung neuer Entitäten: erst nach einem Zustandswechsel der Steckdose erscheinen Daten in InfluxDB. Steckdose kurz ein-/ausschalten um den ersten Wert zu triggern.

> [!note] Alle Flux-Query-Muster und Dashboard-Details
> Liegen in [[homelab-monitoring]] — nicht hier duplizieren.

## Verwandte Notizen

- [[02 Projekte/homelab-monitoring]] — Betrieb: Flux-Queries, Grafana-Dashboards, Entity-IDs
- [[03 Bereiche/Unterricht/Ansible]] — Rollen-Architektur, Stolperfallen, Unterrichtsbeispiele
- [[02 Projekte/Notenerfassung_IT]] — App die auf `noten_it_lxc` läuft
