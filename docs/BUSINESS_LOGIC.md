# Business Logic

**Last updated:** 2026-11-01

**Status:** Partially confirmed. Only rules explicitly approved by the project owner are listed as confirmed below. All other rules remain undecided.

## Areas requiring decisions

### Flowers and products

**Confirmed — Flower Management (2026-11-02)**
- Flower Code is automatically generated.
- Flower Name, Category, and Description are required.
- One Primary Flower Image may be attached; it is optional.
- Status is Active or Inactive.
- Search supports Flower Name and Flower Code; filters support Category and Status.
- A flower with related records should be deactivated rather than permanently deleted.
- Unit and Selling Price are not yet included in the approved Flower Management scope.

**Still to decide**
- What counts as a sellable product?
- Are flowers sold by stem, bunch, bundle, arrangement, or another unit?
- Are bouquets and non-flower accessories included?

### Inventory and batches

**Confirmed — Inventory Management (2026-11-02)**
- Inventory is batch-based.
- Every Stock-In transaction must create a corresponding Batch Record.
- Overall inventory is calculated from the quantities in its batches; it is not maintained as a separate independent quantity.
- FIFO (First In, First Out) is the default Stock-Out allocation method. The oldest received batch is consumed first.
- Stock movements must be recorded in Inventory History.

**Still to decide**
- How are damaged, missing, reserved, returned, and adjusted quantities handled?
- What is the stock adjustment and physical inventory counting workflow?
- What other movement types, validations, and permissions are needed?

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
