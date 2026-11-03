# UI/UX Guidelines

**Last updated:** 2026-11-03

## 1. Overall design direction

- Platform: web application.
- Target: desktop-first and responsive.
- Build the UI first and complete/review it before backend development.
- Supplier/Admin and Client must have clearly different user experiences.
- Aim for a polished, professional, usable commercial product.
- Prioritize clarity, hierarchy, consistency, and readability over decoration.

## 2. Supplier/Admin Panel

- Management-oriented interface.
- Follow the owner-provided module order in `MODULES.md`.
- Use clear navigation, data tables, filters, forms, status indicators, and dashboard visualizations where relevant.
- Each module screen must be designed and reviewed individually.
- Follow the approved visual direction and design prompt in `SUPPLIER_UI_DESIGN_REFERENCE.md`.
- Use a distinctive, polished floral/botanical identity while keeping the interface practical and readable.
- Do not copy the reference shop's name, logo, business identity, or sample content.
- The reference screenshot is not the official FlowerTrack IMS logo. If the owner supplies the official logo separately, use it as the primary brand reference and do not alter it.

## 3. Client Panel

- Ecommerce / online flower shop experience.
- Distinct from the Supplier/Admin management interface.
- Product browsing and shopping should be intuitive.
- Product cards, catalog filters, cart, order views, and tracking should be designed as relevant.
- Exact storefront style and checkout details remain to be decided.

## 4. Styling technology rules

| Tool | Intended use |
|---|---|
| Bootstrap | Primary UI components, grid, utilities, and responsive behavior |
| Flexbox | Custom alignment and layout |
| Custom CSS | Colors, font sizes, spacing, dimensions, typography, and visual details |
| HTML | Semantic page structure |
| JavaScript | Interactive behavior |

Bootstrap is the primary UI framework preference. Do not add Tailwind or another competing UI framework without discussing the reason first. Use Bootstrap Icons as the standard icon library for interface icons across both Supplier/Admin and Client panels.


## 5. Iconography

- Use **Bootstrap Icons** for interface icons throughout the application, including sidebars, navbars, dashboard summary cards, panel headings, quick actions, and activity indicators.
- Install and load the `bootstrap-icons` npm package through the Vite CSS pipeline using `@import 'bootstrap-icons/font/bootstrap-icons.css';` in `resources/css/app.css`.
- Use Bootstrap Icons markup such as `<i class="bi bi-house-door"></i>`, keeping the icon inside the existing wrapper element when one is used for layout/styling.
- Avoid Unicode symbols and emoji as replacements for interface icons.
- Product imagery is different: flower emoji or images may be used as product illustrations/content where appropriate, such as the Top Selling Products list.

## 6. Design system to establish during UI Foundation

- Brand color tokens informed by the official FlowerTrack logo when available.
- Neutral/background/surface colors.
- Typography family and type scale.
- Spacing scale.
- Border, radius, and shadow rules.
- Button variants and sizes.
- Form controls and validation states.
- Table styles and empty/loading/error states.
- Responsive breakpoints and navigation behavior.
- Reusable layout and component patterns.
- Consistent chart, status, and data-visualization treatments.

## 7. Supplier Side reference image and prompt

The owner has approved the uploaded flower-shop dashboard screenshot and supplied design prompt as a **visual reference for the Supplier Side**. This newer approval supersedes the earlier exclusion.

- Refer to `SUPPLIER_UI_DESIGN_REFERENCE.md` for the image, prompt requirements, visual cues, brand safeguards, and application scope.
- Use the image as inspiration for the botanical atmosphere, green/dark navigation, light content surfaces, floral imagery, dashboard hierarchy, and professional finish.
- Aim for an original FlowerTrack IMS interface, not a pixel-for-pixel recreation.
- The screenshot's shop name and logo are not FlowerTrack IMS branding.
- If the official FlowerTrack logo is supplied separately, analyze its colors and personality to establish the primary brand palette. Do not redesign, distort, or change that logo.
- When a task specifically requests a UI image, the supplied prompt calls for one complete high-fidelity image, without variants or accompanying explanation.
- This reference does not change the Client Panel direction, module order, or any operational business rule.

## 8. UI completion criteria

A UI phase is complete only when:
- All planned screens for that phase have been created.
- Navigation between screens is coherent.
- Responsive behavior has been checked.
- Reusable elements are consistent.
- Empty, loading, validation, and error states are considered where relevant.
- The owner has reviewed and approved the visual direction.
- Progress and relevant design decisions are documented with a date.


## 9. Frontend file organization

Keep page structure, styling, and browser behavior separated for maintainability as Supplier modules grow.

For a module with its own styles and interactions, use separate files following this pattern:

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

- Blade files contain the page's semantic markup and Laravel Blade directives.
- Module CSS contains page-specific presentation rules.
- Module JavaScript contains page-specific interactions and behavior.
- Keep `app.css` and `app.js` for shared/global styles, imports, and initialization; do not put every module's page-specific code into these entry files.
- Load assets through Vite according to the project's established Laravel/Vite setup.
- Reuse shared styles and scripts where appropriate. Do not create empty or unnecessary module files when a page has no module-specific styling or interaction.
- Follow this pattern for Flower Management and subsequent Supplier modules as applicable.


## Approved module UI reference — Flower Management (2026-11-03)

The owner approved Flower Management UI and explicitly selected it as the visual reference for the next Supplier/Admin modules. Preserve the approved dashboard and shared shell; use Flower Management for module-page cards, toolbars, tables and dialogs.

- Put the module heading, short description and primary action inside the card header.
- Use a compact toolbar with Show Entries on the left and a compact search field on the right, separated by space; stack controls on narrow screens.
- Keep the approved colors, typography, button/icon treatment, card borders/radii, spacing, row density, status badges, modal forms and footer/pagination styling consistent.
- Use DataTables native sortable column headers. Actions is unsortable; do not add a separate Sort dropdown.
- Keep header labels aligned with their body values. The accepted Flower table uses left-aligned Flower Details/Category and centered Minimum Stock/Status/Actions.
- Keep Flower Details proportional to the other columns rather than allowing it to consume the remaining table width. The Flower-specific approved widths are 30% / 22% / 18% / 14% / 16%, with a fixed-layout table and horizontal scrolling when needed. Adapt widths to each future module's actual data.
- Flower Management has no duplicate Flower Catalog heading, record-count badge, category/status toolbar filters or separate sort dropdown in the approved UI.
- Adapt fields, columns, actions and any necessary filters to each module's approved scope. Reuse the design, not Flower Management's business data or an identical column list.
- Keep Blade, CSS and JavaScript separate. Consolidate component styles rather than appending repeated conflicting overrides.
- Review each subsequent module with the owner and follow the documentation/build/commit/push/verification cycle before proceeding.

Reference application files: `resources/views/supplier/flowers.blade.php`, `resources/css/supplier/flowers.css`, and `resources/js/supplier/flowers.js`. The owner approved the local Flower UI; its application commit `3f580e5` was subsequently verified on GitHub.


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
