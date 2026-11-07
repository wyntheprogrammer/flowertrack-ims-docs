# FlowerTrack — Sales & Payments: full replacement code

Prepared against application checkpoint `9556331`. Each section contains one complete project file. Copy only its code block into the indicated path. PHP tests and browser acceptance remain pending.

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

## `app/Http/Controllers/Supplier/SalesPaymentController.php`

```php
<?php

namespace App\Http\Controllers\Supplier;

use App\Http\Controllers\Controller;
use App\Http\Requests\ConfirmPaymentRequest;
use App\Http\Requests\RefundPaymentRequest;
use App\Http\Requests\SalesPriceRequest;
use App\Http\Requests\StorePaymentRequest;
use App\Models\Flower;
use App\Models\OrderPayment;
use App\Services\SalesPaymentService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\View\View;

class SalesPaymentController extends Controller
{
    public function __construct(private readonly SalesPaymentService $sales)
    {
    }

    public function index(): View
    {
        return view('supplier.sales-payments');
    }

    public function data(Request $request): JsonResponse
    {
        return response()->json($this->sales->data($request->user()));
    }

    public function price(Flower $flower, SalesPriceRequest $request): JsonResponse
    {
        return response()->json([
            'message' => 'Selling price saved. Existing order prices are unchanged.',
            'data' => $this->sales->price($flower, $request->validated(), $request->user()),
        ]);
    }

    public function recordPayment(StorePaymentRequest $request): JsonResponse
    {
        return response()->json([
            'message' => 'Full payment recorded as Pending. Confirm after checking actual collection.',
            'data' => $this->sales->recordPayment($request->validated(), $request->user()),
        ], 201);
    }

    public function confirm(OrderPayment $payment, ConfirmPaymentRequest $request): JsonResponse
    {
        return response()->json([
            'message' => 'Payment confirmed. Order fulfillment and inventory are unchanged.',
            'data' => $this->sales->confirm($payment, $request->validated(), $request->user()),
        ]);
    }

    public function refund(OrderPayment $payment, RefundPaymentRequest $request): JsonResponse
    {
        return response()->json([
            'message' => 'Refund recorded. Original receipt retained; this does not transfer money or return stock.',
            'data' => $this->sales->refund($payment, $request->validated(), $request->user()),
        ]);
    }
}
```

## `app/Http/Requests/ConfirmPaymentRequest.php`

```php
<?php

namespace App\Http\Requests;

use App\Services\SalesPaymentService;

class ConfirmPaymentRequest extends SalesFinanceRequest
{
    public function rules(): array
    {
        return SalesPaymentService::rules('confirm');
    }
}
```

## `app/Http/Requests/RefundPaymentRequest.php`

```php
<?php

namespace App\Http\Requests;

use App\Services\SalesPaymentService;

class RefundPaymentRequest extends SalesFinanceRequest
{
    public function authorize(): bool
    {
        return parent::authorize() && $this->user()->role === \App\Models\User::ROLE_ADMIN;
    }

    public function rules(): array
    {
        return SalesPaymentService::rules('refund');
    }
}
```

## `app/Http/Requests/SalesFinanceRequest.php`

```php
<?php

namespace App\Http\Requests;

use App\Models\User;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Support\Str;

abstract class SalesFinanceRequest extends FormRequest
{
    public function authorize(): bool
    {
        $user = $this->user();
        return $user !== null && $user->is_active && !$user->must_change_password
            && in_array($user->role, [User::ROLE_ADMIN, User::ROLE_SUPPLIER], true);
    }

    protected function prepareForValidation(): void
    {
        if (is_string($this->input('request_key'))) {
            $this->merge(['request_key' => Str::lower($this->input('request_key'))]);
        }
    }
}
```

## `app/Http/Requests/SalesPriceRequest.php`

```php
<?php

namespace App\Http\Requests;

use App\Services\SalesPaymentService;

class SalesPriceRequest extends SalesFinanceRequest
{
    public function rules(): array
    {
        return SalesPaymentService::rules('price');
    }
}
```

## `app/Http/Requests/StorePaymentRequest.php`

```php
<?php

namespace App\Http\Requests;

use App\Services\SalesPaymentService;

class StorePaymentRequest extends SalesFinanceRequest
{
    public function rules(): array
    {
        return SalesPaymentService::rules('record');
    }
}
```

## `app/Models/FinanceOperation.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Model;

#[Fillable(['request_key', 'request_hash', 'reference_key', 'kind', 'actor_id', 'order_id', 'payment_id', 'flower_id', 'result_id', 'metadata'])]
class FinanceOperation extends Model
{
    protected function casts(): array
    {
        return ['actor_id' => 'integer', 'order_id' => 'integer', 'payment_id' => 'integer',
            'flower_id' => 'integer', 'result_id' => 'integer', 'metadata' => 'array'];
    }
}
```

## `app/Models/PaymentRefund.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

#[Fillable(['code', 'payment_id', 'amount_cents', 'reason', 'actor_id', 'actor_name'])]
class PaymentRefund extends Model
{
    protected function casts(): array
    {
        return ['payment_id' => 'integer', 'amount_cents' => 'integer', 'actor_id' => 'integer'];
    }

    public function payment(): BelongsTo
    {
        return $this->belongsTo(OrderPayment::class, 'payment_id');
    }
}
```

## `app/Services/OrderService.php`

```php
<?php

namespace App\Services;

use App\Models\Client;
use App\Models\Flower;
use App\Models\FlowerPrice;
use App\Models\InventoryBatch;
use App\Models\InventoryReservation;
use App\Models\Order;
use App\Models\OrderHistory;
use App\Models\OrderItem;
use App\Models\SalesRecord;
use App\Models\User;
use App\Support\Money;
use Illuminate\Database\QueryException;
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Str;
use Illuminate\Validation\ValidationException;
use InvalidArgumentException;

class OrderService
{
    /** Create a pending order without reserving or deducting stock. */
    public function store(array $input, User $actor): Order
    {
        $payload = $this->normalize($input);
        $key = strtolower($input['request_key']);
        $hash = hash('sha256', json_encode(['actor' => (int) $actor->id, ...$payload], JSON_THROW_ON_ERROR));
        try {
            return DB::transaction(function () use ($payload, $key, $hash, $actor): Order {
                $existing = Order::query()->where('request_key', $key)->first();
                if ($existing !== null) {
                    return $this->replay($existing, $hash);
                }
                $client = $this->lockClient($payload['client_id'], false, false);
                // A retry may have waited for the original transaction's account lock.
                $existing = Order::query()->where('request_key', $key)->lockForUpdate()->first();
                if ($existing !== null) {
                    return $this->replay($existing, $hash);
                }
                $this->assertClient($client, true);
                $flowers = $this->lockFlowers(array_keys($payload['items']));
                $lines = $this->lines($flowers, $payload['items'], collect());
                $this->assertAvailability($flowers, $payload['items']);
                [$subtotal, $fee, $total] = $this->totals($lines, $payload['method']);
                $order = Order::create([
                    'code' => 'TMP-' . Str::ulid(),
                    'request_key' => $key,
                    'request_hash' => $hash,
                    'client_id' => $client->id,
                    'client_code' => $client->code,
                    'client_name' => $client->user->name,
                    'client_email' => $client->user->email,
                    'method' => $payload['method'],
                    'address' => $payload['address'],
                    'notes' => $payload['notes'],
                    'status' => 'Pending',
                    'subtotal_cents' => $subtotal,
                    'fee_cents' => $fee,
                    'total_cents' => $total,
                    'stocked_out' => false,
                ]);
                $order->code = 'ORD-' . str_pad((string) $order->id, 3, '0', STR_PAD_LEFT);
                $order->save();
                $this->replaceItems($order, $lines);
                $this->history($order, $actor, 'Order created.');

                return $order;
            }, 3);
        } catch (QueryException $exception) {
            if (in_array((int) ($exception->errorInfo[1] ?? 0), [1062, 19], true)) {
                $existing = Order::query()->where('request_key', $key)->first();
                if ($existing !== null) {
                    return $this->replay($existing, $hash);
                }
            }
            throw $exception;
        }
    }

    /** Edit pending/confirmed orders and atomically rebalance confirmed reservations. */
    public function update(Order $order, array $input, User $actor): Order
    {
        $payload = $this->normalize($input);

        return DB::transaction(function () use ($order, $input, $payload, $actor): Order {
            $locked = Order::query()->lockForUpdate()->findOrFail($order->id);
            $locked->load(['items', 'payment']);
            $this->assertVersion($locked, $input['version']);
            if (!in_array($locked->status, ['Pending', 'Confirmed'], true) || $locked->payment !== null) {
                $this->invalid('order', 'Only pending or confirmed orders without any payment record can be edited.');
            }
            $changedClient = (int) $locked->client_id !== $payload['client_id'];
            $client = $this->lockClient($payload['client_id'], $changedClient);
            $old = $locked->items->keyBy('flower_id');
            $flowers = $this->lockFlowers(array_unique([...$old->keys()->all(), ...array_keys($payload['items'])]));
            $lines = $this->lines($flowers, $payload['items'], $old);
            $this->assertAvailability($flowers, $payload['items'], $locked->code);
            [$subtotal, $fee, $total] = $this->totals($lines, $payload['method'], $locked);
            if ($changedClient) {
                $locked->client_id = $client->id;
                $locked->client_code = $client->code;
                $locked->client_name = $client->user->name;
                $locked->client_email = $client->user->email;
            }
            $locked->fill([
                'method' => $payload['method'],
                'address' => $payload['address'],
                'notes' => $payload['notes'],
                'subtotal_cents' => $subtotal,
                'fee_cents' => $fee,
                'total_cents' => $total,
            ])->save();
            $this->replaceItems($locked, $lines);
            if ($locked->status === 'Confirmed') {
                $this->syncHolds($locked, $payload['items']);
            }
            $this->history($locked, $actor, 'Order details updated.');
            $locked->unsetRelations();

            return $locked;
        });
    }

    /** Coordinate explicit transitions, reservations, and once-only pickup completion. */
    public function action(Order $order, array $input, User $actor): Order
    {
        return DB::transaction(function () use ($order, $input, $actor): Order {
            $locked = Order::query()->lockForUpdate()->findOrFail($order->id);
            $locked->load(['items', 'payment']);
            $desired = $input['status'];
            if ($desired === 'Confirmed' && $actor->role !== User::ROLE_ADMIN) {
                abort(403, 'Only an administrator can confirm an order.');
            }
            if ($locked->status === $desired) {
                return $locked;
            }
            $this->assertVersion($locked, $input['version']);
            if (in_array($locked->status, ['Completed', 'Cancelled'], true)) {
                $this->invalid('status', 'Completed and cancelled orders cannot change status.');
            }
            $next = $this->nextStatus($locked);
            if ($desired !== 'Cancelled' && $desired !== $next) {
                $this->invalid('status', 'This transition is not available from the current order state.');
            }
            $flowers = $this->lockFlowers($locked->items->pluck('flower_id')->all());
            $quantities = $locked->items->mapWithKeys(fn(OrderItem $item) => [(int) $item->flower_id => (int) $item->quantity])->all();
            $note = 'Order marked ' . $desired . '.';
            if ($desired === 'Cancelled') {
                $reason = trim($input['reason'] ?? '');
                if ($reason === '') {
                    $this->invalid('reason', 'Explain why this order is cancelled.');
                }
                InventoryReservation::query()->where('reference', $locked->code)->where('status', 'Active')->update(['status' => 'Released']);
                $note = $reason;
            } elseif ($desired === 'Confirmed') {
                $this->assertAvailability($flowers, $quantities, $locked->code);
                $this->syncHolds($locked, $quantities);
            } elseif ($desired === 'Completed') {
                if ($locked->method !== 'Customer Pickup' || !filter_var($input['collected'] ?? false, FILTER_VALIDATE_BOOLEAN)) {
                    $this->invalid('collected', 'Confirm actual pickup collection before completing this order.');
                }
                if ($locked->stocked_out) {
                    $this->invalid('order', 'This order has already issued stock. Refresh its current state.');
                }
                foreach ($locked->items->sortBy('flower_id') as $item) {
                    $reservation = InventoryReservation::query()->where('flower_id', $item->flower_id)
                        ->where('reference', $locked->code)->lockForUpdate()->first();
                    if ($reservation === null || $reservation->status !== 'Active' || (int) $reservation->quantity !== (int) $item->quantity) {
                        $this->invalid('order', 'The order reservation is missing or inconsistent.');
                    }
                    app(InventoryService::class)->fulfillOrderReservation($reservation, $item->fulfillment_key, $actor, $locked->code);
                }
                $locked->stocked_out = true;
                $locked->completed_at = now();
                $sale = SalesRecord::create([
                    'order_id' => $locked->id,
                    'client_id' => $locked->client_id,
                    'code' => 'TMP-' . Str::ulid(),
                    'client_name' => $locked->client_name,
                    'client_code' => $locked->client_code,
                    'items' => $locked->items->map(fn(OrderItem $item) => $this->itemRow($item))->values()->all(),
                    'subtotal_cents' => $locked->subtotal_cents,
                    'fee_cents' => $locked->fee_cents,
                    'total_cents' => $locked->total_cents,
                    'completed_at' => $locked->completed_at,
                ]);
                $sale->code = 'SALE-' . str_pad((string) $sale->id, 3, '0', STR_PAD_LEFT);
                $sale->save();
                $note = 'Pickup collected; reserved stock issued and sale recorded.';
            }
            $locked->status = $desired;
            $locked->save();
            $this->history($locked, $actor, $note);

            if ($desired === 'Ready for Delivery') {
                app(DeliveryService::class)->createForOrder($locked, $actor);
            }

            $locked->unsetRelations();

            return $locked;
        });
    }

    /** Expose a credential-free order DTO with immutable item snapshots and exact cents. */
    public function row(Order $order, User $actor): array
    {
        $order->loadMissing(['items', 'histories', 'payment']);
        $payment = $order->payment;
        return [
            'dbId' => (int) $order->id,
            'id' => $order->code,
            'clientDbId' => (int) $order->client_id,
            'clientId' => $order->client_code,
            'clientName' => $order->client_name,
            'date' => $order->created_at->copy()->setTimezone('Asia/Manila')->format('Y-m-d'),
            'method' => $order->method,
            'address' => $order->address ?? '',
            'notes' => $order->notes ?? '',
            'status' => $order->status,
            'payment' => $payment?->status === 'Refunded' ? 'Refunded' : ($payment?->status === 'Confirmed' ? 'Paid' : 'Unpaid'),
            'paidCents' => in_array($payment?->status, ['Confirmed', 'Refunded'], true) ? (int) $payment->amount_cents : 0,
            'subtotalCents' => (int) $order->subtotal_cents,
            'deliveryFeeCents' => (int) $order->fee_cents,
            'totalCents' => (int) $order->total_cents,
            'stockedOut' => (bool) $order->stocked_out,
            'version' => $this->version($order),
            'items' => $order->items->sortBy('flower_id')->map(fn(OrderItem $item) => $this->itemRow($item))->values()->all(),
            'history' => $order->histories->sortBy('id')->map(fn(OrderHistory $entry) => [
                'status' => $entry->status,
                'date' => $entry->created_at->copy()->setTimezone('Asia/Manila')->toIso8601String(),
                'actor' => $entry->actor_name,
                'note' => $entry->note,
            ])->values()->all(),
            'canConfirm' => $actor->role === User::ROLE_ADMIN && $order->status === 'Pending',
            'editable' => in_array($order->status, ['Pending', 'Confirmed'], true) && $payment === null,
            'nextStatus' => $order->status === 'Pending' && $actor->role !== User::ROLE_ADMIN ? null : $this->nextStatus($order),
        ];
    }

    /** Read the explicitly configured fee; null means new Delivery orders are unavailable. */
    public function configuredDeliveryFee(): ?int
    {
        $value = config('orders.delivery_fee');
        if ($value === null || $value === '') {
            return null;
        }
        try {
            if (!is_string($value) && !is_int($value)) {
                throw new InvalidArgumentException('Fee must be an exact decimal string.');
            }
            return Money::parse((string) $value);
        } catch (InvalidArgumentException $exception) {
            $this->invalid('method', 'The delivery fee configuration is invalid. Ask an administrator to configure an approved decimal amount.');
        }
    }

