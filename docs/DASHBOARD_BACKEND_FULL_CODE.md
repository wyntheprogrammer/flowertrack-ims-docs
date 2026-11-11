# Supplier Dashboard backend — complete replacement files

Prepared 2026-11-11 against the reviewed app checkpoint `0ed6c7f` (Audit). Installation, PHP tests, full build and browser acceptance are pending. Do not call this module complete or pushed in the app repository yet.

Read [DASHBOARD_BACKEND_SETUP.md](DASHBOARD_BACKEND_SETUP.md) first. Each code block below is the **whole file** for the stated application path. Create the new files and replace the existing files; do not append the code blocks. No application migration or npm dependency is added.

Local checks: shared shell passed Node syntax checking; the refund join was exercised in SQLite with multiple refunds and an unfulfilled order; static checks verified isolated test connection and removal of sample chart code. PHP and browser rendering were not available in the assistant environment. Owner must run the supplied checks.

## 1. `app/Services/DashboardService.php`

```php
<?php

namespace App\Services;

use App\Models\Flower;
use App\Models\InventoryBatch;
use App\Models\InventoryMovement;
use App\Models\InventoryReservation;
use App\Models\Order;
use App\Models\OrderHistory;
use App\Models\SalesRecord;
use App\Models\User;
use Carbon\CarbonImmutable;
use Illuminate\Support\Facades\DB;
use Illuminate\Validation\ValidationException;

class DashboardService
{
    /** Read operational summaries without accessing Admin-only audit data. */
    public function data(User $actor, string $unit = 'Stem'): array
    {
        abort_unless(
            in_array($actor->role, [User::ROLE_ADMIN, User::ROLE_SUPPLIER], true)
                && $actor->is_active && ! $actor->must_change_password,
            403
        );

        if (! in_array($unit, ['Stem', 'Bouquet'], true)) {
            throw ValidationException::withMessages(['unit' => 'Choose Stem or Bouquet.']);
        }

        return DB::transaction(function () use ($unit) {
            $physical = InventoryBatch::where('spoiled', false)
                ->select('flower_id')->selectRaw('SUM(remaining_quantity) AS quantity')
                ->groupBy('flower_id')->pluck('quantity', 'flower_id');

            $holds = InventoryReservation::where('status', 'Active')
                ->select('flower_id')->selectRaw('SUM(quantity) AS quantity')
                ->groupBy('flower_id')->pluck('quantity', 'flower_id');

            $inventory = Flower::orderBy('code')->get()->map(function ($flower) use ($physical, $holds) {
                $onHand = $this->number($physical[$flower->id] ?? 0);
                $reserved = $this->number($holds[$flower->id] ?? 0);

                return [
                    'code' => $flower->code,
                    'name' => $flower->name,
                    'unit' => $flower->product_type,
                    'active' => $flower->status === 'Active',
                    'onHand' => $onHand,
                    'reserved' => $reserved,
                    'available' => $onHand - $reserved,
                    'minimum' => $this->number($flower->minimum_stock),
                ];
            });

            $totals = ['Stem' => 0, 'Bouquet' => 0];

            foreach ($inventory as $item) {
                if (! array_key_exists($item['unit'], $totals)) {
                    throw ValidationException::withMessages(['dashboard' => 'Unknown product stock unit.']);
                }

                $totals[$item['unit']] = $this->number($totals[$item['unit']] + $item['onHand']);
            }

            // Include active items with no receipts; preserve negative availability.
            $low = $inventory->filter(fn ($item) => $item['active']
                && $item['available'] < $item['minimum'])->values();

            $pending = Order::where('status', 'Pending')->count();

            // Aggregate refunds per order before joining to avoid multiplying sales.
            $refunds = DB::table('payment_refunds AS refunds')
                ->join('order_payments AS payments', 'payments.id', '=', 'refunds.payment_id')
                ->select('payments.order_id')
                ->selectRaw('SUM(refunds.amount_cents) AS refunded')
                ->groupBy('payments.order_id');

            $sales = DB::table('sales_records AS sales')
                ->leftJoinSub($refunds, 'refunds', 'refunds.order_id', '=', 'sales.order_id')
                ->selectRaw('COALESCE(SUM(sales.total_cents), 0) AS gross')
                ->selectRaw('COALESCE(SUM(refunds.refunded), 0) AS refunded')
                ->selectRaw('COALESCE(MAX(CASE WHEN COALESCE(refunds.refunded, 0) > sales.total_cents THEN 1 ELSE 0 END), 0) AS invalid_refund')
                ->first();

            $gross = $this->number($sales->gross);
            $refunded = $this->number($sales->refunded);

            if ((int) $sales->invalid_refund !== 0 || $refunded > $gross) {
                throw ValidationException::withMessages(['dashboard' => 'Refund exceeds a recorded sale. Review finance records.']);
            }

            // Completed-order snapshots, not mutable catalog prices or payment totals.
            // Refunds cannot be allocated to individual products without line-level refund data.
            $top = DB::table('order_items AS items')
                ->join('sales_records AS sales', 'sales.order_id', '=', 'items.order_id')
                ->select('items.flower_id', 'items.product_type')
                ->selectRaw('MAX(items.flower_code) AS code, MAX(items.name) AS name')
                ->selectRaw('SUM(items.quantity) AS quantity, SUM(items.subtotal_cents) AS gross')
                ->groupBy('items.flower_id', 'items.product_type')
                ->orderByDesc('gross')->orderBy('items.flower_id')->orderBy('items.product_type')
                ->limit(5)->get()->map(fn ($row) => [
                    'code' => $row->code,
                    'name' => $row->name,
                    'unit' => $row->product_type,
                    'quantity' => $this->number($row->quantity),
                    'grossCents' => $this->number($row->gross),
                ])->all();

            $chart = $inventory->where('unit', $unit)
                ->sort(function ($a, $b) {
                    return ($b['onHand'] <=> $a['onHand']) ?: strcmp($a['code'], $b['code']);
                })->take(6)->values()->all();

            return [
                'generatedAt' => now('Asia/Manila')->toIso8601String(),
                'timezone' => 'Asia/Manila',
                'summary' => [
                    'stems' => $totals['Stem'],
                    'bouquets' => $totals['Bouquet'],
                    'grossSalesCents' => $gross,
                    'refundCents' => $refunded,
                    'netSalesCents' => $gross - $refunded,
                    'lowStockItems' => $low->count(),
                    'pendingOrders' => $pending,
                ],
                'chart' => ['unit' => $unit, 'items' => $chart],
                'topProducts' => $top,
                'recent' => $this->recent(),
            ];
        });
    }

    /** Shared business activity only: no authentication/security/audit records. */
    private function recent(): array
    {
        $events = [];

        foreach (OrderHistory::with('order:id,code')->orderByDesc('created_at')->orderByDesc('id')->limit(5)->get() as $history) {
            $events[] = [
                'key' => 'order-'.$history->id,
                'date' => $this->timestamp($history->created_at),
                'icon' => 'bi-cart-check',
                'label' => ($history->order?->code ?? 'Order').' · '.$history->status,
                'detail' => 'Order update',
            ];
        }

        foreach (InventoryMovement::with('batch:id,code')->orderByDesc('created_at')->orderByDesc('id')->limit(5)->get() as $movement) {
            $events[] = [
                'key' => 'stock-'.$movement->id,
                'date' => $this->timestamp($movement->created_at),
                'icon' => 'bi-box-seam',
                'label' => ($movement->batch?->code ?? 'Batch').' · '.$movement->type,
                'detail' => 'Inventory movement',
            ];
        }

        foreach (SalesRecord::orderByDesc('created_at')->orderByDesc('id')->limit(5)->get(['id', 'code', 'created_at']) as $sale) {
            $events[] = [
                'key' => 'sale-'.$sale->id,
                'date' => $this->timestamp($sale->created_at),
                'icon' => 'bi-cash-stack',
                'label' => $sale->code.' · Sale recorded',
                'detail' => 'Fulfilled order',
            ];
        }

        usort($events, fn ($a, $b) => strcmp($b['date'], $a['date']) ?: strcmp($b['key'], $a['key']));

        return array_slice($events, 0, 5);
    }

    private function timestamp($value): string
    {
        return CarbonImmutable::parse($value)->setTimezone('Asia/Manila')->toIso8601String();
    }

    /** Reject unsupported totals instead of emitting rounded or fabricated figures. */
    private function number(mixed $value): int
    {
        $number = filter_var($value, FILTER_VALIDATE_INT);

        if ($number === false || $number < 0 || $number > 9007199254740991) {
            throw ValidationException::withMessages(['dashboard' => 'A dashboard total exceeds the supported numeric range.']);
        }

        return $number;
    }
}
```

