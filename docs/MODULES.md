# System Modules

**Last updated:** 2026-11-03




## Client Management UI — owner-approved 2026-11-03

- Owner approved Supplier Client Management after removing Account Status, Registered From/To and Reset filters.
- Approved UI: Add/Edit Client, view profile, Activate/Deactivate confirmation, sample order history and derived purchase summary.
- Keep Show Entries, compact Client ID/name/email search, native column sorting, pagination and table-only horizontal scrolling. Reuse Flower typography and controls; do not reintroduce filter rows automatically.
- Required name/email/contact/address, generated Client ID, automatic registration date and Active/Inactive status. Prototype email uniqueness is case-insensitive.
- Temporary password input is creation-only and preview-only; it is cleared and never copied into client records or persisted. Actual password hashing, first-login password change, authentication, authorization and inactive-account enforcement remain backend work.
- All client changes use in-memory sample data and reset on refresh. Sample order history is read-only. Completed Purchases sums completed sample order amounts; it is not a payment balance or final financial integration.
- Application areas: `resources/views/supplier/clients.blade.php`, `resources/css/supplier/clients.css`, `resources/js/supplier/clients.js`, app CSS/JS imports and `routes/web.php`. Keep route `supplier.clients` at `/supplier/clients`; remove its placeholder mapping.
- Validation: owner accepted the local UI. No independent current production build, browser execution or Client application commit/push has been verified.
- Next task: build/check and push Client Management, verify the application push, then begin Order Management UI. The separate Client storefront panel remains deferred.


## Client Management controls — owner decision 2026-11-03

- Remove Account Status, Registered From, Registered To and Reset controls from Client Management.
- Keep Show Entries, compact search by Client ID/name/email, native DataTables column sorting and pagination.
- Remove obsolete Blade markup, CSS selectors, JavaScript event bindings and date/status filtering logic together. Never reference removed element IDs.
- For upcoming Supplier module UI drafts, use this compact toolbar by default. Do not add extra filter rows automatically; ask for module-specific confirmation before introducing them. Previously documented Reports/Audit and other specialized filter requirements remain recorded for review when those modules are built.
- This decision supersedes the earlier Client status/registration-date filter requirement. Client account status and registration date remain record/profile fields; status editing and activation/deactivation remain available.
- Client Management UI is now owner-approved after filter removal. Its application build and push remain unverified.
## Spoilage Prediction UI — owner-approved 2026-11-03

- Owner approved the screen after typography was matched to Flower Management.
- Four summary cards: monitored batches, Low, Medium and combined High/Critical risk.
- Card-contained heading, compact Show Entries/search, native DataTables column sorting (risk severity order), pagination and table-only horizontal scrolling.
- View-only assessment modal shows batch/flower information, remaining stems, receiving/harvest/optional freshness dates, sample assessment date, indicators and recommended actions.
- Typography reference: title 25px, description 13px, table headers 11px, body/name 12px, code and risk badges 10px, search/select 12px, results info 11px and pagination 12px. Controls are 40px high; desktop action buttons 32px; modal title 19px.
- Fixed sample assessment date: 2026-11-03. Risk levels are explicit sample values, not computed predictions. No trained ML model, confidence score or accuracy claim.
- No stock deduction or batch-status mutation. Confirmed spoilage remains an Inventory Stock-Out with required remarks; a passed freshness date does not establish spoilage.
- Files: `resources/views/supplier/spoilage-prediction.blade.php`, `resources/css/supplier/spoilage-prediction.css`, `resources/js/supplier/spoilage-prediction.js`; CSS/JS imports in `resources/css/app.css` and `resources/js/app.js`.
- Replace the spoilage placeholder with an explicit view route, keeping `/supplier/spoilage-prediction` and `supplier.spoilage-prediction`.
- Validation: owner approved the local appearance. No independent browser run, current production build or application commit/push was verified for this module.
- Next exact task: build/check, commit and push Spoilage Prediction UI, verify push, then begin Supplier Client Management UI. The separate Client panel remains deferred.
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