    /** Normalize duplicate lines before hashing, pricing, and availability checks. */
    private function normalize(array $input): array
    {
        $items = [];
        foreach ($input['items'] as $line) {
            $id = (int) $line['flower_id'];
            $quantity = (int) $line['quantity'];
            if ($quantity < 1 || $quantity > 100000) {
                $this->invalid('items', 'Each requested quantity must be from 1 to 100000.');
            }
            $items[$id] = ($items[$id] ?? 0) + $quantity;
            if ($items[$id] > 100000) {
                $this->invalid('items', 'The combined quantity for each catalog item must not exceed 100000.');
            }
        }
        if (count($input['items']) < 1 || count($input['items']) > 50) {
            $this->invalid('items', 'Provide from one to fifty item rows.');
        }
        ksort($items, SORT_NUMERIC);
        $method = $input['method'];
        if (!in_array($method, ['Customer Pickup', 'Delivery'], true)) {
            $this->invalid('method', 'Select Customer Pickup or Delivery.');
        }
        $address = $method === 'Delivery' ? trim($input['address'] ?? '') : null;
        if ($method === 'Delivery' && $address === '') {
            $this->invalid('address', 'A delivery address is required.');
        }
        $notes = trim($input['notes'] ?? '');
        return [
            'client_id' => (int) $input['client_id'],
            'method' => $method,
            'address' => $address,
            'notes' => $notes === '' ? null : $notes,
            'items' => $items
        ];
    }

    /** Lock client account first, then its profile, matching Client Management. */
    private function lockClient(int $id, bool $requireActive, bool $validate = true): Client
    {
        $profile = Client::query()->findOrFail($id);
        $user = User::query()->lockForUpdate()->findOrFail($profile->user_id);
        $client = Client::query()->lockForUpdate()->findOrFail($id);
        if ((int) $client->user_id !== (int) $user->id) {
            $this->invalid('client_id', 'The client account association changed. Refresh before retrying.');
        }
        $client->setRelation('user', $user);
        if ($validate) {
            $this->assertClient($client, $requireActive);
        }
        return $client;
    }

    /** Validate a locked account only after checking for a completed creation replay. */
    private function assertClient(Client $client, bool $requireActive): void
    {
        if ($client->user->role !== User::ROLE_CLIENT) {
            $this->invalid('client_id', 'Select a valid client account.');
        }
        if ($requireActive && !$client->user->is_active) {
            $this->invalid('client_id', 'Only active clients can be selected for a new order.');
        }
    }

    /** Lock the full old/new item union in ascending order before touching stock or holds. */
    private function lockFlowers(array $ids): Collection
    {
        $ids = array_values(array_unique(array_map('intval', $ids)));
        sort($ids, SORT_NUMERIC);
        $rows = Flower::query()->whereIn('id', $ids)->orderBy('id')->lockForUpdate()->get()->keyBy('id');
        if ($rows->count() !== count($ids)) {
            $this->invalid('items', 'A requested catalog item no longer exists.');
        }
        return $rows;
    }

    /** Preserve existing line snapshots; new selections use active Sales-owned current prices. */
    private function lines(Collection $flowers, array $quantities, Collection $old): array
    {
        $prices = FlowerPrice::query()->whereIn('flower_id', array_keys($quantities))->lockForUpdate()->get()->keyBy('flower_id');
        $lines = [];
        foreach ($quantities as $id => $quantity) {
            $flower = $flowers->get($id);
            $previous = $old->get($id);
            if ($flower->status !== 'Active' && ($previous === null || $quantity > $previous->quantity)) {
                $this->invalid('items', 'Inactive items may only retain or reduce an existing line quantity.');
            }
            $price = $previous !== null ? (int) $previous->unit_price_cents : (int) ($prices->get($id)?->price_cents ?? 0);
            if ($price < 1 || $price > Money::MAX_CENTS || $quantity > intdiv(Money::MAX_CENTS, $price)) {
                $this->invalid('items', 'A valid selling price is required and item totals must fit the supported monetary range.');
            }
            $lines[] = [
                'flower_id' => $id,
                'flower_code' => $previous?->flower_code ?? $flower->code,
                'name' => $previous?->name ?? $flower->name,
                'product_type' => $previous?->product_type ?? $flower->product_type,
                'base_unit' => $previous?->base_unit ?? $flower->product_type,
                'quantity' => $quantity,
                'unit_price_cents' => $price,
                'subtotal_cents' => $price * $quantity,
                'fulfillment_key' => $previous?->fulfillment_key ?? (string) Str::uuid(),
            ];
        }
        return $lines;
    }

    /** Compute bounded integer monetary totals while preserving existing Delivery fee snapshots. */
    private function totals(array $lines, string $method, ?Order $old = null): array
    {
        $subtotal = 0;
        foreach ($lines as $line) {
            if ($subtotal > Money::MAX_CENTS - $line['subtotal_cents']) {
                $this->invalid('items', 'The order exceeds the supported monetary range.');
            }
            $subtotal += $line['subtotal_cents'];
        }
        $fee = $method === 'Customer Pickup' ? 0 : ($old?->method === 'Delivery' ? (int) $old->fee_cents : $this->configuredDeliveryFee());
        if ($fee === null) {
            $this->invalid('method', 'Delivery orders are unavailable until the delivery fee is configured.');
        }
        if ($subtotal > Money::MAX_CENTS - $fee) {
            $this->invalid('items', 'The order total exceeds the supported monetary range.');
        }
        return [$subtotal, $fee, $subtotal + $fee];
    }

    /** Use current locking reads even when earlier account reads created a repeatable-read snapshot. */
    private function assertAvailability(Collection $flowers, array $quantities, ?string $ownReference = null): void
    {
        $ids = $flowers->keys()->all();
        $batches = InventoryBatch::query()->whereIn('flower_id', $ids)->where('spoiled', false)->lockForUpdate()->get();
        $holds = InventoryReservation::query()->whereIn('flower_id', $ids)->where('status', 'Active')->lockForUpdate()->get();
        foreach ($quantities as $id => $quantity) {
            $physical = (int) $batches->where('flower_id', $id)->sum('remaining_quantity');
            $reserved = (int) $holds->where('flower_id', $id)->filter(fn($hold) => $hold->reference !== $ownReference)->sum('quantity');
            if ($quantity > $physical - $reserved) {
                $this->invalid('items', 'Insufficient stock is available for one or more requested items.');
            }
        }
    }

    /** Replace editable lines only; immutable snapshots and fulfillment UUIDs are retained per item. */
    private function replaceItems(Order $order, array $lines): void
    {
        $order->items()->delete();
        foreach ($lines as $line) {
            $order->items()->create($line);
        }
        $order->unsetRelation('items');
    }

    /** Rebalance one active reservation per order/item, releasing removed lines. */
    private function syncHolds(Order $order, array $quantities): void
    {
        $existing = InventoryReservation::query()->where('reference', $order->code)->lockForUpdate()->get();
        foreach ($existing as $hold) {
            if (!isset($quantities[$hold->flower_id])) {
                $hold->status = 'Released';
                $hold->save();
            }
        }
        foreach ($quantities as $id => $quantity) {
            InventoryReservation::updateOrCreate(['flower_id' => $id, 'reference' => $order->code], [
                'quantity' => $quantity,
                'status' => 'Active',
            ]);
        }
    }

    /** Determine only transitions owned by this module; Delivery completion is intentionally absent. */
    private function nextStatus(Order $order): ?string
    {
        return match ($order->status) {
            'Pending' => 'Confirmed', 'Confirmed' => 'Processing',
            'Processing' => $order->method === 'Customer Pickup' ? 'Ready for Pickup' : 'Ready for Delivery',
            'Ready for Pickup' => $order->method === 'Customer Pickup' ? 'Completed' : null,
            default => null,
        };
    }

    /** Fingerprint public business state plus the current payment record, excluding credentials. */
    private function version(Order $order): string
    {
        $order->loadMissing(['items', 'payment']);
        return hash('sha256', json_encode([
            'order' => $order->only([
                'id',
                'code',
                'client_id',
                'client_code',
                'client_name',
                'client_email',
                'method',
                'address',
                'notes',
                'status',
                'subtotal_cents',
                'fee_cents',
                'total_cents',
                'stocked_out',
                'completed_at'
            ]),
            'items' => $order->items->sortBy('flower_id')->map(fn($item) => $item->only([
                'flower_id',
                'flower_code',
                'name',
                'product_type',
                'base_unit',
                'quantity',
                'unit_price_cents',
                'subtotal_cents',
                'fulfillment_key',
            ]))->values()->all(),
            'payment' => $order->payment?->only(['id', 'code', 'amount_cents', 'status', 'method', 'reference', 'remarks', 'confirmed_by', 'confirmed_at']),
        ], JSON_THROW_ON_ERROR));
    }

    /** Reject an edit/action issued against stale business state. */
    private function assertVersion(Order $order, string $version): void
    {
        if (!hash_equals($this->version($order), strtolower($version))) {
            $this->invalid('version', 'This order changed. Refresh and review its latest state before saving.');
        }
    }

    /** Return the existing request's order only when actor and canonical payload match. */
    private function replay(Order $order, string $hash): Order
    {
        if (!hash_equals($order->request_hash, $hash)) {
            $this->invalid('request_key', 'This creation request key was already used for different order details.');
        }
        return $order;
    }

    /** Expose the immutable commercial item snapshot in exact integer cents. */
    private function itemRow(OrderItem $item): array
    {
        return [
            'flowerDbId' => (int) $item->flower_id,
            'flowerId' => $item->flower_code,
            'flowerName' => $item->name,
            'itemType' => $item->product_type,
            'baseUnit' => $item->base_unit,
            'quantity' => (int) $item->quantity,
            'unitPriceCents' => (int) $item->unit_price_cents,
            'subtotalCents' => (int) $item->subtotal_cents
        ];
    }

    /** Append workflow history with an actor-name snapshot. */
    private function history(Order $order, User $actor, string $note): void
    {
        OrderHistory::create([
            'order_id' => $order->id,
            'status' => $order->status,
            'note' => Str::limit($note, 1000, ''),
            'actor_id' => $actor->id,
            'actor_name' => Str::limit($actor->name, 255, '')
        ]);
    }

    /** Surface domain failures using standard JSON validation errors. */
    private function invalid(string $field, string $message): never
    {
        throw ValidationException::withMessages([$field => $message]);
    }
}
```

## `app/Services/SalesPaymentService.php`

```php
<?php

namespace App\Services;

use App\Models\FinanceOperation;
use App\Models\Flower;
use App\Models\FlowerPrice;
use App\Models\Order;
use App\Models\OrderPayment;
use App\Models\PaymentRefund;
use App\Models\SalesRecord;
use App\Models\User;
use App\Support\Money;
use Illuminate\Database\QueryException;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Validator;
use Illuminate\Support\Str;
use Illuminate\Validation\Rule;
use Illuminate\Validation\ValidationException;

class SalesPaymentService
{
    public const METHODS = ['Cash', 'GCash', 'Bank Transfer'];

    /** Share strict HTTP/service boundaries without accepting client-owned totals or states. */
    public static function rules(string $kind): array
    {
        $rules = [
            'request_key' => ['required', 'uuid'],
            'version' => ['required', 'string', 'size:64', 'regex:/\A[a-f0-9]{64}\z/'],
        ];
        $amount = ['required', 'string', 'max:16', 'regex:/\A\d{1,10}(?:\.\d{1,2})?\z/'];
        $reason = ['required', 'string', 'max:500'];
        $rules += match ($kind) {
            'price' => ['amount' => $amount],
            'record' => [
                'order_id' => ['required', 'integer', 'min:1'],
                'method' => ['required', Rule::in(self::METHODS)],
                'amount' => $amount,
                'reference' => ['nullable', 'string', 'max:100', 'required_unless:method,Cash'],
                'remarks' => ['nullable', 'string', 'max:500'],
            ],
            'confirm' => ['reason' => $reason],
            'refund' => ['amount' => $amount, 'reason' => $reason],
            default => throw new \InvalidArgumentException('Unknown finance operation.'),
        };
        foreach (['id', 'code', 'status', 'payment_status', 'paid', 'paidCents', 'amount_cents',
            'price_cents', 'total', 'total_cents', 'subtotal_cents', 'fee_cents', 'confirmed_by',
            'confirmed_at', 'actor_id', 'stocked_out', 'remaining_quantity', 'created_at'] as $field) {
            $rules[$field] = ['prohibited'];
        }
        return $rules;
    }

    /** Load actual prices, commercial snapshots, receipts and append-only refunds. */
    public function data(User $actor): array
    {
        $this->authorize($actor);
        return DB::transaction(function () use ($actor) {
            $prices = FlowerPrice::all()->keyBy('flower_id');
            $orders = Order::with(['items', 'payment'])->orderByDesc('id')->get();
            $ordersById = $orders->keyBy('id');
            $payments = OrderPayment::orderByDesc('id')->get();
            $paymentsById = $payments->keyBy('id');
            $refunds = PaymentRefund::orderBy('id')->get();
            $sums = [];
            foreach ($refunds as $refund) {
                $id = (int) $refund->payment_id;
                $sums[$id] = ($sums[$id] ?? 0) + (int) $refund->amount_cents;
            }
            $orderRows = $orders->map(fn (Order $order) => [
                'dbId' => (int) $order->id, 'id' => $order->code, 'client' => $order->client_name,
                'status' => $order->status, 'method' => $order->method,
                'items' => $order->items->map(fn ($item) => $this->item($item))->values()->all(),
                'deliveryFee' => (int) $order->fee_cents,
                'version' => $this->orderVersion($order),
            ])->all();

            return [
                'flowers' => Flower::orderBy('id')->get()->map(function (Flower $flower) use ($prices) {
                    $price = $prices->get($flower->id);
                    return [
                        'dbId' => (int) $flower->id, 'id' => $flower->code, 'name' => $flower->name,
                        'category' => $flower->category, 'itemType' => $flower->product_type,
                        'baseUnit' => $flower->product_type === 'Bouquet' ? 'Bouquet' : 'Stem',
                        'status' => $flower->status, 'canEdit' => $flower->status === 'Active',
                        'price' => $price ? (int) $price->price_cents : null,
                        'updated' => $this->time($price?->updated_at),
                        'version' => $this->priceVersion($flower, $price),
                    ];
                })->all(),
                'orders' => $orderRows,
                'sales' => SalesRecord::orderByDesc('id')->get()->map(function (SalesRecord $sale) use ($ordersById) {
                    return [
                        'dbId' => (int) $sale->id, 'id' => $sale->code,
                        'orderId' => $ordersById->get($sale->order_id)?->code,
                        'client' => $sale->client_name, 'date' => $this->time($sale->completed_at),
                        'gross' => (int) $sale->total_cents,
                        'items' => collect($sale->items)->map(fn ($item) => [
                            'flowerId' => $item['flowerId'], 'name' => $item['flowerName'],
                            'itemType' => $item['itemType'], 'baseUnit' => $item['baseUnit'],
                            'quantity' => (int) $item['quantity'], 'unitPrice' => (int) $item['unitPriceCents'],
                        ])->all(),
                    ];
                })->all(),
                'payments' => $payments->map(function (OrderPayment $payment) use ($ordersById, $sums) {
                    $order = $ordersById->get($payment->order_id);
                    return [
                        'dbId' => (int) $payment->id, 'id' => $payment->code,
                        'orderId' => $order?->code, 'client' => $order?->client_name,
                        'method' => $payment->method, 'reference' => $payment->reference ?? '',
                        'amount' => (int) $payment->amount_cents, 'status' => $payment->status,
                        'date' => $this->time($payment->created_at), 'confirmedAt' => $this->time($payment->confirmed_at),
                        'version' => $this->paymentVersion($payment, $order, $sums[$payment->id] ?? 0),
                    ];
                })->all(),
                'refunds' => $refunds->map(function (PaymentRefund $refund) use ($paymentsById, $ordersById) {
                    $payment = $paymentsById->get($refund->payment_id);
                    return [
                        'dbId' => (int) $refund->id, 'id' => $refund->code,
                        'paymentId' => $payment?->code,
                        'orderId' => $ordersById->get($payment?->order_id)?->code,
                        'amount' => (int) $refund->amount_cents,
                        'date' => $this->time($refund->created_at), 'reason' => $refund->reason,
                    ];
                })->all(),
                'history' => FinanceOperation::whereNotNull('order_id')->orderBy('id')->get()
                    ->map(fn (FinanceOperation $operation) => [
                        'orderId' => $ordersById->get($operation->order_id)?->code,
                        'note' => $operation->metadata['note'] ?? $operation->kind,
                        'actor' => $operation->metadata['actor_name'] ?? 'Recorded user',
                        'date' => $this->time($operation->created_at),
                    ])->all(),
                'permissions' => ['managePrices' => true, 'recordPayment' => true, 'confirm' => true,
                    'refund' => $actor->role === User::ROLE_ADMIN],
            ];
        });
    }

