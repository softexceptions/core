---
name: rust-solid
description: |
  SOLID principles for Rust projects. Applies Clean Architecture with three distinct layers:
  Domain (structs, enums, trait interfaces, use cases — no external crates), Infrastructure
  (trait implementations, HTTP clients, DTOs), and Application (main, CLI, TUI, orchestration).
  Uses traits as interfaces (Rust has no interface keyword), constructor injection via generics
  or Box<dyn Trait>, and thiserror for domain errors. Structs with impl blocks replace classes.
  All business logic lives in domain — infrastructure only handles I/O and data access.

  Use /rust-solid when:
  - Creating new Rust structs, traits, services, or repositories
  - Reviewing Rust code for SOLID violations
  - Planning module structure for a new Rust project
  - Adding a new data source, protocol, or external dependency
---

# Rust SOLID Architecture

Du bist jetzt ein Senior Rust-Entwickler, der produktionsreifen Code nach SOLID-Prinzipien und Clean Architecture schreibt.

## Core Principle

**Clean Architecture mit SOLID für Rust — drei distinkte Schichten.**

1. **Domain Layer** — Kernlogik (keine externen Crates außer `std`, `thiserror`, `uuid`)
2. **Infrastructure Layer** — Trait-Implementierungen, HTTP, WebSocket, Datei-I/O, DTOs
3. **Application Layer** — `main.rs`, CLI-Parsing, Orchestrierung, DI-Setup

**Rust hat kein `interface`-Keyword. Traits übernehmen diese Rolle.**

---

## Architecture Rules

### Layer 1: Domain (Kernlogik)

**Location:** `src/domain/`

**Komponenten:**
- **Models:** Immutable Domain-Entitäten (`struct` + `impl`, kein `pub` auf Felder ohne Grund)
- **Trait-Interfaces:** `trait IRepository` — definiert Vertrag, keine Implementierung
- **Use Cases:** Eine Struct pro Operation (`FindNearestStation`, `SendDestination`)
- **Domain Errors:** `thiserror`-Enums — klare Fehlertypen ohne Boilerplate

**Regeln:**
- NIEMALS externe HTTP-Crates (`reqwest`, `tokio-tungstenite`) importieren
- NIEMALS Infrastruktur-Structs direkt verwenden — nur eigene Traits
- Alle Business-Regeln leben hier
- Traits definieren den Vertrag (DIP)

**Beispiel:**

```rust
// src/domain/models/board.rs
#[derive(Debug, Clone, PartialEq)]
pub struct Board {
    pub id: String,
    pub name: String,
    pub local_ip: String,
}

impl Board {
    pub fn new(id: impl Into<String>, name: impl Into<String>, local_ip: impl Into<String>) -> Self {
        Self {
            id: id.into(),
            name: name.into(),
            local_ip: local_ip.into(),
        }
    }
}

// src/domain/errors.rs
use thiserror::Error;

#[derive(Debug, Error)]
pub enum DomainError {
    #[error("Board mit PIN '{0}' nicht gefunden")]
    BoardNotFound(String),
    #[error("Verbindung zum Board fehlgeschlagen: {0}")]
    ConnectionFailed(String),
    #[error("Stream-Fehler: {0}")]
    StreamError(String),
}

// src/domain/repositories/i_board_repository.rs
use async_trait::async_trait;
use crate::domain::{models::Board, errors::DomainError};

#[async_trait]
pub trait IBoardRepository: Send + Sync {
    async fn find_by_pin(&self, pin: &str) -> Result<Board, DomainError>;
}

// src/domain/repositories/i_stream_service.rs
#[async_trait]
pub trait IStreamService: Send + Sync {
    async fn start_stream(&self, board: &Board) -> Result<(), DomainError>;
    async fn stop_stream(&self) -> Result<(), DomainError>;
}

// src/domain/use_cases/connect_to_board.rs
use crate::domain::{
    errors::DomainError,
    models::Board,
    repositories::{IBoardRepository, IStreamService},
};

pub struct ConnectToBoard<R: IBoardRepository, S: IStreamService> {
    board_repo: R,
    stream_service: S,
}

impl<R: IBoardRepository, S: IStreamService> ConnectToBoard<R, S> {
    pub fn new(board_repo: R, stream_service: S) -> Self {
        Self { board_repo, stream_service }
    }

    pub async fn execute(&self, pin: &str) -> Result<Board, DomainError> {
        let board = self.board_repo.find_by_pin(pin).await?;
        self.stream_service.start_stream(&board).await?;
        Ok(board)
    }
}
```

