---
tags:
  - referenz
  - obsidian
aliases:
  - Interne Links
date: 2026-05-14
---

# Wikilinks

Wikilinks sind Obsidians System für interne Verknüpfungen zwischen Notizen. Sie werden automatisch aktualisiert, wenn eine Datei umbenannt wird.

## Syntax

```markdown
[[Notizname]]                        Link zur Notiz
[[Notizname|Anzeigetext]]            Eigener Linktext
[[Notizname#Überschrift]]            Link zu einer Überschrift
[[Notizname#^block-id]]              Link zu einem Block
[[#Überschrift in dieser Notiz]]     Link innerhalb derselben Notiz
```

## Einbettungen

Mit `!` wird der Inhalt direkt eingebettet statt nur verlinkt:

```markdown
![[Notizname]]                       Ganze Notiz einbetten
![[Notizname#Abschnitt]]             Abschnitt einbetten
![[bild.png]]                        Bild einbetten
![[bild.png|400]]                    Bild mit Breite einbetten
```

## Konvention in diesem Vault

- **Wikilinks** für alle internen Vault-Verknüpfungen — Obsidian pflegt sie bei Umbenennung automatisch
- **Markdown-Links** `[Text](URL)` nur für externe URLs
- Neue, noch nicht existierende Notizen dürfen verlinkt werden — Obsidian erstellt sie bei Bedarf

## Beispiele aus diesem Vault

```markdown
[[00 Kontext/Über mich]]             Profil-Notiz
[[02 Projekte/SchulOrdnungHems]]     Aktives Projekt
[[03 Bereiche/Unterricht/Unterricht]] Bereichs-Übersicht
```

> [!tip] Block-IDs definieren
> Einen Block verlinkbar machen: `^block-id` ans Ende des Absatzes hängen, dann mit `[[Notiz#^block-id]]` darauf verweisen.