    /** Price changes affect new order lines only; existing snapshots remain unchanged. */
    public function price(Flower $flower, array $input, User $actor): array
    {
        $values = $this->validate('price', $input);
        $amount = $this->positive($values['amount']);
        $payload = ['flower_id' => (int) $flower->id, 'amount' => $amount];
        return $this->write('price', $values, $payload, $actor,
            function () use ($flower) {
                $locked = Flower::query()->lockForUpdate()->findOrFail($flower->id);
                $price = FlowerPrice::where('flower_id', $locked->id)->lockForUpdate()->first();
                return ['flower' => $locked, 'price' => $price];
            },
            function (array $context, FinanceOperation $operation) use ($values, $amount) {
                $flower = $context['flower']; $price = $context['price'];
                $this->assertVersion($this->priceVersion($flower, $price), $values['version']);
                if ($flower->status !== 'Active') $this->invalid('flower', 'Only active catalog items can be priced.');
                $before = $price ? (int) $price->price_cents : null;
                $price ??= new FlowerPrice(['flower_id' => $flower->id]);
                $price->price_cents = $amount;
                $price->save();
                $operation->result_id = $price->id;
                return ['note' => $flower->code.' price changed from '.($before === null ? 'not set' : Money::format($before)).' to '.Money::format($amount).'.'];
            });
    }

    /** Record exactly one full Pending payment; never issue stock or create a sale. */
    public function recordPayment(array $input, User $actor): array
    {
        $values = $this->validate('record', $input);
        $amount = $this->positive($values['amount']);
        $reference = trim($values['reference'] ?? '');
        if ($values['method'] !== 'Cash' && $reference === '') $this->invalid('reference', 'Enter the actual payment reference.');
        $remarks = trim($values['remarks'] ?? '');
        $payload = ['order_id' => (int) $values['order_id'], 'method' => $values['method'],
            'amount' => $amount, 'reference' => Str::lower($reference), 'remarks' => $remarks];
        $referenceKey = $values['method'] === 'Cash' ? null
            : hash('sha256', json_encode([$values['method'], Str::lower($reference)], JSON_THROW_ON_ERROR));

        return $this->write('record', $values, $payload, $actor,
            function () use ($values) {
                $order = Order::query()->lockForUpdate()->findOrFail($values['order_id']);
                $order->load('items');
                $payment = OrderPayment::where('order_id', $order->id)->lockForUpdate()->first();
                $order->setRelation('payment', $payment);
                return ['order' => $order, 'payment' => $payment];
            },
            function (array $context, FinanceOperation $operation) use ($values, $amount, $reference, $remarks) {
                $order = $context['order'];
                $this->assertVersion($this->orderVersion($order), $values['version']);
                if ($order->status === 'Cancelled') $this->invalid('order', 'Cancelled orders cannot receive a new payment.');
                if ($context['payment'] !== null) $this->invalid('order', 'This order already has a payment record. Additional payments are not allowed.');
                if ((int) $order->total_cents !== $amount) $this->invalid('amount', 'Payment must equal the full order total. Partial and excess payments are not allowed.');
                if ($values['method'] !== 'Cash' && OrderPayment::query()
                    ->where('method', $values['method'])
                    ->whereRaw('LOWER(TRIM(reference)) = ?', [Str::lower($reference)])
                    ->lockForUpdate()->first() !== null) {
                    $this->invalid('reference', 'This payment reference is already recorded for the selected method.');
                }
                $payment = OrderPayment::create([
                    'code' => 'TMP-'.Str::ulid(), 'order_id' => $order->id,
                    'method' => $values['method'], 'amount_cents' => $amount, 'status' => 'Pending',
                    'reference' => $reference === '' ? null : $reference,
                    'remarks' => $remarks === '' ? null : $remarks,
                ]);
                $payment->code = 'PAY-'.str_pad((string) $payment->id, 3, '0', STR_PAD_LEFT);
                $payment->save();
                $operation->payment_id = $payment->id;
                $operation->result_id = $payment->id;
                return ['note' => $payment->code.': '.Money::format($amount).' recorded as Pending via '.$payment->method.'.'];
            }, $referenceKey);
    }

    /** Confirm only after verifying the actual receipt/transfer/cash collection. */
    public function confirm(OrderPayment $payment, array $input, User $actor): array
    {
        $values = $this->validate('confirm', $input);
        $reason = $this->reason($values['reason']);
        $payload = ['payment_id' => (int) $payment->id, 'reason' => $reason];
        return $this->write('confirm', $values, $payload, $actor,
            fn () => $this->lockPayment($payment),
            function (array $context, FinanceOperation $operation) use ($values, $reason, $actor) {
                $payment = $context['payment']; $order = $context['order'];
                $sum = $this->refundTotal($payment, true);
                $this->assertVersion($this->paymentVersion($payment, $order, $sum), $values['version']);
                if ($order->status === 'Cancelled' || $payment->status !== 'Pending' || $sum !== 0) {
                    $this->invalid('payment', 'Only an eligible Pending payment can be confirmed.');
                }
                if ((int) $payment->amount_cents !== (int) $order->total_cents) {
                    $this->invalid('amount', 'The receipt must match the complete order total.');
                }
                $payment->status = 'Confirmed';
                $payment->confirmed_by = $actor->id;
                $payment->confirmed_at = now();
                $payment->save();
                $operation->result_id = $payment->id;
                return ['note' => $payment->code.' confirmed. '.$reason];
            });
    }

    /** Record an Admin-approved refund; preserve original receipt and all sale/stock history. */
    public function refund(OrderPayment $payment, array $input, User $actor): array
    {
        $this->authorize($actor);
        abort_unless($actor->role === User::ROLE_ADMIN, 403);
        $values = $this->validate('refund', $input);
        $amount = $this->positive($values['amount']);
        $reason = $this->reason($values['reason']);
        $payload = ['payment_id' => (int) $payment->id, 'amount' => $amount, 'reason' => $reason];
        return $this->write('refund', $values, $payload, $actor,
            fn () => $this->lockPayment($payment),
            function (array $context, FinanceOperation $operation) use ($values, $amount, $reason, $actor) {
                $payment = $context['payment']; $order = $context['order'];
                $sum = $this->refundTotal($payment, true);
                $this->assertVersion($this->paymentVersion($payment, $order, $sum), $values['version']);
                if (!in_array($payment->status, ['Confirmed', 'Refunded'], true) || !$payment->confirmed_at) {
                    $this->invalid('payment', 'Only a previously confirmed payment can be refunded.');
                }
                if ($amount > (int) $payment->amount_cents - $sum) {
                    $this->invalid('amount', 'Refund exceeds the remaining refundable amount.');
                }
                $refund = PaymentRefund::create([
                    'code' => 'TMP-'.Str::ulid(), 'payment_id' => $payment->id,
                    'amount_cents' => $amount, 'reason' => $reason,
                    'actor_id' => $actor->id, 'actor_name' => $actor->name,
                ]);
                $refund->code = 'REF-'.str_pad((string) $refund->id, 3, '0', STR_PAD_LEFT);
                $refund->save();
                // Any refund is visible to existing Order and Client financial status DTOs.
                $payment->status = 'Refunded';
                $payment->save();
                $operation->result_id = $refund->id;
                return ['note' => $refund->code.': '.Money::format($amount).' refunded against '.$payment->code.'. '.$reason];
            });
    }

    /** Serialize by Order or Flower before testing UUID replay and performing any write. */
    private function write(string $kind, array $input, array $payload, User $actor,
        callable $lock, callable $work, ?string $referenceKey = null): array
    {
        $this->authorize($actor);
        $hash = hash('sha256', json_encode([
            'kind' => $kind, 'actor_id' => (int) $actor->id, 'payload' => $payload,
        ], JSON_THROW_ON_ERROR));
        try {
            DB::transaction(function () use ($kind, $input, $actor, $hash, $lock, $work, $referenceKey) {
                $context = $lock();
                $previous = FinanceOperation::where('request_key', $input['request_key'])->lockForUpdate()->first();
                if ($previous) {
                    if (!hash_equals($previous->request_hash, $hash)) $this->invalid('request_key', 'This request key was already used with different details.');
                    return;
                }
                $operation = FinanceOperation::create([
                    'request_key' => $input['request_key'], 'request_hash' => $hash,
                    'reference_key' => $referenceKey, 'kind' => $kind, 'actor_id' => $actor->id,
                    'order_id' => isset($context['order']) ? $context['order']->id : null,
                    'payment_id' => isset($context['payment']) ? $context['payment']->id : null,
                    'flower_id' => isset($context['flower']) ? $context['flower']->id : null,
                ]);
                $operation->metadata = ['actor_name' => $actor->name, ...$work($context, $operation)];
                $operation->save();
            }, 3);
        } catch (QueryException $exception) {
            if (!in_array((int) ($exception->errorInfo[1] ?? 0), [1062, 19], true)) throw $exception;
            $previous = FinanceOperation::where('request_key', $input['request_key'])->first();
            if ($previous) {
                if (!hash_equals($previous->request_hash, $hash)) $this->invalid('request_key', 'This request key was already used with different details.');
            } elseif ($referenceKey !== null && FinanceOperation::where('reference_key', $referenceKey)->exists()) {
                $this->invalid('reference', 'This payment reference is already recorded for the selected method.');
            } else {
                throw $exception;
            }
        }
        return $this->data($actor);
    }

    private function lockPayment(OrderPayment $payment): array
    {
        $order = Order::query()->lockForUpdate()->findOrFail($payment->order_id);
        $locked = OrderPayment::query()->lockForUpdate()->findOrFail($payment->id);
        abort_unless((int) $locked->order_id === (int) $order->id, 409);
        return ['order' => $order, 'payment' => $locked];
    }

    private function refundTotal(OrderPayment $payment, bool $lock = false): int
    {
        $query = PaymentRefund::where('payment_id', $payment->id)->orderBy('id');
        if ($lock) $query->lockForUpdate();
        return (int) $query->get()->sum('amount_cents');
    }

    private function priceVersion(Flower $flower, ?FlowerPrice $price): string
    {
        return $this->version(['flower', (int) $flower->id, $flower->status,
            $price ? (int) $price->id : null, $price ? (int) $price->price_cents : null,
            $price?->updated_at?->format('Y-m-d H:i:s')]);
    }

    private function orderVersion(Order $order): string
    {
        $order->loadMissing(['items', 'payment']);
        return $this->version(['order', (int) $order->id, $order->status, (int) $order->total_cents,
            $order->updated_at?->format('Y-m-d H:i:s'), $order->payment?->id,
            $order->items->sortBy('flower_id')->map(fn ($item) => $this->item($item))->values()->all()]);
    }

    private function paymentVersion(OrderPayment $payment, ?Order $order, int $refunded): string
    {
        return $this->version(['payment', (int) $payment->id, (int) $payment->order_id,
            $payment->status, (int) $payment->amount_cents, $payment->method, $payment->reference, $payment->remarks,
            $payment->confirmed_by === null ? null : (int) $payment->confirmed_by,
            $payment->confirmed_at?->format('Y-m-d H:i:s'),
            $payment->updated_at?->format('Y-m-d H:i:s'), $refunded,
            $order?->status, $order ? (int) $order->total_cents : null]);
    }

    private function version(array $values): string
    {
        return hash_hmac('sha256', json_encode($values, JSON_THROW_ON_ERROR), (string) config('app.key'));
    }

    private function assertVersion(string $current, string $expected): void
    {
        if (!hash_equals($current, $expected)) $this->invalid('version', 'Financial details changed. Refresh and reopen this action.');
    }

    private function item($item): array
    {
        return ['flowerId' => $item->flower_code, 'name' => $item->name,
            'itemType' => $item->product_type, 'baseUnit' => $item->base_unit,
            'quantity' => (int) $item->quantity, 'unitPrice' => (int) $item->unit_price_cents];
    }

    private function time($date): ?string
    {
        return $date?->copy()->setTimezone('Asia/Manila')->toIso8601String();
    }

    private function authorize(User $actor): void
    {
        abort_unless($actor->is_active && !$actor->must_change_password
            && in_array($actor->role, [User::ROLE_ADMIN, User::ROLE_SUPPLIER], true), 403);
    }

    private function validate(string $kind, array $input): array
    {
        if (isset($input['request_key']) && is_string($input['request_key'])) {
            $input['request_key'] = Str::lower($input['request_key']);
        }
        return Validator::make($input, self::rules($kind))->validate();
    }

    private function positive(string $decimal): int
    {
        try { $amount = Money::parse($decimal); }
        catch (\InvalidArgumentException $exception) { $this->invalid('amount', 'Enter a valid exact monetary amount with at most two decimal places.'); }
        if ($amount <= 0) $this->invalid('amount', 'Amount must be greater than zero.');
        return $amount;
    }

    private function reason(string $reason): string
    {
        $reason = trim($reason);
        if ($reason === '') $this->invalid('reason', 'Enter the verification or refund reason.');
        return $reason;
    }

    private function invalid(string $field, string $message): never
    {
        throw ValidationException::withMessages([$field => $message]);
    }
}
```

## `database/migrations/2026_11_07_090000_create_finance_tables.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /** Add financial retry/audit and refund ledgers; retain existing payment/sale tables. */
    public function up(): void
    {
        Schema::create('finance_operations', function (Blueprint $table) {
            $table->id();
            $table->uuid('request_key')->unique();
            $table->string('request_hash', 64);
            $table->string('reference_key', 64)->nullable()->unique();
            $table->string('kind', 30);
            $table->foreignId('actor_id')->nullable()->constrained('users')->nullOnDelete();
            $table->foreignId('order_id')->nullable()->constrained('orders')->restrictOnDelete();
            $table->foreignId('payment_id')->nullable()->constrained('order_payments')->restrictOnDelete();
            $table->foreignId('flower_id')->nullable()->constrained('flowers')->restrictOnDelete();
            $table->unsignedBigInteger('result_id')->nullable();
            $table->json('metadata')->nullable();
            $table->timestamps();
        });

        Schema::create('payment_refunds', function (Blueprint $table) {
            $table->id();
            $table->string('code', 30)->unique();
            $table->foreignId('payment_id')->constrained('order_payments')->restrictOnDelete();
            $table->unsignedBigInteger('amount_cents');
            $table->string('reason', 500);
            $table->foreignId('actor_id')->nullable()->constrained('users')->nullOnDelete();
            $table->string('actor_name', 255);
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('payment_refunds');
        Schema::dropIfExists('finance_operations');
    }
};
```

## `resources/js/supplier/sales-payments.js`

```javascript
import DataTable from 'datatables.net-bs5';
import { Modal } from 'bootstrap';

// Escape API text before inserting it into approved table and modal markup.
const escapeHtml = (value) =>
    String(value ?? '').replace(/[&<>"']/g, (character) => ({
        '&': '&amp;',
        '<': '&lt;',
        '>': '&gt;',
        '"': '&quot;',
        "'": '&#039;',
    })[character]);

const MAX_CENTS = 999999999999;

