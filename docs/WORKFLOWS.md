# Workflows

## Live daily observations and user-activity catch-up - approved checkpoint, 2026-11-17

Steps 2 and 3 use live daily drafts and user-activity catch-up. Stock-In inserts one draft per batch and Philippine date; spoilage, actual pickup/delivery completion and other inventory movements rebuild that same day's values from the ledger inside the inventory transaction. Successful eligible Admin/Supplier login and logout verify missing completed days, including zero-movement days. Today stays a draft. There is no periodic/date-based scheduler; without eligible login/logout, catch-up waits.

Late changes invalidate derived history and rewind the checkpoint; a revision prevents advancing past a concurrent invalidation. Login/logout share the lock, process at most seven pending dates per event by default, and leave failures pending without blocking authentication. Failed, inactive, temporary-password, Client and Delivery accounts do not trigger catch-up. Existing business and prediction-eligibility rules are preserved. The prior Windows task is disabled and its installer/runner scripts are removed. See [Daily Observations](DAILY_OBSERVATIONS.md) for setup and review.

**Approved checkpoint:** [`bb8d9bb`](https://github.com/wyntheprogrammer/flowertrack-ims/commit/bb8d9bbbc989e8069587e78fefc364286c4fe468) is published on application main after Wyn's review/approval. Agent-run full regression: **137 passed / 1,760 assertions**; activity observation suite: **28 passed / 239 assertions**. No periodic observation scheduler remains; the prior local Windows task is disabled. This section supersedes older collector-only and timer-based descriptions below.

**Last updated:** 2026-11-03

**Status:** The Inventory-owned spoilage workflow below is confirmed as of 2026-11-03. Other sections retain their previously documented topic/proposal status.



## Pickup Stock-Out timing — owner decision 2026-11-04

This decision supersedes all earlier Ready for Pickup Stock-Out instructions.

- Confirmed reserves the full order quantity without reducing Inventory On-Hand or Batch Remaining.
- Ready for Pickup means prepared and awaiting collection. Keep the reservation active; do not create Stock-Out or deduct batch quantities.
- When staff confirms the customer actually collected the order and marks the Pickup order Completed, Inventory performs Stock-Out using FIFO batch allocation. Reduce On-Hand and Batch Remaining and consume the reservation in the same transaction, exactly once.
- Repeated completion requests must not deduct twice. Keep an order-linked movement and enforce transactional/idempotent fulfillment processing in the backend.
- Delivery timing remains successful Delivered; payment confirmation alone never deducts stock.
- Cancellation before actual pickup releases the reservation without a stock restoration because no pickup Stock-Out has occurred.
- Documentation updated only for this decision. Existing Order/Inventory prototype code still needs its trigger, active reservation states, confirmation text and sample data changed from Ready for Pickup to actual collection/Completed. No application change/build/push is claimed here.


## Payment policy — single full payment (2026-11-04)

This owner decision supersedes all earlier partial/installment/multiple-payment requirements and UI descriptions below.

- Each order accepts one payment for exactly its entire order total, including any delivery fee. Partial payments, installments, split payments and second collections are prohibited.
- Record Payment lists only non-cancelled orders with no existing payment record and no recorded refund. A pending full payment also removes the order from the dropdown.
- Amount is auto-filled/read-only. Validate exact equality again on record and confirmation; reject underpayment, overpayment and duplicate order payments. Future backend must enforce the same rule transactionally with an order-level uniqueness constraint.
- Recording remains Pending verification; Pending is a transaction state, not paid status. After Admin verifies the whole payment, the order becomes Paid. Unpaid orders may exist awaiting payment; remove Partially Paid from order status options/samples.
- Preserve Cash, Bank Transfer, E-wallet and required online references. Refunds retain their original payment/history, remain capped by the refundable confirmed amount and do not allow another collection in this prototype.
- Payment timing relative to fulfillment is not newly decided by this rule; this change does not add an unapproved payment gate to Order/Delivery transitions.
- Updated prototype logic prepared for `resources/js/supplier/sales-payments.js` and the partial sample in `resources/js/supplier/orders.js`. Isolated checks passed for exact totals, partial rejection, pending lock, duplicates and confirmation. Local build/browser verification remains pending.
- Application repository write was blocked by GitHub integration HTTP 403; application changes have not been pushed. Apply the supplied local changes and build/push separately. Documentation commits are independent.
- Current task: apply and verify the single-full-payment change, then proceed to Reports UI. Backend/database integration remains deferred.


## Supplier/Admin workflows to map

1. Dashboard review.
2. Flower/product creation and maintenance.
3. Stock receiving and inventory movement.
4. Batch creation and batch monitoring.
5. Spoilage risk review and action.
6. Client record management.
7. Order review and processing.
8. Delivery preparation and tracking.
9. Sales and payment handling.
10. Report generation.
11. User account and permission administration.
12. Audit activity review.

## Client workflows to map

1. Visit client dashboard.
2. Browse flower catalog.
3. View product details.
4. Add items to shopping cart.
5. Review cart and proceed to checkout.
6. Submit order.
7. View order status and tracking.
8. View payment records.
9. Manage profile.

The checkout and order submission steps are candidate flow details inferred from the online-shop requirement; exact behavior, payment rules, and guest/account requirements remain to be confirmed.


### Updated — spoilage workflow and Batch controls (2026-11-03)

This owner-approved change supersedes the earlier separate manual Mark as Spoiled action in Batch Management.

- The user records a Stock-Out with purpose Spoilage and required explanatory remarks in Inventory Management.
- For partial spoilage, deduct only the spoiled quantity; retain the usable remainder and its quantity-derived batch status.
- When a spoilage Stock-Out consumes a batch's entire remaining quantity, the backend transaction must also set that batch's status to Spoiled, without a second stock deduction.
- A batch depleted by ordinary consumption remains Depleted; a passed freshness date or ML prediction does not mark it Spoiled.
- Batch Management displays the resulting status and supports viewing/history and metadata editing. It has no separate Mark as Spoiled button.
- Keep depleted/spoiled batches and their transaction history.
- Batch listing uses search, Show Entries, native column sorting and pagination. Flower/status/received-date filter controls were removed from the approved UI.
- These cross-module status updates are the agreed backend behavior; current separate sample-data prototypes do not implement shared persistence.
