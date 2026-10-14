# US stream

## Workspaces and contracts

In `datalens-us-private`, `projects/opensource` holds shared storage behavior;
`projects/extensions` adds reusable Enterprise/Yandex functionality; Enterprise
and Yandex consume both. Yandex has separate Russia/YC and Yandex Team modes.
Read root/path AGENTS and relevant `.agents/context/` files before changing APIs,
queries, migrations or tests. Cross-workspace production imports use the exposed
`api/` boundary, not another workspace's private `src/`.

For standalone watch, first prepare protected
`projects/enterprise/.env.development` with local connection overrides following
the checkout's tracked templates/setup script. It must exist before `en:dev`;
setup creates `.env` if missing. Check main/DLS/AUX DSNs and Auth/US endpoints
without printing values. Compose source-watch prepares the environment separately.

From the private monorepo root with pinned Node/pnpm:

```bash
pnpm install --frozen-lockfile
pnpm run en:dev
pnpm run en:build
pnpm run en:lint
pnpm run en:typecheck
```

Use `ext:lint`/`ext:typecheck` and consuming installations when shared/extensions
code changes. `os:*`, `ya:ru:*`, `ya:yt:*` target different installations.
For the complete local integration bootstrap use Enterprise `--dev-us --auth`.
Its watch hook invokes migrations/preflight; standalone `en:dev` prepares the
environment and watches but does not imply the same migration sequence.

## Databases and transactions

Main migrations: `projects/opensource/src/db/migrations`.
DLS: `projects/extensions/src/dls/db/migrations`.
AUX: `projects/extensions/src/aux/db/migrations`.
Use Enterprise workspace commands against an explicitly local/disposable database:

```bash
pnpm -F enterprise run us:db:migrate
pnpm -F enterprise run dls:db:migrate
pnpm -F enterprise run dls:db:seed
pnpm -F enterprise run aux:db:migrate
```

Root `db:main:migrate`, `db:dls:migrate`, `db:aux:migrate` target Yandex in the
inspected checkout; do not use them as generic Enterprise migrations. Confirm
main/DLS/AUX database connections and migration tables independently.
Use repository-provided typed service arguments, branded transactions,
`queryPrimary`/`queryReplica` and `dbTransaction`; propagate the callback's
transaction through every participating query. Do not add deprecated/untyped
query helpers. Follow current migration rules, including reversible `down` and
concurrent-index constraints. Root `lint:migrations` fetches `origin/main` and
compares committed added migrations; it is not an offline check of all uncommitted
files. Follow the repo's verification procedure rather than inventing a new
migration test framework.

## Tests and production image

After the relevant build:

```bash
pnpm -F enterprise run test:unit
pnpm -F enterprise run test:int:run <suite-substring>
pnpm -F enterprise run test:int-dls:run <suite-substring>
```

Integration suites exercise database/API/DLS behavior and need their test services.
Use the checkout's suite selection syntax; shared tests are compiled JavaScript.
For cross-workspace fixes include consuming Enterprise behavior, plus the affected
shared/Yandex contract when required by repository rules.

Production image, **from the US repository root**:

```bash
docker buildx build --load --platform linux/amd64 \
  -f deploy/enterprise/Dockerfile \
  --build-arg app_version=local-dev -t enterprise-us:local-dev .
```

This includes shared/extensions source and Enterprise runtime preflight.
Do not substitute an opensource workspace Dockerfile/context or mutate release
image pins. Start the local image via the common Compose override, then verify
its ID and database startup.

For object changes check create/save/reopen, revisions, update/delete/recovery,
owner IDs and persistence after the appropriate restart. For ACL/DLS changes
exercise permitted and denied users, not just an admin or master-token request.
For migrations check both a fresh local DB and an existing-data upgrade where
relevant; process readiness alone is not a schema/runtime check.
