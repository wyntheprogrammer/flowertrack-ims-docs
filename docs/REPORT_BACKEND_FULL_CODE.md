# FlowerTrack — Reports backend: full replacement code

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

## `app/Http/Controllers/Supplier/ReportController.php`

```php
<?php

namespace App\Http\Controllers\Supplier;

use App\Http\Controllers\Controller;
use App\Services\ReportService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\View\View;

class ReportController extends Controller
{
    public function __construct(private readonly ReportService $reports)
    {
    }

    public function index(): View
    {
        return view('supplier.reports');
    }

    public function data(Request $request): JsonResponse
    {
        return response()->json($this->reports->data($request->user()), 200, [
            'Cache-Control' => 'private, no-store',
            'X-Content-Type-Options' => 'nosniff',
        ]);
    }
}
```

## `app/Services/ReportService.php`

```php
<?php

namespace App\Services;

use App\Models\Client;
use App\Models\Delivery;
use App\Models\Flower;
use App\Models\InventoryBatch;
use App\Models\InventoryMovement;
use App\Models\InventoryReservation;
use App\Models\Order;
use App\Models\OrderPayment;
use App\Models\PaymentRefund;
use App\Models\SalesRecord;
use App\Models\User;
use Illuminate\Support\Facades\DB;
use Illuminate\Validation\ValidationException;

class ReportService
{
    private const MAX_SAFE_INTEGER = 9007199254740991;

    /** Read one canonical business snapshot; reports never write operational records. */
    public function data(User $actor): array
    {
        abort_unless($actor->role === User::ROLE_ADMIN
            && $actor->is_active && !$actor->must_change_password, 403);

        return DB::transaction(function () {
            $orders = Order::orderBy('id')->get();
            $ordersById = $orders->keyBy('id');
            $payments = OrderPayment::orderBy('id')->get();
            $paymentsByOrder = $payments->keyBy('order_id');
            $refunds = [];
            foreach (PaymentRefund::orderBy('id')->get() as $refund) {
                $id = (int) $refund->payment_id;
                $refunds[$id] = $this->add($refunds[$id] ?? 0, $this->nonnegative($refund->amount_cents));
            }

            $clientOrders = [];
            foreach ($orders as $order) {
                $id = (int) $order->client_id;
                $clientOrders[$id] = $this->add($clientOrders[$id] ?? 0, 1);
            }
            $clientPurchases = [];
            $sales = [];
            foreach (SalesRecord::orderBy('id')->get() as $sale) {
                $payment = $paymentsByOrder->get($sale->order_id);
                $refunded = $payment ? ($refunds[$payment->id] ?? 0) : 0;
                $gross = $this->nonnegative($sale->total_cents);
                if ($refunded > $gross) $this->invalid('Refund exceeds the recorded sale amount. Review financial records.');
                $net = $gross - $refunded;
                $clientId = (int) $sale->client_id;
                $clientPurchases[$clientId] = $this->add($clientPurchases[$clientId] ?? 0, $net);
                $sales[] = [
                    'id' => $sale->code,
                    'order' => $ordersById->get($sale->order_id)?->code ?? 'Unavailable',
                    'client' => $sale->client_name,
                    'date' => $this->day($sale->completed_at),
                    'gross' => $gross, 'refund' => $refunded, 'net' => $net,
                ];
            }

            $paymentRows = [];
            foreach ($payments as $payment) {
                $amount = $this->nonnegative($payment->amount_cents);
                $refunded = $refunds[$payment->id] ?? 0;
                $verified = in_array($payment->status, ['Confirmed', 'Refunded'], true) ? $amount : 0;
                if ($refunded > $verified) $this->invalid('Refund exceeds verified collection. Review financial records.');
                $order = $ordersById->get($payment->order_id);
                $paymentRows[] = [
                    'id' => $payment->code, 'order' => $order?->code ?? 'Unavailable',
                    'client' => $order?->client_name ?? 'Unavailable',
                    'date' => $this->day($payment->created_at), 'method' => $payment->method,
                    'amount' => $amount, 'status' => $payment->status,
                    'refund' => $refunded, 'verified' => $verified, 'net' => $verified - $refunded,
                ];
            }

            $physical = [];
            foreach (InventoryBatch::where('spoiled', false)->orderBy('id')->get() as $batch) {
                $id = (int) $batch->flower_id;
                $physical[$id] = $this->add($physical[$id] ?? 0, $this->nonnegative($batch->remaining_quantity));
            }
            $holds = [];
            foreach (InventoryReservation::where('status', 'Active')->orderBy('id')->get() as $reservation) {
                $id = (int) $reservation->flower_id;
                $holds[$id] = $this->add($holds[$id] ?? 0, $this->nonnegative($reservation->quantity));
            }
            $inventory = [];
            foreach (Flower::orderBy('id')->get() as $flower) {
                $onHand = $physical[$flower->id] ?? 0;
                $reserved = $holds[$flower->id] ?? 0;
                $available = $onHand - $reserved;
                $minimum = $this->nonnegative($flower->minimum_stock);
                $inventory[] = [
                    'code' => $flower->code, 'item' => $flower->name, 'unit' => $flower->product_type,
                    'onHand' => $onHand, 'reserved' => $reserved, 'available' => $available,
                    'minimum' => $minimum,
                    'status' => $available <= 0 ? 'Out of Stock' : ($available < $minimum ? 'Low Stock' : 'Normal'),
                ];
            }

            $orderRows = $orders->map(fn (Order $order) => [
                'id' => $order->code, 'client' => $order->client_name,
                'date' => $this->day($order->created_at), 'method' => $order->method,
                'status' => $order->status, 'total' => $this->nonnegative($order->total_cents),
            ])->all();

            $deliveryRows = [];
            foreach (Delivery::with('personnel:id,name')->orderBy('id')->get() as $delivery) {
                $order = $ordersById->get($delivery->order_id);
                if (!$order || $order->method !== 'Delivery') continue;
                $deliveryRows[] = [
                    'id' => $delivery->code, 'order' => $order->code, 'client' => $order->client_name,
                    'date' => $delivery->scheduled_at ? $this->day($delivery->scheduled_at) : 'Not scheduled',
                    'person' => $delivery->personnel?->name ?? 'Unassigned', 'status' => $delivery->status,
                ];
            }

            $clients = [];
            foreach (Client::with('user:id,name,email,role,is_active')->orderBy('id')->get() as $client) {
                if (!$client->user || $client->user->role !== User::ROLE_CLIENT) continue;
                $clients[] = [
                    'id' => $client->code, 'name' => $client->user->name, 'email' => $client->user->email,
                    'registered' => $this->day($client->created_at),
                    'status' => $client->user->is_active ? 'Active' : 'Inactive',
                    'orders' => $clientOrders[$client->id] ?? 0,
                    'purchases' => $clientPurchases[$client->id] ?? 0,
                ];
            }

            $spoilage = [];
            foreach (InventoryMovement::with(['flower:id,name', 'batch:id,code,receiving_unit'])
                ->where('type', 'Stock-Out')->where('quantity', '<', 0)->orderBy('id')->get() as $movement) {
                if (($movement->metadata['reason'] ?? null) !== 'Spoilage') continue;
                $quantity = -(int) $movement->quantity;
                $this->nonnegative($quantity);
                $spoilage[] = [
                    'id' => 'MOV-'.$movement->id, 'date' => $this->day($movement->created_at),
                    'item' => $movement->flower?->name ?? 'Unavailable',
                    'batch' => $movement->batch?->code ?? 'Unavailable',
                    // Receiving-unit snapshots preserve the historical Stem/Bouquet distinction.
                    'unit' => $movement->batch?->receiving_unit === 'Bouquet' ? 'Bouquet' : 'Stem',
                    'quantity' => $quantity, 'reason' => $movement->remarks ?? '',
                ];
            }

            return [
                'data' => ['sales' => $sales, 'payments' => $paymentRows, 'inventory' => $inventory,
                    'orders' => $orderRows, 'delivery' => $deliveryRows, 'clients' => $clients, 'spoilage' => $spoilage],
                'generatedAt' => now()->setTimezone('Asia/Manila')->toIso8601String(),
                'timezone' => 'Asia/Manila',
            ];
        });
    }

    /** Preserve exact integer sums within the browser's safe integer range. */
    private function add(int $left, int $right): int
    {
        if ($left < 0 || $right < 0 || $left > self::MAX_SAFE_INTEGER - $right) {
            $this->invalid('Report totals exceed the supported exact display range.');
        }
        return $left + $right;
    }

    private function nonnegative($value): int
    {
        if (!is_numeric($value) || (string) (int) $value !== (string) $value
            || (int) $value < 0 || (int) $value > self::MAX_SAFE_INTEGER) {
            $this->invalid('A report contains an invalid or unsupported numeric value.');
        }
        return (int) $value;
    }

    private function day($date): string
    {
        return $date->copy()->setTimezone('Asia/Manila')->format('Y-m-d');
    }

    private function invalid(string $message): never
    {
        throw ValidationException::withMessages(['report' => $message]);
    }
}
```

