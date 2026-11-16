# 0001. Stack and architecture: rationale

Decision record for [index.md](index.md). `/develop` does not need this file.

## Context

You are building a feedback widget that your team drops into its own React apps during UAT. The apps run on Azure today, may run on AWS or GCP later, and already sign users in with Entra ID. Each app team wants feedback (text, screenshots, click steps, error logs) kept in its own cloud, not in a central service someone else runs. The developer is solo with no fixed deadline.

The forces that shaped the stack:
- **Embedding**: the widget runs inside someone else's React app, on React 18 or 19, so it must be a small library with React as a peer, not a bundled copy.
- **Credentials**: anything shipped to the browser is public. Storage access must stay on a server.
- **Bring your own cloud**: each team must deploy and run its part with as little infrastructure as possible, on any of three clouds.
- **Volume**: UAT produces hundreds to low thousands of reports per app, not millions.
- **Shared shape**: widget, server and dashboard all touch the same report structure; drift between them is the most likely bug source.

## Options considered

### Option 1: TypeScript monorepo, portable Hono collector in one container, storage only persistence (chosen)

Widget, collector, dashboard and shared types in one repo. One container per team serves the API and the dashboard. Reports live as JSON plus files in object storage.

**Pros**: one deploy shape on every cloud; no database to run; shared types end to end; widget calls are attributed through the existing Entra login.
**Cons**: no ad hoc queries over reports; listing depends on storage layout; each team does an Entra app registration and a role assignment.

### Option 2: Serverless functions per cloud plus a SQL database

Azure Functions (then Lambda, Cloud Functions) for the API, a managed Postgres for metadata, the dashboard on static hosting.

**Pros**: pay per use when idle; rich querying and filtering from day one.
**Cons**: three deploy shapes to maintain; every team now runs a database too; cold starts on first report after idle; more moving parts per team.

### Option 3: Browser writes straight to storage, no server

The widget uploads to a storage container using a key or an open container policy; the dashboard reads the same container.

**Pros**: nothing to deploy beyond storage.
**Cons**: a write credential or an open container is exposed to anyone who opens the app; no validation or rate limiting; no safe place to check who sent a report.

### Option 4: Next.js full stack app for collector and dashboard

One Next.js app with API routes for the collector and pages for the dashboard.

**Pros**: one framework for server and UI; familiar to React developers.
**Cons**: heavier runtime and build for a small internal API; server rendering adds nothing behind a login; ties the collector's portability to a framework's hosting model.

## Rationale

Credentials rule out Option 3 outright. Between the remaining options, the deciding force is bring your own cloud: every team has to deploy and run its piece, so the number of resources and deploy shapes per team matters more than anything else. Option 1 needs one container and one storage account on any cloud. Option 2 needs a function app, a database and static hosting, multiplied by three clouds. At UAT volumes a database buys querying power you won't use yet. The data model spec can add an index file in storage if listing gets slow, and a database stays a later upgrade, not a rewrite, because storage sits behind an adapter.

Hono over Next.js (Option 4), Fastify or Express: the collector is a small JSON and file API. Hono is typed, small, and built on web standard `Request` and `Response`, so the same code can move to a Functions or Lambda adapter if a team ever insists on serverless. Fastify is the strongest runner up (mature, and Microsoft's own upload sample uses it), but it is tied to Node. The dashboard is a page behind a login, so a static Vite build served by the collector is the lightest option and avoids CORS for the dashboard entirely.

For widget auth, you already use Entra ID. Validating the host app's Entra token at the collector reuses that login, so no shared secret goes into a browser bundle, and every report arrives with a verified identity. The runner up was a public project key plus allowed origins: simpler, but it can't prove who sent a report, and a key copied from one app works from any script that fakes the origin. Implementing the check as generic OIDC validation (with `jose`) rather than an Entra specific SDK keeps a non Entra issuer possible later.

On tooling, you picked npm workspaces and Node 24 over pnpm and Node 26. Both work: npm workspaces handle four packages without trouble, and Node 24 meets every dependency's floor. The cost is looser dependency isolation, plus a runtime bump to plan before Node 24's April 2028 end of life. tsdown replaced tsup because tsup now points its users to tsdown. Azurite keeps local development offline on the same SDK calls.

## References

**Project sources**:
- `docs/scope/scope.md`: scope rows 1, 3, 4, 5, 6 and the deferred AWS and GCP adapters
- Installed skills: `azure-blob-storage`, `azure-container-apps`, `entra-app-registration`, `vite`, `tsdown`

**Practices & standards**:
- Monolith first; one deployable unit until a bottleneck forces a split
- Never ship credentials to the browser; servers hold storage access
- OIDC bearer token validation (issuer, audience, signature, expiry)
- Peer dependencies for host provided frameworks in libraries

**Links** (web verified during the landscape check):
- Microsoft Learn, browser file upload to Azure Blob with user delegation SAS: https://learn.microsoft.com/en-us/azure/developer/javascript/tutorial/browser-file-upload-azure-storage-blob
- `@azure/storage-blob` registry entry (12.34.0, Node 22+): https://registry.npmjs.org/@azure/storage-blob/latest
- Hono on Azure Functions: https://hono.dev/docs/getting-started/azure-functions
- tsdown, migrating from tsup: https://tsdown.dev/guide/migrate-from-tsup
- Vite blog (Vite 8): https://vite.dev/blog
- React versions: https://react.dev/versions
- Node.js release status: https://nodejs.org/en/about/previous-releases
- Azure MCP Server: https://github.com/microsoft/mcp
