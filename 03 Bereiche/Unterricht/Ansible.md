---
tags: [unterricht, devops, ansible, infrastruktur]
date: 2026-06-19
---

# Ansible

Entstanden beim Deployment der [[02 Projekte/Notenerfassung_IT|Notenerfassung IT]] auf Proxmox.

## Kernidee: Warum Rollen?

Monolithische Playbooks werden schnell unübersichtlich und sind nicht wiederverwendbar. Rollen sind das Ansible-Äquivalent zu Funktionen oder Klassen — jede Rolle hat **eine Aufgabe**, ist **parametrisierbar** und **wiederverwendbar**.

```
# Ohne Rollen: alles in einem Playbook
- name: LXC erstellen
  ...200 Zeilen...
- name: Docker installieren
  ...
- name: App deployen
  ...

# Mit Rollen: klar getrennt
roles:
  - proxmox_lxc     # kann für JEDEN neuen LXC wiederverwendet werden
  - docker_install  # kann für Vaultwarden, Node-RED etc. wiederverwendet werden
  - app_deploy      # spezifisch für diese App
```

## Projektstruktur (homelab-ansible) — Stand 2026-06-23

```
homelab-ansible/
├── ansible.cfg                  # roles_path + inventory — Ansible findet alles automatisch
├── hosts.ini                    # Inventar: Proxmox-Host + alle LXC-Container
├── geheimnisse.yml              # Ansible-Vault (verschlüsselt)
├── site.yml                     # Nur noch import_playbook-Einträge
├── group_vars/
│   ├── proxmox_host.yml         # Variablen für alle Hosts in [proxmox_host]
│   └── lxc_containers.yml      # SSH-Einstellungen für alle LXC-Container
├── playbooks/
│   └── noten_it.yml             # Alle Phasen für die Notenerfassungs-App
└── roles/
    ├── proxmox_lxc/
    │   ├── defaults/main.yml    # Standardwerte (vmid, memory, disk …)
    │   └── tasks/main.yml
    ├── docker_install/
    │   ├── handlers/main.yml    # "Docker starten" — nur bei Erstinstallation
    │   └── tasks/main.yml
    └── app_deploy/
        ├── defaults/main.yml    # app_repo, app_version, app_dest, .env-Werte
        ├── meta/main.yml        # dependencies: docker_install
        └── tasks/main.yml       # git clone + .env erstellen + docker compose up
```

## Tags — selektives Ausführen

```bash
# Nur Docker installieren (ohne LXC neu erstellen):
ansible-playbook site.yml -i hosts.ini --tags docker

# Nur App neu deployen (z.B. nach Code-Änderung):
ansible-playbook site.yml -i hosts.ini --tags app

# LXC erstellen braucht Vault-Passwort, docker/app nicht:
ansible-playbook site.yml -i hosts.ini --tags lxc --ask-vault-pass
```

> [!important] Unterrichts-Diskussionspunkt
> Warum braucht `--tags docker` kein `--ask-vault-pass`?
> Weil `vars_files` in `site.yml` beim Play-Start ausgewertet werden — auch für übersprungene Plays.
> Lösung: Secrets per `include_vars` in die Rolle selbst laden, nicht ins Haupt-Playbook.

## group_vars — Variablen nach Gruppe trennen

`group_vars/lxc_containers.yml` gilt automatisch für **alle Hosts** in der `[lxc_containers]`-Gruppe in `hosts.ini`:

```yaml
# group_vars/lxc_containers.yml
ansible_user: root
ansible_python_interpreter: auto_silent
ansible_ssh_private_key_file: "/home/norbert/.ssh/id_rsa"
```

Das ersetzt die verstreuten `vars:` und `ansible_*`-Parameter in jedem Playbook.

## Defaults vs. Vars — Priorität in Ansible

```yaml
# roles/proxmox_lxc/defaults/main.yml (niedrigste Priorität)
lxc_vmid: 900
lxc_memory: 1024

# site.yml vars: (höhere Priorität — überschreibt defaults)
vars:
  lxc_vmid: 901
  lxc_memory: 2048
```

`defaults/` sind Dokumentation und Fallback-Werte. `vars:` im Playbook überschreiben sie.

## Stolperfallen (aus der Praxis)

### 1. apt_key existiert in Debian 13 nicht mehr

```yaml
# FALSCH (deprecated, auf Debian 13 gar nicht vorhanden):
- ansible.builtin.apt_key:
    url: https://download.docker.com/linux/debian/gpg

# RICHTIG (modernes deb822-Format):
- ansible.builtin.get_url:
    url: https://download.docker.com/linux/debian/gpg
    dest: /etc/apt/keyrings/docker.asc

- ansible.builtin.deb822_repository:
    name: docker
    uris: https://download.docker.com/linux/debian
    suites: bookworm   # Debian 13 (Trixie) hat noch kein eigenes Docker-Repo
    signed_by: /etc/apt/keyrings/docker.asc
```

