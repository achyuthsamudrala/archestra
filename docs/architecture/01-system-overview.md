# System Overview

Archestra is an open-source enterprise AI platform: a single deployable system that gives an
organization one URL for chat, LLM access, and MCP (Model Context Protocol) tool access, with
enterprise security (SSO, RBAC), guardrails (Dual-LLM, Lethal Trifecta protections), sandboxed code
execution, and observability (OpenTelemetry, Prometheus) built in rather than bolted on.

This document is the entry point to `docs/architecture/` — an internal, code-grounded documentation
set for engineers who want to understand how the system is built and why, distinct from the
user/admin-facing product docs in [`docs/pages/`](../pages/). Every other file in this set drills into
one subsystem in depth; this file gives the map.

## Repository layout

The repository is a monorepo with three largely independent top-level projects plus shared licensing:

```
archestra/
├── platform/        the product itself — backend, frontend, native Rust, infra, tests
├── ai-labs/         a separate Rust workspace: archestra-bench, an internal agentic eval harness
├── migration-kit/   a zero-dependency Python skill that migrates other agent setups into Archestra
├── docs/            documentation — docs/pages (product docs site) and docs/architecture (this set)
├── LICENSE.md        dual-license router (AGPL-3.0 default, Enterprise License for tagged code)
└── README.md
```

`platform/` is where almost all product engineering happens and is what the rest of this document
focuses on. `ai-labs/` and `migration-kit/` are covered in their own files
([13](./13-ai-labs-benchmark.md), [14](./14-migration-kit.md)) because they are structurally separate
Rust/Python projects with their own build and release lifecycles, not workspaces of the `platform/`
pnpm monorepo.

## What the platform actually is

Read literally, "Archestra" is the union of several gateways and runtimes sitting in front of a
Postgres database, all reachable behind one auth boundary:

- A **chat UI** for end users, backed by **Agents** — named configurations of a system prompt, an LLM
  model, a set of assigned MCP tools, and optional Skills.
- An **LLM proxy** (`/v1/...` style routes) that fronts any supported provider (Anthropic, OpenAI,
  Azure, Bedrock, DeepSeek, …) behind one API surface, enforcing cost limits, virtual API keys, and
  security policies, and emitting traces/metrics for every call. See
  [LLM Proxy & Gateway](./04-llm-proxy-and-gateway.md).
- An **MCP gateway** (`/v1/mcp/:profileId`) that authenticates tool calls, enforces tool-invocation and
  trusted-data policies, applies response modifiers, and logs every call — in front of both remote MCP
  servers and local MCP servers that Archestra itself runs as Kubernetes pods (the **orchestrator**).
  See [MCP Gateway & Orchestrator](./05-mcp-gateway-and-orchestrator.md).
- A **sandboxed code execution runtime** (Dagger-backed, driven through MCP tools) for Agent Skills —
  reproducible, replayable, per-conversation. See
  [Agents, Skills & Sandbox](./07-agents-skills-sandbox.md).
- A **security layer** — RBAC, tool invocation policies, trusted data policies, a Dual-LLM verification
  pattern, and Lethal-Trifecta detection — that sits across the LLM proxy and MCP gateway rather than
  inside either one. See [Security & Guardrails](./06-security-and-guardrails.md).
- An **identity and access layer** — Better-Auth sessions, SSO (OIDC/SAML/Okta/Entra), team-based
  resource access, and a fully custom RBAC system (30 resources × CRUD, unlimited custom roles) — see
  [Backend Architecture](./02-backend.md).
- **Observability** wired through every one of the above: OpenTelemetry traces, Prometheus metrics on a
  dedicated port, Grafana/Tempo for visualization. See [Observability](./10-observability.md).

Nothing here is a separate product; they are views onto the same Postgres-backed backend, which is why
the backend document is the one most other documents point back to.

## The two runtime processes

At its simplest, a running Archestra instance is two Node.js processes plus Postgres plus (optionally)
Kubernetes:

```
                 ┌─────────────────────────┐
 browser  ─────► │  frontend (Next.js)     │  port 3000
                 │  platform/frontend      │
                 └───────────┬─────────────┘
                              │ generated API client (OpenAPI) + streaming fetch
                              ▼
                 ┌─────────────────────────┐        ┌───────────────────┐
 API clients ──► │  backend (Fastify)      │ ─────► │ PostgreSQL        │
 (Claude Code,   │  platform/backend       │        │ (+ pgvector for   │
  Cursor, etc.)  │  port 9000              │        │  embeddings)      │
                 │  metrics on port 9050   │        └───────────────────┘
                 └───────────┬─────────────┘
                              │ (when K8s configured)
                              ▼
                 ┌─────────────────────────┐
                 │ Kubernetes namespace     │  one pod per local MCP server;
                 │ (orchestrator)           │  Dagger engine for skill sandboxes
                 └─────────────────────────┘
```

The backend is the system's center of gravity: it owns the database, the LLM proxy, the MCP gateway,
auth, and the orchestrator client. The frontend is a conventional Next.js app that talks to the backend
exclusively through a generated OpenAPI client — see [Frontend](./08-frontend.md). Native functionality
that needs to live outside the Node.js/V8 sandbox (image processing, sandboxed code execution
plumbing, an embedded runtime) is implemented in Rust and exposed to the backend via NAPI bindings
under `platform/archestra-rs/` — see [Native Rust Components](./09-native-rust-components.md).