- **2026-11-03:** Flower Management UI is owner-approved and is the visual reference for the next Supplier modules. Inventory, Batch Management and Spoilage Prediction UI are also approved; Supplier Client Management UI is next after the Spoilage application push is verified. Reuse styling while preserving each module's own business workflow.

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
- Minimum Stock Threshold — required, positive integer; used by Inventory Management for low-stock monitoring.
- Status — Active or Inactive.

**Actions and controls**
- Add, view, and edit flower records.
- Deactivate a flower instead of permanently deleting it when related records exist.
- Search by flower name or flower code.
- Approved UI (2026-11-03): Show Entries and compact search toolbar, native DataTables header sorting, and bottom pagination; category/status toolbar filters and a separate Sort dropdown are removed.

**Confirmed rules**
- A flower can be saved without an image, but not without its name, category, or description.
- Inventory unit/conversion belongs to Inventory Management. The minimum stock threshold is configured per flower in Flower Management. Selling prices are owned by Sales and Payment Management, not Flower Management.

#### Inventory Management — approved scope (2026-11-02)

**UI status — 2026-11-03:** Owner-approved local frontend prototype. Includes overview, Inventory List/History, Stock-In/Out, FIFO allocation preview, batch details, Stock-In editing and unit configuration. Follow the Flower Management design reference. Backend persistence and application push are tracked separately.

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

#### Batch Management — approved scope (updated 2026-11-03)

**UI status:** Owner-approved. Keep View/Edit actions, compact Show Entries/search and native sorting. Remove the filter row and separate Mark as Spoiled action; scroll only the table, with results/pagination outside the scroll area.

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
- Spoiled — set by the backend Inventory spoilage Stock-Out that consumes the batch's remaining quantity; no separate Batch marking action.

**Actions and controls**
- View batches and complete Batch Details.
- Search by Batch Number or Flower Name.
- Flower/Status/Date Received filter controls are removed from the approved Batch UI (2026-11-03).
- View batch-specific Stock-In and Stock-Out history.
- Edit batch metadata only; quantities and inventory movements cannot be edited directly from Batch Management.
- View the Spoiled status produced by Inventory; required spoilage remarks are recorded on the Inventory Stock-Out.
- Retain depleted and spoiled batch records for traceability; do not permanently delete batches with inventory history.

**Confirmed rules**
- Every Stock-In automatically creates one Batch Record with a unique Batch Number.
- Remaining Quantity is updated through Inventory Management transactions and batch allocations.
- FIFO remains the default Stock-Out allocation method; authorized FIFO overrides follow the Inventory Management rules.
- Inventory records the spoiled quantity as a Stock-Out with a required reason/remark. If it consumes the batch's remaining quantity, the same backend transaction sets the batch to Spoiled. Partial spoilage leaves the usable remainder; Batch Management does not perform a second deduction.
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
- Ready for Delivery or Ready for Pickup — preparation is complete and the order is awaiting handover.
- Out for Delivery — handed to the delivery process; applies to Delivery only.
- Completed — delivered to the client or collected through Customer Pickup.
- Cancelled — terminal status; retain the order and cancellation reason for history and audit.

**Payment status display**
- Track Unpaid, Partially Paid, Paid, and Refunded separately from Order Status.
- Order Management displays payment status and summary only.
- Sales and Payment Management owns payment recording and transaction operations.

**Inventory coordination**
- Check available-to-allocate inventory before confirmation, subtracting quantities already reserved by other confirmed orders. Reject an order if the full quantity is unavailable; do not create a pending-stock queue or allow overselling.
- Reserve required inventory when the order becomes Confirmed. Reservation does not reduce physical on-hand stock and is not an actual Stock-Out.
- Record actual customer-order Stock-Out through Inventory Management at the fulfillment-specific trigger: Delivery orders are stocked out when successfully marked Delivered; Customer Pickup orders are stocked out when marked Ready for Pickup.
- Apply Inventory Management batch allocation and FIFO rules to actual Stock-Out, coordinating reservation and deduction to prevent double-counting.
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
- No pending-stock queue is included. Reject a new order when available-to-allocate stock, after existing reservations, is insufficient for full fulfillment.
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



#### Reports — approved scope (2026-11-02)

