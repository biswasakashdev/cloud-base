# Cloud Base — Build Phases

> Build the project **one phase at a time**. Do not start a phase until the previous phase's exit criteria are met and ticked.
> Requirement IDs refer to `requirements.md`. Structure and contracts live in `architecture.md`. Rules in `rules.md`.

**Current phase: 0 — Foundation**  *(update this line when a phase is completed)*

How to use this file:
- Tick `[x]` items as they are finished and merged. Add a short note under an item if something was deferred or changed.
- A phase is complete only when **all tasks and all exit criteria** are ticked.
- "Not in this phase" lists things agents must **not** build yet.

---

## Phase 0 — Foundation

**Goal:** an empty but production-shaped skeleton that builds, tests, and runs with one command.

**Tasks**
- [ ] Create repo layout from `architecture.md` §6 (`backend/`, `frontend/`, `runtime/`, `plugins/`, `deploy/`, `docs/`).
- [ ] Maven multi-module parent (`cloudbase-protocol`, `cloudbase-server`) on Java 21, Spring Boot 3.x; formatter and static checks configured.
- [ ] Spring Modulith set up with empty module packages and a `ModularityTests` verification test.
- [ ] `application.yml` with `local`, `test`, `prod` profiles; `@ConfigurationProperties` for app settings.
- [ ] PostgreSQL + Flyway wired; first migration `V001__init.sql` (extensions, base conventions).
- [ ] Global error handling with `ProblemDetail`, request logging with correlation id, JSON logs in non-local profiles.
- [ ] Actuator health and Micrometer Prometheus endpoint; springdoc OpenAPI at `/swagger-ui`.
- [ ] `deploy/docker-compose.yml` with Postgres, Redis, Prometheus, Grafana (server runs from the IDE for now).
- [ ] Frontend skeleton (pnpm): Vite + React + TypeScript strict + Tailwind CSS + shadcn/ui + TanStack Query + router; typed API client folder.
- [ ] GitHub Actions CI: build + unit tests + frontend lint/test; Docker-tagged tests in a separate job.
- [ ] Root `README.md` with run/test instructions.

**Exit criteria**
- [ ] `docker compose up` starts infra; server boots and `/actuator/health` returns `UP`.
- [ ] CI is green on a clean clone.
- [ ] `ModularityTests` passes.

**Not in this phase:** any business feature.

---

## Phase 1 — Authentication

**Goal:** users can sign up, log in (email/password and Google), and call protected APIs safely. *(FR-AUTH-1…8)*

**Tasks**
- [ ] Migrations: `users`, `auth_identities`, `refresh_tokens`, `audit_log`.
- [ ] Email/password signup with validation and BCrypt (cost ≥ 12) or Argon2id. *(FR-AUTH-1)*
- [ ] Email/password login. *(FR-AUTH-2)*
- [ ] JWT access tokens (RS256, 15 min) via `oauth2-resource-server`; keys from env vars. *(FR-AUTH-4)*
- [ ] Refresh tokens: stored hashed, rotation on use, **reuse detection revokes the token family**. *(FR-AUTH-4)*
- [ ] Logout and logout-everywhere. *(FR-AUTH-5)*
- [ ] Google sign-in: verify Google ID token server-side, link accounts by verified email. *(FR-AUTH-3)*
- [ ] Profile endpoints: get/update display name, change password. *(FR-AUTH-6)*
- [ ] Rate limiting on auth endpoints (Bucket4j + Redis) and temporary lockout after repeated failures. *(FR-AUTH-7)*
- [ ] Audit events for signup, login, failed login, token reuse, password change.
- [ ] Current-user resolver in `shared/` so other modules never read identity from request bodies.
- [ ] Email verification and password reset with a logging mailer stub. *(FR-AUTH-8, stretch)*
- [ ] Frontend: signup, login, "Continue with Google", token handling, protected route wrapper, logout.
- [ ] Tests: unit tests for token and password services; Testcontainers integration tests for every endpoint.

