# Design Decisions & Tradeoffs

Every other file in this set ends with its own local "Design Decisions & Tradeoffs" section — the
specific tradeoff of choosing Fastify, or Drizzle, or pod-per-MCP-server. This file is different: it
reads across all of them and names the handful of *recurring* engineering positions that show up again
and again, in different subsystems, written by different parts of the team, at different times. A
single decision is a fact about one module. A position that repeats six times across a database schema,
a Rust sandbox, a benchmark harness, and a migration tool is a fact about how this engineering
organization thinks. Those are the ones worth naming explicitly, because they predict how *unwritten*
parts of the codebase will probably be built too.

Each theme below cites the specific decisions it's drawn from, in the per-subsystem docs, so you can go
read the concrete code behind the pattern.

## 1. Deterministic control is preferred over probabilistic judgment, everywhere a security or grading decision is made

This is the single most consistent position in the codebase. Every place the system has to decide
"is this allowed" or "is this correct," it reaches for a rule a human can read and predict ahead of
time, not a model call:

- **Tool invocation policies and trusted data policies** are condition-based (`key`/`operator`/`value`
  triples), not LLM-judged moderation. The product docs say this outright: "Many platforms use
  probabilistic LLM guardrails... they are not ideal as the final control plane for tool execution."
  See [Security & Guardrails](./06-security-and-guardrails.md).
