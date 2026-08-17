---
name: Provision and offboard Malt users via SCIM 2.0
description: >-
  Manage an organization's Malt user lifecycle through Malt's SCIM 2.0 endpoint — query, create,
  replace, deactivate and (rarely) delete users — with the retry and offboarding rules Malt's
  implementation actually enforces.
api: openapi/malt-exposed-apis-openapi.yml
operations:
  - findUsers
  - getUserById
  - createUser
  - replaceUser
  - modifyUser
  - deleteUser
generated: '2026-08-17'
method: generated
source: >-
  openapi/malt-exposed-apis-openapi.yml (operationIds verified against the spec),
  conventions/malt-conventions.yml, errors/malt-problem-types.yml,
  conformance/malt-conformance.yml
---

# Provision and offboard Malt users via SCIM 2.0

Malt exposes a real SCIM 2.0 (RFC 7643 / RFC 7644) `Users` endpoint so an enterprise can drive
Malt account lifecycle from its own identity provider. This skill covers using it directly.

## Before you start

- You need an **organization token** (or client team token). Unlike freelancer tokens these are
  **not self-served** — Malt's guidelines say to obtain them "from your Malt representative".
- Base URL: `https://api.malt.com`, resource path `/scim/v2/Users`
- Send the token as a bare value in the `Authorization` header (no `Bearer ` prefix).
- Accept `application/scim+json` or `application/json` — Malt offers both.

### What Malt's SCIM does NOT implement

Check these before wiring an IdP:

- **No `/scim/v2/Groups`.** User provisioning only — group and team sync are not available.
- **No `/ServiceProviderConfig`, `/Schemas` or `/ResourceTypes`** discovery endpoints. Many IdPs
  probe these on connection setup and will need manual configuration instead.
- **No `attributes` / `excludedAttributes`** sparse-fieldset parameters.
- **PATCH is restricted** to setting `active` to `false` (see step 4).
- **No SCIM error bodies are declared.** A SCIM-shaped `ErrorResponse` schema exists in the
  document but no operation returns it, so error handling is status-code-only.

## Steps

### 1. Look a user up before you create one

`findUsers` — `GET /scim/v2/Users?filter={scim filter}&startIndex={n}&count={n}`

- `filter` uses the SCIM filter grammar (e.g. `userName eq "someone@example.com"`).
- Pagination is SCIM index-based: `startIndex` is **1-based**, `count` is the page size. The
  response is a `UserPage` ListResponse carrying `totalResults`, `startIndex`, `itemsPerPage` and
  `Resources[]`.
- **Always do this first.** It is the only defence against the duplicate-creation problem in
  step 2.

`getUserById` — `GET /scim/v2/Users/{userId}` when you already hold the SCIM id.

### 2. Create a user

`createUser` — `POST /scim/v2/Users`

Body is a `SubmittedUserResource`:

- `userName` (**required**) — the unique identifier for the user
- `name` (**required**) — object with `givenName` and `familyName`, both required
- `externalId` — your own identifier for the user
- `phoneNumbers[]` — `{value, primary}`, formatted per RFC 3966
- `urn:ietf:params:scim:schemas:extension:malt:2.0:User` — Malt's extension, carrying
  `companyAttributionId` for your cost-attribution scheme

Returns **201** with the created `UserResource`.

> **Critical: this operation is not idempotent and Malt supplies no idempotency key.** There is no
> `Idempotency-Key` header anywhere in Malt's contract. If a `POST` times out you do **not** know
> whether the user was created. **Never blind-retry.** Re-run `findUsers` with
> `filter=userName eq "<userName>"` and only re-POST if the query comes back empty.

### 3. Replace a user's attributes

`replaceUser` — `PUT /scim/v2/Users/{userId}`

- Body is a full `SubmittedUserResource` — this is a **wholesale replace**, so send every attribute
  you intend to keep, not just the changed ones.
- Returns **200** with the updated `UserResource`.
- Idempotent by HTTP semantics: repeating the same PUT converges on the same state.

### 4. Offboard — deactivate, do not delete

`modifyUser` — `PATCH /scim/v2/Users/{userId}`

Body is a `UserPatchBody`: a `schemas` array plus an `Operations[]` array of `{op, value}`.

> Malt's own operation summary states PATCH **"only accepts setting `active` to `false` for now"**.
> Treat this endpoint as *deactivate user*, not as general SCIM PATCH. Do not attempt to patch
> `name`, `emails`, `phoneNumbers` or the Malt extension through it.

Returns **204 No Content**.

**This is the correct offboarding path.** Deactivation is reversible from Malt's side and does not
destroy history.

### 5. Delete only when deactivation is not enough

`deleteUser` — `DELETE /scim/v2/Users/{userId}`

- Returns **204** on success.
- Returns **403** when the user cannot be deleted — Malt's own description gives the reason as the
  user having activity on the platform. That is the common case for anyone who has actually worked
  through Malt, so expect this to fail for real users.
- **On a 403, fall back to step 4 (deactivate).** Do not retry the delete; it will keep failing.

## Error handling

| Status | Meaning | What to do |
|---|---|---|
| 401 | Token missing, malformed, expired or revoked | Request a fresh organization token |
| 403 | Wrong token type or insufficient scope | Confirm an **organization** token, not a freelancer token |
| 403 on DELETE | User has platform activity | Deactivate via PATCH instead |
| 404 | No user with that id | Resolve the id with `findUsers` and a filter |

No `429` and no `5xx` are declared. Bodies are frequently empty — a live unauthenticated request
returns 401 with `content-length: 0`. There are **no application error codes**, so do not attempt
to parse one.

## Notes for agents

- `createUser`, `replaceUser`, `modifyUser` and `deleteUser` all **mutate a real organization's
  user directory**. Confirm intent with a human before calling any of them.
- The sequence that is always safe: `findUsers` → decide → act. Reading first is not optional here,
  because it is the only duplicate protection the API offers.
- `deleteUser` is destructive and usually refused. Prefer `modifyUser` with `active: false`.
- The `*/*` request media type declared on all three write operations means a generated client may
  have no request schema. Construct SCIM bodies by hand against the schemas named above.
