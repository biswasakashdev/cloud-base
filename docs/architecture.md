# Cloud Base — Architecture

> Tech stack, repo layout, and "where to find what". Requirements: `requirements.md`. Rules: `rules.md`. Build order: `phases.md`.
> If code and this file disagree, **update this file in the same change**.

## 1. Tech stack

| Area | Choice |
|---|---|
| Server language / framework | Java 21, Spring Boot 3.x, Maven (multi-module) |
| Module boundaries | Spring Modulith (modular monolith) |
| Web / API | Spring Web (REST), springdoc-openapi, RFC 7807 `ProblemDetail` errors |
| Auth | Spring Security, `oauth2-resource-server` (JWT, RS256), BCrypt/Argon2, Google ID token verification |
| Persistence | PostgreSQL 16, Spring Data JPA (jOOQ allowed for complex reads), Flyway |
| Cache / rate limit | Redis + Bucket4j |
| Concurrency | Java 21 virtual threads (`Executors.newVirtualThreadPerTaskExecutor()`) |
| Container control | `docker-java` (build, run, logs, stats) |
| Node runtime | Python 3.12 base image + platform **shim** (FastAPI/uvicorn or stdlib HTTP) |
| Messaging (internal) | Spring application events first; NATS JetStream later for worker commands/logs |
| Logs | Structured JSON → collector → Postgres table (Loki optional later) |
| Metrics | Micrometer → Prometheus → Grafana |
| Live streams | SSE (`SseEmitter`) |
| Frontend | React + TypeScript + Vite, React Flow (`@xyflow/react`), Tailwind CSS, shadcn/ui (Radix UI primitives) for all UI components, TanStack Query |
| Frontend package manager | **pnpm** only (commit `pnpm-lock.yaml`; never use npm or yarn; pin the version via the `packageManager` field in `package.json`) |
| Tests | JUnit 5, AssertJ, Mockito, Testcontainers, REST Assured; Vitest for frontend |
| Local run | Docker Compose |
| CI | GitHub Actions (build, test, lint) |

**Frontend UI approach:** build every screen from **shadcn/ui** components styled with **Tailwind CSS** utility classes. Add components with the shadcn CLI into `frontend/src/components/ui/` and compose them in feature folders; do not introduce another component library. Theme through Tailwind/CSS variables (light and dark). React Flow node renderers on the canvas are custom components that also use shadcn/ui and Tailwind for consistency.

## 2. System overview

```
                    ┌──────────────────────────────┐
                    │   Frontend (React Flow UI)   │
                    └──────────────┬───────────────┘
                                   │ REST + SSE
┌──────────────────────────────────▼───────────────────────────────────┐
│                       MASTER  (cloudbase-server)                     │
│  auth │ workspace │ flow │ nodelib │ build │ engine │ marketplace    │
│  observability │ ai │ worker-registry/scheduler │ shared             │
│                                                                      │
│               RuntimeGateway  (interface)                            │
│        ┌─────────────────┴──────────────────┐                        │
│   LocalDockerRuntime                 RemoteWorkerRuntime             │
│  (single-node mode, in-process)      (calls worker agents)           │
└──────┬───────────────────────────────────────┬───────────────────────┘
       │ docker-java                            │ HTTPS + join token
┌──────▼───────────┐                   ┌────────▼─────────────────────┐
│ Local Docker     │                   │ WORKER (cloudbase-worker)    │
│  node containers │                   │  agent → Docker on its host  │
└──────────────────┘                   └──────────────────────────────┘

Postgres (state)   Redis (rate limit)   Prometheus + Grafana
```

**Key idea:** the engine only knows `RuntimeGateway`. Whether a container runs on the master machine or a remote worker is a runtime detail. Phase 1–8 use `LocalDockerRuntime` only.

## 3. Core concepts and data model

Tables (all have `id`, `created_at`, `updated_at`; user-owned rows have `owner_id`):