**Purpose:** Provide Supplier/Admin with a centralized, read-only view of business performance and operational activity using records owned by the relevant modules.

**Report categories**
- **Sales Reports:** gross sales, refunds, net sales, sales by date/period, top-selling flowers, and sales trends.
- **Payment Reports:** confirmed payments, unpaid/outstanding balances, payment methods, and refund totals/history.
- **Inventory Reports:** current stock by flower, Normal/Low Stock/Out of Stock status, Stock-In/Stock-Out history, and batch quantities/remaining stock.
- **Order Reports:** order counts, order status breakdown, Delivery versus Customer Pickup, and cancelled orders.
- **Delivery Reports:** delivered and failed deliveries, rescheduled deliveries, delivery performance, and delivery volume by period.
- **Client Reports:** registered clients, Active/Inactive counts, top clients by completed purchases, and client order activity.
- **Spoilage Reports:** actual spoiled quantities, spoilage events by flower and batch, and spoilage trends.

**Filters and presentation**
- Support Daily, Weekly, Monthly, Yearly, and Custom Date Range filters.
- Provide category-appropriate search and filters.
- Present suitable summary cards, charts, and detailed tables.
- Provide a printable report layout.

**Export and access**
- Support PDF export, Excel export, and direct printing.
- Reports access is restricted to Supplier/Admin users.

**Confirmed business rules**
- Reports are read-only and use the source records maintained by their respective modules; Reports does not create or modify operational transactions.
- Sales figures follow Sales and Payment Management rules: completed orders less recorded refunds. Partial payments are not additional sales, and cancelled orders are excluded.
- Inventory reporting reflects batch-based physical quantities and reservations accurately; reservations are not physical Stock-Out transactions.
- Spoilage reporting uses actual recorded Spoilage Events, not predicted risk alone.
- No additional features such as scheduled email delivery, custom report builder, automatic sharing, or predictive business analytics are included in the initial scope.

**Not implemented**
- This is a requirements/documentation milestone only. No Reports UI, backend, database schema, report generator, PDF/Excel export, or print implementation has been created.


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
- Owner decision (2026-11-03): no Account Status or Registration Date filters or Reset control. Keep Show Entries, search, native column sorting and pagination.
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


#### User and Access Management — approved requirements

### User and Access Management — approved permissions and account lifecycle (2026-11-02)

#### Fixed role permissions

Permissions are fixed by the three initial system roles; individually configurable permission sets are excluded from the initial scope.

| Module / Function | Admin | Supplier/Staff | Delivery Personnel |
|---|---|---|---|
| Dashboard | Full access | Operational dashboard | Delivery-focused dashboard |
| Flower Management | Full access | View, Add, Edit, Deactivate | No access |
| Inventory Management | Full access | View, Stock-In, Stock-Out | No access |
| Batch Management | Full access | View and edit metadata | No access |
| Spoilage Prediction | Full access | View predictions and recommendations | No access |
| Client Management | Full access | View, Add, Edit, Activate/Deactivate | No access |
| Order Management | Full access | View, Confirm, Update Status, Cancel | No access |
| Delivery Management | Full access; manage and assign deliveries | Authorized operational access | Assigned deliveries only |
| Sales and Payment Management | Full access | Manage sales and payment operations | No access |
| Reports | Full access | No access | No access |
| User and Access Management | Full access | No access | No access |
| Audit Trail | Full access | No access | No access |

- Delivery Personnel may update delivery status and submit required Proof of Delivery only for deliveries assigned to their account.
- Delivery Personnel cannot modify orders, inventory, payments, delivery fees, or delivery assignments.
- Supplier/Staff cannot access Reports, User and Access Management, or Audit Trail.
- Enforce permissions on the server/backend, not only by hiding navigation items or UI controls.
- Supplier/Staff permissions are role-defined and are not individually configurable in the initial scope.

#### Account lifecycle rules

