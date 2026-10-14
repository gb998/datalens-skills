# Shared local stack and production-image loop

## Sources of truth and prerequisites

In `datalens-enterprise`, read `DEV.md`, `dev.sh`, `docker-compose.base.yml`,
`docker-compose.dev.yml`, `versions-config.json` and the selected feature overlays.
Check Docker/Compose/Buildx, disk/RAM, ports, architecture and access to pinned
images. These repositories are not guaranteed to be public. Ask for authorized
registry access; do not copy another engineer's credentials or trigger shared CI.

`dev.sh` is a mutating bootstrap: it may clone repositories, install dependencies,
create/update `.env`, initialize/migrate databases, authenticate a registry,
stop/recreate services and start containers. Review it before execution. It is
not a read-only environment detector. Do not invoke `--rm-volumes`, `--reinit-db`
or root-container mode without a concrete authorized reason.

Use matching runtime/tool versions from the component checkout. The inspected
Node monorepos select Node 24/pnpm 10.17.1; Snapter uses npm. Recheck manifests
and lockfiles instead of upgrading tools globally. On an ARM development host,
choose whether the build is native or amd64/emulated before interpreting failures.

## Source-watch examples

From the Enterprise repository root, after access and data handling are settled:

```bash
./dev.sh --dev-ui --auth
./dev.sh --dev-us --auth
./dev.sh --dev-auth
./dev.sh --dev-snapter --auth --nginx
```

Choose one relevant command or combine only necessary flags. `--dev-ui`,
`--dev-us`, `--dev-auth`, `--dev-snapter` select source-mounted Node watch services;
unchanged dependencies remain pinned. `REPO_UI_PATH`, `REPO_US_PATH`,
`REPO_AUTH_PATH`, `REPO_SNAPTER_PATH` override default sibling paths.

There is **no working `--dev-backend` or `--dev-meta-manager` flag** in the
inspected bootstrap. `--dev-build` does not build changed production application
images or the Python backend. Meta Manager's Compose dev service exists, but
requires an explicit supported Compose command; `--export` starts its pinned image.
Check the current script before assuming a flag was added later.

Common optional flags: `--files` for file connectors, `--sec-embeds` for secure
embeds, `--background-exports` (plus `--xlsx-background-exports`) for chart data,
`--export` for workbook transfer, `--subscriptions --mailpit` for local mail
testing, and `--ldap`/`--oidc` for identity-provider testing. Backend owns the full
[EAS application matrix](backend.md#select-the-released-application); choose its
feature group before building. Workbook jobs and subscriptions share Temporal,
with different workers/queues; read [temporal.md](temporal.md) when using either.
Respect the current overlays and feature wiring.
`--snapter` also starts its browser dependency and S3; `--nginx` provides the
screenshot reverse-proxy route. Do not enable unrelated services by default.

## Host and container addresses

Inspected defaults; check actual mappings before using them:

| Service | Host port | Container address |
|---|---:|---|
| UI | 8080 (watch also 8880) | `ui:8080` |
| US | 3030 | `us:8080` |
| Auth | 8088 | `auth:8080` |
| Control API | 8010 | `control-api:8080` |
| Data API | 8020 | `data-api:8080` |
| Secure-embeds Data API | 8050 | `data-api-sec-embeds:8080` |
| File Uploader API | 8081 | `file-uploader-api:8080` |
| Export Data API / Export API | 8030 / 8040 | `export-data-api:8080` / `export-api:8080` |
| Subscription API | 8060 | `subscription-api:8080` |
| Meta Manager | 3050 | `meta-manager:8080` |
| UI API for workbook transfer | 3040 | `ui-api:8080` |
| Temporal gRPC | Not published by base Compose | `temporal:7233` |
| Temporal UI (`--temporal-ui`) | 8233 | `temporal-ui:8080` |
| Mailpit UI / SMTP (`--mailpit`) | 8025 / 1025 | `mailpit:8025` / `mailpit:1025` |
| Snapter | 8090 | `snapter:8080` |
| Playwright browser | See Compose | `snapter-pw:3000` |

Inside a container, localhost is that container. Preserve service-name endpoints
for container-to-container traffic; use host ports only from the host/browser.
For a host-run service, use the bootstrap's supported `--no-ui`, `--no-us`,
`--no-auth` endpoint wiring and verify host reachability on that OS.

US main/DLS, AUX, Auth and Meta Manager may share a PostgreSQL container but use
different databases/schemas. Temporal also has separate history and visibility
databases. Keep US master tokens, Auth signing/public keys
and backend credential-encryption keys consistent. Changing keys can invalidate
sessions or make stored connection credentials unreadable.

## Test a changed production image

Build from the component reference with a unique local tag, not `latest` or a
release tag. Example override file outside the tracked release configuration:

```yaml
services:
  ui:
    image: enterprise-ui:local-dev
    environment:
      AUTH_ENABLED: 'true'
      AUTH_ENDPOINT: http://auth:8080
  us:
    image: enterprise-us:local-dev
    environment:
      AUTH_ENABLED: 'true'
      AUTH_ENDPOINT: http://auth:8080
```

This example assumes bootstrap used `--auth`. Set the actual scenario's
non-secret runtime flags/endpoints explicitly: `dev.sh` exports some settings
only to its own Compose process, so a later Compose invocation can revert them.
For example, UI authentication defaults to off in base Compose and screenshot
routing uses `SNAPTER_ENDPOINT`; preserve that endpoint and `UI_APP_ENDPOINT`
when screenshots/proxying are enabled. Backend uses `AUTH__TYPE`; retain its
chosen mode and the matching keys. Preserve other enabled feature flags too.
Verify only a whitelist of non-secret runtime settings from selected containers,
not full env/config output. Existing secret values remain in protected `.env`
files or the configured secret mechanism; do not copy them into the override.

Include **only services changed for the scenario**. Apply it to the same Compose
project and environment already bootstrapped; preserve any required feature
files. For example, from Enterprise root with dependencies already running:

```bash
docker compose -f docker-compose.base.yml -f /tmp/enterprise-local.yml \
  up -d --no-deps --pull never ui us
docker compose -f docker-compose.base.yml -f /tmp/enterprise-local.yml images
```

This deliberately switches selected services from watch containers to runtime
images. Verify each container's `.Image` ID against `docker image inspect` of
the local tag, not just the displayed repository name. Restarting the pinned
stack with `dev.sh` can revert the override; repeat verification after any
bootstrap/recreation. Do not change `versions-config.json` to test local images.

Read logs with a bounded tail and inspect only needed fields, avoiding full
config output that may expose secrets. Validate both endpoints and the complete
user scenario. Keep evidence of the revision/image used; do not publish images,
import them into a shared cluster or modify shared registries unless requested.
