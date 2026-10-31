# Development Plan

**Last updated:** 2026-11-01

## Guiding rule

**Finish and review the UI before beginning backend implementation.**

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

For each module:
- Confirm screen requirements.
- Sketch/plan the page structure.
- Build the UI.
- Add realistic placeholder/sample content where appropriate, clearly marked as mock data.
- Check responsive behavior and visual consistency.
- Review with the owner.
- Record completion and update documentation.

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
