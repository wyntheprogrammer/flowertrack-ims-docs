# Reports Backend — Implementation Contract

**Prepared:** 2026-11-07
**Status:** Source review and implementation plan only. No Reports backend code, PHP tests, browser verification, or application push is claimed.

## Existing approved UI
- Keep `resources/views/supplier/reports.blade.php`, `resources/js/supplier/reports.js`, and their existing styles/visual design.
- Current Reports UI has seven report types: Sales, Payments, Inventory, Orders, Delivery, Clients, Spoilage.
- Keep search, page length, sorting, summary cards, chart, PDF, Excel, and Print behavior. Replace independent sample datasets with a read-only server-provided snapshot; do not present demo data as actual records.
- Existing route `/supplier/reports` is inside `role:admin`. Any new data endpoint must be protected by the same Admin-only middleware.

## Data contract and correctness
- Sales: only immutable completed-fulfillment Sales Records; gross is snapshotted completed total; subtract actual recorded refunds associated with that sale/order. Attribute refunds to original sale completion day (Asia/Manila), not refund entry day. Do not count payment confirmation as a sale.
- Payments: show full recorded payment, pending/confirmed/refunded state, and confirmed collections distinctly; refunds are separate deductions and do not create a second payment. Preserve integer-cent arithmetic.
- Inventory: physical on-hand from usable batch remaining quantities, active reservations separately, available = on-hand minus reserved; distinguish Stem and Bouquet quantities and never sum unlike units.
- Orders: all actual orders, current order status and independent payment state; cancellation is not a completed sale.
- Delivery: actual delivery records only; pickup orders excluded; assigned personnel and actual statuses, without treating scheduled as delivered.
- Clients: actual client records and associated order counts; net completed purchases from completed sales less refunds, keyed by client ID rather than display name.
- Spoilage: actual spoilage inventory movements only, not predicted risks; preserve batch, item, unit, quantity, remarks, and recorded time.
- No reports endpoint may mutate inventory, sales, orders, receipts, refunds, reservations, or audit history.

## Implementation sequence
1. Inspect the live migrations/models for exact column names and relationship keys, plus the full Reports JS rendering/export path.
2. Implement a read-only Reports service and Admin-only JSON controller route; return consistent DB-derived rows, summary inputs, and data definitions.
3. Replace hardcoded report samples with authenticated fetch, explicit loading/empty/error states, and consistent snapshot export/print; preserve the approved layout.
4. Add feature tests for Admin-only access, zero-data behavior, exact-cent/refund rules, inventory/reservation distinction, mixed units, and fulfillment-only sales.
5. Run PHP tests, JS syntax/build, and browser/export verification locally. Record actual results; never infer passes.
6. On owner acceptance, update AI_CONTEXT.md, DEVELOPMENT_PROGRESS.md and CHANGELOG.md; stage only intended application files, commit, and supply `git push origin main`. Verify remote commit before marking complete.

## Source review checkpoint
- Application `README.md` is still Laravel's stock README.
- `routes/web.php` currently renders Reports as an Admin-only view with no Reports data endpoint.
- `resources/js/supplier/reports.js` contains independent sample arrays and client-side PDF/Excel dependencies (jsPDF, jspdf-autotable, ExcelJS).
- `app/Services/SalesPaymentService.php` and `app/Services/OrderService.php` already implement financial and fulfillment boundaries; reuse their semantics rather than creating competing financial definitions.
