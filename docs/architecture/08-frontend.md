# Frontend

## Role and tech stack

The frontend is a Next.js App Router application at `platform/frontend/`, served on port 3000 in development. It renders the entire operator/end-user surface — chat, MCP tool management, LLM/MCP gateway configuration, cost analytics, RBAC, white-labeling — and talks to the Fastify backend on port 9000 almost exclusively through Next.js rewrites (`platform/frontend/next.config.ts`), not through a custom API proxy layer of its own.

Key versions and libraries, from `platform/frontend/package.json`:

- **Next.js `16.3.0-canary.56`** (App Router, `output: "standalone"`), **React `19.2.4`**.
- **TanStack Query `^5.90`** for all server-state data fetching, **TanStack Table `^8.21`** for tabular UI.
- **Radix UI primitives** (`@radix-ui/react-*`, also re-exported as the `radix-ui` umbrella package) plus **shadcn/ui**-style generated components under `src/components/ui/` — `class-variance-authority`, `tailwind-merge`, `clsx` are the styling-composition primitives shadcn/ui components rely on.
- **Tailwind CSS v4** (`tailwindcss`, `@tailwindcss/postcss`) for styling; `tw-animate-css` for animation utilities.
- **react-hook-form `^7.71`** with `@hookform/resolvers` (Zod resolver) for forms; **`zod`** for schema validation (pulled from the pnpm catalog, so version-pinned repo-wide).
- **`@ai-sdk/react` (`^3.0.195`) / Vercel `ai`** for the chat streaming hook (`useChat`) — see [Chat and streaming UI](#chat-and-streaming-ui) below.
- **`@hey-api/openapi-ts`** generates the typed API client from the backend's OpenAPI spec (see next section) — this is not hand-maintained.
- **`@sentry/nextjs`** for error reporting, **`posthog-js`** for product analytics.
- **Biome** (not ESLint/Prettier) for linting/formatting, with a custom Grit plugin (`biome-plugins/icon-button-aria-label.grit`) enforced repo-wide via `platform/biome.json`.
- **Vitest** + Testing Library for unit tests, **Playwright** for e2e/integration tests, **MSW** (`msw`) for mocking API responses in both.
- **`knip`** to catch unused exports as part of `check:ci`.

For backend routing/permissions context behind these calls, see [Backend Architecture](./02-backend.md); for the LLM streaming protocol the chat UI consumes, see [LLM Proxy & Gateway](./04-llm-proxy-and-gateway.md).

## App Router structure

Everything lives under `platform/frontend/src/app/`. The directory is large and feature-organized rather than mirroring a flat URL list; folders map to the CLAUDE.md product URL map roughly 1:1, with a few nesting/naming differences worth calling out explicitly since they diverge from what's documented elsewhere in the repo:

- `chat/`, `chat/[conversationId]/`, `chat/new/`, `chat/browser-preview/[conversationId]/` — the `/chat` surface.
- `mcp/registry/`, `mcp/registry/[id]/`, `mcp/registry/catalog/`, `mcp/registry/installation-requests/`, `mcp/registry/new/`, `mcp/logs/`, `mcp/logs/[id]/`, `mcp/gateways/`, `mcp/tool-guardrails/` — MCP management.
- `llm/logs/`, `llm/logs/[id]/`, `llm/logs/session/`, `llm/model-providers/`, `llm/models/`, `llm/proxies/`, and a route group `llm/(costs)/` containing `costs/`, `limits/`, `optimization-rules/` — the route group's parentheses are stripped from the URL, so these serve `/llm/cost/statistics`-style paths without adding a `costs` path segment for every child.
- `settings/` with subpages `account/`, `agents/`, `api-keys/`, `auth/`, `environments/`, `github/`, `identity-providers/`, `knowledge/`, `llm/`, `mcp/`, `organization/`, `roles/`, `secrets/`, `security/`, `service-accounts/`, `service-accounts/[id]/`, `teams/`, `users/`. Tab visibility is computed in `settings/settings-tabs.ts` (`useSettingsTabs()`), gated per-tab by `usePermissionMap(requiredPagePermissionsMap)` from `@archestra/shared/access-control` and by a Vault-only check for the Secrets tab.
- **`/settings/appearance` does not exist as a route in the current tree.** White-labeling controls (logo, favicon, theme, app name, chat links, onboarding wizard, etc.) live inside `settings/organization/page.tsx`, rendered under the "Organization" settings tab — an "Appearance" `<h3>` section heading inside that page, not a separate route. This is a real discrepancy against the CLAUDE.md URL map and worth flagging to anyone navigating from stale docs or bookmarks.
- **`/tools` (server-side paginated tool management) does not exist as a top-level route either.** No `src/app/tools/` directory exists; tool assignment/pagination UI (`useAllProfileTools()` in `src/lib/agent-tools.query.ts`) is consumed from `mcp/registry/_parts/mcp-assignments-dialog.tsx` and agent-scoped tool views instead of a standalone `/tools` page. Treat the CLAUDE.md "Tools" URL entry as stale relative to the code.
- `agents/`, `apps/`, `a/[appId]/`, `a/catalog/[catalogId]/` (a chrome-less standalone app-run surface — `next.config.ts` permanently redirects the older `/apps/catalog/:catalogId/run` path here), `projects/`, `projects/[id]/`, `projects/[id]/schedules/`, `skills/`, `skills/new/`, `credentials/` (+ `oauth-clients/`, `virtual-keys/`), `connection/`, `knowledge/` (+ `connectors/`, `knowledge-bases/`), `messaging-channels/` (+ `a2a/`, `email/`, `ms-teams/`, `slack/`, `telegram/`), `audit/logs/`.
- `auth/` — sign-in, sign-up-with-invitation, SSO callback/linked-callback, and `api/auth/[...path]/route.ts`, the one hand-written API route (see below).
- `internal-test/msw-handlers/` — a dev-only route for exercising MSW handlers directly.

**Route-local organization convention**: files and folders prefixed with an underscore (`_parts/`, `_components/`) are Next.js "private folders" — excluded from routing, used to colocate a page's helper components/hooks next to it without creating an accidental route segment. `src/app/_parts/` holds app-shell-wide pieces (`app-shell.tsx`, `sidebar.tsx`, `theme-provider.tsx`, `query-client-provider.tsx`, `with-auth-check.tsx`, `with-page-permissions.tsx`, `websocket-initializer.tsx`, `posthog-provider.tsx`, `error-boundary.tsx`, `maintenance-mode-overlay.tsx`, `site-notification-bar.tsx`).

**`.query.ts` co-location**: many page/component directories interleave `foo.tsx` with `foo.query.ts` (the TanStack Query hooks for that feature) and `foo.test.ts`/`foo.test.tsx`. `page.tsx` vs `page.client.tsx` splits (seen throughout — `agents/`, `apps/`, `projects/`, `skills/`, `llm/logs/`) mark a Server Component wrapper (`page.tsx`, does an initial server-side data fetch) around a `"use client"` component (`page.client.tsx`, owns interactivity and re-fetches via TanStack Query) — see [The `initialData` hydration pattern](#the-initialdata-hydration-pattern) below.

### Layout hierarchy and app shell

`src/app/layout.tsx` is the single root layout for the entire app (no nested route-group layouts split the provider tree). It:

1. Registers 16 self-hosted white-label fonts via `next/font/local` (Lato, Inter, Open Sans, Roboto, Source Sans, JetBrains Mono, DM Sans, Poppins, Oxanium, Montserrat, Source Code Pro, Merriweather, Quicksand, Outfit, Plus Jakarta Sans, Libre Baskerville — plus a disc-glyph "secret mask" font for `.secret-masked` fields), all with `preload: false` because only one is active per organization theme and eager-preloading all of them would be wasted bandwidth.
2. Nests providers in this order: `MswInit` → `ArchestraQueryClientProvider` → `ChatProvider` → `ThemeProvider` (next-themes, light/dark) → `PostHogProviderWrapper` → `OrgThemeLoader` + `DynamicHead` (side-effect-only components) → `WithAuthCheck` → `WebsocketInitializer` → `AppShell` → `WithPagePermissions` → `{children}`.
3. `WithAuthCheck` (`src/app/_parts/with-auth-check.tsx`) is the client-side auth gate: it reads `useSession()` and `usePublicConfig()`, redirects unauthenticated users to `/auth/sign-in` (preserving `redirectTo`), and redirects authenticated users away from auth pages. It also owns the **dev-auto-login** trigger (see [Feature flags](#feature-flags-via-apiconfig)).
4. `AppShell` (`src/app/_parts/app-shell.tsx`) renders the persistent sidebar/nav chrome that wraps every authenticated page.

### The hand-written API route and the proxy layer

Almost every backend call goes through Next.js `rewrites()` in `next.config.ts`, not through code the frontend team maintains: `/api/:path*`, `/v1/:path*`, `/.well-known/:path*`, `/health`, `/_sandbox/:path*`, `/skills/m/:path*`, and `/ws` are all rewritten straight to `ARCHESTRA_INTERNAL_API_BASE_URL` (default `http://localhost:9000`). `experimental.proxyTimeout` is set to 300000ms (5 minutes) specifically so this rewrite proxy doesn't cut off long-lived SSE chat streams, and `experimental.proxyClientMaxBodySize` is bumped to `200mb` — comfortably above the backend's runtime-configurable `ARCHESTRA_API_BODY_LIMIT` default — because that limit is baked into the frontend image at build time (`output: "standalone"`) and can't be changed at deploy time the way the backend's can.

The one exception is `src/app/api/auth/[...path]/route.ts`, a real Next.js Route Handler. It exists (per its comment in `next.config.ts`) because API routes take precedence over `rewrites()`, and `/api/auth/*` needs custom handling for SAML SSO ACS callbacks — see `src/proxy.ts` below.

### `src/proxy.ts` — Next.js middleware

`src/proxy.ts` exports `proxy(req)`, Next's middleware entry point (the file is the modern replacement for `middleware.ts`). It does three things, none of which is general-purpose API proxying (that's `rewrites()`'s job):

1. Redirects `/` to `/chat` before any client component renders.
2. Rewrites SAML SSO ACS POST callbacks (`/api/auth/sso/saml2/sp/acs/*`) to replace a `null` `Origin` header with the real frontend origin — SAML IdPs POST via cross-origin form submission, browsers send `Origin: null` for that, and Better Auth rejects `Origin: null` with `MISSING_OR_NULL_ORIGIN`.
3. Injects `X-Forwarded-Host`/`X-Forwarded-Proto` onto `/v1/*`, `/.well-known/*`, and `/api/*` requests when not already set, so the backend's `getPublicRequestOrigin()` advertises the correct OAuth protected-resource/token/jwks origin. Without this, an MCP client connecting through the frontend origin (`localhost:3000`) would see the backend's own origin (`localhost:9000`) in OAuth metadata and fail with a resource mismatch.

It also does structured request logging (`console.log` gated by `shouldLogApiRequest`), deliberately suppressing noisy GET polling requests to `/v1/mcp/*`.

## Component organization

`src/components/` is a hybrid of feature-based and primitive/design-system organization:

- `components/ui/` — the shadcn/ui-style primitive layer: `button.tsx`, `input.tsx`, `dialog.tsx`, `dropdown-menu.tsx`, `data-table.tsx` (+ `data-table-pagination.tsx`), `form.tsx`, `chart.tsx` (Recharts wrapper), `date-time-picker.tsx`, `multi-select.tsx`, `secret-input.tsx`, `role-select.tsx`, `assignment-combobox.tsx`, `cron-expression-picker.tsx`, and more — each backed by a Radix primitive where one exists. Per the `archestra-dev-frontend` skill, new primitives are added with `npx shadcn@latest add <component>`, and existing code is expected to reuse `components/ui/*` over raw HTML elements (`Button` over `<button>`, `Input` over `<input>`, etc.).
- `components/ai-elements/` — chat-specific building blocks (`message.tsx`, `tool.tsx`, `reasoning.tsx`, `response.tsx`, `code-block.tsx`) that the chat feature composes into the message stream. These have a project-specific pixel-alignment contract documented in `src/components/chat/CLAUDE.md`: every block in an assistant turn must sit on a 16px (`mb-4`) vertical rhythm, verified with Playwright rather than trusted from the wrapper class alone.
- `components/chat/` — the largest feature folder (~60 files): message rendering, tool-call cards, MCP app containers/panels, elicitation dialogs, policy-denial UI, file previews, conversation header/actions, model selector, editable message editors, etc.
- Other feature folders following the same pattern: `components/connection/`, `components/mcp-app/`, `components/projects/`, `components/roles/`, `components/scheduled-tasks/`, `components/teams/`, `components/tokens/`, `components/files/`.
- Top-level one-off components live directly in `components/` (e.g. `dynamic-head.tsx`, `org-theme-loader.tsx`, `version.tsx`, `error-fallback.tsx`, `external-docs-link.tsx`) rather than in a catch-all `common/` bucket.

This is closer to Brad Frost's "feature folders + a shared primitive layer" than strict atomic design (no formal atoms/molecules/organisms taxonomy) — `components/ui/` plays the role of atoms/molecules, everything else is grouped by product feature.

## Data fetching: generated API client + TanStack Query

### Where the generated client lives and how it's regenerated

The typed API client is **not** hand-written and does not live inside `frontend/` — it's generated into `platform/shared/hey-api/clients/api/` (and a second client, `platform/shared/hey-api/clients/archestra-catalog/`, for the separate MCP catalog service) by `@hey-api/openapi-ts`, configured in `platform/shared/hey-api/openapi-ts.ts`:

```ts
const archestraApiConfig = await defineConfig({
  input: process.env.CODEGEN === "true"
    ? "../../docs/openapi.json"        // committed spec, used by `pnpm codegen`
    : "http://localhost:9000/openapi.json", // live dev server, manual regen
  output: { path: "./hey-api/clients/api", clean: false, indexFile: true, format: "biome" },
  plugins: [{ name: "@hey-api/client-fetch", runtimeConfigPath: "./custom-client" }],
});
```

`platform/shared/hey-api/clients/api/custom-client.ts` supplies the `@hey-api/client-fetch` runtime config (`createClientConfig`) that lets the frontend set the client's `baseUrl` at runtime rather than baking it in at codegen time: relative URLs (`""` base) on the client so requests go through the Next.js rewrite proxy, and the absolute `ARCHESTRA_INTERNAL_API_BASE_URL` on the server for SSR/Route-Handler calls. It also installs a custom query serializer (comma-joined array params, e.g. `agentTypes=llm_proxy,agent` instead of repeated keys) to match what the Fastify backend expects, and sets `throwOnError: false` so TanStack Query — not a thrown exception — owns error-state handling.

Generated output includes `sdk.gen.ts` (one function per operation, e.g. `getConfig`, `getAllAgentTools`, `assignToolToAgent`), `types.gen.ts` (request/response types per operation), and `client.gen.ts`/`core/*.gen.ts` runtime plumbing. `platform/shared/index.ts` re-exports the whole thing as two namespaces: `export * as archestraApiSdk from "./hey-api/clients/api/sdk.gen"` and `export * as archestraApiTypes from "./hey-api/clients/api/types.gen"` — so frontend code imports `{ archestraApiSdk, archestraApiTypes } from "@archestra/shared"` rather than reaching into `hey-api/` paths directly.

Regeneration:

```bash
pnpm codegen   # root-level: CODEGEN=true turbo codegen — reads docs/openapi.json, writes the client, then generates theme CSS
```

or, scoped to just the API client with a live backend:

```bash
cd platform && CODEGEN=true pnpm --filter @archestra/shared codegen:api-client
```

(`codegen:api-client` runs `tsx hey-api/openapi-ts.ts && biome check --write ./hey-api/clients` — codegen output is Biome-formatted like hand-written code, not gitignored/vendored-looking.)

### The `.query.ts` convention

Per `archestra-dev-frontend` (the project skill for this area) and confirmed throughout the codebase (e.g. `src/lib/agent-tools.query.ts`, `src/lib/config/config.query.ts`), all server-state access goes through TanStack Query hooks in files named `*.query.ts`, and those files are the *only* place allowed to call the generated SDK — components never call `archestraApiSdk.*` or `fetch()` directly against the Archestra backend. (Raw `fetch()` is reserved for third-party APIs the SDK doesn't cover, e.g. `src/lib/github/*.query.ts` calling GitHub's API.)

Concrete example, `src/lib/config/config.query.ts`:

```ts
const { getConfig } = archestraApiSdk;

export function useConfig() {
  const isAuthenticated = useIsAuthenticated();
  return useQuery({
    queryKey: ["config"],
    queryFn: async () => {
      const { data, error } = await getConfig();
      throwOnApiError(error, { toastOnError: false });
      return data ?? null;
    },
    staleTime: 5 * 60 * 1000,
    enabled: isAuthenticated,
  });
}

export function useFeature<K extends keyof FeaturesResponse>(flag: K) {
  const { data } = useConfig();
  if (!data) return undefined;
  return data.features[flag];
}
```

Error handling is centralized in `src/lib/utils/api.ts`:

- `throwOnApiError(error, { toastOnError?, allowNotFound? })` — used inside `queryFn`s. It throws (putting the query into its error state) unless `allowNotFound` is set and the error is specifically `api_not_found_error` (a 404 on a detail endpoint meaning "doesn't exist," not "the backend is down"). By default it also toasts via `handleApiError`; screens that render their own retry UI pass `toastOnError: false` to avoid a duplicate/repeating toast. The comment on this function is explicit about why silent defaults are banned: *"A swallowed error makes an outage indistinguishable from a genuinely empty result, which is how 'Add an LLM Provider Key' showed up offline."*
- `handleApiError(error)` — used inside mutation `onError` callbacks: shows a `sonner` toast and reports to Sentry (`captureException`, dynamically imported).
- `toApiError(error)` / `getApiErrorMessage(error)` — unwrap the generated SDK's error shape into a plain `Error`/string.

Mutations follow the mirror-image convention: `mutationFn` calls `handleApiError(error)` + `throw toApiError(error)` on failure, success/error toasts live in the mutation's `onSuccess`/`onError`, and `onSuccess` is where the relevant `queryClient.invalidateQueries()` calls live (see `useAssignTool()` in `src/lib/agent-tools.query.ts`, which invalidates eight-plus related query keys after a tool assignment — `agents`, `tools`, `agent-tools`, `mcp-servers`, `mcp-catalog`, and the affected agent's chat tool cache — because a single tool assignment fans out into many independently-cached views). Components themselves never wrap SDK calls in `try`/`catch`; that responsibility stays in `.query.ts` files.

The QueryClient itself (`src/app/_parts/query-client-provider.tsx`) sets `staleTime: 60_000`, `throwOnError: false`, `retry: false` as global defaults — a single client instance is created lazily with `useState(() => new QueryClient(...))` per the standard Next.js App Router pattern (one client per browser tab, not per request).

### The `initialData` hydration pattern

Rather than the more common Next.js RSC pattern of `prefetchQuery` + `HydrationBoundary`, this codebase mostly uses a lighter-weight variant: a Server Component (`page.tsx`) calls the generated SDK directly with `getServerApiHeaders()` (`src/lib/utils/server.ts`, forwards the session cookie from `next/headers` `cookies()`), and passes the result as an `initialData` prop into a client component that re-issues the same query with `useQuery({ initialData })`. Example, `src/app/llm/logs/page.tsx`:

```tsx
export const dynamic = "force-dynamic";

export default async function LlmProxyLogsPageServer() {
  const headers = await getServerApiHeaders();
  const [interactionsResponse, agentsResponse] = await Promise.all([
    archestraApiSdk.getInteractions({ headers, query: { limit: DEFAULT_TABLE_LIMIT, offset: 0, ... } }),
    archestraApiSdk.getAllAgents({ headers, query: { excludeBuiltIn: true, agentTypes: [...] } }),
  ]);
  return <LlmProxyLogsPage initialData={{ interactions: ..., agents: ... }} />;
}
```

This avoids a client-side loading spinner on first paint for data-heavy pages (logs, tables) while keeping all subsequent fetches, pagination, and cache invalidation on the normal client-side TanStack Query path — no server-side query cache serialization/dehydration machinery is needed. The client-side `DataTable` (`src/components/ui/data-table.tsx`, built on `@tanstack/react-table`) is driven with `manualPagination`/`manualSorting` props wired to `useDataTableQueryParams()`, so paging/sorting state round-trips through URL query params and refetches from the backend rather than paginating an in-memory array — the mechanism referred to elsewhere as "server-side paginated tables."

## Chat and streaming UI

Chat state is centralized in a single React context, `src/lib/chat/global-chat.context.tsx` (`ChatProvider`, mounted once in the root layout), built on **`useChat` from `@ai-sdk/react`** (Vercel AI SDK) rather than a hand-rolled SSE reader:

```ts
const { messages, sendMessage, regenerate, resumeStream, status, setMessages, stop, error, ... } = useChat({
  messages: initialMessages,
  transport: new DefaultChatTransport({
    api: "/api/chat",
    credentials: "include",
    headers: { [EXTERNAL_AGENT_ID_HEADER]: getChatExternalAgentId(appName) },
    prepareReconnectToStreamRequest: ({ id, headers, credentials }) => ({
      api: `/api/chat/conversations/${id}/active-run`,
      headers, credentials,
    }),
  }),
  experimental_throttle: 100,
  id: conversationId,
  onFinish: async ({ message, isAbort, isError }) => { ... },
});
```

`api: "/api/chat"` is not a Next.js Route Handler — like the rest of `/api/*`, it's rewritten straight through to the Fastify backend (`next.config.ts`'s `/api/:path*` rewrite), which is why `experimental.proxyTimeout` is raised to 5 minutes: the SSE/streaming response has to survive the rewrite proxy for the life of the chat turn. `DefaultChatTransport` is the AI SDK's abstraction over the request/response streaming protocol; `experimental_throttle: 100` batches UI re-renders to at most every 100ms so token-by-token streaming doesn't thrash React. `prepareReconnectToStreamRequest` points reconnection at a distinct `/active-run` endpoint, supporting resumable streams — the context tracks `shouldResumeActiveRun(initialMessages)` on mount and calls `resumeStream()` so a page refresh or dropped connection mid-turn can reattach to a still-running backend generation instead of losing it.

Per the model documented in CLAUDE.md, tool execution is **not** performed server-side by the LLM proxy — the proxy returns `tool_use`/`tool_calls` to the client, which is expected to run the standard agentic loop: call the proxy, receive tool calls, execute them via the MCP Gateway (`POST /v1/mcp/${profileId}` with `Authorization: Bearer ${archestraToken}`), send results back, repeat until a final answer. `src/lib/chat/api-call.ts` and the various `use-chat-apps.ts`/`apps-context.tsx` files in `components/chat/` are where this loop's tool-call rendering and MCP app (interactive UI) integration live on the frontend side. `onFinish`'s handling of `isAbort` (stripping dangling tool-call parts left behind when a user stops mid-tool-call) and of `isError` (deliberately *not* clearing the in-flight recovery flag, because the SDK fires `onFinish` from a `finally` block right after `onError`) show the amount of care taken to keep client-rendered message state consistent with what the backend persists.

Conversation-list management (rename, delete, select) lives in the main sidebar via `src/app/_parts/chat-sidebar-section.tsx`, separate from the message-stream context.

## Theming and white-labeling

Themes are data-driven from a single external source, not hand-authored CSS-in-JS: `platform/shared/themes/tweakcn-themes.json`, sourced from the [tweakcn](https://github.com/jnsahaj/tweakcn) registry (per the comment in `frontend/src/themes.ts`). The pipeline:

1. `platform/shared/themes/theme-config.ts` — `SUPPORTED_THEMES`, an explicit allowlist of theme IDs from the tweakcn registry the platform actually ships (e.g. `modern-minimal`, `caffeine` (default), `claude`, `vercel`, `catppuccin`, `solarized-dark`, `gruvbox-dark`, `neo-brutalism`, ~20 total).
2. `platform/shared/themes/generate-theme-css.ts` (run via `pnpm codegen:theme-css`, part of `pnpm codegen`) reads `tweakcn-themes.json` and `theme-config.ts` and emits CSS custom-property classes (colors, radius, font tokens, letter-spacing/tracking) into `frontend/src/app/themes.css` — this file is committed generated output, keeping the JSON registry as the single source of truth rather than duplicating values by hand.
3. `frontend/src/themes.ts` re-exports theme metadata (`getThemeMetadata()`, `getThemeById()`) from `@archestra/shared`, appending `" (Default)"` to whichever theme is `DEFAULT_THEME_ID`.
4. Applying a theme is a plain DOM class swap, not a React re-render of styled values: `src/lib/theme.hook.ts`'s `applyThemeOnUI()` strips any `theme-*` class off `document.documentElement` and adds `theme-${themeId}`, matching the CSS classes `generate-theme-css.ts` emitted.

**White-labeling settings** (organization theme, logo/logo-dark, icon logo, favicon, app name, OG description, footer text, chat links, onboarding wizard, chat placeholders, chat error support message) live on the backend as "organization appearance settings" and are exposed through a **public, unauthenticated** endpoint — `useAppearanceSettings()` (`src/lib/organization.query.ts`) — deliberately, so the sign-in page and other pre-auth surfaces can render the correct branding before a session exists. `src/lib/theme.hook.ts`'s `useOrgTheme()` layers this with `localStorage` (`archestra-theme` key) to avoid a flash of the wrong theme on load: it seeds React state from `localStorage`, then reconciles with the backend value once it arrives, and (only for non-auth pages) exposes `setPreviewTheme`/`saveAppearance` for live-preview-then-save editing.

The editing UI is **`src/app/settings/organization/page.tsx`**, gated behind `organizationSettings: ["update"]` permission checks (`WithPermissions`, admin-only in practice), composing `ThemeSelector` (grid of theme swatches with live preview via `setPreviewTheme`), `LogosSection`, `FaviconUpload`, and a "Branding" card for app name/OG description/footer/chat links/placeholders/onboarding wizard — as noted above, this is the actual home of what CLAUDE.md calls "Appearance Settings," despite living at `/settings/organization` rather than `/settings/appearance`.

`OrgThemeLoader` (mounted once, root-layout-wide) and `DynamicHead` (`src/components/dynamic-head.tsx`, sets `document.title`, favicon `<link>`, and OG meta tags from the same `useAppearanceSettings()` data) are the two side-effect components that make white-labeling apply globally rather than per-page. Per the `archestra-dev-frontend` skill, UI copy is expected to call `useAppName()` (`src/lib/hooks/use-app-name.ts`) instead of hardcoding "Archestra," so white-labeled deployments render their configured name consistently.

## Feature flags via `/api/config`

`useConfig()` (`src/lib/config/config.query.ts`) fetches the authenticated `/api/config` endpoint (`getConfig` from the generated SDK) with a 5-minute `staleTime` and `enabled: isAuthenticated`. `useFeature<K>(flag)` reads `data.features[flag]` off that response, typed against `FeaturesResponse = archestraApiTypes.GetConfigResponses["200"]["features"]` — so the set of valid flag names is generated from the backend's OpenAPI schema, not a hand-maintained union. Documented examples: `orchestratorK8sRuntime` (gates K8s-backed local MCP server functionality, disabled unless the backend has `ARCHESTRA_ORCHESTRATOR_KUBECONFIG` or in-cluster config) and `devAutoLoginEnabled`.

A separate, **unauthenticated** `usePublicConfig()` hook hits `/api/public-config` (`getPublicConfig`) for flags needed before a session exists — `disableBasicAuth`, `disableInvitations`, `enterpriseCoreActive`, and `devAutoLoginEnabled`. `WithAuthCheck` reads `devAutoLoginEnabled` from this public-config query specifically so it can gate the dev-auto-login POST without waiting on an authenticated round trip that can't happen yet.

Per the CLAUDE.md convention, any new backend env var the frontend needs to reference must be threaded through `backend/src/routes/config.ts`'s response and consumed via `useFeature()` — there is no separate frontend-only flag system.

## Frontend observability

- **Error reporting**: `@sentry/nextjs`, wired through three files that mirror Next.js's execution environments — `sentry.client.config` (browser, initialized from `src/instrumentation-client.ts`), `sentry.server.config` (Node.js runtime, initialized from `register()` in `src/instrumentation.ts` when `NEXT_RUNTIME === "nodejs"`), and `sentry.edge.config` (edge runtime, e.g. middleware). All three are gated on `NEXT_PUBLIC_ARCHESTRA_SENTRY_FRONTEND_DSN` being set — Sentry is not initialized at all otherwise. `src/instrumentation-client.ts` also filters "The destination stream closed early." errors, a benign Next.js exception thrown when a client navigates away mid-render/prefetch, matched by message substring since Next throws a plain `Error` with no stable code. `getFrontendBrowserSentryOptions` is shared from `frontend/sentry.shared.ts`. `withSentryConfig(nextConfig, sentryWebpackOptions)` wraps the Next config in production builds only (`process.env.NODE_ENV === "development" ? nextConfig : withSentryConfig(...)`) — source maps are uploaded to a self-hosted-region Sentry org (`de.sentry.io`) and browser requests are tunneled through `/monitoring` to dodge ad blockers.
- **Product analytics**: `posthog-js` via `PostHogProviderWrapper` (`src/app/_parts/posthog-provider.tsx`), mounted inside the auth-check boundary in the root layout.
- **Distributed tracing / metrics**: no OpenTelemetry SDK is wired into `instrumentation.ts`/`instrumentation-client.ts` in the frontend — those files are Sentry- and MSW-focused only. OpenTelemetry span/metric emission for the platform (LLM/MCP spans, Tempo, Grafana, Prometheus) is a backend-owned concern; see [Observability](./10-observability.md) for where traces originate. This is worth stating explicitly since the frontend's `instrumentation.ts` filename invites the assumption that it configures OTel the way the backend's likely does — in this codebase it doesn't.
- **Structured request logging**: `src/proxy.ts` logs `API Request: <method> <url>` for `/api` and `/v1` traffic at the middleware layer (before the Next.js rewrite proxies it onward), with MCP gateway GET-polling noise explicitly suppressed.

## Design Decisions & Tradeoffs

**Generated API client (`@hey-api/openapi-ts`) over hand-written fetch wrappers.**
*Rationale*: the client and its types (`archestraApiSdk`, `archestraApiTypes`) are derived mechanically from the backend's OpenAPI spec (`docs/openapi.json` for `pnpm codegen`, or a live `localhost:9000/openapi.json` for ad hoc regeneration), so a backend route/schema change becomes a type error at every call site instead of a silent runtime mismatch. `archestra-dev-frontend`'s explicit instruction to "reuse API types from `@archestra/shared`... do not define duplicate frontend API types" backs this up as policy, not just convention.
*Cost*: an extra build step (`pnpm codegen`) has to run and be committed whenever the backend's API surface changes, and the generated output (`*.gen.ts`) is large, machine-formatted (Biome-run over generated code), and excluded from Biome's own lint globs (`!**/*.gen.ts`) — it's read-only in practice, and a developer who hand-edits it will lose the edit on the next regen.

**Next.js App Router + Server Components used narrowly (initial-data fetch), not for full SSR data flow.**
*Rationale*: `page.tsx` Server Components fetch first-paint data server-side (avoiding a loading spinner) and hand it to a `"use client"` sibling as `initialData`, but all subsequent interactivity — pagination, filters, mutations, cache invalidation — stays on the client with plain TanStack Query, skipping the more invasive `HydrationBoundary`/dehydrate-rehydrate machinery.
*Cost*: this is a lighter-weight, easier-to-reason-about hybrid than "pure" RSC data-fetching, but it means every such page effectively fetches its initial data twice in spirit (once server-side for first paint, and the client `useQuery` still re-validates against the same key) and the pattern isn't formalized as a shared helper — each `page.tsx`/`page.client.tsx` pair reimplements the same `initialData` plumbing by hand (see `src/app/llm/logs/page.tsx`).

**Biome instead of ESLint + Prettier, with a custom Grit plugin.**
*Rationale*: one Rust-based tool (`biome check`/`biome check --write`) replaces two Node-based tools, and it's used identically across `frontend/`, `shared/`, `e2e-tests/`, and `standalone-scripts/` from a single root `platform/biome.json` — one linter config to maintain monorepo-wide. The custom `biome-plugins/icon-button-aria-label.grit` plugin encodes a project-specific accessibility rule (icon-only buttons need `aria-label`) that a generic Biome/ESLint ruleset wouldn't catch, showing the team is willing to hand-write structural lint rules rather than rely purely on convention/review.
*Cost*: Biome's rule set and plugin ecosystem (Grit) is younger and smaller than ESLint's; anyone porting a shareable ESLint config or plugin from elsewhere doesn't have a drop-in path.

**Server-side (manual) pagination/sorting for data tables, not client-side `react-table` pagination.**
*Rationale*: `components/ui/data-table.tsx` accepts `manualPagination`/`manualSorting` flags and `onPaginationChange`/`onSortingChange` callbacks rather than letting `@tanstack/react-table`'s built-in row models paginate an in-memory array; paging/sorting state is synced to URL query params (`useDataTableQueryParams()`) and refetched from the backend. For tables like LLM proxy interaction logs or agent tool assignments, which can have arbitrarily many rows, this avoids ever shipping the full dataset to the browser.
*Cost*: every such page has to wire up query-param state, a `manualPagination`-aware query hook, and the loading/error states for each page transition — more moving parts than `getPaginationRowModel()` handles for free on a small in-memory dataset, so it's deliberately reserved for tables backed by potentially-large collections.

**Dev-auto-login is a real, cookie-backed session mint, not a mocked/bypassed auth state.**
*Rationale*: `WithAuthCheck` POSTs to `/api/auth/dev-auto-login` (proxied through to the backend's `dev-auto-login` Better Auth plugin) when `devAutoLoginEnabled` (from the *public*, unauthenticated `/api/public-config` response) is true and the user isn't logged in, then invalidates the session query so the app re-renders as genuinely authenticated. Because it produces a real session for a real seeded user, RBAC, permissions, and every other authenticated code path behave exactly as they would for a manual login — there's no separate "dev mode" branch elsewhere in the frontend to keep in sync.
*Cost*: the flag has to be threaded correctly through backend config *and* checked via public (pre-auth) config on the frontend, and a bug in the enable/disable gating for either side is a real authentication bypass risk — this is presumably why the backend hard-disables it outside development (`NODE_ENV=production`/`prod`) rather than trusting the frontend flag alone.

**White-labeling appearance settings are served over a public (unauthenticated) endpoint.**
*Rationale*: `useAppearanceSettings()` — logo, theme, favicon, app name — has to render correctly on the sign-in page and other pre-auth surfaces, so the data can't sit behind the normal authenticated `/api/config` endpoint the way most settings do (`useConfig()`/`useFeature()`).
*Cost*: organization branding metadata (app name, footer text, chat links, OG description, logos) is exposed to anyone who can reach the frontend origin without logging in — an accepted tradeoff for a feature whose entire purpose is visual branding, but a deliberate widening of what's publicly readable compared to the rest of the settings surface.

**Chat streaming built on the Vercel AI SDK (`@ai-sdk/react`'s `useChat`) rather than a hand-rolled SSE client.**
*Rationale*: `useChat` + `DefaultChatTransport` supplies throttled re-rendering (`experimental_throttle: 100`), a `messages` state machine with typed message parts (text/tool-call/reasoning), and — critically — a `prepareReconnectToStreamRequest`/`resumeStream()` primitive for reattaching to a still-running generation after a page reload or dropped connection, which the frontend uses against a dedicated backend `/api/chat/conversations/:id/active-run` endpoint.
*Cost*: the frontend's chat/tool-execution loop is coupled to the AI SDK's message-part model and transport abstractions; the extensive `onFinish`/`onError`/`isAbort` bookkeeping visible in `global-chat.context.tsx` (stripping dangling tool-call parts on abort, not clearing the recovery flag on error because `onFinish` fires from the SDK's own `finally` block) shows real edge-case cost in keeping client-rendered state faithful to what the backend persists — this is not a thin wrapper, it's an area the codebase's own `components/chat/CLAUDE.md` singles out as needing "extra attention."
