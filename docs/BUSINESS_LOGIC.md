# Business Logic

**Last updated:** 2026-11-02

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
- Inventory is batch-based. Every Stock-In automatically creates its own Batch Record with a unique Batch Number.
- Consolidated inventory is derived from batch quantities; there is no separate independent stock quantity.
- Each flower has a configurable inventory unit. Unit conversions are supported, and Stem is the standard base unit for all inventory computation.
- Stock-In accepts the configured unit and converts the quantity to Stems.
- Each flower has a fixed minimum stock threshold configured in Flower Management. Low Stock alerts use that per-flower threshold.
- FIFO is the default Stock-Out allocation method: consume the oldest available batch first.
- Partial batch consumption is allowed; the remaining quantity stays in that batch until depleted. Depleted batch records remain for traceability.
- Stock-Out automatically splits its quantity across batches when required by FIFO.
- Authorized users may override FIFO and select another batch; the reason must be recorded in Remarks.
- Stock-Out is blocked when available inventory is insufficient. Negative inventory is not allowed.
- Stock-Out is manual, including for spoiled flowers; spoilage prediction does not automatically deduct stock.
- Every Stock-In and Stock-Out has a Remarks field/column, and every movement is recorded in Inventory History.
- Inventory History records date/time, flower, batch, movement type, quantity, Remarks, and the user who performed the transaction.
- Stock-In records may be edited. For stock correction, the owner specified editing the Stock-In record and its Remarks; a separate reversal-transaction workflow is not included in the approved scope.

**Deferred**
- Stock Adjustment / Physical Inventory Counting is explicitly deferred and is not part of the current implementation scope.

**Confirmed — Batch Management (2026-11-02)**
- Every Stock-In creates one batch with an automatically generated unique Batch Number.
- Batch records link to the originating Stock-In and flower. Date Received and Initial Quantity come from Stock-In; quantities are normalized to Stems.
- Supplier / Source is not recorded because the business sources the flowers directly.
- Harvest Date is required. Expected Freshness / Expiry Date is optional.
- Remaining Quantity is controlled by Inventory Management Stock-Out allocations; Batch Management cannot directly change quantities.
- Available, Partially Used, and Depleted statuses are derived from the relationship between Initial Quantity and Remaining Quantity.
- Spoiled is a manually assigned status. Before a batch is marked Spoiled, its remaining spoiled quantity must be recorded as a manual Stock-Out with a required spoilage reason/remark.
- Batch Management supports viewing batch details and movement history, searching by Batch Number or Flower Name, filtering by Flower, Status, and Date Received, and editing metadata only.
- Depleted and spoiled batch records are retained for traceability and cannot be permanently deleted when inventory history exists.
- Approaching freshness/expiry dates produce warnings only. Warnings do not automatically deduct stock or change batch status.
- Spoilage Prediction does not automatically mark batches Spoiled or deduct inventory.

**Confirmed — Spoilage Prediction (2026-11-02)**
- Use Machine Learning (ML) to estimate spoilage risk for flower batches.
- Use Harvest Date and Expected Freshness / Expiry Date as the confirmed date basis. The expiry/freshness date remains optional; batches without it must be handled using supported available data.
- Use four risk levels: Low, Medium, High, and Critical. Numerical thresholds and algorithm selection remain pending dataset assessment and model validation.
- Show spoilage-risk summaries/alerts on the Supplier/Admin Dashboard and provide a dedicated Spoilage Prediction page.
- Provide warnings and recommendations only, including a recommendation to prioritize high-risk batches for Stock-Out.
- Predictions must not automatically change batch status, deduct stock, alter FIFO allocation, or create Stock-Out transactions. Actual spoilage remains manual.
- Active risk monitoring applies to batches with remaining usable inventory.
- Record actual Spoilage Events linked to Flower and Batch, including Spoilage Date, Spoiled Quantity, and Spoilage Reason. These records form the labeled historical dataset for future ML training and evaluation.
- Do not claim model accuracy until training and validation demonstrate it. The ML algorithm and risk thresholds are not yet selected.
- Historical risk-assessment snapshots and manual condition-observation scoring are excluded from the initial scope.

### Client accounts

