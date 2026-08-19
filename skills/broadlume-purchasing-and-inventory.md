---
name: Purchase and receive flooring inventory
description: Raise and manage purchase orders in Broadlume BMS, receive material against them with the handheld barcode operations, and assign stock to a job.
api: openapi/broadlume-bms-openapi.yml
operations:
  - getCatalogItemsBelowSafetyStockLimit
  - vendorDetails
  - getVendorContacts
  - createPo
  - createPoLine
  - getPoLines
  - updatePoLine
  - updatePo
  - purchaseOrderDetail
  - listOfPurchaseOrdersByDate
  - readPurchaseOrderLinesThatAreOpen
  - findPoSByBarcode
  - inventoryFromBarcodeScan
  - inventoryByLocation
  - assignInventory
  - unassignInventory
  - shipsWorkOrder
  - assignInventoryToJob
  - getInventoryAssignedOnJob
  - committedInventory
generated: '2026-08-13'
method: generated
source: openapi/broadlume-bms-openapi.yml
---

# Purchase and receive flooring inventory

This flow spans the Purchase Order, Handheld, Inventory, Vendor and Committed groups. The
Handheld group is the warehouse-floor surface — it is barcode-first, and it is where PO
creation actually happens in practice.

**Prerequisite:** an authenticated session with company and active branches set — see
`broadlume-authenticate-and-scope-session.md`. Branch scoping matters more here than
anywhere else in the API, because stock is held per branch.

## 1. Decide what to buy

- `getCatalogItemsBelowSafetyStockLimit` (`GET /{alias}/lowstock`) is the reorder trigger.
  It is one of the 13 paginated operations: pass `page` (default 1) and `pagelimit`
  (default 1000).
- `committedInventory` shows what is already promised to jobs, so you do not reorder
  against stock that is spoken for.
- `getInventoryPointRecords` and `getInventoryFileRecords` give the underlying inventory
  records.

## 2. Check the vendor

`vendorDetails` and `getVendorContacts` for the supplier. `vendorTypeList` gives valid
vendor type codes.

## 3. Raise the purchase order

**Note the split.** PO *creation* lives in the Handheld group, PO *maintenance* lives in
the Purchase Order group. This is not a mistake in the documentation — it reflects where
the work happens.

- `createPo` (Handheld) creates the PO.
- `createPoLine` (Handheld) adds lines; `getPoLines` reads them; `updatePoLine` revises.
- `updatePo` (Purchase Order group) maintains the header — status, vendor, terms, freight,
  exchange rates and the three cost buckets.
- PO-level notes: `createsAPoNote`, `editsAPoNote`, `readsPoNotes`. Line-level comments:
  `createPoLineComment` and `getPoLineCommments`.
- Reference lists you should read rather than hard-code: `poShipvia`, `poFreightTerms`,
  `poShipmentMethodOfPayment`.

## 4. Track it

- `listOfPurchaseOrdersByDate` over a `startdate`/`enddate` window, or `lastdays`.
- `readPurchaseOrderLinesThatAreOpen` for what is still outstanding.
- `purchaseOrderDetail` for a single PO by `ponumber`.
- `getPoPointers` for status pointers.

## 5. Receive it on the floor

The Handheld group is designed for a barcode scanner:

- `findPoSByBarcode` to resolve a scanned barcode to its PO.
- `findPoSByVendorThatCanHaveAdditionalMaterialAdded` when adding material to an
  existing order.
- `inventoryFromBarcodeScan` and `inventoryLocationsFromBarcodeScan` to identify material
  and where it sits.
- `inventoryByLocation` and `getUnpickedInventory` for what is on hand and not yet picked.
- `assignInventory` / `unassignInventory` to bind rolls to work orders.
- `readWoHeaderFromBarcode` and `readWoLinesFromBarcode` to read the work order.
- `shipsWorkOrder` to ship it.
- `convertInventoryQuantity` for unit conversion — flooring is sold and stocked in
  different units, so do not do this arithmetic yourself.

## 6. Commit stock to a job

- `assignInventoryToJob` (`POST`) to commit.
- `getInventoryAssignedOnJob` to verify.
- `unassignInventoryOnJob` (`PATCH`) to release.

## Retry rule

As everywhere in this API there is **no idempotency key and no documented error
response** — only `200` is documented on all 256 operations. The write operations here
move physical stock and create financial commitments, so the read-back discipline is not
optional:

- After `createPo`, confirm with `purchaseOrderDetail` or `findPoSByBarcode` before
  retrying. A duplicate PO is a duplicate order to a supplier.
- After `assignInventory` or `assignInventoryToJob`, confirm with
  `getInventoryAssignedOnJob`. A double assignment misstates committed stock.
- After `shipsWorkOrder`, confirm before re-sending.

Never run these writes in parallel against the same branch.