## 2. `app/Http/Controllers/Supplier/DashboardController.php`

```php
<?php

namespace App\Http\Controllers\Supplier;

use App\Http\Controllers\Controller;
use App\Services\DashboardService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Validation\Rule;
use Illuminate\View\View;

class DashboardController extends Controller
{
    public function __construct(private readonly DashboardService $dashboard) {}

    public function index(Request $request): View
    {
        return view('supplier.dashboard', ['dashboard' => $this->snapshot($request)]);
    }

    public function data(Request $request): JsonResponse
    {
        return response()->json($this->snapshot($request))
            ->header('Cache-Control', 'private, no-store');
    }

    private function snapshot(Request $request): array
    {
        $values = $request->validate([
            'unit' => ['nullable', Rule::in(['Stem', 'Bouquet'])],
        ]);

        return $this->dashboard->data($request->user(), $values['unit'] ?? 'Stem');
    }
}
```

## 3. `resources/css/supplier/dashboard.css`

```css
/* ========================================
   LIVE SUPPLIER DASHBOARD
======================================== */
#supplierDashboard .dashboard-live-status{display:flex;align-items:center;justify-content:space-between;gap:12px;margin:0 0 18px;color:var(--supplier-muted);font-size:12px}
#supplierDashboard .dashboard-refresh{display:inline-flex;align-items:center;gap:7px;padding:9px 14px;border:1px solid var(--supplier-border);border-radius:10px;background:#fff;color:var(--supplier-primary);font-size:12px;font-weight:600;text-decoration:none;white-space:nowrap}
#supplierDashboard .dashboard-unit-form{display:flex;align-items:center;gap:8px;flex-wrap:wrap}
#supplierDashboard .dashboard-unit-form select{width:120px;min-height:36px;padding:7px 30px 7px 10px;border:1px solid var(--supplier-border);border-radius:9px;background-color:#fff;font:inherit;font-size:12px;color:var(--supplier-text)}
#supplierDashboard .dashboard-unit-form button{min-height:36px;padding:7px 12px;border:0;border-radius:9px;background:var(--supplier-primary);color:#fff;font:inherit;font-size:12px;font-weight:600}
#supplierDashboard .dashboard-muted{color:var(--supplier-muted);font-size:11px;line-height:1.6}
#supplierDashboard .summary-info{min-width:0}
#supplierDashboard .summary-info h3{overflow-wrap:anywhere;font-variant-numeric:tabular-nums}
#supplierDashboard .dashboard-note{margin:0 0 12px;color:var(--supplier-muted);font-size:11px;line-height:1.6}
#supplierDashboard .dashboard-empty{margin:0;padding:28px 12px;text-align:center;color:var(--supplier-muted);font-size:13px;line-height:1.6}
#supplierDashboard .product-name small{display:block;margin-top:4px;color:var(--supplier-muted);font-size:10px;line-height:1.5}
#supplierDashboard .product-name{overflow-wrap:anywhere}
#supplierDashboard .dashboard-stock-chart{display:block;width:100%;height:auto;max-height:320px}
#supplierDashboard .dashboard-refresh:focus-visible,#supplierDashboard .dashboard-unit-form :focus-visible{outline:2px solid var(--supplier-primary);outline-offset:3px}
@media(max-width:576px){#supplierDashboard .dashboard-live-status{align-items:flex-start}#supplierDashboard .panel-header{align-items:flex-start;flex-wrap:wrap}#supplierDashboard .dashboard-unit-form{margin-left:auto}#supplierDashboard .product-price{font-size:11px}}
```

## 4. `resources/css/app.css`

