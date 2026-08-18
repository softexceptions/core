---
tags: [projekt/aktiv, devops, ansible, proxmox, homelab]
status: aktiv
date: 2026-06-23
updated: 2026-08-17
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
| `grafana_lxc` | 192.168.2.214 | Grafana (VMID 128, statische IP seit 2026-07-05) |

### ansible-control

Deployments laufen von einem dedizierten LXC-Container (`ansible-control`, 192.168.2.101) — nicht vom Desktop. Der Desktop entwickelt und pusht; ansible-control pullt und deployt.

```
Desktop: git push → GitHub
ansible-control: git pull → ansible-playbook
```

## Nächste Aufgabe

— keine offenen Punkte —

> [!note] Bestandsaufnahme aller Container, VMs und Domains
> Steht in [[homelab-infrastruktur]] (erhoben 2026-07-27).

## Wartungsprinzip

`wartung/system_updates.yml` läuft auf `hosts: all` — jeder neue Host der in `hosts.ini` eingetragen wird, wird automatisch mitgepflegt. Kein manuelles Ergänzen im Wartungs-Playbook nötig.

## Aktive Dienste

| Dienst | Playbook | Status |
|---|---|---|
| Notenerfassung IT | `playbooks/noten_it.yml` | deployed, HTTPS aktiv |
| Zigbee2MQTT | `wartung/run_lxc_zigbee2mqtt_updates.yml` | läuft, Update-Playbook vorhanden |
| InfluxDB | — | via Proxmox Community Script, im Inventar |
| Grafana | `playbooks/grafana.yml` | deployed, läuft |
| Node-RED | `wartung/check_lxc_nodered_updates.yml` + `wartung/run_lxc_nodered_updates.yml` | manuell eingerichtet (kein Deploy-Playbook), Wartungs-Playbooks vorhanden, v5.0.1 |
| Vaultwarden | `playbooks/vaultwarden.yml` | Docker-Image 1.37.0, seit 2026-07-27 migriert |

## Runbook: Vaultwarden aktualisieren

Seit der Docker-Migration (Session 2026-07-27) wird **eine Zeile im Repo geändert**, nicht
der Container angefasst.

**1. Version und Release Notes prüfen**

```bash
curl -s https://api.github.com/repos/dani-garcia/vaultwarden/releases/latest | grep '"tag_name"'
gh api repos/dani-garcia/vaultwarden/releases/tags/<version> --jq '.body' | head -40
```

Bei Vaultwarden lohnt der Blick besonders — in den Releases 1.34, 1.36 und 1.37 standen
jeweils Security-Advisories im Kopf, 1.37 zusätzlich ein harter Client-Kompatibilitätshinweis.

**2. Tag setzen** in `roles/vaultwarden_deploy/defaults/main.yml`:

```yaml
vaultwarden_image_tag: "1.38.0"
```

**3. Committen und pushen**

```bash
git commit -am "chore: Vaultwarden auf 1.38.0 aktualisiert" && git push
```

**4. Ausrollen**

```bash
ssh root@192.168.2.101 'cd /root/homelab-ansible && git pull && ansible-playbook playbooks/vaultwarden.yml'
```

Der Lauf zieht das Image, tauscht den Container (Downtime: Sekunden) und verifiziert selbst
Server-Version, Web-Vault-Version, `DOMAIN` und die Länge des `ADMIN_TOKEN`. Die Daten im
Volume werden nicht angefasst, der Kopierschritt überspringt sich.

### Backup vor größeren Sprüngen

Ein Schema-Rollback gibt es **nicht** — Vaultwarden migriert beim Start vorwärts, ältere
Versionen öffnen die Datei danach nicht mehr. Vor mehreren Minor-Versionen auf einmal oder
wenn die Notes Migrationen erwähnen:

```bash
F=~/vw_backup_$(date +%F_%H%M).tar.gz; ssh root@192.168.2.101 "ssh root@192.168.2.17 'docker compose -f /opt/vaultwarden-docker/docker-compose.yml stop; tar czf - -C /opt/vaultwarden-docker data; docker compose -f /opt/vaultwarden-docker/docker-compose.yml start'" > "$F" && ls -lh "$F"
```

Prüfen statt vertrauen — Archiv auspacken und die Datenbank befragen:

```bash
sqlite3 db.sqlite3 "pragma integrity_check; select count(*) from ciphers;"
```

Stand 2026-07-27: **192 Einträge, 1 Benutzer.**