// Format integer centavos without losing fractional cents at large aggregate totals.
function money(centavos) {
    if (!Number.isSafeInteger(centavos)) throw new Error('Financial total exceeds safe integer precision.');
    const absolute = Math.abs(centavos);
    const whole = BigInt(Math.floor(absolute / 100));
    const signed = centavos < 0 ? (whole === 0n ? -0 : -whole) : whole;
    const fraction = String(absolute % 100).padStart(2, '0');
    return new Intl.NumberFormat('en-PH', {style: 'currency', currency: 'PHP'})
        .formatToParts(signed).map((part) => part.type === 'fraction' ? fraction : part.value).join('');
}

// Refuse unsafe addition instead of rounding an accumulated financial amount.
function addCents(total, value) {
    if (!Number.isSafeInteger(total) || !Number.isSafeInteger(value)
        || !Number.isSafeInteger(total + value)) throw new Error('Financial total exceeds safe integer precision.');
    return total + value;
}

// Multiply whole-unit quantities by integer-centavo snapshots with overflow checks.
function itemTotal(item) {
    if (!Number.isSafeInteger(item.quantity) || item.quantity < 1
        || !Number.isSafeInteger(item.unitPrice) || item.unitPrice < 0 || item.unitPrice > MAX_CENTS
        || !Number.isSafeInteger(item.quantity * item.unitPrice)) {
        throw new Error('Invalid or unsafe order-item amount.');
    }
    return item.quantity * item.unitPrice;
}

// Reject partial decimals, negative values and floating-point parsing shortcuts.
function parseMoney(value) {
    const text = String(value).trim();

    if (!/^\d{1,10}(\.\d{1,2})?$/.test(text)) return null;

    const [whole, decimal = ''] = text.split('.');
    const result = Number(whole) * 100 + Number(decimal.padEnd(2, '0'));

    return Number.isSafeInteger(result) && result > 0 && result <= MAX_CENTS ? result : null;
}

// Date-based summaries use the business timezone, including near midnight.
function dateKey(value) {
    const date = new Date(value);
    if (Number.isNaN(date.getTime())) return '';
    const parts = new Intl.DateTimeFormat('en-US', {timeZone: 'Asia/Manila', year: 'numeric', month: '2-digit', day: '2-digit'}).formatToParts(date);
    const part = (type) => parts.find((entry) => entry.type === type)?.value;
    return `${part('year')}-${part('month')}-${part('day')}`;
}

// Display server timestamps in the same timezone used for summary totals.
function dateTime(value) {
    if (!value) return '—';
    const date = new Date(value);
    if (Number.isNaN(date.getTime())) return '—';
    return new Intl.DateTimeFormat('en-PH', {timeZone: 'Asia/Manila', dateStyle: 'medium', timeStyle: 'short'}).format(date);
}

// Convert integer centavos to an exact decimal string for API writes.
function decimalMoney(centavos) {
    if (!Number.isSafeInteger(centavos) || centavos < 0) throw new Error('Invalid money amount.');
    return `${Math.floor(centavos / 100)}.${String(centavos % 100).padStart(2, '0')}`;
}

// Generate one idempotency key for each intended mutation, retained on retry.
function requestKey() {
    if (globalThis.crypto?.randomUUID) return globalThis.crypto.randomUUID();
    if (!globalThis.crypto?.getRandomValues) throw new Error('A secure browser context is required to record financial changes.');
    const bytes = globalThis.crypto.getRandomValues(new Uint8Array(16));
    bytes[6] = (bytes[6] & 15) | 64;
    bytes[8] = (bytes[8] & 63) | 128;
    const hex = [...bytes].map((value) => value.toString(16).padStart(2, '0')).join('');
    return `${hex.slice(0, 8)}-${hex.slice(8, 12)}-${hex.slice(12, 16)}-${hex.slice(16, 20)}-${hex.slice(20)}`;
}

// Render status labels without allowing markup from API values.
function badge(value) {
    return `
        <span class="sales-badge ${String(value).toLowerCase().replaceAll(' ', '-').replace(/[^a-z0-9-]/g, '')}">
            ${escapeHtml(value)}
        </span>
    `;
}

// Keep primary and secondary identifiers visually grouped and escaped.
function cell(primary, secondary = '') {
    return `
        <div class="sales-cell">
            <strong>${escapeHtml(primary)}</strong>
            ${secondary ? `<small>${escapeHtml(secondary)}</small>` : ''}
        </div>
    `;
}

class SalesPaymentsUI {
    // Initialize empty UI collections; database reads populate the tables.
    constructor() {
        this.page = document.querySelector('.sales-page');
        this.baseUrl = this.page.dataset.url.replace(/\/$/, '');
        this.csrf = this.page.dataset.csrf || document.querySelector('meta[name="csrf-token"]')?.content || '';
        this.prices = [];
        this.orders = [];
        this.payments = [];
        this.refunds = [];
        this.history = [];
        this.sales = [];
        this.permissions = {};
        this.tables = {};
        this.action = null;
        this.pendingOperation = null;
        this.submitting = false;
        this.loading = false;
        this.loaded = false;
        this.readPromise = null;
        this.actionModal = Modal.getOrCreateInstance(this.el('salesActionModal'));
        this.viewModal = Modal.getOrCreateInstance(this.el('salesViewModal'));
        this.initializeTables();
        this.bindEvents();
        this.loadData().catch((error) => this.pageError(error.message));
    }

    // Resolve an existing element by its stable UI ID.
    el(id) {
        return document.getElementById(id);
    }

    // Find a canonical order by its display code.
    order(id) {
        return this.orders.find((order) => order.id === id);
    }

    // Calculate the immutable item-price snapshots and delivery fee in centavos.
    orderTotal(order) {
        return addCents(order.items.reduce((sum, item) => addCents(sum, itemTotal(item)), 0), order.deliveryFee);
    }

    // Count confirmed receipts, including receipts later refunded.
    confirmedPaid(orderId) {
        return this.payments
            .filter((payment) =>
                payment.orderId === orderId && ['Confirmed', 'Refunded'].includes(payment.status),
            )
            .reduce((sum, payment) => addCents(sum, payment.amount), 0);
    }

    // Sum recorded refund amounts for an order.
    refunded(orderId) {
        return this.refunds
            .filter((refund) => refund.orderId === orderId)
            .reduce((sum, refund) => addCents(sum, refund.amount), 0);
    }

    // Sum receipts still awaiting manual verification.
    pendingPaid(orderId) {
        return this.payments
            .filter((payment) =>
                payment.orderId === orderId && payment.status === 'Pending',
            )
            .reduce((sum, payment) => addCents(sum, payment.amount), 0);
    }

    // Refunds take precedence; only full confirmed receipts mark an order Paid.
    paymentStatus(order) {
        if (this.refunded(order.id) > 0) return 'Refunded';

        const paid = this.confirmedPaid(order.id);

        if (!paid) return 'Unpaid';
        return paid === this.orderTotal(order) ? 'Paid' : 'Unpaid';
    }

    // Allow one full receipt only for non-cancelled orders without any prior receipt or refund.
    eligibleForPayment(order) {
        return Boolean(order)
            && order.status !== 'Cancelled'
            && this.refunded(order.id) === 0
            && !this.payments.some((payment) => payment.orderId === order.id);
    }

    // Render an accessible action button with escaped identifiers.
    icon(action, id, icon, label, extra = '') {
        return `
            <button
                type="button"
                class="sales-icon-btn ${extra}"
                data-finance-action="${action}"
                data-record-id="${escapeHtml(id)}"
                title="${escapeHtml(label)}"
                aria-label="${escapeHtml(label)}"
            >
                <i class="bi bi-${icon}" aria-hidden="true"></i>
            </button>
        `;
    }

    // Preserve native DataTables sorting and the approved financial table columns.
    initializeTables() {
        const moneyColumn = (getter) => ({
            data: null,
            className: 'sales-money',
            render: (_value, type, row) => {
                const amount = getter(row);
                return type === 'display' ? money(amount) : amount;
            },
        });

        this.tables.prices = this.makeTable('pricesTable', this.prices, [
            {
                data: 'name',
                width: '30%',
                render: (value, type, row) =>
                    type === 'display' ? cell(value, `${row.id} · ${row.baseUnit}`) : `${value} ${row.id} ${row.baseUnit}`,
            },
            {
                data: 'category',
                width: '20%',
                render: (value, type) => type === 'display' ? escapeHtml(value) : value,
            },
            {
                data: 'price', width: '18%', className: 'sales-money',
                render: (value, type) => type === 'display'
                    ? (value === null ? 'Not set' : money(value))
                    : (value ?? -1),
            },
            {
                data: 'updated',
                width: '22%',
                render: (value, type) => type === 'display' ? dateTime(value) : value,
            },
            {
                data: null,
                width: '10%',
                className: 'sales-center',
                orderable: false,
                searchable: false,
                render: (_value, type, row) =>
                    type === 'display'
                        ? (this.permissions.managePrices && row.canEdit !== false
                            ? this.icon('price', row.id, 'pencil-square', `Edit price ${row.name}`, 'edit') : '')
                        : '',
            },
        ]);

        this.tables.records = this.makeTable('recordsTable', this.sales, [
            {
                data: 'id',
                render: (value, type, row) =>
                    type === 'display' ? cell(value, row.orderId) : `${value} ${row.orderId}`,
            },
            {
                data: 'client',
                render: (value, type) => type === 'display' ? escapeHtml(value) : value,
            },
            {
                data: 'date',
                render: (value, type) => type === 'display' ? dateTime(value) : value,
            },
            moneyColumn((row) => row.gross),
            moneyColumn((row) => this.refunded(row.orderId)),
            moneyColumn((row) => row.gross - this.refunded(row.orderId)),
            {
                data: null,
                className: 'sales-center',
                orderable: false,
                searchable: false,
                render: (_value, type, row) =>
                    type === 'display'
                        ? this.icon('view', row.orderId, 'eye', `View ${row.id}`)
                        : '',
            },
        ], [[2, 'desc']]);

        this.tables.payments = this.makeTable('paymentsTable', this.payments, [
            {
                data: 'id',
                render: (value, type, row) =>
                    type === 'display' ? cell(value, row.orderId) : `${value} ${row.orderId}`,
            },
            {
                data: null,
                render: (_value, type, row) => {
                    const client = this.order(row.orderId)?.client || '';
                    return type === 'display' ? escapeHtml(client) : client;
                },
            },
            {
                data: 'date',
                render: (value, type) => type === 'display' ? dateTime(value) : value,
            },
            {
                data: 'method',
                render: (value, type, row) =>
                    type === 'display'
                        ? cell(value, row.reference || 'No reference required')
                        : `${value} ${row.reference}`,
            },
            moneyColumn((row) => row.amount),
            {
                data: 'status',
                className: 'sales-center',
                render: (value, type) => type === 'display' ? badge(value) : value,
            },
            {
                data: null,
                className: 'sales-center',
                orderable: false,
                searchable: false,
                render: (_value, type, row) => {
                    if (type !== 'display') return '';

                    return `
                        <div class="sales-actions">
                            ${this.icon('view', row.orderId, 'eye', `View ${row.id}`)}
                            ${this.permissions.confirm && row.status === 'Pending'
                                && this.order(row.orderId)?.status !== 'Cancelled'
                                ? this.icon('confirm', row.id, 'check-lg', `Confirm ${row.id}`, 'confirm')
                                : ''}
                            ${this.permissions.refund && ['Confirmed', 'Refunded'].includes(row.status)
                                && this.paymentRefundable(row) > 0
                                ? this.icon('refund', row.id, 'arrow-counterclockwise', `Refund ${row.id}`, 'refund')
                                : ''}
                        </div>
                    `;
                },
            },
        ], [[2, 'desc']]);
    }

    // Build one searchable table with compact controls and horizontal scrolling.
    makeTable(id, data, columns, order = [[0, 'asc']]) {
        const tableElement = this.el(id);

        const table = new DataTable(tableElement, {
            data,
            columns,
            order,
            autoWidth: false,
            pageLength: 5,
            layout: {
                topStart: null,
                topEnd: null,
                bottomStart: 'info',
                bottomEnd: 'paging',
            },
            language: {
                emptyTable: 'No records available.',
                zeroRecords: 'No matching records found.',
                info: 'Showing _START_ to _END_ of _TOTAL_ entries',
                infoEmpty: 'Showing 0 to 0 of 0 entries',
                infoFiltered: '(filtered from _MAX_ total entries)',
                paginate: {
                    first: '«',
                    previous: '‹',
                    next: '›',
                    last: '»',
                },
            },
        });

        const wrapper = document.createElement('div');
        wrapper.className = 'sales-table-scroll';
        wrapper.tabIndex = 0;
        wrapper.setAttribute('role', 'region');
        wrapper.setAttribute('aria-label', 'Financial table. Scroll horizontally for all columns.');

        tableElement.before(wrapper);
        wrapper.append(tableElement);

        return table;
    }

    // Connect tabs, compact search controls and manual financial actions.
    bindEvents() {
        document.querySelectorAll('[data-sales-tab]').forEach((button) => {
            button.addEventListener('click', () => this.switchTab(button.dataset.salesTab));

            button.addEventListener('keydown', (event) => {
                const tabs = [...document.querySelectorAll('[data-sales-tab]')];
                const index = tabs.indexOf(button);
                let next = index;

                if (event.key === 'ArrowRight') next = (index + 1) % tabs.length;
                else if (event.key === 'ArrowLeft') next = (index - 1 + tabs.length) % tabs.length;
                else if (event.key === 'Home') next = 0;
                else if (event.key === 'End') next = tabs.length - 1;
                else return;

                event.preventDefault();
                tabs[next].focus();
                this.switchTab(tabs[next].dataset.salesTab);
            });
        });

        document.querySelectorAll('[data-search]').forEach((input) => {
            input.addEventListener('input', () => {
                this.tables[input.dataset.search].search(input.value).draw();
            });
        });

        document.querySelectorAll('[data-length]').forEach((select) => {
            select.addEventListener('change', () => {
                this.tables[select.dataset.length].page.len(Number(select.value)).draw();
            });
        });

        document.querySelector('.sales-page').addEventListener('click', (event) => {
            const button = event.target.closest('[data-finance-action]');
            if (!button) return;

            if (button.dataset.financeAction === 'view') {
                this.showOrder(button.dataset.recordId);
            } else {
                this.openAction(button.dataset.financeAction, button.dataset.recordId);
            }
        });

        this.el('recordPaymentBtn').addEventListener('click', () => this.openAction('payment'));
        this.el('salesRetryRequest').addEventListener('click', () => this.retryRequest());
        this.el('salesReloadData').addEventListener('click', () => {
            this.loadData().catch((error) => this.pageError(error.message));
        });
        this.el('salesActionModal').addEventListener('hide.bs.modal', (event) => {
            if (this.submitting) event.preventDefault();
        });

        this.el('salesActionForm').addEventListener('submit', (event) => {
            event.preventDefault();
            this.saveAction();
        });

        this.el('salesActionFields').addEventListener('change', (event) => {
            if (event.target.id === 'financeMethod') this.syncReference();
            if (event.target.id === 'financeOrder') this.updateOrderHelp();
        });

        this.el('salesActionModal').addEventListener('hidden.bs.modal', () => {
            this.action = null;
            this.el('salesActionFields').innerHTML = '';
        });
    }

    // Update accessible tab state and recalculate the visible table widths.
    switchTab(key) {
        ['prices', 'records', 'payments'].forEach((name) => {
            const selected = key === name;
            const tab = this.el(`${name}Tab`);

            tab.classList.toggle('active', selected);
            tab.setAttribute('aria-selected', String(selected));
            tab.tabIndex = selected ? 0 : -1;
            this.el(`${name}Panel`).hidden = !selected;
        });

        this.tables[key].columns.adjust();
    }

    // Return the confirmed payment amount not already refunded.
    paymentRefundable(payment) {
        const refunded = this.refunds
            .filter((refund) => refund.paymentId === payment.id)
            .reduce((sum, refund) => addCents(sum, refund.amount), 0);

        return payment.amount - refunded;
    }

    // Show validation feedback safely inside the action modal.
    error(message = '') {
        this.el('salesActionError').textContent = message;
        this.el('salesActionError').hidden = !message;
    }

    // Wrap labelled modal input markup using the approved form styles.
    field(label, content) {
        return `<div class="sales-field">${label}${content}</div>`;
    }