---

### Layer 2: Infrastructure (Implementierungen & I/O)

**Location:** `src/infrastructure/`

**Komponenten:**
- **Repository-Implementierungen:** Implementieren Domain-Traits
- **HTTP-Clients:** `reqwest` für externe APIs
- **DTOs:** JSON-Parsing — niemals in die Domain-Schicht leaken
- **External Services:** WebSocket, Videostream, Promethean-API

**Regeln:**
- Implementiert Domain-Traits (`impl IBoardRepository for PrometheanClient`)
- Konvertiert DTOs → Domain-Modelle (niemals rohe DTOs zurückgeben)
- Keine Business-Logik — nur I/O und Datenzugriff
- Fehler werden in `DomainError` übersetzt

**Beispiel:**

```rust
// src/infrastructure/dtos/board_dto.rs
use serde::Deserialize;
use crate::domain::models::Board;

#[derive(Deserialize)]
pub struct BoardDto {
    pub panel_id: String,
    pub name: String,
    pub local_ip: String,
}

impl BoardDto {
    // DTO → Domain-Modell: nur in Infrastructure erlaubt
    pub fn into_domain(self) -> Board {
        Board::new(self.panel_id, self.name, self.local_ip)
    }
}

// src/infrastructure/repositories/promethean_client.rs
use async_trait::async_trait;
use reqwest::Client;
use crate::domain::{errors::DomainError, models::Board, repositories::IBoardRepository};
use crate::infrastructure::dtos::BoardDto;

pub struct PrometheanClient {
    http: Client,
    base_url: String,
}

impl PrometheanClient {
    pub fn new(http: Client, base_url: impl Into<String>) -> Self {
        Self { http, base_url: base_url.into() }
    }
}

#[async_trait]
impl IBoardRepository for PrometheanClient {
    async fn find_by_pin(&self, pin: &str) -> Result<Board, DomainError> {
        let url = format!("{}/checkPanel?panelId={}", self.base_url, pin);

        let response = self.http.get(&url).send().await
            .map_err(|e| DomainError::ConnectionFailed(e.to_string()))?;

        if response.status() == 404 {
            return Err(DomainError::BoardNotFound(pin.to_string()));
        }

        let dto: BoardDto = response.json().await
            .map_err(|e| DomainError::ConnectionFailed(e.to_string()))?;

        Ok(dto.into_domain())
    }
}
```

---

### Layer 3: Application (main + Orchestrierung)

**Location:** `src/main.rs`, `src/app.rs`

**Regeln:**
- Baut den Dependency-Graph auf (DI-Setup)
- Parst CLI-Argumente (`clap`)
- Verbindet Schichten — keine eigene Logik
- Thin: delegiert sofort an Use Cases

**Beispiel:**

```rust
// src/main.rs
use clap::Parser;

#[derive(Parser)]
struct Args {
    /// PIN-Code des Boards (z.B. "512903")
    pin: String,
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let args = Args::parse();

    // DI-Setup: Infrastruktur-Implementierungen injizieren
    let http_client = reqwest::Client::new();
    let board_repo = PrometheanClient::new(http_client, "https://servicediscovery.mypromethean.com");
    let stream_service = WebRtcStreamService::new(/* config */);

    // Use Case mit injizierten Abhängigkeiten
    let use_case = ConnectToBoard::new(board_repo, stream_service);

    match use_case.execute(&args.pin).await {
        Ok(board) => println!("✓ Verbunden mit: {}", board.name),
        Err(e) => eprintln!("✗ Fehler: {}", e),
    }

    Ok(())
}
```

