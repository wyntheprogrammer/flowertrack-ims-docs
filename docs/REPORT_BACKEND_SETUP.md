# Reports backend setup

Prepared 2026-11-10 against the repository Reports UI/routes matching application checkpoint `8e2ec1b`. The current app repository does not yet contain ReportService.php. If you already installed another uncommitted Reports candidate locally, review that local diff before overwriting it.

Copy the six full replacement files to the matching project paths. No new migration, database reset, Model or npm dependency is required. Existing jspdf, jspdf-autotable and exceljs dependencies remain in use.

## Included reports

1. Sales: immutable completed Sales Records, gross/refunds/net. Refunds-to-date restate the original completion day in Asia/Manila.
2. Payments: original payment amounts and Pending/Confirmed/Refunded status, verified original collection, refunds-to-date and net collection. Receipt rows use their original recorded day; this is not a separate refund-date cash-flow statement.
3. Inventory: physical remaining non-spoiled stock, Active reservations, available quantity, minimum and stock status. Negative availability is preserved to reveal an inconsistency. Stem/Bouquet quantities remain separate.
4. Orders: actual order records and independent fulfillment statuses. Cancelled orders do not become completed Sales.
5. Delivery: actual Delivery records, current personnel/status and scheduled day; Pickup is excluded. It is not an attempt-history performance report.
6. Clients: current profile/account fields, order count and net completed purchases grouped by stable client_id. Equal or changed names cannot merge client totals. This report's net purchase metric differs explicitly from the Client profile's gross completed-purchase metric.
7. Spoilage: actual negative Stock-Out ledger movements with metadata reason Spoilage. No risk prediction, elapsed expiry, Other use, receipt correction or customer fulfillment is fabricated as waste. Unit comes from the batch receiving-unit field (Bouquet versus stem receiving units).

Reports are Admin-only at the route and service boundaries. Data is read in one database transaction and returned through an explicit credential-free DTO; the endpoint does not change stock, payments, orders or other operational records. Default MariaDB Repeatable Read provides a consistent snapshot across the read queries.

The UI preserves approved table controls, native sorting, compact search/page length and table-only horizontal scrolling. Refresh loads a new canonical snapshot; failed refresh retains the previous one. All matching rows across pages, using the displayed snapshot's generated time and definitions, are used for PDF/Excel/Print. Print text is escaped. Excel normal monetary cells are numeric with PHP formatting; very large values fall back to exact text to avoid Excel precision loss. Exact cents and overflow checks prevent silent rounding.

## Installation checks

After copying all files into C:\xampp\htdocs\FlowerTrack:

```text
composer dump-autoload
php artisan optimize:clear
php artisan route:list --path=supplier/reports
npm.cmd run build
php artisan test --filter=ReportBackendTest
php artisan test --filter="SalesBackendTest|OrderBackendTest|ClientBackendTest|DeliveryBackendTest|UserBackendTest"
```

Expected: two protected Reports GET routes (page and data). Do not create a Reports migration or use migrate:fresh/rollback for this installation.

Nine isolated SQLite ReportBackendTest cases are supplied. They cover empty/seven real report datasets, no operational writes, stable client IDs, immutable prices, payment/refund separation, actual spoilage only, negative availability, Manila dates and Admin-only signed-session access. Test fixtures use named memory-only SQLite and existing migration files by suffix; they never refresh the owner's MariaDB data.

Verification performed here: Node syntax check and focused Reports UI/export checks passed. PHP is unavailable here; PHP tests, full Vite build, browser export rendering and owner acceptance remain pending.

## Browser acceptance

- Sign in as Admin and compare all seven report types with their actual operational modules.
- Verify historical Sale totals and original completion dates after a refund; verify Pending receipts are excluded from verified collections.
- Verify a client rename or equal display names do not merge totals, and Stem/Bouquet quantities remain separate.
- Search a report, use multiple table pages, and export PDF/Excel/Print. Every matching row across pages must be present, with definitions and snapshot generation time.
- Check Excel ordinary money cells are numeric; confirm long text wraps and print/PDF columns remain readable.
- Verify non-Admin accounts cannot load the report page/data. Refresh should not mutate the underlying records.

After owner approval: update docs with actual results, inspect targeted Git status, provide app commit/push commands, then continue to the next module. Assistant pushes docs; owner pushes the application.

## Vite integration fix — 2026-11-10

Reports JS is already imported by resources/js/app.js. The Reports Blade must not call @vite('resources/js/supplier/reports.js') as a standalone manifest entry. The full code guide now removes that redundant scripts push. Clear compiled views and rebuild app assets after applying the corrected full Blade. Owner reported backend tests passed, but page/browser/export verification is still pending.
