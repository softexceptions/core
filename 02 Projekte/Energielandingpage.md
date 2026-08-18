---
tags: [projekt]
status: aktiv
erstellt: 2026-03-22
deployed: true
url: https://energie.softexceptions.com
---

# Energielandingpage

"Die Energierevolution die wir brauchen" — Informationsplattform zu nachhaltiger Energieversorgung und Transformation.

## Ziel

Zugängliche, informative Landingpage aufbauen, die die Energierevolution verständlich darstellt und zum Handeln inspiriert.

## Status

Zunächst abgeschlossen. Kann erweitert werden.
# Link

[Die Energie Revolution](https://energie.softexceptions.com)

## Deployment

Statische Seite (eine `index.html` + `images/` + `favicon.svg`), ausgeliefert per nginx auf einem LXC-Container.

- **Container:** `root@192.168.2.26` (Hostname `Energierevolution`), passwortloser SSH-Zugang per Key `~/.ssh/id_ed25519`
- **Webserver:** nginx, Site `/etc/nginx/sites-available/erneuerbare`, Port **9082**
- **Webroot:** `/var/www/erneuerbare`
- **Lokales Projekt:** `/home/norbert/Code/Projekt-Erneuerbare`
- **Öffentliche URL:** https://energie.softexceptions.com (Reverse-Proxy vor Port 9082)

**Deploy-Befehl** (nur geänderte Dateien via rsync über SSH):

```bash
rsync -avz --checksum /home/norbert/Code/Projekt-Erneuerbare/index.html \
  root@192.168.2.26:/var/www/erneuerbare/index.html
```

Verifikation nach Upload: `curl -sI http://192.168.2.26:9082` → `Content-Length` und `Last-Modified` müssen zur neuen Datei passen.

> [!WARNUNG] Webroot enthält Projekt-Müll
> Das ursprüngliche Deployment hat den **kompletten** Projektordner gespiegelt — im Webroot liegen daher auch `.agents/`, `.claude/`, `docs/` (PDFs), `Readme.odt`, `skills-lock.json`. Diese sind öffentlich abrufbar und sollten aufgeräumt werden. Künftig nur benötigte Dateien deployen.

## Nächste Schritte

- [ ] Nutzer-Feedback sammeln
- [ ] Erweiterungsmöglichkeiten ermitteln
- [ ] Neue Inhalte/Visionen hinzufügen?
- [ ] Partner für Zusammenarbeit finden?

## Notizen

Persönliches Anliegen. Könnte zu öffentlichem Content werden (nicht nur intern).