| Table | Purpose |
|---|---|
| `users` | email, password_hash (nullable for Google-only), display_name, email_verified, status |
| `auth_identities` | provider (`LOCAL`/`GOOGLE`), provider_user_id, user_id |
| `refresh_tokens` | token_hash, family_id, user_id, expires_at, revoked_at, replaced_by |
| `workspaces` | owner_id, name, slug, public_id (unique, immutable, random; forms the public URL) |
| `workspace_secrets` | workspace_id, key, encrypted_value |
| `flows` | workspace_id (unique), deployed_version_id |
| `flow_versions` | flow_id, version_no, graph_json, created_by |
| `node_definitions` | owner_id, kind, name, runtime_profile, source_plugin_id (nullable) |
| `node_versions` | definition_id, version_no, code, requirements, manifest_json, image_ref, image_digest, build_status |
| `builds` | node_version_id, status, log_text, started_at, finished_at |
| `plugins` | slug, name, category, description, latest_version |
| `plugin_versions` | plugin_id, version, package_path, manifest_json |
| `executions` | flow_version_id, trigger_type, status, input, output, started_at, finished_at |
| `node_runs` | execution_id, node_id, attempt, status, input, output, error, started_at, finished_at |
| `log_lines` | execution_id, node_run_id, workspace_id, node_id, level, message, ts |
| `workers` | name, status, capacity_json, last_heartbeat_at, join_token_hash |
| `container_instances` | node_version_id or infra_node_id, worker_id, container_id, status, port |
| `audit_log` | actor_id, action, target_type, target_id, metadata |

### Node kinds

`SERVICE` (Python function), `DATABASE` (managed Postgres first), `CUSTOM_CONTAINER` (user Docker image), `TRIGGER`, `LOGIC` (If/Switch/Merge/Transform/Response). Future: `BROKER`. Kinds are an enum plus a `NodeExecutor` strategy per kind, so adding one does not touch the engine core.

### Runtime profiles

A runtime profile = base image + shim + dependency installer. `python312` is the only profile now. TS/JS/Go/Java are added as new profiles under `runtime/profiles/<name>/`; nothing outside the profile may assume Python.

## 4. Key contracts

### 4.1 Flow graph JSON

```json
{
  "schemaVersion": 1,
  "nodes": [
    { "id": "trigger1", "type": "http_trigger", "position": {"x": 0, "y": 0},
      "config": { "method": "POST", "path": "/orders" } },
    { "id": "n2", "type": "service", "nodeVersionId": "nv_123",
      "config": { "timeoutMs": 10000, "retries": 2 },
      "input": { "orderId": "{{ nodes.trigger1.output.body.id }}" } },
    { "id": "db1", "type": "database", "config": { "engine": "postgres", "version": "16" } },
    { "id": "resp", "type": "http_response", "config": { "status": 200 },
      "input": { "body": "{{ nodes.n2.output }}" } }
  ],
  "edges": [
    { "id": "e1", "source": "trigger1", "target": "n2" },
    { "id": "e2", "source": "n2", "target": "resp" },
    { "id": "e3", "source": "db1", "target": "n2", "kind": "connection", "alias": "db" }
  ]
}
```

- `kind: "flow"` (default) edges carry data and define execution order.
- `kind: "connection"` edges attach infrastructure (database/custom container) to a service node and inject `ctx.connections[alias]`. They are **not** part of the execution DAG.

### 4.2 Python node contract

```python
# main.py
def handle(input: dict, ctx) -> dict:
    ctx.log.info("processing", extra={"order": input["orderId"]})
    db = ctx.connections["db"]          # {"host":..., "port":..., "user":..., "password":..., "database":...}
    api_key = ctx.secrets.get("STRIPE_KEY")
    return {"status": "ok"}
```

`ctx` fields: `log`, `config` (dict), `secrets` (read-only), `connections` (dict), `execution_id`, `node_run_id`, `node_id`, `workspace_id`.
Return value must be JSON-serialisable and match the node's output schema. Raising an exception marks the run failed.

### 4.3 Node manifest (`node.yaml` / `plugin.yaml`)

```yaml
name: send-email
version: 1.0.0
kind: service
runtime: python312
entry: main.py
dependencies: requirements.txt
description: Sends an email over SMTP
category: communication
input_schema:   { type: object, properties: { to: {type: string}, subject: {type: string}, body: {type: string} }, required: [to, subject] }
output_schema:  { type: object, properties: { messageId: {type: string} } }
config_schema:  { type: object, properties: { smtpHost: {type: string} } }
secrets: [SMTP_PASSWORD]
resources: { cpu: 0.5, memoryMb: 256, timeoutMs: 15000 }
scaling: { minInstances: 0, maxInstances: 5, concurrency: 10 }
```

### 4.4 Shim protocol (inside every node container)

- `GET /healthz` → `200`
- `POST /invoke` body: `{ "executionId", "nodeRunId", "nodeId", "workspaceId", "input", "config", "connections", "secrets" }`
- Response: `{ "ok": true, "output": {...}, "durationMs": 12 }` or `{ "ok": false, "error": { "type", "message", "trace" } }`
- Shim writes one JSON log line per event to stdout: `{ "ts", "level", "msg", "executionId", "nodeRunId", "nodeId" }`.

