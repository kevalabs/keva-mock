# Technical Architecture: keva-mock

| Document | Technical Design (TDD) |
| :--- | :--- |
| **Version** | 1.0|
| **Architecture** | Vertical Slice (Modular Monolith) |
| **Stack** | Rust (Workspace) |

---

## 1. Architectural Pattern

The system follows a **Vertical Slice Architecture**. Code is organized by **Functional Capability** (Traffic) rather than technical layers. Each slice is a **Rust Module** with strict visibility rules (using `pub(crate)` vs `pub`).

**Dependency Flow:**
`main.rs (Wiring) → modules/ (Feature) → shared/ (Kernel)`

```mermaid
graph TD
    classDef service fill:#e3f2fd,stroke:#1565c0,stroke-width:2px;
    classDef module fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px;
    classDef platform fill:#fff3e0,stroke:#ef6c00,stroke-width:2px;

    subgraph Binary ["keva-mock"]
        Main[main.rs]:::service
        
        subgraph Slices ["Modules"]
            Traffic[mod traffic]:::module
        end

        subgraph Kernel ["Shared"]
            Shared[mod shared]:::platform
        end
    end

    Main --> Traffic
    Traffic --> Shared
```

## 2. Directory Structure (Single Crate)

A single binary crate with well-defined module boundaries.

```text
keva_mock/
├── Cargo.toml                 # Single Dependency Manifest
├── src/
│   ├── main.rs                # App Wiring
│   ├── shared.rs              # Shared Kernel Entry
│   ├── shared/
│   │   └── error.rs           # AppError
│   ├── modules.rs             # Modules Entry
│   └── modules/
│       ├── traffic.rs         # Traffic Slice Entry
│       ├── traffic/
│       │   ├── domain/        # Pure Logic
│       │   └── infra/         # Axum Handlers
└── infra/                     
    └── docker-compose.yml     # Local Dev Environment
```

### 2.1 Module Boundaries

- **Visibility**: We use `pub(super)` or `pub(crate)` to hide internal slice details. Only the "Public API" of the slice (e.g., `pub fn init_routes()`) is exposed to `main.rs`.

## 3. Data Flow (Request Processing)

Explicit mapping prevents internal domain details from leaking to the API contract.

```mermaid
sequenceDiagram
    participant Client
    participant API as keva-server
    participant Traffic as keva_traffic

    Client->>API: POST /stripe/charges
    API->>API: Match Workspace (/stripe)
    
    API->>Traffic: Handle Request
    Traffic->>Traffic: Pipeline (Auth -> Limit -> Script)
    
    alt is Mock
        Traffic->>Traffic: Match Config (In-Memory)
        Traffic->>Client: Return Mock Response
    else is Proxy
        Traffic->>Upstream: Forward Request
        Upstream-->>Traffic: Response
        Traffic->>Client: Return Proxy Response
    end
```

## 4. Concurrency & Persistence

- **Configuration**: File-based (TOML). Loaded into memory at startup. Read-heavy.

## 5. Technology Stack

| Component | Technology | Description |
| :--- | :--- | :--- |
| **Language** | Rust 1.75+ | core logic. |
| **Web Framework** | Axum 0.7+ | HTTP layer. |
| **Runtime** | Tokio | Async I/O. |
| **Stack** | Rust (Workspace) |
