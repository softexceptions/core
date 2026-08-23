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
[[2026-08 Journal]]).

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


---

## Netzwerk-Vorfall 18.08.2026 und Fernzugriff

Beide Abschnitte übernommen aus [[Victron Anlage]] am 23.08.2026 — sie sind dort
über eine Ausfallmeldung des Cerbo-Wächters entstanden, betreffen aber Netzwerk und
Fernzugriff. Die Wächter-Seite des Vorfalls bleibt in [[Cerbo Wächter]].

## Ursache bewiesen: Firmware-Auto-Update des USW-24-PoE (18.08.2026)

Mit dem SSH-Zugang zur UDM Pro ist der Fall abgeschlossen — es war kein Verdacht mehr nötig, die Logzeile steht da:

```
[2026-08-18T03:49:19,414+02:00] <inform-32> WARN dev -
Device USL24PB[9c:05:d6:50:6d:93] will be upgraded to version: 7.5.10.17129,
scheduled: [true], rolling: [false], external: [false]
```

**`scheduled: [true]`** = geplantes Auto-Update, kein Handeingriff. Der lückenlose Ablauf aus `/data/unifi/logs/server.log` und dem UDM-Journal:

| Zeit (lokal) | Ereignis |
|---|---|
| 03:49:17 | `autoupdate-check` findet Firmware **7.5.10.17129** für die USL-Switch-Familie |
| 03:49:19 | Controller plant das Upgrade für `USL24PB` (= **USW-24-PoE**, `192.168.2.108`) ein |
| **03:52:24** | Cerbo `eth0: Link is Down` — der Switch startet nach dem Flashen neu |
| **03:53:44** | Cerbo `Link is Up` (80 s Ausfall) |
| 03:53:50 | **DHCPDISCOVER-Welle**: Switch selbst, `einstein` (Cerbo), `Reolink`, `openWBPro`, `Victron-VM-3P75CT` (Gridmeter), diverse Handys; der U6-Pro-AP verbindet sich neu mit dem Controller (`connected_at`) |
| 03:54:25 | Controller provisioniert den Switch neu (`provisioned_at`) |

Die Firmware in der Gerätedatenbank steht jetzt auf `7.5.10.17129` — das Update lief also durch.

**Warum ein DHCPDISCOVER des Switches der Kronzeuge ist:** Ein `DHCPREQUEST` ist eine normale Lease-Erneuerung, ein **`DHCPDISCOVER` dagegen ein Neuanfang ohne jede Zustandskenntnis**. Ein Switch macht das nur nach einem Neustart. Dass er in derselben Sekunde auftritt wie der aller angeschlossenen Geräte, schließt „einzelnes Gerät spinnt" endgültig aus.

**Damit ist auch der 29.07. geklärt:** `uptime -s` der UDM meldet Boot am **2026-07-29 03:45:23** — genau die Minute, in der der Proxmox-Host seine drei Link-Flaps sah. Das war das Update der UDM selbst. Beide bisher rätselhaften Nachtereignisse sind derselbe Mechanismus.

### Die eigentliche Betriebsfrage

Der Cerbo hängt am USW-24-PoE. **Jedes künftige Switch-Firmware-Update kostet die ESS-Regelung rund 80 s** und erzeugt eine Pushover-Ausfallmeldung. Nachts bei Grundlast folgenlos — aber es ist gut, den Auslöser zu kennen, statt jedes Mal Forensik zu betreiben.

Mitbetroffen bei jedem Switch-Neustart (aus der DHCP-Welle): Gridmeter `.182`, openWB Pro `.145`, Reolink `.95`, der U6-Pro-AP `.234` (PoE) und damit alle daran hängenden WLAN-Clients.

**Nicht betroffen:** Proxmox `.241` samt aller LXC/VMs — er hängt direkt an der UDM (Port 1, laut Log `PORT_NOT_POE_CAPABLE`), nicht am Switch. Genau deshalb liefen HA, Z2M und MQTT lückenlos weiter; die Asymmetrie aus der Erstdiagnose ist damit nachträglich erklärt.

