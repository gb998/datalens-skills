# Temporal, subscriptions and workbook jobs

Use this reference for scheduled dashboard/report delivery, workbook transfer,
usage tracking or a stuck asynchronous job. Identify the producer, namespace,
task queue, executing worker and result store before changing an image.

## Ownership and execution paths

| Feature | Producer and worker | Namespace / task queues | Result and dependencies |
|---|---|---|---|
| Workbook export/import | Meta Manager HTTP API and Node workers in the **same process/container** | `meta-manager`: `export-workbook-queue`, `import-workbook-queue`, `clear-expired-queue` | MM PostgreSQL job/result records; activities call UI API internal workbook endpoints, then US/backend as needed |
| Scheduled dashboard/report delivery | Python `subscription-api` submits workflows; `subscription-worker` executes lifecycle/render/delivery work | `subscriptions`: queue `subscriptions` on both API and worker | US-backed subscription storage, Snapter artifacts, Notify delivery, local Mailpit |
| Usage tracking metadata import | Python `ut-worker`, built as `eas_temporal_worker` | `usage-tracking`: configured `worker_task_queue`, baked default `default-task-queue` | `USImportWorkflow` copies US metadata to ClickHouse; schedule `us-import-schedule`, workflow ID default `regular-us-import` |
| Background chart CSV/XLSX exports | Python `export-api`, `export-worker`, `export-data-api` | **Redis/ARQ**, not these Temporal queues | S3 artifacts and export job status; see [backend.md](backend.md) |

The Temporal server is the custom Go image in `datalens-enterprise/temporal`,
built separately from backend applications. It persists workflow history and
visibility in `pg-temporal-db` and `pg-temporal-visibility-db`; these are separate
from MM's `pg-meta-manager-db` and US storage. A server image does not contain
the application workers.

Source entry points:

- Backend: `.github/workflows/eas_release.yaml`, `app_eas/app_configs/`,
  `lib/bi_subscription_api_lib`, `lib/bi_subscription_worker_lib`,
  `lib/bi_subscription_client`, `mainrepo/lib/dl_temporal`,
  `app_eas/eas_temporal_worker`.
- MM: `projects/opensource/src/index.ts`, `components/temporal/workers.ts`,
  `components/temporal/workflows/`, `components/temporal/client/` and
  `components/gateway/schema/ui-api/actions/` beneath `projects/opensource/src`.
- Enterprise: `dev.sh`, `docker-compose.base.yml`, `docker-compose.temporal.yml`,
  `temporal/{dockerfile,init-temporal.sh,init-namespaces.sh}` and `temporal/tests/`.

## Local bootstrap and configuration

After selecting execution mode and the local environment, choose the needed path:

```bash
# From Enterprise root: workbook transfer with a local Temporal UI
./dev.sh --auth --export --temporal-ui

# Or: scheduled emails with screenshots, local SMTP and a local Temporal UI
./dev.sh --auth --subscriptions --mailpit --nginx --temporal-ui
```

Combine `--export` and `--subscriptions` when exercising both. Re-running
bootstrap without a previously selected feature can stop its services.

- `--export` enables workbook UI functionality and starts pinned `ui-api`,
  `meta-manager` and Temporal. `ui-api` uses the UI image with `APP_MODE=api`.
- `--subscriptions` starts API/worker, Notify and Notify migrations, and also
  selects Snapter/browser and secure embeds. Snapter pulls in S3. Notify must
  reach the selected SMTP service; `--mailpit` configures local SMTP port 1025.
- `--temporal-ui` adds the UI overlay but **does not select the Temporal server
  by itself**. Pair it with a feature above or start the required server explicitly.
- Both `--subscriptions` and `--temporal-ui` set `TEMPORAL_AUTH_ENABLED=false`
  for this local stack. Subscription clients use empty metadata providers.
  Auth-enabled workbook-only runs must retain matching Temporal keys; MM signs
  an RS256 token scoped to `meta-manager`. Changing this server flag affects
  every consumer sharing the server, so keep server/client modes consistent.
- Server initialization creates `meta-manager`, `usage-tracking`, `subscriptions`
  and subscription Keyword search attributes `ResultType`/`ResultCode`. A generic
  Temporal dev server does not reproduce this packaging/auth/startup contract.

