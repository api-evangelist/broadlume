---
name: Authenticate and scope a Broadlume BMS session
description: Open an authenticated Broadlume BMS session, discover which companies the user may access, set the active branches, and keep the token alive for the duration of a task.
api: openapi/broadlume-bms-openapi.yml
operations:
  - apiVersion
  - authenticateUserCreateSession
  - userPermissions
  - changeBranches
  - sessionCount
  - sessionKeepalive
  - removeSession
generated: '2026-08-13'
method: generated
source: openapi/broadlume-bms-openapi.yml
---

# Authenticate and scope a Broadlume BMS session

Every other Broadlume BMS skill depends on this one. The API needs **two** headers on
almost every call, and it needs a tenant and branch scope set before reads return the
right rows.

## Before you start

You need three things, all issued by Broadlume — none are self-service:

- `alias` — the tenant client id. It is a **path segment**, not a header: every path is
  `/{alias}/...`.
- `x-api-key` — a static API key header, required on *every* request.
- a username and password. Both are capped at 8 characters by the underlying ERP.

Base URL: `https://api.rmaster.com/api`

## Steps

1. **Check reachability.** Call `apiVersion` (`GET /{alias}/version`). It needs only
   `x-api-key`, so it confirms your key and alias before you send credentials. It returns
   `VERSION` — the deployed build. Record it; the API carries no version in its URL, so
   this string is the only contract-version signal you get.

2. **Open a session.** Call `authenticateUserCreateSession`
   (`POST /{alias}/token`) with `x-api-key` and a body of `username`, `password`, and
   `granttype`.
   - Use `granttype: client` (the default) for interactive work. It idles out after a
     server-specified period, documented as 5 minutes by default.
   - Use `granttype: application` for unattended API-to-API work. It does not idle out
     provided it is used within a year of creation.

   The response carries `TOKEN` and `COMPANIES` — the list of company codes this user may
   act on. Send `TOKEN` as the `token` header on every subsequent call, alongside
   `x-api-key`.

3. **Pick a company.** Take a company code from `COMPANIES`. Nearly every operation in
   this API takes a `company` query or body parameter (203 of 256 do) and it is a 2-character
   code. There is no default — if you guess, you read another company's ledger or get nothing.

4. **Check what the user may do.** Call `userPermissions`
   (`GET /{alias}/permissions?company=`). Do this before attempting writes; the API
   documents no error responses, so a permission failure will not announce itself in a
   shape you can parse.

5. **Set the active branches.** Call `changeBranches` (`POST /{alias}/changebranch`) with
   `company` and a `branch` array. This is stateful and it is a trap worth reading twice:
   **every branch you do not pass is set inactive**, and all subsequent queries filter to
   the active branch list. If a later read returns fewer rows than expected, this is the
   first thing to check.

6. **Keep it alive.** Any API call refreshes the token's last-active time, so a keepalive is
   only needed when idle. When idle, call `sessionKeepalive` (`GET /{alias}/token`) inside
   the idle window. If you are running a long task with gaps, schedule this.

7. **Close it.** Call `removeSession` (`DELETE /{alias}/token`). This matters more than
   usual here: sessions are capped per tenant, and reissuing an **application** token
   requires a DELETE first to reset the issued key.

## Capacity

Call `sessionCount` (`GET /{alias}/sessioncount`) to read `SESSIONCOUNT` and `LIMIT`.
Concurrent sessions are capped per tenant and the cap is **not published as a number** —
this endpoint is the only way to know it. Check it before spinning up parallel workers,
and leave headroom for the humans using the ERP.

## What this API does not tell you

Read these as operating constraints, not as gaps to work around:

- **No documented error responses.** All 256 operations document only `200`. You cannot
  distinguish an expired token from a bad company code from the contract. Treat any
  non-200, or a 200 whose body lacks the fields you expect, as a failure and re-authenticate
  once before escalating.
- **No rate-limit headers.** There is no `Retry-After` and no documented status on
  exhaustion. Back off on your own schedule.
- **No idempotency contract.** See the order and purchase-order skills before you retry
  any write.