**Confirmed — Client Management (2026-11-02)**
- Supplier/Admin users manually create client accounts. Public/self-service client registration is not included in the initial scope.
- Full Name, Email Address, Contact Number, and Address are required. Email Address must be unique per account.
- Client ID is automatically generated; Registration Date is recorded automatically; Account Status is Active or Inactive.
- No email verification is required for the initial account flow.
- Admin assigns a temporary password. The client must change it on first login. Passwords must be securely hashed and never stored as plain text.
- Inactive clients cannot place new orders. Existing orders and transaction history remain accessible after deactivation; reactivation restores eligibility to place new orders.
- Client records with transaction history cannot be permanently deleted.
- Client Management owns client information and account status. Order Management owns order creation, processing, and statuses.
- Client profiles support Order History and Purchase Summary. Summary metrics are derived from Order Management records: total order count, total purchase amount, completed orders, and pending/ongoing orders.

### Orders — confirmed Order Management (2026-11-02)

- Orders may come from the Client Panel or be created manually by Supplier/Admin. Admin manually reviews and confirms orders.
- Every order has a unique generated Order ID, registered Client, automatic Order Date, and at least one item. Each item requires a Flower and positive quantity.
- Unit Price is sourced from Sales and Payment Management and snapshotted per order item; future price changes do not modify existing orders. Item subtotals and order total are calculated.
- Delivery Address is required for Delivery; Order Notes are optional. Customer Pickup does not require a delivery address.
- Order Status and Payment Status are independent. Order statuses are Pending, Confirmed, Processing, Ready for Delivery / Pickup, Out for Delivery (Delivery only), Completed, and Cancelled.
- Payment statuses shown on orders are Unpaid, Partially Paid, Paid, and Refunded. Sales and Payment Management owns payment recording and transactions; Order Management displays status and summary only.
- Admin confirmation is manual. Check stock before confirmation and reject an order that cannot be fulfilled in full.
- Reserve inventory at confirmation. Reservation does not reduce physical stock; actual Stock-Out is a separate Inventory Management transaction at the fulfillment point. Prevent duplicate deduction.
- Partial fulfillment is excluded from the initial scope.
- Clients may edit/cancel Pending orders. Supplier/Admin may edit orders before Processing and may cancel later non-completed orders with a required reason. Completed orders cannot be cancelled.
- Cancellation releases any active reservation. If Stock-Out has already occurred, inventory must be corrected/restored through Inventory Management before cancellation is finalized.
- Preserve cancelled orders and status history, including timestamp and responsible user.
- Support Delivery and Customer Pickup. Delivery Management owns delivery assignment and delivery-specific tracking; Pickup skips Out for Delivery.
- **Still to decide:** the precise fulfillment event that triggers conversion of reserved stock into an actual Stock-Out. Keep reservation and Stock-Out as separate operations.

### Spoilage
- What does spoiled/at-risk mean for each flower type?
- Is there an entered expiry/shelf-life date, condition score, or observation history?
- Assess the volume, quality, and representativeness of Spoilage Event Records before selecting an ML algorithm or claiming prediction accuracy.
- Decide the model implementation/runtime and calibrated numerical risk thresholds during technical design after dataset assessment.

### Orders and sales — remaining decisions
- What is the exact operational event for actual Stock-Out after inventory reservation?
- What return and post-completion correction rules apply?
- At what point is a completed order recognized as a sale, and what sales reporting rules apply?

### Delivery and payments
- Which delivery methods and statuses are supported?
- Which payment methods are accepted?
- Who verifies payments?
- How are failed, pending, partial, and refunded payments handled?

Each rule must be confirmed and dated before implementation.

### Delivery workflow — confirmed rules (2026-11-02)

- Delivery applies only to orders whose fulfillment method is Delivery. Customer Pickup does not create a delivery record.
- When a Delivery order reaches Ready for Delivery, create one linked delivery record with a unique Delivery ID and an address snapshot.
- Admin manually assigns/reassigns active, authorized delivery personnel and schedules the delivery date/time. Preserve assignment and status history with timestamps and responsible users.
- Dispatch is allowed only after the related order is Ready for Delivery.
- Delivery statuses are Pending, Scheduled, Assigned, Out for Delivery, Delivered, Failed, Rescheduled, and Cancelled. These remain distinct from Order Status.
- Delivered requires recipient name, delivery remarks, photo proof, and completion date/time. Coordinate successful delivery with Order Management to complete the order.
- Failed requires a reason, remains in history, and does not automatically cancel the order. Admin handles rescheduling/reassignment. No fixed attempt cap is imposed initially; further attempts are subject to Admin decision.
- Initial delivery tracking is status-based only; no GPS tracking or continuous location tracking.
- Apply a fixed system-configured delivery fee to eligible Delivery orders; fee amount/configuration details are deferred to detailed design.
- Clients can view delivery status and scheduled delivery information; do not expose delivery personnel's personal contact details.
