---
tags: [skill, rust, solid, clean-architecture]
status: aktiv
date: 2026-05-25
---

# rust-solid — Referenz

Skill für SOLID-Architektur in Rust-Projekten mit Clean Architecture.

## Aufruf

```
/rust-solid
```

## Speicherort (funktionale Skill-Datei)

```
core/.claude/skills/rust-solid/skill.md
```

**Symlink für ein neues Projekt:**

```
ln -s /home/norbert/Dokumente/core/.claude/skills/rust-solid \
      /pfad/zum/projekt/.claude/skills/rust-solid
```

## Drei Schichten

| Schicht | Location | Inhalt |
|---|---|---|
| Domain | `src/domain/` | Structs, Traits (Interfaces), Use Cases, Errors — kein I/O |
| Infrastructure | `src/infrastructure/` | Trait-Implementierungen, HTTP, DTOs, externe Services |
| Application | `src/main.rs`, `src/app.rs` | DI-Setup, CLI-Parsing, Orchestrierung |

## Rust-Besonderheit: Traits statt Interfaces

Rust hat kein `interface`-Keyword. Traits übernehmen diese Rolle:

```
trait IBoardRepository → abstract interface
struct PrometheanClient implements IBoardRepository → concrete impl
```

DI via Generics (statisch, zero-cost):
```
struct ConnectToBoard<R: IBoardRepository, S: IStreamService> { ... }
```

Oder via `Box<dyn Trait>` (dynamisch, flexibel für komplexe Graphen).

## Verwandte Ressourcen

- [[tdd]] — TDD (Rust: cargo test + mockall)
- [[python-solid]] — Python SOLID (Vergleich: ABC/Protocol vs. Trait)
- [[flutter-solid]] — Flutter SOLID (Vergleich: abstract class vs. Trait)
- [[../Agent-Skills-Dev]] — Skill-Entwicklung allgemein
- [[../Skills]] — Übersicht aller Skills