    // Reload authoritative data and verify permission before opening a mutation form.
    async openAction(type, id = '') {
        if (this.submitting || this.pendingOperation) {
            this.announce('Resolve the pending request using Retry before starting another change.');
            return;
        }
        try { await this.loadData(); }
        catch (error) { this.pageError(error.message); return; }
        const permission = {price: 'managePrices', payment: 'recordPayment', confirm: 'confirm', refund: 'refund'}[type];
        if (!permission || !this.permissions[permission]) {
            this.announce('You do not have permission to perform this action.');
            return;
        }
        this.action = { type, id };
        this.error();

        let title = '';
        let fields = '';

        if (type === 'price') {
            const flower = this.prices.find((item) => item.id === id);
            if (!flower) return;

            title = `Edit Price — ${flower.name}`;

            fields = `
                <p class="sales-help">
                    Changes apply to future orders. Existing order
                    and Sales Record prices remain unchanged.
                </p>
                ${this.field(
                    `<label for="financeAmount">Selling Price / ${escapeHtml(flower.baseUnit)} *</label>`,
                    `<input type="number" id="financeAmount" min="0.01" max="9999999999.99"
                        step="0.01" value="${flower.price === null ? '' : decimalMoney(flower.price)}" required>`,
                )}
            `;
        }

        if (type === 'payment') {
            const eligible = this.orders.filter((order) => this.eligibleForPayment(order));

            if (!eligible.length) {
                this.announce('No unpaid orders are eligible for a full payment.');
                return;
            }

            title = 'Record Payment';

            fields = `
                ${this.field(
                    '<label for="financeOrder">Order *</label>',
                    `<select id="financeOrder" required>
                        ${eligible.map((order) => `
                            <option value="${order.id}">
                                ${escapeHtml(order.id)} · ${escapeHtml(order.client)}
                            </option>
                        `).join('')}
                    </select>`,
                )}
                <p id="financeOrderHelp" class="sales-help"></p>

                ${this.field(
                    '<label for="financeMethod">Payment Method *</label>',
                    `<select id="financeMethod" required>
                        <option value="Cash">Cash</option>
                        <option value="Bank Transfer">Bank Transfer</option>
                        <option value="GCash">GCash</option>
                    </select>`,
                )}

                ${this.field(
                    '<label for="financeAmount">Full Order Amount *</label>',
                    '<input type="number" id="financeAmount" step="0.01" readonly required>',
                )}

                <div class="sales-field" id="financeReferenceField" hidden>
                    <label for="financeReference">Reference Number *</label>
                    <input type="text" id="financeReference" maxlength="100">
                </div>

                <p class="sales-help">
                    One full payment per order. The amount is fixed to the complete order total.
                    Recording creates Pending verification; only confirmation marks the order Paid.
                </p>
            `;
        }

        if (type === 'confirm') {
            const payment = this.payments.find((item) => item.id === id);
            if (!payment || payment.status !== 'Pending'
                || this.order(payment.orderId)?.status === 'Cancelled') return;

            title = `Confirm ${payment.id}`;

            fields = `
                <p class="sales-help">
                    Confirm receipt of ${money(payment.amount)}
                    via ${escapeHtml(payment.method)}
                    for ${escapeHtml(payment.orderId)}?
                    Verify the actual receipt before confirming this payment.
                </p>
                ${this.field(
                    '<label for="financeReason">Verification Remarks *</label>',
                    '<textarea id="financeReason" rows="3" maxlength="500" required></textarea>',
                )}
            `;
        }

        if (type === 'refund') {
            const payment = this.payments.find((item) => item.id === id);

            if (!payment || !['Confirmed', 'Refunded'].includes(payment.status)
                || this.paymentRefundable(payment) <= 0) return;

            title = `Record Refund — ${payment.id}`;

            fields = `
                <p class="sales-help">
                    Refundable from this payment:
                    <strong>${money(this.paymentRefundable(payment))}</strong>.
                    Saving records an Admin-approved refund.
                    Original payment history is retained.
                </p>
                ${this.field(
                    '<label for="financeAmount">Refund Amount *</label>',
                    `<input type="number" id="financeAmount" min="0.01"
                        max="${decimalMoney(this.paymentRefundable(payment))}"
                        step="0.01" required>`,
                )}
                ${this.field(
                    '<label for="financeReason">Refund Reason *</label>',
                    '<textarea id="financeReason" rows="3" maxlength="500" required></textarea>',
                )}
                <p class="sales-help">
                    This records the refund only; it does not transfer money.
                    Further collection on refunded orders is blocked.
                </p>
            `;
        }

        this.el('salesActionTitle').textContent = title;
        this.el('salesActionFields').innerHTML = fields;

        this.el('salesActionSubmit').textContent = {
            price: 'Save Price',
            payment: 'Record Payment',
            confirm: 'Confirm Payment',
            refund: 'Approve & Record Refund',
        }[type];

        if (type === 'payment') {
            this.syncReference();
            this.updateOrderHelp();
        }

        this.actionModal.show();
    }

    // Require a reference for GCash and bank transfers, but not Cash.
    syncReference() {
        const online = this.el('financeMethod').value !== 'Cash';

        this.el('financeReferenceField').hidden = !online;
        this.el('financeReference').required = online;
        this.el('financeReference').disabled = !online;
    }

    // Fix the receipt amount to the selected order total; installments are excluded.
    updateOrderHelp() {
        const order = this.order(this.el('financeOrder').value);
        if (!order) return;

        this.el('financeAmount').value = decimalMoney(this.orderTotal(order));
        this.el('financeOrderHelp').textContent =
            `Full payment required: ${money(this.orderTotal(order))}. No installments or additional payments.`;
    }

    // Validate the intended mutation and send one versioned idempotent API request.
    async saveAction() {
        this.error();
        if (this.submitting || this.loading) return;
        if (this.pendingOperation) { await this.retryRequest(); return; }
        if (!this.action || !this.el('salesActionForm').reportValidity()) return;
        const { type, id } = this.action;
        const amount = this.el('financeAmount') ? parseMoney(this.el('financeAmount').value) : null;
        const reason = this.el('financeReason')?.value.trim() || '';
        if (['price', 'payment', 'refund'].includes(type) && amount === null) {
            this.error('Enter a positive amount with no more than two decimal places.'); return;
        }
        if (['confirm', 'refund'].includes(type) && !reason) {
            this.error('Enter the required remarks or reason.'); return;
        }
        const required = {price: 'managePrices', payment: 'recordPayment', confirm: 'confirm', refund: 'refund'}[type];
        if (!this.permissions[required]) { this.error('This action is not permitted.'); return; }
        let path;
        let payload;
        if (type === 'price') {
            const flower = this.prices.find((item) => item.id === id);
            if (!flower || flower.canEdit === false) { this.error('This price cannot be edited.'); return; }
            path = `/prices/${flower.dbId}`;
            payload = { version: flower.version, amount: decimalMoney(amount) };
        } else if (type === 'payment') {
            const order = this.order(this.el('financeOrder').value);
            const method = this.el('financeMethod').value;
            const reference = method === 'Cash' ? '' : this.el('financeReference').value.trim();
            if (!this.eligibleForPayment(order)) { this.error('This order is no longer eligible for payment.'); return; }
            if (!['Cash', 'GCash', 'Bank Transfer'].includes(method)) { this.error('Select a valid payment method.'); return; }
            if (method !== 'Cash' && !reference) { this.error('Enter the transfer or GCash reference number.'); return; }
            if (amount !== this.orderTotal(order)) { this.error('A single payment must match the full order total.'); return; }
            path = '/payments';
            payload = { order_id: order.dbId, version: order.version, method, reference, amount: decimalMoney(amount) };
        } else {
            const payment = this.payments.find((item) => item.id === id);
            if (!payment) { this.error('Payment not found. Refresh the records.'); return; }
            const order = this.order(payment.orderId);
            if (type === 'confirm') {
                if (payment.status !== 'Pending' || !order || order.status === 'Cancelled'
                    || this.refunded(order.id) > 0) {
                    this.error('This payment can no longer be confirmed.'); return;
                }
                path = `/payments/${payment.dbId}/confirm`;
                payload = { version: payment.version, reason };
            } else if (type === 'refund') {
                if (!['Confirmed', 'Refunded'].includes(payment.status)
                    || amount > this.paymentRefundable(payment)) {
                    this.error('Refund exceeds the confirmed amount still available.'); return;
                }
                path = `/payments/${payment.dbId}/refund`;
                payload = { version: payment.version, amount: decimalMoney(amount), reason };
            } else return;
        }
        await this.performWrite({ path, payload: { ...payload, request_key: requestKey() } });
    }

    // Keep the exact request key and payload when the server result is uncertain.
    async retryRequest() {
        if (!this.pendingOperation || this.submitting) return;
        await this.performWrite(this.pendingOperation);
    }

    // Accept only canonical server data; never create or change financial rows locally.
    async performWrite(operation) {
        if (this.submitting) return;
        this.pendingOperation = operation;
        this.submitting = true;
        this.setBusy(true);
        try {
            const response = await this.request(operation.path, {method: 'POST', body: JSON.stringify(operation.payload)});
            this.acceptData(response.data);
            this.pendingOperation = null;
            this.el('salesRetryRequest').hidden = true;
            this.pageError('');
            this.error();
            this.submitting = false;
            this.actionModal.hide();
            this.announce(response.message || 'Saved successfully.');
        } catch (error) {
            if (error.definitive) {
                this.pendingOperation = null;
                this.el('salesRetryRequest').hidden = true;
                this.error(error.message);
                this.announce(error.message);
                if ([403, 409, 419, 422].includes(error.status)) {
                    this.submitting = false;
                    this.actionModal.hide();
                    await this.loadData().catch((loadError) => this.pageError(loadError.message));
                }
            } else {
                const message = 'The result could not be confirmed. Use Retry pending request to safely repeat the same request.';
                this.error(message);
                this.pageError(message);
                this.el('salesRetryRequest').hidden = false;
            }
        } finally {
            this.submitting = false;
            this.setBusy(false);
        }
    }

    // Use authenticated same-origin JSON requests with Laravel CSRF protection.
    async request(path, options = {}) {
        let response;
        try {
            response = await fetch(`${this.baseUrl}${path}`, {
                credentials: 'same-origin',
                ...options,
                headers: {
                    'Accept': 'application/json',
                    'Content-Type': 'application/json',
                    'X-Requested-With': 'XMLHttpRequest',
                    'X-CSRF-TOKEN': this.csrf,
                    ...options.headers,
                },
            });
        } catch {
            throw new Error('Connection interrupted. The server result is unknown; retry the same request.');
        }
        let payload;
        try { payload = await response.json(); }
        catch {
            const error = new Error(response.ok ? 'The server returned an unreadable response.' : `Request failed (${response.status}).`);
            error.status = response.status;
            error.definitive = response.status >= 400 && response.status < 500 && response.status !== 408;
            throw error;
        }
        if (!response.ok) {
            const validation = payload.errors && Object.values(payload.errors).flat().filter((value) => typeof value === 'string').join(' ');
            const error = new Error(validation || payload.message || `Request failed (${response.status}).`);
            error.status = response.status;
            error.definitive = response.status >= 400 && response.status < 500 && response.status !== 408;
            throw error;
        }
        return payload;
    }

    // Fetch current database records before opening financial actions or details.
    async loadData() {
        if (this.readPromise) return this.readPromise;
        this.loading = true;
        this.el('salesReloadData').disabled = true;
        this.el('recordPaymentBtn').disabled = true;
        this.readPromise = (async () => {
            try {
                const data = await this.request('/data');
                this.acceptData(data.data || data);
                this.loaded = true;
                if (!this.pendingOperation) this.pageError('');
            } finally {
                this.loading = false;
                this.readPromise = null;
                this.el('salesReloadData').disabled = false;
                this.setBusy(this.submitting);
            }
        })();
        return this.readPromise;
    }

    // Validate the dataset envelope before replacing any UI collection.
    acceptData(data) {
        if (!data || !['flowers', 'orders', 'sales', 'payments', 'refunds', 'history'].every((key) => Array.isArray(data[key]))
            || !data.permissions || typeof data.permissions !== 'object') {
            throw new Error('The server returned an incomplete financial dataset.');
        }
        // Reject invalid or unsafe integer data before replacing the previous canonical state.
        const storedMoney = (value) => Number.isSafeInteger(value) && value >= 0 && value <= MAX_CENTS;
        const amounts = [...data.payments.map((row) => row.amount), ...data.refunds.map((row) => row.amount),
            ...data.sales.map((row) => row.gross), ...data.flowers.filter((row) => row.price !== null).map((row) => row.price),
            ...data.orders.map((row) => row.deliveryFee)];
        if (!amounts.every(storedMoney)) throw new Error('The server returned an invalid financial amount.');
        data.orders.forEach((order) => this.orderTotal(order));
        data.sales.forEach((sale) => (sale.items || []).forEach((item) => itemTotal(item)));
        for (const amounts of [data.payments.map((row) => row.amount), data.refunds.map((row) => row.amount), data.sales.map((row) => row.gross)]) {
            amounts.reduce(addCents, 0);
        }
        this.prices = data.flowers;
        this.orders = data.orders;
        this.sales = data.sales;
        this.payments = data.payments;
        this.refunds = data.refunds;
        this.history = data.history;
        this.permissions = data.permissions;
        this.el('recordPaymentBtn').hidden = !this.permissions.recordPayment;
        this.refresh();
    }

    // Lock write controls while a request is in flight or awaits a safe retry.
    setBusy(busy) {
        this.el('salesActionSubmit').disabled = busy || Boolean(this.pendingOperation);
        this.el('recordPaymentBtn').disabled = busy || this.loading || !this.loaded || Boolean(this.pendingOperation);
        this.el('salesRetryRequest').disabled = busy;
        this.el('salesActionFields').querySelectorAll('input, select, textarea').forEach((input) => {
            input.disabled = busy || Boolean(this.pendingOperation);
        });
        this.page.setAttribute('aria-busy', String(busy || this.loading));
    }

    // Present connection and API errors without injecting response HTML.
    pageError(message) {
        this.el('salesPageError').textContent = message;
        this.el('salesPageError').hidden = !message;
    }

    // Announce results through the live status region.
    announce(message) {
        this.el('salesAnnouncement').textContent = message;
    }

    // Replace displayed table rows with canonical server collections.
    refresh() {
        [
            ['prices', this.prices],
            ['records', this.sales],
            ['payments', this.payments],
        ].forEach(([key, data]) => {
            this.tables[key].clear();
            this.tables[key].rows.add(data);
            this.tables[key].draw(false);
        });

        this.renderSummary();
    }

    // Compute net completed sales and separate Stem/Bouquet rankings from server snapshots.
    renderSummary() {
        const net = (sale) => sale.gross - this.refunded(sale.orderId);
        const today = dateKey(new Date());
        const month = today.slice(0, 7);

        this.el('salesNet').textContent =
            money(this.sales.reduce((sum, sale) => addCents(sum, net(sale)), 0));

        this.el('salesCount').textContent = this.sales.length;

        this.el('salesToday').textContent = money(
            this.sales.filter((sale) => dateKey(sale.date) === today)
                .reduce((sum, sale) => addCents(sum, net(sale)), 0),
        );

        this.el('salesMonth').textContent = money(
            this.sales.filter((sale) => dateKey(sale.date).slice(0, 7) === month)
                .reduce((sum, sale) => addCents(sum, net(sale)), 0),
        );

        const quantities = new Map();
        const dates = new Map();

        this.sales.forEach((sale) => {
            (sale.items || this.order(sale.orderId)?.items || []).forEach((item) => {
                const key = `${item.baseUnit}|${item.name}`;
                quantities.set(key, addCents(quantities.get(key) || 0, item.quantity));
            });

            const key = dateKey(sale.date);
            dates.set(key, addCents(dates.get(key) || 0, net(sale)));
        });

        const trend = [...dates].sort((a, b) => a[0].localeCompare(b[0]));
        // Rank quantities within each unit; do not compare stems to bouquets.
        this.el('salesTopFlowers').innerHTML = ['Stem', 'Bouquet'].map((unit) => {
            const values = [...quantities].filter(([key]) => key.startsWith(`${unit}|`))
                .sort((a, b) => b[1] - a[1]).slice(0, 5);
            const maximum = Math.max(1, ...values.map(([, value]) => value));
            return `<h3 class="sales-section-title">${unit === 'Stem' ? 'Loose Stems' : 'Finished Bouquets'}</h3>`
                + (values.length ? values.map(([key, value]) => `
                    <div class="sales-bar-row">
                        <div class="sales-bar-label"><span>${escapeHtml(key.split('|').slice(1).join('|'))}</span>
                        <strong>${value} ${unit === 'Stem' ? 'stems' : 'bouquets'}</strong></div>
                        <div class="sales-bar-track" aria-hidden="true"><div class="sales-bar-fill" style="width:${value / maximum * 100}%"></div></div>
                    </div>`).join('') : '<p class="sales-help">No completed sales for this unit.</p>');
        }).join('');
        this.renderBars('salesTrend', trend, money);
    }

