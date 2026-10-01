# AGENTS.md — Start Here

You are working on **Cloud Base**: a low-code "backend as plugin" platform. Users build a backend by wiring nodes (Python functions, databases, custom containers) on a canvas. Each node runs in its own isolated container. A master manages workers that run those containers (single-node first). Server: Java 21 + Spring Boot. Node code: Python. Frontend: React + TypeScript.

This file tells you **how to work**. The details live in the docs below.

## 1. Read these first, in order

1. `docs/phases.md` — find the **current phase** and the next unchecked task. This decides what you may do.
2. `docs/requirements.md` — what the product must do (requirement IDs like `FR-AUTH-4`).
3. `docs/architecture.md` — stack, repo layout, contracts, data model. Use it to find where code belongs.
4. `docs/rules.md` — mandatory coding, security, testing and git rules.
5. `docs/adr/` — past design decisions (if the folder exists).

Do not start coding before you have read the sections of these files relevant to your task.

## 2. Ground rules (summary — full list in `docs/rules.md`)

- **One phase at a time. One task at a time.** Never build future-phase features, and never try to build the whole system at once.
- **Ask when unsure.** If a requirement is ambiguous, conflicts with another doc, or needs a design choice not covered, stop and ask. Do not invent requirements.
- **User code never runs in the server JVM.** Node code runs only in sandboxed containers created through `RuntimeGateway`.
- **Tenant safety first.** Every query on user data filters by owner; every new resource gets a cross-user access test.
- **Respect module boundaries** (Spring Modulith). No imports from another module's internals.
- **Tests are part of the task.** Code without tests is not done.
- **Never commit secrets**, never edit applied Flyway migrations, never disable checks to get a build green.

## 3. Workflow for every task

1. **Pick** the next unchecked task in the current phase of `docs/phases.md`. Do not skip ahead.
2. **Plan** briefly: list files you will create or change, the requirement IDs covered, the tests you will write, and any open questions. If there are open questions, ask before coding.
3. **Branch**: `phase-<n>/<short-task-name>`.
4. **Implement** in small steps. Keep the build green after each step.
5. **Test**: write unit and integration tests alongside the code (see `docs/rules.md` §7).
6. **Verify** with the commands in §4. Everything must pass.
7. **Update docs** if you added a module, table, endpoint family, contract, or made a design decision (`architecture.md`, ADR). Tick the task in `docs/phases.md` and note anything deferred.
8. **Commit** using `<type>(<module>): <summary> [FR-XXX-N]`.
9. **Report** using the format in §6.

When the last task and all exit criteria of a phase are ticked, update the **Current phase** line at the top of `docs/phases.md` and stop. Wait for confirmation before starting the next phase.

## 4. Commands

Adjust only if the build tooling changes, and update this section when it does.

| Purpose | Command |
|---|---|
| Start infra (Postgres, Redis, Prometheus, Grafana) | `docker compose -f deploy/docker-compose.yml up -d` |
| Build + unit tests (no Docker needed) | `./mvnw -f backend/pom.xml verify -DexcludedGroups=docker` |
| Docker-dependent tests | `./mvnw -f backend/pom.xml verify -Dgroups=docker` |
| Run server (local profile) | `./mvnw -f backend/cloudbase-server/pom.xml spring-boot:run -Dspring-boot.run.profiles=local` |
| Frontend dev server | `pnpm --dir frontend dev` |
| Frontend tests + lint | `pnpm --dir frontend test && pnpm --dir frontend lint` |
| Python shim / plugin tests | `pytest runtime/ plugins/` |
| API docs | `http://localhost:8080/swagger-ui` |

A task is not finished until the relevant commands pass.

## 5. Where things go (quick map; full map in `docs/architecture.md` §6)

- New REST endpoint → `backend/cloudbase-server/src/main/java/com/cloudbase/server/<module>/api/`
- New table → new Flyway file in `backend/cloudbase-server/src/main/resources/db/migration/`
- Anything that talks to Docker → `runtime/` module only (never `engine/`)
- Language-specific node code (shim, base image) → `runtime/profiles/<language>/`
- Official plugin → `plugins/official/<slug>/`
- Frontend feature → `frontend/src/features/<feature>/`
- Design decision → `docs/adr/NNNN-title.md`

## 6. How to report back

After finishing a task (or when blocked), reply with:

1. **Done:** what you built, in two or three sentences.
2. **Requirements covered:** IDs.
3. **Files:** key files added/changed.
4. **Tests:** what you added and the result of the commands you ran.
5. **Decisions / deferred:** anything you chose or postponed, and why.
6. **Next task:** the next unchecked item.

If blocked, state the blocker, what you tried, and the specific question you need answered.

## 7. Stop and ask before…

- Adding a new dependency or framework not listed in `docs/architecture.md`.
- Changing a contract (flow graph JSON, node manifest, shim protocol, `RuntimeGateway`, API shapes already released in a completed phase).
- Changing the data model beyond the current task.
- Touching security-sensitive code (auth, token handling, sandbox limits, secrets) in a way the task does not call for.
- Anything that would make a previously completed phase's exit criteria fail.

## 8. Quality bar

- Readable over clever. Small classes and methods, clear names, no dead code.
- Errors are explicit, typed, and mapped to `ProblemDetail`.
- Logs are structured and never contain secrets, tokens, or passwords.
- Every endpoint has an authorization rule and is documented in OpenAPI.
- Prefer boring, well-known solutions; this project is meant to be reviewed by humans.