## `resources/js/supplier/reports.js`

```javascript
import DataTable from 'datatables.net-bs5';
import { jsPDF } from 'jspdf';
import autoTable from 'jspdf-autotable';
import ExcelJS from 'exceljs';

const escapeHTML = (value) => String(value ?? '').replace(/[&<>"']/g, (c) => ({
    '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#039;',
})[c]);
// Format integer cents without dividing into an imprecise floating currency value.
function money(value) {
    if (!Number.isSafeInteger(value)) throw new Error('Financial total exceeds safe integer precision.');
    const absolute = Math.abs(value), whole = BigInt(Math.floor(absolute / 100));
    const signed = value < 0 ? (whole === 0n ? -0 : -whole) : whole;
    return new Intl.NumberFormat('en-PH', {style:'currency', currency:'PHP'}).formatToParts(signed)
        .map((part) => part.type === 'fraction' ? String(absolute % 100).padStart(2,'0') : part.value).join('');
}
// Reject aggregate overflow rather than silently rounding totals or quantities.
function checkedAdd(total, value) {
    if (!Number.isSafeInteger(total) || !Number.isSafeInteger(value) || !Number.isSafeInteger(total + value)) throw new Error('Report total exceeds safe integer precision.');
    return total + value;
}
const sum = (rows, key) => rows.reduce((total, row) => checkedAdd(total, row[key]), 0);
const textColumn = (key, title) => ({ key, title });
const numericColumn = (key, title) => ({ key, title, number: true });
const moneyColumn = (key, title) => ({ key, title, money: true });

const definitions = {
    sales: { title: 'Sales Report', note: 'Completed-order sales less recorded refunds. Refunds are attributed to the original completion date.', columns: [textColumn('id','Sale ID'), textColumn('order','Order ID'), textColumn('client','Client'), textColumn('date','Completed'), moneyColumn('gross','Gross Sales'), moneyColumn('refund','Refunds'), moneyColumn('net','Net Sales')] },
    payments: { title: 'Payments Report', note: 'Original amounts are retained after refunds. Pending payments are excluded from confirmed collections; refunds-to-date and net collections are attributed to the original recorded receipt date. This is not a cash-period statement.', columns: [textColumn('id','Payment ID'),textColumn('order','Order ID'),textColumn('client','Client'),textColumn('date','Recorded'),textColumn('method','Method'),moneyColumn('amount','Amount'),textColumn('status','Status'),moneyColumn('refund','Refunds'),moneyColumn('net','Net Collection')] },
    inventory: { title: 'Inventory Report', note: 'Physical stock and reservations are distinct. Stem and Bouquet quantities must never be combined into one stock total.', columns: [textColumn('code','Item Code'),textColumn('item','Item'),textColumn('unit','Unit'),numericColumn('onHand','On-Hand'),numericColumn('reserved','Reserved'),numericColumn('available','Available'),numericColumn('minimum','Minimum'),textColumn('status','Status')] },
    orders: { title: 'Orders Report', note: 'Order status and payment status are independent. Cancelled order amounts are not completed sales.', columns: [textColumn('id','Order ID'),textColumn('client','Client'),textColumn('date','Order Date'),textColumn('method','Fulfillment'),textColumn('status','Status'),moneyColumn('total','Order Total')] },
    delivery: { title: 'Delivery Report', note: 'Delivery records only; Pickup orders are excluded. This summary counts current delivery statuses, not historical attempt performance.', columns: [textColumn('id','Delivery ID'),textColumn('order','Order ID'),textColumn('client','Client'),textColumn('date','Scheduled Date'),textColumn('person','Personnel'),textColumn('status','Status')] },
    clients: { title: 'Clients Report', note: 'Client activity uses registered client identities. Net completed purchases include recorded refunds.', columns: [textColumn('id','Client ID'),textColumn('name','Client'),textColumn('email','Email'),textColumn('registered','Registered'),textColumn('status','Status'),numericColumn('orders','Orders'),moneyColumn('purchases','Net Completed Purchases')] },
    spoilage: { title: 'Spoilage Report', note: 'Recorded Stock-Out movements explicitly marked Spoilage only; prediction risk and expiry are not counted as spoilage. Quantities remain separate by unit.', columns: [textColumn('id','Event ID'),textColumn('date','Recorded'),textColumn('item','Item'),textColumn('batch','Batch'),textColumn('unit','Unit'),numericColumn('quantity','Spoiled Quantity'),textColumn('reason','Remarks')] },
};

// Validate the entire response before replacing the previous canonical snapshot.
function validateSnapshot(snapshot) {
    if (!snapshot || snapshot.timezone !== 'Asia/Manila' || typeof snapshot.generatedAt !== 'string' || !Number.isFinite(Date.parse(snapshot.generatedAt))) throw new Error('Invalid report snapshot timestamp.');
    for (const [key, definition] of Object.entries(definitions)) {
        const rows = snapshot.data?.[key];
        if (!Array.isArray(rows)) throw new Error(`Missing ${key} report records.`);
        for (const row of rows) {
            if (!row || typeof row !== 'object') throw new Error('Invalid report record.');
            for (const column of definition.columns) {
                const value = row[column.key];
                if (column.money || column.number) {
                    if (!Number.isSafeInteger(value) || (column.money && Math.abs(value) > 999999999999)) throw new Error(`Invalid ${key} ${column.title}.`);
                } else if (typeof value !== 'string') throw new Error(`Invalid ${key} ${column.title}.`);
            }
            if (key === 'sales' && checkedAdd(row.net,row.refund) !== row.gross) throw new Error('Sales totals do not reconcile.');
            if (key === 'payments') {
                if (!['Pending','Confirmed','Refunded'].includes(row.status) || !Number.isSafeInteger(row.verified) || row.amount < 0 || row.refund < 0 || row.refund > row.amount || row.verified !== (row.status === 'Pending' ? 0 : row.amount) || row.net !== row.verified - row.refund) throw new Error('Payment totals do not reconcile.');
            }
            if (['inventory','spoilage'].includes(key) && !['Stem','Bouquet'].includes(row.unit)) throw new Error('Invalid stock unit.');
            if (key === 'spoilage' && row.quantity <= 0) throw new Error('Invalid spoilage quantity.');
        }
        // Validate every numeric aggregate, including chart groups, before rendering.
        for (const column of definition.columns.filter((column) => column.money || column.number)) {
            sum(rows,column.key);
            // Any filtered subset or chart group must also remain exactly representable.
            sum(rows.filter((row) => row[column.key] > 0),column.key);
            sum(rows.filter((row) => row[column.key] < 0),column.key);
        }
        metricsFor(key, rows);
    }
    return structuredClone(snapshot);
}

function metricsFor(key, rows) {
    const count = (status) => rows.filter((row) => row.status === status).length;
    const byUnit = (unit, field) => sum(rows.filter((row) => row.unit === unit), field);
    if (key === 'sales') return [['Sales',rows.length],['Gross Sales',money(sum(rows,'gross'))],['Refunds',money(sum(rows,'refund'))],['Net Sales',money(sum(rows,'net'))]];
    if (key === 'payments') return [['Confirmed Collections',money(sum(rows,'verified'))],['Pending Verification',money(sum(rows.filter((row) => row.status === 'Pending'),'amount'))],['Refunds',money(sum(rows,'refund'))],['Net Collections',money(sum(rows,'net'))]];
    if (key === 'inventory') return [['Stem On-Hand',byUnit('Stem','onHand')],['Bouquet On-Hand',byUnit('Bouquet','onHand')],['Stem Reserved',byUnit('Stem','reserved')],['Bouquet Reserved',byUnit('Bouquet','reserved')]];
    if (key === 'orders') return [['Orders',rows.length],['Completed',count('Completed')],['Cancelled',count('Cancelled')],['Ongoing',rows.length-count('Completed')-count('Cancelled')]];
    if (key === 'delivery') return [['Deliveries',rows.length],['Delivered',count('Delivered')],['Failed',count('Failed')],['Awaiting Completion',rows.length-count('Delivered')-count('Cancelled')]];
    if (key === 'clients') return [['Clients',rows.length],['Active',count('Active')],['Inactive',count('Inactive')],['Net Completed Purchases',money(sum(rows,'purchases'))]];
    return [['Recorded Events',rows.length],['Spoiled Stems',byUnit('Stem','quantity')],['Spoiled Bouquets',byUnit('Bouquet','quantity')],['Affected Batches',new Set(rows.map((row) => row.batch)).size]];
}

class ReportsUI {
    constructor() {
        this.key = 'sales';
        this.table = null;
        this.busy = false;
        this.loaded = false;
        this.snapshot = {data:Object.fromEntries(Object.keys(definitions).map((key) => [key,[]]))};
        this.base = document.getElementById('reportPage').dataset.url.replace(/\/$/,'');
        this.render();
        document.getElementById('reportRefresh').addEventListener('click',() => this.load());
        this.load();
        document.getElementById('reportType').addEventListener('change',(event) => { this.key=event.target.value; this.render(); });
        document.getElementById('reportSearch').addEventListener('input',(event) => this.table.search(event.target.value).draw());
        document.getElementById('reportLength').addEventListener('change',(event) => this.table.page.len(Number(event.target.value)).draw());
        document.querySelectorAll('[data-export]').forEach((button) => button.addEventListener('click',() => this.export(button.dataset.export)));
    }

    // Refresh is explicit; exports never fetch or mix snapshots.
    async load() {
        if (this.busy) return;
        this.busy = true;
        this.setControls();
        document.getElementById('reportStatus').textContent = 'Loading report records…';
        try {
            const response = await fetch(`${this.base}/data`, {headers:{Accept:'application/json'},credentials:'same-origin'});
            const payload = await response.json();
            if (!response.ok) throw new Error(payload.message || `Unable to load reports (${response.status}).`);
            this.snapshot = validateSnapshot(payload);
            this.loaded = true;
            this.render();
            document.getElementById('reportSnapshot').textContent = `Snapshot: ${this.snapshot.generatedAt} · Asia/Manila`;
            document.getElementById('reportStatus').textContent = 'Report snapshot refreshed.';
        } catch (error) {
            document.getElementById('reportStatus').textContent = `Refresh failed: ${error.message}${this.loaded ? ' Previous snapshot retained.' : ''}`;
        } finally { this.busy = false; this.setControls(); }
    }

    // Disable conflicting operations while a snapshot or export is being prepared.
    setControls() {
        document.getElementById('reportRefresh').disabled = this.busy;
        document.querySelectorAll('[data-export]').forEach((button) => button.disabled = this.busy || !this.loaded);
    }

    // Rebuild the native sortable table for the chosen report type.
    render() {
        if (this.table) this.table.destroy();
        const definition=definitions[this.key];
        document.getElementById('reportDefinition').textContent=definition.note;
        document.getElementById('reportSearch').value='';
        document.getElementById('reportStatus').textContent='';
        const host=document.getElementById('reportTableHost');
        host.innerHTML='<table id="reportTable"><thead><tr>'+definition.columns.map((column) => `<th scope="col">${escapeHTML(column.title)}</th>`).join('')+'</tr></thead><tbody></tbody></table>';
        this.table=new DataTable('#reportTable',{
            data:this.snapshot.data[this.key],autoWidth:false,pageLength:Number(document.getElementById('reportLength').value),order:[[0,'asc']],
            layout:{topStart:null,topEnd:null,bottomStart:'info',bottomEnd:'paging'},
            language:{emptyTable:'No report records available.',zeroRecords:'No matching records found.',info:'Showing _START_ to _END_ of _TOTAL_ entries',infoEmpty:'Showing 0 to 0 of 0 entries',infoFiltered:'(filtered from _MAX_ total entries)',paginate:{first:'«',previous:'‹',next:'›',last:'»'}},
            columns:definition.columns.map((column) => ({data:column.key,className:column.number||column.money?'report-number':'',render:(value,type) => type==='display'?escapeHTML(column.money?money(value):value):value})),
        });
        const table=document.getElementById('reportTable');const wrapper=document.createElement('div');
        wrapper.className='report-table-scroll';wrapper.tabIndex=0;wrapper.setAttribute('role','region');wrapper.setAttribute('aria-label',`${definition.title}. Scroll horizontally for all columns.`);table.before(wrapper);wrapper.append(table);
        this.table.on('draw',() => this.updateSummary());
        this.updateSummary();
    }

    // Export and summaries use every matching row, regardless of the visible page.
    selectedRows() { return this.table.rows({search:'applied',order:'applied'}).data().toArray(); }

    // Keep summaries and chart synchronized with the current table search.
    updateSummary() {
        const rows=this.selectedRows();
        document.getElementById('reportSummary').innerHTML=metricsFor(this.key,rows).map(([label,value]) => `<div><span>${escapeHTML(label)}</span><strong>${escapeHTML(value)}</strong></div>`).join('');
        const definition=definitions[this.key];const groups=new Map();
        rows.forEach((row) => {
            let label;let value;
            if(this.key==='sales'){label=row.date;value=row.net;}
            else if(this.key==='inventory'){label=row.status;value=1;}
            else if(this.key==='spoilage'){label=`${row.unit} events`;value=1;}
            else {label=row.status;value=1;}
            groups.set(label,checkedAdd(groups.get(label)||0,value));
        });
        const values=[...groups].sort((a,b) => a[0].localeCompare(b[0]));const maximum=Math.max(1,...values.map(([,value])=>value));
        document.getElementById('reportChart').innerHTML=`<h2>${this.key==='sales'?'Net Sales by Completion Date':'Record Count by Status / Type'}</h2>`+(values.length?values.map(([label,value])=>`<div class="report-bar"><div class="report-bar-label"><span>${escapeHTML(label)}</span><strong>${escapeHTML(this.key==='sales'?money(value):value)}</strong></div><div class="report-track" aria-hidden="true"><div class="report-fill" style="width:${Math.max(0,value/maximum*100)}%"></div></div></div>`).join(''):'<p class="report-definition">No matching records.</p>');
    }

    // Capture the loaded snapshot timestamp and all matching rows before async export.
    async export(kind) {
        if(this.busy || !this.loaded)return;
        this.busy=true;document.querySelectorAll('[data-export]').forEach((button)=>button.disabled=true);
        const definition=definitions[this.key];const rows=this.selectedRows();
        const timestamp=this.snapshot.generatedAt;const filename=`flowertrack-${this.key}-${timestamp.slice(0,10)}`;
        const columns=definition.columns;const headings=columns.map((column)=>column.title);
        const cells=rows.map((row)=>columns.map((column)=>column.money?money(row[column.key]):String(row[column.key])));
        try {
            if(kind==='pdf'){
                const pdf=new jsPDF({orientation:'landscape'});
                pdf.setFontSize(16);pdf.text(`FlowerTrack IMS - ${definition.title}`,14,16);
                pdf.setFontSize(9);pdf.text('Search results across all pages · Asia/Manila',14,23);
                pdf.text(`Generated: ${timestamp}`,14,29);
                // Built-in PDF font lacks the peso symbol; label amounts as PHP.
                autoTable(pdf,{startY:35,head:[headings],body:cells.map((row)=>row.map((value)=>value.replaceAll('₱','PHP '))),styles:{fontSize:8,overflow:'linebreak'},headStyles:{fillColor:[23,107,58]},margin:{left:14,right:14}});
                pdf.save(`${filename}.pdf`);
            } else if(kind==='excel'){
                const workbook=new ExcelJS.Workbook();workbook.creator='FlowerTrack IMS';
                const sheet=workbook.addWorksheet('Report');
                sheet.addRow([`FlowerTrack IMS - ${definition.title}`]);sheet.addRow(['Search results across all pages · Asia/Manila']);sheet.addRow([`Generated: ${timestamp}`]);sheet.addRow([definition.note]);sheet.addRow([]);sheet.addRow(headings);
                rows.forEach((row)=>sheet.addRow(columns.map((column)=>{
                    const cents=row[column.key];
                    const value=column.money?(Math.abs(cents)<=999999999999?cents/100:money(cents).replaceAll('₱','PHP ')):cents;
                    // Text is written as a string, never as an Excel formula object.
                    return typeof value==='number'?value:String(value);
                })));
                columns.forEach((column,index)=>{sheet.getColumn(index+1).width=column.money?18:25;if(column.money)sheet.getColumn(index+1).numFmt='"PHP "#,##0.00';});
                sheet.getRow(6).font={bold:true,color:{argb:'FFFFFFFF'}};sheet.getRow(6).fill={type:'pattern',pattern:'solid',fgColor:{argb:'FF176B3A'}};
                sheet.views=[{state:'frozen',ySplit:6}];sheet.autoFilter={from:{row:6,column:1},to:{row:6,column:columns.length}};
                const bytes=await workbook.xlsx.writeBuffer();this.download(new Blob([bytes],{type:'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'}),`${filename}.xlsx`);
            } else if(kind==='print'){
                const frame=document.createElement('iframe');frame.style.position='fixed';frame.style.width='0';frame.style.height='0';frame.style.border='0';
                const html=`<!doctype html><html><head><title>${escapeHTML(definition.title)}</title><style>@page{size:A4 landscape;margin:12mm}body{font-family:Arial,sans-serif;color:#344054}h1{font-size:20px}p{font-size:11px}table{width:100%;border-collapse:collapse;font-size:10px}th,td{padding:8px;border:1px solid #ddd;text-align:left}th{background:#eee7f3}thead{display:table-header-group}tr{break-inside:avoid}</style></head><body><h1>FlowerTrack IMS — ${escapeHTML(definition.title)}</h1><p>All matching rows · Asia/Manila · ${escapeHTML(timestamp)}</p><p>${escapeHTML(definition.note)}</p><table><thead><tr>${headings.map((value)=>`<th>${escapeHTML(value)}</th>`).join('')}</tr></thead><tbody>${cells.map((row)=>`<tr>${row.map((value)=>`<td>${escapeHTML(value)}</td>`).join('')}</tr>`).join('')}</tbody></table></body></html>`;
                frame.onload=()=>{frame.contentWindow.addEventListener('afterprint',()=>frame.remove(),{once:true});frame.contentWindow.focus();frame.contentWindow.print();};frame.srcdoc=html;document.body.append(frame);
            }
            document.getElementById('reportStatus').textContent=`${definition.title}: ${rows.length} matching records prepared for ${kind}.`;
        }catch(error){document.getElementById('reportStatus').textContent=`Export failed: ${error.message}`;}
        finally{this.busy=false;document.querySelectorAll('[data-export]').forEach((button)=>button.disabled=false);}
    }

    // Release the download URL after the browser has consumed it.
    download(blob,name){const url=URL.createObjectURL(blob);const link=document.createElement('a');link.href=url;link.download=name;document.body.append(link);link.click();link.remove();setTimeout(()=>URL.revokeObjectURL(url),60000);}
}

