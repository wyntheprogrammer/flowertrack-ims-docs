# Requirements

**Last updated:** 2026-11-02

Status labels:
- **Confirmed** — explicitly provided or approved by the project owner.
- **To decide** — information still needed before implementation.
- **Proposed** — a suggestion, not an approved requirement.

## 1. General requirements

| ID | Requirement | Status |
|---|---|---|
| GEN-01 | The system is named FlowerTrack IMS. | Confirmed |
| GEN-02 | The system is a web application. | Confirmed |
| GEN-03 | The design direction is desktop-first and responsive. | Confirmed |
| GEN-04 | Supplier/Admin and Client interfaces must be different. | Confirmed |
| GEN-05 | The Client Panel should provide an online-shop-style experience. | Confirmed |
| GEN-06 | UI must be completed and reviewed before backend development begins. | Confirmed |
| GEN-07 | Development must be taught and completed step by step. | Confirmed |
| GEN-08 | Meaningful updates and newly learned technical knowledge must be recorded in Markdown and pushed to the docs repository, subject to authorization/access. | Confirmed |
| GEN-09 | The uploaded flower-shop dashboard screenshot and supplied design prompt are the visual reference for the Supplier Side UI. | Confirmed |

## 2. Supplier/Admin modules

The initial module order supplied by the project owner is the intended UI implementation order:

1. Dashboard
2. Flower Management
3. Inventory Management
4. Batch Management
5. Spoilage Prediction
6. Client Management
7. Order Management
8. Delivery Management
9. Sales and Payment Management
10. Reports
11. User and Access Management
12. Audit Trail

Descriptions are in `MODULES.md`.

## 3. Client navigation

The confirmed initial navigation order is:
1. Dashboard
2. Flower Catalog
3. Shopping Cart
4. My Orders
5. Order Tracking
6. My Payments
7. My Profile

## 4. Technology requirements

| Technology | Intended role | Status |
|---|---|---|
| Laravel | PHP web application framework | Confirmed direction |
| PHP | Server-side language | Confirmed |
| MySQL | Database | Confirmed direction |
| HTML | Page structure | Confirmed |
| JavaScript | Frontend interactions | Confirmed |
| Bootstrap | UI components and responsive layout | Confirmed preference |
| Flexbox | Layout and alignment | Confirmed preference |
| Custom CSS | Colors, sizes, spacing, typography and visual styling | Confirmed preference |
| Python | Possible ML model implementation/runtime | To decide during technical design |

## 5. UI requirements

- Bootstrap is the primary UI framework.
- Flexbox may be used for layout and alignment.
- Custom CSS controls the color scheme, font sizes, spacing, dimensions, and other visual details.
- Build reusable UI patterns where appropriate.
- Supplier/Admin is management-oriented and must follow the approved visual reference in `SUPPLIER_UI_DESIGN_REFERENCE.md`.
- The reference guides visual style and quality; it is not a pixel-for-pixel copy requirement.
- Do not copy the reference shop's name, logo, or business identity into FlowerTrack IMS.
- If the official FlowerTrack IMS logo is supplied separately, use it as the primary brand reference and do not alter it.
- Client is ecommerce-oriented and must remain visually distinct.
- The Supplier Side screenshot and prompt do not change module scope, workflows, or business rules.

## 6. Confirmed module requirements

### Flower Management — confirmed 2026-11-02
- Flower Code is automatically generated.
- Flower Name, Category, and Description are required.
- Primary Flower Image is optional, with one primary image per flower.
- Status options are Active and Inactive.
- Users can add, view, and edit flower records.
- Users can search by Flower Name or Flower Code and filter by Category or Status.
- Deactivate flowers with related records instead of permanently deleting them.
- Unit and Selling Price are not included in the approved scope yet.

### Inventory Management — approved 2026-11-02
- Provide an Inventory Overview with total and per-flower available quantities, stock statuses (Normal, Low Stock, Out of Stock), and Low Stock alerts.
- Use batch-based inventory. Every Stock-In automatically creates a Batch Record with a unique Batch Number.
- Derive consolidated inventory from batch quantities; do not maintain a separate independent stock quantity.
- Each flower has a configurable inventory unit and unit conversion. Stem is the standard base unit for inventory computation.
- Configure a fixed minimum stock threshold per flower in Flower Management; use it for Low Stock monitoring.
- Stock-In accepts the configured unit, converts the quantity to Stems, and includes a Remarks field/column. Stock-In records can be edited.
- Stock-Out is manual, includes a Remarks field/column, and covers outgoing stock such as customer orders and spoiled flowers.
- FIFO is the default Stock-Out allocation method. Automatically split a Stock-Out across batches as needed, consuming the oldest available batch first.
- Allow authorized users to override FIFO and select another batch; record the reason in Remarks.
- Allow partial batch consumption and retain the remaining quantity in the batch. Retain depleted batch records for traceability.
- Block Stock-Out when available stock is insufficient; negative inventory is not allowed.
- Record every Stock-In and Stock-Out in Inventory History with date/time, flower, batch, movement type, quantity, Remarks, and user.
- For stock correction, edit the Stock-In record and its Remarks as specified by the owner; no separate reversal-transaction workflow is included.
- Stock Adjustment / Physical Inventory Counting is explicitly deferred and excluded from the current scope.