```css
@import 'bootstrap/dist/css/bootstrap.min.css';
@import 'bootstrap-icons/font/bootstrap-icons.css';
@import 'datatables.net-bs5/css/dataTables.bootstrap5.css';

@import './supplier/flowers.css';
@import './supplier/inventory.css';
@import './supplier/batches.css';
@import './supplier/spoilage-prediction.css';
@import './supplier/clients.css';
@import './supplier/orders.css';
@import './supplier/deliveries.css';
@import './supplier/sales-payments.css';
@import './supplier/reports.css';
@import './supplier/users.css';
@import './supplier/audit-trail.css';
@import './supplier/dashboard.css';



/* ========================================
   FLOWERTRACK IMS - SUPPLIER DESIGN
======================================== */

:root {
    --supplier-primary: #176b3a;
    --supplier-primary-dark: #073f23;
    --supplier-primary-light: #e4f2e4;
    --supplier-accent: #9bc99a;

    --supplier-bg: #f6f8f5;
    --supplier-surface: #ffffff;

    --supplier-text: #202b24;
    --supplier-muted: #7b867d;
    --supplier-border: #e8eee7;

    --supplier-danger: #e84b55;
    --supplier-warning: #f5b943;
}

* {
    box-sizing: border-box;
}

body {
    margin: 0;
    font-family: Arial, Helvetica, sans-serif;
    color: var(--supplier-text);
    background: var(--supplier-bg);
}

/* ========================================
   MAIN LAYOUT
======================================== */

.supplier-layout {
    display: flex;
    min-height: 100vh;
}

.supplier-main {
    flex: 1;
    min-width: 0;
    display: flex;
    flex-direction: column;
}

/* ========================================
   SIDEBAR
======================================== */

.supplier-sidebar {
    width: 250px;
    min-height: 100vh;
    flex-shrink: 0;
    display: flex;
    flex-direction: column;
    padding: 28px 16px 20px;
    color: #fff;
    background:
        radial-gradient(ellipse at bottom left, rgba(90, 153, 91, .24), transparent 40%),
        linear-gradient(160deg, #064b27, #06391f 75%, #052e1a);
}



.sidebar-brand {
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 0 8px 10px;
    color: #fff;
    text-align: center;
}

.sidebar-logo {
    width: 100%;
    padding: 0;
    margin: 0;
    background: transparent;
    border: none;
}

.sidebar-logo img {
    display: block;
    width: 100%;
    max-height: 125px;
    object-fit: contain;
    object-position: center;
}


.sidebar-brand .brand-name {
    font-size: 21px;
    font-weight: 700;
    font-style: normal;
    letter-spacing: -.4px;
    color: #ffffff;
}

.sidebar-brand .brand-subtitle {
    font-size: 10px;
    letter-spacing: 1.2px;
    color: #cce5ce;
}



.sidebar-nav {
    display: flex;
    flex-direction: column;
    gap: 3px;
}

.sidebar-link {
    display: flex;
    align-items: center;
    gap: 13px;
    min-height: 43px;
    padding: 8px 13px;
    border-radius: 9px;
    color: rgba(255,255,255,.86);
    text-decoration: none;
    font-size: 14px;
    line-height: 1.2;
    transition: .2s ease;
}


.sidebar-link:hover {
    color: #fff;
    background: rgba(255, 255, 255, .12);
}

.sidebar-link.active {
    color: #103b22;
    background: #d9efd0;
    font-weight: 700;
}

.sidebar-icon {
    width: 23px;
    text-align: center;
    font-size: 19px;
}

.sidebar-divider {
    height: 1px;
    margin: 24px 5px 18px;
    background: rgba(255, 255, 255, .2);
}

.sidebar-user {
    display: flex;
    align-items: center;
    gap: 11px;
    margin-top: auto;
    padding: 15px 5px 5px;
}

.sidebar-avatar {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 42px;
    height: 42px;
    flex-shrink: 0;
    border-radius: 50%;
    background: #d7ead4;
    color: var(--supplier-primary-dark);
    font-size: 19px;
    font-weight: 700;
}

.sidebar-user-info strong {
    display: block;
    color: #fff;
    font-size: 13px;
}

.sidebar-user-info small {
    color: rgba(255, 255, 255, .7);
    font-size: 11px;
}

/* ========================================
   TOP NAVBAR
======================================== */

.supplier-navbar {
    min-height: 76px;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 20px;
    padding: 13px 26px;
    background: #fff;
    border-bottom: 1px solid var(--supplier-border);
}

.navbar-left,
.navbar-right {
    display: flex;
    align-items: center;
    gap: 18px;
}

.navbar-menu {
    border: 0;
    background: transparent;
    color: var(--supplier-primary-dark);
    font-size: 25px;
    cursor: pointer;
}


.navbar-brand {
    display: flex;
    flex-direction: column;
    justify-content: center;
    margin-right: auto;
    line-height: 1.2;
}

.navbar-brand-name {
    color: var(--supplier-primary-dark);
    font-size: 19px;
    font-weight: 800;
    letter-spacing: -.4px;
}

.navbar-brand-subtitle {
    margin-top: 4px;
    color: var(--supplier-muted);
    font-size: 10px;
    letter-spacing: 1px;
}



.navbar-icon {
    position: relative;
    border: 0;
    background: transparent;
    color: var(--supplier-primary-dark);
    font-size: 21px;
}

.notification-count {
    position: absolute;
    top: -7px;
    right: -7px;
    display: flex;
    align-items: center;
    justify-content: center;
    width: 17px;
    height: 17px;
    border-radius: 50%;
    color: #fff;
    background: var(--supplier-danger);
    font-size: 10px;
}

.navbar-date {
    padding: 10px 14px;
    border: 1px solid var(--supplier-border);
    border-radius: 25px;
    color: var(--supplier-text);
    background: #fff;
    font-size: 12px;
}

/* ========================================
   DASHBOARD CONTENT
======================================== */

.dashboard-content {
    padding: 22px;
}

.dashboard-heading {
    margin-bottom: 17px;
}

.dashboard-heading h1 {
    margin: 0;
    font-size: 25px;
    font-weight: 700;
}

.dashboard-heading p {
    margin: 5px 0 0;
    color: var(--supplier-muted);
    font-size: 13px;
}

/* ========================================
   WELCOME BANNER
======================================== */



.welcome-banner {
    position: relative;
    overflow: hidden;
    min-height: 155px;
    display: flex;
    align-items: center;
    padding: 25px 30px;
    margin-bottom: 18px;
    border-radius: 17px;
    background-color: #f1f7ed;
    background-image:
        linear-gradient(
            90deg,
            #f1f7ed 0%,
            rgba(241,247,237,.95) 40%,
            rgba(241,247,237,.15) 75%,
            rgba(241,247,237,0) 100%
        ),
        url('/images/flower-banner.png');
    background-position: center, right center;
    background-size: 100% 100%, auto 100%;
    background-repeat: no-repeat;
    border-left: 4px solid var(--supplier-primary);
}

.welcome-banner::after {
    content: none;
}

.welcome-text {
    position: relative;
    z-index: 1;
    width: 55%;
}

.welcome-text h2 {
    margin: 0 0 8px;
    color: #151d17;
    font-size: clamp(24px, 3vw, 36px);
    font-weight: 800;
}

.welcome-text h2 span {
    color: var(--supplier-primary);
    font-style: italic;
}

.welcome-text p {
    margin: 0;
    color: #3f4941;
    font-size: 15px;
}

/* ========================================
   SUMMARY CARDS
======================================== */

.summary-grid {
    display: grid;
    grid-template-columns: repeat(4, minmax(0, 1fr));
    gap: 15px;
    margin-bottom: 18px;
}

.summary-card {
    position: relative;
    overflow: hidden;
    min-height: 116px;
    display: flex;
    align-items: center;
    gap: 14px;
    padding: 17px;
    border: 1px solid var(--supplier-border);
    border-radius: 15px;
    background: #fff;
    box-shadow: 0 3px 12px rgba(28, 65, 37, .035);
}

.summary-icon {
    width: 53px;
    height: 53px;
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 50%;
    color: #fff;
    background: var(--supplier-primary);
    font-size: 24px;
}

.summary-icon.red {
    background: var(--supplier-danger);
}

.summary-icon.yellow {
    color: #493812;
    background: #ffc451;
}

.summary-info small {
    display: block;
    margin-bottom: 4px;
    color: var(--supplier-muted);
    font-size: 12px;
}

.summary-info h3 {
    margin: 0;
    color: #18231b;
    font-size: 24px;
    font-weight: 800;
}

.summary-change {
    display: block;
    margin-top: 4px;
    color: var(--supplier-primary);
    font-size: 12px;
    font-weight: 700;
}

.summary-change.negative {
    color: var(--supplier-danger);
}

/* ========================================
   DASHBOARD PANELS
======================================== */

.dashboard-grid {
    display: grid;
    grid-template-columns: minmax(0, 2.1fr) minmax(270px, .9fr);
    gap: 17px;
    margin-bottom: 18px;
}

.dashboard-panel {
    min-width: 0;
    padding: 20px;
    border: 1px solid var(--supplier-border);
    border-radius: 15px;
    background: #fff;
    box-shadow: 0 3px 12px rgba(28, 65, 37, .035);
}


.panel-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
    min-height: 32px;
    margin-bottom: 18px;
}

.panel-title {
    margin: 0;
    font-size: 17px;
    font-weight: 700;
}

.panel-title span {
    margin-right: 8px;
    color: var(--supplier-primary);
}

.panel-link {
    color: var(--supplier-primary);
    font-size: 12px;
    font-weight: 700;
    text-decoration: none;
}

/* ========================================
   INVENTORY CHART
======================================== */

.chart-legend {
    display: flex;
    justify-content: flex-end;
    gap: 18px;
    margin-bottom: 8px;
    color: var(--supplier-muted);
    font-size: 11px;
}

.legend-dot {
    display: inline-block;
    width: 8px;
    height: 8px;
    margin-right: 5px;
    border-radius: 50%;
    background: var(--supplier-primary);
}

.legend-dot.light {
    background: #a6ce91;
}

.inventory-chart {
    width: 100%;
    height: auto;
    display: block;
}

.chart-label {
    fill: #66736a;
    font-size: 11px;
}

/* ========================================
   QUICK ACTIONS
======================================== */

.quick-actions {
    display: grid;
    grid-template-columns: repeat(2, minmax(0, 1fr));
    gap: 12px;
}

.quick-action {
    min-height: 91px;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 8px;
    border: 1px solid var(--supplier-border);
    border-radius: 12px;
    color: var(--supplier-text);
    background: #fff;
    text-decoration: none;
    transition: .2s ease;
}

.quick-action:hover {
    border-color: var(--supplier-accent);
    background: #f5faf3;
    transform: translateY(-2px);
}

.quick-action span {
    color: var(--supplier-primary);
    font-size: 25px;
}

.quick-action strong {
    font-size: 12px;
}



/* Equal height for Quick Actions panel */
.dashboard-grid > .dashboard-panel:nth-child(2) {
    display: flex;
    flex-direction: column;
}

.dashboard-grid > .dashboard-panel:nth-child(2) .quick-actions {
    flex: 1;
    grid-template-rows: repeat(2, minmax(0, 1fr));
}

.dashboard-grid > .dashboard-panel:nth-child(2) .quick-action {
    min-height: 0;
}

/* ========================================
   LOWER DASHBOARD PANELS
======================================== */

.lower-grid {
    display: grid;
    grid-template-columns: 1.1fr 1.1fr .95fr;
    gap: 16px;
}

.product-row,
.activity-row {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 11px 0;
    border-bottom: 1px solid #f0f2ef;
}

.product-row:last-child,
.activity-row:last-child {
    border-bottom: 0;
}

.product-rank {
    width: 29px;
    height: 29px;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-shrink: 0;
    border-radius: 50%;
    color: var(--supplier-text);
    background: #f0f3ef;
    font-size: 12px;
    font-weight: 700;
}

.product-image {
    width: 35px;
    height: 35px;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-shrink: 0;
    border-radius: 50%;
    background: #f5f8f2;
    font-size: 22px;
}

.product-name,
.activity-description {
    flex: 1;
    min-width: 0;
    font-size: 12px;
}

.product-price,
.activity-value {
    color: var(--supplier-text);
    font-size: 12px;
    font-weight: 700;
    white-space: nowrap;
}

.activity-icon {
    width: 34px;
    height: 34px;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-shrink: 0;
    border-radius: 50%;
    color: var(--supplier-primary);
    background: #eff6ed;
    font-size: 17px;
}

.activity-description small {
    display: block;
    margin-top: 4px;
    color: var(--supplier-muted);
    font-size: 10px;
}

/* ========================================
   PROMOTIONAL PANEL
======================================== */

.promo-panel {
    min-height: 220px;
    overflow: hidden;
    position: relative;
    display: flex;
    flex-direction: column;
    justify-content: center;
    padding: 23px;
    border-radius: 15px;
    background:
        linear-gradient(
            90deg,
            rgba(3, 48, 27, .94) 0%,
            rgba(3, 48, 27, .78) 52%,
            rgba(3, 48, 27, .12) 100%
        ),
        url('/images/bloom-business.jpg') center / cover no-repeat;
}

.promo-panel::after {
    content: none;
}

.promo-panel h3 {
    position: relative;
    z-index: 1;
    max-width: 260px;
    margin: 0 0 8px;
    color: #fff;
    font-family: Georgia, serif;
    font-size: 28px;
    font-style: italic;
    line-height: 1.15;
}

.promo-panel p {
    position: relative;
    z-index: 1;
    max-width: 260px;
    margin-bottom: 15px;
    color: rgba(255,255,255,.9);
    font-size: 12px;
}

.promo-button {
    position: relative;
    z-index: 1;
    width: fit-content;
    padding: 9px 15px;
    border-radius: 25px;
    color: var(--supplier-primary-dark);
    background: #fff;
    font-size: 11px;
    font-weight: 700;
    text-decoration: none;
}

/* ========================================
   RESPONSIVE
======================================== */

@media (max-width: 1100px) {
    .summary-grid {
        grid-template-columns: repeat(2, minmax(0, 1fr));
    }

    .lower-grid {
        grid-template-columns: repeat(2, minmax(0, 1fr));
    }

    .promo-panel {
        grid-column: span 2;
    }
}

@media (max-width: 768px) {
    .supplier-sidebar {
        width: 76px;
        padding: 22px 8px;
    }

    .sidebar-brand {
        padding: 0 0 25px;
        text-align: center;
    }

    .sidebar-brand .brand-name {
        font-size: 13px;
        letter-spacing: 0;
    }

    .sidebar-brand .brand-subtitle,
    .sidebar-link .sidebar-label,
    .sidebar-user-info,
    .sidebar-divider {
        display: none;
    }

    .sidebar-link {
        justify-content: center;
        padding: 12px 5px;
    }

    .sidebar-icon {
        font-size: 20px;
    }

    .sidebar-user {
        justify-content: center;
    }

    .navbar-search {
        width: 40vw;
    }

    .dashboard-content {
        padding: 15px;
    }

    .dashboard-grid {
        grid-template-columns: 1fr;
    }
}

@media (max-width: 576px) {
    .supplier-navbar {
        min-height: 65px;
        padding: 10px;
    }

    .navbar-left,
    .navbar-right {
        gap: 8px;
    }

    .navbar-search {
        width: 38vw;
        padding: 8px;
    }

    .navbar-date {
        display: none;
    }

    .navbar-brand-name {
        font-size: 15px;
    }

    .navbar-brand-subtitle {
        font-size: 8px;
        letter-spacing: .5px;
    }

    .summary-grid {
        grid-template-columns: 1fr;
    }

    .lower-grid {
        grid-template-columns: 1fr;
    }

    .promo-panel {
        grid-column: auto;
    }

    .welcome-banner {
        min-height: 125px;
        padding: 20px;
    }

    .welcome-banner::after {
        right: -15px;
        font-size: 95px;
    }

    .dashboard-content {
        padding: 12px;
    }

    .dashboard-panel {
        padding: 15px;
    }
}



/* Supplier UI final-review corrections */
.supplier-layout.sidebar-collapsed .supplier-sidebar{width:76px;padding:22px 8px}
.supplier-layout.sidebar-collapsed .sidebar-label,.supplier-layout.sidebar-collapsed .sidebar-user-info,.supplier-layout.sidebar-collapsed .sidebar-divider{display:none}
.supplier-layout.sidebar-collapsed .sidebar-link,.supplier-layout.sidebar-collapsed .sidebar-user{justify-content:center}
.supplier-layout.sidebar-collapsed .sidebar-logo img{max-width:52px;width:100%;height:auto}
.supplier-notifications{position:relative}
.supplier-notification-panel{position:absolute;right:0;top:calc(100% + 14px);z-index:1050;width:300px;max-width:calc(100vw - 32px);padding:18px;background:#fff;border:1px solid var(--supplier-border);border-radius:12px;box-shadow:0 12px 30px rgb(0 0 0 / 12%)}
.supplier-notification-panel[hidden]{display:none}
.supplier-notification-panel h2{margin:0 0 12px;font-size:15px;font-weight:700;color:var(--supplier-text)}
.supplier-notification-panel h2 small{margin-left:8px;font-size:10px;font-weight:400;color:var(--supplier-muted)}
.supplier-notification-panel a{display:block;padding:10px 0;border-bottom:1px solid var(--supplier-border);color:var(--supplier-primary);font-size:12px;text-decoration:none}
.supplier-notification-panel p{margin:12px 0 0;color:var(--supplier-muted);font-size:11px;line-height:1.6}
.supplier-chart-note,.dashboard-sample-label{font-size:11px;line-height:1.6;color:var(--supplier-muted)}
.supplier-chart-note{margin:0 0 10px}.dashboard-sample-label{display:block;margin-top:8px}
.navbar-menu:focus-visible,.navbar-icon:focus-visible,.supplier-notification-panel a:focus-visible{outline:2px solid var(--supplier-primary);outline-offset:3px}
@media(max-width:768px){
 .supplier-layout:not(.sidebar-expanded) .sidebar-logo img{max-width:52px;width:100%;height:auto}
 .supplier-layout.sidebar-expanded .supplier-sidebar{width:230px;padding:22px 16px}
 .supplier-layout.sidebar-expanded .sidebar-label,.supplier-layout.sidebar-expanded .sidebar-user-info,.supplier-layout.sidebar-expanded .sidebar-divider{display:block}
 .supplier-layout.sidebar-expanded .sidebar-link{justify-content:flex-start;padding:12px 14px}
 .supplier-layout.sidebar-expanded .sidebar-user{justify-content:flex-start}
 .supplier-layout.sidebar-expanded .supplier-sidebar{position:fixed;inset:0 auto 0 0;height:100dvh;overflow-y:auto;z-index:1040;box-shadow:8px 0 25px rgb(0 0 0 / 15%)}
 .supplier-layout.sidebar-expanded .supplier-main{margin-left:76px}
}
```

