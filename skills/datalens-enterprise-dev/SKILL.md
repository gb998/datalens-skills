---
name: datalens-enterprise-dev
description: Develop and test DataLens Enterprise locally from component source checkouts. Guide UI, US or backend work, local container builds and Compose integration, including file connectors, exports, subscriptions, Temporal, Auth, Meta Manager and Snapter. Use for local Enterprise development or testing unreleased component changes, not installing a customer distribution or deploying shared environments.
license: Apache-2.0
metadata:
  domain: datalens
---

# Local DataLens Enterprise development

## Start with the developer's stream

Before choosing commands, ask: **“Which stream are you working in—UI, US, or
backend—and what change or scenario do you want to test?”** Reuse an explicitly
stated role instead of asking again. For cross-stream work, establish the primary
owner and affected contracts, then load the additional relevant reference.
Auth, Meta Manager and Snapter are supporting components; ask which stream owns
an ambiguous change rather than treating them as the same service.

Also establish whether the developer wants execution or advice. In execution mode,
operate only on the named local environment. In advice mode, provide commands and
expected results for the developer to run. Registry/SSH access does not authorize
shared CI, publishing, production deployment or destructive database operations.

## Select the reference

| Stream / task | Read |
|---|---|
| UI: React, charts/editor, server proxy, public API, Enterprise UI | [references/ui.md](references/ui.md) |
| US: object persistence, revisions, ACL/DLS, migrations, storage APIs | [references/us.md](references/us.md) |
| Backend: connectors, datasets, formulas, secure embeds, file uploads, chart exports, subscriptions, usage tracking; all EAS release targets | [references/backend.md](references/backend.md) |
| Temporal workflows, scheduled dashboard emails, workbook export/import, stuck jobs | [references/temporal.md](references/temporal.md), plus the owning stream |
| Login/tokens, Meta Manager source/build, screenshots | [references/supporting-services.md](references/supporting-services.md) |

All streams first use [references/local-stack.md](references/local-stack.md).
For backend work, identify the affected application from the EAS release matrix
before choosing a build. For asynchronous work, identify its producer, worker,
queue/namespace and result store; an API-only build may leave the changed code
out of the executing worker.
Do not load unrelated stream references just because their repositories exist.
For customer installation from an archive use `datalens-onprem-install` instead;
its K3s workflow does not provide source checkouts or a development toolchain.

## Establish the actual checkout

Find the developer's existing repositories; never assume paths under someone
else's home directory. Record repo/remote, branch/HEAD, dirty changes, target
installation and runtime configuration without printing secrets. Read each repo's
AGENTS and relevant path-scoped rules before edits; let those rules override
these command examples when checkout versions differ.

Typical checkouts are `datalens-enterprise`, `datalens-ui-private`,
`datalens-us-private`, `datalens-backend-private`, `datalens-meta-manager-private`,
`snapter`, and `datalens-auth-private`. The Auth package is named `datalens-auth`;
a public `datalens-auth` checkout may have different scripts/build contexts.
Ask which checkout is available. Do not silently substitute public code for
private Enterprise implementations or clone inaccessible repositories.

Use the Enterprise checkout's `versions-config.json` and canonical Compose as
pins for unchanged dependencies. Modify source in the component that owns the
behavior, not in the packaging repo or generated ignored component clones.
Keep public/shared code independent of private-only imports where that repo
requires it. Never overwrite unrelated work or reset/rebase a developer's branch.

## Choose and complete the feedback loop

1. Reproduce the requested symptom on the selected revision and installation.
2. Choose hot reload for fast source iteration, or a production image when the
   behavior depends on packaging, startup, pruning or dependencies. They are
   separate verification steps, not interchangeable containers.
3. Run the smallest meaningful stream-specific lint/typecheck/test checks.
4. Build changed production images locally with unique tags when needed; keep
   all other services pinned. Use a temporary Compose override to wire them in.
5. Confirm the running container uses the built image and exercise the requested
   scenario through the actual UI/API and dependencies. A successful build or
   healthy endpoint does not prove the new code is running or the feature works.
6. Report checkout SHA, local image tag/ID, affected services, tests executed,
   end-to-end result and untested paths. Leave the agreed local environment usable.

Watch/preflight may migrate databases and create users. Preserve local data and
crypto/Auth keys; use disposable test databases for integration tests. Do not
use volume removal, database reinitialization or a remote test task as a generic
fix for startup failures. Credentials, `.env`, private keys and full rendered
Compose/Helm config must not be printed or committed.
