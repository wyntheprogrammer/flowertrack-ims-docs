# AI Context — FlowerTrack IMS

**Last updated:** 2026-11-04





## Delivery Management UI — owner-approved 2026-11-04

This is the current milestone and next-task reference. Earlier dated milestone sections are historical.

- Owner approved Delivery Management UI. Preserve Flower Management typography, compact Show Entries/search, native column sorting, pagination and table-only horizontal scrolling; no extra filter row.
- Approved operations: View Details, Schedule/Reschedule with remarks, Assign/Reassign active authorized delivery personnel, Dispatch, Delivered with proof, Failed attempt with reason, Cancel Delivery and timestamped actor/status history.
- No Add Delivery button: future backend integration creates one linked delivery record when a Delivery order reaches Ready for Delivery. Pickup orders do not create deliveries.
- Dispatch requires a Ready for Delivery linked order, schedule and active assigned personnel. Dispatch changes sample delivery/order to Out for Delivery without Stock-Out.
- Delivered requires recipient name, delivery remarks and a readable JPEG/PNG/WebP photo up to 5 MB. Completion time is automatic. The prototype changes its local linked-order status to Completed and marks the sample fulfillment trigger once; no actual Inventory/FIFO deduction or Sales record is created.
- Proof uses a browser-session object URL only: no upload or persistent storage. Data and images reset on refresh.
- Failed attempts remain in history and do not cancel the order. Reschedule before redispatch; reassignment alone does not clear Failed. No fixed attempt cap.
- Cancellation requires a reason, preserves history and does not cancel the linked order or release its reservation. Prototype cancellation is available before dispatch; Out for Delivery exposes Delivered/Failed operations.
- Delivered and Cancelled are terminal/read-only in this prototype. Revision and busy checks guard repeated updates.
- Standalone sample delivery/personnel/order states are not synchronized with the existing Order module. Actual permissions, delivery creation, durable proof uploads, Inventory Stock-Out and order completion coordination remain backend work.
- Files: `resources/views/supplier/deliveries.blade.php`, `resources/css/supplier/deliveries.css`, `resources/js/supplier/deliveries.js`, app CSS/JS imports and `routes/web.php`. Keep `/supplier/deliveries` and `supplier.deliveries`; remove its placeholder mapping.
- Validation: owner approved the local UI. Independent current build/browser checks and Delivery application commit/push have not been verified.
- Current task: build/check, stage, commit and push Delivery Management; verify push success. Next module: Sales & Payments UI. Separate Client storefront and backend/database work remain deferred.


## Order Management UI — owner-approved 2026-11-03

This is an earlier UI milestone; the Delivery Management milestone above is the current next-task reference.

- Owner accepted the supplied Order Management UI after clarification of approval and completion flows.
- Approved screen: Add/Edit orders before Processing, multiple flower items, generated Order ID, registered client selection, Delivery/Pickup, required delivery address, optional notes, price snapshots, calculated subtotal/fee/total, view details, status history and cancellation with a reason.
- Reuse Flower typography, Show Entries, compact search, native column sorting, pagination and table-only horizontal scrolling. No extra filter row.
- Approval is in Order Management: Pending → Confirmed via the next-status arrow and Mark Confirmed. Confirmation checks full sample availability and reserves stock without reducing physical on-hand.
- Pickup flow: Pending → Confirmed → Processing → Ready for Pickup → Completed. Ready for Pickup simulates Stock-Out once; completion confirms collection without another deduction.
- Delivery flow: Pending → Confirmed → Processing → Ready for Delivery → Out for Delivery → Completed. This prototype stops at Ready for Delivery; Delivery Management will own dispatch and Delivered proof. Future successful Delivered coordinates Inventory Stock-Out and Order completion.
- Order Status and Payment Status are separate. Payments are read-only samples; Sales & Payments owns transactions and refunds.
- Reject insufficient full sample allocation; aggregate repeated flower quantities. No partial fulfillment or pending-stock queue. Pending orders do not reserve stock.
- Preserve cancellation reasons/history; release active reservations on cancellation. Completed orders cannot be cancelled. Already-stocked-out orders are blocked from cancellation in this prototype until future Inventory correction integration.
- Prototype boundaries: separate in-memory sample clients, flowers, aggregate stock and orders; reset on refresh. No actual Inventory/FIFO, Delivery record, payment, Sales record, database, authorization or cross-module synchronization.
- Sample fixed delivery fee is PHP 150 for UI review only; actual system configuration remains undecided. Editing an order with sample payments blocks total changes until future financial adjustment integration.
- Files: `resources/views/supplier/orders.blade.php`, `resources/css/supplier/orders.css`, `resources/js/supplier/orders.js`, app CSS/JS imports and `routes/web.php`. Keep `/supplier/orders` and `supplier.orders`; replace the orders placeholder mapping.
- Validation: owner approved local UI; independent production build, browser execution and Order application commit/push have not been verified.
- Current task: build/check, stage, commit and push Order Management; verify push. Next module: Delivery Management UI. Separate Client storefront and backend/database work remain deferred.


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
## Current development state — 2026-11-03

