---
tags: [ressource, claude, workflow]
status: aktiv
date: 2026-05-24
---

# Neues Projekt anlegen

Workflow zum Aufsetzen eines neuen Code-Projekts mit Vault-Anbindung über einen Core-Symlink.

## Schritte

1. **Projektordner anlegen**
   ```
   mkdir /home/norbert/Code/[Projektname]
   cd /home/norbert/Code/[Projektname]
   ```

2. **Core-Symlink erstellen**
   ```
   ln -s /home/norbert/Dokumente/core core
   ```
   → Das Vault ist danach über den relativen Pfad `core/` aus dem Projekt erreichbar.

3. **`.gitignore` anlegen — Symlink ausschließen**
   ```
   core
   ```
   > [!warning] Niemals den Vault in Git committen!

4. **Vault-Notiz anlegen**
   - Vorlage kopieren: [[04 Ressourcen/Claude/Projektvorlage]]
   - Ablegen unter: `02 Projekte/[Projektname].md`
   - Ausfüllen: Ziel, Stack, Architektur, Agent-Team, Skills, Start-Befehle

5. **Claude Code starten + `/init` ausführen**
   → `CLAUDE.md` wird generiert.

6. **In die generierte `CLAUDE.md` eintragen**
   ```
   Projektbeschreibung: core/02 Projekte/[Projektname].md
   ```

> [!important] Reihenfolge beachten
> Vorlage **vor** dem ersten Claude-Start ausfüllen — sonst fehlt der Projektkontext beim Start.

> [!tip] Aufgaben-Format
> Jede Aufgabe braucht drei Teile, damit Claude autonom arbeiten kann:
> `**Aufgabe:** Was — **Kontext:** Warum — **Ergebnis:** Was soll anders sein`

## Verknüpfungen

- [[04 Ressourcen/Claude/Projektvorlage]] — Vorlage für die Vault-Notiz
- [[04 Ressourcen/Claude/Claude]] — Allgemeine Claude-Ressourcen
- [[03 Bereiche/KI-Softwareentwicklung/KI-Softwareentwicklung]] — Übergeordneter Bereich
