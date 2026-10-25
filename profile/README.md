# Sentin8 - The Enterprise Platform

**Secure Post-Quantum-Safe Infrastructure Platform That Heals and Scales Itself.**

Sentin8 is the shared infrastructure platform beneath all verticals — identity (Auth/Proxy), event transport (Bus), data storage, scheduling, workflow orchestration, admin UI rendering, and telemetry. Every vertical (bank8, bill8, hotel8, exchange8, trade8, email8) is composed entirely on these platform services; no vertical implements its own identity, events, or infrastructure.

## Platform Services

Sentin8 ships as 17 independent Rust services organized in a boot-tier DAG:

### Tier 1 — Certificate Authority

| Service | Domain | What it does |
|---|---|---|
| **Sentin8:Acme** | PKI / Certificate Authority | Root CA, workload certificate issuance, TLS cert lifecycle for all platform and vertical services |

### Tier 2 — Identity & Ingress

| Service | Domain | What it does |
|---|---|---|
| **Sentin8:Auth** | Identity Provider | JWT/M2M/ID/refresh/OIDC token issuance, JWKS authority, certificate-bound M2M, OAuth scopes, RBAC authorization (Shield engine), OIDC FAPI/SAAI suites |
| **Sentin8:Proxy** | TLS Ingress Router | L4 SNI routing → L7 TLS termination, hostname→instance resolution, per-request context injection (`X-Sentin8-*` headers) |

### Tier 3 — Event Transport

| Service | Domain | What it does |
|---|---|---|
| **Sentin8:Bus** | Event Transport | Durable Valkey Streams transport, local outbox relay, producer path (outbox → `POST /v1/produce`), consumer groups, SSE push, DLQ management |

### Tier 4 — Domain Services

| Service | Domain | What it does |
|---|---|---|
| **Sentin8:DB** | Database Platform | PostgreSQL connection pool capping, OAuth-issued workload certificates per service, schema-per-service tenant isolation (RLS) |
| **Sentin8:Storage** | Object Storage | File/object storage for documents, attachments, statements — used by all services for non-relational data |
| **Sentin8:Post** | Email Platform | SMTP/IMAP/POP3/SIEVE email infrastructure, message routing and delivery, mail accounts provisioned from Auth events |
| **Sentin8:Cron** | Scheduler | Recurring job scheduling — invoice runs, statement generation, session timing, EOD processing |
| **Sentin8:Stat** | Telemetry | Business metrics aggregation, event ingestion, observability data for dashboards and KPIs |
| **Sentin8:Secrets** | Secret Management | Secure credential storage, secret references — never plaintext in application tables |
| **Sentin8:Func** | WASM Functions | Tenant-authored executable hooks via WASM module execution — custom logic triggered by events or schedules |
| **Sentin8:Flow** | Workflow Engine | Multi-step workflow orchestration with retries, wait states, idempotency keys — payment flows, provisioning, subscription lifecycle |

### Tier 5 — Organization & Lifecycle

| Service | Domain | What it does |
|---|---|---|
| **Sentin8:Sys** | Org/Env Lifecycle | Organization and environment creation/deletion, platform policy templates, service enablement events (`sentin8:sys:service:updated`) |
| **Sentin8:App** | App Store | Application registry, listing management, vertical app store administration |

### Tier 6 — Client Infrastructure

| Service | Domain | What it does |
|---|---|---|
| **Sentin8:BFF** | Proxy & Delegation Gateway | OAuth token custody server-side, proxy to downstream services (`/proxy/*`), delegation gateway for CIBA/m2m, tenant resolution, scope downscoping |
| **Sentin8:Web** | Web Services | Bidirectional WebSocket tier — chat and real-time communication endpoints |

## Flutter Apps & SDKs