### Rollback

Tag zurücksetzen, Schritt 3 und 4 wiederholen — das alte Image liegt lokal, der Rücktausch
dauert Sekunden. **Aber nur, solange das Schema unverändert ist.** Der Notausgang auf die
alte Quellcode-Installation unter `/opt/vaultwarden` gilt nur für den Stand vom 2026-07-27
und verfällt mit dem ersten Update, das die Datenbank anfasst.

### Läuft daneben weiter

- **Betriebssystem:** `ansible-playbook wartung/system_updates.yml` — `vaultwarden_lxc` steht
  in `hosts.ini` und wird automatisch mitgenommen (siehe Wartungsprinzip oben).
- **Alte Images aufräumen:** `docker image prune -a -f` im Container. Bei 6 GB Platte
  summieren sich ~200 MB je Version. Erst ausführen, wenn die neue Version stabil läuft —
  es entfernt auch das vorherige Image und damit den schnellen Rücktausch.

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
> VMID 128, IP 192.168.2.214 (statisch, bis 2026-07-05: 192.168.2.92 via DHCP), Debian 13, Grafana 13.1.0
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

---

## Session 2026-07-05 — Node-RED Update auf 5.0.1

### Was getan wurde

- [x] `wartung/check_lxc_nodered_updates.yml` auf `ansible-control` ausgeführt — System-Updates standen an, Node-RED `4.1.11 → 5.0.1` verfügbar
- [x] Release Notes zu Node-RED 5.0.0/5.0.1 auf GitHub geprüft vor dem Update
- [x] `wartung/run_lxc_nodered_updates.yml` erfolgreich ausgeführt — Backup + System-Update + npm-Update + Service-Neustart

### Erkenntnisse

**Node-RED 5.0 verlangt Node.js ≥ 22.9** (empfohlen 24) — einzige echte Breaking Change des Major-Releases. `node_red_lxc` lief bereits auf Node.js v22.23.1, damit unkritisch.

**5.0 ist überwiegend ein Editor-UX-Release** — überarbeitete Sidebar, neues Deploy-Menü, eingebautes Dark Theme, Accessibility-Verbesserungen, Monaco-Update. Kein Bruch bei Flow-Dateiformat, Node-API oder Community-Nodes.

**Vor Major-Version-Updates lohnt sich ein kurzer Blick in die GitHub-Releases** (`gh api repos/<org>/<repo>/releases/tags/<version> --jq '.body'`) statt blind zu updaten — hier zeigte sich schnell, dass das Risiko gering war.

---

## Session 2026-07-05 (2) — Grafana-LXC: neue IP nach Memory-Resize, statische IP eingeführt

### Was getan wurde

- [x] Ursache für „Grafana unerreichbar" nach `--tags lxc`-Lauf diagnostiziert: Container-Neustart durch Memory-Update (512→1024 MB) → neue DHCP-Lease `192.168.2.92 → 192.168.2.214`
- [x] Entscheidung: neue IP `192.168.2.214` behalten und statisch festschreiben
- [x] `proxmox_lxc`-Rolle: `net0` über neue Variable `lxc_netif` konfigurierbar (Default bleibt DHCP)
- [x] `playbooks/grafana.yml`: `lxc_netif` mit statischer IP `192.168.2.214/24, gw 192.168.2.1`
- [x] `hosts.ini`: `grafana_lxc` auf `192.168.2.214` aktualisiert
- [x] HA-Dashboards geprüft: kein iFrame referenziert mehr die alte Grafana-IP (wurde in Session 2026-07-01 durch native Karten ersetzt) — nichts anzupassen

### Erkenntnisse

**`community.proxmox.proxmox` (2.0.0) aktualisiert bestehende Container bei `state: present`** — ein erneuter `--tags lxc`-Lauf ist kein No-op; Config-Änderungen werden übernommen und der Container startet dabei neu.

**DHCP + fest eingetragene IPs im Inventar sind fragil** — jeder Container-Neustart kann die Lease wechseln. Dienste, auf die andere Systeme zeigen, bekommen ab jetzt eine statische IP über `lxc_netif`.

**Statische IP im DHCP-Pool braucht eine Router-Reservierung/Ausnahme** für die MAC `BC:24:11:5D:D0:6F`, sonst droht Adresskonflikt.

---

## Session 2026-07-27 — Vaultwarden: vom Quellcode-Build auf das Docker-Image