**Stellschraube:** Das Auto-Update lässt sich in der UniFi-UI unter *Settings → System → Updates* abschalten oder auf ein Zeitfenster legen. Die Einstellung liegt auf UniFi-OS-Ebene, nicht in der `setting`-Collection der Network-App (dort steht nur `super_mgmt` mit `live_updates: auto`). Abwägung: Automatische Switch-Updates bringen Sicherheitsfixes ohne Zutun, kosten aber je Update ~80 s Netz. Bei einer Anlage ohne Notstromfunktion am AC-Out ist das vertretbar — nach dem Whole-home-Umbau lohnt die Frage neu.

### UDM-SSH: Runbook (Stand 18.08.2026)

**Der Zugang ist im Normalbetrieb ABGESCHALTET** und wird nur bei Bedarf aktiviert. Begründung: Der Nutzen ist rein diagnostisch und fällt selten an; `sshd` lauscht auf `0.0.0.0:22`, also auch auf dem WAN-Interface, und wird ausschliesslich von der Firewall gedeckt. Die hält zwar nachweislich dicht (Kette `UBIOS_WAN_LOCAL_USER` endet im DROP, über 463.000 verworfene Pakete), aber ein Dienst, der nicht läuft, braucht keine Firewall.

**Aktivieren:** UniFi-UI → *Settings → Control Plane → Console* → SSH einschalten. Danach genügt `ssh udm`.

**Danach wieder abschalten** — derselbe Schalter.

| Was | Wo |
|---|---|
| Alias `udm` | `~/.ssh/config` → `root@192.168.2.1`, `IdentitiesOnly yes` |
| Schlüssel | `~/.ssh/id_ed25519_udm` (ed25519, **ohne Passphrase**) |
| Auf der UDM | `/root/.ssh/authorized_keys` |

Key und Config-Eintrag bleiben beim Abschalten des Dienstes erhalten — nach dem Wiedereinschalten sollte `ssh udm` also sofort funktionieren. ⚠️ **Nicht verifiziert:** ob UniFi OS die `authorized_keys` bei einem Firmware-Update (oder beim Umschalten) verwirft. Falls doch wieder nach einem Passwort gefragt wird, einmal wiederholen:

```
ssh-copy-id -i ~/.ssh/id_ed25519_udm.pub udm     # nur im echten Terminal!
```

**Drei Stolperfallen, die 20 Minuten gekostet haben:**

1. **Die UniFi-UI hat kein Key-Feld für die Konsole** — nur Benutzer und Passwort. Das Key-Feld unter *Network → Settings → System → Device SSH Authentication* gilt den **verwalteten Geräten** (Switch, AP), nicht der UDM. Keys müssen deshalb per `ssh-copy-id` hinterlegt werden.
2. **Passwort-Logins brauchen ein echtes TTY.** In der Claude-Code-Session — und generell bei gesetztem `DISPLAY` ohne installiertes `ssh-askpass` — erscheint gar kein Prompt, sondern sofort `Permission denied (publickey,keyboard-interactive)`. Das sieht wie eine Fehlkonfiguration aus, ist aber nur die fehlende Eingabemöglichkeit. `ssh-copy-id` gehört in ein normales Terminalfenster.
3. **Kein Benutzername in der UI = `root`.** Fehlt das Feld, ist der Default gemeint. Ein neu gesetztes Passwort legt den SSH-Account zuverlässig an; der blosse Schalter genügt nicht immer.

**Auth-Lage** (für spätere Bewertung): `PermitRootLogin yes`, `PasswordAuthentication no`, aber `ChallengeResponseAuthentication yes` — der Passwort-Login läuft also über PAM/keyboard-interactive. Die `sshd_config` zu härten lohnt nicht: Sie ist auf UniFi OS nicht persistent und beim nächsten Firmware-Update wieder überschrieben.

