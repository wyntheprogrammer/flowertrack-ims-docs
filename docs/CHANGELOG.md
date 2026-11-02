# Changelog

**Last updated:** 2026-11-02

All meaningful documentation and project-planning updates are recorded here.

## 2026-11-02

### Approved — Reports requirements
- Approved all seven categories: Sales, Payments, Inventory, Orders, Delivery, Clients, and Spoilage.
- Confirmed Daily, Weekly, Monthly, Yearly, and Custom Date Range filters, category-appropriate search/filters, summary cards, charts, and detailed tables.
- Confirmed PDF, Excel, and direct print support.
- Restricted access to Supplier/Admin users.
- Confirmed read-only reporting based on source records, with no transaction creation or modification.
- Recorded sales calculation, reservation/physical inventory distinction, and actual Spoilage Event reporting rules.
- Excluded scheduled email delivery, custom report builder, automatic sharing, and predictive business analytics from the initial scope.
- No Reports UI, backend, database schema, report generator, exports, or print implementation was created.
- Next planning task: User and Access Management.


### Finalized — Order fulfillment and Inventory Stock-Out rules
- Confirmed that reservation at order confirmation does not reduce physical on-hand stock.
- Confirmed that reserved quantities are excluded from stock available for new orders.
- Confirmed that orders must be rejected when the full quantity cannot be allocated; no pending-stock queue or overselling.
- Confirmed Delivery Stock-Out on successful `Delivered` status and Customer Pickup Stock-Out on `Ready for Pickup`.
- Kept reservation and actual Stock-Out as separate coordinated operations to prevent double deduction.
- Updated Order Management, Inventory Management, requirements, business logic, progress, and AI continuity documentation. No application code or database behavior changed.
- Next planning task: Reports.

### Approved — Sales and Payment Management requirements
- Confirmed manual flower selling-price management in Sales and Payment Management; price changes apply to new orders while existing order items retain price snapshots.
- Confirmed automatic Sales Record creation when an order becomes Completed, with unique Sales ID, Order ID/client linkage, and duplicate prevention.
- Confirmed Cash, Bank Transfer, and E-wallet, with Admin verification/confirmation and applicable online payment reference numbers.
- Confirmed partial and multiple payments, payment history and outstanding balances, and Admin-recorded/approved refunds.
- Confirmed sales reporting based on completed orders less refunds; partial payments are not double-counted and cancelled orders are excluded.
- Confirmed daily, weekly, monthly, and yearly summaries plus PDF and Excel exports.
- Updated the next planning task to Reports. No application UI, backend, database, payment gateway, or report generator was implemented.


### Approved — Order Management requirements
- Confirmed Client Panel and Supplier/Admin order creation, with manual Admin review and confirmation.
- Confirmed generated Order ID, client/order/item data, positive quantities, automatic totals, conditional delivery address, and optional order notes.
- Confirmed price ownership in Sales and Payment Management and immutable unit-price snapshots on order items.
- Confirmed separate order/payment statuses, with payment transactions owned by Sales and Payment Management.
- Confirmed inventory check and reservation at confirmation, full-order fulfillment only, and a separate actual Stock-Out through Inventory Management at fulfillment.
- Confirmed Pending-order client editing/cancellation, Admin editing before Processing, reason-required Admin cancellation for later non-completed orders, reservation release, and inventory correction when Stock-Out already occurred.
- Confirmed Delivery and Customer Pickup, with Delivery Management owning delivery assignment/tracking.
- Recorded the remaining operational decision: the exact event that converts reserved inventory into actual Stock-Out. No UI, backend, database schema, or inventory transaction code was implemented.


### Approved — Client Management requirements and account access flow
- Confirmed that only Supplier/Admin users create client accounts manually; public/self-service client registration is excluded from the initial scope.
- Confirmed required Full Name, Email Address, Contact Number, and Address; automatically generated Client ID; automatically recorded Registration Date; and Active/Inactive account status.
- Confirmed unique email per client, no email verification, and temporary-password first-login flow: Admin assigns the temporary password and the client must change it on first login.
- Recorded the security requirement that passwords must be securely hashed and never stored as plain text.
- Confirmed inactive-client ordering restrictions, retention of existing order/transaction history, reactivation behavior, and prohibition on permanently deleting clients with transaction history.
- Confirmed Order History and Purchase Summary, derived from Order Management records, and the separation of responsibilities between Client Management and Order Management.
- Updated the next planning task to Order Management. This is a documentation/planning update only; no UI, authentication, backend, or database schema was implemented.

### Approved — Spoilage Prediction requirements
- Confirmed ML-based batch spoilage-risk prediction using Harvest Date and optional Expected Freshness / Expiry Date.
- Confirmed Low, Medium, High, and Critical risk levels; numerical thresholds and algorithm selection are deferred until dataset assessment and model validation.
- Confirmed Dashboard risk summaries/alerts and a dedicated prediction page with batch risk details, search, and filters.
- Confirmed advisory-only recommendations; predictions do not automatically alter batch status, stock quantities, FIFO allocation, or create Stock-Out transactions.
- Added approved Spoilage Event Records linked to Flower and Batch, capturing Spoilage Date, Spoiled Quantity, and Spoilage Reason, as labeled historical data for future ML training/evaluation.
- Recorded that model accuracy cannot be claimed until sufficient representative data and validation are available. No UI, backend, database, or ML model was implemented.
- Updated planning status: Spoilage Prediction is approved; Client Management is next to plan. Requirements planning remains active.