### Ausgangslage

Vaultwarden lief auf **1.33.2**, im April 2025 von Hand aus dem Quellcode gebaut
(Quellen in `/vaultwarden`, Rust-Toolchain unter `/root`, systemd-Dienst als User
`vaultwarden`, LXC 108). Erreichbar über den Nginx Proxy Manager unter
`vaultwarden.softexceptions.com` → `192.168.2.17:8000`.

Das vorhandene `wartung/run_lxc_vaultwarden_updates.yml` hatte **nie funktioniert**:
In allen zehn Commits vom 15.06.2026 waren die GitHub-URLs auf `https://github.com`
bzw. `https://githubusercontent.com` gekürzt. Dadurch blieb `vaultwarden_remote_version`
leer, der Versionsvergleich schlug immer an, und Schritt 6 legte bei jedem Lauf ein
Backup an — daher die fünf Archive `data_backup_2026-06-15_22{34,37,40,42,44}.tar.gz`.
Die Vault-Tabelle war mit „manuell eingerichtet (noch kein Playbook)" also näher an
der Wahrheit als das Repository.

### Warum das Update dringend war

- **1.37.0 ist Pflicht für Clients ab 2026.7.0** — die Migration fiel auf den 27.07.
- Security-Advisories in 1.34 (CVE-2025-25188), 1.36 (SSRF Icon-Endpoint,
  User/Org-Enumeration) und 1.37 (8 Advisories). Die SSO-Advisories aus 1.36 betrafen
  1.33.2 nicht — SSO gibt es erst ab 1.35.

### Entscheidung: Docker statt Neukompilieren

Das Community-Skript kompiliert mit `--features sqlite,mysql,postgresql` aus dem
Quellcode. Bei 1,6 GB freiem Speicher hätte das eine Plattenvergrößerung erfordert,
20–40 Minuten gedauert und wäre bei jedem Update erneut angefallen. Der Image-Weg
(83 MB komprimiert) macht Updates zu `pull` + `up -d` und erlaubt es, die 2,5 GB
Build-Werkzeug abzuräumen. Ausschlaggebend war aber die Rollback-Eigenschaft:
Das Image ist ein unveränderliches, versioniertes Artefakt, der Quellcode-Build
überschreibt das laufende Binary an Ort und Stelle.

### Was getan wurde

- [x] Diagnose: `/vaultwarden` als Quellcode-Checkout vom April 2025 identifiziert (1,1 GB `target/`)
- [x] Backup `data` + `.env` weg vom Container auf den Desktop, mit `pragma integrity_check`
      und Zählung verifiziert: **192 Einträge, 1 Benutzer**
- [x] Neue Rolle `roles/vaultwarden_deploy` (nicht `app_deploy` — kein Repo, kein eigener Build),
      `meta`-Abhängigkeit auf `docker_install`, Muster wie `grafana_install`
- [x] `playbooks/vaultwarden.yml` ohne Phase `lxc` — Container existiert, `nesting=1` war gesetzt
- [x] Migration gelaufen: Vaultwarden **1.37.0**, Web-Vault 2026.6.0, 192 Einträge unverändert
- [x] `DOMAIN` ergänzt (war nie gesetzt, siehe unten)
- [x] `SIGNUPS_ALLOWED=false` — bewusste Änderung, vorher stand die Registrierung offen

### Erkenntnisse

**`DOMAIN` war nie gesetzt.** `/api/config` meldete `http://localhost` für `api`,
`identity`, `notifications` und `vault`. Hinter einem Reverse Proxy bricht das
WebAuthn/FIDO2 (Relying-Party-ID), alle Links in E-Mails und Send-URLs. Aufgefallen
nur, weil vor der Migration der Ist-Zustand über die öffentliche API geprüft wurde.

**Der `ADMIN_TOKEN` überlebt Docker Compose nicht ungeschützt.** Details in der
Auto-Memory (`feedback_technical_gotchas`) — kurz: Compose interpoliert `$` auch in
`env_file`-Werten, der Argon2-Token kam mit 88 statt 176 Zeichen an. Der Fehler ist
still: Vaultwarden startet normal, nur `/admin` bleibt gesperrt. Fix ist `$$`-Escaping
plus eine Längenprüfung in der Rolle.

