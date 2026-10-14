# UI stream

## Layout and installation selection

`datalens-ui-private` is a Node/React/TypeScript monorepo. `projects/opensource`
contains shared source; it is not an independent app package in this checkout.
Application and Enterprise entry points live under `projects/yandex` (including
Enterprise UI/server units). Enterprise builds use `dl-main-enterprise` and
`secure-embed-enterprise`; YC/YT/Public modes are different installations.
Read root AGENTS plus applicable `.agents/rules/` before changing TS/React/i18n,
and current `docs/dev/running.md`/`docs/e2e/README.md` for commands.

At the root, with the pinned Node/pnpm selected:

```bash
pnpm install --frozen-lockfile
pnpm ent:dev
```

Use `ent:*`, not Snapter's `en:*`. `pnpm build`/`pnpm docker:build` are not the
Enterprise defaults. For hot reload in containers use Enterprise `--dev-ui`.
For a host dev server, start dependencies with `./dev.sh --no-ui --auth`, then
prepare the UI environment according to its own docs: in the inspected checkout
it is `projects/yandex/.env`, **not `projects/enterprise/.env`**.
Existing `.env.enterprise` can take precedence and point to another environment;
check its presence/purpose without exposing credentials. Do not overwrite it blindly.

## Build and tests

```bash
pnpm ent:build
pnpm exec eslint <changed-paths> --quiet
pnpm typecheck
pnpm test:jest
```

Choose available scoped `typecheck:*` and Jest suites for the change. Enterprise
server compilation can emit without type diagnostics, so build success is not
a replacement for typecheck. UI tests include shared and installation-specific
behavior; i18n generation and script tests may also run.

For host UI Enterprise e2e, inspect current scripts, then use:
`pnpm ent:test:e2e:docker:up-no-ui`, `pnpm ent:dev`, `pnpm ent:test:e2e`.
The all-container alternative is `pnpm ent:test:e2e:docker`; clean up that test
stack with `pnpm ent:test:e2e:docker:down`. Do not point these tasks at a shared
or production installation. Include shared Enterprise-tagged suites relevant to
routes, editor, auth or public API, not just the Enterprise directory.

Production image, **from the UI repository root**:

```bash
docker buildx build --load --platform linux/amd64 \
  -f deploy/enterprise/Dockerfile \
  --build-arg app_version=local-dev -t enterprise-ui:local-dev .
```

The Dockerfile performs Enterprise manifest filtering, source pruning, compilation
and runtime assembly. Do not run `ent:prepare-install` or
`ent:clear-internal-files` in a normal checkout: they replace manifests or remove
source and belong in isolated build stages. Wire the local image into Enterprise
Compose using the common override loop. Runtime nginx listens on 8080 with the
Node server behind it; watch mode additionally uses the development asset/proxy
ports. Do not assume native dev success proves nginx/runtime routing.

## Acceptance by change

Trace browser → UI server/proxy → control/data API or US → visible result.
For a chart/editor change, save, reopen and reload it with known test data.
For auth, test redirects, cookies, roles and a permitted/denied object.
For public API, use the actual Enterprise schema and authorization; Cloud/YT
routes and token behavior are not interchangeable. For screenshots include
Snapter and its browser/network dependencies from the supporting-services guide.

Check generated i18n, stale build caches, heap pressure, wrong installation flags,
occupied ports and stale `.env.enterprise` as hypotheses when startup differs.
Do not fix them by deleting user data or switching to a different installation.
