---
name: pixiebrix-audit-agent-activity
description: Review PixieBrix activity governance — what policies are in force, which security events and shadow-AI detections fired, and what administrators changed.
api: PixieBrix Developer API
base_url: https://app.pixiebrix.com/api/
generated: '2026-08-26'
method: generated
source: openapi/pixiebrix-openapi.yml
operations:
  - listActivityPolicies
  - retrieveActivityPolicy
  - retrieveEffectivePolicyResponse
  - listMemberPolicies
  - listSecurityEvents
  - listShadowAis
  - listMemberTimelines
  - listAuditEvents
  - listAuditGroups
  - listAuditDeployments
  - createActivityExportJob
  - retrieveActivityExportJob
---

# Audit PixieBrix agent and user activity

PixieBrix's activity layer is what the product sells to customer-care and
compliance teams: policies constrain what people and AI agents may do in the
browser, and the reporting endpoints show what actually happened.

**Feature-flagged.** The effective-policy endpoint is explicitly "gated by the
activity-tracking organization feature flag". If it 403s or 404s on a tenant, the
feature is not enabled — that is a configuration fact, not a permission bug.

## Steps

1. **List the policies in force.** `listActivityPolicies` —
   `GET /api/activity/policies/`, detail via `retrieveActivityPolicy`
   (`GET /api/activity/policies/{id}/`). Policy families include AI prompt,
   prohibited language, clipboard and page-pattern rules.

2. **Resolve what a caller is actually subject to.**
   `retrieveEffectivePolicyResponse` — `GET /api/activity/policy/?organization=...`.
   Read the semantics carefully: the effective policy is the **additive union of
   every policy assigned to the user's groups**. There is no override or
   precedence — adding a user to a second group can only ever *tighten* what
   applies to them. Do not reason about it as last-write-wins.

3. **Map policies to people.** `listMemberPolicies` —
   `GET /api/activity/member-policies/`.

4. **Pull the violations.** `listSecurityEvents` —
   `GET /api/activity/reports/security/`, and `listShadowAis` for unsanctioned-AI
   detections. Bound them with the `start`, `end` and `tz` query parameters; filter
   with `user` and `groups`.

5. **Reconstruct a person's session.** `listMemberTimelines` —
   `GET /api/activity/reports/member-timeline/`.

6. **Read the administrative audit trail.** `listAuditEvents`
   (`GET /api/audit/organizations/{id}/`), `listAuditGroups`
   (`GET /api/audit/groups/{id}/`), `listAuditDeployments`
   (`GET /api/audit/deployments/{id}/`) — who changed policy, group and deployment
   configuration.

7. **Export for a regulator or a warehouse.** `createActivityExportJob`
   (`POST /api/activity/exports/`) then poll `retrieveActivityExportJob`
   (`GET /api/activity/exports/{id}/`). Request CSV with
   `Accept: "text/csv; version=2.0"`.

## Do not call this

`destroyPurgeActivityData` — `DELETE /api/activity/data/` — is an unscoped purge of
the organization's recorded activity. There is no dry-run, no confirmation
parameter, no scoping argument and **no published recovery path or window**. An
autonomous agent must never invoke it; route it to a human.

## Rules that apply here

- Time-bound every report query with `start`/`end`; these collections grow without limit.
- Paginate with `page`/`page_size` (max 1000) and follow the `Link` header.
- 429 on throttle, `{"detail": "..."}` envelope, no error codes to branch on.
