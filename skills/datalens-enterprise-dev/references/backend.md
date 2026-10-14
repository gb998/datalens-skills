# Backend stream

## Layers and ownership

`datalens-backend-private/mainrepo/` is an editable public-source snapshot, not
a submodule. Shared `mainrepo/lib/dl_*` code must remain independently buildable
without private dependencies. Private packages use `lib/bi_*`; Enterprise
wrappers live in `app_eas/eas_*`; `app_ya_team` and `app_yc` are other installations.
Follow current repository ownership/mirroring rules rather than editing a public
mirror as a substitute for the private checkout.

## Select the released application

Start with `.github/workflows/eas_release.yaml`: its AMD and ARM build matrices
and manifest list define the released images. Cross-check each selected target
in `docker_build/bake_apps_eas.hcl`, its Dockerfile/entrypoint, the dependency
group in `ops/eas/pyproject.toml`, and `app_eas/app_configs/`. The inspected matrix
contains the following **13 images**, including the acceptance-test runner.
Recheck the matrix when the checkout changes; backend work is not limited to
Control and Data API.

| Bake target | Release image | Local Compose service | Responsibility and source owner |
|---|---|---|---|
| `eas_control_api` | `enterprise-control-api` | `control-api` | Connection/dataset lifecycle; `app_eas/eas_control_api`, shared `mainrepo/lib/dl_api_lib` and connector libraries |
| `eas_data_api` | `enterprise-data-api` | `data-api` | Query execution and result processing; `app_eas/eas_data_api`, shared API/query/connector libraries |
| `eas_data_api_sec_embeds` | `enterprise-data-api-sec-embeds` | `data-api-sec-embeds` | Query execution under secure-embedding identity/token rules; `app_eas/eas_data_api_sec_embeds`, `lib/bi_api_lib_sec_embeds_eas` |
| `eas_file_uploader_api` | `enterprise-file-uploader-api` | `file-uploader-api` | Upload/preview/source-management HTTP API; `app_eas/eas_file_uploader_api`, `lib/dl_file_uploader_api_lib` and shared uploader libraries |
| `eas_file_uploader_worker` | `enterprise-file-uploader-worker` | `file-uploader-worker` | Redis/ARQ ingestion tasks and file data loading; `app_eas/eas_file_uploader_worker`, `lib/dl_file_uploader_worker_lib` |
| `eas_file_secure_reader` | `enterprise-file-secure-reader` | `file-secure-reader` | Separate file parsing service; `app_eas/eas_file_secure_reader`, `lib/dl_file_secure_reader_lib` |
| `eas_temporal_worker` | `enterprise-temporal-worker` | `ut-worker` | Scheduled US metadata import into usage-tracking ClickHouse; `app_eas/eas_temporal_worker` |
| `eas_export_data_api` | `enterprise-export-data-api` | `export-data-api` | Data extraction/result endpoints for background chart export; `app_eas/eas_export_data_api`, `lib/bi_export_data_api_lib` |
| `eas_export_worker` | `enterprise-export-worker` | `export-worker` | Redis/ARQ background chart-data export and S3 artifacts; `app_eas/eas_export_worker`, `lib/bi_export_worker_lib` |
| `eas_export_api` | `enterprise-export-api` | `export-api` | Submit/status/download API for chart-data export; `app_eas/eas_export_api`, `lib/bi_export_api_lib` |
| `eas_subscription_api` | `enterprise-subscription-api` | `subscription-api` | Subscription CRUD/test/history HTTP API and Temporal workflow requests; `app_eas/eas_subscription_api`, `lib/bi_subscription_api_lib` |
| `eas_subscription_worker` | `enterprise-subscription-worker` | `subscription-worker` | Temporal subscription schedules, content rendering and notification delivery; image wrapper `app_eas/eas_subscription_worker`, implementation `lib/bi_subscription_worker_lib` |
| `acceptance_tests` | `enterprise-acceptance-tests` | Separate test Compose | `ops/bi_integration_tests_eas/Dockerfile.tier1` and its `bi_integration_tests_tests`; test runner, not a long-running application |

The `eas_*` targets use matching `app_eas/<target>` build contexts; the acceptance
target uses the separate context above. Shared exports also use `lib/bi_export_lib`;
subscriptions use `lib/bi_subscription_client` and `mainrepo/lib/dl_temporal`.
Private file packages happen to have `dl_*` names: find their actual location
rather than inferring public/private ownership from the prefix alone.

Temporal **server** is `datalens-enterprise/temporal`; Meta Manager is a separate
Node repository. Neither is built by this backend release matrix. The Python
`eas_temporal_worker`, subscription worker and Meta Manager workers have different
jobs; use [temporal.md](temporal.md) for their namespaces, queues and debugging.