### 4.5 RuntimeGateway (Java)

```java
public interface RuntimeGateway {
    BuildResult buildImage(BuildSpec spec);
    InstanceHandle ensureInstance(NodeVersionRef node, ResourceSpec res, WorkspaceRef ws);
    InvocationResult invoke(InstanceHandle instance, InvocationRequest request, Duration timeout);
    void stopInstance(InstanceHandle instance);
    Stream<LogLine> streamLogs(InstanceHandle instance);
    InstanceStats stats(InstanceHandle instance);
}
```

`LocalDockerRuntime` implements it with `docker-java`. `RemoteWorkerRuntime` implements it by calling worker agents.

### 4.6 Master ↔ worker protocol (Phase 9)

- Worker registers: `POST /internal/workers/register` with join token → gets worker id and credentials.
- Heartbeat every 5 s with capacity (free CPU, memory) and container list.
- Master sends commands: start/stop container, pull image, fetch logs/stats. Worker never talks to other workers.
- Images are shared through a registry (local registry container in POC).
- If a worker misses N heartbeats, the master marks it `LOST` and reschedules its instances.

### 4.7 Workspace public URL (HTTP ingress)

Every workspace has its own **unique public URL**. HTTP Trigger nodes in that workspace's deployed flow are served under it, so the outside world can call the backend a user built.

- At creation each workspace gets a `public_id`: random, URL-safe, lowercase, at least 12 characters, **unique and immutable**. It is separate from the owner-facing `slug` and must not be derived from it or from user data.
- Public base URL, by `PUBLIC_URL_MODE`:
  - `path` (default for local dev): `{PUBLIC_BASE_URL}/run/{public_id}`, for example `http://localhost:8080/run/k3x9q2m7a1bz`
  - `subdomain` (production): `https://{public_id}.{PUBLIC_BASE_DOMAIN}`
- An HTTP Trigger's `path` and `method` are relative to that base. Trigger `POST /orders` is reached at `{base}/orders`.
- One ingress router (`engine/ingress`) resolves host or path to `public_id` → workspace → deployed flow version → trigger by method and path. Both URL modes use the same resolver.
- Only **deployed** flows respond. Unknown `public_id`, undeployed workspace, and unknown route all return the same `404` (do not leak which workspaces exist).
- Per-trigger access mode: `public` or `api_key` (key checked from a header; keys stored hashed).
- Limits per workspace: request rate, payload size, and concurrent executions.
- The UI shows the full public URL on the workspace page and in the HTTP Trigger config panel, with a copy button.

## 5. Execution flow (happy path)

1. HTTP request hits the workspace's public URL (`/run/{public_id}/{path}` in path mode, or `{public_id}.{PUBLIC_BASE_DOMAIN}/{path}` in subdomain mode) on the master.
2. `engine` finds the deployed flow version and the matching trigger, creates an `execution`.
3. Engine builds the DAG, runs ready nodes on virtual threads.
4. For each node: resolve input expressions → validate against schema → `RuntimeGateway.ensureInstance` → `invoke` → validate output → persist `node_run`.
5. Logs stream from containers into `log_lines`; SSE endpoint fans them out.
6. The HTTP Response node produces the HTTP response; execution is marked complete.

## 6. Repository layout (where to find what)