## 5. `resources/js/supplier/shell.js`

```javascript
/* Shared Supplier navigation and notification-panel controls. */
function initializeSupplierShell() {
    const layout = document.querySelector('.supplier-layout');
    if (!layout || layout.dataset.shellReady) return;
    layout.dataset.shellReady = 'true';
    const menu = document.getElementById('supplierMenuToggle');
    const sidebar = document.getElementById('supplierSidebar');
    const mobile = window.matchMedia('(max-width: 768px)');
    const notificationButton = document.getElementById('supplierNotificationToggle');
    const notificationPanel = document.getElementById('supplierNotificationPanel');

    function syncMenu() {
        const expanded = mobile.matches ? layout.classList.contains('sidebar-expanded') : !layout.classList.contains('sidebar-collapsed');
        menu?.setAttribute('aria-expanded', String(expanded));
    }
    menu?.addEventListener('click', () => {
        layout.classList.toggle(mobile.matches ? 'sidebar-expanded' : 'sidebar-collapsed');
        syncMenu();
    });
    mobile.addEventListener('change', () => {
        layout.classList.remove('sidebar-expanded', 'sidebar-collapsed');
        syncMenu();
    });
    syncMenu();

    function closeNotifications() {
        if (notificationPanel) notificationPanel.hidden = true;
        notificationButton?.setAttribute('aria-expanded', 'false');
    }
    notificationButton?.addEventListener('click', () => {
        if (!notificationPanel) return;
        notificationPanel.hidden = !notificationPanel.hidden;
        notificationButton.setAttribute('aria-expanded', String(!notificationPanel.hidden));
    });
    document.addEventListener('click', (event) => {
        if (!event.target.closest('.supplier-notifications')) closeNotifications();
        if (mobile.matches && !sidebar?.contains(event.target) && !menu?.contains(event.target)) {
            layout.classList.remove('sidebar-expanded'); syncMenu();
        }
    });
    document.addEventListener('keydown', (event) => {
        if (event.key !== 'Escape') return;
        if (notificationPanel && !notificationPanel.hidden) { closeNotifications(); notificationButton?.focus(); }
        if (mobile.matches && layout.classList.contains('sidebar-expanded')) {
            layout.classList.remove('sidebar-expanded'); syncMenu(); menu?.focus();
        }
    });


}
if (document.readyState === 'loading') document.addEventListener('DOMContentLoaded', initializeSupplierShell, { once: true }); else initializeSupplierShell();
```

## 6. `resources/views/supplier/dashboard.blade.php`