- **Phase:** Supplier/Admin UI implementation.
- **Approved UI modules:** Flower Management, Inventory Management, Batch Management, Spoilage Prediction, Supplier Client Management and Order Management (2026-11-03); Delivery Management (2026-11-04).
- **Visual reference:** Keep the approved Flower Management module design and shared Supplier dashboard scheme.
- **Current task:** Build/check, commit and push the approved Delivery Management UI, then verify the application push.
- **Next module:** Sales & Payments UI after Delivery Management application push verification.
- **Batch UI decisions:** View and metadata Edit actions only; flower/status/date filters and the separate Mark as Spoiled action are removed. Keep compact Show Entries/search, native header sorting, and table-only horizontal scrolling with results/pagination outside the scroller.
- **Spoilage ownership:** Inventory records the human-confirmed spoilage Stock-Out with required remarks. In backend integration, when it consumes a batch's entire remaining quantity, that transaction also sets the batch status to Spoiled. Partial spoilage leaves the usable remainder; Batch Management does not perform another deduction or separate marking action.
- **Prototype boundary:** Batch and Inventory currently use separate in-memory sample data. UI approval does not establish cross-module/backend synchronization or database persistence.
- **Checks:** Batch JavaScript syntax and isolated metadata/spoilage-rule checks passed; owner approved the local UI. Full browser validation and the final local production build were not independently run here.
- **Application Git:** Flower commit `3f580e5` and Inventory commit `e758bf83be6b795cafbe85a458edd9c1db8e89c1` are verified on GitHub. Batch Management application commit `4472a2a8f471b81ec91be03db11bb67e3f8d861f` (`Complete approved Batch Management UI`) is verified on GitHub. Navigation application push remains unverified.

This current state supersedes earlier dated next-task and manual-marking instructions below.


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
## Supplier navigation — accepted 2026-11-03

The owner accepted the shared sidebar link and route setup. Keep `Route::prefix('supplier')->name('supplier.')->group(...)` in `routes/web.php`; the root route continues to show `welcome`.

| Module | URL | Named route | View |
|---|---|---|---|
| Dashboard | /supplier/dashboard | supplier.dashboard | supplier.dashboard |
| Flower Management | /supplier/flowers | supplier.flowers | supplier.flowers |
| Inventory Management | /supplier/inventory | supplier.inventory | supplier.inventory |
| Batch Management | /supplier/batches | supplier.batches | supplier.batches |
| Spoilage Prediction | /supplier/spoilage-prediction | supplier.spoilage-prediction | supplier.spoilage-prediction |
| Client Management | /supplier/clients | supplier.clients | supplier.clients |
| Order Management | /supplier/orders | supplier.orders | supplier.orders |
| Delivery Management | /supplier/deliveries | supplier.deliveries | supplier.deliveries |
| Sales & Payments | /supplier/sales-payments | supplier.sales-payments | supplier.module-placeholder |
| Reports | /supplier/reports | supplier.reports | supplier.module-placeholder |
| User Management | /supplier/users | supplier.users | supplier.module-placeholder |
| Audit Trail | /supplier/audit-trail | supplier.audit-trail | supplier.module-placeholder |

- Shared layout: `resources/views/layouts/supplier.blade.php`. Generate links from module key, label and Bootstrap icon; preserve module order and branding.
- Active state: `request()->routeIs($routeName, $routeName . '.*')`; add `aria-current="page"` only to the active link.
- Upcoming routes use `$upcomingModules` and pass `moduleTitle` to `resources/views/supplier/module-placeholder.blade.php`. It extends the Supplier layout and shows a Coming Soon card with a Dashboard link.
- When implementing a module, remove its placeholder mapping and add an explicit route to its actual view using the same URL and route name. Avoid duplicate definitions.
- This is navigation scaffolding. Upcoming modules, authorization and backend functionality are not implemented by placeholder routes.
- Owner accepted the supplied setup; navigation runtime checks and application commit/push have not been independently verified.
- Batch application push is verified: `4472a2a8f471b81ec91be03db11bb67e3f8d861f` — `Complete approved Batch Management UI`.
- Next: verify/commit/push navigation changes, then Spoilage Prediction UI.

## Role and working style

Act as a patient senior software engineer, system analyst, and UI/UX design mentor. Teach the project owner step by step. Explain commands, code, design choices, and technical concepts in straightforward language. Do not simply dump a complete application without teaching the implementation.

## Project identity