Control API manages connections/datasets/configuration (Flask/uWSGI in EAS);
Data API executes queries (aiohttp/Gunicorn in EAS). Shared API/connector code,
Enterprise metapackages and wrapper middleware all matter: a local library change
must also be included in the app that will run it. Read AGENTS, relevant `.ai/`
context, package manifests and connector-development guidance before edits.

## Python environment and package-level checks

The inspected checkout selects Python 3.12 and Poetry 1.7.1; recheck its version
files. From the backend-private root:

```bash
task dev:init
```

This prepares the centralized environment under `ops/ci/.venv` with the private
CI metapackage. It is not equivalent to running Poetry install in an arbitrary
root/application directory. Then run from the affected package, for example
`mainrepo/lib/dl_connector_postgresql`:

```bash
task dev:lint
task dev:test -- dl_connector_postgresql_tests/unit
task dev:compose-start
task dev:test -- dl_connector_postgresql_tests/db
task dev:compose-stop
```

Select tests from that package's actual configuration. Unit suites often run without Compose. DB suites require the package's declared
local Compose dependencies; EAS acceptance requires the Enterprise stack.
`task dev:lint` checks formatting; use `task dev:lint-fix` when formatting changes
are intended and inspect its diff. The inspected pipeline uses Black, standalone isort,
Ruff, TOML sorting, deptry and mypy; do not assume Ruff replaces every tool.

## Build the selected Enterprise containers locally

For any row above, run the wrapper from backend-private root. Example for a
subscription worker change; replace target/tag with the selected matrix row:

```bash
./docker_build/run-project-bake eas_subscription_worker \
  --set 'eas_subscription_worker.tags=enterprise-subscription-worker:local-dev' \
  --set '*.platform=linux/amd64' --load
```

Choose one platform compatible with the local runtime (`linux/arm64` when
supported by all required bases/dependencies). The release workflow builds both
architectures and pushes manifests; local iteration uses a single loaded image.
An API/worker contract change needs both corresponding builds. A shared library
change may affect several rows: follow imports/dependency groups, not just the
application named in the bug. Do not invoke `eas_release.yaml` for local builds.

For a connection/query change, for example:

```bash
./docker_build/run-project-bake eas_control_api \
  --set 'eas_control_api.tags=datalens-eas-control-api:local' --load
./docker_build/run-project-bake eas_data_api \
  --set 'eas_data_api.tags=datalens-eas-data-api:local' --load
```

The wrapper supplies shared/private HCL and named source/base-image contexts.
Plain root-context `docker build` is not an equivalent recipe. EAS application
contexts select dependencies from `ops/eas` and produce a runtime venv.
Build bases, native DB/ODBC dependencies, architecture and image access can be
required even for a small Python change. Inspect bake definitions before forcing
platform/cache options. Use unique tags for concurrent developer sessions.

Enterprise `dev.sh` has no working backend source-watch mode; `--dev-build`
is not this build. Start authorized local dependencies, then use an override:

```yaml
services:
  control-api:
    image: datalens-eas-control-api:local
    environment:
      AUTH__TYPE: NATIVE
  data-api:
    image: datalens-eas-data-api:local
    environment:
      AUTH__TYPE: NATIVE
```

This example assumes the local Auth dependency is enabled; choose the actual
auth mode explicitly and retain the matching public key from the protected
configuration. Apply it with the common Compose loop to `control-api data-api`, preserving
project/features and keys. Host defaults: control 8010, data 8020; internal UI
endpoints remain `control-api:8080` and `data-api:8080`. For the inspected default container names, compare each pair of IDs:

```bash
docker image inspect datalens-eas-control-api:local --format '{{.Id}}'
docker inspect datalens-enterprise-control-api --format '{{.Image}}'
docker image inspect datalens-eas-data-api:local --format '{{.Id}}'
docker inspect datalens-enterprise-data-api --format '{{.Image}}'
```

Resolve actual container names from the selected project if different. Basic checks include
`/api/v1/ping` on control and `/ping_ready` on data, but neither proves queries.
Verify both running image IDs, especially after rerunning Enterprise bootstrap.

## Enable the feature being developed

Run the relevant local bootstrap after agreeing execution versus advice. Combine
flags for a scenario that spans features; rerunning bootstrap selects a new
service set and can stop features omitted from the new command.

| Scenario | Bootstrap example | Required path and observable result |
|---|---|---|
| Secure embeds | `./dev.sh --auth --sec-embeds` | UI embed route/token handling → `data-api-sec-embeds`; check expected rows plus a rejected invalid token/denied access |
| File connectors | `./dev.sh --auth --files` | Uploader API → Redis/ARQ worker → secure reader/S3/ClickHouse → dataset/query; upload CSV/XLSX, inspect types/rows, then replace the file and verify the saved connection |
| Background chart CSV/XLSX export | `./dev.sh --auth --background-exports --xlsx-background-exports` | Export API → Redis/ARQ worker → export-data-api/S3; poll job and download/check rows, columns and format |
| Workbook export/import | `./dev.sh --auth --export --temporal-ui` | Meta Manager → Temporal → Node activities → UI API/US; see [temporal.md](temporal.md) |
| Scheduled dashboard mail | `./dev.sh --auth --subscriptions --mailpit --nginx --temporal-ui` | Subscription API → Temporal worker → Snapter/Notify → local Mailpit; see [temporal.md](temporal.md) |