**Exit criteria**
- [ ] Integration test proves refresh-token reuse revokes the family.
- [ ] Integration test proves unauthenticated calls get `401` and expired tokens are rejected.
- [ ] Rate-limit and lockout behavior covered by tests.
- [ ] A user can sign up, log in, and see a protected "me" page in the UI.
- [ ] OpenAPI documents all auth endpoints.

**Not in this phase:** orgs/teams, roles beyond a single `USER` role, social providers other than Google.

---

## Phase 2 — Workspaces and Flows

**Goal:** users own multiple workspaces, each with exactly one versioned flow graph. *(FR-WS-1…7)*

**Tasks**
- [ ] Migrations: `workspaces`, `workspace_secrets`, `flows`, `flow_versions`.
- [ ] Workspace CRUD scoped to owner; unique slug per owner. *(FR-WS-1)*
- [ ] Generate an immutable, random, unique `public_id` per workspace and expose the computed public URL in workspace responses and the workspace page. *(FR-WS-8)*
- [ ] Flow auto-created with each workspace (one-to-one). *(FR-WS-2)*
- [ ] Save flow graph → new immutable `flow_version`; list versions; fetch a version. *(FR-WS-3)*
- [ ] Graph validator: JSON schema check, acyclic data edges, valid node/edge references, required config, allowed trigger nodes. Returns precise error list. *(FR-WS-4)*
- [ ] Workspace secrets: AES-GCM encrypted at rest, write-only via API (never returned). *(FR-WS-6)*
- [ ] Deploy/undeploy state on the flow (marks a version deployed; no runtime effect yet). *(FR-WS-5, state only)*
- [ ] Delete workspace cascades flow data (container cleanup hook left as an interface). *(FR-WS-7)*
- [ ] Frontend: workspace list/create/rename/delete pages.
- [ ] Tests: cross-user access denied on every endpoint; validator has a table of valid/invalid graphs.

**Exit criteria**
- [ ] User B gets `404` for user A's workspace on every endpoint (tested).
- [ ] Invalid graphs are rejected with clear errors; valid ones save as new versions.
- [ ] Secrets never appear in any API response or log (tested).

**Not in this phase:** executing flows, canvas UI, node definitions.

---

## Phase 3 — Node Library, Builds, and Local Runtime

**Goal:** users create custom Python nodes, build them into isolated images, and test-run them. *(FR-NODE-1…7)*

