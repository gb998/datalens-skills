# Backend stream

## Layers and ownership

`datalens-backend-private/mainrepo/` is an editable public-source snapshot, not
a submodule. Shared `mainrepo/lib/dl_*` code must remain independently buildable
without private dependencies. Private packages use `lib/bi_*`; Enterprise
wrappers live in `app_eas/eas_*`; `app_ya_team` and `app_yc` are other installations.
Follow current repository ownership/mirroring rules rather than editing a public
mirror as a substitute for the private checkout.

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
local Compose dependencies; EAS acceptance requires the Enterprise stack. Lint may format source;
inspect resulting diffs. The inspected pipeline uses Black, standalone isort,
Ruff, TOML sorting, deptry and mypy; do not assume Ruff replaces every tool.

## Build the Enterprise containers locally

From backend-private root:

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