### Batch Management — confirmed 2026-11-02
- Every Stock-In automatically creates one Batch Record with a unique, automatically generated Batch Number.
- Each batch is linked to its flower and originating Stock-In. Date Received and Initial Quantity are captured from Stock-In; inventory quantities use Stem as the standard base unit.
- Do not include a Supplier / Source field or column; the owner confirmed that the business sources the flowers directly.
- Harvest Date is required. Expected Freshness / Expiry Date is optional.
- Remaining Quantity is updated through Inventory Management Stock-Out allocations and is not directly editable in Batch Management.
- Batch statuses: Available, Partially Used, Depleted, and Spoiled. Available, Partially Used, and Depleted are derived from initial and remaining quantities; Spoiled is marked manually.
- Before marking a batch Spoiled, record its remaining spoiled quantity through a manual Stock-Out transaction with a required spoilage reason/remark.
- Batch Management supports viewing batch details and movement history; searching by Batch Number or Flower Name; filtering by Flower, Status, and Date Received; and editing metadata only.
- Do not allow direct quantity edits in Batch Management.
- Retain depleted and spoiled batch records for traceability; do not permanently delete batches with inventory history.
- Show warnings as the optional Expected Freshness / Expiry Date approaches. Warnings do not automatically deduct stock or mark a batch Spoiled.
- Spoilage Prediction does not automatically change batch status or deduct stock.

### Spoilage Prediction — confirmed 2026-11-02
- Use Machine Learning (ML) to estimate spoilage risk for flower batches.
- Use Harvest Date and Expected Freshness / Expiry Date as the two confirmed date inputs. Expected Freshness / Expiry Date is optional; batches without it must be handled using supported available data.
- Classify risk into Low, Medium, High, and Critical. Do not hardcode numerical thresholds until the model is calibrated and validated.
- Display spoilage-risk summaries and alerts on the Supplier/Admin Dashboard.
- Provide a dedicated prediction page with a batch risk list, search by Batch Number or Flower Name, filters by risk level and flower, prediction details, and available prediction explanations.
- Show warnings and recommendations only, including recommendations to prioritize high-risk batches for Stock-Out.
- Predictions must not automatically change batch status, deduct inventory, alter FIFO allocation, or create Stock-Out transactions. Actual spoilage remains manual.
- Include only batches with remaining usable inventory in active risk monitoring.
- Add Spoilage Event Records linked to Flower and Batch, capturing Spoilage Date, Spoiled Quantity, and Spoilage Reason. Use accumulated actual outcomes as labeled historical data for future ML training and evaluation.
- Do not claim model accuracy until the model has been trained, tested, and validated against sufficiently representative data.
- The exact ML algorithm, numerical risk thresholds, and model performance criteria remain for technical design after dataset assessment.
- Historical risk-assessment snapshots and manual condition-observation scoring are not included in the initial scope.

### Client Management — confirmed 2026-11-02

- Client ID is automatically generated.
- Full Name, Email Address, Contact Number, and Address are all required.
- Registration Date is recorded automatically; Account Status is Active or Inactive.
- Only Supplier/Admin users can create client accounts manually. Public/self-service client registration is not included in the initial scope.
- No email verification is required.
- Admin assigns a temporary password at account creation. The client must change it on first login.
- Passwords must be securely hashed and must never be stored as plain text.
- Email Address must be unique per client account.
- Support viewing all clients, adding clients, viewing complete profiles, editing client information, and activating/deactivating accounts.
- Search by Client ID, Full Name, or Email Address; filter by Account Status and Registration Date.
- Inactive clients cannot place new orders, but existing orders and transaction history remain accessible. Reactivation restores eligibility to place new orders.
- Client records with transaction history cannot be permanently deleted.
- Provide each client's Order History and Purchase Summary.
- Derive purchase summary values from Order Management records: total order count, total purchase amount, completed orders, and pending/ongoing orders. Do not maintain separate manually edited totals.
- Client Management owns client information and account status; Order Management owns order creation, processing, and statuses.

### Order Management — approved 2026-11-02

