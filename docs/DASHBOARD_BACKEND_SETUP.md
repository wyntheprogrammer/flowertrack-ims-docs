# Supplier Dashboard backend setup

Verified by owner 2026-11-11: **9 Dashboard tests passed / 130 assertions; full suite 98 passed / 1,377 assertions; production build passed**. Owner confirmed the Dashboard working with MySQL and approved proceeding. Application commit/push remains pending; latest confirmed app checkpoint is `0ed6c7f` (Audit).

## Resulting behavior

- Existing green dashboard layout retained; sample figures, invented percentage growth and seeded period chart removed.
- On-hand inventory is shown as separate Stem and Bouquet quantities; includes remaining non-spoiled stock, including inactive products that still physically exist.
- Low Stock Items counts active catalog items whose available quantity (on-hand minus Active reservations) is below their configured minimum. Catalog items with no receipts count when below minimum. Negative availability is retained rather than clamped.
- Pending Orders means status `Pending` only: awaiting confirmation. Confirmed/Processing/Ready orders are not included in this card.
- Net Sales is all-time completed SalesRecord total (including recorded delivery fees), less actual refunds linked to those completed orders. Payment collection for an unfulfilled order is not recognized as a completed sale. Multiple refunds are aggregated before joining to prevent multiplied sales totals.
- Top Selling ranks completed order item snapshots by all-time gross item subtotal. It excludes delivery fees and does not allocate order-level refunds to products. Product identity/type stay separate; names come from retained order snapshots (MAX name is the deterministic label where multiple names exist for the same identity/type). Current catalog price changes do not reprice history.
- Inventory Overview is the current on-hand stock for up to six products with the highest quantities in the chosen Stem/Bouquet unit. The old period selector was sample-only; it is replaced with a unit selector. No invented historical inventory or ML predictions.
- Recent Activities comes from order histories, inventory movements and completed sale records. It does not read or expose Admin-only audit/security data.
- Dashboard displays current signed-in account name, refresh time in Asia/Manila, and role-appropriate quick actions. Admin-only sidebar links are hidden for Supplier users while backend access protection remains.
- Notification badge/counts use the Dashboard snapshot. Other pages provide inventory/order links and a Dashboard link without a fake or stale count. No polling, notifications table, push delivery or prediction alert engine is claimed.
- Page refresh or Apply re-reads actual DB data. This package deliberately uses server-rendered Blade and accessible SVG; no chart dependency or extra Vite entry is required.

## Install complete files

Open [DASHBOARD_BACKEND_FULL_CODE.md](DASHBOARD_BACKEND_FULL_CODE.md).

Create these four new files with exactly the provided paths:

1. `app/Services/DashboardService.php`
2. `app/Http/Controllers/Supplier/DashboardController.php`
3. `resources/css/supplier/dashboard.css`
4. `tests/Feature/DashboardBackendTest.php`

Replace these five existing files with their complete code blocks:

1. `resources/css/app.css` — retains module imports and adds Dashboard CSS.
2. `resources/js/supplier/shell.js` — retains navigation/panel controls and removes seeded chart generation.
3. `resources/views/supplier/dashboard.blade.php`
4. `resources/views/layouts/supplier.blade.php`
5. `routes/web.php` — preserves existing routes/middleware, connects DashboardController and adds `supplier.dashboard.data`.

Save all files (VS Code: Ctrl+K, then S). If your local versions differ from checkpoint `0ed6c7f`, review your diff before replacing the shared files. Do not overwrite unrelated uncommitted work.

No `php artisan migrate` is needed for this module. Do not generate Dashboard tables. Existing business tables are its data source. Do not change `vite.config.js` or add a standalone `@vite('resources/js/supplier/dashboard.js')` entry.

## Check from C:\xampp\htdocs\FlowerTrack

```powershell
php -l app/Services/DashboardService.php
php -l app/Http/Controllers/Supplier/DashboardController.php
php -l routes/web.php
php -l tests/Feature/DashboardBackendTest.php
node --check resources/js/supplier/shell.js
php artisan optimize:clear
php artisan route:list --path=supplier/dashboard
php artisan test --filter=DashboardBackendTest
npm.cmd run build
```

Expected Dashboard routes: `supplier.dashboard` and `supplier.dashboard.data`. Expected dedicated test cases: **9 passed**; no assertion count or success is claimed before owner execution. The tests use their own named SQLite `:memory:` database and verify the connection before creating fixture schemas. A skipped pdo_sqlite test is not a passing test.

After dedicated tests pass:

```powershell
php artisan test
```

## Browser acceptance

Sign in as Admin, open `/supplier/dashboard`, and hard refresh (Ctrl+F5).

- Match on-hand Stem/Bouquet values against Inventory. Reservations affect available/low-stock counts but do not issue physical stock.
- Match pending count to Orders in Pending status.
- Match net sales against completed sales less linked refunds; pending payments alone must not increase sales.
- Switch Stems/Bouquets, Apply and Refresh. Confirm no mixed-unit chart or sample growth figures.
- Verify empty states when no completed sales exist; no seeded products/activities should appear.
- Check narrow/mobile layout and sidebar toggle. Verify account name and operational notification counts.
- Sign in as Supplier: Dashboard works; Reports/Users/Audit links are absent; the direct Admin endpoints must still reject Supplier access.
- Confirm Inventory/Order/Sales workflows keep working after the shared layout/shell changes.

## Save only after owner verification

Owner pushes the application; assistant updates and pushes docs. Do not stage the unrelated deleted `resources/views/supplier/sales-payment.blade.php` or untracked `tatus`.

```powershell
git status --short
```

Send dedicated/full test results, build summary and current status before final targeted application commit instructions. This prepared package has not modified or pushed the application repository.