**Die Alt-Installation bleibt als Rollback-Punkt stehen.** `/opt/vaultwarden` ist
unberührt, der systemd-Dienst nur `disabled`. Zurück auf 1.33.2 geht mit
`docker compose down` + `systemctl start vaultwarden`. Nötig ist das, weil 1.37 das
Datenbankschema migriert und die alte Version die Datei danach nicht mehr öffnet —
die Docker-Variante arbeitet deshalb auf einer Kopie unter `/opt/vaultwarden-docker/data`.

**`--check` taugt nicht für „Repo anlegen, dann daraus installieren".** Der Trockenlauf
scheiterte an `No package matching 'docker-ce'`, weil `deb822_repository` im Check-Modus
nichts schreibt. Er hat aber bestätigt, dass der Lauf folgenlos bleibt.

**`meta`-Abhängigkeiten laufen vor den eigenen Tasks der Rolle.** Die Preflight-Prüfung
in `vaultwarden_deploy` feuert erst nach der kompletten Docker-Installation. Für den
nächsten Dienst besser als `pre_tasks` im Playbook.

### Offen

- [x] Altlasten abgeräumt: `/vaultwarden`, `/root/.rustup`, `/root/.cargo` — 2,5 GB frei,
      Belegung von 87 % auf 43 %. Rollback-Punkt (`bin/vaultwarden` + `data`) unberührt.
- [x] `wartung/run_lxc_vaultwarden_updates.yml` gelöscht (funktionslos)
- [x] `mode` auf dem Datenverzeichnis entfernt — meldete bei jedem Lauf `changed`, weil
      `cp -a src/.` die Rechte des Quellverzeichnisses auf das Ziel überträgt
- [x] „Websockets Support" im Nginx Proxy Manager geprüft — war bereits aktiv (`ws=1`),
      Ziel unverändert `192.168.2.17:8000`. Nichts zu tun.

## Session 2026-08-17 — evcc: Doppel-Instanz stillgelegt, Update-Playbook gebaut

### Was getan wurde

- [x] Doppelte evcc-Instanz aufgelöst: **119 (`192.168.2.208`) ist produktiv**, 105 (`.48`)
      gestoppt und `onboot: 0`. Beweis war der identische `chargeTotalImport` (6960.671) in
      beiden `/api/state` — beide steuerten **dieselbe openWB Pro** (`192.168.2.145`).
      Vollständige Beweiskette in [[homelab-infrastruktur]].
- [x] Config + DB von 105 vor dem Stoppen auf dem Proxmox-Host gesichert:
      `/root/evcc-105-stilllegung/` (enthält die Kia-EV6- und Victron-Konfiguration)
- [x] `evcc_lxc ansible_host=192.168.2.208` in `hosts.ini` aufgenommen
- [x] `wartung/check_lxc_evcc_updates.yml` — prüft OS-Updates, `apt-cache policy evcc`
      und ob gerade geladen wird (Update-Fenster offen?)
- [x] `wartung/run_lxc_evcc_updates.yml` — Sicherheitsprüfung → DB-Backup → OS → evcc →
      API-Health-Check → Backup-Rotation
- [x] SSH-Key auf 119 hinterlegt (`/root/.ssh/authorized_keys` war leer), `ansible evcc_lxc
      -m ping` → `pong`
- [x] Beide Playbooks gegen den echten Host gelaufen. **evcc steht jetzt auf 0.314.1**,
      Konfiguration und Historie unbeschädigt: „DIY Akku" 14 kWh, `bufferSoc: 95`,
      Ladepunkt `West_Garage`, `chargeTotalImport` 6960.671 unverändert,
      **91 Ladesessions / 1601,91 kWh** vollständig erhalten. Dienst `active` + `enabled`.
- [x] Playbook-Fehler gefunden und behoben (s. Erkenntnisse): `upgrade: dist` zog evcc
      ungefragt mit hoch. Jetzt `dpkg_selections: hold` in `block`/`always`.
- [x] Commit `198a212` auf `main` gepusht (3 Dateien, 341 Zeilen) — pfadgenau committet,
      damit die offenen Vaultwarden-Änderungen im Index unberührt blieben
- [x] **`ansible-control` (LXC 900) ist wieder die Referenz-Steuerstelle:** RSA-Key
      `root@ansible-control` auf 119 nachgelegt (fehlte, weil 119 nie im Ansible-Setup war),
      Repo dort von Stand 5. Juli auf `198a212` gezogen, `ansible evcc_lxc -m ping` → `pong`,
      Check-Playbook dort mit `changed=0` gelaufen. Ansible-Version dort: `core 2.19.4`,
      alle genutzten Module enthalten.
