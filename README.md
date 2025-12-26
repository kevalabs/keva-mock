# keva-mock

> **A high-performance, configuration-driven Mock Server built in Rust.**

`keva-mock` is designed to simulate complex backend systems, third-party services (e.g., Payment Gateways, ERPs), and internal microservices. It focuses on **Developer Experience**, **Performance**, and **Correctness**.

## 🚀 Key Features

- **Multi-Tenant Workspaces**: Run multiple isolated mock environments (e.g., `/stripe`, `/transferwise`) on a single server instance.
- **Dynamic Routing**: zero-copy path matching with support for wildcard and parameterized routes.
- **Strict Schema Validation**: Validate incoming requests (Body, Query, Headers) and outgoing responses against defined schemas (JSON Schema / TOML).
- **Scripting Hooks (Rhai)**: Attach dynamic logic (`onRequest`, `onResponse`) using [Rhai](https://rhai.rs) - a safe, embedded scripting language for Rust.
- **Scheduled Jobs**: Native Cron support to trigger webhooks or internal actions.
- **Chaos Engineering**: Built-in support for simulating Latency, Timeouts, and Errors via headers.
- **Vertical Slice Architecture**: Codebase organized by Functional Capability for maintainability.

## 📦 Installation

```bash
# Clone the repository
git clone https://github.com/kevalabs/keva-mock.git

# Build via Cargo
cargo build --release

# Run
./target/release/keva-mock --server-config server.toml
```

## 🛠 Configuration

`keva-mock` uses a hierarchical configuration system.

### 1. Global Server Config (`server.toml`)

Defines the listener and workspace mappings.

```toml
[server]
port = 3000
host = "0.0.0.0"

[workspaces]
# Mount workspaces to context roots
"/stripe" = "./mocks/stripe"
"/transferwise" = "./mocks/transferwise"
```

### 2. Workspace Config (`mocks/stripe/config.toml`)

Defines the behavior for a specific domain.

```toml
root = "/stripe"

[database]
url = "postgres://user:pass@localhost:5432/stripe_mock"

[[endpoints]]
path = "/v1/payment/initiate"
method = "POST"
    [endpoints.request]
    schema_file = "schemas/initiate_req.toml"

    [endpoints.response]
    source = "file"
    file = "data/success_response.toml"
    # Or use a script for dynamic logic
    # source = "script"
    # script_file = "scripts/initiate_payment.rhai"
```

## 🧠 Architecture

`keva-mock` follows a **Vertical Slice Architecture** to keep related logic (Routing, Validation, Mocking) co-located.

- **Core**: Async Request Pipeline built on `Axum` and `Tokio`.
- **Engine**: `Rhai` scripting engine for dynamic behavior without recompilation.
- **Storage**: `SQLx` for DB interaction and `Redis` for distributed state (e.g., Rate Limiting).

## 🤝 Contributing

1. Fork the repo.
2. Create a feature branch (`git checkout -b feature/amazing-feature`).
3. Commit your changes.
4. Open a Pull Request.

## 📄 License

MIT