- Project name: **FlowerTrack IMS**
- Documentation repository: `flowertrack-ims-docs`
- Application repository: [`wyntheprogrammer/flowertrack-ims`](https://github.com/wyntheprogrammer/flowertrack-ims) (public; initialized and pushed to `main`)
- Application type: Web application
- Main interfaces: Supplier/Admin Panel and Client Panel
- Supplier/Admin and Client must have distinct user experiences.

## Confirmed technology direction

- Laravel and PHP for server-side application development.
- MySQL for the relational database.
- HTML for page structure.
- JavaScript for browser-side interactions.
- Bootstrap for UI components and responsive layout.
- Flexbox for layout and alignment when appropriate.
- Custom CSS for brand colors, typography, sizes, spacing, dimensions, and visual refinements.
- Bootstrap Icons for interface icons across Supplier/Admin and Client panels; use the package through the Vite CSS pipeline.
- Python is being considered for spoilage prediction; its final role and implementation are not yet decided.

Do not interpret the user's earlier mention of Java as a requirement for Java programming. The listed frontend category and intended use refer to **JavaScript**.

## Local development environment — installed and verified

The owner has completed the initial local setup on Windows:

| Component | Installed / selected version | Location or verification |
|---|---|---|
| XAMPP | Installed; exact XAMPP package version not recorded | `C:\xampp` |
| PHP | 8.3.36 | `C:\xampp\php\php.exe`; `php -v` verified |
| Composer | 2.11.1 | `composer -V` verified |
| Laravel application skeleton | 13.11.0 | Installed in `C:\xampp\htdocs\FlowerTrack` |
| MariaDB bundled with XAMPP | 10.11.16 | Version shown in XAMPP MariaDB startup log |
| Visual Studio Code | Installed and project opened | `code .` from project directory |
| Node.js | 26.13.0 | `node -v` verified |
| npm | 11.20.0 | `npm -v` verified |
| Git | 2.52.0.windows.1 | `git --version` verified; global identity configured as `wyn` / `wyntheprogrammer@gmail.com` |
| PHP ZIP extension | Enabled | `php -m` lists `zip` |
| Required Laravel PHP extensions | Verified present | Includes BCMath, Ctype, cURL, DOM, Fileinfo, Mbstring, OpenSSL, PDO, pdo_mysql, Tokenizer, and XML |

XAMPP's local dashboard/phpMyAdmin became accessible after resolving port conflicts with pre-existing MySQL and Apache processes. Do not assume the old services were uninstalled; only their conflicts were resolved during setup. The Laravel development server was started with `php artisan serve`, and the default Laravel welcome page at `http://127.0.0.1:8000` was confirmed working.

Database connectivity from Laravel will be tested when database work begins; the UI-first phase does not require database configuration yet.

## Bootstrap and Vite foundation — locally verified

The owner completed and browser-tested the initial Bootstrap integration locally on 2026-11-01:

- Installed bootstrap@5.3.8 and @popperjs/core@2.11.8 through npm.
- Installed the `bootstrap-icons` npm package; exact package version has not been recorded.
- Removed the Tailwind packages tailwindcss and @tailwindcss/vite.
- Updated vite.config.js to use the Laravel Vite plugin without the Tailwind Vite plugin.
- Updated resources/css/app.css to import Bootstrap CSS and `bootstrap-icons/font/bootstrap-icons.css`.
- Updated resources/js/app.js to import Bootstrap JavaScript.
- Ran npm run build successfully; Vite generated the production manifest and CSS/JS assets.
- Replaced the Laravel starter welcome view with a simple FlowerTrack IMS Bootstrap verification page.
- Confirmed in the browser at http://127.0.0.1:8000 that the page displays the FlowerTrack IMS heading and Bootstrap 5.3.8 Ready badge.

The Bootstrap foundation and initial Supplier/Admin UI were committed and pushed to the application repository. The initial Supplier/Admin and Client starter UI commit was `488a4eb` (`Build initial Supplier and Client UI layouts`). The owner later confirmed the logo update and Bootstrap Icons refinement build/push. Client feature screens and backend/database work remain future work. Historical requirements-planning snapshot (superseded by the current development state above): Flower Management, Inventory Management, Batch Management, Spoilage Prediction, Client Management, Order Management, Delivery Management, and Sales and Payment Management are approved. Delivery Management includes a fixed system-configured delivery fee and Client Panel tracking of delivery status and scheduled delivery information. Sales and Payment Management includes manual flower pricing, sales recognition on order completion, Cash/Bank Transfer/E-wallet, Admin payment verification, partial/multiple payments, Admin-managed refunds, completed-order sales reporting less refunds, and daily-to-yearly PDF/Excel reports. Reports is the next Supplier/Admin module to plan. Do not begin module UI implementation until the owner confirms the full planning phase is complete.

## Verified application repository structure

The following snapshot was inspected directly from the public application repository's `main` branch on 2026-11-01 (initial tree commit `ca10822eeea300e23042c5910697f8bc86210e9b`):

```text
flowertrack-ims/
├── app/
│   ├── Http/
│   │   └── Controllers/
│   │       └── Controller.php
│   ├── Models/
│   │   └── User.php
│   └── Providers/
│       └── AppServiceProvider.php
├── bootstrap/
│   ├── app.php
│   ├── cache/
│   └── providers.php
├── config/
│   ├── app.php
│   ├── auth.php
│   ├── cache.php
│   ├── database.php
│   ├── filesystems.php
│   ├── logging.php
│   ├── mail.php
│   ├── queue.php
│   ├── services.php
│   └── session.php
├── database/
│   ├── factories/
│   │   └── UserFactory.php
│   ├── migrations/
│   │   ├── 0001_01_01_000000_create_users_table.php
│   │   ├── 0001_01_01_000001_create_cache_table.php
│   │   └── 0001_01_01_000002_create_jobs_table.php
│   └── seeders/
│       └── DatabaseSeeder.php
├── public/
├── resources/
│   ├── css/
│   │   └── app.css
│   ├── js/
│   │   └── app.js
│   └── views/
│       └── welcome.blade.php
├── routes/
│   ├── console.php
│   └── web.php
├── storage/
├── tests/
│   ├── Feature/
│   │   └── ExampleTest.php
│   ├── Unit/
│   │   └── ExampleTest.php
│   └── TestCase.php
├── .editorconfig
├── .env.example
├── .gitattributes
├── .gitignore
├── .npmrc
├── AGENTS.md
├── CLAUDE.md
├── artisan
├── composer.json
├── composer.lock
├── package.json
├── phpunit.xml
├── vite.config.js
└── README.md
```

Current UI-relevant baseline:
- `resources/views/welcome.blade.php` is now a simple local Bootstrap verification page; reusable application layouts and feature screens have not yet been created.
- `resources/css/app.css` imports Bootstrap CSS in the owner's locally updated working copy; the public GitHub snapshot may still show the earlier starter version until pushed.
- `resources/js/app.js` imports Bootstrap in the owner's locally updated working copy; verify GitHub before assuming the change has been pushed.
- `routes/web.php` currently maps `/` to the `welcome` view.
- `app/Providers/AppServiceProvider.php` is still the default empty provider.
- No custom Supplier/Admin or Client views/layouts have been created yet.
- This is a snapshot of the inspected `main` branch, not a permanent inventory; re-check GitHub when the structure may have changed. The Bootstrap changes above are confirmed locally, not confirmed as pushed to GitHub.

### Repository inspection rule for AI assistance

Before asking the owner to paste or list application folders/files:
1. Inspect the latest application repository tree on GitHub (`main`, unless the owner specifies another branch).
2. Fetch the relevant source file directly from GitHub when its contents are needed.
3. Use the verified repository structure and the documentation as the baseline; do not repeatedly ask the owner to provide structure that can be inspected from the repository.
4. Ask the owner to paste code or output only when the relevant content is not available in the repository, local uncommitted changes may matter, or a command/test must be run on the owner's machine.
5. Clearly distinguish the committed GitHub version from any local changes that have not been pushed.

## Current UI implementation — Supplier/Admin dashboard (2026-11-01)

The owner has completed the first Supplier/Admin dashboard UI pass locally in `C:\\xampp\\htdocs\\FlowerTrack`. The following implementation details are confirmed from the working session:

- Created separate Blade layouts and views under `resources/views/layouts/` and `resources/views/supplier/`, including `supplier.blade.php` and `dashboard.blade.php`.
- Supplier sidebar uses Gemma's Flower Shop logo as the business/client branding; the logo is displayed without a white background.
- FlowerTrack IMS and the subtitle “Inventory Management System” were moved to the top navbar so the client business logo and application identity are visually distinct.
- Removed the navbar search field at the owner's request.
- Refined sidebar navigation spacing to be more compact.
- Kept the existing dashboard content/design after the owner requested reverting an alternative dashboard redesign; do not replace the current dashboard with the discarded redesign.
- Added two owner-supplied local image assets under `public/images/`: `flower-banner.jpg` for the welcome banner and `bloom-business.jpg` for the “Bloom Your Business” promotional panel. These are existing supplied images, not generated assets.
- The welcome banner uses the flower image on the right side, with the welcome text on the left; the image should preserve its aspect ratio and must not be stretched.
- Inventory Overview and Quick Actions panel spacing was adjusted to reduce excess empty space under Quick Actions.
- The owner confirmed the current dashboard design is acceptable as of 2026-11-01.

Relevant local files include `resources/css/app.css`, `resources/views/layouts/supplier.blade.php`, `resources/views/supplier/dashboard.blade.php`, and the two image assets in `public/images/`. These changes are local and must not be described as pushed until the application repository is checked and a push is confirmed.

### UI implementation rules learned

- Preserve approved UI and make targeted changes only; do not redesign unrelated dashboard sections when the owner asks to adjust one component.
- Keep Gemma's Flower Shop (business identity) separate from FlowerTrack IMS (application identity).
- Use the user's supplied images directly when requested; do not generate replacement images.
- When positioning banner imagery, preserve the original image proportions and use CSS background sizing/positioning intentionally.


### Bootstrap Icons standard — 2026-11-01

- **Confirmed icon standard:** Use Bootstrap Icons for interface/navigation icons throughout FlowerTrack IMS, including Supplier/Admin sidebar, Client sidebar, navbar, dashboard cards, panel headings, quick actions, and activity indicators.
- Installed the `bootstrap-icons` npm package and imported `bootstrap-icons/font/bootstrap-icons.css` in `resources/css/app.css` after the Bootstrap CSS import.
- Use the standard Bootstrap Icons markup pattern: `<i class="bi bi-icon-name"></i>`, typically inside the existing icon wrapper span so current layout styling is preserved.
- Do not use Unicode symbols or emoji as substitutes for interface icons. Emoji or images may remain when they are actual product imagery/content (for example, flower illustrations in the Top Selling Products list), not interface controls.
- The Supplier sidebar and Supplier Dashboard use Bootstrap Icons. The owner confirmed the Bootstrap Icons refinement build and application-repository push. Preserve the approved dashboard layout, colors, chart, sample values, and spacing; only change the icon presentation unless the owner requests a broader redesign.

## Batch Management — approved requirements (2026-11-02)

- Every Stock-In automatically creates one batch with a unique, automatically generated Batch Number.
- Batch information: linked Flower, Date Received, Initial Quantity, Remaining Quantity, required Harvest Date, optional Expected Freshness / Expiry Date, and optional Remarks.
- Do not include Supplier / Source; the business sources flowers directly.
- Remaining Quantity is controlled by Inventory Management Stock-Out allocations; Batch Management cannot directly edit quantities.
- Available, Partially Used and Depleted derive from quantities. Spoiled is set by the integrated Inventory spoilage Stock-Out that consumes the batch's remaining quantity; no separate Batch marking action (updated 2026-11-03).
- Inventory records spoilage Stock-Out with required remarks. A full remaining-quantity spoilage Stock-Out also updates batch status to Spoiled in the same backend transaction; partial spoilage retains the usable remainder. Do not deduct stock again in Batch Management.
- Batch Management supports details/history, Batch Number/Flower search, pagination, native sorting and metadata-only editing. The Flower/Status/Date filter row and Mark as Spoiled button are removed from the approved UI.
- Retain depleted and spoiled batch records for traceability.
- Approaching freshness/expiry dates produce warnings only; they do not automatically deduct stock or mark a batch Spoiled.
- Spoilage Prediction does not automatically change batch status or deduct stock.
- Client Management requirements are now approved; Order Management is the next module to plan. Do not start module UI implementation until the owner confirms requirements planning is complete.

## Spoilage Prediction — approved requirements (2026-11-02)

- Use Machine Learning (ML) to estimate spoilage risk for flower batches.
- Confirmed date inputs: Harvest Date and optional Expected Freshness / Expiry Date. Handle batches without an expiry/freshness date using supported available data.
- Four risk levels: Low, Medium, High, Critical. Algorithm and numerical thresholds are not selected until dataset assessment and model validation.
- Show risk summaries/alerts on the Supplier/Admin Dashboard and provide a dedicated prediction page with batch risk details, search, and filters.
- Provide advisory warnings and recommendations, including prioritizing high-risk batches for Stock-Out.
- Predictions must not automatically change batch status, deduct inventory, alter FIFO allocation, or create Stock-Out transactions. Actual spoilage remains manual.
- Only batches with remaining usable inventory are active prediction targets.
- Record actual Spoilage Events linked to Flower and Batch with Spoilage Date, Spoiled Quantity, and Spoilage Reason. These are labeled historical outcomes for future ML training and evaluation.
- Do not claim prediction accuracy until sufficient representative data is available and the model has been tested and validated.
- Historical risk-assessment snapshots and manual condition-observation scoring are excluded from the initial scope.
- Client Management is approved. The next planning module is Order Management. Requirements planning remains active; no module UI implementation until the owner confirms planning is complete.

## Client Management — approved requirements (2026-11-02)

- Only Supplier/Admin users create client accounts manually; public/self-service client registration is excluded from the initial scope.
- Client ID is automatically generated. Full Name, Email Address, Contact Number, and Address are required. Registration Date is recorded automatically; Account Status is Active or Inactive.
- Email Address must be unique per client account. No email verification is required.
- Admin assigns a temporary password when creating the account. The client must change it on first login. Passwords must be securely hashed and never stored as plain text.
- Inactive clients cannot place new orders; existing orders and transaction history remain accessible. Reactivating the account restores eligibility to place new orders.
- Client records with transaction history cannot be permanently deleted.
- Client Management supports listing, adding, viewing, editing, activating/deactivating, searching by Client ID / Full Name / Email, with Show Entries, native column sorting and pagination. Account Status / Registration Date filters and Reset were removed by owner decision on 2026-11-03.
- Provide Order History and Purchase Summary. Derive summary metrics from Order Management records: total order count, total purchase amount, completed orders, and pending/ongoing orders.
- Client Management owns client information and account status; Order Management owns order creation, processing, and statuses.
- No Client Management UI, authentication implementation, backend, or database schema has been implemented by this planning update.

## Order Management — approved requirements (2026-11-02)

- Orders can be created through the Client Panel or manually by Supplier/Admin; Admin manually reviews and confirms.
- Generated unique Order ID; registered Client; automatic Order Date; at least one item; each item has a Flower and positive quantity.
- Unit Price comes from Sales and Payment Management and is snapshotted per order item; subtotals and order total are calculated. Delivery Address is required for Delivery; Order Notes are optional.
- Separate Order Status and Payment Status. Order statuses: Pending, Confirmed, Processing, Ready for Delivery or Ready for Pickup, Out for Delivery (Delivery only), Completed, Cancelled. Payment statuses: Unpaid, Partially Paid, Paid, Refunded.
- Sales and Payment Management owns payment recording/transactions; Order Management displays payment status and summary only.
- Check available-to-allocate stock before confirmation, accounting for reservations already held by other confirmed orders. Reject an order if the full quantity is unavailable; no pending-stock queue or overselling.
- Reserve stock when Confirmed. Reservation does not reduce physical on-hand quantity and is not a Stock-Out.
- Actual Stock-Out triggers: Delivery when successfully marked Delivered; Customer Pickup when marked Ready for Pickup. Coordinate reservation and Stock-Out to prevent double deduction. No partial fulfillment.
- Clients may edit/cancel Pending orders. Admin may edit before Processing and cancel later non-completed orders with a required reason. Completed orders cannot be cancelled.
- Cancellation releases reservations. If Stock-Out occurred, inventory must be corrected/restored before cancellation is finalized. Retain cancelled orders and status history.
- Support Delivery and Customer Pickup. Delivery Management owns assignment and delivery-specific tracking; Pickup skips Out for Delivery.
- No pending-stock queue: reject an order when available-to-allocate stock after existing reservations is insufficient.
- No Order Management UI, backend, database schema, or transaction code has been implemented.

## UI-first rule

Supplier requirements planning is complete. Flower Management, Inventory Management and Batch Management UI are owner-approved. Flower Management remains the module-page reference. Verify the Batch application build/commit/push before starting Spoilage Prediction UI. Finish Supplier UI before Client UI; full UI review precedes backend/database work. Implement one focused step at a time.

## UI design rules

- Supplier/Admin Panel is a business-management interface.
- Client Panel must look and behave like an online flower shop.
- Use Bootstrap as the primary component/layout framework.
- Use Flexbox for custom alignment and layout.
- Use custom CSS for colors, font sizes, spacing, sizing, and brand-specific styling.
- Prefer reusable components and consistent patterns.
- Build one screen or component at a time, explain it, and review it before moving on.
- Do not invent or alter the official FlowerTrack IMS logo.
- The uploaded flower-shop dashboard screenshot is an approved **visual reference for the Supplier Side only**. Follow the visual direction and design prompt in `SUPPLIER_UI_DESIGN_REFERENCE.md`.
- Do not copy the reference business name/logo or treat that screenshot as the official FlowerTrack IMS logo. If the official FlowerTrack logo is supplied separately, use it as the primary brand reference.
- Keep the Client Panel distinct and ecommerce-oriented.

## Learning and implementation rules

1. Start each task by stating the goal and expected result.
2. Give the user the exact next action or code change.
3. Explain important code and commands.
4. Avoid skipping setup steps.
5. Do not silently add features, libraries, or architecture.
6. Ask for a decision when a business rule is unknown and materially affects implementation.
7. Mark assumptions clearly; do not present them as confirmed requirements.
8. Test each meaningful UI milestone before proceeding.
9. Keep the documentation synchronized with actual decisions and completed work; after every completed module, update the relevant Markdown before providing the application Git push script.
10. Work in small steps. Give only the current implementation step, wait for the owner's result, and then continue. Do not dump the entire setup or implementation plan as a sequence of actions to execute at once. At module completion, however, provide one complete, ready-to-run Git script for that module's build/check, staging, commit, push, and final status verification.

## Per-module completion, documentation, and Git workflow

This is the required workflow after **every completed module or independently reviewable feature**. Follow it consistently; do not batch several completed modules before documenting or pushing them.

1. **Complete and review the module.** Finish the requested UI/feature, preserve approved designs, and let the owner review it. Run the relevant build, test, or manual checks.
2. **Update documentation immediately after approval.** Update the relevant Markdown files to record what was completed, important decisions, files/areas changed, checks performed, remaining issues, and the next exact task. Update `DEVELOPMENT_PROGRESS.md` for progress/status changes and add a dated entry to `CHANGELOG.md`. Update `AI_CONTEXT.md`, `UI_UX_GUIDELINES.md`, or other requirement/design documents when the module establishes a reusable rule or changes a confirmed decision. Keep `README.md` current when the project status or workflow summary changes.
3. **Provide the complete application Git script.** Give the owner the exact commands to run from `C:\xampp\htdocs\FlowerTrack`: relevant build/test command(s), `git status`, targeted `git add` for the completed module, another `git status` review, a descriptive `git commit`, `git push origin main`, and a final `git status`. Use `git add -A` only when its scope is safe and ignored files have been verified. Never stage `.env`, `vendor/`, or `node_modules/`.
4. **Wait for the owner's terminal output and verify.** Inspect staged files before commit when needed. Do not claim commit/push success until the owner provides successful output or the remote state is independently verified. If build/tests fail, stop the Git sequence and resolve the issue first.
5. **Proceed only after confirmation.** Once the application push is confirmed and the final working tree is clean (or remaining changes are understood), identify the next module from the documented module order and begin it one step at a time.

The documentation repository and application repository are separate. Update and push the documentation repository when documenting the approved module, then provide the owner the application-repository script for the code changes. Keep both repository statuses distinct and never imply that updating one repository pushed the other.

## Application Git repository

- Local project: `C:\xampp\htdocs\FlowerTrack`.
- Git repository initialized locally; current branch: `main`.
- Initial commit: `ca10822` (`Initial Laravel project setup`).
- Remote `origin`: `https://github.com/wyntheprogrammer/flowertrack-ims.git`.
- Initial push succeeded; local `main` tracks `origin/main`.
- `.env`, `vendor`, and `node_modules` were not included in the initial commit based on the staged-file review.
- Git global identity configured by the owner: `wyn` / `wyntheprogrammer@gmail.com`.

## GitHub documentation update rule

Whenever a meaningful milestone is completed, a new technical concept is learned, or a project decision/requirement changes:
- Update the relevant Markdown document(s).
- Update `DEVELOPMENT_PROGRESS.md` only when project progress or status changes.
- Add an entry to `CHANGELOG.md`.
- Set the **Last updated** date using `YYYY-MM-DD`.
- Push the documentation changes to `flowertrack-ims-docs` after the user authorizes the push and repository access is available.

Never claim a GitHub push succeeded unless the host confirms it. If direct repository access is unavailable, prepare the changed files and provide the user with clear commit/push instructions.

## Continuity protocol

At the start of each resumed work session:
1. Read this file and `DEVELOPMENT_PROGRESS.md`.
2. Check the latest requirements and relevant module/design documents.
3. Identify the current phase and next module from the planning status.
4. Continue requirements planning until the owner explicitly confirms that planning is complete; do not assume an approved module is a signal to start its UI.

At the end of each session, record:
- What was completed.
- What was learned or decided.
- Files changed.
- Tests or checks performed.
- Remaining issues.
- The next exact task.

## Accuracy rule

Only treat items marked **Confirmed** in the documentation as approved requirements. Items marked **To decide**, **Proposed**, or **Assumption** must not be implemented as settled business rules.


## Frontend file organization standard — 2026-11-02

For new Supplier/Admin feature modules, keep the Blade view, module-specific CSS, and module-specific JavaScript in separate files when the module has its own styling or browser interactions. Do not combine a module's full CSS and JavaScript into its Blade view.

Recommended pattern for Flower Management:

```text
resources/
├── css/
│   ├── app.css
│   └── supplier/
│       └── flowers.css
├── js/
│   ├── app.js
│   └── supplier/
│       └── flowers.js
└── views/
    └── supplier/
        └── flowers.blade.php
```

- `resources/views/supplier/flowers.blade.php`: Blade markup and Laravel view directives; keep page-specific CSS/JS out of this file.
- `resources/css/supplier/flowers.css`: Flower Management-specific styles.
- `resources/js/supplier/flowers.js`: Flower Management interactions and behavior.
- `resources/css/app.css` and `resources/js/app.js`: shared/global entry points and shared imports or initialization, not containers for every module's page-specific code.
- Load module assets through the existing Vite pipeline. Ensure the relevant CSS and JavaScript are included by the page/layout using the project's established Vite conventions.
- Apply the same organization to later Supplier modules when they have module-specific styles or interactions. Avoid creating empty or unnecessary files for modules that do not need them; shared styles and scripts should remain shared.
- This is an agreed frontend organization standard. It does not imply that Flower Management has already been implemented or tested.


## Delivery Management — approved initial requirements (2026-11-02)

- Delivery records are created automatically when a Delivery order reaches Ready for Delivery; Customer Pickup does not create a delivery.
- Each delivery has a unique Delivery ID and links to one order. Retain an address snapshot, client/order reference, assigned personnel, schedule, status, optional notes, and delivery history.
- Admin/Supplier and dedicated Delivery Personnel accounts participate. Admin manually assigns/reassigns active, authorized personnel and sets delivery date/time.
- Delivery statuses: Pending, Scheduled, Assigned, Out for Delivery, Delivered, Failed, Rescheduled, Cancelled. Keep delivery status separate from Order Status.
- Dispatch requires the related order to be Ready for Delivery. Preserve assignment and status history with timestamp and responsible user.
- Proof of Delivery requires recipient name, delivery remarks, photo proof, and completion date/time.
- Failed attempts require a reason, remain in history, and do not automatically cancel the order. Admin handles rescheduling/reassignment. No fixed attempt limit initially; further attempts are subject to Admin decision.
- Initial tracking is status updates only; no GPS or continuous location tracking.
- Delivery fee policy and Client Panel visibility are still To decide. Do not invent or implement either as a confirmed rule.
- Apply a fixed delivery fee configured in system settings to eligible Delivery orders; exact fee amount/configuration controls are deferred to detailed design.
- Clients can view delivery status and scheduled delivery information. Do not expose delivery personnel's personal contact details.
- No Delivery Management UI, backend, database schema, or GPS integration has been implemented.



## Sales and Payment Management — approved requirements (2026-11-02)

- Supplier/Admin manually sets selling prices for each flower in Sales and Payment Management; price changes affect new orders only. Existing order item prices remain immutable snapshots.
- Automatically create one unique Sales Record when an order becomes Completed. Link it to the Order ID and client, preserve item/price/total snapshots and applicable delivery fee, and prevent duplicate records.
- Sales reporting counts completed orders less recorded refunds. Partial payments are not additional sales; cancelled orders are excluded.
- Supported payment methods: Cash, Bank Transfer, and E-wallet. Require reference numbers for applicable online payments. Admin manually verifies online payments and records/confirms cash payments.
- Allow partial and multiple payments per order. Track Unpaid, Partially Paid, Paid, and Refunded; retain payment history and outstanding balances. Admin records and approves refunds.
- Provide daily, weekly, monthly, and yearly summaries with PDF and Excel export.
- Sales and Payment Management owns prices, Sales Records, payment transactions, and refunds; Order Management displays payment status and summary only.
- No Sales and Payment Management UI, backend, database schema, payment gateway integration, or report generator has been implemented. Reports is the next planning module.


## Reports — approved requirements (2026-11-02)

- Seven categories: Sales, Payments, Inventory, Orders, Delivery, Clients, and Spoilage.
- Filters: Daily, Weekly, Monthly, Yearly, and Custom Date Range; category-appropriate search and filters.
- Presentation: summary cards, charts, detailed tables, and print-friendly layout.
- Output: PDF export, Excel export, and direct printing.
- Access: Supplier/Admin only.
- Reports is read-only, uses source records owned by other modules, and must not create or modify transactions.
- Sales reporting counts completed orders less recorded refunds; partial payments are not separate sales; cancelled orders are excluded.
- Inventory reporting distinguishes physical batch quantities from reservations; reservation is not Stock-Out.
- Spoilage reporting uses actual recorded Spoilage Events, not prediction risk alone.
- Exclude scheduled email delivery, custom report builder, automatic sharing, and predictive business analytics from the initial scope.
- No Reports UI, backend, database schema, report generator, exports, or print implementation has been created.
- Next Supplier/Admin module to plan: User and Access Management.


## User and Access Management — approved initial account setup (2026-11-02)

- Three initial roles: Admin, Supplier/Staff, and Delivery Personnel.
- Admin has full system access, including User and Access Management and Reports.
- Supplier/Staff can access authorized operational modules; Delivery Personnel can access delivery-work functions in Delivery Management.
- Only Admin can create Supplier/Staff and Delivery Personnel accounts; no public/self-service registration.
- Required account fields: Full Name, Email Address, Contact Number, and Role.
- User ID is automatically generated and unique; Email Address must be unique across user accounts.
- Admin assigns a temporary password; first login requires the account holder to change it before continuing.
- Passwords must be securely hashed and never stored as plain text.
- Account Status: Active or Inactive.
- Fixed role permissions and account lifecycle rules are approved; see the User and Access Management section in `REQUIREMENTS.md` and `BUSINESS_LOGIC.md`.
- Account-creation email notifications are excluded from the initial scope.
- No User and Access Management UI, authentication changes, backend authorization, session invalidation, or database schema has been implemented.
- Next Supplier/Admin planning module: Audit Trail.


## Inventory Management UI — owner-approved (2026-11-03)

The owner approved the Inventory Management UI after reviewing its screens and the meanings of its stock fields. Keep the approved Flower Management visual pattern for this and later Supplier modules.

### Approved prototype screens and interactions

- Overview cards for On-Hand, Reserved, Available and Stock Alerts.
- Inventory List and Inventory History tabs with compact Show Entries/search controls, native DataTables header sorting and pagination; Actions remains unsortable.
- Stock-In modal with generated Batch Number, flower, quantity/unit conversion to Stems, Date Received, required Harvest Date, optional freshness/expiry date and Remarks.
- Stock-Out modal with FIFO allocation preview, automatic multi-batch splitting, insufficient-stock prevention, spoilage remarks and an authorized batch-override prototype.
- Customer-order Stock-Out sample flow honors Delivered for Delivery and Ready for Pickup for Pickup, issues the full quantity and prevents repeating the same sample order Stock-Out.
- Batch Details with quantities, dates, statuses and movement history; its Receipt edit action opens Edit Stock-In.
- Per-flower Bunch/Bundle/Box conversion settings; Stem remains the base unit and existing batch conversion snapshots are preserved.
- Separate `resources/views/supplier/inventory.blade.php`, `resources/css/supplier/inventory.css` and `resources/js/supplier/inventory.js`; assets use the existing app imports/Vite pipeline, with the Inventory route and sidebar link.

### Stock terminology and ownership clarified

| Term | Meaning |
| --- | --- |
| On-Hand | Physical stock remaining in usable batches, including reserved quantities |
| Reserved | Stock held for confirmed orders and not yet stocked out |
| Available | On-Hand minus Reserved |
| Initial | Recorded quantity received into a batch; corrected when its Stock-In is corrected |
| Remaining | Batch quantity after actual Stock-Out; reservations alone do not reduce it |
| Receipt action | Edits the Stock-In that created the batch; not a payment receipt |
| Minimum Stock | Configured in Flower Management; Inventory displays/uses it for low-stock monitoring |

Stock-In correction requires explanatory Remarks. The prototype prevents reducing a receipt below quantities already issued or reducing on-hand stock below existing reservations. No separate reversal or physical-count/stock-adjustment screen was added.

### Implementation and next task

This approval covers the local UI prototype. Sample data is held separately from Flower Management and resets on refresh; common persisted flower/minimum-stock values, authorization and server-side transaction rules remain for backend integration. The documentation update does not claim a successful Inventory application push. Verify that build/commit/push before starting Batch Management UI.


## Batch Management UI — owner-approved (2026-11-03)

- Owner approved the Batch Management local UI using Flower Management as the visual reference.
- Approved Available / Partially Used / Depleted / Spoiled overview cards, batch listing, freshness warnings, details/history and metadata-only editing.
- Retained compact Show Entries and search controls, native DataTables header sorting and pagination; Actions is unsortable.
- Removed the Flower, Status, Received From/To and Reset filter row.
- Horizontal scrolling belongs to the table only. Keep the card header, overview, toolbar, results text and pagination outside the horizontal scroller.
- Removed the separate Mark as Spoiled action. Inventory spoilage Stock-Out owns the future integrated status update; see the updated workflow below.
- Metadata editing cannot change initial/remaining quantities, linked flower, receiving records or inventory movements.
- Source files: `resources/views/supplier/batches.blade.php`, `resources/css/supplier/batches.css`, `resources/js/supplier/batches.js`; app CSS/JS imports, route and sidebar provide integration.
- Sample data is temporary and separate from Inventory. The three-day approaching-freshness window is a UI demonstration setting, not a finalized business or ML threshold.
- JavaScript syntax and isolated batch-rule checks passed. The owner approved the local UI; the final local build/full browser checks and application push were not independently confirmed here.
- Next: verify Batch Management application build/commit/push, then start Spoilage Prediction UI. Keep Client UI/backend work deferred according to the UI-first sequence.

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
