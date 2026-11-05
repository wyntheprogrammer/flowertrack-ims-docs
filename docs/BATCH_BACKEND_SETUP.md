# Batch Management Backend Setup

**Last updated:** 2026-11-05

## Status

Complete Batch source prepared; owner installation and PHP/browser validation are pending. Last owner-confirmed Inventory commit is 2025b83. No Batch application commit or push has been claimed.

## Scope and files

Use existing Inventory tables, models, validated request base and ledger. There is no new migration.

- Replace app/Services/InventoryService.php with the full supplied file. Diff review confirms only two added methods: updateBatchMetadata and batchVersion; existing Inventory operations are unchanged.
- Add app/Http/Requests/UpdateBatchRequest.php (extends existing InventoryRequest).
- Add app/Http/Controllers/Supplier/BatchController.php.
- Replace resources/views/supplier/batches.blade.php.
- Replace resources/js/supplier/batches.js.
- Add tests/Feature/BatchBackendTest.php.

Keep the approved Batch CSS and existing app.js batches import. No npm dependency changes.

## Routes

Import App\\Http\\Controllers\\Supplier\\BatchController once near the top of routes/web.php.

Replace only existing Batch routes INSIDE the supplier group already providing supplier prefix/name, auth, active-account, changed-password, and admin/supplier authorization:

```php
Route::get('/batches', [BatchController::class, 'index'])->name('batches');
Route::get('/batches/data', [BatchController::class, 'data'])->name('batches.data');
Route::post('/batches/{batch}/update', [BatchController::class, 'update'])
    ->whereNumber('batch')->name('batches.update');
```

No create/delete/spoil endpoint is added. New batches and physical deductions remain Inventory operations.

## CMD checks

Run from C:\\xampp\\htdocs\\FlowerTrack.

```cmd
php -l app/Services/InventoryService.php
php -l app/Http/Requests/UpdateBatchRequest.php
php -l app/Http/Controllers/Supplier/BatchController.php
php -l tests/Feature/BatchBackendTest.php

php artisan config:clear
php artisan route:clear
php artisan view:clear
php artisan route:list --path=supplier/batches

npm run build

php artisan test --filter=InventoryBackendTest
php artisan test --filter=BatchBackendTest
```

Expected route count: 3. Do not create another Inventory migration. Existing Inventory tests remain regression checks for the shared service.

## Behavior

- Listing and details use actual inventory_batches and inventory_movements; no independent sample stock.
- Edit permits harvest date, optional expected freshness/expiry date and remarks only.
- Code, item, received date, receiving quantity/unit/factor, initial/remaining quantity and spoilage status cannot change through this endpoint.
- Metadata edits use the same parent Flower lock followed by a batch lock, in one transaction with an InventoryOperation and a signed zero-quantity Batch Metadata Update movement.
- Before/after snapshots appear in Batch movement details. Stock-Out quantities already have a negative sign and are displayed correctly; metadata changes display zero.
- UUID retries replay original results before checking stale versions. Reusing a key with another payload is rejected.
- The edit's version fingerprint detects intervening receipt, stock or metadata changes. A rejected stale edit must be refreshed and reopened.
- Freshness warnings retain the approved 3-day advisory threshold and use the server's Manila date. They never automatically deduct stock or mark spoilage.
- Only Inventory Stock-Out records actual spoilage. There is no Batch Mark Spoiled button/modal/action.
- Spoiled batch notes may be edited without restoring stock or removing spoilage history.
- Inventory and Batch pages share the same records; no duplicate ledger/deduction.

## Verification evidence

Database and Backend specialists reviewed actual code, invariants and JS/Blade/PHP contracts. Service diff was checked against the approved Inventory source. JavaScript syntax and 14 focused mocked-request/model checks passed.

Seven isolated SQLite service/validation tests were prepared, covering unchanged stock, date validation, stale versions, UUID replay/reuse, spoiled-batch preservation and prohibited field input. They were not run here because PHP is unavailable. Tests use a named :memory: connection and find exactly one *_create_inventory_tables.php migration, independent of its generated timestamp. No production database refresh/migration occurs in these tests.

Owner must still run PHP lint, Inventory and Batch test suites, and browser checks on the actual MariaDB-backed app.

## Browser acceptance

1. Receive stock in Inventory, then open Batch Management: the same batch/code/balance appears.
2. Edit harvest/freshness/notes, save and refresh: changes persist while quantities remain unchanged.
3. View history: actual receipt/issues/corrections plus a zero-quantity information update are visible; before/after changes expand safely.
4. Stock-Out in Inventory, then return to Batch: remaining/status match the physical quantity.
5. A past freshness date only displays a warning. Actual Inventory spoilage is reflected in Batch status/history.
6. Open two edits, save one, then save the older form: the stale save must be rejected and refreshed.

After owner acceptance, update MD progress and provide targeted application commit/push commands. Leave unrelated Audit Trail/sales-payment/tatus changes untouched.