- [ ] **Offen:** 119 steht auf `dhcp`, HA-iFrame zeigt aber fest auf `.208` → statische IP
      über `lxc_netif` setzen (gleiche Falle wie Grafana am 2026-07-05)
- [ ] **Offen:** 105 nach Bewährungszeit entfernen (`vzdump` + `pct destroy`)
- [ ] **Zu beobachten:** Das API-Schema hat sich mit 0.314 geändert — `battery` ist jetzt ein
      Objekt mit `devices`-Liste statt einer Liste. Betrifft künftige API-Konsumenten
      (Grafana, Node-RED), nicht die Playbooks (die lesen `loadpoints` und `config`).

### Erkenntnisse

**Ein HA-iFrame beweist Sichtbarkeit, nicht Wirksamkeit.** Dass HA auf `.208` zeigt, sagte
nur, welche Oberfläche angeschaut wird — nicht, welche Instanz die Wallbox regelt. Erst der
Vergleich der Live-API beider Instanzen brachte Gewissheit. Als Unterrichtsbeispiel brauchbar:
Beobachtung ≠ Wirkung.

**evcc kann komplett ohne `evcc.yaml` laufen.** Auf 119 ist `"config": ""` — die Einrichtung
liegt in `/var/lib/evcc/evcc.db`, weil sie über die Web-UI erfolgte. Ein `grep` in
`/etc/evcc.yaml` findet dort nichts und lässt den Container fälschlich unkonfiguriert
aussehen. Konsequenz für das Playbook: Das DB-Backup bei gestopptem Dienst ist die einzige
Rückfallebene, `sqlite3` ist auf dem Container nicht installiert.

**Zwei Regler auf einer Wallbox fallen erst unter Last auf.** Beide Ladepunkte standen auf
`mode: off`, deshalb war der Konflikt latent. Das Stoppen von 105 war eine Reparatur, keine
Aufräumaktion.

**Der Doppel-Regler hat die Ladehistorie verfälscht — messbar.** Norbert fiel auf, dass openWB
und evcc unterschiedliche Kosten zeigen: openWB 32,19 kWh / 2,36 €, evcc 29,5 kWh / 2,19 €
für denselben Ladevorgang am 15.08.2026. Die Auflösung:

Der **Preis** ist nicht die Ursache — beide rechnen mit ~0,073 €/kWh, der Einspeisevergütung
(openWB 0,07331, evcc 0,07424, Abweichung 1,3 %). Bei einem Solaranteil von 99,7 % bewertet
evcc die geladene Energie korrekt mit den Opportunitätskosten statt mit dem Netzpreis (0,35).

Die Ursache ist die **Energiemenge**. Der Vergleich derselben Sessions aus beiden Datenbanken
(105 aus der Sicherung, 119 live) zeigt eine systematische Untererfassung von rund 9 %:

| Datum | LXC 105 | LXC 119 | Differenz |
|---|---|---|---|
| 2026-08-15 | 32,43 kWh | 29,46 kWh | −9,2 % |
| 2026-08-10 | 11,22 kWh | 10,38 kWh | −7,5 % |
| 2026-07-28 | 15,04 kWh | 13,39 kWh | −11,0 % |
| 2026-07-26 | 19,10 kWh | 17,55 kWh | −8,1 % |
| 2026-07-23 | 48,36 kWh | 43,94 kWh | −9,1 % |
| 2026-07-16 | 9,67 kWh | 8,53 kWh | −11,8 % |
| **Summe** | **135,82 kWh** | **123,25 kWh** | **−9,3 %** |

**Nicht die Messung ist kaputt, sondern die Energiebuchung.** Der Zählerstandsvergleich
entlastet beide Systeme: vor dem Laden evcc 6928,1 / openWB 6928,48 kWh, danach evcc 6960,7 /
openWB 6960,67 kWh — Abweichung 0,03 bis 0,38 kWh, also Ablesezeitpunkt und Rundung. Beide
lesen den Eastron-Zähler korrekt.

Der Widerspruch steckt **innerhalb eines einzigen evcc-Datensatzes**:

```
created  : 2026-08-15T18:28:41   finished: 2026-08-16T13:12:41
meterStart = 6928.064
meterStop  = 6960.671     -> Delta 32.607 kWh
chargedEnergy = 29.461    -> 3.146 kWh weniger (-9.6 %)
price = 2.1875            -> folgt der zu kleinen chargedEnergy
```