**Was der Zugang liefert** (Quellen, die es über die UI nicht gibt):
- `/data/unifi/logs/server.log` — die eigentliche Historie (Firmware-Upgrades, Provisioning). Format `[2026-08-18T03:49:19,414+02:00]`, **Grep-Muster braucht das `T`**, sonst leeres Ergebnis.
- `journalctl` — Zeitzone **Europe/Berlin** (der Cerbo läuft dagegen auf UTC). `dnsmasq-dhcp`-Zeilen verraten Netzausfälle: **`DHCPDISCOVER` = Gerät ohne Zustand, also neu gestartet; `DHCPREQUEST` = normale Lease-Erneuerung.**
- MongoDB `127.0.0.1:27117`: `mongo --quiet --port 27117 ace --eval '...'`, Collection `device` (`version`, `connected_at`, `provisioned_at`). **`event` ist bei UniFi OS 5.1.26 leer** — die Historie steckt im server.log.
- ⚠️ `db.setting` mit key `mgmt` enthält `x_api_token` und `x_mgmt_key` im Klartext. Bei Abfragen gezielt projizieren, nicht das ganze Dokument ausgeben.


## Runbook: Home Assistant von außen (NPM-Proxy, Alexa-Skill)

**Stand 19.08.2026.** Anlass: `homeassistant.softexceptions.com` war unerreichbar, Chrome bot beim Login fremde Anmeldedaten an.

### Aufbau

`homeassistant.softexceptions.com` → WAN `87.158.194.87` → Nginx Proxy Manager **192.168.2.78** (LXC, Debian 12, NPM 2.12.6, Admin-UI Port 81) → HA `192.168.2.186:8123`. Proxy-Host-ID **1**, generierte Datei `/data/nginx/proxy_host/1.conf`, Zertifikat **npm-4**. SSH: **im Normalbetrieb kein Key hinterlegt** — der dedizierte Key für die Reparatur am 19.08.2026 wurde danach wieder entfernt, `/root/.ssh/authorized_keys` ist leer. Zugang bei Bedarf über die Proxmox-Console des LXC (dort Key neu hinterlegen), analog zum UDM-Verfahren.

### Die Härtung (Dauerzustand)

HA ist von außen **deny-by-default**. Im NPM-Advanced-Feld des Proxy-Hosts:

```
if ($request_uri !~* "^/(api/alexa/smart_home|auth/token|\.well-known/acme-challenge/)") {
    return 444;
}
```

Damit sind exakt drei Dinge erreichbar: die Alexa-Sprachbefehle, Amazons Token-Erneuerung und die Zertifikatsvalidierung. Frontend, Login und REST-API bleiben unsichtbar. Kontrolle von außen: die beiden Alexa-Pfade müssen **405** liefern (kommt von HA, nur POST erlaubt), alles andere **000** (= 444).

### Was am 19.08.2026 kaputt war

`\.well-known/acme-challenge/` fehlte in der Ausnahme. Ein `if` auf Server-Ebene läuft in der *rewrite*-Phase, also **vor** der Location-Auswahl — es erschlägt damit auch die ACME-Location, die in der Config sichtbar darüber steht. Let's Encrypt bekam stündlich `301` auf HTTP und `444` auf HTTPS; die Erneuerung scheiterte ab Mitte Juli lautlos, am **12.08.2026** lief das Zertifikat ab. Folge: Alexa tot (Amazon akzeptiert kein abgelaufenes Zertifikat), Browser-Login unbrauchbar. Letzter erfolgreicher Alexa-Kontakt 12.08. 01:13 Uhr.

Diagnose-Signatur fürs nächste Mal: **Reset nach ~7 ms** statt 502 nach 60 s; Port 80 antwortet host-spezifisch (301), Port 443 gar nicht; alle *anderen* NPM-Zertifikate sind gültig.

### Runbook: Kontoverknüpfung erneuern (Linking-Fenster)

Nur nötig, wenn die Verknüpfung wirklich weg ist — im Normalfall überlebt sie alles, da Amazon per Refresh-Token arbeitet. Prüfen lässt sich das an `POST /auth/token → 200` von `54.240.197.x` im Log.

1. **Advanced-Feld** temporär erweitern:
   ```nginx
   if ($request_uri !~* "^/(api/alexa/smart_home|auth/token|auth/authorize|auth/providers|auth/login_flow|frontend_latest/|static/|manifest\.json|\.well-known/acme-challenge/)") {
       return 444;
   }
   ```