`--files` selects Redis, S3, ClickHouse and the three file services, and adds
`file` to the connector list. `--background-exports` selects Redis/S3 and the
three chart-export services. `--xlsx-background-exports` only sets the XLSX flag
inside that enabled path; use both flags. Workbook `--export` is a separate
feature and does not start the Python chart-export services.

After bootstrap, override only the changed services' images with the common
[local Compose loop](local-stack.md#test-a-changed-production-image). Preserve
the scenario flags when recreating UI/services. UI's actual environment names are
`ENABLE_SECURE_EMBEDDING` (bootstrap input `SEC_EMBEDS_ENABLED`),
`ENABLE_ASYNC_CHART_DATA_EXPORT`, `ENABLE_ASYNC_XLSX_CHART_DATA_EXPORT`,
`EXPORT_WORKBOOK_ENABLED`, and `DASH_MAILING_ENABLED` (bootstrap input
`SUBSCRIPTION_ENABLED`) as applicable.
Connector availability must still include `file` after recreating Control API.
Retain the existing shared encryption keys, export HMAC key, US/Auth credentials
and storage endpoints in protected configuration. Confirm every changed worker's
image ID as well as the API's before testing a job.

Usage tracking needs explicit attention: in the inspected `dev.sh`,
`--usage-tracking` selects logging/ClickHouse, but does **not** select `ut-worker`
or enable Temporal by itself. Start the required local `temporal` and `ut-worker`
services explicitly with matching auth/config after checking the script. Details
and the startup schedule side effect are in [temporal.md](temporal.md).

## Package-specific tests

Use the package-level Task commands above with that package's actual test path.
Most EAS wrappers declare empty `testpaths`; meaningful coverage lives in the
owning libraries. Select explicit test paths instead of assuming a wrapper's
default test command exercises its behavior.
Useful owners beyond the core query/connector suites:

| Change | Tests to inspect |
|---|---|
| Secure-embedding identity/access | `lib/bi_api_lib_sec_embeds_eas/bi_api_lib_sec_embeds_eas_tests/unit` plus shared query suites |
| File parsing/upload/ingestion | `lib/dl_file_secure_reader_lib`, `lib/dl_file_uploader_api_lib`, `lib/dl_file_uploader_worker_lib` and shared uploader packages |
| Chart exports | `lib/bi_export_api_lib`, `lib/bi_export_data_api_lib`, `lib/bi_export_worker_lib`, `lib/bi_export_lib` |
| Subscription HTTP/contracts | `lib/bi_subscription_api_lib`, `lib/bi_subscription_client` |
| Scheduled delivery/rendering | `lib/bi_subscription_worker_lib`: unit tests plus `bi_subscription_worker_lib_tests/db/app/test_workflow_subscription.py` and startup/readiness suites |
| Temporal client/worker infrastructure | `mainrepo/lib/dl_temporal`: unit tests and local Temporal-backed `dl_temporal_tests/db` |
| Usage tracking | `app_eas/eas_temporal_worker/eas_temporal_worker_tests/unit/activities/test_us_import.py` |

For example, from `lib/bi_subscription_worker_lib`, select unit tests first,
then its declared Compose fixtures and workflow suite:

```bash
task dev:test -- bi_subscription_worker_lib_tests/unit
task dev:compose-start
task dev:test -- bi_subscription_worker_lib_tests/db/app/test_workflow_subscription.py
task dev:compose-stop
```

Read fixtures to distinguish mocked gateways from real service coverage. The
acceptance image's Compose entrypoint is `.github/scripts/acceptance_tests/`;
inspect its target endpoints and protected inputs before adapting it to a local
stand. The release job includes global Docker container/volume cleanup: never
copy that cleanup into a developer's Docker session. Building the test image
alone does not execute acceptance tests.

## Acceptance and failure isolation

Through local UI create a connection to a source reachable from backend containers,
list tables/fields, save/reopen a dataset, preview known rows, calculate a field,
create/save a chart and reload expected values. Inspect failing browser requests
and correlated control/data/US logs. A new connector needs package entry points,
API/formula registration and metapackage inclusion as well as core implementation.
Exercise credential persistence and actual query/formula behavior across both APIs.

Keep Auth public keys, US tokens and connection-encryption keys consistent.
A source address reachable from the host may not be reachable from containers.
Do not mistake opensource `task e2e:*` for EAS/Auth acceptance: it builds other
apps and a public UI. The inspected `test_enterprise` task targets a shared remote
stand and obtains secrets; do not run it as a local test. Inspect destinations
before any integration/remote task, and use explicitly local endpoints.