---

## SOLID in Rust

### Single Responsibility (SRP)

```rust
// ✅ GUT — jede Struct hat eine Aufgabe
struct ConnectToBoard { ... }    // nur Verbindungsaufbau
struct EncodeFrame { ... }       // nur H.264-Encoding
struct PrometheanClient { ... }  // nur HTTP zu Promethean

// ❌ SCHLECHT — God-Struct
struct StreamManager {
    fn connect_and_encode_and_send_and_log() { ... }
}
```

### Open/Closed (OCP)

```rust
// ✅ GUT — Erweiterung via neue Implementierung
trait IBoardRepository { ... }
struct PrometheanClient;  // implements IBoardRepository
struct SmartBoardClient;  // implements IBoardRepository — kein bestehender Code ändert sich

// ❌ SCHLECHT — Code-Änderung für jede neue Quelle
fn find_board(brand: &str) {
    if brand == "promethean" { ... }
    else if brand == "smart" { ... }  // Änderung nötig
}
```

### Liskov Substitution (LSP)

```rust
// ✅ GUT — alle Implementierungen erfüllen den Vertrag
trait IStreamService {
    async fn start_stream(&self, board: &Board) -> Result<(), DomainError>;
}

struct WebRtcStreamService;    // echte Implementierung
struct MockStreamService;      // für Tests — gleiches Interface

// Use Case funktioniert mit BEIDEN
let use_case = ConnectToBoard::new(board_repo, MockStreamService);
```

### Interface Segregation (ISP)

```rust
// ✅ GUT — kleine, fokussierte Traits
trait IBoardFinder {
    async fn find_by_pin(&self, pin: &str) -> Result<Board, DomainError>;
}

trait IChannelProvider {
    async fn get_channel(&self, pin: &str, client_id: &str) -> Result<Channel, DomainError>;
}

// Use Cases hängen nur von dem ab, was sie brauchen
struct ConnectToBoard<R: IBoardFinder> { repo: R }         // braucht nur find
struct StartStream<C: IChannelProvider> { provider: C }    // braucht nur channel

// ❌ SCHLECHT — Allzweck-Trait
trait IPrometheanApi {
    async fn find_by_pin(&self, ...) -> ...;
    async fn get_channel(&self, ...) -> ...;
    async fn check_health(&self) -> ...;   // ConnectToBoard braucht das nicht
}
```

### Dependency Inversion (DIP)

```rust
// ✅ GUT — Abhängigkeit auf Trait-Ebene (statisch via Generics)
struct ConnectToBoard<R: IBoardRepository, S: IStreamService> {
    board_repo: R,
    stream_service: S,
}

// ✅ GUT — dynamischer Dispatch (für heterogene Collections oder trait objects)
struct App {
    board_repo: Box<dyn IBoardRepository>,
    stream_service: Arc<dyn IStreamService + Send + Sync>,
}

// ❌ SCHLECHT — direkter Typ, keine Abstraktion
struct ConnectToBoard {
    client: PrometheanClient,  // konkrete Klasse — nicht mockbar
}
```

---

## Dependency Injection Patterns

### Pattern 1: Generics (bevorzugt — Zero Cost)

```rust
// Kompilier-Zeit-DI — keine Runtime-Kosten
struct ConnectToBoard<R, S>
where
    R: IBoardRepository,
    S: IStreamService,
{
    board_repo: R,
    stream_service: S,
}

impl<R: IBoardRepository, S: IStreamService> ConnectToBoard<R, S> {
    pub fn new(board_repo: R, stream_service: S) -> Self {
        Self { board_repo, stream_service }
    }
}
```

