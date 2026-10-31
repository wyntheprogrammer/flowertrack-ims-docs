# Business Logic

**Last updated:** 2026-11-01

**Status:** Requirements to be gathered. Do not implement assumptions as confirmed rules.

## Areas requiring decisions

### Flowers and products
- What counts as a sellable product?
- Are flowers sold by stem, bunch, bundle, arrangement, or another unit?
- Are bouquets and non-flower accessories included?

### Inventory and batches
- How is stock received?
- Does every receiving transaction create a batch?
- How is total stock calculated from batch quantities?
- Which actions create stock movements?
- How are damaged, missing, reserved, returned, and adjusted quantities handled?

### Spoilage
- What does spoiled/at-risk mean for each flower type?
- Is there an entered expiry/shelf-life date, condition score, or observation history?
- What historical data is available?
- Should prediction be a warning, risk score, estimated date, or another output?
- Is Python/ML necessary, or would transparent date/rule-based alerts meet the requirement?

### Orders and sales
- When is stock reserved or deducted?
- Which order statuses are needed?
- How are cancellations, refunds, and returns handled?
- When does an order become a sale?

### Delivery and payments
- Which delivery methods and statuses are supported?
- Which payment methods are accepted?
- Who verifies payments?
- How are failed, pending, partial, and refunded payments handled?

Each rule must be confirmed and dated before implementation.