- Orders can originate from the Client Panel or be created manually by Supplier/Admin. Admin manually reviews and confirms orders.
- Each order has an automatically generated unique Order ID, a registered Client, automatically recorded Order Date, and at least one order item.
- Each order item links to a Flower and requires a quantity greater than zero. Unit Price comes from Sales and Payment Management and is saved as a snapshot; later price changes must not alter existing orders. Item subtotals and order total are calculated automatically.
- Delivery Address is required for Delivery and not required for Customer Pickup. Order Notes are optional.
- Order Status and Payment Status are tracked separately.
- Order statuses: Pending, Confirmed, Processing, Ready for Delivery or Ready for Pickup, Out for Delivery (Delivery only), Completed, and Cancelled.
- Payment statuses displayed by Order Management: Unpaid, Partially Paid, Paid, and Refunded. Sales and Payment Management owns payment recording and transactions.
- Check available-to-allocate stock before confirmation, subtracting quantities already reserved by other confirmed orders. Reject an order if the full quantity is unavailable; do not create a pending-stock queue or allow overselling.
- Reserve inventory when an order is Confirmed. Reservation does not reduce physical on-hand stock and is not a Stock-Out.
- Actual customer-order Stock-Out is recorded through Inventory Management at the fulfillment-specific trigger: Delivery is stocked out only when successfully marked Delivered; Customer Pickup is stocked out when marked Ready for Pickup. Coordinate reservation and Stock-Out to prevent double deduction.
- Partial fulfillment is not allowed in the initial scope.
- Clients may edit and cancel Pending orders. Supplier/Admin may edit orders before Processing and may cancel later non-completed orders with a required reason. Completed orders cannot be cancelled.
- Release reservations when an order is cancelled. If Stock-Out already occurred, inventory must be corrected/restored through Inventory Management before cancellation is finalized.
- Support Delivery and Customer Pickup. Delivery Management owns delivery assignment and delivery-specific tracking; Customer Pickup skips Out for Delivery.
- Preserve cancelled orders and status history, including date/time and responsible user.
- No pending-stock queue is included. Reject an order when available-to-allocate stock, after existing reservations, is insufficient for full fulfillment.

### Sales and Payment Management — approved 2026-11-02

**Selling prices and Sales Records**
- Supplier/Admin manually sets the selling price for each flower in Sales and Payment Management.
- A price change applies to new orders only. Existing orders retain their saved unit-price snapshots.
- Automatically create one Sales Record when an order becomes Completed. Generate a unique Sales ID and link it to the Order ID and client.
- Preserve the completed order's flower items, quantities, price snapshots, subtotals, order total, and applicable fixed delivery fee.
- Prevent duplicate Sales Records for the same order.
- Dashboard metrics: Total Sales, Number of Sales Transactions, Sales Today, Sales This Month, Top Selling Flowers, and Sales Trend Chart.
- Support sales transaction listing, details, search, filters, reports, and exports.

**Payments**
- Generate a unique Payment ID for each payment transaction and link it to the Order ID; client details come from the order.
- Record payment date/time automatically, payment method, amount paid, status, and applicable reference number.
- Supported methods: Cash, Bank Transfer, and E-wallet.
- Reference Number is required for applicable online payments (Bank Transfer and E-wallet).
- Admin manually verifies/confirms online payments and records/confirms cash payments.
- Allow partial payments and multiple payment transactions per order.
- Payment statuses: Unpaid, Partially Paid, Paid, and Refunded.
- Show payment history, total confirmed amount paid, and outstanding balance.
- Admin records and approves refunds; retain original payment and refund history.

**Reporting**
- Provide daily, weekly, monthly, and yearly sales summaries.
- Support PDF and Excel exports.

**Confirmed business rules**
- Recognize sales when the related order becomes Completed; payment receipt alone does not create a Sales Record.
- Sales reports count completed orders and subtract recorded refunds. Partial payments are not separate sales and must not be double-counted.
- Payment transactions are separate from Sales Records.
- Cancelled orders are excluded from completed sales.
- Include the fixed delivery fee in the order total when applicable.
- Sales and Payment Management owns selling prices, Sales Records, payment transactions, and refunds. Order Management displays payment status and summary only.
- No Sales and Payment Management UI, backend, database schema, payment gateway integration, or report generator has been implemented by this planning update.


### Reports — approved 2026-11-02