    // Render accessible text values alongside decorative chart bars.
    renderBars(id, values, formatter) {
        const maximum = Math.max(1, ...values.map(([, value]) => value));

        this.el(id).innerHTML = values.length ? values.map(([label, value]) => `
            <div class="sales-bar-row">
                <div class="sales-bar-label">
                    <span>${escapeHtml(label)}</span>
                    <strong>${escapeHtml(formatter(value))}</strong>
                </div>
                <div class="sales-bar-track" aria-hidden="true">
                    <div class="sales-bar-fill" style="width:${Math.max(0, value / maximum * 100)}%"></div>
                </div>
            </div>
        `).join('') : '<p class="sales-help">No completed sales yet.</p>';
    }

    // Reload and display preserved order, receipt and refund history.
    async showOrder(orderId) {
        if (this.submitting) return;
        try { await this.loadData(); }
        catch (error) { this.pageError(error.message); return; }
        const order = this.order(orderId);
        if (!order) return;

        const payments = this.payments.filter((payment) => payment.orderId === orderId);
        const refunds = this.refunds.filter((refund) => refund.orderId === orderId);
        const history = this.history.filter((entry) => entry.orderId === orderId);
        const sale = this.sales.find((record) => record.orderId === orderId);
        const paid = this.confirmedPaid(orderId);
        const refunded = this.refunded(orderId);

        this.el('salesViewTitle').textContent = `Financial Details — ${orderId}`;

        this.el('salesViewContent').innerHTML = `
            <dl class="sales-detail-grid">
                <div><dt>Client</dt><dd>${escapeHtml(order.client)}</dd></div>
                <div><dt>Order Status</dt><dd>${badge(order.status)}</dd></div>
                <div><dt>Payment Status</dt><dd>${badge(this.paymentStatus(order))}</dd></div>
                <div><dt>Sales Record</dt><dd>${escapeHtml(sale?.id || 'No Sales Record')}</dd></div>
                <div><dt>Order Total</dt><dd>${money(this.orderTotal(order))}</dd></div>
                <div><dt>Confirmed Payments</dt><dd>${money(paid)}</dd></div>
                <div><dt>Refunded</dt><dd>${money(refunded)}</dd></div>
                <div><dt>Net Collected</dt><dd>${money(paid - refunded)}</dd></div>
                <div><dt>Outstanding Before Refunds</dt><dd>${money(Math.max(0, this.orderTotal(order) - paid))}</dd></div>
                <div><dt>Pending Payments</dt><dd>${money(this.pendingPaid(orderId))}</dd></div>
            </dl>

            <p class="sales-help">
                Refunds are shown separately and do not automatically reopen collection
                for this order. Net sales are recognized from Completed orders,
                independently of confirmed payments.
            </p>

            <h6 class="sales-section-title">Order Price Snapshots</h6>
            <div class="sales-detail-scroll" tabindex="0" role="region" aria-label="Order items">
                <table class="sales-detail-table">
                    <thead>
                        <tr>
                            <th scope="col">Flower</th>
                            <th scope="col">Quantity / Unit</th>
                            <th scope="col">Unit Price</th>
                            <th scope="col">Subtotal</th>
                        </tr>
                    </thead>
                    <tbody>
                        ${order.items.map((item) => `
                            <tr>
                                <td>${escapeHtml(item.name)}</td>
                                <td>${item.quantity} ${escapeHtml(item.baseUnit)}</td>
                                <td class="sales-money">${money(item.unitPrice)}</td>
                                <td class="sales-money">${money(itemTotal(item))}</td>
                            </tr>
                        `).join('')}
                        <tr>
                            <td colspan="3">Delivery Fee</td>
                            <td class="sales-money">${money(order.deliveryFee)}</td>
                        </tr>
                    </tbody>
                </table>
            </div>

            <h6 class="sales-section-title">Payment History</h6>
            <div class="sales-detail-scroll" tabindex="0" role="region" aria-label="Payment history">
                <table class="sales-detail-table">
                    <thead>
                        <tr>
                            <th scope="col">Payment</th>
                            <th scope="col">Date</th>
                            <th scope="col">Method / Reference</th>
                            <th scope="col">Amount</th>
                            <th scope="col">Status</th>
                        </tr>
                    </thead>
                    <tbody>
                        ${payments.length ? payments.map((payment) => `
                            <tr>
                                <td>${escapeHtml(payment.id)}</td>
                                <td>${dateTime(payment.date)}</td>
                                <td>${cell(payment.method, payment.reference)}</td>
                                <td class="sales-money">${money(payment.amount)}</td>
                                <td>${badge(payment.status)}</td>
                            </tr>
                        `).join('') : '<tr><td colspan="5">No payments recorded.</td></tr>'}
                    </tbody>
                </table>
            </div>

            <h6 class="sales-section-title">Refund History</h6>
            <div class="sales-detail-scroll" tabindex="0" role="region" aria-label="Refund history">
                <table class="sales-detail-table">
                    <thead>
                        <tr>
                            <th scope="col">Refund / Payment</th>
                            <th scope="col">Date</th>
                            <th scope="col">Reason</th>
                            <th scope="col">Amount</th>
                        </tr>
                    </thead>
                    <tbody>
                        ${refunds.length ? refunds.map((refund) => `
                            <tr>
                                <td>${cell(refund.id, refund.paymentId)}</td>
                                <td>${dateTime(refund.date)}</td>
                                <td>${escapeHtml(refund.reason)}</td>
                                <td class="sales-money">${money(refund.amount)}</td>
                            </tr>
                        `).join('') : '<tr><td colspan="4">No refunds recorded.</td></tr>'}
                    </tbody>
                </table>
            </div>

            <h6 class="sales-section-title">Activity History</h6>
            <ol class="sales-history">
                ${history.length ? history.map((entry) => `
                    <li>
                        ${escapeHtml(entry.note)}
                        <small>${dateTime(entry.date)} · ${escapeHtml(entry.actor)}</small>
                    </li>
                `).join('') : '<li>No recorded activity for this order.</li>'}
            </ol>
        `;

        this.viewModal.show();
    }
}

// Initialize once when the Sales & Payments page is present.
function initializeSalesPayments() {
    if (!document.getElementById('pricesTable')
        || DataTable.isDataTable('#pricesTable')) return;

    new SalesPaymentsUI();
}

if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', initializeSalesPayments, {
        once: true,
    });
} else {
    initializeSalesPayments();
}
```

## `resources/views/supplier/sales-payments.blade.php`

```blade
@extends('layouts.supplier')

@section('title', 'Sales & Payments')

@section('content')
    <div class="sales-page" data-url="{{ route('supplier.sales-payments') }}" data-csrf="{{ csrf_token() }}" aria-busy="true">
        <div class="sales-summary">
            <div>
                <span>Net Sales</span>
                <strong id="salesNet">₱0.00</strong>
            </div>
            <div>
                <span>Sales Transactions</span>
                <strong id="salesCount">0</strong>
            </div>
            <div>
                <span>Net Sales Today</span>
                <strong id="salesToday">₱0.00</strong>
            </div>
            <div>
                <span>Net Sales This Month</span>
                <strong id="salesMonth">₱0.00</strong>
            </div>
        </div>

        <div class="sales-card">
            <div class="sales-header">
                <div>
                    <h1>Sales & Payments</h1>
                    <p>Manage selling prices, completed sales, and client payments.</p>
                </div>

                <button type="button" class="sales-primary-btn" id="recordPaymentBtn" hidden disabled>
                    <i class="bi bi-plus-lg" aria-hidden="true"></i>
                    Record Payment
                </button>
            </div>

            <div class="sales-notice">
                <i class="bi bi-info-circle" aria-hidden="true"></i>
                <span>
                    Payment recording and confirmation are manual. Verify actual receipts before
                    confirming payments. Recording a refund does not transfer money automatically.
                </span>
            </div>

            <div class="sales-notice" aria-live="polite">
                <span id="salesPageError" class="sales-error" role="alert" hidden></span>
                <button type="button" class="sales-secondary-btn" id="salesReloadData">Refresh Records</button>
                <button type="button" class="sales-secondary-btn" id="salesRetryRequest" hidden>Retry pending request</button>
            </div>

            <div class="sales-tabs" role="tablist" aria-label="Sales and payment sections">
                <button
                    type="button"
                    id="pricesTab"
                    role="tab"
                    aria-controls="pricesPanel"
                    aria-selected="true"
                    data-sales-tab="prices"
                    class="active"
                >
                    Selling Prices
                </button>
                <button
                    type="button"
                    id="recordsTab"
                    role="tab"
                    aria-controls="recordsPanel"
                    aria-selected="false"
                    tabindex="-1"
                    data-sales-tab="records"
                >
                    Sales Records
                </button>
                <button
                    type="button"
                    id="paymentsTab"
                    role="tab"
                    aria-controls="paymentsPanel"
                    aria-selected="false"
                    tabindex="-1"
                    data-sales-tab="payments"
                >
                    Payments
                </button>
            </div>

            <section id="pricesPanel" role="tabpanel" aria-labelledby="pricesTab">
                <div class="sales-toolbar">
                    <div class="sales-length">
                        <label for="pricesLength">Show</label>
                        <select id="pricesLength" data-length="prices">
                            <option value="5">5</option>
                            <option value="10">10</option>
                            <option value="25">25</option>
                            <option value="50">50</option>
                        </select>
                        <span>entries</span>
                    </div>
                    <div class="sales-search">
                        <label class="visually-hidden" for="pricesSearch">Search flowers</label>
                        <i class="bi bi-search" aria-hidden="true"></i>
                        <input
                            type="search"
                            id="pricesSearch"
                            data-search="prices"
                            placeholder="Search flower name or code..."
                            autocomplete="off"
                        >
                    </div>
                </div>

                <div class="sales-table-container">
                    <table id="pricesTable" class="sales-table">
                        <thead>
                            <tr>
                                <th scope="col">Flower Details</th>
                                <th scope="col">Category</th>
                                <th scope="col">Price / Unit</th>
                                <th scope="col">Updated</th>
                                <th scope="col">Actions</th>
                            </tr>
                        </thead>
                        <tbody></tbody>
                    </table>
                </div>
            </section>

            <section
                id="recordsPanel"
                role="tabpanel"
                aria-labelledby="recordsTab"
                hidden
            >
                <div class="sales-toolbar">
                    <div class="sales-length">
                        <label for="recordsLength">Show</label>
                        <select id="recordsLength" data-length="records">
                            <option value="5">5</option>
                            <option value="10">10</option>
                            <option value="25">25</option>
                            <option value="50">50</option>
                        </select>
                        <span>entries</span>
                    </div>
                    <div class="sales-search">
                        <label class="visually-hidden" for="recordsSearch">Search sales</label>
                        <i class="bi bi-search" aria-hidden="true"></i>
                        <input
                            type="search"
                            id="recordsSearch"
                            data-search="records"
                            placeholder="Search sale, order or client..."
                            autocomplete="off"
                        >
                    </div>
                </div>

                <div class="sales-table-container">
                    <table id="recordsTable" class="sales-table">
                        <thead>
                            <tr>
                                <th scope="col">Sale / Order</th>
                                <th scope="col">Client</th>
                                <th scope="col">Completed</th>
                                <th scope="col">Gross Sale</th>
                                <th scope="col">Refunds</th>
                                <th scope="col">Net Sale</th>
                                <th scope="col">Actions</th>
                            </tr>
                        </thead>
                        <tbody></tbody>
                    </table>
                </div>
            </section>

            <section
                id="paymentsPanel"
                role="tabpanel"
                aria-labelledby="paymentsTab"
                hidden
            >
                <div class="sales-toolbar">
                    <div class="sales-length">
                        <label for="paymentsLength">Show</label>
                        <select id="paymentsLength" data-length="payments">
                            <option value="5">5</option>
                            <option value="10">10</option>
                            <option value="25">25</option>
                            <option value="50">50</option>
                        </select>
                        <span>entries</span>
                    </div>
                    <div class="sales-search">
                        <label class="visually-hidden" for="paymentsSearch">Search payments</label>
                        <i class="bi bi-search" aria-hidden="true"></i>
                        <input
                            type="search"
                            id="paymentsSearch"
                            data-search="payments"
                            placeholder="Search payment, order or client..."
                            autocomplete="off"
                        >
                    </div>
                </div>

                <div class="sales-table-container">
                    <table id="paymentsTable" class="sales-table">
                        <thead>
                            <tr>
                                <th scope="col">Payment / Order</th>
                                <th scope="col">Client</th>
                                <th scope="col">Recorded</th>
                                <th scope="col">Method / Reference</th>
                                <th scope="col">Amount</th>
                                <th scope="col">Status</th>
                                <th scope="col">Actions</th>
                            </tr>
                        </thead>
                        <tbody></tbody>
                    </table>
                </div>
            </section>

            <p id="salesAnnouncement" class="sales-announcement" role="status"></p>
        </div>

        <div class="sales-insights">
            <div class="sales-card sales-insight-card">
                <h2>Top Selling Items</h2>
                <p>Completed-order quantities grouped by Stem and Bouquet, before refund adjustments.</p>
                <div id="salesTopFlowers"></div>
            </div>

            <div class="sales-card sales-insight-card">
                <h2>Sales Trend</h2>
                <p>Net sales by completion date, less recorded refunds.</p>
                <div id="salesTrend"></div>
            </div>
        </div>
    </div>

    {{-- Create payment / edit price / confirm / refund --}}
    <div
        class="modal fade"
        id="salesActionModal"
        tabindex="-1"
        aria-labelledby="salesActionTitle"
        aria-hidden="true"
    >
        <div class="modal-dialog modal-dialog-centered">
            <div class="modal-content sales-modal">
                <div class="modal-header">
                    <h5 class="modal-title" id="salesActionTitle">Update</h5>
                    <button
                        type="button"
                        class="btn-close"
                        data-bs-dismiss="modal"
                        aria-label="Close"
                    ></button>
                </div>

                <form id="salesActionForm">
                    @csrf
                    <div class="modal-body">
                        <div id="salesActionError" class="sales-error" role="alert" hidden></div>
                        <div id="salesActionFields"></div>
                    </div>
                    <div class="modal-footer">
                        <button
                            type="button"
                            class="sales-secondary-btn"
                            data-bs-dismiss="modal"
                        >
                            Cancel
                        </button>
                        <button type="submit" class="sales-primary-btn" id="salesActionSubmit">
                            Save
                        </button>
                    </div>
                </form>
            </div>
        </div>
    </div>

    {{-- Order financial details --}}
    <div
        class="modal fade"
        id="salesViewModal"
        tabindex="-1"
        aria-labelledby="salesViewTitle"
        aria-hidden="true"
    >
        <div class="modal-dialog modal-dialog-centered modal-xl">
            <div class="modal-content sales-modal">
                <div class="modal-header">
                    <div>
                        <span class="sales-eyebrow">FINANCIAL DETAILS</span>
                        <h5 class="modal-title" id="salesViewTitle">Order Summary</h5>
                    </div>
                    <button
                        type="button"
                        class="btn-close"
                        data-bs-dismiss="modal"
                        aria-label="Close"
                    ></button>
                </div>
                <div class="modal-body" id="salesViewContent"></div>
                <div class="modal-footer">
                    <button
                        type="button"
                        class="sales-secondary-btn"
                        data-bs-dismiss="modal"
                    >
                        Close
                    </button>
                </div>
            </div>
        </div>
    </div>
