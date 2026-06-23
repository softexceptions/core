---
tags: [projekt/aktiv, devops, ansible, proxmox, homelab]
status: aktiv
date: 2026-06-23
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

### ansible-control

Deployments laufen von einem dedizierten LXC-Container (`ansible-control`, 192.168.2.101) — nicht vom Desktop. Der Desktop entwickelt und pusht; ansible-control pullt und deployt.

```
Desktop: git push → GitHub
ansible-control: git pull → ansible-playbook
```

## Aktive Dienste

| Dienst | Playbook | Status |
|---|---|---|
| Notenerfassung IT | `playbooks/noten_it.yml` | deployed, HTTPS aktiv |
| Node-RED | — | manuell eingerichtet (noch kein Playbook) |
| Zigbee2MQTT | — | manuell eingerichtet (noch kein Playbook) |
| Vaultwarden | — | manuell eingerichtet (noch kein Playbook) |

## Verwandte Notizen

- [[03 Bereiche/Unterricht/Ansible]] — Rollen-Architektur, Stolperfallen, Unterrichtsbeispiele
- [[02 Projekte/Notenerfassung_IT]] — App die auf `noten_it_lxc` läuft
