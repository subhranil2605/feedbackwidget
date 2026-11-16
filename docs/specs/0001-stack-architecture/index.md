# 0001. Stack and architecture for the UAT feedback widget

**Date**: 2026-11-17
**Status**: Proposed

## Summary

This spec picks the tools and the overall shape of the project. There are three parts in one TypeScript repo: a React widget that host apps install, a small collector server that each team runs in their own cloud, and a review dashboard that the collector serves. The widget never holds cloud keys. It sends the tester's existing Entra ID login token to the collector, and only the collector talks to storage. Everything about a report lives in object storage (Azure Blob first), so a team deploys one container plus one storage account and nothing else.

## Rationale

Reasoning, options and references: see [rationale.md](rationale.md).

## Decision

**Chosen option**: Option 1: TypeScript monorepo with a portable Hono collector in one container, storage only persistence.

One npm workspaces monorepo holds the widget, the collector, the dashboard and a shared types package. The collector runs as one container per team, serves many apps, authenticates widget calls with OIDC bearer tokens (Entra ID first), and is the only component with storage access.

## Proposed stack

| Layer | Choice | Reason |
|---|---|---|
| Language | TypeScript, `strict` on, ESM only | One language across widget, server and dashboard; the shared report types catch drift at compile time. |
| Runtime | Node.js 24 LTS (`engines: ">=24"`), pinned in `.nvmrc` and the container base image | Engineer's pick; supported until April 2028, and it meets the floors of `@azure/storage-blob` (Node 22+) and tsdown (Node 22.18+). |
| Repo layout | npm workspaces: `packages/shared`, `packages/widget`, `apps/collector`, `apps/dashboard` | Engineer's pick; four workspaces need no task orchestrator. |
| Shared types | `packages/shared` (`@uat-feedback/shared`, private, never published): Zod schemas plus inferred types for the report and API payloads | A single definition of the report shape, validated at the collector and typed in the widget and dashboard. |
| Widget package | `packages/widget`, published as `@uat-feedback/react`; peer dependency `react` and `react-dom` `^18.2.0 \|\| ^19.0.0` | Host apps on React 18 or 19 can use it without upgrading. |
| Widget bundler | tsdown: ESM output plus `.d.ts`; React kept external; `@uat-feedback/shared` bundled in | The maintained successor to tsup. Bundling shared code means host apps install a single package. |
| Widget distribution | Azure Artifacts private npm feed | Access follows your existing Azure DevOps and Entra permissions. |
| Collector framework | Hono on `@hono/node-server`; request validation with Zod via `@hono/zod-validator` | Small and typed, built on web standards; the same app could later run on Functions or Lambda adapters with no rewrite. |
| Collector shape | One collector per team, serving many apps. Every report carries an `appId`; an apps config maps each `appId` to its allowed origins. | One deploy per team; adding an app only takes config. |
| Widget to collector auth | OIDC bearer token validation (JWT signature checked against the issuer's published keys, plus issuer, audience and expiry checks) using `jose`. Entra ID is the first configured issuer. The host app supplies the token through a `getAccessToken()` prop on the widget. CORS allows only configured origins. | Reuses your existing Entra login, so no shared secret sits in the browser, and the token also identifies the reporter. `jose` is the standard, dependency free JWT library. |
| Dashboard | React 19 + Vite 8 single page app with React Router (library mode). Built to static files and served by the collector on the same origin. | Same React as the widget. A page behind a login needs no server rendering, and serving from the same origin means no CORS for the dashboard. |
| Dashboard sign in | Decided in the dashboard sign in spec (scope row 6) | Its own decision; Entra ID is the likely fit. |
| Persistence | Object storage only. Report metadata is stored as JSON next to its files, in one container per collector, behind a storage adapter interface. Layout and indexing are decided in the data model spec (scope row 3). | A team provisions just one storage resource; UAT volume (hundreds to low thousands of reports) needs no database. |
| File storage | Azure Blob Storage via `@azure/storage-blob`. The collector authenticates with managed identity (`DefaultAzureCredential` from `@azure/identity`). AWS and GCP adapters are deferred. | No storage keys in config for production; the identity is granted the Storage Blob Data Contributor role. |
| Upload path | The browser never receives storage credentials. Whether file bytes pass through the collector or go direct via a short lived write link is decided in the storage adapter spec (scope row 5). | That is its own decision, with tradeoffs in security and load. |
| Hosting | One container image (multi stage `Dockerfile`, Node 24 slim base) running the collector plus the dashboard's static files. First target: Azure Container Apps with system assigned managed identity. | The same image runs on App Runner or Cloud Run later. A managed platform means no cluster to operate. |
| Local development | Azurite storage emulator (in Docker), using a connection string for local only; collector and dashboard dev servers run with Vite proxying `/api` to the collector | Works offline with no Azure account, using the same SDK calls. |
| Observability | Structured JSON log lines to stdout (a tiny logger in the collector, no library), one line per request with `requestId`, `appId`, route, status and duration; errors are logged with their stack | Every container platform collects stdout (Azure Monitor, CloudWatch, Cloud Logging), with no vendor lock. |
| Lint, format, tests | Decided by `/audit` (coding standards & tooling, scope row 2) and `/test` preferences | Captured from the real scaffold, not guessed here. |

**Configuration required** (collector):
- `PORT`: listen port (default `8080`)
- `STORAGE_PROVIDER`: `azure` (only value for now)
- `AZURE_STORAGE_ACCOUNT_URL`: blob endpoint, for example `https://<account>.blob.core.windows.net` (production, used with managed identity)
- `AZURE_STORAGE_CONNECTION_STRING`: local and Azurite only; when set it takes precedence over the account URL
- `AZURE_STORAGE_CONTAINER`: container that holds reports (default `feedback`)
- `OIDC_ISSUER`: token issuer, for Entra `https://login.microsoftonline.com/<tenant-id>/v2.0`
- `OIDC_AUDIENCE`: the collector API's app registration ID or URI
- `APPS_CONFIG`: JSON mapping each `appId` to its allowed origins, for example `{"billing-portal":["https://uat.billing.example.com"]}`
- `LOG_LEVEL`: `info` by default

**Key invariants**:
- No cloud credential, storage key or write link ever ships in the widget bundle.
- Every collector write requires a valid bearer token from the configured issuer and audience, plus an `Origin` header listed for the report's `appId`.
- Only the collector reads or writes storage; the dashboard goes through the collector API.
- `packages/shared` is the single definition of the report shape; no workspace redefines it.

**Implementation skills**: `azure-blob-storage` (`microsoftdocs/agent-skills`, `.claude/skills/azure-blob-storage/`) · `azure-container-apps` (`microsoftdocs/agent-skills`, `.claude/skills/azure-container-apps/`) · `entra-app-registration` (`microsoft/azure-skills`, `.claude/skills/entra-app-registration/`) · `vite` (`antfu/skills`, `.claude/skills/vite/`) · `tsdown` (`rolldown/tsdown`, `.claude/skills/tsdown/`)

## Consequences

**Positive**:
- A team deploys one container plus one storage account. No database, no queue, no extra services.
- The widget reuses the host app's Entra login, so reports arrive already attributed to a person.
- The storage adapter and the OIDC issuer are configuration points, which keeps the AWS and GCP paths open without a rewrite.

**Negative / tradeoffs**:
- With storage only persistence, the dashboard cannot run ad hoc queries. Listing and filtering depend on the layout the data model spec chooses, and very large volumes would need an index or a database later.
- Requiring an Entra token means a host app without Entra cannot use the widget until another issuer is configured.
- Each team must create an Entra app registration for the collector API and grant the managed identity a storage role, which is extra setup before first use.
- Node 24 moves to maintenance in this window (end of life April 2028); plan the runtime bump to the next LTS before then.
- The collector serves the dashboard, so a dashboard change ships as a new container image.

**Neutral**:
- Everything is ESM only; host apps on old CommonJS only bundlers would need a modern bundler.
- Lint, format and test tooling are owned by `/audit` and `/test`, not this spec.

## Follow-up

- [ ] Data model spec (scope row 3) decides the report JSON shape and the blob layout, including how the dashboard lists and filters without a database.
- [ ] Storage adapter spec (scope row 5) decides the upload path: proxied through the collector, or direct via a short lived write link.
- [ ] Dashboard sign in spec (scope row 6) decides how the dashboard authenticates (Entra ID is the likely fit).
- [ ] Design system spec (scope row 4) decides widget style isolation from the host app.
- [ ] Record the installed skills in root `AGENTS.md` `## Agent skills` (vite, tsdown project wide; azure-blob-storage, azure-container-apps, entra-app-registration area scoped to the collector), plus `MCP servers: Azure MCP Server` once connected. `/audit` owns this.
- [ ] No valid Hono Agent Skill was found (`yusukebe/hono-skill` has no `SKILL.md`); check again before the collector build.
- [ ] Document the per team setup (Entra app registration for the collector API, the managed identity role assignment, the Azure Artifacts feed access) when the first deploy happens.
