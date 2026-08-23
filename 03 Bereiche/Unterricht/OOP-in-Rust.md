---
tags: [bereich, unterricht, programmiersprachen, rust, oop]
date: 2026-06-22
---

# OOP in Rust — Kapselung & Traits statt Vererbung

Lehreinheit zur Frage: **Ist Rust objektorientiert?** Konkretisiert am Projekt [[StreamPrometh]]. Verwandt mit [[Sprachwahl-begründen-StreamPrometh]].

## Lernziel

Die Schüler:innen erkennen, dass „OOP" **kein Monolith** ist, sondern mehrere trennbare Konzepte bündelt. Sie lernen, dass eine Sprache objektorientiertes *Design* unterstützen kann, ohne eine „OOP-Sprache" im Java-Sinn zu sein — und warum Rusts Verzicht auf Vererbung kein Mangel, sondern eine bewusste Entscheidung ist.

## Kernantwort

> Rust ist **keine OOP-Sprache im klassischen Sinn**, aber man kann darin sauber objektorientiert **entwerfen** — über Kapselung und Trait-Polymorphie statt über Vererbung. Rust ist multiparadigmatisch (OOP + funktional).

## „OOP" in seine Bestandteile zerlegt

Der Begriff vermischt vier Konzepte. Rust erfüllt drei, lehnt eines bewusst ab:

| OOP-Konzept | Rust? | Wie |
|---|---|---|
| **Kapselung** (Daten + Verhalten zusammen, Internes verstecken) | ✅ | `struct` + `impl`, `pub`/privat, Module |
| **Abstraktion** (Interface statt Implementierung) | ✅ | `trait` als Interface |
| **Polymorphie** (ein Interface, viele Typen) | ✅ | Generics (statisch) + `dyn Trait` (dynamisch) |
| **Vererbung** (`class B extends A`) | ❌ | stattdessen Komposition + Traits |

> Diese Einordnung steht sinngemäß im offiziellen Rust-Buch, Kapitel „Is Rust an Object-Oriented Programming Language?" — nach der Gang-of-Four-Definition (Objekt = Daten + Operationen) qualifiziert sich Rust; nach dem Java/C#-Verständnis (Vererbungshierarchien) nicht.

## Was Rust hat — mit Code

**Kapselung** — Verhalten an der Datenstruktur, Felder privat:
```
pub struct DiscoveryClient {
    http: reqwest::Client,   // privat — von außen unsichtbar
    base_url: String,
}
impl DiscoveryClient {
    pub fn new() -> Self { ... }              // Konstruktor
    pub fn client_id(&self) -> &str { ... }   // kontrollierter Zugriff
}
```

**Abstraktion** — Traits sind die Interfaces:
```
pub trait DiscoveryService {
    fn check_panel(&self, pin: &str) -> impl Future<Output = Result<Panel>>;
}
```

**Polymorphie — Rust hat ZWEI Sorten** (Java nur die zweite):
- *Statisch* via Generics (Compile-Zeit, „monomorphisiert", kein Overhead):
  ```rust
  pub struct Streamer<C: ScreenCapture, P: PipelineFactory> { ... }
  ```
- *Dynamisch* via Trait-Objekt (`dyn`, vtable — Pendant zu virtuellen Methoden):
  ```rust
  Arc::clone(&video_track) as Arc<dyn TrackLocal + Send + Sync>
  ```
  → Rust **lässt wählen**: Geschwindigkeit (Generics) vs. Heterogenität/Flexibilität (`dyn`).

## Was Rust weglässt — und wodurch es das ersetzt

Keine Implementierungsvererbung (`struct B extends A` gibt es nicht). Ersatz:

| Wofür man in Java Vererbung nutzt | Rust-Ersatz |
|---|---|
| Code-Wiederverwendung | **Komposition** (Struct enthält Struct) + **Default-Methoden** im Trait |
| gemeinsames Interface | **Trait** (mehrere implementierbar, kein Diamond-Problem) |
| „ist-ein"-Polymorphie | **Trait-Objekt** `dyn Trait` |

Default-Methoden wirken wie Mixins:
```
trait Begruessung {
    fn name(&self) -> String;
    fn hallo(&self) -> String {           // Default — „geerbtes" Verhalten
        format!("Hallo, {}", self.name())
    }
}
```

## Didaktischer Kernpunkt

Rust **erzwingt**, was modernes OOP ohnehin empfiehlt: *„composition over inheritance"* (Joshua Bloch, Effective Java). Weil Vererbung gar nicht existiert, kann man die typischen Fallen nicht bauen: fragile Basisklassen, tiefe Hierarchien, Diamond-Problem. Die Empfehlung wird zur Sprachgarantie.

Gleichzeitig ist Rust stark **funktional**: Immutability als Default, Pattern Matching, Summentypen (`enum`), Closures, Iterator-Ketten.

## Java vs. Rust — derselbe Gedanke, andere Mittel

| | Java | Rust |
|---|---|---|
| Klasse | `class` | `struct` + `impl` |
| Interface | `interface` | `trait` |
| Wiederverwendung | Vererbung | Komposition + Default-Methoden |
| Polymorphie | virtuelle Methoden | `dyn Trait` **oder** Generics |
| Null | `null` | kein Null → `Option<T>` |

## Schüler-Übung

1. Eine kleine Klassenhierarchie aus Java (z. B. `Tier → Hund/Katze`) **ohne Vererbung** in Rust modellieren — mit Trait + Structs.
2. Begründen: Welche Stellen wurden dadurch *besser*, welche umständlicher?
3. Eine Methode als **Default-Methode** im Trait umsetzen und zeigen, dass sie „geerbt" wirkt.
4. Diskussion: Warum ist „kein Null" (Rust `Option`) eine OOP-/Design-Verbesserung gegenüber Java?

**Bewertungsfokus:** Verständnis, dass OOP-*Ziele* (Kapselung, Polymorphie) von OOP-*Mechanismen* (Vererbung) trennbar sind.

---

Verwandt: [[OOP-Konzept-Zerlegung-Diagramm]] · [[StreamPrometh]] · [[Sprachwahl-begründen-StreamPrometh]] · [[REST-API-Architektur]]
