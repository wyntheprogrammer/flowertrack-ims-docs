# AI Context — FlowerTrack IMS

**Last updated:** 2026-11-02

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

The Bootstrap foundation and initial Supplier/Admin UI were committed and pushed to the application repository. The initial Supplier/Admin and Client starter UI commit was `488a4eb` (`Build initial Supplier and Client UI layouts`). The owner later confirmed the logo update and Bootstrap Icons refinement build/push. Client feature screens and backend/database work remain future work. The project is currently in requirements-planning: Flower Management, Inventory Management, Batch Management, Spoilage Prediction, Client Management, Order Management, Delivery Management, and Sales and Payment Management are approved. Delivery Management includes a fixed system-configured delivery fee and Client Panel tracking of delivery status and scheduled delivery information. Sales and Payment Management includes manual flower pricing, sales recognition on order completion, Cash/Bank Transfer/E-wallet, Admin payment verification, partial/multiple payments, Admin-managed refunds, completed-order sales reporting less refunds, and daily-to-yearly PDF/Excel reports. Reports is the next Supplier/Admin module to plan. Do not begin module UI implementation until the owner confirms the full planning phase is complete.

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
- Available, Partially Used, and Depleted statuses are derived from initial and remaining quantities. Spoiled is marked manually.
- Before marking a batch Spoiled, record its remaining spoiled quantity through a manual Stock-Out with a required spoilage remark.
- Batch Management supports batch details and movement history, search by Batch Number or Flower Name, filters by Flower / Status / Date Received, and metadata-only editing.
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
- Client Management supports listing, adding, viewing, editing, activating/deactivating, searching by Client ID / Full Name / Email, and filtering by Account Status / Registration Date.
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

The user wants to complete requirements planning before proceeding with module UI implementation. The current phase is requirements planning, not UI coding. Flower Management, Inventory Management, Batch Management, Spoilage Prediction, Client Management, Order Management, Delivery Management, Sales and Payment Management, and Reports requirements are approved. Continue through the remaining Supplier/Admin modules in the order in `MODULES.md`, then plan the Client modules. Only after the owner confirms that the planned module requirements have been finalized should module UI implementation proceed. After UI review/finalization, begin backend and database work.

High-level sequence:
1. Environment setup and shared UI foundation (completed baseline).
2. Requirements planning and approval for all Supplier/Admin modules.
3. Requirements planning and approval for Client modules.
4. Supplier/Admin UI implementation in the documented module order.
5. Client ecommerce-style UI implementation.
6. UI review and finalization.
7. Backend and database.
8. Integration, algorithms, testing, and deployment.

Do not jump from an approved module scope directly into its UI while the owner is still planning the remaining modules. Do not begin backend implementation before UI review/finalization.

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
- Still to finalize: detailed role permissions and fixed/configurable permission approach; deactivation/reactivation and session/assigned-work effects; password reset/recovery; account creation notifications.
- No User and Access Management UI, authentication changes, backend, or database schema has been implemented.
- Current planning task: finalize role-specific permissions and account lifecycle rules.