`meterStop` trifft den openWB-Wert exakt. evcc führt `chargedEnergy` aber nicht als
Zählerdifferenz, sondern summiert die Ladeleistung über die Session-Dauer — und in dieser
Summe fehlen die Zeiträume, in denen 119 keine Daten von der Wallbox bekam. Die Session lief
19 Stunden (Fahrzeug über Nacht angesteckt), da summieren sich Aussetzer.

**Kausalität durch die Monatsauswertung belegt** (Zählerdelta gegen gemeldete Energie, alle
91 Sessions):

| Monat | Sessions | Zähler-Delta | gemeldet | Fehler |
|---|---|---|---|---|
| 2026-01 | 5 | 64,48 | 64,15 | −0,5 % |
| 2026-02 | 16 | 275,23 | 273,44 | −0,7 % |
| 2026-03 | 20 | 313,07 | 310,36 | −0,9 % |
| 2026-04 | 12 | 243,13 | 240,31 | −1,2 % |
| **2026-05** | 17 | 330,54 | 314,58 | **−4,8 %** |
| 2026-06 | 11 | 236,77 | 211,27 | −10,8 % |
| 2026-07 | 8 | 165,99 | 147,96 | −10,9 % |
| 2026-08 | 2 | 43,93 | 39,84 | −9,3 % |

Bis April liegt der Fehler in der Messtoleranz. **Beide Container laufen seit dem 18.05.2026
gemeinsam** — Mai ist der Mischmonat (−4,8 %), ab Juni greift der Effekt voll (~10 %). Damit
ist der Doppel-Betrieb als Ursache belegt, nicht nur plausibel.

**Bilanz:** über alle Sessions 1673,15 kWh laut Zähler gegen 1601,91 kWh gemeldet — **71,24 kWh
und rund 10 € zu wenig** (225,28 € gemeldet, 235,30 € auf Delta-Basis).

**Gute Nachricht:** Die korrekten Werte stecken in der Datenbank. `meterStart`/`meterStop` sind
pro Session vorhanden, die Historie ist also nachträglich rekonstruierbar, ohne die Sicherung
von 105 zu brauchen. Ab dem 2026-08-17 (105 gestoppt) sollten `chargedEnergy` und Zählerdelta
wieder zusammenpassen — der nächste Ladevorgang ist die Gegenprobe.

> [!important] Folgethema: Abrechnung gegenüber dem Mieter
> Aus dieser Analyse ergab sich die Frage, ob die openWB Pro überhaupt eichrechtskonform ist
> (Antwort nach derzeitiger Erkenntnis: nein, nur MID). Das ist ein eigenes Vorhaben und steht
> in [[Wallbox_Mieterabrechnung]]. **Für Abrechnungszwecke immer die openWB-Zählerstände
> nehmen, nie die evcc-Session-Historie.**

**`upgrade: dist` hebelt die eigene Update-Choreografie aus.** Der Testlauf sollte nur von
0.300.6 auf 0.300.8 gehen. Weil das OS-Update im Playbook *vor* der Prüfung „steht ein
evcc-Update an?" stand, holte `dist-upgrade` evcc still auf 0.314.1 — vorbei am DB-Backup, am
Ladevorgang-Schutz und am `evcc_zielversion`-Parameter. Die Prüfung danach verglich gegen eine
bereits überschriebene Version und meldete „nichts zu tun". Für ein aus dem APT-Repo
installiertes Programm ist `dist-upgrade` eben nicht wählerisch. Behoben mit
`dpkg_selections: hold` in `block`/`always`, damit die Sperre auch bei Fehlern fällt.

Gut ausgegangen: Der Sprung über 14 Minor-Versionen lief sauber durch, weil das Backup aus dem
vorherigen Lauf existierte und evcc die DB korrekt migriert hat. Der **Test dafür ist der
zweite Lauf** — meldet er eine Versionsänderung, obwohl der Update-Block geskippt wurde,
greift genau dieser Fehler.

## Verwandte Notizen

- [[02 Projekte/homelab-monitoring]] — Betrieb: Flux-Queries, Grafana-Dashboards, Entity-IDs
- [[03 Bereiche/Unterricht/Ansible]] — Rollen-Architektur, Stolperfallen, Unterrichtsbeispiele
- [[02 Projekte/Notenerfassung_IT]] — App die auf `noten_it_lxc` läuft
