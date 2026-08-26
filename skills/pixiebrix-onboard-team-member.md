---
name: pixiebrix-onboard-team-member
description: Invite a person to a PixieBrix team, place them in the right authorization group, and confirm the mods deployed to that group now reach them.
api: PixieBrix Developer API
base_url: https://app.pixiebrix.com/api/
generated: '2026-08-26'
method: generated
source: openapi/pixiebrix-openapi.yml
operations:
  - listOrganizations
  - listGroups
  - createOrganizationInvitation
  - listOrganizationInvitations
  - listMemberships
  - createGroupMembership
  - listGroupMemberships
  - listMemberDeployments
---

# Onboard a PixieBrix team member

Adds a person to a team and to the group that determines which mods they receive.
In PixieBrix, deployments are assigned to **groups**, not to individuals — so an
invitation alone delivers nothing. The group membership is the step that matters.

## Before you start

Every call needs `Authorization: Token <service-account-token>` and
`Accept: application/json; version=2.0`. The token comes from a Service Account
(Admin Console → Service Accounts). Its **Role is fixed at creation** — if a call
returns 403, the account is under-permissioned and must be replaced, not edited.

## Steps

1. **Find the organization.** `listOrganizations` — `GET /api/organizations/`.
   Keep the `organization_pk` for every subsequent path.

2. **List the groups.** `listGroups` —
   `GET /api/organizations/{organization_pk}/groups/`.
   Choose the group whose deployments the new member should receive. If none fits,
   create one with `createGroup` (`POST /api/organizations/{organization_pk}/groups/`).

3. **Check they are not already invited or a member.**
   `listOrganizationInvitations` — `GET /api/organizations/{organization_pk}/invitations/`
   and `listMemberships` — `GET /api/organizations/{organization_pk}/memberships/`.
   **Do this — do not skip it.** The API publishes no idempotency key, so a repeated
   `createOrganizationInvitation` is not deduplicated for you.

4. **Send the invitation.** `createOrganizationInvitation` —
   `POST /api/organizations/{organization_pk}/invitations/` with the email address.
   Returns 201. The invitee accepts through the app (`acceptMeInvitation`); you
   cannot accept on their behalf.

5. **Add them to the group once the membership exists.** `createGroupMembership` —
   `POST /api/groups/{group_pk}/memberships/`. Verify with `listGroupMemberships`
   (`GET /api/groups/{group_pk}/memberships/`).

6. **Confirm delivery.** `listMemberDeployments` —
   `GET /api/organizations/{organization_pk}/members/{user_pk}/deployments/`.
   The mods deployed to the group should now be listed against the user.

## Rules that apply here

- **No idempotency.** Steps 4 and 5 are `POST`s returning 201 with no replay
  protection. Always list first; on a network timeout, re-list rather than re-POST.
- **Pagination.** Add `page` and `page_size` (default 100, max 1000). Follow the
  `Link` header (`rel="next"`); `X-Total-Count` gives the total.
- **Errors.** The spec declares no error responses. Expect `{"detail": "..."}` with
  401 (bad token), 403 (role), 404 (wrong organization), 429 (throttled — back off,
  there is no `Retry-After`).
- **Reversal.** Remove a group membership with `destroyGroupMembership`
  (`DELETE /api/groups/{group_pk}/memberships/{id}/`) or the whole team membership
  with `destroyOrganizationMembership` (`DELETE /api/memberships/{id}/`). Neither
  has a published recovery window — removal is immediate.