```blade
@extends('layouts.supplier')

@section('title', 'Dashboard')

@section('content')
    @php
        $summary = $dashboard['summary'];
        $chart = $dashboard['chart'];
        $isAdmin = auth()->user()->role === \App\Models\User::ROLE_ADMIN;
        $money = static fn (int $cents): string => '₱'
            .number_format(intdiv($cents, 100), 0, '.', ',')
            .'.'.str_pad((string) ($cents % 100), 2, '0', STR_PAD_LEFT);
        $generated = \Carbon\CarbonImmutable::parse($dashboard['generatedAt']);
    @endphp

    <div id="supplierDashboard">
        <section class="welcome-banner">
            <div class="welcome-text">
                <h2>Welcome back, <span>{{ auth()->user()->name }}!</span></h2>
                <p>Here's what's happening with your flower business today.</p>
            </div>
        </section>

        <div class="dashboard-live-status">
            <span>Updated {{ $generated->format('M d, Y · h:i A') }} PHT</span>
            <a class="dashboard-refresh" href="{{ route('supplier.dashboard', ['unit' => $chart['unit']]) }}">
                <i class="bi bi-arrow-clockwise" aria-hidden="true"></i> Refresh
            </a>
        </div>

        <section class="summary-grid" aria-label="Business summary">
            <div class="summary-card">
                <div class="summary-icon"><i class="bi bi-box-seam" aria-hidden="true"></i></div>
                <div class="summary-info">
                    <small>On-Hand Inventory</small>
                    <h3>{{ number_format($summary['stems']) }} <span class="dashboard-muted">stems</span></h3>
                    <span class="summary-change">{{ number_format($summary['bouquets']) }} bouquets</span>
                </div>
            </div>

            <div class="summary-card">
                <div class="summary-icon red"><i class="bi bi-cash-stack" aria-hidden="true"></i></div>
                <div class="summary-info">
                    <small>Net Sales · All Time</small>
                    <h3>{{ $money($summary['netSalesCents']) }}</h3>
                    <span class="dashboard-muted">Completed sales less refunds</span>
                </div>
            </div>

            <div class="summary-card">
                <div class="summary-icon"><i class="bi bi-exclamation-triangle" aria-hidden="true"></i></div>
                <div class="summary-info">
                    <small>Low Stock Items</small>
                    <h3>{{ number_format($summary['lowStockItems']) }}</h3>
                    <span class="dashboard-muted">Active items below minimum available</span>
                </div>
            </div>

            <div class="summary-card">
                <div class="summary-icon yellow"><i class="bi bi-clock-history" aria-hidden="true"></i></div>
                <div class="summary-info">
                    <small>Pending Orders</small>
                    <h3>{{ number_format($summary['pendingOrders']) }}</h3>
                    <span class="dashboard-muted">Awaiting confirmation</span>
                </div>
            </div>
        </section>

        <section class="dashboard-grid">
            <div class="dashboard-panel">
                <div class="panel-header">
                    <h4 class="panel-title"><i class="bi bi-bar-chart-line" aria-hidden="true"></i> Inventory Overview</h4>
                    <form method="GET" action="{{ route('supplier.dashboard') }}" class="dashboard-unit-form">
                        <label class="visually-hidden" for="dashboardUnit">Product unit</label>
                        <select id="dashboardUnit" name="unit">
                            <option value="Stem" @selected($chart['unit'] === 'Stem')>Stems</option>
                            <option value="Bouquet" @selected($chart['unit'] === 'Bouquet')>Bouquets</option>
                        </select>
                        <button type="submit">Apply</button>
                    </form>
                </div>

                <p class="dashboard-note">Current on-hand stock · up to six items with the highest stock. Reservations are included in on-hand quantities.</p>

                @if (count($chart['items']))
                    @php
                        $maximum = max(1, ...array_column($chart['items'], 'onHand'));
                        $height = max(170, 30 + count($chart['items']) * 36);
                    @endphp
                    <svg class="dashboard-stock-chart" viewBox="0 0 700 {{ $height }}"
                         xmlns="http://www.w3.org/2000/svg" role="img"
                         aria-labelledby="liveStockTitle liveStockDescription">
                        <title id="liveStockTitle">Current {{ strtolower($chart['unit']) }} inventory</title>
                        <desc id="liveStockDescription">{{ collect($chart['items'])->map(fn ($item) => $item['name'].': '.$item['onHand'])->join('; ') }}</desc>
                        @foreach ($chart['items'] as $index => $item)
                            @php
                                $y = 18 + $index * 36;
                                $width = round($item['onHand'] / $maximum * 440, 2);
                            @endphp
                            <text x="0" y="{{ $y + 15 }}" fill="#66736a" font-size="12">{{ \Illuminate\Support\Str::limit($item['name'], 19) }}</text>
                            <rect x="150" y="{{ $y }}" width="440" height="23" rx="6" fill="#eef5eb" />
                            <rect x="150" y="{{ $y }}" width="{{ $width }}" height="23" rx="6" fill="#176b3a" />
                            <text x="605" y="{{ $y + 15 }}" fill="#202b24" font-size="12">{{ number_format($item['onHand']) }}</text>
                        @endforeach
                    </svg>
                @else
                    <p class="dashboard-empty">No {{ strtolower($chart['unit']) }} products registered yet.</p>
                @endif
            </div>

            <div class="dashboard-panel">
                <div class="panel-header"><h4 class="panel-title"><i class="bi bi-lightning-charge" aria-hidden="true"></i> Quick Actions</h4></div>
                <div class="quick-actions">
                    <a href="{{ route('supplier.flowers', ['create' => 1]) }}" class="quick-action">
                        <span><i class="bi bi-plus-circle" aria-hidden="true"></i></span><strong>Add Flower</strong>
                    </a>
                    <a href="{{ route('supplier.orders', ['create' => 1]) }}" class="quick-action">
                        <span><i class="bi bi-cart-plus" aria-hidden="true"></i></span><strong>New Order</strong>
                    </a>
                    <a href="{{ route($isAdmin ? 'supplier.reports' : 'supplier.inventory') }}" class="quick-action">
                        <span><i class="bi {{ $isAdmin ? 'bi-file-earmark-bar-graph' : 'bi-box-seam' }}" aria-hidden="true"></i></span>
                        <strong>{{ $isAdmin ? 'View Reports' : 'View Inventory' }}</strong>
                    </a>
                    <a href="{{ route('supplier.clients') }}" class="quick-action">
                        <span><i class="bi bi-people" aria-hidden="true"></i></span><strong>Manage Clients</strong>
                    </a>
                </div>
            </div>
        </section>

        <section class="lower-grid">
            <div class="dashboard-panel">
                <div class="panel-header">
                    <h4 class="panel-title"><i class="bi bi-trophy" aria-hidden="true"></i> Top Selling Products</h4>
                    <a href="{{ route('supplier.sales-payments') }}" class="panel-link">View All →</a>
                </div>
                <p class="dashboard-note">All-time gross item sales · before refunds · excludes delivery fees. Names use retained order snapshots.</p>
                @forelse ($dashboard['topProducts'] as $index => $product)
                    <div class="product-row">
                        <span class="product-rank">{{ $index + 1 }}</span>
                        <span class="product-image"><i class="bi bi-flower1" aria-hidden="true"></i></span>
                        <span class="product-name">{{ $product['name'] }}
                            <small>{{ $product['code'] }} · {{ number_format($product['quantity']) }} {{ $product['unit'] === 'Stem' ? 'stems' : 'bouquets' }}</small>
                        </span>
                        <span class="product-price">{{ $money($product['grossCents']) }}</span>
                    </div>
                @empty
                    <p class="dashboard-empty">No completed product sales yet.</p>
                @endforelse
            </div>

            <div class="dashboard-panel">
                <div class="panel-header">
                    <h4 class="panel-title"><i class="bi bi-activity" aria-hidden="true"></i> Recent Activities</h4>
                    @if ($isAdmin)
                        <a href="{{ route('supplier.audit-trail') }}" class="panel-link">Audit Trail →</a>
                    @endif
                </div>
                @forelse ($dashboard['recent'] as $activity)
                    <div class="activity-row">
                        <span class="activity-icon"><i class="bi {{ $activity['icon'] }}" aria-hidden="true"></i></span>
                        <span class="activity-description">{{ $activity['label'] }}
                            <small>{{ \Carbon\CarbonImmutable::parse($activity['date'])->format('M d, Y · h:i A') }} PHT</small>
                        </span>
                    </div>
                @empty
                    <p class="dashboard-empty">No operational activity yet.</p>
                @endforelse
            </div>

            <div class="promo-panel">
                <h3>Bloom<br>Your Business</h3>
                <p>Manage your flowers and grow your business with FlowerTrack.</p>
                <a href="{{ route('supplier.inventory') }}" class="promo-button">View Inventory →</a>
            </div>
        </section>
    </div>
@endsection
```

## 7. `resources/views/layouts/supplier.blade.php`