Container clients use `temporal:7233`. Base Compose publishes **no gRPC host
port**; a host-run MM/worker needs an explicit local mapping such as
`127.0.0.1:7233:7233` and a host endpoint. Temporal UI is at
`http://localhost:8233`; MM at 3050, UI API at 3040, subscription API at 8060,
and Mailpit at 8025. Use container names/endpoints for container traffic.

## Change and test subscriptions

Build the selected subscription target with the Bake recipe in
[backend.md](backend.md); build both API and worker if their contract changes.
Use owner-library unit tests and local Temporal-backed DB suites before a full
mail scenario. `app_eas/eas_subscription_worker` is an installation wrapper;
the workflows and activities live under `lib/bi_subscription_worker_lib`.

Current worker flow is subscription lookup → recipient/permission checks →
Snapter artifact rendering → Notify. Its rendering registry supports dashboard
PNG and report PDF. EAS supplies its own recipient/auth behavior, including a
no-op license checker; do not infer Enterprise licensing from another wrapper.
The inspected source has no direct `DATA_API_CLIENT` consumer, even though base
Compose retains that variable; secure embeds can be involved via UI/Snapter.
Trace the actual selected image/source before asserting a dependency.

Compose deliberately pins `subscription-worker` separately (`# update: ignore`).
When replacing it with current source, check `subscription_worker.yaml`, settings
and the library's factory. Preserve the effective runtime configuration:

- Subscription client points to US, not the baked `subscription-api` default;
  local auth provider is US master token.
- Notify endpoint is `http://notify:3455`; Snapter is `http://snapter:8080`.
- API and worker agree on namespace/task queue `subscriptions`.
- Keep the valid Fernet key under `CRYPTO__MAP_ID_KEY__FERNET_KEY_1` with matching
  actual key ID, and the same unsubscribe secret on API/worker. Baked placeholders
  cannot serve as local credentials.

Example production-image override after the mail bootstrap, with both images
built locally. Keep secrets inherited from the protected base configuration:

```yaml
services:
  subscription-api:
    image: enterprise-subscription-api:local-dev
    environment:
      AUTH__TYPE: NATIVE
  subscription-worker:
    image: enterprise-subscription-worker:local-dev
```

