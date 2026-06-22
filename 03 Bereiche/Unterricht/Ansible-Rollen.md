---
tags: [unterricht, devops, ansible, infrastruktur]
date: 2026-06-19
---

# Ansible Rollen-Architektur — Unterrichtsbeispiel

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

## Projektstruktur (homelab-ansible)

```
homelab-ansible/
├── hosts.ini                    # Inventar: Proxmox-Host + alle LXC-Container
├── geheimnisse.yml              # Ansible-Vault (verschlüsselt)
├── site.yml                     # Haupt-Playbook — orchestriert die Rollen
├── group_vars/
│   ├── proxmox_host.yml         # Variablen für alle Hosts in [proxmox_host]
│   └── lxc_containers.yml      # SSH-Einstellungen für alle LXC-Container
└── roles/
    ├── proxmox_lxc/
    │   ├── defaults/main.yml    # Standardwerte (überschreibbar in site.yml)
    │   └── tasks/main.yml
    ├── docker_install/
    │   └── tasks/main.yml
    └── app_deploy/
        ├── defaults/main.yml
        └── tasks/main.yml
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

## Unterrichts-Lernziele

- [ ] Unterschied monolithisches Playbook vs. Rollen erklären können
- [ ] `group_vars` vs. `vars` in einem Playbook einordnen
- [ ] Tags für selektives Ausführen einsetzen
- [ ] Idempotenz am Beispiel demonstrieren
- [ ] `.dockerignore` und rsync-Excludes als Sicherheitsnetz verstehen