- Users may change their own passwords.
- Forgot-password/password-reset requests are handled by Admin. Admin issues a temporary password; the user must change it at next login.
- Admin alone may deactivate, reactivate, or change the role of an account.
- Admin alone may change a user's email address; the new address must remain unique.
- Deactivate accounts instead of permanently deleting them. Preserve associated records and audit history.
- Block login and invalidate active sessions when an account is deactivated.
- Record account creation, deactivation, reactivation, role changes, and Admin password resets in Audit Trail.
- Do not display or reveal passwords to Admin or other users; store only secure password hashes.
- Prevent deactivation of the last active Admin account.
- Account-creation email notifications are excluded from the initial scope.

#### Implementation status

These are approved requirements only. No User and Access Management UI, authentication implementation, backend authorization, session invalidation, or database schema has been implemented by this documentation update.

- Next Supplier/Admin planning module: Audit Trail.


## Audit Trail — finalized requirements (2026-11-02)

### Purpose and access
- Provide a centralized, chronological record of important operational, financial, administrative, and security-related system activities.
- Admin only may view, search, filter, export, and print Audit Trail records.
- Audit events are generated automatically by the system; ordinary users cannot manually create audit records.

### Activity coverage
Record important activities across these categories:
- Authentication: successful login, failed login, logout, password change, and password reset.
- Flower Management: flower created, updated, activated, and deactivated.
- Inventory Management: Stock-In, Stock-Out, and inventory record edits.
- Batch Management: batch created, metadata updated, and marked Spoiled.
- Spoilage Prediction: prediction generated.
- Client Management: client created, updated, activated, and deactivated.
- Order Management: order created, confirmed, status changed, and cancelled.
- Delivery Management: delivery created, assigned, reassigned, rescheduled, status changed, delivered/completed, and failed.
- Sales and Payment Management: selling price changed, Sales Record created, payment recorded/confirmed, and refund recorded.
- User and Access Management: account created, role changed, account deactivated/reactivated, and password reset.
- Include failed attempts for security-sensitive actions, not only successful actions.
- Ordinary page views and routine navigation are not included in the initial scope.

### Audit record fields
Each audit event should store, where applicable:
- Unique, automatically generated Audit ID.
- Server-side date and time.
- User ID, user-name snapshot, and role snapshot for the actor, when an authenticated account exists.
- Action and module/category.
- Affected Record ID, when applicable.
- Short description of the activity.
- Relevant previous values and new values for changes.
- Source IP address, when available.
- Result: Success or Failed.
- For unauthenticated events, retain available non-secret context even when no authenticated User ID exists.
- Never record passwords, temporary passwords, password hashes, or other authentication secrets.

### Audit log interface and retrieval
- Display audit records in a table, newest first.
- Provide search by User Name, User ID, Action, Module, and Record ID.
- Provide date filters: Today, Yesterday, Last 7 Days, Last 30 Days, and Custom Date Range.
- Provide filters by Module, User, Action, and Result (Success/Failed).
- Support sorting and pagination.
- Provide an activity detail view with the available actor, timestamp, action, module, affected record, description, previous/new values, IP address, and result.
- Support PDF export, Excel export, and direct printing of the filtered audit results.

### Integrity, retention, and security rules
- Audit records are append-only in the application: no user, including Admin, may edit or permanently delete them through the application.
- Corrections to business data create new audit events; they do not rewrite prior audit entries.
- Preserve audit history when related user accounts or business records are deactivated or otherwise changed.
- Use server-side date and time as the official event timestamp.
- Record the responsible actor and result; for unauthenticated activity, record the available context without inventing an account identity.
- Access is Admin only and must be enforced server-side.
- Keep records indefinitely in the initial scope. No automatic deletion or archival is included.
- Exclude real-time activity monitoring, email/SMS alerts, automatic suspicious-activity detection, advanced user-behavior analytics, and external log-storage integrations from the initial scope.

### Essential unresolved technical detail
- Define the actor-identification policy for events without an authenticated user (for example, failed login attempts): store no authenticated User ID and capture only safe, available context such as the submitted account identifier where appropriate, IP address when available, timestamp, and result. Never store submitted passwords or secrets.
- The append-only requirement is an application rule. Implementation design must also consider restricted database permissions and protected backups so ordinary application operations cannot silently rewrite or erase audit history.

### Implementation status
These requirements are finalized and documented only. No Audit Trail UI, event-capture implementation, export/print functionality, backend, or database schema has been implemented.

