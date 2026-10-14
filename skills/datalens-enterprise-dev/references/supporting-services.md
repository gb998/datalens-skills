# Auth, Meta Manager and Snapter

Load only the dependency involved in the developer's scenario. Trace callers
and runtime wiring; do not test each as an interchangeable Node service.

## Auth

Private Auth has `projects/opensource` and `projects/enterprise`, no Yandex or
extensions workspace. Package name `datalens-auth` does not identify the checkout.
At private root use pinned Node/pnpm and `pnpm install --frozen-lockfile`.
Standalone dev requires protected `projects/enterprise/.env.development` with
local DB/endpoints before setup; setup creates `.env` when absent. Compose watch
prepares its environment separately. Commands include `en:dev`, `en:build`, `en:lint`, `en:typecheck`, `en:test`;
`os:*` targets the public installation. Enterprise bootstrap `--dev-auth`
preflights migrations/admin setup; standalone watch is different.

After build use `pnpm -F enterprise run test:unit` and `test:int:run` for the
actual HTTP/PostgreSQL/LDAP test fixtures. Test setup can rewrite workspace env
files. Enterprise DB commands are `pnpm -F enterprise run db:migrate` (and
supported rollback commands); root aliases target opensource. Use local/test DBs.

Production build from private root:

```bash
docker buildx build --load --platform linux/amd64 \
  -f projects/enterprise/Dockerfile \
  --build-arg app_version=local-dev -t enterprise-auth:local-dev .
```

The Dockerfile copies root `projects/...` paths: a workspace-level `docker:build`
with workspace context is not equivalent. Test local login/logout/refresh, cookie
behavior, password changes and existing sessions, role changes, and LDAP/OIDC
when affected. Propagate signing/public keys to UI/US/backend; never log tokens.

## Meta Manager

Meta Manager runs HTTP plus Node Temporal workers for workbook export/import
and expired-job cleanup in one process. Read [temporal.md](temporal.md) for
namespaces/queues, dependencies, local overrides and job diagnostics. Backend
chart-data exports use a separate Redis/ARQ pipeline.

Standalone Meta Manager dev requires protected
`projects/opensource/.env.development` following its tracked templates/setup;
setup creates `.env` if absent. Configure explicitly local endpoints and DSNs.

Private Meta Manager has opensource and Yandex workspaces; **no Enterprise
workspace**. Enterprise images consume opensource. Use root `os:dev`, `os:build`,
`os:lint`, `os:typecheck`. `pnpm test` covers unit/workflow tests, not integration;
`pnpm test:int` additionally exercises HTTP/PostgreSQL/Temporal suites.
A full build is needed for the Temporal workflow bundle; `build:test` alone
clears output and does not create the bundle required for integration.
Use `pnpm -F opensource run db:migrate` for the shared/Enterprise path, not
root Yandex migration aliases. Its integration DB helper uses a dedicated
`TEST_POSTGRES_DSN`/disposable container, not the application's production DSN.

Production build from Meta Manager root:

```bash
docker buildx build --load --platform linux/amd64 \
  -f deploy/enterprise/Dockerfile \
  --build-arg app_version=local-dev -t enterprise-meta-manager:local-dev .
```

Enterprise `--export` enables a pinned MM image; it does not select local source.
Use an explicit MM dev Compose service or the production-image override.
Validate export/import content and job completion with US/UI API/Temporal,
including the required database and workflow dependencies.

## Snapter

Subscriptions use Snapter to render delivery artifacts; use the
[subscription workflow](temporal.md#change-and-test-subscriptions) when testing
scheduled dashboard/report mail so the Notify/SMTP and Temporal path is included.

Snapter uses root npm and **`en:*`** Enterprise commands, unlike UI `ent:*`.
Use `npm ci --no-audit --no-fund`. `npm run en:dev` prepares the Enterprise
environment and browser; `en:watch` assumes those dependencies already exist.
Checks include `npm run lint:ci`, `npm run typecheck`, `npm run test:jest`,
`npm run en:typecheck`; plain `lint` auto-fixes and the inspected plain `npm test`
points to a missing legacy file. Inspect current scripts instead of assuming them.

Production build from Snapter root:

```bash
docker buildx build --load --platform linux/amd64 \
  -f deploy/enterprise/Dockerfile \
  --build-arg app_version=local-dev -t enterprise-snapter:local-dev .
```

Default `docker:build` chooses Yandex, not Enterprise. Enterprise manifest-pruning
scripts are for isolated builds, not destructive execution in the working checkout.
The runtime requires **`PLAYWRIGHT_WS_ENDPOINT` and a compatible browser server**;
installed Chromium alone does not prove screenshots work. Compose starts
`snapter-pw` from the same image; watch mode uses `--dev-snapter` and the nginx
screenshot route when requested. Browser version/network assumptions matter.
When overriding a production Snapter image, keep its browser service compatible.
Test an authorized local chart screenshot through the intended route, S3 access
and browser-to-UI connectivity; internal-auth e2e defaults are not clean Enterprise
smoke tests. Recheck selected Node/Playwright versions: dev and Docker/CI versions
can differ in this checkout.
