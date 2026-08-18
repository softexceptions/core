---
tags: [homelab, proxmox, infrastruktur]
---

# Homelab-Infrastruktur

Bestandsaufnahme des Proxmox-Hosts und aller Gäste. Erhoben am **2026-07-27** direkt
aus `pct list` / `qm list` und der Datenbank des Nginx Proxy Managers.

> [!note] Betrieb und Deployment
> Ansible-Rollen, Deployment-Muster und Sessions stehen in [[homelab-ansible]],
> Grafana/InfluxDB-Betrieb in [[homelab-monitoring]].

## Host

| | |
|---|---|
| Adresse | `192.168.2.241` |
| Version | Proxmox VE 8.4.11, Kernel 6.8.12-13-pve |
| CPU / RAM | 12 Kerne, 62 GB (18 GB belegt) |
| `local` | 94 GB, 13,7 % belegt |
| `local-lvm` | 816 GB, 22,0 % belegt |

Reichlich Luft — Plattenknappheit tritt bei einzelnen Containern auf, nicht auf dem Host.

## LXC-Container

Laufend:

| VMID | Name | IP | Cores | RAM | Disk |
|---|---|---|---|---|---|
| 101 | mqtt | 192.168.2.233 | 1 | 512 MB | 2 GB |
| 103 | fhem | 192.168.2.68 | 2 | 2048 MB | 8 GB |
| 104 | node-red | 192.168.2.80 | 1 | 1024 MB | 4 GB |
| 107 | adguard | 192.168.2.69 | 1 | 512 MB | 4 GB |
| 108 | vaultwarden | 192.168.2.17 | 4 | 6144 MB | 6 GB |
| 109 | nginxproxymanager | 192.168.2.78 | 2 | 1024 MB | 4 GB |
| 112 | influxdb | 192.168.2.119 | 2 | 2048 MB | 8 GB |
| 118 | zigbee2mqtt | 192.168.2.147 | 2 | 1024 MB | 8 GB |
| 119 | evcc (produktiv, `evcc_lxc`) | 192.168.2.208 | 1 | 1024 MB | 4 GB |
| 121 | IHKBewertung | 192.168.2.83 | 2 | 8192 MB | 20 GB |
| 122 | Energierevolution | 192.168.2.26 | 1 | 512 MB | 8 GB |
| 123 | excalidraw | 192.168.2.12 | 2 | 3072 MB | 10 GB |
| 124 | LFPraesHems | 192.168.2.246 | 2 | 2080 MB | 10 GB |
| 126 | SchutzHEMS | 192.168.2.79 | 2 | 2048 MB | 10 GB |
| 127 | KiaConnect | 192.168.2.211 | 1 | 512 MB | 8 GB |
| 128 | grafana | 192.168.2.214 | 1 | 1024 MB | 8 GB |
| 900 | ansible-control | 192.168.2.101 | 1 | 1024 MB | 10 GB |
| 901 | noten-it | 192.168.2.6 | 1 | 1024 MB | 10 GB |

Gestoppt:

| VMID | Name | IP laut Config | RAM | Disk |
|---|---|---|---|---|
| 105 | evcc (alt, stillgelegt 2026-08-17) | 192.168.2.48 | 1024 MB | 4 GB |
| 106 | esphome | dhcp | 15360 MB | 16 GB |
| 110 | UnifiController | 192.168.2.205 | 2048 MB | 16 GB |
| 113 | n8n | dhcp | 2048 MB | 6 GB |
| 114 | ollama | 192.168.2.100 | 16384 MB | 35 GB |
| 115 | openwebui | 192.168.2.149 | 8192 MB | 25 GB |
| 116 | redis | dhcp | 1024 MB | 4 GB |
| 117 | n8nTestServer | dhcp | 2048 MB | 16 GB |
| 120 | OpenWB | dhcp | 512 MB | 8 GB |
| 125 | OrdnungHEMS | dhcp | 1024 MB | 10 GB |

## Virtuelle Maschinen

| VMID | Name | Status | RAM |
|---|---|---|---|
| 100 | haos14.2 (Home Assistant) | läuft | 16384 MB |
| 111 | EOSD | gestoppt | 4096 MB |
| 200 | Win11pro | gestoppt | 8192 MB |

