---
name: pixiebrix-deploy-mod-to-group
description: Publish a PixieBrix mod/package version and deploy it to a group of end users, then watch the deployment for runtime errors.
api: PixieBrix Developer API
base_url: https://app.pixiebrix.com/api/
generated: '2026-08-26'
method: generated
source: openapi/pixiebrix-openapi.yml
operations:
  - listPackageMeta
  - listPackageVersionHeavies
  - listPackagePromotionPipelines
  - promotePackagePromotionPipeline
  - createDeployment
  - listDeployments
  - createDeploymentPermission
  - retrieveDeploymentDependencies
  - listActiveDeploymentGroups
  - listDeploymentErrors
  - createDeploymentAlertEmail
---

# Deploy a PixieBrix mod to a group

Takes a mod from the registry to a set of end users' browsers, and sets up the
error feedback loop so a bad deployment is visible.

## Before you start

`Authorization: Token <service-account-token>` on every call. Package publishing is
**append-only and versioned** — there is no delete-version operation — so a bad
release is rolled *forward* by promoting the previous version again, never rolled
back.

## Steps

1. **Find the package.** `listPackageMeta` — `GET /api/bricks/`. Filter with
   `q` or `package__name`. Registry browsing is also available unauthenticated-ish
   via `listRegistryBricks` (`GET /api/registry/bricks/`); note
   `GET /api/services/` is **deprecated** in favour of
   `GET /api/registry/bricks/?kind=3`.

2. **Pick the version.** `listPackageVersionHeavies` —
   `GET /api/bricks/{id}/versions/` returns versions newest-first with metadata and
   the last editor. Read `retrievePackageVersionConfig`
   (`GET /api/bricks/{id}/versions/{version}/`) to inspect the exact config you are
   about to ship.

3. **Promote it if you are moving between environments.**
   `listPackagePromotionPipelines` (`GET /api/pipelines/`) then
   `promotePackagePromotionPipeline` (`POST /api/pipelines/{id}/promote/`) copies a
   source-package version into the target package as a new version, with a commit
   message. This is the API form of the documented `@myorg-dev` → `@myorg` scope
   promotion.

4. **Lock the version if it must not move.** `createPackageLockCreate` —
   `POST /api/bricks/{id}/lock/`. Reversible with `destroyPackageLockCreate`
   (`DELETE /api/bricks/{id}/lock/`).

5. **Create the deployment.** `createDeployment` —
   `POST /api/organizations/{organization_pk}/deployments/`. Returns 201. **List
   `listDeployments` first** — there is no idempotency key and a retry creates a
   second deployment.

6. **Assign it to groups.** `createDeploymentPermission` —
   `POST /api/deployments/{deployment_pk}/groups/`. This is what actually puts the
   mod in users' browsers. Confirm reach with `listActiveDeploymentGroups`
   (`GET /api/deployments/{deployment_pk}/users/`).

7. **Check dependencies resolve.** `retrieveDeploymentDependencies` —
   `GET /api/deployments/{deployment_pk}/dependencies/` returns the dependency tree.
   A missing integration configuration surfaces here, not at runtime.

8. **Wire up feedback.** `createDeploymentAlertEmail` —
   `POST /api/deployments/{deployment_pk}/contacts/` to route alerts, then poll
   `listDeploymentErrors` (`GET /api/deployments/{deployment_pk}/errors/`) after
   rollout.

## Rules that apply here

- **Integration credentials are per-team.** They are configured separately for each
  team and selected when configuring the deployment — a mod promoted from dev does
  not carry dev credentials into production.
- **Reversal.** `destroyDeploymentDetail` (`DELETE /api/deployments/{id}/`)
  un-deploys from every assigned user immediately. No recovery window is published.
  Prefer narrowing with `destroyDeploymentPermission`
  (`DELETE /api/deployments/{deployment_pk}/groups/{id}/`) over deleting the
  deployment.
- **Errors.** `{"detail": "..."}`; 429 on throttle with no `Retry-After`.
