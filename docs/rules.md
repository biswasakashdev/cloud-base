# Cloud Base — Rules for Coding Agents

These rules are mandatory. If a rule blocks you, stop and ask; do not work around it.

## 1. Scope and process

1. Work **only on the current phase** in `phases.md`. Never implement future-phase features "while you're there".
2. Do one **task** (one checklist item) at a time. Keep changes small and reviewable.
3. Before coding a task, restate its acceptance criteria (from `requirements.md` IDs) in your plan.
4. If requirements are ambiguous or conflict with `architecture.md`, **ask** instead of guessing. Record the answer in the relevant doc.
5. A task is **done** only when: code compiles, tests pass, migrations run on a clean DB, docs are updated, and the checklist item is ticked.
6. Never leave the build broken between tasks. No commented-out code, no TODOs without an issue reference and phase number.
7. Do not add dependencies without a one-line justification in the PR/commit message. Prefer what is already in `architecture.md`.
8. Do not rewrite or reformat unrelated files.

## 2. Java / Spring conventions

1. Java 21. Use records for DTOs and value objects, `var` sparingly, `Optional` only for returns.
2. Package by **feature module** (see `architecture.md`), not by technical layer at the top level. Inside a module: `api`, `domain`, `service`, `repo`.
3. **Module boundaries** (Spring Modulith): a module may only use another module's **public API package/interfaces or published events**. Never import another module's `repo` or `domain` internals. Add/keep a `ModularityTests` verifying this.
4. Controllers are thin: validate, call one service method, map to DTO. No business logic or repositories in controllers.
5. Never expose JPA entities from the API. Use DTO records with explicit mapping.
6. Constructor injection only. No field `@Autowired`. No static mutable state.
7. Transactions belong in the service layer. Keep them short; never hold a DB transaction while calling Docker or a node container.
8. Use `ProblemDetail` for every error. Define domain exceptions in `shared/errors` and map them in one `@ControllerAdvice`.
9. Validate all input with Bean Validation on DTOs plus explicit checks for business rules.
10. Configuration via `@ConfigurationProperties` records, never scattered `@Value`.
11. Time: `Instant`/UTC everywhere. Inject a `Clock` so logic is testable.
12. Code style: Google Java Format or the repo's formatter config; run the formatter and static checks before committing.

## 3. Database rules

1. All schema changes through **Flyway** migrations named `V<NNN>__<description>.sql`. Never edit an applied migration; add a new one.
2. Every user-owned table has `owner_id` and an index on it. Every query touching user data filters by owner.
3. Use UUID primary keys. Add foreign keys and sensible indexes. Use `TIMESTAMPTZ`.
4. No business logic in SQL triggers.
5. Store secrets only encrypted. Never store plain passwords or tokens (hash refresh tokens and API keys).

## 4. Security rules (non-negotiable)

