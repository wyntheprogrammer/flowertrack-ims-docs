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

#### Order Management — approved scope (2026-11-02)

**Purpose:** Manage client orders from submission through fulfillment, coordinate inventory reservation and stock movement, and display payment status while Sales and Payment Management owns payment transactions.

**Order information**
- Order ID — automatically generated and unique.
- Client — linked to a registered client account.
- Order Date — recorded automatically.
- Each order must contain at least one item; every item links to a Flower and requires a quantity greater than zero.
- Unit Price comes from Sales and Payment Management and is saved as a snapshot on each order item. Later price changes must not alter existing orders.
- Item Subtotal and Order Total are calculated automatically.
- Delivery Address is required for Delivery, not Customer Pickup. Order Notes are optional.
- Order Status and Payment Status are tracked separately.

**Order sources and confirmation**
- Orders may be submitted through the Client Panel or created manually by Supplier/Admin.
- Admin manually reviews and confirms submitted orders.
- Inactive clients cannot place new orders, consistent with Client Management rules.

**Order statuses**
- Pending — submitted and awaiting Admin review.
- Confirmed — reviewed and accepted by Admin.
- Processing — preparation is underway.
- Ready for Delivery / Pickup — preparation is complete and the order is awaiting handover.
- Out for Delivery — handed to the delivery process; applies to Delivery only.
- Completed — delivered to the client or collected through Customer Pickup.
- Cancelled — terminal status; retain the order and cancellation reason for history and audit.

**Payment status display**
- Track Unpaid, Partially Paid, Paid, and Refunded separately from Order Status.
- Order Management displays payment status and summary only.
- Sales and Payment Management owns payment recording and transaction operations.

**Inventory coordination**
- Check available inventory before confirmation; do not confirm an order that cannot be fulfilled in full.
- Reserve required inventory when the order becomes Confirmed.
- Reservation is not an actual Stock-Out and must not deduct stock a second time.
- Actual Stock-Out is recorded through Inventory Management at the fulfillment point, following its batch allocation and FIFO rules.
- Partial fulfillment is not allowed in the initial scope.
- Cancellation releases any active reservation. If Stock-Out has already occurred, inventory must be corrected/restored through Inventory Management before cancellation is finalized.
- Negative inventory is prohibited.

**Editing and cancellation**
- Clients may edit their own Pending orders.
- Supplier/Admin may edit an order before it reaches Processing.
- Clients may cancel Pending orders.
- Supplier/Admin may cancel later non-completed orders with a required cancellation reason.
- Completed orders cannot be cancelled.
- Preserve cancelled orders and status history; record each status change with date/time and responsible user.

**Fulfillment**
- Support Delivery and Customer Pickup.
- Delivery orders require a delivery address. Delivery Management owns delivery assignment and delivery-specific tracking.
- Customer Pickup skips Out for Delivery.
- Order Management displays fulfillment progress; Delivery Management owns delivery operations.

**Still to resolve before implementation**
- Exact operational event when reserved quantity becomes an actual Inventory Stock-Out (for example, preparation completion or handover/dispatch). Reservation and Stock-Out must remain distinct to prevent double deduction.
#### Delivery Management — approved scope (2026-11-02)

**Purpose:** Coordinate and monitor deliveries for orders using the Delivery fulfillment method. Customer Pickup is not managed as a delivery.

**Delivery information**
- Delivery ID — automatically generated and unique.
- Order ID — linked to one Delivery order.
- Client Name and Contact Number — sourced from the related order/client record.
- Delivery Address — required and retained as a snapshot for the delivery.
- Assigned Delivery Personnel — selected from active, authorized delivery personnel.
- Scheduled Delivery Date and Time — set by Admin.
- Delivery Status — tracked separately from Order Status.
- Delivery Notes — optional.
- Proof of Delivery — recipient name, delivery remarks, and photo proof.

**Core features**
- View all delivery records and complete delivery details.
- Search by Delivery ID, Order ID, or Client Name.
- Filter by Delivery Status and scheduled date.
- Assign and reassign delivery personnel; retain assignment history.
- Schedule and reschedule deliveries.
- Update delivery status and record failed attempts with reasons.
- Record Proof of Delivery and view delivery history per order.
- Coordinate delivery progress with Order Management.

**Delivery statuses**
- Pending — delivery record exists but is not yet scheduled/assigned.
- Scheduled — delivery date and time have been set.
- Assigned — delivery personnel has been assigned.
- Out for Delivery — order has been dispatched.
- Delivered — successfully received by the client/recipient.
- Failed — a delivery attempt was unsuccessful.
- Rescheduled — the delivery schedule has been changed.
- Cancelled — delivery operation has been cancelled.

**Confirmed workflow and business rules**
- A delivery record is automatically created when a Delivery order reaches Ready for Delivery.
- Only Delivery orders may have delivery records; Customer Pickup does not create a delivery.
- Admin/Supplier and dedicated Delivery Personnel accounts may participate in delivery operations. Admin manually assigns or reassigns active, authorized delivery personnel.
- Admin sets the delivery date and time.
- An order must be Ready for Delivery before it can be dispatched / marked Out for Delivery.
- Reassignment and status changes retain their history, including date/time and responsible user.
- Marking a delivery Delivered requires recipient name, delivery remarks, and photo proof, plus the delivery completion date/time.
- Marking a delivery Failed requires a failure reason. Failed attempts remain in history and do not automatically cancel the order.
- Admin handles rescheduling and reassignment after a failed attempt.
- No fixed attempt limit is imposed initially; further attempts are subject to Admin decision.
- Delivery status remains separate from Order Status. A successfully delivered order is eligible to move to Completed in Order Management; delivery status updates must be coordinated with that module.
- Delivery tracking uses status updates only; live GPS tracking and continuous location tracking are excluded from the initial scope.

