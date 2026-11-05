# Inventory Backend Setup

**Last updated:** 2026-11-05

## Migration filename correction — 2026-11-05

The test originally required a fixed migration timestamp. Owner reported eight setup failures before assertions because Artisan generated a different filename. The corrected InventoryBackendTest locates exactly one migrations/*_create_inventory_tables.php file, so timestamps do not need to match. Keep the generated filename, particularly if it has already been migrated; do not create a duplicate migration. If necessary, use php artisan migrate instead of the example fixed-path migrate command below, after checking pending migrations. Updated PHP test still requires owner execution.

## Status and scope

Complete replacement source prepared for the Inventory module; owner installation and PHP/runtime verification are pending. This is not a claim that the application repository was updated or pushed.

Includes Stock-In, batch balances, receipt corrections, Other use/Spoilage Stock-Out, FIFO and documented specific-batch override, stem receiving conversions, database movement history and reserved-stock protection.

Customer-order deduction is not a manual Stock-Out option. It will be connected to actual pickup collection / delivery completion in the Order and Delivery backend. Reservation storage is a foundation only; there is no public reservation-writing endpoint yet. Reserved quantities will be zero until that integration supplies real reservations. The standalone Batch Management page is a later backend integration; Inventory's own batch details use the new database.

## Files

- database/migrations/2026_11_05_130000_create_inventory_tables.php
- app/Models/InventoryBatch.php
- app/Models/InventoryUnitSetting.php
- app/Models/InventoryMovement.php
- app/Models/InventoryOperation.php
- app/Models/InventoryReservation.php
- app/Http/Requests/InventoryRequest.php
- app/Http/Requests/StockInRequest.php
- app/Http/Requests/CorrectReceiptRequest.php
- app/Http/Requests/StockOutRequest.php
- app/Http/Requests/UpdateInventoryUnitsRequest.php
- app/Services/InventoryService.php
- app/Http/Controllers/Supplier/InventoryController.php
- resources/views/supplier/inventory.blade.php
- resources/js/supplier/inventory.js
- tests/Feature/InventoryBackendTest.php

Existing Flower model, flowers migration, approved Inventory CSS and app.js Inventory import remain in use. No new npm packages are required.

## Routes

Import App\\Http\\Controllers\\Supplier\\InventoryController once in routes/web.php. Replace only the old Inventory routes INSIDE the existing supplier group, which supplies prefix supplier, name supplier., authentication, active-account, changed-password and admin/supplier role middleware.

```php
Route::get('/inventory', [InventoryController::class, 'index'])->name('inventory');
Route::get('/inventory/data', [InventoryController::class, 'data'])->name('inventory.data');
Route::post('/inventory/stock-in', [InventoryController::class, 'stockIn'])->name('inventory.stock-in');
Route::post('/inventory/stock-out', [InventoryController::class, 'stockOut'])->name('inventory.stock-out');
Route::post('/inventory/batches/{batch}/receipt', [InventoryController::class, 'correctReceipt'])
    ->whereNumber('batch')->name('inventory.correct-receipt');
Route::post('/inventory/flowers/{flower}/units', [InventoryController::class, 'updateUnits'])
    ->whereNumber('flower')->name('inventory.units');
```

Do not duplicate the supplier prefix or replace unrelated routes.

## Local verification (CMD)

Run in C:\\xampp\\htdocs\\FlowerTrack. Stop and address any PHP syntax error before migration.

```cmd
for %f in (app\\Models\\Inventory*.php) do @php -l "%f"
for %f in (app\\Http\\Requests\\InventoryRequest.php app\\Http\\Requests\\StockInRequest.php app\\Http\\Requests\\CorrectReceiptRequest.php app\\Http\\Requests\\StockOutRequest.php app\\Http\\Requests\\UpdateInventoryUnitsRequest.php) do @php -l "%f"
php -l app/Services/InventoryService.php
php -l app/Http/Controllers/Supplier/InventoryController.php
php -l database/migrations/2026_11_05_130000_create_inventory_tables.php
php -l tests/Feature/InventoryBackendTest.php
php artisan config:clear
php artisan route:clear
php artisan view:clear
php artisan migrate --path=database/migrations/2026_11_05_130000_create_inventory_tables.php
php artisan route:list --path=supplier/inventory
npm run build
php artisan test --filter=InventoryBackendTest
```

Use single %f when pasting in CMD. The test class creates a named SQLite :memory: connection before schema operations, does not use RefreshDatabase, and requires only the Inventory migration against that ephemeral connection. It skips if pdo_sqlite is unavailable. No fresh/rollback command is needed.

## Domain behavior

- On hand = usable batch remaining quantities; available = on hand minus active reservations.
- Receiving units: Stem factor 1; per-item Bunch/Bundle/Box factors default 10/20/100. Bouquet always factor 1, no component stem deduction.
- The server calculates factors; it rejects client factor overrides. Each batch snapshots the factor.
- Updating unit settings affects new receipts. A same-unit correction retains its historical factor.
- Receipt corrections require remarks; preserve consumed quantity; cannot reduce below consumption or active reservations; append signed ledger deltas with before/after metadata.
- FIFO is received date then numeric batch ID. Specific-batch overrides require remarks.
- Spoilage requires remarks. Partial spoilage leaves usable remainder; a batch whose remainder reaches zero through spoilage is marked spoiled. Expiry dates never automatically deduct stock.
- Every stock mutation uses a UUID, normalized payload fingerprint and actor snapshot in the same transaction as stock and ledger writes. A matching retry returns its original result; a conflicting reuse is rejected.
- All stock writes lock the parent Flower first. Future reservation writers must use the same parent-lock convention.
- No physical quantity is deducted on payment, Ready for Pickup, Ready for Delivery, or dispatch. Actual order completion integration remains later work.

## Verification evidence

Database and Backend specialists reviewed schema/transaction and UI/server contracts. JavaScript syntax and 14 focused mocked-request/preview checks passed. Eight Laravel service tests were prepared for FIFO, reservation protection, atomic rollback, retry/reuse, corrections, conversions and spoilage, but were not executed here because PHP is unavailable. These tests do not establish MariaDB concurrency behavior.

## Browser acceptance

1. Existing catalog items appear with zero balances if no real receipts exist.
2. Receive 2 Bunch with factor 10: on hand becomes 20 stems and survives refresh.
3. Change future Bunch factor to 12: the old receipt remains factor 10; new receipts use 12.
4. Receive a finished Bouquet: its stock is separate and stem balances do not change.
5. Stock out across two batches: earlier received stock is consumed first.
6. Correct a receipt with remarks: consumed quantity stays deducted; history gains a correction entry.
7. Record partial and full spoilage, confirm remaining quantities/status and history.
8. Refresh to confirm persistence. Send output/errors before marking the module accepted or pushing it.

After owner acceptance: update progress/changelog, then provide a targeted application commit/push script. Leave unrelated Audit Trail changes, deleted sales-payment.blade.php and untracked tatus untouched.