Apply with the [common Compose loop](local-stack.md#test-a-changed-production-image)
to `subscription-api subscription-worker`, using `--no-deps --pull never` after
dependencies are ready. Check both container image IDs. If also recreating UI,
retain `DASH_MAILING_ENABLED=true` (bootstrap input `SUBSCRIPTION_ENABLED`),
`ENABLE_SECURE_EMBEDDING=true` (bootstrap input `SEC_EMBEDS_ENABLED`), Auth mode,
Snapter endpoint and the agreed UI URL.
Preserve Notify's local SMTP configuration if recreating Notify.

Acceptance uses local test users and Mailpit: save a subscription, send a test,
observe the Temporal run and the actual email/artifact, then verify a scheduled
run. Check rendered content and permissions, editing the schedule, unsubscribe
and deletion. A workflow may complete with an application error: inspect result
status/search attributes and actual delivery, not only Temporal's Completed state.
Use a controlled local dependency failure for retry changes; distinguish retry
attempts from duplicate deliveries. Do not send test email through shared SMTP.

## Change and test Meta Manager

Read [supporting-services.md](supporting-services.md#meta-manager) for its
opensource workspace, environment, production build and migration commands.
MM starts its HTTP API, all three workers and the hourly `clear-expired` schedule
from the executable entrypoint. Importing the app in a test does not start those
workers. Rebuilding only backend `export-worker` cannot change workbook transfer.

The path is UI → MM job record → Temporal → Node activities → UI API
`/api/internal/v1/workbooks/export` or `/import` → US/backend → MM result/status.
Export/import workflow IDs match the encoded job IDs returned by MM; the
workflow progress query is `getProgress`. Inspect MM routes for status/result
and cancellation, and correlate the same job with UI API and US logs.

`APP_DEV_MODE=1` builds workers from workflow source; production resolves
`workflow-bundle.js`. Use a full `pnpm run os:build`; `build:test` alone does not
produce that bundle. From MM root, useful focused checks include:

```bash
pnpm run os:build
pnpm -F opensource run test:unit
pnpm -F opensource run test:int:run --testPathPattern=workbook-roundtrip
```

MM integration tests use a Temporal time-skipping server, test PostgreSQL and
mocked upstream APIs. They exercise workflow logic but do not prove compatibility
with the pinned Enterprise server/auth or real UI API. For that, build the MM
production image and override only `meta-manager` in the bootstrapped stack:

```yaml
services:
  meta-manager:
    image: enterprise-meta-manager:local-dev
    environment:
      AUTH_ENABLED: 'true'
      TEMPORAL_AUTH_ENABLED: 'false'
```

This example matches `--auth --export --temporal-ui`; use the actual chosen mode
if testing authenticated Temporal. Retain UI `EXPORT_WORKBOOK_ENABLED=true`
when recreating UI. Override `ui-api` too when changing its workbook endpoints;
it uses the same production UI image, with its API-mode configuration preserved.

Verify worker pollers for the three queues, export a small known workbook,
download/inspect it, import into a disposable workbook and check objects/data
references. Exercise cancellation or interrupted activities when affected.
Validate MM job status **and** Temporal state **and** result content.

## Usage-tracking worker

Build `eas_temporal_worker` and override Compose service `ut-worker`. In the
inspected bootstrap, `--usage-tracking`/`--ut` starts logging/ClickHouse but not
this worker or Temporal. Select these explicitly in the same local project,
with a ready server and matching namespace/auth. Base Compose overrides baked
defaults to `temporal:7233`, `usage-tracking` and `clickhouse:9000`; preserve them.

Its startup deletes/recreates `us-import-schedule` before polling its queue, so
do not attach a development worker to a shared namespace. The signing helper
treats `None` as no key; an empty string is not equivalent. For an auth-disabled
local server, omit/reset the inherited key at the settings layer and verify the
loaded mode without printing it; otherwise retain a matching namespace-scoped key.
Check the app's US-import unit tests, then a local scheduled run and expected
ClickHouse metadata rows. Logging ingestion alone does not verify this worker.

## Diagnose a queued or failed job

Start with read-only checks on the selected local stack. Commands below assume
the inspected default container names and **auth-disabled local Temporal**:

```bash
docker exec datalens-enterprise-temporal \
  curl -fsS --max-time 5 http://localhost:8080/healthcheck-init
docker exec datalens-enterprise-temporal temporal --address 127.0.0.1:7233 \
  operator namespace describe --namespace meta-manager
docker exec datalens-enterprise-temporal temporal --address 127.0.0.1:7233 \
  operator search-attribute list --namespace subscriptions
```

For auth-enabled runs, use the existing protected client authentication mechanism;
do not disable auth or print a JWT just to inspect a job. The image contains the
`temporal` CLI; check its `--help` for that image's supported syntax. In Temporal
UI choose the owning namespace, locate the known job/workflow ID and inspect its
history, pending activity, retry error and queue pollers. For example, describe
an MM workflow while selecting only status/queue from the JSON:

```bash
docker exec datalens-enterprise-temporal temporal --address 127.0.0.1:7233 \
  workflow describe --namespace meta-manager --workflow-id "$EXPORT_ID" -o json \
  | jq '{status: .workflowExecutionInfo.status, taskQueue: .executionConfig.taskQueue.name}'
```

Set `EXPORT_ID` to the actual local test job ID. A queue with no poller points
to worker startup/bundle/namespace/queue mismatch. Activity retries point to the
named downstream dependency: inspect its bounded logs with the same job ID.
Check US/UI API/Notify/Snapter endpoints, database connectivity and auth modes.
Keep workflow inputs, user content and credentials out of shared diagnostics.
`/healthcheck` is server liveness; `/healthcheck-init` also checks databases and
namespaces. Neither proves that an application worker polls or finishes its job.

## When changing the Temporal server image

Keep the pinned image for application-only work. Server code/build lives in
Enterprise, with version pins in `versions-config.json` and `temporal/go.mod`.
`bash scripts/build-temporal.sh --dev` builds/loads a local image without pushing;
inspect its resulting tag and override only `temporal` after preserving DB/auth
configuration. Its non-dev path publishes an image, so do not use it for this loop.

Existing `temporal/tests/test-integration.sh`, `test-applications.sh`,
`test-postgres-tls.sh` and `test-postgres-ha.sh` cover different server contracts.
Read the selected test before running: they create isolated resources, and the
application suite exercises real MM/US/UI API export/import, cancellation and
PostgreSQL failover. They select tracked image versions; they do **not** test an
arbitrary local tag automatically. Adapt image inputs in an isolated temporary
copy when validating a changed image, keeping unrelated images pinned. Report
which exact image and scenarios actually ran.