@endsection
```

## `routes/web.php`

```php
<?php

use App\Http\Controllers\AuthController;
use Illuminate\Support\Facades\Route;
use App\Http\Controllers\Supplier\FlowerController;
use App\Http\Controllers\Supplier\InventoryController;
use App\Http\Controllers\Supplier\BatchController;
use App\Http\Controllers\Supplier\ClientController;
use App\Http\Controllers\Supplier\OrderController;
use App\Http\Controllers\Supplier\UserController;
use App\Http\Controllers\Supplier\SalesPaymentController;
use App\Http\Middleware\EnsureAccountSessionIsCurrent;
use App\Http\Controllers\Supplier\DeliveryController;

// Public home
Route::view('/', 'welcome');

// Login
Route::get('/login', [AuthController::class, 'showLogin'])
    ->name('login');

Route::post('/login', [AuthController::class, 'login'])
    ->middleware('guest')
    ->name('login.store');

// Signed-in accounts
Route::middleware('auth')->group(function () {
    Route::post('/logout', [AuthController::class, 'logout'])
        ->name('logout');

    Route::middleware(['account.active', EnsureAccountSessionIsCurrent::class])->group(function () {
        // These routes must remain accessible while a password change is required.
        Route::get('/change-password', [
            AuthController::class,
            'showChangePassword',
        ])->name('password.change');

        Route::put('/change-password', [
            AuthController::class,
            'updatePassword',
        ])
            ->middleware('throttle:5,1')
            ->name('password.update');

        // Delivery personnel can access only their own assigned records.
        Route::prefix('delivery')
            ->name('delivery.')
            ->middleware(['password.changed', 'role:delivery'])
            ->group(function () {
                Route::get('/deliveries', [DeliveryController::class, 'index'])
                    ->name('deliveries');
                Route::get('/deliveries/data', [DeliveryController::class, 'data'])
                    ->name('deliveries.data');
                Route::get('/deliveries/{delivery}/proof', [DeliveryController::class, 'proof'])
                    ->whereNumber('delivery')->name('deliveries.proof');
                Route::get('/deliveries/{delivery}/proof', [DeliveryController::class, 'proof'])
                    ->whereNumber('delivery')->name('deliveries.proof');

                Route::get('/deliveries/{delivery}', [DeliveryController::class, 'show'])
                    ->whereNumber('delivery')->name('deliveries.show');
                Route::post('/deliveries/{delivery}/action', [DeliveryController::class, 'action'])
                    ->whereNumber('delivery')->name('deliveries.action');
            });

        // Supplier/Admin operational screens
        Route::prefix('supplier')
            ->name('supplier.')
            ->middleware([
                'password.changed',
                'role:admin,supplier',
            ])
            ->group(function () {
                Route::view('/dashboard', 'supplier.dashboard')
                    ->name('dashboard');

                Route::get('/flowers', [FlowerController::class, 'index'])
                    ->name('flowers');

                Route::get('/flowers/data', [FlowerController::class, 'data'])
                    ->name('flowers.data');

                Route::post('/flowers', [FlowerController::class, 'store'])
                    ->name('flowers.store');

                Route::post('/flowers/{flower}/update', [FlowerController::class, 'update'])
                    ->whereNumber('flower')
                    ->name('flowers.update');

                Route::post('/flowers/{flower}/deactivate', [FlowerController::class, 'deactivate'])
                    ->whereNumber('flower')
                    ->name('flowers.deactivate');

                Route::get('/inventory', [InventoryController::class, 'index'])
                    ->name('inventory');

                Route::get('/inventory/data', [InventoryController::class, 'data'])
                    ->name('inventory.data');

                Route::post('/inventory/stock-in', [InventoryController::class, 'stockIn'])
                    ->name('inventory.stock-in');

                Route::post('/inventory/stock-out', [InventoryController::class, 'stockOut'])
                    ->name('inventory.stock-out');

                Route::post('/inventory/batches/{batch}/receipt', [InventoryController::class, 'correctReceipt'])
                    ->whereNumber('batch')
                    ->name('inventory.correct-receipt');

                Route::post('/inventory/flowers/{flower}/units', [InventoryController::class, 'updateUnits'])
                    ->whereNumber('flower')
                    ->name('inventory.units');

                Route::get('/batches', [BatchController::class, 'index'])
                    ->name('batches');

                Route::get('/batches/data', [BatchController::class, 'data'])
                    ->name('batches.data');

                Route::post('/batches/{batch}/update', [BatchController::class, 'update'])
                    ->whereNumber('batch')
                    ->name('batches.update');

                Route::view('/spoilage-prediction', 'supplier.spoilage-prediction')
                    ->name('spoilage-prediction');

                Route::get('/clients', [ClientController::class, 'index'])
                    ->name('clients');

                Route::get('/clients/data', [ClientController::class, 'data'])
                    ->name('clients.data');

                Route::get('/clients/{client}', [ClientController::class, 'show'])
                    ->whereNumber('client')
                    ->name('clients.show');

                Route::post('/clients', [ClientController::class, 'store'])
                    ->name('clients.store');

                Route::post('/clients/{client}/update', [ClientController::class, 'update'])
                    ->whereNumber('client')
                    ->name('clients.update');

                Route::post('/clients/{client}/status', [ClientController::class, 'status'])
                    ->whereNumber('client')
                    ->name('clients.status');

                Route::get('/orders', [OrderController::class, 'index'])
                    ->name('orders');

                Route::get('/orders/data', [OrderController::class, 'data'])
                    ->name('orders.data');

                Route::get('/orders/{order}', [OrderController::class, 'show'])
                    ->whereNumber('order')
                    ->name('orders.show');

                Route::post('/orders', [OrderController::class, 'store'])
                    ->name('orders.store');

                Route::post('/orders/{order}/update', [OrderController::class, 'update'])
                    ->whereNumber('order')
                    ->name('orders.update');

                Route::post('/orders/{order}/action', [OrderController::class, 'action'])
                    ->whereNumber('order')
                    ->name('orders.action');

                Route::get('/deliveries', [DeliveryController::class, 'index'])
                    ->name('deliveries');

                Route::get('/deliveries/data', [DeliveryController::class, 'data'])
                    ->name('deliveries.data');

                Route::get('/deliveries/{delivery}', [DeliveryController::class, 'show'])
                    ->whereNumber('delivery')
                    ->name('deliveries.show');

                Route::post('/deliveries/{delivery}/action', [DeliveryController::class, 'action'])
                    ->whereNumber('delivery')
                    ->name('deliveries.action');

                Route::get('/sales-payments', [SalesPaymentController::class, 'index'])
                    ->name('sales-payments');
                Route::get('/sales-payments/data', [SalesPaymentController::class, 'data'])
                    ->name('sales-payments.data');
                Route::post('/sales-payments/prices/{flower}', [SalesPaymentController::class, 'price'])
                    ->whereNumber('flower')->name('sales-payments.price');
                Route::post('/sales-payments/payments', [SalesPaymentController::class, 'recordPayment'])
                    ->name('sales-payments.record');
                Route::post('/sales-payments/payments/{payment}/confirm', [SalesPaymentController::class, 'confirm'])
                    ->whereNumber('payment')->name('sales-payments.confirm');
                Route::post('/sales-payments/payments/{payment}/refund', [SalesPaymentController::class, 'refund'])
                    ->whereNumber('payment')->name('sales-payments.refund');

                // Admin-only screens
                Route::middleware('role:admin')->group(function () {
                    Route::view('/reports', 'supplier.reports')
                        ->name('reports');

                    Route::get('/users', [UserController::class, 'index'])->name('users');
                    Route::get('/users/data', [UserController::class, 'data'])->name('users.data');
                    Route::get('/users/{user}', [UserController::class, 'show'])
                        ->whereNumber('user')->name('users.show');
                    Route::post('/users', [UserController::class, 'store'])->name('users.store');
                    Route::post('/users/{user}/update', [UserController::class, 'update'])
                        ->whereNumber('user')->name('users.update');
                    Route::post('/users/{user}/status', [UserController::class, 'status'])
                        ->whereNumber('user')->name('users.status');
                    Route::post('/users/{user}/reset-password', [UserController::class, 'resetPassword'])
                        ->whereNumber('user')->name('users.reset-password');

                    Route::view('/audit-trail', 'supplier.audit-trail')
                        ->name('audit-trail');
                });
            });
    });
});
```

## `tests/Feature/SalesBackendTest.php`

```php
<?php

namespace Tests\Feature;

use App\Http\Requests\StorePaymentRequest;
use App\Models\Client;
use App\Models\Flower;
use App\Models\FlowerPrice;
use App\Models\FinanceOperation;
use App\Models\InventoryBatch;
use App\Models\InventoryMovement;
use App\Models\InventoryReservation;
use App\Models\Order;
use App\Models\OrderPayment;
use App\Models\PaymentRefund;
use App\Models\SalesRecord;
use App\Models\User;
use App\Services\InventoryService;
use App\Services\OrderService;
use App\Services\SalesPaymentService;
use App\Support\Money;
use App\Support\AccountSession;
use Illuminate\Auth\Access\AuthorizationException;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Schema;
use Illuminate\Support\Facades\Validator;
use Illuminate\Support\Str;
use Illuminate\Validation\ValidationException;
use Symfony\Component\HttpKernel\Exception\HttpException;
use Tests\TestCase;

class SalesBackendTest extends TestCase
{
    private OrderService $orders;
    private InventoryService $inventory;
    private SalesPaymentService $sales;
    private User $admin;
    private User $supplier;
    private Client $client;
    private Flower $stem;
    private Flower $bouquet;
    private bool $testConnectionConfigured = false;

    /** Isolate every schema operation in a named memory-only SQLite connection. */
    protected function setUp(): void
    {
        parent::setUp();
        if (! extension_loaded('pdo_sqlite')) {
            $this->markTestSkipped('Isolated sales tests require pdo_sqlite.');
        }
        config([
            'database.connections.sales_backend_test' => [
                'driver' => 'sqlite', 'database' => ':memory:', 'prefix' => '',
                'foreign_key_constraints' => true,
            ],
            'database.default' => 'sales_backend_test',
            'orders.delivery_fee' => null,
            'session.driver' => 'array',
            'cache.default' => 'array',
        ]);
        $this->testConnectionConfigured = true;
        DB::purge('sales_backend_test');
        $this->assertSame('sqlite', DB::connection()->getDriverName());
        $this->assertSame(':memory:', DB::connection()->getDatabaseName());
        Schema::create('users', function (Blueprint $table) {
            $table->id(); $table->string('name'); $table->string('email')->unique();
            $table->string('password'); $table->string('role');
            $table->boolean('is_active')->default(true);
            $table->boolean('must_change_password')->default(false);
            $table->timestamp('email_verified_at')->nullable(); $table->rememberToken(); $table->timestamps();
        });
        Schema::create('flowers', function (Blueprint $table) {
            $table->id(); $table->string('code', 30)->unique(); $table->string('name', 150);
            $table->enum('product_type', ['Stem', 'Bouquet']); $table->string('category', 100);
            $table->unsignedInteger('minimum_stock'); $table->text('description');
            $table->string('image_path')->nullable(); $table->enum('status', ['Active', 'Inactive']); $table->timestamps();
        });
        foreach (['create_clients_table', 'create_inventory_tables', 'create_order_tables', 'create_delivery_tables', 'create_finance_tables'] as $name) {
            $files = glob(database_path('migrations/*_'.$name.'.php')) ?: [];
            $this->assertCount(1, $files, 'Expected one '.$name.' migration.');
            (require $files[0])->up();
        }
        $this->admin = $this->user('admin', User::ROLE_ADMIN);
        $this->supplier = $this->user('supplier', User::ROLE_SUPPLIER);
        $account = $this->user('client', User::ROLE_CLIENT);
        $this->client = Client::create([
            'code' => 'CLI-001', 'user_id' => $account->id,
            'phone' => '09123456789', 'address' => 'Test delivery address',
        ]);
        $this->stem = $this->item('FLW-001', 'Stem', 1250);
        $this->bouquet = $this->item('FLW-002', 'Bouquet', 125000);
        $this->inventory = app(InventoryService::class);
        $this->orders = app(OrderService::class);
        $this->sales = app(SalesPaymentService::class);
        $this->receive($this->stem, 20);
        $this->receive($this->bouquet, 5);
    }

    /** Disconnect only the ephemeral named connection. */
    protected function tearDown(): void
    {
        if ($this->testConnectionConfigured) DB::disconnect('sales_backend_test');
        parent::tearDown();
    }

    /** Collection is exactly one full payment; under/over amounts cannot mutate stock or orders. */
    public function test_recording_requires_exact_full_payment_without_inventory_changes(): void
    {
        $order = $this->orders->store($this->input(), $this->supplier);
        $movements = InventoryMovement::count();
        $this->rejected(fn () => $this->sales->recordPayment($this->paymentInput($order, ['amount' => '24.99']), $this->supplier));
        $this->rejected(fn () => $this->sales->recordPayment($this->paymentInput($order, ['amount' => '25.01']), $this->supplier));
        $this->assertSame(0, OrderPayment::count());
        $this->sales->recordPayment($this->paymentInput($order), $this->supplier);
        $payment = OrderPayment::firstOrFail();
        $this->assertSame(2500, $payment->amount_cents);
        $this->assertSame('Pending', $payment->status);
        $this->assertNull($payment->confirmed_at);
        $this->assertSame('Pending', $order->fresh()->status);
        $this->assertSame(20, $this->remaining($this->stem));
        $this->assertSame(0, InventoryReservation::count());
        $this->assertSame($movements, InventoryMovement::count());
        $this->assertSame(0, SalesRecord::count());
        $mixed = $this->orders->store($this->input(['items' => [
            ['flower_id' => $this->stem->id, 'quantity' => 2],
            ['flower_id' => $this->bouquet->id, 'quantity' => 1],
        ]]), $this->supplier);
        $this->assertSame(127500, $this->record($mixed)->amount_cents);
        $this->assertSame(5, $this->remaining($this->bouquet));
        $this->assertSame($movements, InventoryMovement::count());
    }

    /** A replay is safe even when its version changed; another UUID cannot create a second payment. */
    public function test_payment_uuid_replay_conflict_and_order_uniqueness(): void
    {
        $order = $this->orders->store($this->input(), $this->supplier);
        $input = $this->paymentInput($order, ['request_key' => 'a0000000-aaaa-4aaa-8aaa-aaaaaaaaaaaa']);
        $this->sales->recordPayment($input, $this->supplier);
        $input['request_key'] = strtoupper($input['request_key']);
        $input['version'] = $this->orderFinanceRow($order)['version'];
        $this->sales->recordPayment($input, $this->supplier);
        $this->assertSame(1, OrderPayment::count());
        $this->assertSame(1, FinanceOperation::count());
        $this->rejected(fn () => $this->sales->recordPayment($input, $this->admin), 'request_key');
        $changed = [...$input, 'remarks' => 'Different semantic request.'];
        $this->rejected(fn () => $this->sales->recordPayment($changed, $this->supplier), 'request_key');
        $this->rejected(fn () => $this->sales->recordPayment($this->paymentInput($order), $this->supplier));
        $this->assertSame(1, OrderPayment::count());
        $this->assertSame(1, FinanceOperation::count());
    }

    /** Actual receipt confirmation is independent of fulfillment and does not generate a sale. */
    public function test_confirmation_is_replay_safe_and_never_issues_inventory(): void
    {
        $order = $this->orders->store($this->input(), $this->supplier);
        $payment = $this->record($order);
        $input = $this->confirmInput($payment);
        $movements = InventoryMovement::count();
        $this->sales->confirm($payment, $input, $this->supplier);
        $input['version'] = $this->paymentFinanceRow($payment)['version'];
        $this->sales->confirm($payment->fresh(), $input, $this->supplier);
        $this->assertSame('Confirmed', $payment->fresh()->status);
        $this->assertSame($this->supplier->id, $payment->fresh()->confirmed_by);
        $this->assertNotNull($payment->fresh()->confirmed_at);
        $this->assertSame('Paid', $this->orders->row($order->fresh(), $this->supplier)['payment']);
        $this->assertSame('Pending', $order->fresh()->status);
        $this->assertFalse($order->fresh()->stocked_out);
        $this->assertSame(20, $this->remaining($this->stem));
        $this->assertSame($movements, InventoryMovement::count());
        $this->assertSame(0, SalesRecord::count());
    }