Home Assistant ist unter `192.168.2.186` erreichbar.

## Öffentliche Domains (Nginx Proxy Manager)

Erhoben aus `/data/database.sqlite` in LXC 109. Admin-Oberfläche: `http://192.168.2.78:81`.
Von außen laufen alle über `home.softexceptions.com` → `84.188.243.17`.

| Domain | Ziel | Websockets | Aktiv |
|---|---|---|---|
| draw.softexceptions.com | 192.168.2.12:3000 | ✅ | ✅ |
| energie.softexceptions.com | 192.168.2.26:9082 | ✅ | ✅ |
| esphome.softexceptions.com | 192.168.2.137:6053 | ✅ | ❌ |
| gmail.softexceptions.com | 192.168.2.81:5678 | ✅ | ✅ |
| homeassistant.softexceptions.com | 192.168.2.186:8123 | ✅ | ✅ |
| ihkmatrix.softexceptions.com | 192.168.2.83:9080 | ✅ | ✅ |
| kia.softexceptions.com | 192.168.2.211:9081 | ✅ | ✅ |
| lernfeld.softexceptions.com | 192.168.2.246:9080 | ✅ | ✅ |
| n8n.softexceptions.com | 192.168.2.81:5678 | ✅ | ✅ |
| n8ntestserver.softexceptions.com | 192.168.2.63:5678 | ✅ | ✅ |
| notenit.softexceptions.com | 192.168.2.6:80 | ❌ | ✅ |
| ordnung.softexceptions.com | 192.168.2.242:8125 | ✅ | ✅ |
| schutz.softexceptions.com | 192.168.2.79:8080 | ✅ | ✅ |
| vaultwarden.softexceptions.com | 192.168.2.17:8000 | ✅ | ✅ |

## Auffälligkeiten

**Vier Proxy-Ziele gehören zu keinem laufenden LXC.** Die Adressen `192.168.2.81`
(`gmail` und `n8n`), `192.168.2.63` (`n8ntestserver`), `192.168.2.242` (`ordnung`) und
`192.168.2.137` (`esphome`, ohnehin deaktiviert) tauchen in der Container-Liste nicht auf.
Entweder laufen die Dienste auf anderer Hardware, oder die Einträge sind Karteileichen aus
der Zeit vor einem Umzug. Die zugehörigen LXCs (113 n8n, 117 n8nTestServer, 125 OrdnungHEMS)
sind gestoppt und hatten `dhcp` — die alten Leases sind damit verloren.

### Doppelte evcc-Instanz aufgelöst (2026-08-17) ✅

**Zwei Container hießen `evcc`** (105 auf `.48`, 119 auf `.208`), beide liefen — und beide
steuerten **dieselbe Wallbox**. Aufgelöst: **119 ist produktiv, 105 ist seit 2026-08-17
gestoppt** (`onboot: 0`).

Beweiskette:

| | LXC 105 (`.48`) | LXC 119 (`.208`) |
|---|---|---|
| HA-Einbindung | keine | Dashboard `dashboard-evcc`, iFrame `:7070` |
| Konfiguration | `/etc/evcc.yaml` (Datum Aug 2025) | **DB / Web-UI** (`config: ""`) |
| Batterie | ohne Titel, `capacity: 1`, `bufferSoc: 0` | **"DIY Akku", 14 kWh, `bufferSoc: 95`** |
| Ladepunkt | `Garage`, Fahrzeug `Kia_EV6` (YAML) | `West_Garage`, Fahrzeug `db:8` (UI) |
| evcc-Version | 0.300.5 | 0.300.6 |
| `chargeTotalImport` | **6960.671** | **6960.671** ← identisch |

Der identische Zählerstand ist der Beweis: Beide sprachen mit derselben openWB Pro
(`192.168.2.145`) und hätten sich bei aktiver Ladung um den Sollstrom gestritten. Das Stoppen
von 105 war also keine Aufräumaktion, sondern hat einen **latenten Steuerungskonflikt**
beseitigt. Dass es nie aufgefallen ist, lag an `mode: off` auf beiden Ladepunkten.

