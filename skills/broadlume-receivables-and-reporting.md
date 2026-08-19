---
name: Read Broadlume BMS receivables and financial reporting
description: Pull dashboards, aged receivables, customer payment history, invoices and general-ledger detail out of Broadlume BMS for collections and financial reporting.
api: openapi/broadlume-bms-openapi.yml
operations:
  - salesQuicklist
  - arQuicklist
  - apQuicklist
  - inventoryValuationQuicklist
  - workInProcessQuicklist
  - fiscalYearQuicklist
  - arBucketDefinitions
  - arByBucket
  - apBucketDefinitions
  - apByBucket
  - arNotesByBucket
  - arCustomerRegister
  - arOpenItems
  - agingInvoices
  - customerInvoices
  - paymentHistory
  - invoiceHeader
  - getInvoiceLines
  - invoiceDownload
  - getGlChartOfAccounts
  - getGlDetailByPeriod
  - getPreBuiltSalesReport
generated: '2026-08-13'
method: generated
source: openapi/broadlume-bms-openapi.yml
---

# Read Broadlume BMS receivables and financial reporting

This is the read-only reporting surface: Dashboard, Pivot, Accounts Receivable, Invoice,
General Ledger and Reports. Almost everything here is a `GET`, which makes it the safest
part of the API to automate.

**Prerequisite:** an authenticated session with company and active branches set — see
`broadlume-authenticate-and-scope-session.md`. Note that active branches filter these
reads, so a total that looks wrong is usually a branch-scope problem, not a data problem.

## Start at the dashboard

The Dashboard group is six cheap summary calls, and it is the right first stop before
pulling detail:

- `salesQuicklist`, `arQuicklist`, `apQuicklist`
- `inventoryValuationQuicklist`, `workInProcessQuicklist`
- `fiscalYearQuicklist` — call this early. It tells you the fiscal calendar the other
  numbers are keyed to, which you should not assume matches the civil year.

## Aged receivables

The Pivot group models ageing as **buckets**, and the bucket boundaries are configured per
company. Do not hard-code 30/60/90:

1. `arBucketDefinitions` to read the boundaries this company actually uses.
2. `arByBucket` for the aged balances.
3. `arNotesByBucket` (Accounts Receivable group) for collection notes against a bucket.

The payables side mirrors it exactly: `apBucketDefinitions` then `apByBucket`.

## Customer-level collections

- `arOpenItems` — open items for a customer.
- `agingInvoices` — aged invoices.
- `customerInvoices` — invoice list for a customer.
- `paymentHistory` — what they have paid.
- `arCustomerRegister` and `arCustomerRegisterHistory` — the register.
- `customer5YearSales` and `customerSales` for the relationship context behind a balance.
- `getListOfPossibleCustomerCreditStatusCodes` for valid credit status values.
- `paymentTypeCodes` for valid payment types.

Contact routing for collections: `customerArContacts`, `customerArContactNotes`,
`createCustomerArContactNotes`, `updateCustomerArContactNotes`, and
`allCustomerContactNotes`. These are the only writes in this skill — see the retry note
below.

## Invoices

- `invoiceHeader` then `getInvoiceLines` for detail.
- `invoiceDownload` returns the invoice as a **file**, not JSON. So does `edocDownload`
  in the Edocs group and `generateInstallerInvoicePdf` in the Install group. Handle these
  three as binary responses.
- `getInvoiceCommissionInfo` and `getJobCostInfo` for margin analysis.

## General ledger

- `getGlChartOfAccounts` for the account structure.
- `getGlReportFormatList` for the configured report formats.
- `getGlDetailByPeriod` for period detail.

## Prebuilt reports

- `getPreBuiltSalesReport`
- `getWrittenBusinessReport` and `getWrittenBusinessReportDetail`
- `salesVolumeByDate` and `inventoryValuation` (Pivot group)

## Practical notes

- **Dates are `YYYYMMDD` strings**, 8 characters, as `startdate`/`enddate`. Most
  date-ranged reads also accept `lastdays` for a relative window — prefer it for recurring
  jobs so you are not recomputing dates.
- **Response fields are uppercase** ERP column names (`COMPANY`, `BRANCH`, `STATUS`), and
  many list responses are an object whose keys are numeric strings (`"0"`) holding arrays,
  rather than a top-level JSON array. Parse defensively.
- **Pagination is opt-in.** Only 13 operations take `page`/`pagelimit`, among them
  `getSalesRepCommissionDetails`. The rest return whatever the window contains, with no
  total-count or next-cursor field on any response — so narrow by date rather than
  expecting to page.
- **No rate-limit headers exist**, and concurrent sessions are capped per tenant. If you
  are running scheduled reporting, check `sessionCount` and leave headroom for the humans
  in the ERP.
- **Retry rule for the contact-note writes:** there is no idempotency key and no documented
  error response anywhere in this API. Read back with `customerArContactNotes` before
  re-sending a note rather than retrying blindly.
