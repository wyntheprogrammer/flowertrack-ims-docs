# Order Management Backend Setup

**Last updated:** 2026-11-06

## Status

Full module source prepared for local installation/testing. Owner explicitly requested proceeding after the Client checkpoint instructions; no Client application push output was supplied. Last confirmed application commit remains Batch 330b885. Do not claim the Client or Order changes pushed.

Spoilage Prediction remains owner-deferred. Client storefront/login is also deferred.

## Scope and dependencies

- Manual Supplier/Admin orders use real clients, inventory and Sales-owned configured prices.
- Admin confirms Pending orders; operational staff may create/process/cancel and confirm actual pickup collection.
- Confirmation reserves full quantities without reducing on hand. Processing and ready states do not deduct.
- Actual complete pickup collection consumes reservations and performs FIFO stock-out once, with one immutable Sales Record in the same transaction.
- Delivery orders stop at Ready for Delivery here. Assignment, dispatch, actual Delivered proof and its coordinated completion belong to the later Delivery backend.
- Payment is read-only and independent of fulfillment. The existing single-full-payment recording rule is preserved; no new requirement to be Paid before completion is introduced. Payment recording/verification/refunds remain later Sales & Payments work.
- New flower_prices, order_payments and sales_records storage are shared foundations for that later module, not completed Sales/Payments UI integration.
- Existing order item product name/code/type/unit/price snapshots remain unchanged by later price/catalog edits. Same-item edits retain price snapshots. Any existing payment record blocks editing financial terms through this module.
- Duplicate requested flower rows aggregate, capped at 100000 per catalog item and 50 input rows. Stem and finished Bouquet are never converted into one another.
- Exact monetary arithmetic uses integer centavos bounded to 999999999999; no client-provided prices/totals are accepted.

## Files

- database/migrations/2026_11_06_160000_create_order_tables.php
- app/Models/FlowerPrice.php
- app/Models/Order.php
- app/Models/OrderItem.php
- app/Models/OrderHistory.php
- app/Models/OrderPayment.php
- app/Models/SalesRecord.php
- app/Support/Money.php
- app/Services/OrderService.php
- app/Services/InventoryService.php (full replacement; existing Inventory/Batch behavior retained plus reservation fulfillment)
- app/Http/Requests/OrderRequest.php
- app/Http/Requests/StoreOrderRequest.php
- app/Http/Requests/UpdateOrderRequest.php
- app/Http/Requests/OrderActionRequest.php
- app/Http/Controllers/Supplier/OrderController.php
- app/Http/Controllers/Supplier/ClientController.php (full replacement for real order history/summary)
- app/Console/Commands/SetSellingPrice.php
- config/orders.php
- resources/views/supplier/orders.blade.php
- resources/js/supplier/orders.js
- resources/js/supplier/clients.js (full replacement for live summary)
- tests/Feature/OrderBackendTest.php
- tests/Feature/ClientBackendTest.php (updated for real Order schema/summary)

Keep approved CSS, existing app.js imports, User model and consolidated Client migration. Use exactly one *_create_order_tables.php migration; timestamps may vary. New migration creates flower_prices, orders, order_items, order_histories, order_payments and sales_records; existing stock/clients are not reset.

## Routes

Import App\\Http\\Controllers\\Supplier\\OrderController once at the top of routes/web.php. Replace only Order routes inside the existing protected supplier group (supplier prefix/name, auth, active-account, changed-password, admin/supplier):

```php
Route::get('/orders', [OrderController::class, 'index'])->name('orders');
Route::get('/orders/data', [OrderController::class, 'data'])->name('orders.data');
Route::get('/orders/{order}', [OrderController::class, 'show'])->whereNumber('order')->name('orders.show');
Route::post('/orders', [OrderController::class, 'store'])->name('orders.store');
Route::post('/orders/{order}/update', [OrderController::class, 'update'])->whereNumber('order')->name('orders.update');
Route::post('/orders/{order}/action', [OrderController::class, 'action'])->whereNumber('order')->name('orders.action');
```

## Price and fee setup

No sample selling price or delivery fee is silently installed.

After migration, set each item's actual current selling price using the trusted local setup command:

```text
php artisan sales:set-price FLW-001 60.00
```

FLW-001 and 60.00 are examples only. Use actual catalog code and approved amount, per ONE base unit: stem or finished bouquet. This writes Sales-owned flower_prices; it does not reprice existing order snapshots. Later Sales/Payments UI will manage this same data.