### 2. rsync überträgt Datenbankdateien mit

`ansible.posix.synchronize` (rsync) überträgt **alles** im Quellverzeichnis — inklusive lokaler `.db`-Dateien.

Docker-Volume-Initialisierung: Wenn ein Volume beim ersten Mount leer ist, kopiert Docker den Image-Inhalt hinein. Eine mitgebackene `notenerfassung.db` wird so zur "Produktionsdatenbank".

```yaml
# Immer explizit ausschließen:
rsync_opts:
  - "--exclude=*.db"
  - "--exclude=*.sqlite"
  - "--exclude=backend/data/"
  - "--exclude=.env"
```

Und im `backend/.dockerignore`:
```
data/
*.db
*.sqlite
.env
```

### 3. Idempotenz — Ansible-Grundprinzip

Ansible-Tasks sind **idempotent**: Mehrfaches Ausführen ändert nichts, wenn der Zielzustand bereits erreicht ist. `state: present` heißt "stelle sicher dass es vorhanden ist", nicht "installiere es nochmal".

```
1. Lauf: changed=3  (Docker installiert, Repo hinzugefügt, Dienst gestartet)
2. Lauf: changed=0  (alles bereits vorhanden — kein Eingriff)
```

Das macht Ansible ideal für **wiederholbare Deployments** und **Infrastructure as Code**.

---

## Erweiterung: Handlers, Meta-Abhängigkeiten & import_playbook

*Erarbeitet 2026-06-23 — vollständiger Umbau auf professionelle Struktur*

| Vorher | Nachher |
|---|---|
| Alles in `site.yml` | `site.yml` + `playbooks/` pro Dienst |
| rsync vom Desktop | `git clone` aus GitHub |
| Kein Handler | Handler für idempotentes Docker-Setup |
| Keine Meta-Abhängigkeiten | `app_deploy` hängt explizit von `docker_install` ab |
| Hardcodierter SSH-Pfad | Plattformunabhängig |
| API-Key in `docker-compose.yml` | Verschlüsselt im Ansible Vault |
| Deployment nur vom Desktop | Deployment vom `ansible-control`-LXC — 24/7 erreichbar |

### Handlers — nur bei Änderung ausführen

Ein Handler ist ein Task der nur läuft wenn er explizit per `notify` benachrichtigt wurde — und selbst dann nur **einmal**, am Ende des Plays.

```yaml
# roles/docker_install/tasks/main.yml
- name: Docker installieren
  ansible.builtin.apt:
    name: [docker-ce, docker-ce-cli, containerd.io, docker-compose-plugin]
    state: present
  notify: Docker starten        # ← nur bei Erstinstallation

# roles/docker_install/handlers/main.yml
- name: Docker starten
  ansible.builtin.systemd:
    name: docker
    enabled: yes
    state: started
```

**Warum:** Ohne Handler startet Docker bei jedem Playbook-Aufruf neu — auch wenn sich nichts geändert hat. Das unterbricht laufende Container unnötig.

### Meta-Abhängigkeiten — explizite Role-Dependencies

```yaml
# roles/app_deploy/meta/main.yml
dependencies:
  - role: docker_install
```

`app_deploy` kann jetzt nie ohne Docker laufen — egal wie das Playbook aufgerufen wird. Die Abhängigkeit ist im Code, nicht nur im Kopf des Entwicklers.

### import_playbook — skalierbare Struktur

```yaml
# site.yml — bleibt immer schlank
- import_playbook: playbooks/noten_it.yml
- import_playbook: playbooks/vaultwarden.yml   # jeder Dienst bekommt sein Playbook
```

**Faustregel:** Wenn `site.yml` mehr als ~30 Zeilen hat, ist `import_playbook` fällig.

### ansible.cfg — Konventionen statt Parameter

```ini
# ansible.cfg im Projektverzeichnis
[defaults]
inventory  = hosts.ini
roles_path = roles
```

Ohne `ansible.cfg` sucht Ansible Roles relativ zum Playbook-Verzeichnis — bei `playbooks/noten_it.yml` wäre das `playbooks/roles/` statt `roles/`. Das führt zu Fehlern die verwirren.

---

## Architektur: ansible-control als Deployment-Node

### Single Source of Truth

Deployments kommen von **einem** Ort — dem `ansible-control`-LXC (192.168.2.101). Nie von zwei Orten gleichzeitig:

```
Desktop: Code schreiben → git push
ansible-control: git pull → ansible-playbook
```

Zwei Deployment-Quellen erzeugen inkonsistente Zustände — die häufigste Ursache für "funktioniert bei mir, nicht auf dem Server".

### git-basiertes Deployment statt rsync