```blade
<!DOCTYPE html>
<html lang="{{ str_replace('_', '-', app()->getLocale()) }}">

<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <title>@yield('title', 'Dashboard') - FlowerTrack IMS</title>

    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>

<body>

    <div class="supplier-layout">

        <!-- Sidebar -->
        <aside class="supplier-sidebar" id="supplierSidebar">

            <div class="sidebar-brand">
                <div class="sidebar-logo">
                    <img src="{{ asset('images/gemmas-logo.png') }}" alt="Gemma's Flower Shop Logo">
                </div>
            </div>

            <div class="sidebar-divider"></div>

            @php
                $supplierNavigation = [
                    ['dashboard', 'Dashboard', 'bi-house-door'],
                    ['flowers', 'Flower Management', 'bi-flower1'],
                    ['inventory', 'Inventory Management', 'bi-box-seam'],
                    ['batches', 'Batch Management', 'bi-layers'],
                    ['spoilage-prediction', 'Spoilage Prediction', 'bi-graph-up-arrow'],
                    ['clients', 'Client Management', 'bi-people'],
                    ['orders', 'Order Management', 'bi-cart3'],
                    ['deliveries', 'Delivery Management', 'bi-truck'],
                    ['sales-payments', 'Sales & Payments', 'bi-cash-coin'],
                    ['reports', 'Reports', 'bi-bar-chart'],
                    ['users', 'User Management', 'bi-person-gear'],
                    ['audit-trail', 'Audit Trail', 'bi-clock-history'],
                ];
            @endphp

            @php
                $account = auth()->user();
                $isAdmin = $account->role === \App\Models\User::ROLE_ADMIN;
                $roleLabel = $isAdmin ? 'Administrator' : 'Supplier';
                $dashboardSummary = $dashboard['summary'] ?? null;
                $alertCount = $dashboardSummary
                    ? $dashboardSummary['lowStockItems'] + $dashboardSummary['pendingOrders']
                    : null;
            @endphp

            <nav class="sidebar-nav">
                @foreach ($supplierNavigation as [$module, $label, $icon])
                    @continue(! $isAdmin && in_array($module, ['reports', 'users', 'audit-trail'], true))
                    @php
                        $routeName = 'supplier.' . $module;
                        $isActive = request()->routeIs($routeName, $routeName . '.*');
                    @endphp

                    <a href="{{ route($routeName) }}" class="sidebar-link {{ $isActive ? 'active' : '' }}" @if ($isActive)
                    aria-current="page" @endif aria-label="{{ $label }}" title="{{ $label }}">
                        <span class="sidebar-icon">
                            <i class="bi {{ $icon }}" aria-hidden="true"></i>
                        </span>

                        <span class="sidebar-label">{{ $label }}</span>
                    </a>
                @endforeach
            </nav>


            <div class="sidebar-divider"></div>

            <div class="sidebar-user">
                <div class="sidebar-avatar">{{ mb_strtoupper(mb_substr($account->name, 0, 1)) }}</div>
                <div class="sidebar-user-info">
                    <strong>{{ $account->name }}</strong>
                    <small>{{ $roleLabel }}</small>
                </div>
            </div>

            <form method="POST" action="{{ route('logout') }}" class="mt-3">
                @csrf

                <button type="submit" class="btn btn-outline-light w-100">
                    <i class="bi bi-box-arrow-right" aria-hidden="true"></i>
                    Sign Out
                </button>
            </form>

        </aside>

        <!-- Main Area -->
        <main class="supplier-main">

            <!-- Top Navbar -->
            <header class="supplier-navbar">

                <div class="navbar-left">


                    <button class="navbar-menu" type="button" id="supplierMenuToggle" aria-controls="supplierSidebar"
                        aria-expanded="true" aria-label="Toggle navigation">
                        <i class="bi bi-list" aria-hidden="true"></i>
                    </button>

                    <div class="navbar-brand">
                        <div class="navbar-brand-name">FlowerTrack IMS</div>
                        <div class="navbar-brand-subtitle">Inventory Management System</div>
                    </div>

                </div>

                <div class="navbar-right">

                    <div class="supplier-notifications">
                        <button class="navbar-icon" type="button" id="supplierNotificationToggle"
                            aria-controls="supplierNotificationPanel" aria-expanded="false"
                            aria-label="Show inventory and order alerts">
                            <i class="bi bi-bell" aria-hidden="true"></i>
                            @if ($alertCount !== null && $alertCount > 0)
                                <span class="notification-count">{{ $alertCount > 99 ? '99+' : $alertCount }}</span>
                            @endif
                        </button>
                        <section id="supplierNotificationPanel" class="supplier-notification-panel"
                            aria-labelledby="supplierNotificationHeading" hidden>
                            <h2 id="supplierNotificationHeading">Operational Alerts</h2>
                            <a href="{{ route('supplier.inventory') }}">
                                Review low-stock items
                                @if ($dashboardSummary) · {{ number_format($dashboardSummary['lowStockItems']) }} @endif
                            </a>
                            <a href="{{ route('supplier.orders') }}">
                                Review pending orders
                                @if ($dashboardSummary) · {{ number_format($dashboardSummary['pendingOrders']) }} @endif
                            </a>
                            @if ($dashboardSummary)
                                <p>Counts reflect the Dashboard snapshot. Refresh the Dashboard for current counts.</p>
                            @else
                                <p><a href="{{ route('supplier.dashboard') }}">Open Dashboard for current counts.</a></p>
                            @endif
                        </section>
                    </div>

                    <div class="navbar-date">
                        <i class="bi bi-calendar3" aria-hidden="true"></i> &nbsp; {{ now('Asia/Manila')->format('M d, Y') }}
                    </div>

                </div>

            </header>

            <!-- Page Content -->
            <section class="dashboard-content">
                @yield('content')
            </section>

        </main>

    </div>

</body>

</html>
```

## 8. `routes/web.php`

```php
<?php

use App\Http\Controllers\AuthController;
use App\Http\Controllers\Supplier\AuditTrailController;
use App\Http\Controllers\Supplier\BatchController;
use App\Http\Controllers\Supplier\ClientController;
use App\Http\Controllers\Supplier\DeliveryController;
use App\Http\Controllers\Supplier\DashboardController;
use App\Http\Controllers\Supplier\FlowerController;
use App\Http\Controllers\Supplier\InventoryController;
use App\Http\Controllers\Supplier\OrderController;
use App\Http\Controllers\Supplier\ReportController;
use App\Http\Controllers\Supplier\SalesPaymentController;
use App\Http\Controllers\Supplier\UserController;
use App\Http\Middleware\EnsureAccountSessionIsCurrent;
use Illuminate\Support\Facades\Route;

// ========================================
// PUBLIC PAGES AND LOGIN
// ========================================

Route::view('/', 'welcome');

Route::get('/login', [AuthController::class, 'showLogin'])
    ->name('login');

Route::post('/login', [AuthController::class, 'login'])
    ->middleware('guest')
    ->name('login.store');

// ========================================
// AUTHENTICATED ACCOUNTS
// ========================================

Route::middleware('auth')->group(function () {
    Route::post('/logout', [AuthController::class, 'logout'])
        ->name('logout');

    Route::middleware([
        'account.active',
        EnsureAccountSessionIsCurrent::class,
    ])->group(function () {

        // Accessible while a password change is required.
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

        // ========================================
        // DELIVERY PERSONNEL
        // ========================================

        Route::prefix('delivery')
            ->name('delivery.')
            ->middleware(['password.changed', 'role:delivery'])
            ->group(function () {
                Route::get('/deliveries', [DeliveryController::class, 'index'])
                    ->name('deliveries');

                Route::get('/deliveries/data', [DeliveryController::class, 'data'])
                    ->name('deliveries.data');

                Route::get('/deliveries/{delivery}/proof', [DeliveryController::class, 'proof'])
                    ->whereNumber('delivery')
                    ->name('deliveries.proof');

                Route::get('/deliveries/{delivery}', [DeliveryController::class, 'show'])
                    ->whereNumber('delivery')
                    ->name('deliveries.show');

                Route::post('/deliveries/{delivery}/action', [DeliveryController::class, 'action'])
                    ->whereNumber('delivery')
                    ->name('deliveries.action');
            });

        // ========================================
        // SUPPLIER AND ADMIN
        // ========================================

        Route::prefix('supplier')
            ->name('supplier.')
            ->middleware([
                'password.changed',
                'role:admin,supplier',
            ])
            ->group(function () {
                Route::get('/dashboard', [DashboardController::class, 'index'])
                    ->name('dashboard');

                Route::get('/dashboard/data', [DashboardController::class, 'data'])
                    ->name('dashboard.data');

                // Flower Management
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

                // Inventory Management
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

                // Batch Management
                Route::get('/batches', [BatchController::class, 'index'])
                    ->name('batches');

                Route::get('/batches/data', [BatchController::class, 'data'])
                    ->name('batches.data');

                Route::post('/batches/{batch}/update', [BatchController::class, 'update'])
                    ->whereNumber('batch')
                    ->name('batches.update');

                // Spoilage Prediction
                Route::view('/spoilage-prediction', 'supplier.spoilage-prediction')
                    ->name('spoilage-prediction');

                // Client Management
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

                // Order Management
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

                // Delivery Management
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

                // Sales and Payments
                Route::get('/sales-payments', [SalesPaymentController::class, 'index'])
                    ->name('sales-payments');

                Route::get('/sales-payments/data', [SalesPaymentController::class, 'data'])
                    ->name('sales-payments.data');

                Route::post('/sales-payments/prices/{flower}', [SalesPaymentController::class, 'price'])
                    ->whereNumber('flower')
                    ->name('sales-payments.price');

                Route::post('/sales-payments/payments', [SalesPaymentController::class, 'recordPayment'])
                    ->name('sales-payments.record');

                Route::post('/sales-payments/payments/{payment}/confirm', [SalesPaymentController::class, 'confirm'])
                    ->whereNumber('payment')
                    ->name('sales-payments.confirm');

                Route::post('/sales-payments/payments/{payment}/refund', [SalesPaymentController::class, 'refund'])
                    ->whereNumber('payment')
                    ->name('sales-payments.refund');

                // ========================================
                // ADMIN ONLY
                // ========================================

                Route::middleware('role:admin')->group(function () {
                    // Reports
                    Route::get('/reports', [ReportController::class, 'index'])
                        ->name('reports');

                    Route::get('/reports/data', [ReportController::class, 'data'])
                        ->name('reports.data');

                    // User Management
                    Route::get('/users', [UserController::class, 'index'])
                        ->name('users');

                    Route::get('/users/data', [UserController::class, 'data'])
                        ->name('users.data');

                    Route::get('/users/{user}', [UserController::class, 'show'])
                        ->whereNumber('user')
                        ->name('users.show');

                    Route::post('/users', [UserController::class, 'store'])
                        ->name('users.store');

                    Route::post('/users/{user}/update', [UserController::class, 'update'])
                        ->whereNumber('user')
                        ->name('users.update');

                    Route::post('/users/{user}/status', [UserController::class, 'status'])
                        ->whereNumber('user')
                        ->name('users.status');

                    Route::post('/users/{user}/reset-password', [UserController::class, 'resetPassword'])
                        ->whereNumber('user')
                        ->name('users.reset-password');

                    // Audit Trail
                    Route::get('/audit-trail', [AuditTrailController::class, 'index'])
                        ->name('audit-trail');

                    Route::get('/audit-trail/data', [AuditTrailController::class, 'data'])
                        ->name('audit-trail.data');
                });
            });
    });
});
```