### Approved — Batch Management requirements
- Finalized Batch Management as the module for individual batch details, status monitoring, freshness warnings, and traceability; Inventory Management remains responsible for stock movement and quantity calculations.
- Confirmed the batch fields and removed Supplier / Source because the business sources flowers directly.
- Confirmed required Harvest Date and optional Expected Freshness / Expiry Date.
- Confirmed Available, Partially Used, and Depleted as quantity-derived statuses; Spoiled is marked manually.
- Confirmed metadata-only editing, batch search/filter, detail and movement-history views, and retention of depleted/spoiled records.
- Confirmed expiry warnings only; no automatic stock deduction or spoilage marking.
- Confirmed the spoilage workflow: first record remaining spoiled quantity through manual Stock-Out with a required spoilage remark, then mark the batch Spoiled.
- Updated the planning status: Batch Management is approved; Spoilage Prediction is the next module to plan. Requirements planning remains active; no module UI or backend work was started.

## 2026-11-01

### Clarified — per-module documentation and Git workflow
- Confirmed the repeatable completion cycle: finish and review one module, update the relevant Markdown documentation immediately, provide a complete application Git build/check + stage + commit + push script, verify the owner's push output, and only then proceed to the next module.
- Clarified that the documentation repository and application repository are separate and their updates/pushes must be tracked independently.
- Required targeted staging and explicit protection of `.env`, `vendor/`, and `node_modules/`.


### Added
- Initial FlowerTrack IMS documentation structure.
- Project overview and requirements baseline.
- Supplier/Admin module list in the owner's specified order.
- Client Panel navigation and ecommerce-style requirement.
- Technology stack and supplied repository reference catalog.
- System architecture direction and open architecture decisions.
- UI/UX guidelines for Bootstrap, Flexbox, and custom CSS.
- UI-first development plan.
- Development progress tracker and AI continuity instructions.
- Date requirement for meaningful Markdown updates.
- Supplier Side UI design-reference document with the approved dashboard screenshot and supplied design-prompt requirements.
- Supplier Side dashboard screenshot stored at `assets/ui-references/supplier-dashboard-reference.jpg`.
- Verified application repository tree snapshot and current frontend file baseline in `docs/AI_CONTEXT.md`.
- Repository-first inspection rule: inspect the live application repository before asking the owner to provide committed folder/file structure.

### Clarified
- JavaScript is the intended frontend scripting language.
- Python is a possible technology for spoilage prediction, pending requirements.
- The docs repository is separate from the application-code repository.
- The Supplier Side dashboard screenshot is an approved visual reference only; it is not the official FlowerTrack IMS logo and must not be copied as a branded template.
- The earlier exclusion of the external Supplier dashboard reference is superseded by the owner's later approval.
- The local development environment uses XAMPP's bundled MariaDB 10.11.16; do not refer to that server as Oracle MySQL.
- The public application repository `wyntheprogrammer/flowertrack-ims` exists on branch `main`; its initial Laravel skeleton is committed and pushed.

### Completed — initial environment setup
- Installed XAMPP under `C:\xampp` and resolved initial Apache/MySQL port conflicts so the local XAMPP dashboard/phpMyAdmin became accessible.
- Installed and verified PHP 8.3.36 and Composer 2.11.1.
- Enabled the PHP ZIP extension and verified the PHP extensions required by the Laravel application.
- Installed Laravel application skeleton 13.11.0 in `C:\xampp\htdocs\FlowerTrack`.
- Started the Laravel development server with `php artisan serve` and confirmed the default welcome page at `http://127.0.0.1:8000`.
- Opened the project in Visual Studio Code using `code .`.
- Installed and verified Node.js 26.13.0 (`node -v`) and npm 11.20.0 (`npm -v`).
- Verified Git 2.52.0.windows.1 with `git --version`.
- Configured Git global commit identity with user name `wyn` and email `wyntheprogrammer@gmail.com`.

### Documentation updates
- Recorded the actual local environment versions, verified setup, remaining checks, and next task in `DEVELOPMENT_PROGRESS.md`.
- Updated `AI_CONTEXT.md` with the local setup baseline and the instruction to proceed one step at a time.
- Updated `TECHNOLOGY_STACK_AND_REFERENCES.md` to distinguish XAMPP's MariaDB from Oracle MySQL and record installed versions.
- Updated `AI_CONTEXT.md` and `DEVELOPMENT_PROGRESS.md` with the verified application repository tree and current frontend entry-point status.
- Corrected the documentation README to identify the application repository as already created and pushed.

