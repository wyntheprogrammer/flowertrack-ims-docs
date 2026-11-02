# System Modules

**Last updated:** 2026-11-02

## A. Supplier / Admin Panel

The project owner supplied the following module order. This is also the initial Supplier/Admin UI implementation order after the shared UI foundation.

| UI order | Module | Purpose |
|---:|---|---|
| 1 | Dashboard | Overall business overview |
| 2 | Flower Management | Manage flower information |
| 3 | Inventory Management | Monitor stocks and movements |
| 4 | Batch Management | Track individual flower batches |
| 5 | Spoilage Prediction | Predict which flower batches are at risk of spoilage |
| 6 | Client Management | Manage customers |
| 7 | Order Management | Manage client orders |
| 8 | Delivery Management | Manage deliveries |
| 9 | Sales and Payment Management | Manage sales and payments |
| 10 | Reports | Generate business reports |
| 11 | User and Access Management | Manage accounts and permissions |
| 12 | Audit Trail | Track system activities |

### Supplier/Admin UI notes

- Management-oriented layout.
- Desktop-first and responsive.
- Bootstrap components, custom CSS, and Flexbox as appropriate.
#### Flower Management — approved scope (2026-11-02)

**Purpose:** Maintain the flower records used by inventory and the Client catalog.

**Fields**
- Flower Code — automatically generated.
- Flower Name — required.
- Category — required.
- Description — required.
- Primary Flower Image — optional; one primary image per flower.
- Status — Active or Inactive.

**Actions and controls**
- Add, view, and edit flower records.
- Deactivate a flower instead of permanently deleting it when related records exist.
- Search by flower name or flower code.
- Filter by category and status.

**Confirmed rules**
- A flower can be saved without an image, but not without its name, category, or description.
- Unit and selling price are not part of the approved Flower Management scope yet; decide them with the inventory and pricing workflows.

#### Inventory Management — approved scope (2026-11-02)

**Purpose:** Monitor consolidated flower stock, receive and issue stock, track batches, and review every stock movement.

**Inventory overview**
- Show total available inventory and per-flower quantities.
- Show stock status: Normal, Low Stock, or Out of Stock.
- Show Low Stock alerts using each flower's fixed minimum stock threshold, configured in Flower Management.

**Stock-In**
- Record incoming stock.
- Every Stock-In automatically creates a corresponding Batch Record with a unique Batch Number.
- Allow entry using the flower's configured inventory unit (for example, Bunch, Bundle, or Box).
- Convert entered quantities to the standard base unit, Stem, for inventory computation.
- Include a Remarks field/column for every Stock-In movement.
- Allow editing of Stock-In records.

**Stock-Out**
- Record outgoing stock manually, including customer-order stock and spoiled flowers.
- Include a Remarks field/column for every Stock-Out movement.
- Use FIFO by default: allocate from the oldest available batch first.
- Automatically split a Stock-Out across multiple batches when the requested quantity exceeds the remaining quantity in the oldest batch.
- Allow authorized users to override FIFO and select another batch; record the reason in Remarks.
- Block the transaction when available stock is insufficient. Negative inventory is not allowed.

**Batch-based inventory and units**
- Consolidated available quantity is derived from batch quantities; do not maintain a separate independent stock quantity.
- Each flower has a configurable inventory unit and conversion to the standard base unit, Stem (for example, 1 Bunch = 10 Stems).
- Every Stock-In creates its own batch.
- Support partial batch consumption; a batch remains available with its remaining quantity until depleted.
- Retain batch records after depletion for history and traceability.

**Inventory History**
- Record every Stock-In and Stock-Out movement.
- Show date/time, flower, affected batch, movement type, quantity, Remarks, and the user who performed the movement.
- Preserve the history of batch allocations, including FIFO allocation or an authorized manual override.

**Stock correction**
- The owner specified that a correction should be handled by editing the Stock-In record and its Remarks; no separate reversal-transaction workflow is included in the approved scope.

**Deferred**
- Stock Adjustment / Physical Inventory Counting is explicitly excluded for now.

#### Batch Management — approved scope (2026-11-02)

**Purpose:** Manage individual flower batches created by Stock-In, including batch details, status, freshness monitoring, and traceability. Inventory Management remains responsible for stock movement transactions and quantity calculations.

**Batch information**
- Batch Number — automatically generated and unique.
- Flower — linked automatically from the Stock-In that created the batch.
- Date Received — recorded automatically from Stock-In.
- Initial Quantity — captured from Stock-In and normalized to the standard base unit, Stem.
- Remaining Quantity — maintained through Inventory Management Stock-Out allocations; not directly editable in Batch Management.
- Harvest Date — required.
- Expected Freshness / Expiry Date — optional.
- Remarks — optional batch notes.
- Supplier / Source is not recorded; the owner confirmed that the flowers are sourced directly by the business, so no Supplier/Source column is needed.

