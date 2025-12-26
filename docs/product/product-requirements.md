# Product Requirement Document (PRD): keva-mock

## 1. Executive Summary

**keva-mock** is a robust, highly configurable **Dynamic Mock Server**. Designed as a generic tool to mimic *any* RESTful API behavior, it empowers developers and QA engineers to simulate complex backend systems, proxy requests to real services, and enforce strict protocol compliance through a unified, configuration-driven interface.

## 2. Goals & Objectives

- **Decouple Development**: Allow frontend and downstream service development to proceed independently of actual upstream API availability.
- **Enable Edge-Case Testing**: Provide mechanisms to simulate hard-to-reproduce scenarios like network timeouts, database errors, and specific error codes.
- **Ensure Protocol Compliance**: Enforce strict request validation against defined schemas to ensure integration correctness.
- **Developer Frictionless**: Offer a "plug-and-play" experience with hot-reloading configurations, auto-documentation, and clear logging.

## 3. Target Audience

- **Backend Developers**: Integrating external third-party services who need a stable endpoint during dev.
- **Frontend Developers**: Can mock any API to test their UIs with realistic data.
- **QA Engineers**: Automating integration tests and verifying system resilience (timeouts, rate limits).

## 4. Functional Requirements

### 4.1 Configuration & Routing

- **Multi-Workspace Architecture (Multi-Tenant)**:
  - **Server Config**: `server.toml`. Can optionally implicitly scan dirs OR explicitly map roots via `[workspaces]` table.
  - **Collision Policy**: If multiple workspaces claim the same `root`, the server MUST fail to start (Fatal Error). No ambiguity allowed.
  - **Context Path**: Each workspace defines a `root` (e.g., `/stripe` or `/transferwise`). All routes in that workspace are mounted under this prefix (e.g., `/stripe/v1/charges`).
  - **Isolation**: Workspaces have isolated Configuration, Database connections, and plugins.
- **Dynamic Routing**: Routes are matched based on Workspace Root + Endpoint Path.
- **Hot Reload**: Users MUST be able to reload the configuration at runtime via a `POST /api/refresh` endpoint without restarting the process.
- **Routing**: Routes are defined dynamically in the config.
- **Versioning Support**:
  - **URL Strategy**: Routes can be prefixed with versions (e.g., `/v1/api/...`).
  - **Header Strategy**: Routes can be versioned via headers (e.g., `X-API-Version: v1`).
  - Strategy MUST be configurable globally and per-endpoint.

### 4.2 Request Handling

- **HTTP Methods**: Support *ALL* standard HTTP methods (GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS).
- **Authentication**:
  - Support `Bearer` token validation.
  - Support `API Key` validation (custom header).
  - Return `401 Unauthorized` on failure.
- **Schema Validation**:
  - **Location-Aware**: Validate parameters in `body`, `query`, `path params`, or `headers`.
  - Validate Types, Length, and Formats.
  - Return `400 Bad Request` with detailed errors.
- **Request Tracing**:
  - Generate a unique Trace ID for every incoming request.
  - Include Trace ID in response headers (`X-Trace-Id`).
  - Make Trace ID available for logging and side-effects.
- **Rate Limiting**:
  - Limit requests per endpoint within a specified time window.
- Return `429 Too Many Requests` when exceeded.

### 4.3 Response Logic

- **Data Sources**:
  - **File**: Serve static JSON or HTML files.
  - **Database**: Execute SQL queries against a PostgreSQL database to generate responses.
  - **Proxy**: Forward requests to a real upstream server (Pass-through mode).
- **Templating**: Support dynamic value injection in responses (e.g., `{{request.body.transactionId}}`).
- **Advanced Mocking**:
  - **Latency Simulation**: Configurable artificial delays (`delay_ms`) to simulate network lag.
  - **Failure Scenarios**: API clients can trigger specific failure modes (e.g., 500 Error, Timeout) via a special header `X-Mock-Scenario`.
  - **Stateful Scenarios**: Configure scenarios to trigger based on state (e.g., "fail after 5 successful requests").
- **Side Effects**:
  - Execute "On-Request" SQL queries (e.g., logging request payloads to a DB table) before sending a response.

### 4.4 Simulation & Testing

- **Request History**: Store the last N requests in memory and expose them via an API (`GET /api/history`) to allow test scripts to verify sent payloads.
- **Custom Headers**: Allow configuring arbitrary response headers (static or dynamic).
- **Format Support**: Explicit support for `application/json` and `text/html` content types.

### 4.5 Documentation

- **Auto-Swagger**: Automatically generate an OpenAPI 3.0 specification from the current configuration and schemas.
- **UI**: Serve a Swagger UI at `/docs` for interactive exploration.

### 4.6 Extensibility (Plugin Architecture)

- **Pluggable System**: Support loading external Rhai scripts as plugins via `config.toml`.
- **Hooks**:
  - `onRequest`: Intercept request before processing (e.g., custom decryption).
  - `onAuth`: Custom authentication logic beyond Bearer/API Key.
  - `onResponse`: Transform response data before sending (e.g., custom encryption, signing).
- **Context**: Plugins access the full Request/Response objects and Server Config.

### 4.7 Scheduled Jobs (Cron)

- **Job Scheduler**: Run tasks at defined intervals (Cron syntax) or specific times.
- **Actions**: Send HTTP requests to external or internal URLs.
- **Payload Sources**:
  - **File**: Send a static JSON payload from a file.
  - **Database**: Execute a query; for *each row* returned, send a request.
- **Validation**:
  - **Request**: Validate outgoing payload against a schema.
  - **Response**: Validate incoming response from external API against a schema.
- **Logging**: Trace execution results.

## 5. Assumptions & Constraints

- While designed for production, users must ensure upstream proxy targets are secure.
- Database queries defined in config are executed as-is; strict code review of `config.toml` is required in production environments.
- **Workspace Conflicts**: Duplicate `root` paths are a critical configuration error. The server must validate unicast roots at startup and exit with a non-zero status code if duplicates are found.

## 6. Success Metrics

- Server successfully validates 100% of defined request schemas.
- Configuration changes are reflected < 500ms after calling `/refresh`.
- Supports > 1000 concurrent requests (with clustering enabled).

## 7. Production Readiness (Non-Functional Requirements)

- **Scalability**: Utilize all available CPU cores for concurrent processing.
- **State Persistence**: Use Redis (optional) for Rate Limiting counters and Stateful Scenario tracking across multiple instances.
- **Observability**: Expose Prometheus-compatible metrics at `/metrics`. MUST include `workspace` label for all request metrics.
- **Health Checks**:
  - `/health/live`: Returns 200 OK if process is running.
  - `/health/ready`: Returns 200 OK only if Database, Redis, and Critical Files are accessible.
- **Security**: Implement robust header security, proper CORS configuration, and ensure no sensitive data leaks in logs.
