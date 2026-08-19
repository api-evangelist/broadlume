---
name: Take a flooring lead through quote to order
description: Walk the Broadlume BMS retail selling flow — capture a lead, work it, convert it to a quote, price the quote lines, and turn the accepted quote into an order with lines.
api: openapi/broadlume-bms-openapi.yml
operations:
  - createsLead
  - leadsList
  - leadDetail
  - updateLead
  - createLeadRoom
  - convertsAnExistingLeadIntoAQuote
  - createQuote
  - createQuoteLine
  - getQuoteLines
  - updateQuoteLine
  - getQuote
  - createOrder
  - createOrderLine
  - getOrder
  - updateOrder
generated: '2026-08-13'
method: generated
source: openapi/broadlume-bms-openapi.yml
---

# Take a flooring lead through quote to order

This is the marquee revenue flow in Broadlume BMS. It spans three of the API's resource
groups — Lead, Quote and Order — and the handoffs between them are explicit operations,
not implicit state changes.

**Prerequisite:** an authenticated session with a company and active branches set. See
`broadlume-authenticate-and-scope-session.md`. Every call below also needs `x-api-key`,
`token`, and the `company` parameter.

## 1. Capture the lead

Call `createsLead` (`POST /{alias}/lead`). Then work it:

- `leadsList` and `leadDetail` to find and read leads.
- `updateLead` to change lead fields.
- `createLeadRoom` (`POST`) to record the rooms being measured — the flooring-specific
  detail that makes a quote possible. Read them back with `getLeadRoom`.
- The notes surface is deliberately granular: `updatesLeadNotes`, `updatesLeadQuickNote`,
  `updatesLeadDispoNotes`, `updatesLeadDispStyleQuantityNotes`, and
  `createLeadOfficeNotes`. Pick the one that matches the note's purpose rather than
  overloading a single field — reporting reads these separately.
- If the lead does not qualify, use `updatesNonqualifiersOnALeadSendingExistingDescriptionWillRemoveFromALead`.
  Read the name carefully: **sending an existing description removes it**, so this operation
  toggles rather than sets.

## 2. Convert the lead to a quote

Call `convertsAnExistingLeadIntoAQuote`. Prefer this over creating a standalone quote with
`createQuote` — the conversion carries the lead's context forward, and a quote created
independently is not linked back to the lead that produced it.

## 3. Build the quote

- `createQuoteLine` per line item, `getQuoteLines` to read them back, `updateQuoteLine`
  to revise, `deleteQuoteLine` to remove.
- `createQuoteLineComment` / `updateQuoteLineComment` for line-level notes.
- `getQuote` for the header.
- Price against the catalog with `productPrice`, `productStock` and `catalogItems` from
  the Product group so you quote what is actually stocked.

Track follow-ups with `quotesByFollowUpDateRange`.

## 4. Turn the quote into an order

Call `createOrder` (`POST /{alias}/order`), then `createOrderLine` per line. Read back
with `getOrder`. Revise the header with `updateOrder`.

The order group then carries the job through fulfilment:

- `updateOrderLineInstallerSingleDateOnly` to assign the installer.
- `createOrderLineComment` and `createsAJobNote` for job-level notes.
- `assignInventoryToJob` (Inventory group) to commit stock.
- `jobStatusList` for the valid status values — read it rather than assuming.

## Retry rule — read before you write

**The Broadlume BMS API publishes no idempotency contract.** There is no idempotency-key
header or parameter on any of the 70 write operations, and no error responses are
documented at all — every operation documents only `200`.

Practically, that means:

- If `createOrder`, `createQuoteLine` or `createsLead` fails or times out, **do not blindly
  retry**. A retry may create a duplicate order, quote line or lead.
- Instead, read back first: `getOrder`, `getQuoteLines`, or `leadsList` filtered to the
  window, and confirm whether the write landed before re-sending.
- Make writes one at a time in this flow, not in parallel. There is no way to reconcile a
  partial batch from the contract.

## Dates and scoping

Date parameters throughout are 8-character `YYYYMMDD` strings, usually as
`startdate`/`enddate` with a `lastdays` alternative for relative windows. Remember that
active branches filter your reads — if a quote or order you just created does not appear
in a list, check `changeBranches` before assuming the write failed.
