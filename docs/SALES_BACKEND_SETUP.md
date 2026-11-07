# Sales & Payments backend setup

Prepared 2026-11-07 against application checkpoint `9556331`.

Copy all 15 full replacement files to the exact paths shown. Create the new migration file directly as `database/migrations/2026_11_07_090000_create_finance_tables.php`; do not create a second migration with the same suffix and do not edit the older Order migration.

## Included behavior

- Database-backed selling prices for active Stem/Bouquet catalog items, priced per one base unit. Existing Order item prices and completed Sales snapshots remain unchanged.
- One full payment per order, recorded as Pending and manually Confirmed after verifying actual collection. Cash, GCash and Bank Transfer are supported; noncash references are required and unique per method after trim/case normalization.
- Admin and Supplier/Staff can set prices, record payments and confirm receipts. Refund recording is Admin-only and requires a reason. Financial access is blocked for Client/Delivery/inactive/temporary-password accounts.
- Only non-cancelled orders without any payment can receive a new full receipt. Partial/excess payments and additional receipts are rejected. Payment state remains independent from fulfillment; no new Paid-before-Completion gate is added.
- Any refund marks the payment/order financial status Refunded. The original confirmed amount, method, reference and confirmation metadata remain; append-only refund rows record the amounts. Additional refunds can use the remaining refundable amount, but total refunds cannot exceed the original confirmed payment. No new collection on refunded orders.
- Refunds record an already verified manual refund; they do not transfer money, return stock or delete the original receipt or sale. Refunds against completed Sales reduce net completed sales at the original completion-date grouping. Refunds for orders with no completed Sale are not counted as negative Sales.
- Payment record/confirm/refund never modifies inventory, reservations, fulfillment status or immutable Sales Records. Actual completed collection/delivery remains the only order fulfillment stock-out trigger.
- The new finance_operations table provides atomic UUID replay, actor-bound request hashing, financial history and durable noncash-reference uniqueness. payment_refunds stores append-only refund entries. Existing flower_prices, order_payments and sales_records remain in use.
- Financial writes use Order→Payment or Flower→Price locks and current version checks. Successful matching UUID retries replay before stale checks; changed payloads cannot reuse a UUID. Financial dataset reads use a consistent read transaction.
- UI keeps approved tables/tabs/native sorting, compact search/page-length controls and table-only horizontal scrolling. Money uses exact cents, bounded amounts and checked JS integer aggregates. Philippine date summaries use Asia/Manila. Uncertain network/server writes retain the exact UUID and payload for retry; no optimistic receipt/refund mutation or browser financial storage.
- OrderService replacement retains the original confirmed payment amount in its public paidCents field after a refund, while the financial status remains Refunded. No other Order workflow change is introduced.

## Installation checks

Run from `C:\xampp\htdocs\FlowerTrack` after all files are copied:

```text
composer dump-autoload
php artisan optimize:clear
php artisan migrate
php artisan route:list --path=supplier/sales-payments
npm.cmd run build
php artisan test --filter="SalesBackendTest|OrderBackendTest|ClientBackendTest|DeliveryBackendTest|UserBackendTest"
```

Expected: the new finance migration runs and six Sales & Payments routes appear. Never use migrate:fresh, refresh or rollback against existing application data for this installation.

Eleven isolated SalesBackendTest cases are supplied, including signed-session HTTP routes, exact full payments, reference/UUID uniqueness, role boundaries, refunds and historical snapshot protection. PHP is unavailable in the preparation environment, so no PHP test result, full build or browser acceptance is claimed. Node syntax and focused financial UI tests passed; backend/database specialists reviewed actual source and tests.

## Browser acceptance

1. Sign in as Admin; open Sales & Payments and set an actual selling price. Verify an existing order keeps its original item price.
2. Record a payment for an eligible order: amount equals the full total and receipt is Pending. Noncash reference is required. Verify no stock/order status change.
3. Confirm only after verifying actual receipt/transfer/cash. Order financial status becomes Paid; stock/fulfillment remains unchanged.
4. Check a completed Pickup/Delivery sale appears once with immutable item/price snapshots. Net Sales is a commercial completed-sales metric, not a cash collection metric.
5. As Admin, record a partial refund, inspect retained original receipt and refund history, then test remaining/full refund. Over-refunds and additional payment collection must be rejected. Supplier accounts must not see or execute refunds.
6. Verify filters were not added, native sorting/search/page length still work, and financial/order/client screens show the correct Paid/Refunded status.

After owner approval, update MD with actual results, inspect targeted Git status and save/push the application checkpoint. Final personnel/Delivery browser verification was still unreported at package preparation time.
