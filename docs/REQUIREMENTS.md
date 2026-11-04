# Requirements

**Last updated:** 2026-11-03

Status labels:
- **Confirmed** — explicitly provided or approved by the project owner.
- **To decide** — information still needed before implementation.
- **Proposed** — a suggestion, not an approved requirement.



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
- Batch statuses: Available, Partially Used, Depleted and Spoiled. The first three derive from quantities. Updated 2026-11-03: a human-recorded Inventory spoilage Stock-Out that consumes the remaining batch quantity also sets Spoiled in the backend; Batch has no separate marking action.
- Require remarks on Inventory spoilage Stock-Out. Partial spoilage deducts only its quantity and retains the usable remainder. A full remaining-quantity spoilage Stock-Out updates the batch status without another deduction.
- Batch Management supports details/history, Batch Number/Flower search, native sorting/pagination and metadata-only editing. The Flower/Status/Date filter row and Mark as Spoiled button were removed from the approved UI on 2026-11-03.
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
- Payment statuses displayed by Order Management: Unpaid, Paid, and Refunded. Sales and Payment Management owns payment recording and transactions.
- Check available-to-allocate stock before confirmation, subtracting quantities already reserved by other confirmed orders. Reject an order if the full quantity is unavailable; do not create a pending-stock queue or allow overselling.
- Reserve inventory when an order is Confirmed. Reservation does not reduce physical on-hand stock and is not a Stock-Out.
- Actual customer-order Stock-Out is recorded through Inventory Management at the fulfillment-specific trigger: Delivery is stocked out only when successfully marked Delivered; Customer Pickup is stocked out only on confirmed actual collection / Completed. Coordinate reservation and Stock-Out to prevent double deduction.
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
- Require one full payment exactly equal to the order total; installments and additional payment transactions are prohibited.
- Payment statuses: Unpaid, Paid, and Refunded.
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


### User and Access Management — approved role structure (2026-11-02)

- The initial account roles are **Admin**, **Supplier/Staff**, and **Delivery Personnel**.
- Admin has full system access, including User and Access Management and Reports.
- Supplier/Staff can access authorized operational modules according to the system's permission rules.
- Delivery Personnel can access Delivery Management functions needed for assigned delivery work.
- Detailed account fields, account creation and password workflows, role-specific permission boundaries, deactivation behavior, and whether permissions are fixed or configurable remain to be finalized.
- No User and Access Management UI, authentication changes, backend, or database schema has been implemented.


### User and Access Management — account creation approved (2026-11-02)

**Confirmed**
- Only Admin can create Supplier/Staff and Delivery Personnel accounts. Public/self-service registration is excluded.
- Required fields are Full Name, Email Address, Contact Number, and Role.
- Generate a unique User ID automatically.
- Email Address must be unique across user accounts.
- Admin assigns a temporary password during account creation.
- The user must change the temporary password on first login before continuing.
- Store passwords using secure hashing; never store plain-text passwords.
- Each account has an Active or Inactive status.

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
