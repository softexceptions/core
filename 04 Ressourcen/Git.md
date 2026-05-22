---
tags:
  - ressource
  - git
  - terminal
status: aktiv
date: 2026-05-23
---

# Git — Referenz

## Aliasse

Git-Aliase verkürzen häufige Befehle. Es gibt zwei Wege sie zu definieren:

### Per Terminal (global)

```
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.lg "log --oneline --graph --decorate --all"
```

### Direkt in `~/.gitconfig`

```
[alias]
    st  = status
    co  = checkout
    br  = branch
    lg  = log --oneline --graph --decorate --all
    undo = reset --soft HEAD~1
    save = stash push -m
```

## Meine Aliasse

```
[alias]
    cleanbrain = !git add . && git commit -m "chore(core): AI cleanup $(date +\"%d.%m.%Y %H:%M\")"
    list       = config --get-regexp '^alias\\.'
```

### Aufrufen

**Vault aufräumen und committen:**

```
git cleanbrain
```

**Alle Aliasse anzeigen:**

```
git list
```

### Alias aufrufen

```
git st        # → git status
git co main   # → git checkout main
git lg        # → schöner Log-Baum
git undo      # → letzten Commit rückgängig (Änderungen bleiben)
git save "WIP vor Meeting"   # → Stash mit Namen
```

> [!tip] Alle Aliasse anzeigen
> ```
> git config --global --list | grep alias
> ```

> [!info] Wo liegt `.gitconfig`?
> Linux/Mac: `~/.gitconfig`
> Windows: `C:\Users\<Name>\.gitconfig`