    /** Refunds are Admin-only, exact, bounded cumulatively, and append-only with safe retries. */
    public function test_partial_and_full_refunds_preserve_history_and_prevent_over_refund(): void
    {
        $order = $this->orders->store($this->input(), $this->supplier);
        $payment = $this->record($order);
        $this->sales->confirm($payment, $this->confirmInput($payment), $this->supplier);
        $confirmedAt = $payment->fresh()->confirmed_at->toIso8601String();
        $input = $this->refundInput($payment, '10.00');
        try {
            $this->sales->refund($payment, $input, $this->supplier);
            $this->fail('Supplier must not refund payments.');
        } catch (AuthorizationException|HttpException $exception) {
            if ($exception instanceof HttpException) {
                $this->assertSame(403, $exception->getStatusCode());
            }
        }
        $this->sales->refund($payment, $input, $this->admin);
        $input['version'] = $this->paymentFinanceRow($payment)['version'];
        $this->sales->refund($payment->fresh(), $input, $this->admin);
        $this->assertSame(1, PaymentRefund::count());
        $this->assertSame(1000, (int) PaymentRefund::sum('amount_cents'));
        $this->assertSame('Refunded', $payment->fresh()->status);
        $this->assertSame('Refunded', $this->orders->row($order->fresh(), $this->supplier)['payment']);
        $this->assertSame(2500, $this->orders->row($order->fresh(), $this->supplier)['paidCents']);
        $this->rejected(fn () => $this->sales->refund($payment, $this->refundInput($payment, '15.01'), $this->admin));
        $this->assertSame(1, PaymentRefund::count());
        $this->sales->refund($payment, $this->refundInput($payment, '15.00'), $this->admin);
        $this->assertSame('Refunded', $payment->fresh()->status);
        $this->assertSame(2500, (int) PaymentRefund::sum('amount_cents'));
        $this->assertSame(2500, $payment->fresh()->amount_cents);
        $this->assertSame(2, PaymentRefund::count());
        $this->assertSame($confirmedAt, $payment->fresh()->confirmed_at->toIso8601String());
        $this->assertSame('Cash', $payment->fresh()->method);
        $this->rejected(fn () => $this->sales->refund($payment, $this->refundInput($payment, '0.01'), $this->admin));
        $this->assertSame(2, PaymentRefund::count());
        $this->assertSame(20, $this->remaining($this->stem));
        $this->assertSame(1, OrderPayment::count());
    }

    /** Cancellation disallows new collections while preserving legitimate refund eligibility. */
    public function test_cancelled_orders_reject_collection_but_existing_payment_can_be_refunded(): void
    {
        $cancelled = $this->orders->store($this->input(), $this->supplier);
        $this->action($cancelled, 'Cancelled', $this->supplier, ['reason' => 'Client cancelled before paying.']);
        $this->rejected(fn () => $this->sales->recordPayment($this->paymentInput($cancelled), $this->supplier));
        $order = $this->orders->store($this->input(), $this->supplier);
        $payment = $this->record($order);
        $this->sales->confirm($payment, $this->confirmInput($payment), $this->supplier);
        $this->action($order, 'Cancelled', $this->supplier, ['reason' => 'Client cancelled after paying.']);
        $this->sales->refund($payment, $this->refundInput($payment, '25.00'), $this->admin);
        $this->assertSame('Cancelled', $order->fresh()->status);
        $this->assertSame('Refunded', $payment->fresh()->status);
        $this->assertSame(20, $this->remaining($this->stem));
    }

    /** Price changes leave historical orders intact and require current optimistic versions. */
    public function test_selling_prices_are_exact_and_do_not_reprice_existing_orders(): void
    {
        $order = $this->orders->store($this->input(), $this->supplier);
        $priceInput = $this->priceInput($this->stem, '20.01');
        $this->sales->price($this->stem, $priceInput, $this->supplier);
        $this->assertSame(2001, FlowerPrice::where('flower_id', $this->stem->id)->firstOrFail()->price_cents);
        $this->assertSame(1250, $order->items()->firstOrFail()->unit_price_cents);
        $this->assertSame(2500, $order->fresh()->total_cents);
        $stale = [...$priceInput, 'request_key' => (string) Str::uuid(), 'amount' => '30.00'];
        $this->rejected(fn () => $this->sales->price($this->stem, $stale, $this->supplier), 'version');
        foreach (['0', '10.001', '-1', '1e2'] as $amount) {
            $this->rejected(fn () => $this->sales->price($this->stem, $this->priceInput($this->stem, $amount), $this->supplier));
        }
        $this->assertSame(2001, FlowerPrice::where('flower_id', $this->stem->id)->firstOrFail()->price_cents);
        $this->stem->status = 'Inactive';
        $this->stem->save();
        $this->rejected(fn () => $this->sales->price($this->stem, $this->priceInput($this->stem, '40.00'), $this->supplier));
        $this->assertSame(2001, FlowerPrice::where('flower_id', $this->stem->id)->firstOrFail()->price_cents);
    }

    /** Stale order/payment snapshots cannot be used to record or confirm changed financial state. */
    public function test_stale_order_and_payment_finance_versions_are_rejected(): void
    {
        $order = $this->orders->store($this->input(), $this->supplier);
        $input = $this->paymentInput($order);
        $this->orders->update($order, $this->edit($order, ['items' => [['flower_id' => $this->stem->id, 'quantity' => 3]]]), $this->supplier);
        $input['amount'] = '37.50';
        $this->rejected(fn () => $this->sales->recordPayment($input, $this->supplier), 'version');
        $payment = $this->record($order->fresh());
        $confirm = $this->confirmInput($payment);
        $this->sales->confirm($payment, $confirm, $this->supplier);
        $staleConfirm = [...$confirm, 'request_key' => (string) Str::uuid()];
        $this->rejected(fn () => $this->sales->confirm($payment, $staleConfirm, $this->supplier), 'version');
        $this->assertSame('Confirmed', $payment->fresh()->status);
    }

    /** Noncash references are normalized and unique within their payment method, not globally. */
    public function test_noncash_reference_normalization_and_method_scope(): void
    {
        $first = $this->orders->store($this->input(), $this->supplier);
        $this->record($first, ['method' => 'GCash', 'reference' => 'TXN-001']);
        $second = $this->orders->store($this->input(), $this->supplier);
        $this->rejected(fn () => $this->sales->recordPayment($this->paymentInput($second, [
            'method' => 'GCash', 'reference' => ' txn-001 ',
        ]), $this->supplier));
        $this->record($second, ['method' => 'Bank Transfer', 'reference' => 'TXN-001']);
        $third = $this->orders->store($this->input(), $this->supplier);
        $this->rejected(fn () => $this->sales->recordPayment($this->paymentInput($third, [
            'method' => 'GCash', 'reference' => null,
        ]), $this->supplier));
        $this->assertSame(2, OrderPayment::count());
    }

    /** Payment requests reject direct status, identity, and money-state tampering. */
    public function test_payment_request_rejects_client_assigned_confirmation_and_stock_state(): void
    {
        $order = $this->orders->store($this->input(), $this->supplier);
        $payload = $this->paymentInput($order);
        $rules = (new StorePaymentRequest())->rules();
        $this->assertTrue(Validator::make($payload, $rules)->passes());
        foreach (['status' => 'Confirmed', 'confirmed_by' => $this->admin->id,
            'confirmed_at' => '2026-01-01', 'stocked_out' => true, 'amount_cents' => 1,
            'code' => 'PAY-FORGED'] as $field => $value) {
            $validator = Validator::make([...$payload, $field => $value], $rules);
            $this->assertTrue($validator->fails(), $field);
            $this->assertArrayHasKey($field, $validator->errors()->toArray());
        }
        $this->assertTrue(Validator::make([...$payload, 'amount' => 25.00], $rules)->fails());
        $this->assertTrue(Validator::make([...$payload, 'amount' => '25.001'], $rules)->fails());
        $this->assertSame(0, OrderPayment::count());
    }

    /** Direct service entry points reject nonoperational, inactive, and temporary-password accounts. */
    public function test_finance_read_and_write_role_boundaries(): void
    {
        $input = $this->priceInput($this->stem, '20.00');
        $inactive = clone $this->supplier;
        $inactive->is_active = false;
        $temporary = clone $this->supplier;
        $temporary->must_change_password = true;
        foreach ([$this->client->user, $inactive, $temporary] as $actor) {
            foreach ([fn () => $this->sales->data($actor), fn () => $this->sales->price($this->stem, $input, $actor)] as $action) {
                try {
                    $action();
                    $this->fail('Unauthorized financial access must be rejected.');
                } catch (AuthorizationException|HttpException $exception) {
                    if ($exception instanceof HttpException) $this->assertSame(403, $exception->getStatusCode());
                }
            }
        }
        $this->assertSame(0, FinanceOperation::count());
        $this->assertSame(1250, FlowerPrice::where('flower_id', $this->stem->id)->firstOrFail()->price_cents);
    }

    /** Actual named routes enforce session signatures, financial validation, and Admin-only refunds. */
    public function test_http_finance_routes_validate_and_enforce_roles(): void
    {
        $order = $this->orders->store($this->input(), $this->supplier);
        $this->actingAs($this->supplier)->withSession([
            AccountSession::KEY => AccountSession::signature($this->supplier),
        ]);
        $this->getJson(route('supplier.sales-payments.data'))->assertOk()->assertJsonStructure(['flowers', 'orders', 'payments']);
        $payload = $this->paymentInput($order);
        $this->postJson(route('supplier.sales-payments.record'), [...$payload, 'status' => 'Confirmed'])
            ->assertUnprocessable()->assertJsonValidationErrors('status');
        $this->postJson(route('supplier.sales-payments.record'), $payload)
            ->assertCreated()->assertJsonStructure(['message', 'data' => ['payments']]);
        $payment = OrderPayment::firstOrFail();
        $this->assertSame('Pending', $payment->status);
        $this->postJson(route('supplier.sales-payments.refund', ['payment' => $payment->id]),
            $this->refundInput($payment, '1.00'))->assertForbidden();
        $account = $this->client->user;
        $this->actingAs($account)->withSession([
            AccountSession::KEY => AccountSession::signature($account),
        ]);
        $this->getJson(route('supplier.sales-payments.data'))->assertForbidden();
        $this->assertSame(0, PaymentRefund::count());
        $this->assertSame(20, $this->remaining($this->stem));
    }

    /** Read the current order financial DTO version, including cancelled historical orders. */
    private function orderFinanceRow(Order $order): array
    {
        return $this->financeRow('orders', (int) $order->id);
    }

    /** Read a fresh payment DTO version after confirmation/refund changes. */
    private function paymentFinanceRow(OrderPayment $payment): array
    {
        return $this->financeRow('payments', (int) $payment->id);
    }

    /** Find one explicitly identified current financial record. */
    private function financeRow(string $key, int $id): array
    {
        $rows = $this->sales->data($this->supplier)[$key];
        $row = collect($rows)->first(fn ($row) => (int) $row['dbId'] === $id);
        $this->assertNotNull($row, 'Missing '.$key.' DTO '.$id);

        return $row;
    }

    /** Build a complete payment request in exact decimal currency units. */
    private function paymentInput(Order $order, array $overrides = []): array
    {
        return array_replace([
            'request_key' => (string) Str::uuid(), 'order_id' => $order->id,
            'version' => $this->orderFinanceRow($order)['version'], 'method' => 'Cash',
            'amount' => Money::format((int) $order->fresh()->total_cents), 'reference' => null, 'remarks' => null,
        ], $overrides);
    }

    /** Record a real pending payment through the service. */
    private function record(Order $order, array $overrides = []): OrderPayment
    {
        $this->sales->recordPayment($this->paymentInput($order, $overrides), $this->supplier);

        return OrderPayment::where('order_id', $order->id)->firstOrFail();
    }

    /** Build a documented receipt-verification request. */
    private function confirmInput(OrderPayment $payment): array
    {
        return ['request_key' => (string) Str::uuid(), 'version' => $this->paymentFinanceRow($payment)['version'],
            'reason' => 'Verified the actual receipt and full collection.'];
    }

    /** Build a new bounded refund request from the current financial state. */
    private function refundInput(OrderPayment $payment, string $amount): array
    {
        return ['request_key' => (string) Str::uuid(), 'version' => $this->paymentFinanceRow($payment)['version'],
            'amount' => $amount, 'reason' => 'Approved return to the client.'];
    }

    /** Build a current selling-price edit without changing historical commercial snapshots. */
    private function priceInput(Flower $flower, string $amount): array
    {
        return ['request_key' => (string) Str::uuid(), 'version' => $this->financeRow('flowers', (int) $flower->id)['version'],
            'amount' => $amount];
    }

    /** Create one isolated account with the existing User hashing/casting behavior. */
    private function user(string $label, string $role): User
    {
        return User::forceCreate([
            'name' => 'Test '.$label, 'email' => $label.'@example.test', 'password' => 'Temporary-test-password!',
            'role' => $role, 'is_active' => true, 'must_change_password' => false,
        ]);
    }

    /** Define a catalog item and its current Sales-owned price. */
    private function item(string $code, string $type, int $price): Flower
    {
        $flower = Flower::forceCreate([
            'code' => $code, 'name' => 'Test '.$type, 'product_type' => $type,
            'category' => 'Roses', 'minimum_stock' => 1, 'description' => 'Test item.', 'status' => 'Active',
        ]);
        FlowerPrice::create(['flower_id' => $flower->id, 'price_cents' => $price]);

        return $flower;
    }

    /** Receive real test stock through the shared Inventory service. */
    private function receive(Flower $flower, int $quantity): void
    {
        $this->inventory->stockIn([
            'request_key' => (string) Str::uuid(), 'flower_id' => $flower->id, 'quantity' => $quantity,
            'unit' => $flower->product_type, 'received' => '2026-01-02', 'harvest' => '2026-01-01',
            'expiry' => null, 'remarks' => 'Test receipt.',
        ], $this->admin);
    }

    /** Read physical stock directly from batches, independently of reservation logic. */
    private function remaining(Flower $flower): int
    {
        return (int) InventoryBatch::where('flower_id', $flower->id)
            ->sum('remaining_quantity');
    }

    /** Supply a valid creation payload with a stable unique request identity. */
    private function input(array $overrides = []): array
    {
        return array_replace([
            'request_key' => (string) Str::uuid(), 'client_id' => $this->client->id,
            'method' => 'Customer Pickup', 'address' => null, 'notes' => null,
            'items' => [['flower_id' => $this->stem->id, 'quantity' => 2]],
        ], $overrides);
    }

    /** Build an optimistic editable payload using the current version. */
    private function edit(Order $order, array $overrides = []): array
    {
        $current = $order->fresh()->load('items');

        return array_replace([
            'client_id' => $current->client_id, 'method' => $current->method,
            'address' => $current->address, 'notes' => $current->notes,
            'items' => $current->items->map(fn ($item) => ['flower_id' => $item->flower_id, 'quantity' => $item->quantity])->all(),
            'version' => $this->orders->row($current, $this->supplier)['version'],
        ], $overrides);
    }

    /** Apply an explicit desired state through the same API used by the controller. */
    private function action(Order $order, string $status, User $actor, array $overrides = []): Order
    {
        return $this->orders->action($order, array_replace([
            'status' => $status, 'version' => $this->orders->row($order->fresh(), $actor)['version'],
        ], $overrides), $actor);
    }

    /** Assert a domain rejection while permitting subsequent atomicity assertions. */
    private function rejected(callable $action, ?string $field = null): void
    {
        try {
            $action(); $this->fail('Expected financial validation to reject this operation.');
        } catch (ValidationException $exception) {
            if ($field !== null) $this->assertArrayHasKey($field, $exception->errors());
        }
    }
}
```