2. **Alexa-App** (Handy genügt, kein Echo im Raum nötig): Mehr → Skills & Spiele → Meine Skills → Reiter **„Dev"** → HA-Skill → Zur Verwendung aktivieren. Die Developer Console zeigt nur den Statustext „Kontoverknüpfung erforderlich" und hat **keinen** Startknopf — der Vorgang läuft ausschließlich über die App. Nach einem Fehlversuch die App **komplett schließen**, sie hält den alten Fehlerzustand fest.
3. **Sofort wieder** auf die enge Regel zurück.

Erfolgssignatur im Log (19.08.2026, 19:27:40): `POST /auth/token → 200` von Amazon, danach mehrere `POST /api/alexa/smart_home → 200`, darunter einer mit **~5 KB** — das ist die Geräte-Discovery.

Der Skill ist korrekt auf die **EU-Region** konfiguriert (`client_id=https://layla.amazon.com/`). Die vielerorts kopierte `pitangui.amazon.com` ist die US-Adresse und lässt das Verknüpfen in Europa scheitern.

### Erledigt am 19.08.2026

- ⚠️ **Korrektur: der `http:`-Block gehört NICHT mehr in die `configuration.yaml`.** Ab HA 2026.x ist die HTTP-Konfiguration in `/config/.storage/http` migriert (`"yaml_migration_done": true`, hier am 06.08.2026 passiert) und wird per UI unter *Einstellungen → System → Netzwerk* gepflegt; ein YAML-Block wird **ignoriert** und erzeugt eine Reparatur-Meldung. Die vermeintliche Lücke „`http:` fehlt in der aktiven configuration.yaml" war ein Fehlschluss — die Werte lebten längst in der migrierten Form: `use_x_forwarded_for: true`, `trusted_proxies: 192.168.2.78/32, 127.0.0.1/32, ::1/128`, `ip_ban_enabled: true`. Der am 19.08. ergänzte Block wurde wieder entfernt (Sicherungen `configuration.yaml.bak-20260819-194102` und `.bak2-*`). **Restbefund erledigt (19.08.2026):** `login_attempts_threshold` stand auf `-1` (automatische Sperre AUS; `ip_ban_enabled: true` allein aktiviert nur die manuelle `ip_bans.yaml`) und ist jetzt **5**. Gesetzt per WebSocket, da die Netzwerk-UI das Feld nicht anzeigt: `http/config` (lesen) → `http/config/configure` mit allen Feldern in einem `config:`-Wrapper → HA antwortet `{"restart": true}`, bindet den Webserver neu (Sekunden, kein HA-Neustart) und befördert die Konfiguration nach erfolgreichem Rebind selbst zu `stable`. Kontrolle: `http/config` zeigt `login_attempts_threshold: 5` unter `stable`, `trusted_proxies` unverändert.
- ✅ **`worker_connections` auf 2048 angehoben** — update-sicher über die von NPM vorgesehenen Einhängepunkte: `/data/nginx/custom/events.conf` (`worker_connections 2048;`) und `/data/nginx/custom/root_top.conf` (`worker_rlimit_nofile 8192;`, nötig weil der Container `ulimit -n` = 1024 hat). Das Verzeichnis `/data/nginx/custom/` existierte vorher nicht. Per `nginx -s reload` übernommen, ohne Aussetzer; neue Worker zeigen `Max open files 8192`.

### Offene Punkte

- ⬜ **Sprachbefehl-Test steht aus** — die Geräte-Discovery lief am 19.08. sauber durch (5-KB-Antwort), ob die Geräte nach einer Woche Ausfall auch schalten, zeigt erst der Betrieb. Bei „Gerät reagiert nicht“ hilft „Alexa, suche nach neuen Geräten“.

## Verwandte Notizen

- [[02 Projekte/homelab-ansible]] — Deployment, Rollen, Sessions
- [[02 Projekte/homelab-monitoring]] — Grafana, InfluxDB, Flux-Queries
- [[02 Projekte/ipNginx]] — eigenes Nginx-Projekt (nicht der Proxy Manager)
- [[02 Projekte/Victron Anlage/Cerbo Wächter|Cerbo Wächter]] — hat den Ausfall vom 18.08. gemeldet
- [[02 Projekte/Victron Anlage/Victron Anlage|Victron Anlage]] — PV-/Speicheranlage, Node-RED-Flows