### Completed — application Git setup
- Created the public GitHub application repository `wyntheprogrammer/flowertrack-ims`.
- Initialized Git in `C:\\xampp\\htdocs\\FlowerTrack`, staged the Laravel starter project, and created initial commit `ca10822` (`Initial Laravel project setup`).
- Renamed the local branch to `main`, added the GitHub repository as `origin`, and successfully pushed `main` to GitHub. The local branch tracks `origin/main`.

### Not yet done
- No UI screens have been implemented.
- No database schema has been approved.
- No backend or database implementation has started.


### Completed — Bootstrap foundation (local working copy)
- Installed Bootstrap 5.3.8 and Popper 2.11.8 with npm; removed the Tailwind packages and Tailwind Vite plugin.
- Updated Vite configuration and frontend CSS/JavaScript entry points to load Bootstrap.
- Ran npm run build successfully.
- Replaced the Laravel starter welcome view with a simple FlowerTrack IMS Bootstrap verification page.
- Confirmed the page displays correctly in the browser at http://127.0.0.1:8000.
- Recorded that these application changes are verified locally; their push to the application GitHub repository has not been confirmed.

### Completed — Supplier/Admin dashboard UI first pass (local working copy)
- Created the Supplier/Admin Blade layout and dashboard view.
- Kept Gemma's Flower Shop logo in the sidebar and moved FlowerTrack IMS application name/subtitle to the top navbar to separate business branding from application identity.
- Removed the navbar search field at the owner's request.
- Refined sidebar navigation spacing and Quick Actions panel spacing.
- Added the two owner-supplied images as `public/images/flower-banner.jpg` and `public/images/bloom-business.jpg`.
- Used the wide flower image on the right side of the welcome banner without stretching it, and the pink rose image in the Bloom Your Business panel.
- Preserved the previously approved dashboard content/design after the owner requested reverting an alternative redesign.
- Owner confirmed the current dashboard is acceptable.
- These application changes are local; the system repository push is the next task and is not yet confirmed.


### Updated — Bootstrap Icons standard
- Adopted Bootstrap Icons as the standard icon library for interface icons across Supplier/Admin and Client panels.
- Installed the `bootstrap-icons` npm package and imported `bootstrap-icons/font/bootstrap-icons.css` through `resources/css/app.css`.
- Updated Supplier sidebar and Supplier Dashboard interface icons (summary cards, panel headings, Quick Actions, and Recent Activities) to Bootstrap Icons.
- Kept flower emojis in Top Selling Products as product visuals rather than interface icons.
- Preserved the approved Supplier Dashboard layout and existing content.
- Application changes are local; build verification and push to `wyntheprogrammer/flowertrack-ims` are pending.

### Still pending
- Remaining Supplier/Admin modules in the order documented in `MODULES.md`.
- Client ecommerce UI screens.
- Full UI review and approval before backend/database work.


## 2026-11-02

### Approved — complete Inventory Management scope
- Finalized Inventory Overview, Stock-In, Stock-Out, Inventory List, Batch Details, and Inventory History sections.
- Confirmed unique batch creation for every Stock-In and consolidated inventory derived from batch quantities.
- Confirmed configurable inventory units per flower, unit conversion, and Stem as the standard base unit.
- Confirmed fixed minimum stock thresholds per flower, configured in Flower Management, for Low Stock alerts.
- Confirmed manual Stock-Out, including spoiled flowers; FIFO default allocation; automatic batch splitting; partial batch consumption; and authorized FIFO override with reason in Remarks.
- Confirmed insufficient-stock validation: block Stock-Out and prohibit negative inventory.
- Confirmed a Remarks field/column for every Stock-In and Stock-Out and Inventory History fields for date/time, flower, batch, movement type, quantity, Remarks, and user.
- Confirmed Stock-In record editing and Remarks as the agreed stock-correction approach; no separate reversal workflow.
- Explicitly deferred Stock Adjustment / Physical Inventory Counting.
- Updated MODULES.md, BUSINESS_LOGIC.md, REQUIREMENTS.md, and DEVELOPMENT_PROGRESS.md. This is a documentation/planning update only; no application code, database schema, or backend behavior was changed.

### Clarified — frontend module file organization
- Established separate Blade, CSS, and JavaScript files as the standard for Supplier modules that have page-specific styling or interactions.
- Recorded the Flower Management example structure: `resources/views/supplier/flowers.blade.php`, `resources/css/supplier/flowers.css`, and `resources/js/supplier/flowers.js`.
- Clarified that `resources/css/app.css` and `resources/js/app.js` remain shared/global Vite entry points, rather than holding all module-specific code.
- Required module assets to be loaded through the existing Vite pipeline and discouraged unnecessary empty files.
- This records the agreed structure only; Flower Management implementation has not started yet.


### 2026-11-02 — Delivery Management requirements

- Documented the approved initial Delivery Management workflow, delivery fields, statuses, assignment/scheduling, Proof of Delivery, failed-delivery handling, and status-only tracking.
- Confirmed a fixed system-configured delivery fee and Client Panel visibility for delivery status and scheduled delivery information; personnel personal contact details remain private.
- No application code, UI, backend, database, or GPS integration changed.