- Provide seven report categories: Sales, Payments, Inventory, Orders, Delivery, Clients, and Spoilage.
- Sales Reports include gross sales, refunds, net sales, sales by date/period, top-selling flowers, and sales trends.
- Payment Reports include confirmed payments, unpaid/outstanding balances, payment methods, and refund totals/history.
- Inventory Reports include current stock by flower, stock status, Stock-In/Stock-Out history, and batch quantities/remaining stock.
- Order Reports include order counts, status breakdown, Delivery versus Customer Pickup, and cancelled orders.
- Delivery Reports include delivered/failed/rescheduled deliveries, delivery performance, and delivery volume by period.
- Client Reports include registered clients, Active/Inactive counts, top clients by completed purchases, and client order activity.
- Spoilage Reports use actual spoilage quantities/events by flower and batch, plus spoilage trends.
- Support Daily, Weekly, Monthly, Yearly, and Custom Date Range filters, with category-appropriate search and filters.
- Present suitable summary cards, charts, and detailed tables.
- Support PDF export, Excel export, and direct printing through a print-friendly layout.
- Restrict Reports access to Supplier/Admin users.
- Reports are read-only and consume source records from their owning modules; they do not create or modify transactions.
- Sales calculations follow Sales and Payment Management: completed orders less recorded refunds; partial payments are not additional sales and cancelled orders are excluded.
- Inventory reports reflect batch-based physical stock and reservations accurately; reservations are not Stock-Out.
- Spoilage reports use actual recorded Spoilage Events, not predicted risk alone.
- Scheduled email delivery, custom report builder, automatic sharing, and predictive business analytics are excluded from the initial scope.
- This is a planning milestone only; no Reports UI, backend, database schema, report generator, exports, or print implementation has been created.

## 7. Business requirements still to gather

- Flower product categories, varieties, selling units, and additional product attributes beyond the confirmed Flower Management fields.
- Whether products include bouquets, plants, vases, wrappers, gifts, or other items.
- Any additional batch receiving validations or lifecycle behavior beyond the confirmed Batch Management scope.
- Other stock movement rules, including damaged, missing, reserved, returned, and adjusted quantities.
- Batch aging, shelf-life, and spoilage definition.
- Spoilage prediction data, method, and intended output.
- Whether a separate supplier/vendor directory is needed for any future purchasing workflow; Supplier / Source is not part of the approved Batch Management record.
- Checkout presentation and guest shopping policy; client account creation and initial access are already confirmed.
- Returns and post-completion order corrections.
- Detailed delivery personnel administration and any future delivery enhancements beyond the approved delivery workflow.
- Detailed reconciliation/accounting treatment beyond the approved Sales and Payment Management rules.
- Roles, permissions, and account lifecycle.
- Audit trail events and retention.
- Single-store or multi-branch operation.
- Security, backup, hosting, and deployment requirements.

## 8. Open decisions

Do not implement these as settled decisions until confirmed:
- Exact Laravel/PHP/Node versions.
- Laravel starter/authentication approach.
- Whether to use a UI admin template or build the Supplier/Admin UI from Bootstrap components.
- Whether Python is needed and how it communicates with Laravel.
- Exact FlowerTrack brand colors, fonts, and final design tokens; derive the brand palette from the official FlowerTrack logo when supplied.
- Exact database schema and technical implementation details.

### Delivery Management — approved scope (2026-11-02)

**Confirmed**
- Delivery Management handles Delivery fulfillment only; Customer Pickup is excluded.
- Automatically create a unique Delivery ID and delivery record when a Delivery order reaches Ready for Delivery. Link one delivery to one order.
- Retain Order ID, client name/contact, delivery-address snapshot, assigned personnel, scheduled date/time, status, optional notes, and delivery proof.
- Admin/Supplier and dedicated Delivery Personnel accounts participate. Admin manually assigns/reassigns active, authorized personnel and sets the delivery schedule.
- Search by Delivery ID, Order ID, or Client Name; filter by Delivery Status and scheduled date; view delivery details and per-order delivery history.
- Statuses: Pending, Scheduled, Assigned, Out for Delivery, Delivered, Failed, Rescheduled, Cancelled.
- An order must be Ready for Delivery before dispatch. Reassignment and status changes retain date/time and responsible-user history.
- Proof of Delivery requires recipient name, delivery remarks, photo proof, and delivery completion date/time.
- Failed attempts require a reason, remain in history, and do not automatically cancel the order. Admin manages rescheduling/reassignment. No fixed maximum attempt count is imposed initially; further attempts are subject to Admin decision.
- Delivery status is separate from Order Status. Successful delivery is coordinated with Order Management so the order can become Completed.
- Initial tracking is status updates only; no live GPS or continuous location tracking.
- Clients can view delivery status and scheduled delivery information in the Client Panel. Personal contact details of delivery personnel are not exposed.
- Apply a fixed delivery fee configured in system settings to eligible Delivery orders. The fee amount and detailed configuration controls will be determined during detailed design.
- No Delivery Management UI, backend, database schema, or GPS integration has been implemented.