**Confirmed decisions**
- Apply a fixed delivery fee configured in system settings to eligible Delivery orders; the actual fee amount and detailed configuration controls will be specified during detailed design.
- Clients can view delivery status and scheduled delivery information in the Client Panel. Do not expose personal contact details of delivery personnel.

**Not implemented**
- This is a requirements/documentation milestone only. No Delivery Management UI, backend, database schema, GPS integration, or delivery transaction code has been implemented.


#### Sales and Payment Management — approved scope (2026-11-02)

**Purpose:** Manage flower selling prices, recognize completed orders as sales, record payment transactions, track balances/refunds, and provide sales/payment summaries and reports.

**Selling price and sales records**
- Supplier/Admin manually sets the selling price for each flower in Sales and Payment Management.
- A price change applies to new orders only. Existing orders retain the unit-price snapshot saved on each order item.
- Automatically create one Sales Record when an order becomes Completed.
- Generate a unique Sales ID and link the record to its Order ID and client.
- Preserve the completed order's flower items, quantities, unit-price snapshots, subtotals, order total, and applicable fixed delivery fee.
- Retain historical sales records; do not create duplicate sales records for the same order.
- Dashboard metrics: Total Sales, Number of Sales Transactions, Sales Today, Sales This Month, Top Selling Flowers, and Sales Trend Chart.
- Support viewing, searching, and filtering sales transactions, plus sales reports.

**Payment records and transactions**
- Generate a unique Payment ID for each payment transaction and link it to the related Order ID; obtain client details from the order.
- Record payment date/time automatically, payment method, amount paid, payment status, and applicable reference number.
- Supported payment methods: Cash, Bank Transfer, and E-wallet.
- Require a Reference Number for applicable online payments (Bank Transfer and E-wallet).
- Admin manually verifies and confirms online payments. Cash payments are also recorded and confirmed by Admin.
- Allow partial payments and multiple payment transactions for one order.
- Track payment statuses: Unpaid, Partially Paid, Paid, and Refunded.
- Show payment history, total amount paid, and outstanding balance per order.
- Admin records and approves refunds; retain the original payment and refund history.
- Support searching/filtering payment records and payment summaries.

**Reports and exports**
- Provide daily, weekly, monthly, and yearly sales summaries.
- Support PDF and Excel export for reports.

**Confirmed business rules**
- Sales are recognized when the related order reaches Completed, not merely when payment is received.
- Sales reporting is based on completed orders and subtracts recorded refunds. Partial payments must not be counted as additional sales.
- Payment transactions are separate from Sales Records.
- Payment Status is Unpaid until a confirmed payment is recorded; Partially Paid when confirmed payments are below the required total; Paid when confirmed payments meet the required total; Refunded when a refund is recorded according to the refund workflow.
- Order Management displays payment status and summary; Sales and Payment Management owns price setting, Sales Records, payment transactions, and refunds.
- Cancelled orders are not recognized as completed sales.
- Delivery fees are included in the order total when applicable, using the fixed fee configured in system settings.

**Not implemented**
- This is a requirements/documentation milestone only. No Sales and Payment Management UI, backend, database schema, payment gateway integration, or report generator has been implemented.


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
- Client accounts are created manually by Supplier/Admin; public/self-service client registration is not part of the approved initial scope.
- Clients receive a temporary password from the account-creation process and must change it on first login.
- No email verification is required for this initial account flow.
- Checkout, guest shopping, and detailed ordering rules remain to be confirmed.

#### Client Management — approved scope (2026-11-02)

**Purpose:** Allow Supplier/Admin users to create and maintain client accounts, manage account status, and review each client's order and purchase history.

**Client information**
- Client ID — automatically generated.
- Full Name — required.
- Email Address — required and unique per client account.
- Contact Number — required.
- Address — required.
- Registration Date — recorded automatically.
- Account Status — Active or Inactive.

**Account creation and access**
- Supplier/Admin users create client accounts manually. Public/self-service client registration is not included in the initial scope.
- No email verification is required.
- The Admin assigns a temporary password when creating the account.
- The client uses the temporary password for the first login and must change it before continuing.
- Passwords must be securely hashed; plain-text password storage is prohibited.

**Features and controls**
- View all registered clients and each client's complete profile.
- Add a client manually, edit client information, and activate/deactivate client accounts.
- Search by Client ID, Full Name, or Email Address.
- Filter by Account Status and Registration Date.
- View the client's complete Order History and Purchase Summary.

**Confirmed business rules**
- Each email address can belong to only one client account.
- Inactive clients cannot place new orders.
- Existing orders and transaction history remain accessible after deactivation.
- Reactivating an account restores eligibility to place new orders.
- Client records with transaction history cannot be permanently deleted.
- Client Management owns client information and account status; Order Management owns order creation, processing, and order statuses.
- Order History is sourced from Order Management records.
- Purchase Summary is derived from order records and includes total order count, total purchase amount, completed orders, and pending/ongoing orders; summary figures are not manually maintained as separate totals.

## C. Module dependencies to investigate

These are analysis topics, not finalized business rules:
- Flower Management supplies the flower/product information used by inventory and catalog.
- Batch Management records individual stock lots and supports batch-level tracking.
- Inventory Management presents overall stock and stock movements.
- Spoilage Prediction uses batch-related information, subject to the eventual prediction design.
- Sales and payment workflows still need detailed planning. Delivery Management requirements are approved, including a system-configured fixed delivery fee and client-side status/schedule tracking.