**Batch status**
- Available — remaining quantity equals the initial quantity and the batch is usable.
- Partially Used — remaining quantity is greater than zero but lower than the initial quantity.
- Depleted — remaining quantity is zero due to stock consumption.
- Spoiled — manually marked after the remaining spoiled quantity has first been recorded as a Stock-Out in Inventory Management.

**Actions and controls**
- View batches and complete Batch Details.
- Search by Batch Number or Flower Name.
- Filter by Flower, Status, and Date Received.
- View batch-specific Stock-In and Stock-Out history.
- Edit batch metadata only; quantities and inventory movements cannot be edited directly from Batch Management.
- Manually mark a batch as Spoiled, with a required reason/remark.
- Retain depleted and spoiled batch records for traceability; do not permanently delete batches with inventory history.

**Confirmed rules**
- Every Stock-In automatically creates one Batch Record with a unique Batch Number.
- Remaining Quantity is updated through Inventory Management transactions and batch allocations.
- FIFO remains the default Stock-Out allocation method; authorized FIFO overrides follow the Inventory Management rules.
- A spoiled batch must first have its remaining spoiled quantity recorded through a manual Stock-Out with a spoilage remark. Only then may its status be marked Spoiled. This keeps stock balances and Inventory History consistent.
- Expiry/freshness dates generate warnings as the date approaches; warnings do not automatically deduct stock or mark a batch as Spoiled.
- Spoilage Prediction does not automatically change batch status or deduct stock.

#### Spoilage Prediction — approved scope (2026-11-02)

**Purpose:** Use a Machine Learning (ML) model to estimate spoilage risk for flower batches and help the owner prioritize attention. Predictions are advisory only.

**Prediction inputs**
- Harvest Date and Expected Freshness / Expiry Date are the two confirmed date inputs.
- Expected Freshness / Expiry Date is optional; the prediction workflow must handle batches where it is not provided, using only supported available inputs.
- Relevant batch context may include Flower, Date Received, and Remaining Quantity when supported by the trained model and available data.
- The exact ML algorithm, numerical risk thresholds, and model metrics are not selected yet; determine these only after dataset assessment and model validation.

**Risk levels**
- Low — low predicted spoilage risk.
- Medium — increasing spoilage concern.
- High — high predicted spoilage risk.
- Critical — very high predicted risk requiring prompt attention.
- Do not invent fixed percentage thresholds before model calibration and validation.

**Dashboard and prediction page**
- Show spoilage-risk summaries and at-risk batch counts on the Supplier/Admin Dashboard.
- Provide a dedicated Spoilage Prediction page with batch risk list, search by Batch Number or Flower Name, filters by risk level and flower, and prediction details.
- Show the prediction basis/reason where the model can provide a reliable explanation.
- Provide recommendations to prioritize high-risk batches for Stock-Out.

**Advisory-only rules**
- Predictions and recommendations do not automatically change batch status, deduct inventory, change FIFO allocation, or create Stock-Out transactions.
- Actual spoilage remains a manual process under Batch Management and Inventory Management.
- Only batches with remaining usable inventory are included in active risk monitoring.
- Spoiled/depleted batches remain in records for traceability but are not active prediction targets.

**Spoilage Event Records for ML training**
- Record actual spoilage outcomes and link each event to the relevant Flower and Batch.
- Capture Spoilage Date, Spoiled Quantity, and Spoilage Reason.
- Use accumulated event records as labeled historical data for future model training and evaluation.
- Do not claim prediction accuracy until a model has been trained, tested, and validated using sufficiently representative data.
- Historical risk-assessment snapshots and manual condition-observation scoring are not included in the initial scope.

- For modules without an approved scope below, detailed fields, actions, filters, table columns, metrics, and workflows remain to be gathered.
- Do not invent operational rules from module names alone.

## B. Client Panel

The Client Panel must have a distinct ecommerce-style user experience.

| UI order | Navigation | Initial purpose |
|---:|---|---|
| 1 | Dashboard | Client landing/overview |
| 2 | Flower Catalog | Browse flowers |
| 3 | Shopping Cart | Review selected items |
| 4 | My Orders | View client orders |
| 5 | Order Tracking | Track order progress |
| 6 | My Payments | View payment information |
| 7 | My Profile | Manage client profile |

### Client UI notes

- Online-shop-style browsing and purchasing experience.
- Do not reuse the Supplier/Admin dashboard as the Client design.
- Product detail, checkout, registration, and guest-shopping rules need confirmation.

## C. Module dependencies to investigate

These are analysis topics, not finalized business rules:
- Flower Management supplies the flower/product information used by inventory and catalog.
- Batch Management records individual stock lots and supports batch-level tracking.
- Inventory Management presents overall stock and stock movements.
- Spoilage Prediction uses batch-related information, subject to the eventual prediction design.
- Orders, deliveries, sales, and payments need a confirmed lifecycle and relationship model.