- **The Dual-LLM quarantine agent's only output channel is a bounded integer** (`z.object({ answer:
  z.number().int() })`) — the one place in the whole system where an LLM's judgment is deliberately
  reduced to "which of these N options," specifically because free text from a model that just read
  untrusted content is the exact channel the pattern exists to close. See
  [Security & Guardrails](./06-security-and-guardrails.md).
- **RBAC permission checks are a static map lookup** (`requiredEndpointPermissionsMap`,
  `TOOL_PERMISSIONS`), fail-closed on any missing entry, never a heuristic. See
  [Backend Architecture](./02-backend.md), [MCP Gateway & Orchestrator](./05-mcp-gateway-and-orchestrator.md).
- **The internal benchmark's grading is an out-of-band pytest verifier**, structurally unreachable
  from the agent's own tool loop, rather than an LLM-judge grading the transcript. See
  [ai-labs / archestra-bench](./13-ai-labs-benchmark.md).
- **Sandbox execution is auditable by replaying a deterministic log**, not by trusting whatever state
  a live container happens to be in. See [Agents, Skills & Sandbox](./07-agents-skills-sandbox.md).

The cost is consistent too: every one of these mechanisms is *less expressive* than an LLM-based
equivalent would be. A condition-based policy cannot express "does this look like a phishing email." A
bounded-integer quarantine channel cannot summarize a document. An out-of-band verifier cannot grade
open-ended creative quality. The team has repeatedly chosen a narrower, auditable, race-free mechanism
over a more flexible, nondeterministic one, in a domain (agentic AI tool execution) where almost every
competing product reaches for the opposite tradeoff.

## 2. Layering rules are stated as absolute, enforced by convention, and explicitly acknowledged as compiler-unenforced

`routes → services → models → database` (`platform/spec/architecture.md`), model-only database access,
services never importing routes, the license router's AGPL/Enterprise boundary, the
`requiredEndpointPermissionsMap` completeness requirement, the core/adapter split in the Rust crates
(`*-core` has no NAPI/Node awareness) — every one of these is a *hard* rule stated in `CLAUDE.md`,
skill files, or architecture docs, and every one of them is enforced by human review and codified
workflow (skills, PR checklists), not by an import-linter, a module boundary tool, or the type system.
See [Backend Architecture](./02-backend.md) §Design Decisions, and the Rust core/adapter split in
[Native Rust Components](./09-native-rust-components.md).

This is a deliberate bet, not an oversight: adding real enforcement (an import-graph linter, a
module-boundary compiler plugin) has a setup and maintenance cost of its own, and the team has instead
invested in making violations *visible fast* — a route missing from the permission map 403s immediately
in testing; a model calling a service would be caught in review because the pattern is so
consistently absent elsewhere. The tradeoff is that the guarantee is only as strong as review discipline
and the "does this match the canonical example file" habit (`platform/backend/CLAUDE.md` literally
tells contributors to copy `virtual-api-key.routes.ts`'s shape) — a rushed or unfamiliar contributor
*can* violate any of these rules and have TypeScript compile happily.

## 3. Postgres is the single source of truth; every other stateful system (Dagger, Kubernetes, Redis-like caches) is a derived, disposable cache

This shows up most sharply in the skill sandbox: `skill_sandboxes` and
`skill_sandbox_replay_events` in Postgres fully describe a sandbox's state, and Dagger's container is
rebuilt from that log on demand — the `skills-sandbox/README.md` says it plainly: "Dagger owns
ephemeral filesystem state. There is no retention guarantee." See
[Agents, Skills & Sandbox](./07-agents-skills-sandbox.md).

The same shape reappears in the MCP orchestrator: Kubernetes Pods/Deployments are provisioned
imperatively from backend-held state, not reconciled continuously from a CRD — the backend's own
database, not `etcd`, is the durable record of "what MCP servers should exist," and Kubernetes is asked
to converge to that record on specific triggering events (install, cold start, boot sweep) rather than
being watched continuously. See [MCP Gateway & Orchestrator](./05-mcp-gateway-and-orchestrator.md).

The payoff in both cases is the same: operational simplicity (no cluster-state reconciliation loop, no
container-drift repair story, ordinary Postgres backup/restore covers everything that matters) and
resilience to the derived layer disappearing (an evicted Dagger cache layer or a pod deleted
out-of-band is a slower rebuild, not data loss). The cost is also the same shape twice: reactions to
out-of-band changes in the derived layer are only as fast as the next triggering event, not
instantaneous — there is no controller watching Pod events, and a sandbox's replayed history is bounded
by how much a from-scratch rebuild can tolerate (`ARCHESTRA_SANDBOX_HISTORY_LIMIT`).

## 4. Isolation boundaries are never collapsed for convenience, even when collapsing them would simplify the code

A recurring shape: two checks that *could* have been merged into one flag are kept structurally
independent, because merging them would silently couple two different questions that need different
answers:

- **"Is this content trusted" vs. "is this caller authorized"** — Archestra's own built-in
  `archestra__*` tools skip tool-invocation/trusted-data *policy* evaluation (they're not
  attacker-controlled upstream content) but never skip the RBAC *permission* check. This exact
  independent-gates decision is documented three times, almost verbatim, in
  [Security & Guardrails](./06-security-and-guardrails.md),
  [MCP Gateway & Orchestrator](./05-mcp-gateway-and-orchestrator.md), and
  [Agents, Skills & Sandbox](./07-agents-skills-sandbox.md) — once per subsystem that has built-in,
  "trusted," but still RBAC-gated tools.
- **One Kubernetes Pod per MCP server**, rather than multiplexing several servers' processes inside a
  shared container, so one compromised or crashing server cannot affect another's process or network
  namespace — accepted at real, un-amortized infrastructure cost (no idle scale-to-zero). See
  [MCP Gateway & Orchestrator](./05-mcp-gateway-and-orchestrator.md).
- **A fresh, isolated backend + database per benchmark environment**, specifically so a task that looks
  like it passed because a *different* task happened to leave the right state behind is structurally
  impossible, not just discouraged. See [ai-labs / archestra-bench](./13-ai-labs-benchmark.md).
- **Fail-closed re-checks at sandbox execution time** instead of trusting a cached authorization
  decision from when a skill was first mounted, closing a permission-revocation race window at the cost
  of an extra query per call. See [Agents, Skills & Sandbox](./07-agents-skills-sandbox.md).
- **A Rust panic is contained at every NAPI boundary** (`catch_unwind`) rather than trusting Rust code
  never to panic, because in-process native code shares fate with the whole Node process unless
  explicitly firewalled. See [Native Rust Components](./09-native-rust-components.md).

The unifying idea: wherever an attacker (a prompt injection, a malicious MCP server, a revoked
permission, a native-code panic) could plausibly cross a boundary, the system pays the cost of keeping
that boundary real, even when a shared-fate shortcut would have been less code.

## 5. Enterprise licensing is interleaved into the codebase, not walled off into a separate repository

[`LICENSE.md`](../../LICENSE.md) routes AGPL-3.0 vs. `LicenseRef-Archestra-Enterprise` by SPDX header,
in-file REUSE snippet regions, or path convention (`*.ee.*`, any `ee/` directory) — file by file, and
sometimes region by region within one file. One binary, one Docker image, one CI pipeline builds both
license tiers together; there is no separate "enterprise build." `EnterpriseTierService` additionally
gates some Enterprise functionality by *live organization size* rather than solely by license key,
auto-enabling below a user-count threshold — see
[Deployment & Infrastructure](./11-deployment-and-infrastructure.md).

This is architecturally consequential, not just a legal formality: it means a contributor reading any
given file cannot assume its license from its directory alone, and nothing at the TypeScript module
level stops an AGPL file from importing an `.ee.ts` file's exports — the boundary is enforced by
license-router tooling and review, matching the same "stated as absolute, enforced by convention"
pattern from §2. The payoff is a single coherent codebase and release process instead of two diverging
trees; see the further discussion in
[Deployment & Infrastructure](./11-deployment-and-infrastructure.md) for the release-pipeline
consequences.

## 6. Supply-chain security is a build-system constraint, not a policy document

The same defensive posture — assume a freshly published, unreviewed artifact might be malicious —
appears at every layer that pulls in third-party code:

- **npm installs**: `ignoreScripts: true` blocks `pre/postinstall` code execution repo-wide, and
  `minimumReleaseAge: 10080` (7 days) blocks installing any package version younger than a week, tuned
  specifically to the empirical window in which most malicious npm releases get caught and pulled. See
  `platform/CLAUDE.md` and [Deployment & Infrastructure](./11-deployment-and-infrastructure.md).
- **Native Rust binaries**: never compiled from source at `pnpm install` time (which would itself be
  exactly the kind of install-time code execution `ignoreScripts` exists to prevent) — every `*-rs`
  crate ships as a prebuilt, platform-tagged `.node` binary built once in CI/Docker. See
  [Native Rust Components](./09-native-rust-components.md).
- **The migration kit's installer**: zero-dependency, stdlib-only Python, enforced permanently by a CI
  test (`test_zero_dependency.py`) rather than left as an easily-eroded convention — specifically
  because its target users are running it on locked-down or air-gapped enterprise hosts where a
  `pip install` dependency chain is either impossible or a security review blocker. See
  [Migration Kit](./14-migration-kit.md).

The common thread: whenever the team had a choice between "richer tooling, more convenient" and "fewer
moving parts an attacker or a compromised upstream package could exploit," they picked the latter, and
then went one step further to make the constraint *structurally enforced* (a CI test, a lockfile
setting) rather than a README instruction someone has to remember.

## 7. Provider/backend fidelity is chosen over a single normalized abstraction, at an accepted N-way maintenance cost

Three unrelated subsystems face the identical shape of decision — "support N different real-world
implementations of the same concept" — and all three resolve it the same way: give each one a
faithful, near-native adapter, and accept that N adapters is more code than one normalized shim:

- **The LLM proxy** gives each provider (Anthropic, OpenAI, Azure, Bedrock, DeepSeek, …) its own route
  that mirrors that provider's real wire format almost exactly, rather than forcing every client through
  one canonical schema — because provider SDKs are picky about exact response shape, and a lossy
  internal translation breaks silently. Translation only happens for the opt-in Model Router, and even
  there an exhaustiveness test (`provider-matrix.test.ts`) exists specifically to stop the N-way surface
  from silently drifting. See [LLM Proxy & Gateway](./04-llm-proxy-and-gateway.md).
- **The Rust sandbox backend** is a closed `enum Backend` dispatched via `match`, not `Box<dyn
  SandboxBackend>` — explicitly so that adding a second backend forces the compiler to flag every call
  site that needs updating, rather than risking a missed dynamic-dispatch registration. See
  [Native Rust Components](./09-native-rust-components.md).
- **The migration kit's entity mapping** treats each source concept (a subagent, a hook, a slash
  command) as its own explicit, individually-reviewed mapping rule into Archestra's model, rather than
  a generic "import anything that looks like a tool" pipeline. See [Migration Kit](./14-migration-kit.md).

## 8. First-party surfaces are just clients of the same public APIs — there is no privileged internal shortcut

The in-product chat UI executes tools by calling the same `POST /v1/mcp/:profileId` MCP gateway an
external client (Claude Code, Cursor, a custom integration) would call, authenticated the same way, and
gets its model responses from the same LLM proxy route external clients use. There is no server-side
"the platform's own agent loop" that bypasses the gateway and talks to tools directly. See
[System Overview](./01-system-overview.md) and [LLM Proxy & Gateway](./04-llm-proxy-and-gateway.md).

This single fact is *why* the guardrail architecture works the way it does: because tool-invocation and
trusted-data policies are enforced at the gateway/proxy boundary rather than "inside the agent," they
apply uniformly whether the caller is Archestra's own chat UI or a third-party integration nobody at
Archestra wrote. The cost, named explicitly in
[LLM Proxy & Gateway](./04-llm-proxy-and-gateway.md), is that policy enforcement has to happen at two
separate surfaces (the proxy checks tool calls the model *proposes*; the gateway checks tool calls a
client actually *executes*) rather than once in a single control point — a direct consequence of
refusing to give the first-party client special-cased, bypass-shaped access.

## 9. The system prefers to log the decision and stay auditable over hiding complexity behind silent behavior

Several unrelated features independently chose "make the non-obvious thing loudly visible in a durable
record" over "just do the smarter thing quietly":

- **Cost-optimization model substitution** persists `baselineModel` and `actualModel` as two distinct
  columns rather than overwriting the requested model in the log — silent substitution in the hot path,
  but never silent after the fact. See [LLM Proxy & Gateway](./04-llm-proxy-and-gateway.md).
- **402, not 429, for budget stops** — a status code choice made specifically so automatic SDK retry
  logic *doesn't* mask a budget block as a transient hang. See
  [LLM Proxy & Gateway](./04-llm-proxy-and-gateway.md).
- **The sandbox replay log** is, by construction, a complete queryable history of every command,
  upload, and mount a sandbox ever ran, in order — not just current-state. See
  [Agents, Skills & Sandbox](./07-agents-skills-sandbox.md).
- **The migration kit's final report** explicitly separates what migrated, what was skipped, what
  failed, and what needs manual review, rather than presenting a single "migration complete" result.
  See [Migration Kit](./14-migration-kit.md).

## 10. Known documentation drift found while building this document set

Writing this documentation required reading the actual code behind every claim in `platform/CLAUDE.md`
and related docs, rather than transcribing them — and that process surfaced a few places where the
project's own internal documentation has drifted from what the code now does. They're recorded here,
rather than silently "corrected away," because a drift-tracking note is itself useful signal about
where to double check before relying on `CLAUDE.md` at face value:

- **"MCP response modifiers (Handlebars.js)"** is listed as a current feature in `platform/CLAUDE.md`,
  but the feature was added and then fully removed (schema migration `0027_blue_komodo.sql` added the
  column, `0184_spicy_nocturne.sql` dropped it) with no corresponding doc update. See
  [Security & Guardrails](./06-security-and-guardrails.md).
- **Team-scoping junction table names have moved.** `platform/CLAUDE.md` refers to `profile_team` and
  `mcp_server_team`; the current schema has unified "profiles" into the `agents` table
  (`agentType: "profile"`) with a junction table actually named `agent_team`, and MCP servers use a
  direct single-team foreign key rather than a many-to-many junction table at all. See
  [Database & Data Model](./03-database-and-data-model.md).
- **Two URLs in `platform/CLAUDE.md`'s Key URLs list don't resolve as written**: white-labeling
  ("Appearance Settings") now lives at `/settings/organization`, not `/settings/appearance`; there is no
  standalone `/tools` page — tool-assignment pagination lives inside the MCP registry UI instead. See
  [Frontend](./08-frontend.md).
- **The MCP proxy route documented as `POST /mcp_proxy/:id`** does not exist under that literal path;
  the closest analogues are session-cookie-authenticated `POST /api/mcp/:agentId` and
  `POST /api/mcp/server/:mcpServerId`. The runtime manager itself lives at
  `backend/src/k8s/mcp-server-runtime/`, not `backend/src/mcp-server-runtime/` as referenced elsewhere.
  See [MCP Gateway & Orchestrator](./05-mcp-gateway-and-orchestrator.md).
- **TOON result conversion is an organization/team-level setting**, not a per-agent
  `convert_tool_results_to_toon` boolean as `platform/CLAUDE.md`'s shorthand implies — verified directly
  against `database/schemas/organization.ts` and `team.ts`. See
  [MCP Gateway & Orchestrator](./05-mcp-gateway-and-orchestrator.md).

None of these are architecturally significant on their own, but the pattern is worth naming per §1–§9
above: this is a codebase that treats its own internal documentation the way it treats everything else
— as something that should be regenerated or checked against ground truth, not trusted blindly. Reading
`CLAUDE.md` alongside the actual schema/route it describes, rather than instead of it, is the right
habit for exactly this reason.
