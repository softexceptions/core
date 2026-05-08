---
description: Vault-Kontext für Norberts core
---

# Vault Context

Dieses Vault ist Norberts core.

## Über mich

Ich bin Norbert, Informatik Lehrer und Softwareentwickler. Ich unterrichte Anwendungsentwickler:innen und bereite sie gezielt auf KI-gestützte Softwareentwicklung vor. Meine Leidenschaft liegt in der Vermittlung moderner Entwicklungspraktiken und innovativen Lösungen. Ausführliches Profil in [[00 Kontext/Über mich]].

## Vault-Struktur

- **00 Kontext/** — Persönliches Profil (Über mich, Schreibstil, Zielgruppe, Fokus). Zentrale Referenz für alle inhaltlichen Aufgaben. Lies diese Dateien, wenn du Content erstellst, Mails schreibst oder Entscheidungen triffst.
- **01 Inbox/** — Schnelle Gedanken, unverarbeitete Notizen. Alles, was noch keinen festen Platz hat, landet hier.
- **02 Projekte/** — Aktive Projekte mit konkretem Ziel und Enddatum: IHK-Bewertungssoftware, Energielandingpage, Ausbilder-Landingpage. Starten als einzelne .md Datei. Unterordner nur wenn ein Projekt mehrere Dateien braucht.
- **03 Bereiche/** — Laufende Verantwortungsbereiche ohne Enddatum. Jeder Bereich ist ein eigener Ordner:
  - **KI-Softwareentwicklung** — Hobby und berufliche Entwicklung mit KI-Tools
  - **Unterricht** — Lehrplanung, Didaktik, Schüler-Vorbereitung
- **04 Ressourcen/** — Referenzmaterial, Wissen, gesammelte Informationen. Jedes Thema ist ein eigener Ordner:
  - Claude, Skills, Agenten-Engineering, MCP, KI-Auswirkungen
- **05 Daily Notes/** — Tägliches Logbuch. Format: YYYY-MM-DD.md
- **06 Archiv/** — Abgeschlossene Projekte und inaktive Bereiche. Alte Code-Projekte und Cowork-Migration.
- **07 Anhänge/** — Bilder, PDFs, Medien.

## Verfügbare Skills

Installiert in `.claude/skills/`:

| Skill | Verwendung |
|---|---|
| `obsidian-markdown` | Obsidian-Flavored Markdown (Wikilinks, Callouts, Frontmatter, Embeds) |
| `obsidian-cli` | Vault-Operationen: Notizen lesen, erstellen, suchen, verwalten |
| `obsidian-bases` | `.base`-Dateien mit Views, Filtern und Formeln |
| `json-canvas` | `.canvas`-Dateien (Mindmaps, Flowcharts) |
| `defuddle` | Webseiten als sauberes Markdown extrahieren (statt WebFetch) |

## Vault-Regeln

- [[Wikilinks]] für Verknüpfungen zwischen Notizen
- Neue Notizen ohne klaren Platz kommen in 01 Inbox/
- Eine Idee pro Notiz. Daily Notes fassen einen ganzen Tag zusammen.
- YAML Frontmatter nutzen: tags, status (aktiv/abgeschlossen/pausiert), date
- Dateinamen in normaler Schreibweise mit Leerzeichen: Beschreibender Name.md
- Wenn Norbert sagt "merk dir das" → in die passende Kontext-Datei speichern
- Dateien löschen oder überschreiben → erst nachfragen
- **Schreibstil beachten:** Formal, technisch, lehrerisch. Max. 20 Wörter/Satz, Gendern, Aktiv vor Passiv, keine Floskeln.

## Session-Routinen

### Session-Start
Prüfe 01 Inbox/ auf neue Notizen. Zeige, was drin liegt, und biete an, die Einträge einzusortieren.

### Kontext-Briefing
Wenn Norbert fragt "Was war nochmal aktuell?" oder "Wo war ich stehen geblieben?": Lies die letzten 2–3 Daily Notes und die aktiven Projekt-Dateien für ein kurzes Briefing.

### Session-Ende

1. **Daily Note anbieten** — Format: `05 Daily Notes/YYYY-MM-DD.md`
   - Was wurde heute erledigt?
   - Was wurde gelernt?
   - **Einstiegs-Satz für morgen** — konkreter Befehl für den nächsten Session-Start (z.B. „Phase 1 starten", „Weiter mit Feature X")
2. Neue Erkenntnisse in passende Kontext-Dateien speichern
3. Inbox aufräumen

**Warum Daily Notes:** Sie sind die Brücke zwischen Sessions. Ohne Daily Note fehlt der Tagesrhythmus — „Was war aktuell?" greift dann nur auf Projekt-Notizen zurück, nicht auf den konkreten Tagesstand.

### Plan merken

Wenn Norbert sagt „Merke dir den Plan" oder „Merke dir den Plan in meinem core":
1. Lies den aktuellen Plan aus der Claude-Code-Plan-Datei (unter `/home/norbert/.claude/plans/`).
2. Wähle den Ablageort:
   - Projekt-Plan → `02 Projekte/[Projektname]/[Titel].md`
   - Bereichs-Plan → `03 Bereiche/[Bereich]/[Titel].md`
   - Kontext unklar → `01 Inbox/[Titel].md`
3. Erstelle die Notiz mit diesem Format:
   - YAML-Frontmatter: `tags`, `status: aktiv`, `date`
   - Abschnitte: Kontext, Schritte (Checkboxen), Verifikation, Quelle (`[[Claude Code]]`)
4. Verknüpfe die neue Notiz per Wikilink im zugehörigen Projekt oder Bereich.

---

## Besonderheiten dieses Vaults

Norberts Vault verbindet drei Welten:

1. **Berufliche Unterrichtstätigkeit** — Vorbereitung von Schüler:innen auf moderne Softwareentwicklung
2. **Persönliche Entwicklung** — Eigene Experimente mit KI-Tools, Claude, Agenten, Skills
3. **Gesellschaftliche Perspektive** — Reflexion über Auswirkungen von KI auf Beruf und Gesellschaft

Das Vault ist nicht nur Wissenssammlung, sondern auch Werkzeug für Denken, Experimentieren und Lehren.