| Repository | Description |
|---|---|
| `sentin8-flutter` | Shared Flutter package (native + web infrastructure for all apps) |
| `sentin8-auth-ui` | Auth platform Flutter app (sign-in/consent UI) |
| `sentin8-dash` | Admin UI rendering engine — renders vertical RFW admin pages at runtime without compile-time knowledge |
| `sentin8-site-engine` | `@sentin8/site-engine` — shared Docusaurus engine (theme, components, harvester) behind sentin8.dev + every `{vertical}.dev` |
| `sentin8-site` | Public platform website at sentin8.dev (harvests docs from all unit repos) |
| `sentin8-sdk` | Shared workspace: core crates + identity + TLS + Bus + Shield policy engine + shell utilities + facade crate (`sentin8`) — all verticals consume exclusively through this git pin |
| `sentin8-cli` | Bootstrap CLI — database creation, vertical init, service enrollment, captain control |
| `sentin8-captain` | Process supervisor — health-gated tier spawning, rolling upgrades, fleet management |
| `sentin8-packaging` | Debian package split + vendored registry distribution |
| `sentin8-e2e` | Playwright cross-service end-to-end test suites |

## Vertical Dependencies

All verticals depend on these platform services:

| Platform Service | What the vertical gets | Direction |
|---|---|---|
| **Sentin8:Acme** | Root CA + workload certificates for every service identity | vertical → acme (cert issuance) |
| **Sentin8:Auth** | Identity, sign-in, JWT/M2M/refresh/OIDC tokens, JWKS, RBAC, Shield authorization | vertical → Auth (verification + M2M identity) |
| **Sentin8:Proxy** | TLS termination, hostname→instance resolution, request context injection | vertical ← Proxy (single ingress) |
| **Sentin8:BFF** | Client token custody, proxy, delegation gateway, tenant resolution | vertical ← BFF (user tokens proxied to services) |
| **Sentin8:Bus** | Durable event transport, outbox relay, consumer groups, SSE push, DLQ | vertical → Bus (produce + consume events) |
| **Sentin8:DB** | PostgreSQL connection pooling with OAuth workload certificates, RLS enforcement | vertical → DB (service-specific schemas) |
| **Sentin8:Flow** | Multi-step workflow orchestration (retries, wait states, idempotency) | vertical → Flow (workflow execution) |
| **Sentin8:Cron** | Recurring job scheduling (EOD, statements, invoice runs) | vertical → Cron (schedule registration) |
| **Sentin8:Func** | Tenant-authored WASM hook execution | vertical ← Func (proxy-back handlers) |
| **Sentin8:Storage** | Object/file storage for documents, attachments, statements | vertical → Storage (document PUT/GET) |
| **Sentin8:Stat** | Business metrics aggregation and telemetry | vertical → Stat (event ingestion) |
| **Sentin8:Sys** | Organization/environment lifecycle, service enablement events | vertical ← Sys (env-init triggers) |
| **Sentin8:App** | App store management and listing administration | vertical → App (store listings) |
| **Sentin8:Secrets** | Secure credential/secret storage (reference-only) | vertical → Secrets (credential refs) |
| **Sentin8:Sign** | Digital signature infrastructure | vertical → Sign (document/signing) |
| **Sentin8:Web** | WebSocket tier for bidirectional communication | vertical ← Web (real-time endpoints) |

## Platform Invariants

- **Single identity plane** — Sentin8:Auth is the *only* issuer, JWKS authority, and authentication authority. No service mints its own sessions, API keys, or credentials
- **Zero compile-time vertical knowledge** — sentin8 never depends on a vertical; verticals depend on sentin8 exclusively through the SDK git pin
- **Certificate identity everywhere** — PostgreSQL 18 + Valkey both run on TLS-only certificate identity; no data-plane passwords anywhere in the stack
- **Event-driven env-init** — platform emits `sentin8:sys:service:updated` events; verticals self-initialize their tenants idempotently
- **Shield is the only name** — the declarative-policy engine (`sentin8-policy`) is branded Shield; never referenced by its vendor name

---

**Architecture & Design:** sentin8 and its underlying concepts were designed by [Hamid Jahanzad](https://www.linkedin.com/in/hamidjahanzad/).
