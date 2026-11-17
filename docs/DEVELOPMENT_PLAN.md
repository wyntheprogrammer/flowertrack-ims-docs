# Development Plan

## Live daily observations and user-activity catch-up - approved checkpoint, 2026-11-17

Steps 2 and 3 use live daily drafts and user-activity catch-up. Stock-In inserts one draft per batch and Philippine date; spoilage, actual pickup/delivery completion and other inventory movements rebuild that same day's values from the ledger inside the inventory transaction. Successful eligible Admin/Supplier login and logout verify missing completed days, including zero-movement days. Today stays a draft. There is no periodic/date-based scheduler; without eligible login/logout, catch-up waits.

Late changes invalidate derived history and rewind the checkpoint; a revision prevents advancing past a concurrent invalidation. Login/logout share the lock, process at most seven pending dates per event by default, and leave failures pending without blocking authentication. Failed, inactive, temporary-password, Client and Delivery accounts do not trigger catch-up. Existing business and prediction-eligibility rules are preserved. The prior Windows task is disabled and its installer/runner scripts are removed. See [Daily Observations](DAILY_OBSERVATIONS.md) for setup and review.

**Approved checkpoint:** [`bb8d9bb`](https://github.com/wyntheprogrammer/flowertrack-ims/commit/bb8d9bbbc989e8069587e78fefc364286c4fe468) is published on application main after Wyn's review/approval. Agent-run full regression: **137 passed / 1,760 assertions**; activity observation suite: **28 passed / 239 assertions**. No periodic observation scheduler remains; the prior local Windows task is disabled. This section supersedes older collector-only and timer-based descriptions below.

**Last updated:** 2026-11-16

## Guiding rule

Preserve Wyn's approved Supplier UI and implement/review backends incrementally. The earlier Supplier UI-first milestone has already been completed; the separate Client storefront remains deferred. Do not interpret the old phase numbering as requiring completed Client UI before continuing the already-approved Supplier backend work.

## Current milestone — 2026-11-16

Supplier operational backends and automatic Spoilage refresh are saved through `46e4145`; full suite 107 passed / 1,439 assertions. Python setup/requirements are saved in `f213583`.

Live daily observations and user-activity catch-up are approved and saved. Next: Dashboard current-risk integration and remaining browser acceptance. No periodic day-closing scheduler is configured. Client storefront remains separately reviewed. See [Daily Observations](DAILY_OBSERVATIONS.md).

The numbered phases below retain the original roadmap. Supplier phases already completed are not new tasks; Client/deployment stages remain uncompleted unless explicitly verified.

## Phase 0 — Requirements and documentation

- Gather business requirements.
- Confirm modules and user roles.
- Clarify flower, batch, inventory, order, delivery, payment, and spoilage rules.
- Maintain documentation and record open questions.

## Phase 1 — Environment setup

- Check the development computer and operating system.
- Check/install a compatible PHP version.
- Install/configure Composer.
- Install/configure Laravel.
- Install/configure Node.js and npm for frontend asset tooling if required by the selected Laravel setup.
- Install/configure MySQL and a database client.
- Configure VS Code and useful extensions.
- Create the separate application repository.
- Run the Laravel starter application locally and verify it works.

Do not assume exact version numbers until compatibility is checked and recorded.

## Phase 2 — Shared UI foundation

- Establish the Laravel view structure for the UI.
- Configure Bootstrap.
- Establish custom CSS organization and design tokens.
- Establish typography, spacing, colors, buttons, forms, tables, and responsive rules.
- Build reusable header, sidebar/navigation, content layout, and feedback components as needed.
- Review and approve the foundation before module screens.

## Phase 3 — Supplier/Admin UI

Build in the exact module order supplied by the owner:
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

For each module, reuse the approved Flower Management card/header, compact toolbar, table, buttons, status, modal and pagination styling, adapting the contents to the module's requirements. See `UI_UX_GUIDELINES.md`.

For each module:
- Confirm screen requirements.
- Sketch/plan the page structure.
- Build the UI.
- Add realistic placeholder/sample content where appropriate, clearly marked as mock data.
- Check responsive behavior and visual consistency.
- Review with the owner.
- Record completion and update documentation first.
- Provide the full application build/check + stage + commit + push script.
- Verify the application push before proceeding to the next module.

## Phase 4 — Client ecommerce UI

Build in this order:
1. Dashboard
2. Flower Catalog
3. Shopping Cart
4. My Orders
5. Order Tracking
6. My Payments
7. My Profile

Confirm whether separate product-detail, login, registration, and checkout screens are needed before adding them to the approved screen list.

## Phase 5 — UI review and finalization

- Review all Supplier/Admin screens.
- Review all Client screens.
- Check desktop and mobile layouts.
- Check navigation and interaction prototypes.
- Fix visual inconsistencies.
- Obtain owner approval that the UI phase is complete.

## Phase 6 — Backend and database

Only begin after the UI phase is approved:
- Finalize database schema from confirmed workflows.
- Implement Laravel models, migrations, controllers, validation, services, and authorization.
- Implement authentication and role/permission rules.
- Implement module business logic.

## Phase 7 — Integration and algorithms

- Connect screens to backend endpoints/forms.
- Implement inventory, batch, order, delivery, sales, and payment workflows.
- Decide and implement spoilage-risk logic based on confirmed data and requirements.
- Add Python only if it is justified by the selected prediction approach.

## Phase 8 — Testing and hardening

- Functional testing.
- Validation and authorization testing.
- Responsive/browser testing.
- Security review.
- Performance checks.
- Backup and recovery checks.

## Phase 9 — Deployment

- Select hosting and production configuration.
- Configure environment variables and secrets securely.
- Deploy and verify.
- Document operations, backups, and maintenance.

## Change control

If the owner changes the order or scope, update this plan, `REQUIREMENTS.md`, `DEVELOPMENT_PROGRESS.md`, and `CHANGELOG.md` with the date and reason.