## 9. `tests/Feature/DashboardBackendTest.php`

```php
<?php

namespace Tests\Feature;

use App\Models\AuditEvent;
use App\Models\Client;
use App\Models\Flower;
use App\Models\FlowerPrice;
use App\Models\InventoryBatch;
use App\Models\InventoryReservation;
use App\Models\Order;
use App\Models\OrderItem;
use App\Models\OrderPayment;
use App\Models\User;
use App\Services\DashboardService;
use App\Services\InventoryService;
use App\Services\OrderService;
use App\Services\SalesPaymentService;
use App\Support\AccountSession;
use App\Support\Money;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Date;
use Illuminate\Support\Facades\Schema;
use Illuminate\Support\Str;
use Symfony\Component\HttpKernel\Exception\HttpException;
use Tests\TestCase;

class DashboardBackendTest extends TestCase
{
    private User $admin;
    private User $supplier;
    private bool $isolated = false;
    private string $timezone;

    /** No RefreshDatabase: all schema changes target this named memory database. */
    protected function setUp(): void
    {
        parent::setUp();

        if (! extension_loaded('pdo_sqlite')) {
            $this->markTestSkipped('Dashboard tests require pdo_sqlite.');
        }

        $this->timezone = date_default_timezone_get();
        date_default_timezone_set('UTC');

        config([
            'database.connections.dashboard_backend_test' => [
                'driver' => 'sqlite',
                'database' => ':memory:',
                'prefix' => '',
                'foreign_key_constraints' => true,
            ],
            'database.default' => 'dashboard_backend_test',
            'session.driver' => 'array',
            'cache.default' => 'array',
            'app.timezone' => 'UTC',
            'hashing.bcrypt.rounds' => 4,
        ]);

        $this->isolated = true;
        DB::purge('dashboard_backend_test');
        $this->assertSame('sqlite', DB::connection()->getDriverName());
        $this->assertSame(':memory:', DB::connection()->getDatabaseName());

        Schema::create('users', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('email')->unique();
            $table->string('password');
            $table->string('role');
            $table->boolean('is_active')->default(true);
            $table->boolean('must_change_password')->default(false);
            $table->timestamp('email_verified_at')->nullable();
            $table->rememberToken();
            $table->timestamps();
        });

        Schema::create('flowers', function (Blueprint $table) {
            $table->id();
            $table->string('code', 30)->unique();
            $table->string('name', 150);
            $table->enum('product_type', ['Stem', 'Bouquet']);
            $table->string('category', 100);
            $table->unsignedInteger('minimum_stock');
            $table->text('description');
            $table->string('image_path')->nullable();
            $table->enum('status', ['Active', 'Inactive']);
            $table->timestamps();
        });

        foreach ([
            'create_clients_table',
            'create_inventory_tables',
            'create_order_tables',
            'create_delivery_tables',
            'create_finance_tables',
        ] as $name) {
            $this->migrateOne($name);
        }

        // Create initial actors before enabling audit storage.
        $this->admin = $this->user('admin', User::ROLE_ADMIN);
        $this->supplier = $this->user('supplier', User::ROLE_SUPPLIER);
        $this->migrateOne('create_audit_events_table');
    }

    protected function tearDown(): void
    {
        Date::setTestNow();

        if ($this->isolated) {
            DB::disconnect('dashboard_backend_test');
        }

        if (isset($this->timezone)) {
            date_default_timezone_set($this->timezone);
        }

        parent::tearDown();
    }

    public function test_empty_dashboard_returns_real_zeroes_and_does_not_write(): void
    {
        $before = AuditEvent::count();
        $data = app(DashboardService::class)->data($this->admin);
        foreach ($data['summary'] as $value) {
            $this->assertSame(0, $value);
        }
        $this->assertSame([], $data['chart']['items']);
        $this->assertSame([], $data['topProducts']);
        $this->assertSame([], $data['recent']);
        $this->assertSame('Asia/Manila', $data['timezone']);
        $this->assertStringEndsWith('+08:00', $data['generatedAt']);
        $this->assertSame($before, AuditEvent::count());
    }

    public function test_stock_units_reservations_and_active_low_stock_are_separate(): void
    {
        $order = $this->newOrder();
        $stem = $order->items()->firstOrFail()->flower;
        InventoryReservation::create([
            'flower_id' => $stem->id, 'reference' => 'manual-audit-fixture',
            'quantity' => 15, 'status' => 'Active',
        ]);
        $bouquet = Flower::create([
            'code' => 'BOUQUET-001', 'name' => 'Finished Bouquet',
            'product_type' => 'Bouquet', 'category' => 'Roses',
            'minimum_stock' => 2, 'description' => 'Fixture', 'status' => 'Active',
        ]);
        app(InventoryService::class)->stockIn([
            'request_key' => (string) Str::uuid(), 'flower_id' => $bouquet->id,
            'quantity' => 1, 'unit' => 'Bouquet', 'received' => '2026-11-01',
            'harvest' => '2026-11-01', 'expiry' => null, 'remarks' => 'Finished bouquet.',
        ], $this->admin);
        $inactive = Flower::create([
            'code' => 'INACTIVE-001', 'name' => 'Inactive Product',
            'product_type' => 'Stem', 'category' => 'Roses', 'minimum_stock' => 10,
            'description' => 'Fixture', 'status' => 'Inactive',
        ]);
        $data = app(DashboardService::class)->data($this->admin);
        $this->assertSame(20, $data['summary']['stems']);
        $this->assertSame(1, $data['summary']['bouquets']);
        $this->assertSame(2, $data['summary']['lowStockItems']);
        $row = collect($data['chart']['items'])->firstWhere('code', $stem->code);
        $this->assertSame(20, $row['onHand']);
        $this->assertSame(15, $row['reserved']);
        $this->assertSame(5, $row['available']);
        $data = app(DashboardService::class)->data($this->admin, 'Bouquet');
        $this->assertCount(1, $data['chart']['items']);
        $this->assertSame($bouquet->code, $data['chart']['items'][0]['code']);
    }

    public function test_negative_availability_is_not_clamped_and_spoiled_batches_are_excluded(): void
    {
        $order = $this->newOrder();
        $flowerId = $order->items()->firstOrFail()->flower_id;
        InventoryReservation::create([
            'flower_id' => $flowerId, 'reference' => 'overbooked-fixture',
            'quantity' => 25, 'status' => 'Active',
        ]);
        $data = app(DashboardService::class)->data($this->admin);
        $this->assertSame(-5, $data['chart']['items'][0]['available']);
        InventoryBatch::where('flower_id', $flowerId)->firstOrFail()
            ->forceFill(['spoiled' => true])->save();
        $data = app(DashboardService::class)->data($this->admin);
        $this->assertSame(0, $data['summary']['stems']);
        $this->assertSame(-25, $data['chart']['items'][0]['available']);
    }

    public function test_pending_card_counts_only_orders_awaiting_confirmation(): void
    {
        $this->actingAs($this->admin);
        $order = $this->newOrder();
        $this->assertSame(1, app(DashboardService::class)->data($this->admin)['summary']['pendingOrders']);
        $this->advance($order, 'Confirmed');
        $data = app(DashboardService::class)->data($this->admin);
        $this->assertSame(0, $data['summary']['pendingOrders']);
        $this->assertSame(20, $data['summary']['stems']);
        $this->assertSame(18, $data['chart']['items'][0]['available']);
        $this->assertNotEmpty($data['recent']);
    }

    public function test_net_sales_use_completed_sales_and_refunds_while_rankings_keep_snapshots(): void
    {
        $this->actingAs($this->admin);
        $order = $this->newOrder();
        foreach (['Confirmed', 'Processing', 'Ready for Pickup', 'Completed'] as $status) {
            $this->advance($order, $status);
        }
        $payment = $this->pay($order->fresh());
        $sales = app(SalesPaymentService::class);
        foreach (['2.00', '3.00'] as $amount) {
            $paymentRow = collect($sales->data($this->admin)['payments'])->firstWhere('dbId', $payment->id);
            $sales->refund($payment, [
                'request_key' => (string) Str::uuid(), 'version' => $paymentRow['version'],
                'amount' => $amount, 'reason' => 'Verified fixture refund.',
            ], $this->admin);
        }
        $price = FlowerPrice::firstOrFail();
        $price->price_cents = 9900;
        $price->save();
        $data = app(DashboardService::class)->data($this->admin);
        $this->assertSame(2000, $data['summary']['grossSalesCents']);
        $this->assertSame(500, $data['summary']['refundCents']);
        $this->assertSame(1500, $data['summary']['netSalesCents']);
        $this->assertSame(18, $data['summary']['stems']);
        $this->assertSame(2000, $data['topProducts'][0]['grossCents']);
        $this->assertSame(2, $data['topProducts'][0]['quantity']);
    }

    public function test_payment_for_unfulfilled_order_does_not_create_sales_or_deduct_stock(): void
    {
        $order = $this->newOrder();
        $this->pay($order);
        $data = app(DashboardService::class)->data($this->admin);
        $this->assertSame(0, $data['summary']['netSalesCents']);
        $this->assertSame(20, $data['summary']['stems']);
        $this->assertSame([], $data['topProducts']);
    }

    public function test_dashboard_routes_render_for_admin_and_supplier_without_admin_link_leaks(): void
    {
        $this->signIn($this->admin);
        $this->get('/supplier/dashboard')->assertOk()
            ->assertSee('On-Hand Inventory')->assertDontSee('sample dashboard figures');
        $this->getJson('/supplier/dashboard/data')->assertOk()
            ->assertJsonPath('summary.stems', 0);
        $this->signIn($this->supplier);
        $this->get('/supplier/dashboard')->assertOk()
            ->assertDontSee('href="'.route('supplier.audit-trail').'"', false)
            ->assertDontSee('href="'.route('supplier.reports').'"', false)
            ->assertDontSee('href="'.route('supplier.users').'"', false);
        $this->getJson('/supplier/dashboard/data')->assertOk();
        $this->getJson('/supplier/dashboard/data?unit=Invalid')->assertUnprocessable();
        $delivery = $this->user('delivery', User::ROLE_DELIVERY);
        $this->signIn($delivery);
        $this->getJson('/supplier/dashboard/data')->assertForbidden();
    }

    public function test_supplier_summary_never_reads_or_exposes_security_audit_details(): void
    {
        $order = $this->newOrder();
        app(\App\Services\AuditService::class)->record(
            'Authentication', 'Private Security Marker',
            description: 'Never expose this security detail on the shared Dashboard.'
        );
        $before = AuditEvent::count();
        $data = app(DashboardService::class)->data($this->supplier);
        $json = json_encode($data, JSON_THROW_ON_ERROR);
        $this->assertStringNotContainsString('Private Security Marker', $json);
        $this->assertStringNotContainsString('Never expose this security detail', $json);
        $this->assertStringNotContainsString($this->admin->password, $json);
        $this->assertStringNotContainsString('client_email', $json);
        $this->assertSame($before, AuditEvent::count());
        $this->assertNotEmpty($data['recent']);
    }

    public function test_inactive_and_password_change_required_accounts_cannot_read_dashboard_service(): void
    {
        foreach (['is_active' => false, 'must_change_password' => true] as $field => $value) {
            $user = $this->supplier->fresh();
            $user->{$field} = $value;
            try {
                app(DashboardService::class)->data($user);
                $this->fail('Restricted account should not read Dashboard data.');
            } catch (HttpException $exception) {
                $this->assertSame(403, $exception->getStatusCode());
            }
        }
    }

    private function pay(Order $order): OrderPayment
    {
        $sales = app(SalesPaymentService::class);
        $row = collect($sales->data($this->admin)['orders'])->firstWhere('dbId', $order->id);
        $sales->recordPayment([
            'request_key' => (string) Str::uuid(), 'order_id' => $order->id,
            'version' => $row['version'], 'method' => 'Cash',
            'amount' => Money::format((int) $order->total_cents),
        ], $this->admin);
        $payment = OrderPayment::where('order_id', $order->id)->firstOrFail();
        $row = collect($sales->data($this->admin)['payments'])->firstWhere('dbId', $payment->id);
        $sales->confirm($payment, [
            'request_key' => (string) Str::uuid(), 'version' => $row['version'],
            'reason' => 'Actual fixture payment verified.',
        ], $this->admin);
        return $payment->fresh();
    }

    private function migrateOne(string $suffix): void
    {
        $files = glob(database_path('migrations/*_'.$suffix.'.php')) ?: [];
        $this->assertCount(1, $files, 'Expected one '.$suffix.' migration.');
        (require $files[0])->up();
    }

    private function user(string $label, string $role): User
    {
        return User::forceCreate([
            'name' => 'Audit '.$label,
            'email' => $label.'@audit.example.test',
            'password' => 'Audit-test-password-2026!',
            'role' => $role,
            'is_active' => true,
            'must_change_password' => false,
        ]);
    }

    private function flower(): Flower
    {
        return Flower::create([
            'code' => 'FLW-AUDIT-001',
            'name' => 'Audit Rose',
            'product_type' => 'Stem',
            'category' => 'Roses',
            'minimum_stock' => 10,
            'description' => 'Isolated audit fixture.',
            'image_path' => null,
            'status' => 'Active',
        ]);
    }

    private function signIn(User $user): void
    {
        $this->actingAs($user)->withSession([
            AccountSession::KEY => AccountSession::signature($user),
        ]);
    }

    private function newOrder(): Order
    {
        $user = $this->user('client', User::ROLE_CLIENT);
        $client = Client::create([
            'code' => 'CLI-AUDIT-001',
            'user_id' => $user->id,
            'phone' => '09170000000',
            'address' => 'Audit test address',
        ]);

        $flower = $this->flower();
        FlowerPrice::create(['flower_id' => $flower->id, 'price_cents' => 1000]);

        app(InventoryService::class)->stockIn([
            'request_key' => (string) Str::uuid(),
            'flower_id' => $flower->id,
            'quantity' => 20,
            'unit' => 'Stem',
            'received' => '2026-11-01',
            'harvest' => '2026-11-01',
            'expiry' => null,
            'remarks' => 'Isolated audit receipt.',
        ], $this->admin);

        return app(OrderService::class)->store([
            'request_key' => (string) Str::uuid(),
            'client_id' => $client->id,
            'method' => 'Customer Pickup',
            'address' => null,
            'notes' => null,
            'items' => [['flower_id' => $flower->id, 'quantity' => 2]],
        ], $this->admin);
    }

    private function advance(Order $order, string $status): void
    {
        $orders = app(OrderService::class);

        $orders->action($order, [
            'status' => $status,
            'version' => $orders->row($order->fresh(), $this->admin)['version'],
            'reason' => 'Isolated cancellation test.',
            'collected' => $status === 'Completed',
        ], $this->admin);
    }
}
```