### Pattern 2: Box<dyn Trait> (flexibel — für komplexe Graphen)

```rust
// Laufzeit-DI — einfacher für tiefe Dependency-Graphen
pub struct App {
    connect: Box<dyn IConnectUseCase>,
}

impl App {
    pub fn build() -> Self {
        let http = reqwest::Client::new();
        let board_repo: Box<dyn IBoardRepository> = Box::new(PrometheanClient::new(http, BASE_URL));
        let stream_service: Box<dyn IStreamService> = Box::new(WebRtcStreamService::new());

        Self {
            connect: Box::new(ConnectToBoard::new(board_repo, stream_service)),
        }
    }
}
```

### Pattern 3: Arc<dyn Trait + Send + Sync> (für Async + Multi-Thread)

```rust
// Für geteilte Ownership in async Kontexten
pub struct App {
    board_repo: Arc<dyn IBoardRepository + Send + Sync>,
}
```

---

## Error Handling

```rust
// src/domain/errors.rs — Domain-Fehler mit thiserror
use thiserror::Error;

#[derive(Debug, Error)]
pub enum DomainError {
    #[error("Board '{0}' nicht gefunden")]
    BoardNotFound(String),
    #[error("Verbindung fehlgeschlagen: {0}")]
    ConnectionFailed(String),
    #[error("Stream-Fehler: {0}")]
    StreamError(String),
}

// Infrastructure übersetzt externe Fehler in DomainError
impl From<reqwest::Error> for DomainError {
    fn from(e: reqwest::Error) -> Self {
        DomainError::ConnectionFailed(e.to_string())
    }
}

// Application-Layer: anyhow für unkritische Fehlerweiterleitung
// main.rs → anyhow::Result<()> ist OK
// Use Cases → Result<T, DomainError> — explizit
```

---

## File Structure

```
src/
├── main.rs                          # Entry Point + DI-Setup
├── domain/
│   ├── mod.rs
│   ├── errors.rs                    # DomainError (thiserror)
│   ├── models/
│   │   ├── mod.rs
│   │   └── board.rs                 # Domain-Entität
│   ├── repositories/
│   │   ├── mod.rs
│   │   ├── i_board_repository.rs    # Trait (Interface)
│   │   └── i_stream_service.rs      # Trait (Interface)
│   └── use_cases/
│       ├── mod.rs
│       └── connect_to_board.rs      # Eine Struct pro Operation
├── infrastructure/
│   ├── mod.rs
│   ├── dtos/
│   │   ├── mod.rs
│   │   └── board_dto.rs             # JSON-Parsing + into_domain()
│   ├── repositories/
│   │   ├── mod.rs
│   │   └── promethean_client.rs     # impl IBoardRepository
│   └── services/
│       ├── mod.rs
│       └── webrtc_stream_service.rs # impl IStreamService
└── app.rs                           # Optional: App-Struct für DI-Graph
```

---

## Best Practices

### ✅ DO:
1. `trait IRepository` für jede externe Abhängigkeit
2. Constructor Injection via `fn new(dep: impl ITrait)` oder Generics
3. Use Cases: eine Struct, eine `async fn execute()`
4. DTOs nur in Infrastructure — `into_domain()` vor dem Rückgeben
5. `thiserror` für Domain-Fehler, `anyhow` für Application-Layer
6. `#[cfg(test)]` Module in jeder Datei für Unit-Tests
7. Alle async Traits mit `#[async_trait]` oder RPITIT (Rust 1.75+)

### ❌ DON'T:
1. Kein `reqwest` / `tokio-tungstenite` im Domain-Layer
2. Kein `unwrap()` im Produktionscode — immer `?` oder explizite Fehlerbehandlung
3. Keine Business-Logik in `main.rs` — nur DI-Setup und Delegation
4. Keine DTOs in Domain-Funktionssignaturen
5. Kein `static` für Dependencies — immer injizieren
6. Kein `todo!()` oder `unimplemented!()` in Produktion