For Delivery, set an approved decimal fee in .env, for example ORDER_DELIVERY_FEE=150.00 only if 150.00 is the actual shop fee. Run php artisan config:clear afterward. Missing configuration disables new Delivery orders; Customer Pickup uses zero delivery fee and remains available. Existing Delivery orders retain their fee snapshot.

## PowerShell checks

Run from C:\\xampp\\htdocs\\FlowerTrack after copying all source files. Stop on missing files or syntax errors before migrating.

```powershell
composer dump-autoload
Get-ChildItem app\\Models\\Order*.php, app\\Models\\FlowerPrice.php, app\\Models\\SalesRecord.php | ForEach-Object { php -l $_.FullName }
Get-ChildItem app\\Http\\Requests\\*Order*.php | ForEach-Object { php -l $_.FullName }
php -l app/Support/Money.php
php -l app/Services/OrderService.php
php -l app/Services/InventoryService.php
php -l app/Http/Controllers/Supplier/OrderController.php
php -l app/Http/Controllers/Supplier/ClientController.php
php -l app/Console/Commands/SetSellingPrice.php
php -l config/orders.php
php -l tests/Feature/OrderBackendTest.php
php -l tests/Feature/ClientBackendTest.php

php artisan config:clear
php artisan route:clear
php artisan view:clear
php artisan migrate
php artisan route:list --path=supplier/orders
npm.cmd run build

php artisan test --filter=InventoryBackendTest
php artisan test --filter=BatchBackendTest
php artisan test --filter=ClientBackendTest
php artisan test --filter=OrderBackendTest
```

Lint the new migration using its actual filename too. Expected Order routes:6. No new npm package required. Tests use named SQLite memory databases and migration-suffix discovery; no fresh/rollback/application-data reset.

ClientBackendTest is replaced because ClientController now returns real order history instead of the previous unavailable state. Run Inventory/Batch regression tests because their shared service has gained fulfillment.

## Transaction and retry safeguards

- Create uses canonical actor+payload hash and UUID uniqueness, including a fresh current replay lookup after waiting for the Client serialization lock. Conflicting reuse is rejected.
- Order actions use explicit desired states plus version. Repeating an already-achieved state does not issue stock again.
- Existing inactive clients/catalog lines may continue already-placed orders; new client selections must be active, and inactive catalog lines may only stay at the same/decreased quantity.
- Order mutations lock the order, relevant client account/profile when needed, then all affected Flower IDs in ascending order.
- Availability and fulfillment use current locking reads, not a stale repeatable-read snapshot created earlier in the transaction.
- Confirmed edits rebalance holds atomically; cancellation releases active holds and retains history/payment records.
- Pickup requires explicit actual-collected confirmation. The full mixed-item order, reservation consumption, inventory ledger, completion status and unique sale either all commit or all roll back.
- Manual Inventory Stock-Out still cannot select Customer order and bypass the completion workflow.
- Sales records capture gross completed-order snapshots; no invented payment, refund or cash receipt is recorded.
- Client profile purchase summary is gross completed-order total including delivery fee, not a payment balance or refund-adjusted collection total.

## Verification boundary

Database and Backend subagents reviewed actual schema, stock invariants, money bounds, request/Blade/JS contracts and replay/current-locking fixes. JavaScript syntax checks passed; 19 Order checks plus 5 live Client-history integration checks passed.

Ten isolated Order service/validation tests and the updated Client suite are supplied but were not executed here because PHP is unavailable. They cover aggregation, creation replay, Admin confirmation/cancellation, insufficient stock rollback, price snapshots/payment edit locks, actual pickup one-time completion, mixed-item full rollback, Delivery boundaries/config, stale versions/inactive-item increases, financial tampering and integer-money parsing. These do not prove real MariaDB concurrency or full browser/HTTP authorization coverage.

## Browser acceptance

1. Prepare active client, actual stock receipts and configured selling prices.
2. Create a mixed Stem/Bouquet Pickup order: Pending does not reserve or deduct.
3. Admin confirmation increases reserved, decreases available, leaves physical on hand unchanged.
4. Move through Processing and Ready for Pickup: still no physical deduction.
5. After actual collection, confirm Completed: all items deduct once FIFO, holds become Consumed, one Sales Record appears.
6. Refresh/retry completion: no duplicate stock-out or sale.
7. Cancel another confirmed order with reason: release holds, do not deduct stock.
8. Edit Pending/Confirmed without payment record: retain old prices for existing item identities and rebalance confirmed reservations.
9. Client View displays real order count/history and completed-order gross total.
10. Delivery with actual configured fee stops at Ready for Delivery until its module is implemented.

After owner approval, update MD and provide a targeted application commit/push script. Do not describe the whole financial, delivery, Client portal or ML system as completed.
