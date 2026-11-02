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

### Spoilage
- What does spoiled/at-risk mean for each flower type?
- Is there an entered expiry/shelf-life date, condition score, or observation history?
- Assess the volume, quality, and representativeness of Spoilage Event Records before selecting an ML algorithm or claiming prediction accuracy.
- Decide the model implementation/runtime and calibrated numerical risk thresholds during technical design after dataset assessment.

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