Kubernetes is used for two distinct purposes that are easy to conflate: locally, Tilt uses it to
orchestrate the dev environment (Postgres, observability stack, etc.); in any environment, the backend
itself uses it as the **runtime for local MCP servers and the code-execution sandbox**, provisioning one
pod per MCP server on demand. See [Deployment & Infrastructure](./11-deployment-and-infrastructure.md).

## Request shape: how a tool call actually happens

A common point of confusion: the LLM proxy does **not** execute tools itself. Tool execution is the
client's responsibility, exactly as the OpenAI/Anthropic APIs already work — Archestra's proxy is a
transparent-but-policed layer in that loop, not a new agentic runtime bolted on top of it:

1. Client calls the LLM proxy with a `tools` list; the proxy forwards to the configured provider (after
   applying cost-limit and virtual-key checks) and returns whatever `tool_use`/`tool_calls` the model
   produced.
2. The client executes each tool call itself by POSTing to the MCP gateway
   (`POST /v1/mcp/:profileId`, `Authorization: Bearer <archestraToken>`). The gateway resolves that
   profile's tool surface, enforces tool-invocation and trusted-data policies, runs the call (against a
   remote server, or a local K8s pod, or the built-in `archestra__*` tools), applies response
   modifiers, and logs the result.
3. The client sends the tool result back to the LLM proxy and repeats until the model produces a final
   answer.

The in-product chat UI is just one such client: it drives this loop against the same public LLM proxy
and MCP gateway surfaces that an external client (Claude Code, Cursor, a custom agent) would use. This
is a deliberate design constraint — see the tradeoffs discussion in
[LLM Proxy & Gateway](./04-llm-proxy-and-gateway.md) — and it's why the security guardrails live at the
proxy/gateway boundary rather than inside a "the platform's own agent loop": every client, first-party
or not, passes through the same enforcement points.

## Licensing as an architectural constraint

The codebase is dual-licensed, and that fact shapes how code is organized, not just how it's
distributed. [`LICENSE.md`](../../LICENSE.md) is a router: AGPL-3.0 is the default; a file, region, or
path can be marked `LicenseRef-Archestra-Enterprise` via an SPDX header, an in-file
`SPDX-SnippetBegin`/`SPDX-SnippetEnd` region, or by naming convention (`*.ee.{ts,tsx,py,rs,...}` or
anywhere under an `ee/` directory). This means Enterprise-gated functionality is interleaved with
open-source code at the file and even line level rather than isolated in a separate private repo — a
choice with real consequences for how contributors read and extend the codebase, discussed further in
[Design Decisions & Tradeoffs](./15-design-decisions-and-tradeoffs.md).

## Layering discipline

The one architectural rule repeated in every corner of the backend
(`platform/spec/architecture.md`, `platform/backend/architecture.md`, `platform/CLAUDE.md`) is a strict
one-way layering:

```
routes → services → models → database
```

Routes parse/validate (Zod via `fastify-type-provider-zod`) and serialize; they hold no business logic
and touch the database only through a service (or directly through a model, if the whole route is one
model call). Services hold business logic, cross-model orchestration, and transactions. Models are the
only code allowed to issue Drizzle queries, one file per table, and never call services — imports only
ever point "down" the stack. This single rule is why the backend document
([02](./02-backend.md)) and the data-model document ([03](./03-database-and-data-model.md)) can be read
mostly independently: the schema defines *what* exists, the layering rule defines *who is allowed to
touch it*.

## How to read the rest of this set

| # | File | Covers |
|---|------|--------|
| 02 | [Backend Architecture](./02-backend.md) | Fastify app, routing/RBAC, config, layering |
| 03 | [Database & Data Model](./03-database-and-data-model.md) | Postgres + Drizzle, schema, migrations |
| 04 | [LLM Proxy & Gateway](./04-llm-proxy-and-gateway.md) | Provider adapters, streaming, cost/virtual keys |
| 05 | [MCP Gateway & Orchestrator](./05-mcp-gateway-and-orchestrator.md) | MCP auth, K8s pod-per-server runtime |
| 06 | [Security & Guardrails](./06-security-and-guardrails.md) | Tool policies, Dual-LLM, Lethal Trifecta |
| 07 | [Agents, Skills & Sandbox](./07-agents-skills-sandbox.md) | Agents, Skills, Dagger sandbox, replay log |
| 08 | [Frontend](./08-frontend.md) | Next.js app, generated client, chat UI, white-labeling |
| 09 | [Native Rust Components](./09-native-rust-components.md) | `archestra-rs` NAPI crates |
| 10 | [Observability](./10-observability.md) | OpenTelemetry, Prometheus, Tempo, Grafana |
| 11 | [Deployment & Infrastructure](./11-deployment-and-infrastructure.md) | Docker, Helm, K8s, licensing enforcement |
| 12 | [Testing Strategy](./12-testing-strategy.md) | Vitest+PGlite, Playwright, WireMock, CI |
| 13 | [ai-labs / archestra-bench](./13-ai-labs-benchmark.md) | Internal agentic eval harness |
| 14 | [Migration Kit](./14-migration-kit.md) | Migrating other agent setups into Archestra |
| 15 | [Design Decisions & Tradeoffs](./15-design-decisions-and-tradeoffs.md) | Cross-cutting decisions log |

Each file is self-contained and cites concrete repo-relative file paths so you can jump straight into
the code; none of them attempt to restate the product docs in `docs/pages/` — read those first if you
want to know what a feature does for a user, read these if you want to know how it's built.