```
cloud-base/
├── AGENTS.md                         Agent entry point (read first)
├── docs/
│   ├── requirements.md               What the product does
│   ├── architecture.md               This file
│   ├── rules.md                      Coding rules
│   ├── phases.md                     Build order + status checklist
│   └── adr/                          Architecture decision records (NNNN-title.md)
├── backend/
│   ├── pom.xml                       Parent POM
│   ├── cloudbase-protocol/           Shared DTOs for master↔worker
│   ├── cloudbase-server/
│   │   └── src/main/java/com/cloudbase/server/
│   │       ├── CloudBaseApplication.java
│   │       ├── auth/                 api/ domain/ service/ repo/ security/ (users, tokens, Google)
│   │       ├── workspace/            workspaces, secrets
│   │       ├── flow/                 flows, versions, graph validation, deploy
│   │       ├── nodelib/              node definitions/versions, manifest parsing
│   │       ├── build/                image build orchestration, build logs
│   │       ├── engine/               DAG executor, expression resolver, retries, node executors, ingress/ (public URL router)
│   │       ├── runtime/              RuntimeGateway, LocalDockerRuntime, warm pool, profiles
│   │       ├── infra/                database nodes, custom container nodes, connection injection
│   │       ├── observability/        log collector, metrics binders, SSE streams
│   │       ├── marketplace/          plugin catalog, install
│   │       ├── ai/                   LLM client, prompt-to-flow, code assist
│   │       ├── workers/              worker registry, scheduler (master side)
│   │       └── shared/               errors, tenant/owner context, events, utilities
│   │   └── src/main/resources/
│   │       ├── application.yml
│   │       └── db/migration/         Flyway: V001__*.sql ...
│   └── cloudbase-worker/             Worker agent (Phase 9)
├── runtime/
│   ├── profiles/python312/           Dockerfile.base, shim/, installer script
│   └── README.md                     How to add a runtime profile
├── plugins/official/<slug>/          plugin.yaml, main.py, requirements.txt, tests/
├── frontend/
│   └── src/
│       ├── app/                      routing, providers
│       ├── features/                 auth/ workspaces/ canvas/ node-builder/ marketplace/ executions/
│       ├── components/
│       │   ├── ui/                   shadcn/ui components (added via the shadcn CLI, owned by this repo)
│       │   └── ...                   app-level shared components composed from components/ui
│       ├── lib/                      utils (e.g. `cn()` class-name helper), shared helpers
│       └── api/                      generated/typed API client
├── deploy/
│   ├── docker-compose.yml            postgres, redis, server, frontend, prometheus, grafana
│   ├── prometheus/prometheus.yml
│   └── grafana/dashboards/*.json
└── .github/workflows/                CI
```

Where to find common things:

| Need | Location |
|---|---|
| Add a REST endpoint | `<module>/api/` controller + DTOs; document in OpenAPI |
| Add a DB table | new Flyway file in `backend/cloudbase-server/src/main/resources/db/migration/` |
| Change how nodes run | `runtime/` and `engine/`; shim in `runtime/profiles/` |
| Add a built-in node | `engine/` node executor + frontend palette entry + docs update |
| Add an official plugin | `plugins/official/<slug>/` + seed migration |
| Add a runtime language | `runtime/profiles/<lang>/` + profile registration in `runtime/` |
| Auth rules / security config | `auth/security/` |
| Metrics / dashboards | `observability/` and `deploy/grafana/` |

## 7. API conventions

- Base path `/api/v1`. Internal worker endpoints under `/internal`.
- JSON, camelCase fields, ISO-8601 UTC timestamps, UUID or prefixed IDs.
- Auth: `Authorization: Bearer <access token>`.
- Errors: RFC 7807 `ProblemDetail` with a stable `code`.
- Pagination: `?page=&size=`, response includes `total`.
- Every endpoint enforces ownership (`owner_id == current user`) in the service layer.
- Public runtime endpoint for deployed flows: the workspace's unique public URL (see §4.7), `/run/{public_id}/**` in path mode (auth via API key or public, per trigger setting).

## 8. Configuration

- `application.yml` with profiles `local`, `test`, `prod`; secrets via env vars only.
- Key env vars: `DB_URL`, `DB_USER`, `DB_PASSWORD`, `REDIS_URL`, `JWT_PRIVATE_KEY`, `JWT_PUBLIC_KEY`, `GOOGLE_CLIENT_ID`, `SECRETS_ENCRYPTION_KEY`, `DOCKER_HOST`, `RUNTIME_MODE` (`local`|`remote`), `PUBLIC_URL_MODE` (`path`|`subdomain`), `PUBLIC_BASE_URL`, `PUBLIC_BASE_DOMAIN`, `LLM_API_KEY`.

## 9. Security model summary

- Access token 15 min, refresh token 14 days with rotation and reuse detection.
- Passwords: BCrypt cost ≥ 12 (or Argon2id).
- Node containers: non-root, read-only root fs with a writable tmp, CPU/memory/pids limits, no network by default (allowed only to attached connections and, if enabled per node, a controlled egress), no Docker socket, dropped capabilities, timeouts enforced by the engine.
- Secrets encrypted with AES-GCM; decrypted only when building an invocation payload.

## 10. Decisions log

Significant decisions go in `docs/adr/NNNN-title.md`. Initial decisions:
1. Modular monolith over microservices for the POC.
2. Container per node instance for isolation; virtual threads only for engine-side dispatch.
3. Build-time dependency installation, warm pools for execution.
4. RuntimeGateway abstraction to enable master/worker without engine changes.
5. Connection edges separate from data edges.