```yaml
# Vorher: rsync vom Desktop (Desktop-Pfad hardcodiert)
- ansible.posix.synchronize:
    src: "/home/norbert/Code/Notenerfassung_IT/"
    dest: "{{ app_dest }}/"

# Nachher: git clone/pull aus GitHub (ortsunabhängig)
- ansible.builtin.git:
    repo: "{{ app_repo }}"
    dest: "{{ app_dest }}"
    version: "{{ app_version }}"
    force: yes
    accept_hostkey: yes
    key_file: "{{ app_deploy_key }}"
```

---

## Stolperfallen aus der Praxis (Session 2026-06-23)

### 4. Hardcodierter SSH-Key-Pfad in group_vars

```yaml
# FALSCH — funktioniert nur auf dem Desktop:
ansible_ssh_private_key_file: "/home/norbert/.ssh/id_rsa"

# RICHTIG — weglassen, Ansible nutzt ~/.ssh/id_rsa als Standard:
ansible_user: root
ansible_python_interpreter: auto_silent
```

### 5. ansible.builtin.git hängt bei nicht-git Verzeichnis

Wenn `dest` existiert aber kein Git-Repo ist (z.B. nach rsync-Deployment), hängt der git-Task.

**Lösung:** Verzeichnis vorher umbenennen:
```bash
ssh root@<host> "mv /opt/app /opt/app-backup"
```

### 6. Privates GitHub-Repo braucht SSH Deploy Key

HTTPS-Clones auf private Repos hängen — git wartet auf Zugangsdaten die nie kommen.

**Lösung: GitHub Deploy Key**
```bash
# Auf dem Managed Node (wo git clone läuft):
ssh-keygen -t ed25519 -f ~/.ssh/github_deploy -N "" -C "deploy key"
cat ~/.ssh/github_deploy.pub
# → Public Key in GitHub: Repo → Settings → Deploy keys
```

```yaml
# In der Role:
app_repo: "git@github.com:user/repo.git"   # SSH-URL, nicht HTTPS
app_deploy_key: "/root/.ssh/github_deploy"
```

### 7. Secrets niemals in docker-compose.yml

```yaml
# GEFÄHRLICH — landet in GitHub:
environment:
  - API_KEY=meinGeheimesPasswort

# RICHTIG — aus .env lesen:
environment:
  - API_KEY=${API_KEY}
```

Die `.env` wird von Ansible erzeugt (nicht in Git), der Wert kommt aus dem Ansible Vault (`geheimnisse.yml`).

### 8. Docker Named Volumes überleben Deployments

```yaml
# docker-compose.yml
volumes:
  - db-data:/app/data    # Named Volume — unabhängig vom Code-Verzeichnis
```

`docker compose up -d --build` überschreibt den Code, lässt aber Named Volumes unberührt. Daten sind sicher — solange kein `docker compose down -v` ausgeführt wird.

---

## Neuer Workflow ab 2026-06-23

```
1. Desktop: Code ändern
   git push origin main

2. ansible-control (192.168.2.101):
   ansible-playbook playbooks/noten_it.yml --ask-vault-pass --tags app
```

Der `ansible-control`-Container holt den App-Code automatisch aus GitHub (`git clone`/`git pull`) und deployt auf den `noten_it_lxc` (192.168.2.6). Der Desktop ist nur noch Entwicklungsumgebung — kein Deployment mehr von dort.

---

## Automatisches Deployment nach git push (noch nicht eingerichtet)

Drei Wege — von einfach bis vollautomatisch:

| Weg | Wie | Voraussetzung |
|---|---|---|
| **Cron Job** | ansible-control prüft regelmäßig (z.B. nachts) | Nichts — sofort umsetzbar |
| **Webhook** | GitHub schickt HTTP-Anfrage bei Push | ansible-control von außen erreichbar |
| **GitHub Actions** | GitHub SSHt nach Push auf ansible-control | SSH-Zugang + Vault-Passwort als Secret |

Für das Homelab empfohlen: erst **Cron Job**, später **Webhook** da `notenit.softexceptions.com` bereits eine öffentliche Domain hat.

---

## Unterrichts-Lernziele

- [ ] Unterschied monolithisches Playbook vs. Rollen erklären können
- [ ] `group_vars` vs. `vars` in einem Playbook einordnen
- [ ] Tags für selektives Ausführen einsetzen
- [ ] Idempotenz am Beispiel demonstrieren
- [ ] Handler erklären: wann läuft ein Handler, wann nicht?
- [ ] Meta-Abhängigkeiten: warum explizit statt implizit?
- [ ] import_playbook: ab wann sinnvoll?
- [ ] Single Source of Truth beim Deployment begründen
- [ ] GitHub Deploy Key vs. Personal Access Token abwägen
- [ ] Docker Named Volumes vs. Bind Mounts unterscheiden