// Initialize once after the page shell exists.
function initializeReports(){if(document.getElementById('reportPage')&&!DataTable.isDataTable('#reportTable'))new ReportsUI();}
if(document.readyState==='loading')document.addEventListener('DOMContentLoaded',initializeReports,{once:true});else initializeReports();
```

## `resources/views/supplier/reports.blade.php`

```blade
@extends('layouts.supplier')
@section('title', 'Reports')
@section('content')
<div class="report-page" id="reportPage" data-url="{{ route('supplier.reports') }}">
    <div class="report-card">
        <div class="report-header">
            <div><h1>Reports</h1><p>Review business activity and export report records.</p></div>
            <div class="report-export-actions">
                <button type="button" id="reportRefresh"><i class="bi bi-arrow-clockwise" aria-hidden="true"></i> Refresh</button>
                <button type="button" data-export="pdf"><i class="bi bi-file-earmark-pdf" aria-hidden="true"></i> PDF</button>
                <button type="button" data-export="excel"><i class="bi bi-file-earmark-excel" aria-hidden="true"></i> Excel</button>
                <button type="button" data-export="print"><i class="bi bi-printer" aria-hidden="true"></i> Print</button>
            </div>
        </div>
        <p class="report-notice" id="reportSnapshot" aria-live="polite">Loading canonical report snapshot…</p>
        <div class="report-selector">
            <label for="reportType">Report Type</label>
            <select id="reportType">
                <option value="sales">Sales</option><option value="payments">Payments</option>
                <option value="inventory">Inventory</option><option value="orders">Orders</option>
                <option value="delivery">Delivery</option><option value="clients">Clients</option>
                <option value="spoilage">Spoilage</option>
            </select>
        </div>
        <div class="report-summary" id="reportSummary"></div>
        <p class="report-definition" id="reportDefinition"></p>
        <div class="report-chart" id="reportChart"></div>
        <div class="report-toolbar">
            <div class="report-length"><label for="reportLength">Show</label><select id="reportLength"><option>5</option><option>10</option><option>25</option><option>50</option></select><span>entries</span></div>
            <div class="report-search"><label class="visually-hidden" for="reportSearch">Search report</label><i class="bi bi-search" aria-hidden="true"></i><input id="reportSearch" type="search" placeholder="Search report records..." autocomplete="off"></div>
        </div>
        <div class="report-table-container" id="reportTableHost"></div>
        <p class="report-status" id="reportStatus" role="status"></p>
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
use App\Http\Controllers\Supplier\ReportController;
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
                    Route::get('/reports', [ReportController::class, 'index'])
                        ->name('reports');
                    Route::get('/reports/data', [ReportController::class, 'data'])
                        ->name('reports.data');

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