Die gepflegten Werte (Akkuname, echte Kapazität, `bufferSoc`) belegen 119 als die Instanz, an
der tatsächlich gearbeitet wurde — passend zum HA-iFrame. Beide liefen seit dem 18.05.2026
(gemeinsamer Host-Neustart).

**Gesichert vor dem Stoppen** auf dem Proxmox-Host in `/root/evcc-105-stilllegung/`:
`evcc.yaml.20260817-2057` (die Kia-EV6- und Victron-Konfiguration) und
`evcc.db.20260817-2057`. Nach Bewährungszeit: `vzdump 105` und dann `pct destroy 105`.

Nach dem Stoppen geprüft: 119 läuft, `:7070` liefert HTTP 200, Wallbox `.145` erreichbar.
Am Cerbo sollte jetzt nur noch **eine** `dbus-modbustcp`-Verbindung offen sein (s.
[[victron_node_red]]).

**`notenit` ist der einzige Proxy-Host ohne Websockets.** Für eine Flask-Anwendung ohne
Live-Updates ist das unkritisch — nur bewusst wissen sollte man es.

**DHCP für Dienste mit festem Verwendungszweck bleibt die Hauptfehlerquelle.** Neun
Container stehen auf `dhcp`, davon mehrere mit Proxy-Einträgen oder Inventar-Referenzen.
Der Vorfall vom 2026-07-05 (Grafana wechselte nach einem Neustart die Lease) ist in
[[homelab-ansible]] beschrieben; die Lösung ist dort die Rollen-Variable `lxc_netif`.

> [!warning] Die Ursache liegt im DHCP-Bereich der UDM Pro (Befund 2026-08-17)
> Der DHCP-Range der Dream Machine Professional ist **`192.168.2.1 – 192.168.2.254`** —
> also das komplette Subnetz. Zwei Konsequenzen:
>
> 1. **Keine Fixed-IP-Reservierung möglich.** UniFi akzeptiert eine feste Zuweisung nur
>    außerhalb des verteilten Bereichs; ein „außerhalb" existiert hier nicht. Der Versuch,
>    `.208` zu fixieren, scheitert deshalb (zuletzt mit „try again later", einem
>    Controller-/Provisioning-Fehler, der die eigentliche Sperre nur verdeckt).
> 2. **Netzweites Konfliktrisiko.** Die UDM darf jede Server-Adresse an ein beliebiges neues
>    Gerät vergeben — Proxmox `.241`, Home Assistant `.186`, Cerbo `.181/.182`, openWB Pro
>    `.145`, Grafana `.214`. Dass bisher nichts kollidierte, liegt an stabilen
>    Lease-Erneuerungen, nicht an der Konfiguration. `.1` (das Gateway selbst) liegt
>    ebenfalls im Pool und gehört dort nicht hin.
>
> **Konsequenz für feste Adressen:** statisch in der Container-Config setzen
> (`pct set <vmid> --net0 ...,ip=<adresse>/24,gw=192.168.2.1`), nicht über die UDM.
> So für 119 am 2026-08-17 gemacht.
>
> **Offen als eigenes Vorhaben:** Range auf einen Teilbereich verkleinern (z. B.
> `.100–.199`) und die Server-Adressen darüber/darunter statisch vergeben. Vor dem Umbau
> Bestandsaufnahme nötig — Wallbox `.145` und Cerbo `.181/.182` sind in evcc **fest
> verdrahtet**, ein Lease-Wechsel dort bricht die Ladesteuerung.

**Container mit auffälliger Dimensionierung.** `esphome` (106) hat 15 GB RAM zugewiesen,
`ollama` (114) 16 GB und 35 GB Platte — beide gestoppt. Bei Bedarf lässt sich dort
Reserve zurückgewinnen.

## Verwandte Notizen

- [[02 Projekte/homelab-ansible]] — Deployment, Rollen, Sessions
- [[02 Projekte/homelab-monitoring]] — Grafana, InfluxDB, Flux-Queries
- [[02 Projekte/ipNginx]] — eigenes Nginx-Projekt (nicht der Proxy Manager)