**Tasks**
- [ ] Migrations: `node_definitions`, `node_versions`, `builds`.
- [ ] `runtime/profiles/python312/`: base Dockerfile, shim (`/healthz`, `/invoke`, JSON logs), dependency install script, shim tests.
- [ ] Manifest model and parser/validator (`node.yaml`) per `architecture.md` §4.3.
- [ ] Node definition CRUD (per user, available in all of the user's workspaces). *(FR-NODE-1, FR-NODE-6)*
- [ ] `RuntimeGateway` interface and `LocalDockerRuntime.buildImage` using `docker-java`; content-hash tagging so identical builds are reused. *(FR-NODE-3)*
- [ ] Async build orchestration with status and streamed/stored build logs.
- [ ] Publish → immutable `node_version` (image digest + schemas). *(FR-NODE-5)*
- [ ] `ensureInstance` / `invoke` / `stopInstance` in `LocalDockerRuntime` with all sandbox limits enforced in one place. *(FR-NODE-7)*
- [ ] Test-run endpoint: run a built node with sample input, return output and logs. *(FR-NODE-4)*
- [ ] Tests: shim tests (pytest); `@Tag("docker")` integration tests for build and invoke; sandbox-limit tests (non-root, no network, memory cap, timeout).

**Exit criteria**
- [ ] A custom Python node with a pip dependency builds once and can be test-run repeatedly without reinstalling.
- [ ] A node that exceeds timeout or memory is stopped and reported as failed.
- [ ] A node cannot reach the network or the Docker socket (tested).
- [ ] Rebuilding unchanged code reuses the cached image.

**Not in this phase:** flow execution, warm pool tuning, database nodes, marketplace.

---

## Phase 4 — Execution Engine

**Goal:** deployed flows run as DAGs with data mapping, branching, retries and persisted history. *(FR-EXEC-1…8, FR-NODE-8…10)*

**Tasks**
- [ ] Migrations: `executions`, `node_runs`.
- [ ] `FakeRuntimeGateway` for tests (configurable outputs, delays, failures).
- [ ] DAG builder and scheduler on virtual threads; parent-counter readiness; parallel branches. *(FR-EXEC-3)*
- [ ] Expression resolver for `{{ nodes.<id>.output... }}` (pick SpEL or CEL in an ADR). *(FR-NODE-9)*
- [ ] Built-in node executors: HTTP Trigger, Manual Trigger, HTTP Response, If/Switch, Merge, Set/Transform. *(FR-NODE-8)*
- [ ] Service node executor: resolve input → validate against schema → invoke → validate output.
- [ ] Per-node timeout, retries with backoff, concurrency limit. *(FR-NODE-10, FR-EXEC-5)*
- [ ] State machine persistence for `executions` and `node_runs`; restart-safe (no duplicate commits on retry). *(FR-EXEC-6)*
- [ ] Deploy now registers live routes; the workspace's public URL (`/run/{public_id}/**` in path mode, subdomain mode via config) triggers executions and returns the Response node's output; unknown or undeployed URLs return a uniform `404`; per-workspace rate and payload limits. *(FR-EXEC-1, FR-WS-5, FR-WS-8)*
- [ ] Manual run endpoint with JSON payload. *(FR-EXEC-2)*
- [ ] Payload size limits. *(FR-EXEC-8)*
- [ ] Execution history endpoints (list, detail with node runs).
- [ ] Warm pool v1: keep N containers per node version, reap idle, per-node concurrency cap.
- [ ] Tests: engine tests with the fake gateway (parallel branches, conditional routing, retries, timeout, failure propagation); end-to-end `@Tag("docker")` test with a real Python node.

**Exit criteria**
- [ ] A flow `HTTP Trigger → Python node → HTTP Response` works end to end over HTTP.
- [ ] Parallel branches run concurrently (verified by timing test).
- [ ] A failing node retries then fails the execution with a clear error.
- [ ] Killing and restarting the server does not lose execution history.
- [ ] Warm invocation overhead meets NFR-4 on a developer machine.

**Not in this phase:** log streaming UI, database nodes, workers, canvas UI.

---

## Phase 5 — Logs and Monitoring

**Goal:** every container's logs and health are visible. *(FR-OBS-1…5)*

**Tasks**
- [ ] Migration: `log_lines`.
- [ ] Log collector: stream container stdout (`logContainerCmd`), parse shim JSON lines, tag with workspace/execution/node run/node id, batch insert.
- [ ] REST endpoint for logs per node run and per execution (paginated).
- [ ] SSE endpoint for live log streaming of a running execution.
- [ ] Container stats poller (CPU, memory, restarts, status) exposed as Micrometer gauges labeled by node version.
- [ ] Engine metrics: executions started/succeeded/failed, node run latency histogram, queue depth, pool size, cold starts.
- [ ] Grafana dashboard JSON committed in `deploy/grafana/dashboards/`; Prometheus scrape config.
- [ ] Log retention setting (delete old `log_lines`).
- [ ] Tests: collector parsing tests; integration test that logs from a real node appear tagged correctly.

**Exit criteria**
- [ ] Running a flow produces per-node logs retrievable via REST and SSE.
- [ ] Grafana dashboard shows container CPU/memory and execution metrics.
- [ ] Logs never include workspace secrets (tested with a secret-printing node).

**Not in this phase:** log search/analytics, Loki, alerting.

---

## Phase 6 — Database and Custom Container Nodes

**Goal:** infrastructure nodes that connect to service nodes. *(FR-INFRA-1…4)*

**Tasks**
- [ ] Migrations: `container_instances` (if not present), infra node tables as needed.
- [ ] `infra/` module: node kinds `DATABASE` (PostgreSQL) and `CUSTOM_CONTAINER`.
- [ ] Managed Postgres container lifecycle with a named volume and generated credentials stored as secrets. *(FR-INFRA-1)*
- [ ] Connection edges: injection of `ctx.connections[alias]` into invocations of connected service nodes. *(FR-INFRA-2)*
- [ ] Private per-workspace Docker network so only attached nodes can reach each other.
- [ ] Custom container node: image, port, env, health check; controlled network access. *(FR-INFRA-3)*
- [ ] Update graph validator for `connection` edges.
- [ ] Cleanup on workspace delete/undeploy (containers, networks, optionally volumes).
- [ ] Tests: Python node reads/writes a connected Postgres; unattached nodes cannot reach it; cleanup removes everything.

**Exit criteria**
- [ ] A flow can create a table and insert/read rows through a connected database node.
- [ ] Credentials are never visible in API responses or logs.
- [ ] Deleting a workspace leaves no orphan containers or networks.

**Not in this phase:** Kafka/other brokers (design only), backups, database UI.

---

## Phase 7 — Plugin Marketplace

**Goal:** install ready-made nodes as plugins. *(FR-PLUG-1…5)*

**Tasks**
- [ ] Migrations: `plugins`, `plugin_versions`.
- [ ] Plugin package format and validator reuse of the node manifest (`plugin.yaml`).
- [ ] Official plugins in `plugins/official/`: HTTP Request, Send Email (SMTP), JSON Transform, LLM Chat (stub provider interface). Each with pinned deps and pytest tests.
- [ ] Seeder that loads the catalog into the DB on startup.
- [ ] Catalog endpoints: list/search/filter by category, plugin detail.
- [ ] Install endpoint: copies the plugin into the user's node library as a definition, triggers a build, records the plugin version for update checks.
- [ ] Update-available detection (version compare).
- [ ] Tests: seed idempotency; install creates a built, usable node; installing twice behaves predictably.

**Exit criteria**
- [ ] A user installs "HTTP Request" and uses it in a flow without writing code.
- [ ] All official plugins pass their own tests in CI.

**Not in this phase:** publishing by third parties, signing, reviews, payments.

---

## Phase 8 — Canvas UI

**Goal:** the full product experience in the browser. *(FR-UI-3…6)*

**Tasks**
- [ ] Canvas editor with React Flow: palette (built-ins, library, installed plugins), drag/drop, connect, delete, pan/zoom, minimap.
- [ ] Graph serializes to the exact flow JSON in `architecture.md`; client-side validation mirrors server rules; server errors map back to nodes.
- [ ] Node config side panel generated from `config_schema` (form renderer); input mapping editor with expression hints.
- [ ] Data edges vs connection edges visually distinct.
- [ ] Save (new version), deploy/undeploy, version history.
- [ ] Node builder: runtime selection, code editor (Monaco), dependencies, schemas, build status/logs, test-run panel. *(FR-UI-4)*
- [ ] Marketplace browser and install flow. *(FR-UI-5)*
- [ ] Executions page: history, per-node input/output, live log panel, container monitoring panel. *(FR-UI-6)*
- [ ] Workspace secrets management UI.
- [ ] Error/empty/loading states; basic accessibility pass.
- [ ] Frontend tests for canvas serialization and forms; one end-to-end Playwright test of build → deploy → call.

**Exit criteria**
- [ ] A new user can sign up, create a workspace, install a plugin, build a custom node, wire a flow, deploy it, call the endpoint, and see logs — all from the UI.
- [ ] Canvas JSON round-trips without loss (tested).

**Not in this phase:** real-time multi-user editing, mobile layout.

---

## Phase 9 — Master / Worker Distribution

**Goal:** run node containers on remote workers managed by the master. *(FR-SCALE-1…5)*

**Tasks**
- [ ] ADR: worker transport (REST vs gRPC) and image distribution (registry).
- [ ] Migrations: `workers`; worker assignment on `container_instances`.
- [ ] `cloudbase-protocol` DTOs for register, heartbeat, commands, results.
- [ ] `cloudbase-worker` agent: registers with a join token, sends heartbeats with capacity, executes start/stop/pull/logs/stats commands against local Docker.
- [ ] Master `workers/` module: registry, heartbeat monitor (LOST after N missed beats), join-token management.
- [ ] Scheduler: place instances by free CPU/memory; spread by workspace; avoid LOST workers.
- [ ] `RemoteWorkerRuntime` implementing `RuntimeGateway`; `RUNTIME_MODE=local|remote|hybrid`.
- [ ] Local registry container in Compose for image sharing.
- [ ] Rescheduling when a worker dies.
- [ ] UI: workers page (status, capacity, running containers).
- [ ] Tests: scheduler unit tests; Docker Compose integration test with master + 2 workers.

**Exit criteria**
- [ ] With two workers, node instances are spread across them and flows still execute.
- [ ] Stopping one worker causes its instances to be rescheduled and the next execution succeeds.
- [ ] Engine code is unchanged apart from configuration (proves the abstraction).

**Not in this phase:** autoscaling workers, Kubernetes backend, HA master.

---

## Phase 10 — AI Integration

**Goal:** AI features that feel native. *(FR-AI-1…3)*

**Tasks**
- [ ] LLM provider interface in `ai/` (one real provider + a fake for tests); keys from workspace secrets.
- [ ] Finish the LLM Chat plugin: prompt templates, text or JSON-schema output.
- [ ] Prompt-to-flow: description → draft graph JSON, validated by the Phase 2 validator; returned as a **proposal** only, with retry/repair on invalid output.
- [ ] AI code assist for node handlers: generate/fix from description, shown as a diff for user approval.
- [ ] UI: "Generate flow" dialog and code-assist panel with accept/reject.
- [ ] Guardrails: size limits, rate limits per user, no auto-deploy, prompts and outputs logged without secrets.
- [ ] Tests: validator rejects malformed AI output; fake provider drives deterministic tests.

**Exit criteria**
- [ ] A text description produces a valid draft flow the user can edit and deploy.
- [ ] Invalid AI output never reaches saved flows.

**Not in this phase:** autonomous agents that modify deployed flows.

---

## Phase 11 — Hardening, Docs, and Demo

**Goal:** make it reviewable and demonstrable.

**Tasks**
- [ ] Security review against `rules.md` §4: dependency scan, secrets scan, container sandbox re-test, authorization test sweep across all endpoints.
- [ ] Load test of a warm node path; record results in the README.
- [ ] Complete `docker compose up` that runs the full stack including frontend and a worker.
- [ ] README: architecture diagram, feature tour, screenshots/GIF, quick start, design decisions (link ADRs).
- [ ] Seed demo workspace (for example "Order API": trigger → validate → database → email).
- [ ] Clean up TODOs, dead code, and docs drift; refresh `architecture.md`.
- [ ] Tag `v0.1.0`.

**Exit criteria**
- [ ] A reviewer can clone the repo, run one command, and complete the demo script in under 10 minutes.

---

## Future backlog (do not build until requested)

- Additional runtime profiles: TypeScript, JavaScript, Go, Java *(FR-FUT-1)*
- Message broker nodes: Kafka, RabbitMQ, NATS *(FR-FUT-2)*
- Teams and organizations, sharing custom nodes *(FR-FUT-3)*
- Public plugin publishing with review and signing *(FR-FUT-4)*
- Kubernetes worker backend and worker autoscaling *(FR-FUT-5)*
- Cron triggers, webhooks with signature verification, flow templates