1. **Never** trust client-supplied owner/user IDs. Derive identity from the authenticated principal.
2. Every endpoint has an explicit authorization rule; default deny.
3. Add a cross-user access test for every new resource type (user B cannot read/modify user A's data).
4. Never log passwords, tokens, secrets, full request bodies of auth endpoints, or user-supplied secrets.
5. **User code never runs in the server JVM.** Node code runs only in containers created through `RuntimeGateway`.
6. Node containers must be created with: non-root user, read-only root filesystem, CPU/memory/pids limits, dropped Linux capabilities, no Docker socket mount, no privileged mode, network disabled unless explicitly attached. Enforce these in one place (`runtime`); do not create containers elsewhere.
7. Treat plugin code, node code, and flow JSON as **untrusted input**. Validate schemas and size limits.
8. Do not build shell commands from user input. Use the Docker API and argument arrays.
9. Rate limiting on all auth endpoints.
10. Secrets, keys, and credentials come from environment variables. Never commit them; add a `.env.example` only.

## 5. Runtime and engine rules

1. The engine talks to containers **only** through `RuntimeGateway`. Nothing in `engine/` imports `docker-java`.
2. Nothing outside `runtime/profiles/<name>/` may assume a specific node language.
3. Install dependencies **at build time only**. Never run `pip install` (or equivalent) during an invocation.
4. Node images are tagged by content hash (runtime + dependencies + code + shim version) so identical builds are reused.
5. Every invocation has a timeout; every container has resource limits; every payload has a size limit.
6. Persist `node_runs` state transitions (`PENDING → RUNNING → SUCCEEDED|FAILED|TIMED_OUT|SKIPPED`) before and after the call.
7. The engine must be testable **without Docker** using a fake `RuntimeGateway`. Write engine tests against the fake first.
8. Idempotency: a retry must never duplicate an already-committed `node_run` result.
9. Clean up: containers, volumes, and images created for a workspace must be removable, and deletion paths must be tested.

## 6. API rules

1. Versioned base path `/api/v1`. Breaking changes need a new version or an ADR.
2. Every endpoint documented via OpenAPI annotations; the generated spec must build in CI.
3. Use proper HTTP status codes: `201` create, `204` delete, `400` validation, `401` unauthenticated, `403` forbidden, `404` not found (also for other users' resources), `409` conflict, `429` rate limited.
4. Lists are paginated.
5. No breaking changes to public runtime endpoints (`/run/...`) once a phase is complete without an ADR.

## 7. Testing rules

1. Every new service class has unit tests; every new endpoint has an integration test (MockMvc or REST Assured with Testcontainers Postgres).
2. Follow **Arrange–Act–Assert**; test names describe behavior: `refresh_reuse_revokes_token_family`.
3. Tests must be deterministic: no sleeps (use awaitility), no reliance on test order, no shared external state.
4. Docker-dependent tests are tagged `@Tag("docker")` and run in a separate CI job. Everything else must run without Docker.
5. Coverage goals: auth, engine, and graph validation ≥ 80% line coverage. Do not add trivial tests just to raise coverage.
6. A bug fix starts with a failing test that reproduces it.
7. Frontend: component tests for forms and canvas logic that matter; do not snapshot everything.

## 8. Logging, errors, observability

1. Use SLF4J with structured (JSON) logging in non-local profiles. Include `executionId`, `nodeRunId`, `workspaceId` via MDC where available.
2. Log levels: `ERROR` = needs attention, `WARN` = recoverable anomaly, `INFO` = lifecycle events, `DEBUG` = details.
3. Do not swallow exceptions. Either handle meaningfully or propagate with context.
4. Every new subsystem exposes at least one Micrometer metric and a health indicator where sensible.

## 9. Python (shim, official plugins) rules

1. Python 3.12, type hints required, format with `ruff`/`black`, lint with `ruff`.
2. Shim code stays small and dependency-light; it must not import user code at module import time in a way that lets user code crash the server loop.
3. Official plugins have a `plugin.yaml`, a `tests/` folder with `pytest` tests, and pinned dependency versions.
4. Plugins must never print secrets or read env vars other than through `ctx`.

## 10. Frontend rules

1. TypeScript strict mode; no `any` without a comment explaining why.
2. Server state via TanStack Query; local UI state via React state/Zustand. No duplicated server state.
3. Use the typed API client in `frontend/src/api`; do not scatter `fetch` calls.
4. The canvas graph model in the UI must serialize to exactly the flow JSON in `architecture.md`. If the format changes, update docs and backend validation in the same change.
5. Access tokens kept in memory; refresh via httpOnly cookie or secure storage per the auth design. Never put tokens in URLs.
6. Basic accessibility: labeled inputs, keyboard-usable forms.

## 11. Git and documentation

1. Branch per task: `phase-<n>/<short-task-name>`. Commit messages: `<type>(<module>): <summary>` (types: `feat`, `fix`, `test`, `docs`, `refactor`, `chore`). Reference requirement IDs, e.g. `feat(auth): rotate refresh tokens [FR-AUTH-4]`.
2. Update `architecture.md` when you add a module, table, endpoint family, or contract. Update `requirements.md` only when requirements truly change (ask first).
3. Tick items in `phases.md` as you finish them; add notes for anything deferred.
4. Significant design choices get an ADR in `docs/adr/`.
5. README stays accurate: how to run, test, and demo.

## 12. Things agents must never do

- Skip a phase or start building the whole system at once.
- Run node/plugin code in the server process.
- Disable security checks, tests, or Modulith verification to make a change pass.
- Commit secrets, keys, or `.env` files.
- Edit applied Flyway migrations.
- Add cross-module imports that bypass module APIs.
- Invent requirements. Ask.