## `tests/Feature/ReportBackendTest.php`

```php
<?php

namespace Tests\Feature;

use App\Models\Client;
use App\Models\Delivery;
use App\Models\Flower;
use App\Models\FlowerPrice;
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
use App\Services\ReportService;
use App\Services\SalesPaymentService;
use App\Support\AccountSession;
use App\Support\Money;
use Carbon\CarbonImmutable;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Schema;
use Illuminate\Support\Str;
use Symfony\Component\HttpKernel\Exception\HttpException;
use Tests\TestCase;

class ReportBackendTest extends TestCase
{
    private ReportService $reports;
    private InventoryService $inventory;
    private OrderService $orders;
    private SalesPaymentService $sales;
    private User $admin;
    private User $supplier;
    private bool $testConnectionConfigured = false;
    private string $previousTimezone;

    /** All schemas are created only in a named ephemeral database, never the owner's DB. */
    protected function setUp(): void
    {
        parent::setUp();
        if (! extension_loaded('pdo_sqlite')) {
            $this->markTestSkipped('Isolated report tests require pdo_sqlite.');
        }
        config([
            'database.connections.report_backend_test' => [
                'driver' => 'sqlite', 'database' => ':memory:', 'prefix' => '',
                'foreign_key_constraints' => true,
            ],
            'database.default' => 'report_backend_test',
            'orders.delivery_fee' => null,
            'session.driver' => 'array',
            'cache.default' => 'array',
        ]);
        $this->previousTimezone = date_default_timezone_get();
        date_default_timezone_set('UTC');
        config(['app.timezone' => 'UTC']);
        $this->testConnectionConfigured = true;
        DB::purge('report_backend_test');
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
        $this->reports = app(ReportService::class);
        $this->inventory = app(InventoryService::class);
        $this->orders = app(OrderService::class);
        $this->sales = app(SalesPaymentService::class);
    }

    /** Disconnect only the ephemeral named connection. */
    protected function tearDown(): void
    {
        if ($this->testConnectionConfigured) DB::disconnect('report_backend_test');
        if (isset($this->previousTimezone)) date_default_timezone_set($this->previousTimezone);
        parent::tearDown();
    }

    /** Empty databases still return every report source without invented rows or schema writes. */
    public function test_empty_dataset_contains_all_seven_real_report_sources(): void
    {
        $before = $this->databaseSnapshot();
        $result = $this->reports->data($this->admin);
        $keys = array_keys($result['data']);
        sort($keys);
        $this->assertSame(['clients', 'delivery', 'inventory', 'orders', 'payments', 'sales', 'spoilage'], $keys);
        foreach ($result['data'] as $rows) $this->assertSame([], $rows);
        $this->assertSame('Asia/Manila', $result['timezone']);
        $this->assertStringEndsWith('+08:00', $result['generatedAt']);
        $this->assertSame($before, $this->databaseSnapshot());
    }

    /** Every report reads the same actual module records and never mutates them. */
    public function test_all_seven_sources_reconcile_with_real_module_records_without_writes(): void
    {
        [$client, $stem, $bouquet] = $this->fixture();
        $completed = $this->complete($this->newOrder($client, $stem, 2));
        $payment = $this->pay($completed);
        $this->refund($payment, '5.00');
        $deliveryOrder = $this->newOrder($client, $bouquet, 1, 'Delivery');
        foreach (['Confirmed', 'Processing', 'Ready for Delivery'] as $status) $this->advance($deliveryOrder, $status);
        $delivery = Delivery::where('order_id', $deliveryOrder->id)->firstOrFail();
        $this->inventory->stockOut([
            'request_key' => (string) Str::uuid(), 'flower_id' => $stem->id,
            'quantity' => 3, 'reason' => 'Spoilage', 'remarks' => 'Three inspected damaged stems.',
        ], $this->supplier);
        $before = $this->databaseSnapshot();
        $data = $this->reports->data($this->admin)['data'];
        foreach (['sales' => 1, 'payments' => 1, 'inventory' => 2, 'orders' => 2,
            'delivery' => 1, 'clients' => 1, 'spoilage' => 1] as $key => $count) {
            $this->assertCount($count, $data[$key], $key);
        }
        $sale = $data['sales'][0];
        $this->assertSame(2500, $sale['gross']);
        $this->assertSame(500, $sale['refund']);
        $this->assertSame(2000, $sale['net']);
        $receipt = $data['payments'][0];
        $this->assertSame(2500, $receipt['amount']);
        $this->assertSame(2500, $receipt['verified']);
        $this->assertSame(500, $receipt['refund']);
        $this->assertSame(2000, $receipt['net']);
        $stock = $this->find($data['inventory'], 'code', $stem->code);
        $this->assertSame(15, $stock['onHand']);
        $this->assertSame('Stem', $stock['unit']);
        $arrangements = $this->find($data['inventory'], 'code', $bouquet->code);
        $this->assertSame(5, $arrangements['onHand']);
        $this->assertSame(1, $arrangements['reserved']);
        $this->assertSame(4, $arrangements['available']);
        $this->assertSame('Bouquet', $arrangements['unit']);
        $this->assertSame($delivery->code, $data['delivery'][0]['id']);
        $this->assertSame($deliveryOrder->code, $data['delivery'][0]['order']);
        $this->assertSame(2000, $data['clients'][0]['purchases']);
        $this->assertSame(3, $data['spoilage'][0]['quantity']);
        $this->assertSame($before, $this->databaseSnapshot());
    }

    /** Equal display names cannot merge distinct client identities or historical order snapshots. */
    public function test_clients_are_joined_by_identity_and_use_current_profile_fields(): void
    {
        [, $stem] = $this->fixture();
        $first = $this->newClient('same-a', 'Same Name');
        $second = $this->newClient('same-b', 'Same Name');
        $this->complete($this->newOrder($first, $stem, 2));
        $this->newOrder($second, $stem, 1);
        $first->user->name = 'Current Renamed Client';
        $first->user->save();
        $data = $this->reports->data($this->admin)['data'];
        $a = $this->find($data['clients'], 'id', $first->code);
        $b = $this->find($data['clients'], 'id', $second->code);
        $this->assertSame('Current Renamed Client', $a['name']);
        $this->assertSame('Same Name', $b['name']);
        $this->assertSame(1, $a['orders']);
        $this->assertSame(1, $b['orders']);
        $this->assertSame(2500, $a['purchases']);
        $this->assertSame(0, $b['purchases']);
        $this->assertSame('Same Name', $data['sales'][0]['client']);
    }

    /** Historical completed revenue is based on immutable commercial snapshots, not today's price. */
    public function test_current_selling_price_changes_do_not_reprice_reports(): void
    {
        [$client, $stem] = $this->fixture();
        $order = $this->complete($this->newOrder($client, $stem, 2));
        $flower = $this->find($this->sales->data($this->supplier)['flowers'], 'dbId', $stem->id);
        $this->sales->price($stem, ['request_key' => (string) Str::uuid(), 'version' => $flower['version'],
            'amount' => '99.99'], $this->supplier);
        $data = $this->reports->data($this->admin)['data'];
        $this->assertSame(9999, FlowerPrice::where('flower_id', $stem->id)->firstOrFail()->price_cents);
        $this->assertSame(1250, $order->items()->firstOrFail()->unit_price_cents);
        $this->assertSame(2500, $data['sales'][0]['gross']);
        $this->assertSame(2500, $this->find($data['orders'], 'id', $order->code)['total']);
    }

    /** Pending, verified and refunded collection amounts remain independent of order fulfillment. */
    public function test_payment_states_and_refunds_do_not_change_reported_physical_stock(): void
    {
        [$client, $stem] = $this->fixture();
        $order = $this->newOrder($client, $stem, 2);
        $payment = $this->pay($order, false);
        $pending = $this->reports->data($this->admin)['data'];
        $this->assertSame(2500, $pending['payments'][0]['amount']);
        $this->assertSame(0, $pending['payments'][0]['verified']);
        $this->assertSame(0, $pending['payments'][0]['net']);
        $this->assertSame([], $pending['sales']);
        $this->confirmPayment($payment);
        $verified = $this->reports->data($this->admin)['data'];
        $this->assertSame(2500, $verified['payments'][0]['verified']);
        $this->assertSame(2500, $verified['payments'][0]['net']);
        $this->refund($payment, '5.00');
        $partial = $this->reports->data($this->admin)['data'];
        $this->assertSame('Refunded', $partial['payments'][0]['status']);
        $this->assertSame(500, $partial['payments'][0]['refund']);
        $this->assertSame(2000, $partial['payments'][0]['net']);
        $this->refund($payment, '20.00');
        $full = $this->reports->data($this->admin)['data'];
        $this->assertSame(2500, $full['payments'][0]['verified']);
        $this->assertSame(2500, $full['payments'][0]['refund']);
        $this->assertSame(0, $full['payments'][0]['net']);
        foreach ([$pending, $verified, $partial, $full] as $data) {
            $this->assertSame(20, $this->find($data['inventory'], 'code', $stem->code)['onHand']);
            $this->assertSame([], $data['sales']);
        }
        $this->assertSame('Pending', $order->fresh()->status);
    }

    /** Only actual negative spoilage issues count, never expiry flags or other stock uses. */
    public function test_spoilage_reports_only_qualifying_movement_events(): void
    {
        [$client, $stem, $bouquet] = $this->fixture();
        InventoryBatch::where('flower_id', $stem->id)->update(['expiry_date' => '2020-01-01']);
        $this->inventory->stockOut([
            'request_key' => (string) Str::uuid(), 'flower_id' => $stem->id,
            'quantity' => 2, 'reason' => 'Other use', 'remarks' => 'Display arrangement.',
        ], $this->supplier);
        $this->complete($this->newOrder($client, $stem, 1));
        $this->assertSame([], $this->reports->data($this->admin)['data']['spoilage']);
        foreach ([[$stem, 3], [$bouquet, 5]] as [$item, $quantity]) {
            $this->inventory->stockOut([
                'request_key' => (string) Str::uuid(), 'flower_id' => $item->id,
                'quantity' => $quantity, 'reason' => 'Spoilage', 'remarks' => 'Inspected actual spoilage.',
            ], $this->supplier);
        }
        $data = $this->reports->data($this->admin)['data'];
        $this->assertCount(2, $data['spoilage']);
        $stemEvent = $this->find($data['spoilage'], 'unit', 'Stem');
        $bouquetEvent = $this->find($data['spoilage'], 'unit', 'Bouquet');
        $this->assertSame(3, $stemEvent['quantity']);
        $this->assertSame(5, $bouquetEvent['quantity']);
        $this->assertSame(14, $this->find($data['inventory'], 'code', $stem->code)['onHand']);
        $this->assertSame(0, $this->find($data['inventory'], 'code', $bouquet->code)['onHand']);
        $this->assertTrue(InventoryBatch::where('flower_id', $bouquet->id)->firstOrFail()->spoiled);
    }

    /** Oversubscribed stock remains visible instead of being silently clamped to zero. */
    public function test_inventory_preserves_negative_availability_and_ignores_released_holds(): void
    {
        [, $stem, $bouquet] = $this->fixture();
        InventoryReservation::create(['flower_id' => $stem->id, 'reference' => 'TEST-OVER', 'quantity' => 21, 'status' => 'Active']);
        InventoryReservation::create(['flower_id' => $stem->id, 'reference' => 'TEST-RELEASED', 'quantity' => 10, 'status' => 'Released']);
        $rows = $this->reports->data($this->admin)['data']['inventory'];
        $stock = $this->find($rows, 'code', $stem->code);
        $this->assertSame(20, $stock['onHand']);
        $this->assertSame(21, $stock['reserved']);
        $this->assertSame(-1, $stock['available']);
        $this->assertSame('Out of Stock', $stock['status']);
        $this->assertSame(5, $this->find($rows, 'code', $bouquet->code)['available']);
    }

    /** Refunds restate the original completion day and do not substitute the refund day. */
    public function test_manila_dates_and_original_sale_refund_attribution(): void
    {
        [$client, $stem] = $this->fixture();
        $order = $this->complete($this->newOrder($client, $stem, 2));
        $payment = $this->pay($order);
        $this->refund($payment, '5.00');
        $sale = SalesRecord::where('order_id', $order->id)->firstOrFail();
        $sale->completed_at = CarbonImmutable::parse('2026-11-06T16:00:00+00:00');
        $sale->save();
        OrderPayment::where('id', $payment->id)->update(['created_at' => '2026-11-06 15:59:59']);
        PaymentRefund::query()->update(['created_at' => '2026-11-09 12:00:00']);
        $data = $this->reports->data($this->admin)['data'];
        $this->assertSame('2026-11-07', $data['sales'][0]['date']);
        $this->assertSame(500, $data['sales'][0]['refund']);
        $this->assertSame(2000, $data['sales'][0]['net']);
        $this->assertSame('2026-11-06', $data['payments'][0]['date']);
    }

    /** The report data route and service both enforce Admin-only operational access. */
    public function test_report_access_is_admin_only_and_responses_exclude_credentials(): void
    {
        foreach ([User::ROLE_SUPPLIER, User::ROLE_DELIVERY, User::ROLE_CLIENT] as $role) {
            $user = $this->user('blocked-'.$role, $role);
            try {
                $this->reports->data($user);
                $this->fail('Non-Admin report access must be rejected.');
            } catch (HttpException $exception) {
                $this->assertSame(403, $exception->getStatusCode());
            }
            $this->actingAs($user)->withSession([AccountSession::KEY => AccountSession::signature($user)])
                ->getJson('/supplier/reports/data')->assertForbidden();
        }
        [$client] = $this->fixture();
        $this->actingAs($this->admin)->withSession([AccountSession::KEY => AccountSession::signature($this->admin)])
            ->getJson('/supplier/reports/data')->assertOk()
            ->assertJsonPath('timezone', 'Asia/Manila')
            ->assertJsonMissingPath('data.clients.0.password')
            ->assertJsonMissingPath('data.clients.0.remember_token');
        $inactive = $this->user('inactive-admin', User::ROLE_ADMIN);
        $inactive->is_active = false;
        $this->assertBlocked($inactive);
        $temporary = $this->user('temporary-admin', User::ROLE_ADMIN);
        $temporary->must_change_password = true;
        $this->assertBlocked($temporary);
    }

    private function assertBlocked(User $user): void
    {
        try { $this->reports->data($user); $this->fail('Access must be rejected.'); }
        catch (HttpException $exception) { $this->assertSame(403, $exception->getStatusCode()); }
    }

    private function fixture(): array
    {
        $client = $this->newClient('base-client', 'Report Client');
        $items = [];
        foreach ([['FLW-001', 'Stem', 1250, 20], ['FLW-002', 'Bouquet', 125000, 5]] as [$code, $type, $price, $quantity]) {
            $flower = Flower::create(['code' => $code, 'name' => 'Report '.$type,
                'product_type' => $type, 'category' => 'Roses', 'minimum_stock' => 3,
                'description' => 'Report fixture.', 'status' => 'Active']);
            FlowerPrice::create(['flower_id' => $flower->id, 'price_cents' => $price]);
            $this->inventory->stockIn([
                'request_key' => (string) Str::uuid(), 'flower_id' => $flower->id,
                'quantity' => $quantity, 'unit' => $type, 'received' => '2026-01-02',
                'harvest' => '2026-01-01', 'expiry' => null, 'remarks' => 'Report fixture receipt.',
            ], $this->supplier);
            $items[] = $flower;
        }
        return [$client, ...$items];
    }

    private function newClient(string $label, string $name): Client
    {
        $user = $this->user($label, User::ROLE_CLIENT);
        $user->name = $name; $user->save();
        return Client::create(['code' => 'CLI-'.$user->id, 'user_id' => $user->id,
            'phone' => '09170000000', 'address' => 'Report test address']);
    }

    private function user(string $label, string $role): User
    {
        return User::forceCreate(['name' => 'Report '.$label, 'email' => $label.'@example.test',
            'password' => 'Temporary-report-password!', 'role' => $role,
            'is_active' => true, 'must_change_password' => false]);
    }

    private function newOrder(Client $client, Flower $item, int $quantity, string $method = 'Customer Pickup'): Order
    {
        config(['orders.delivery_fee' => '150.00']);
        return $this->orders->store(['request_key' => (string) Str::uuid(),
            'client_id' => $client->id, 'method' => $method,
            'address' => $method === 'Delivery' ? 'Report delivery address' : null,
            'notes' => null, 'items' => [['flower_id' => $item->id, 'quantity' => $quantity]],
        ], $this->supplier);
    }

    private function advance(Order $order, string $status): Order
    {
        return $this->orders->action($order, ['status' => $status,
            'version' => $this->orders->row($order->fresh(), $this->admin)['version'],
            'collected' => $status === 'Completed'], $this->admin);
    }

    private function complete(Order $order): Order
    {
        foreach (['Confirmed', 'Processing', 'Ready for Pickup', 'Completed'] as $status) {
            $this->advance($order, $status);
        }
        return $order->fresh();
    }

    private function pay(Order $order, bool $confirm = true): OrderPayment
    {
        $row = $this->find($this->sales->data($this->supplier)['orders'], 'dbId', $order->id);
        $this->sales->recordPayment(['request_key' => (string) Str::uuid(),
            'order_id' => $order->id, 'version' => $row['version'], 'method' => 'Cash',
            'amount' => Money::format((int) $order->total_cents)], $this->supplier);
        $payment = OrderPayment::where('order_id', $order->id)->firstOrFail();
        if ($confirm) $this->confirmPayment($payment);
        return $payment->fresh();
    }

    private function confirmPayment(OrderPayment $payment): void
    {
        $row = $this->find($this->sales->data($this->supplier)['payments'], 'dbId', $payment->id);
        $this->sales->confirm($payment, ['request_key' => (string) Str::uuid(),
            'version' => $row['version'], 'reason' => 'Actual collection verified.'], $this->supplier);
    }

    private function refund(OrderPayment $payment, string $amount): void
    {
        $row = $this->find($this->sales->data($this->admin)['payments'], 'dbId', $payment->id);
        $this->sales->refund($payment, ['request_key' => (string) Str::uuid(),
            'version' => $row['version'], 'amount' => $amount,
            'reason' => 'Admin verified the actual refund.'], $this->admin);
    }

    private function find(array $rows, string $key, mixed $value): array
    {
        $row = collect($rows)->first(fn ($row) => $row[$key] === $value);
        $this->assertNotNull($row, 'Report row not found: '.$key);
        return $row;
    }

    private function databaseSnapshot(): array
    {
        $snapshot = [];
        foreach (['users', 'clients', 'flowers', 'flower_prices', 'inventory_batches',
            'inventory_movements', 'inventory_operations', 'inventory_unit_settings', 'inventory_reservations',
            'orders', 'order_items', 'order_histories', 'order_payments', 'sales_records',
            'deliveries', 'delivery_histories', 'finance_operations', 'payment_refunds'] as $table) {
            $snapshot[$table] = DB::table($table)->orderBy('id')->get()->map(fn ($row) => (array) $row)->all();
        }
        return $snapshot;
    }
}
```
