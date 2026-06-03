---
tags: [kontext, obsidian, regeln]
status: aktiv
date: 2026-05-25
---

# Obsidian-Regeln

Regeln für das Erstellen und Bearbeiten von Notizen in diesem Vault — gilt für Claude und für mich.

## Code-Blöcke: kein Sprach-Label

**Regel:** Code-Blöcke in Obsidian **immer ohne Sprach-Label** öffnen (nur ` ``` `).

```
✓  ```
   npx mein-befehl
   ```

✗  ```bash
   npx mein-befehl
   ```
```

**Warum:** Mit Sprach-Label zeigt Obsidian das Label-Badge (z.B. „shell") statt dem Copy-Icon. Das Copy-Icon erscheint nur ohne Label.

**Gilt für:** Alle `.md`-Dateien im Vault — egal ob Projektnotizen, Ressourcen oder Daily Notes.
Entdeckt: 2026-05-22

## Verlinkung

- Interne Links immer als Wikilinks: `[[Notizname]]`
- Dateipfad bei Mehrdeutigkeit: `[[Ordner/Notizname]]`
- Externe Links: `[Beschreibung](https://...)`

## Frontmatter

Jede neue Notiz bekommt YAML-Frontmatter mit mindestens:
```
---
tags: [...]
status: aktiv | abgeschlossen | pausiert
date: YYYY-MM-DD
---
```

## Dateinamen

- Normale Schreibweise mit Leerzeichen: `Beschreibender Name.md`
- Keine Unterstriche, keine Bindestriche als Trennzeichen
