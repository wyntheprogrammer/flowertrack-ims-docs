# FlowerTrack — User Management: full replacement code

Each section below is one complete file. Create or replace the matching project path and copy only its code block. No migration is required.

Prepared against application checkpoint `43695bf`. PHP tests and owner browser acceptance remain pending.

## Installation

```text
composer dump-autoload
php artisan optimize:clear
npm.cmd run build
php artisan test --filter=UserBackendTest
php artisan test --filter=DeliveryBackendTest
php artisan test --filter=OrderBackendTest
```

Sign in again after installation so the account session signature is initialized.

## `app/Http/Controllers/AuthController.php`

```php
<?php

namespace App\Http\Controllers;

use App\Models\User;
use App\Rules\SafeAccountPassword;
use App\Support\AccountSession;
use Illuminate\Support\Facades\DB;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Facades\RateLimiter;
use Illuminate\Support\Str;
use Illuminate\Validation\Rules\Password;
use Illuminate\Validation\ValidationException;
use Illuminate\View\View;

class AuthController extends Controller
{
    public function showLogin(): View|RedirectResponse
    {
        if (Auth::check()) {
            return $this->redirectAfterLogin(Auth::user());
        }

        return view('auth.login');
    }

    public function login(Request $request): RedirectResponse
    {
        $request->merge([
            'email' => Str::lower(trim((string) $request->input('email'))),
        ]);

        $credentials = $request->validate([
            'email' => ['required', 'email', 'max:254'],
            'password' => ['required', 'string', new SafeAccountPassword()],
        ]);

        $key = 'login:'.hash(
            'sha256',
            $credentials['email'].'|'.$request->ip()
        );

        if (RateLimiter::tooManyAttempts($key, 5)) {
            $seconds = RateLimiter::availableIn($key);

            throw ValidationException::withMessages([
                'email' => "Too many login attempts. Try again in {$seconds} seconds.",
            ]);
        }

        $authenticated = Auth::attempt([
            'email' => $credentials['email'],
            'password' => $credentials['password'],
            'is_active' => true,
        ]);

        if (! $authenticated) {
            RateLimiter::hit($key, 60);

            throw ValidationException::withMessages([
                'email' => 'Unable to sign in with these credentials.',
            ]);
        }

        $user = Auth::user();

        // Client storefront authentication will be connected separately.
        if (! in_array($user->role, [
            User::ROLE_ADMIN,
            User::ROLE_SUPPLIER,
            User::ROLE_DELIVERY,
        ], true)) {
            Auth::logout();

            $request->session()->invalidate();
            $request->session()->regenerateToken();

            RateLimiter::hit($key, 60);

            throw ValidationException::withMessages([
                'email' => 'Unable to sign in with these credentials.',
            ]);
        }

        RateLimiter::clear($key);

        // Prevent reuse of the session ID from before login.
        $request->session()->regenerate();
        AccountSession::remember($request, $user);

        return $this->redirectAfterLogin($user);
    }

    public function showChangePassword(Request $request): View|RedirectResponse
    {
        return view('auth.change-password');
    }

    public function updatePassword(Request $request): RedirectResponse
    {
        $validated = $request->validate([
            'current_password' => [
                'required',
                'string',
                'current_password:web',
            ],
            'password' => [
                'required',
                'string',
                'confirmed',
                Password::min(12),
                new SafeAccountPassword(),
            ],
        ]);

        $user = DB::transaction(function () use ($request, $validated) {
            $locked = User::query()->lockForUpdate()->findOrFail($request->user()->id);
            if (!$locked->is_active || !Hash::check($validated['current_password'], $locked->password)) {
                throw ValidationException::withMessages([
                    'current_password' => 'Account credentials changed. Sign in again.',
                ]);
            }
            if (Hash::check($validated['password'], $locked->password)) {
                throw ValidationException::withMessages([
                    'password' => 'Choose a different password from your current password.',
                ]);
            }
            $locked->password = $validated['password'];
            $locked->must_change_password = false;
            $locked->remember_token = Str::random(60);
            $locked->save();
            return $locked;
        });

        Auth::setUser($user);
        $request->session()->regenerate();
        $request->session()->regenerateToken();
        AccountSession::remember($request, $user);

        return $this->redirectAfterLogin($user)
            ->with('status', 'Your password has been changed.');
    }

    public function logout(Request $request): RedirectResponse
    {
        Auth::logout();

        $request->session()->invalidate();
        $request->session()->regenerateToken();

        return redirect()->route('login');
    }

    private function redirectAfterLogin(User $user): RedirectResponse
    {
        if ($user->must_change_password) {
            return redirect()->route('password.change');
        }

        if ($user->isDeliveryPersonnel()) {
            return redirect()->route('delivery.deliveries');
        }

        return redirect()->route('supplier.dashboard');
    }
}
```

## `app/Http/Controllers/Supplier/UserController.php`

```php
<?php

namespace App\Http\Controllers\Supplier;

use App\Http\Controllers\Controller;
use App\Http\Requests\ResetUserPasswordRequest;
use App\Http\Requests\StoreUserRequest;
use App\Http\Requests\UpdateUserRequest;
use App\Http\Requests\UserStatusRequest;
use App\Models\User;
use App\Services\UserService;
use App\Support\AccountSession;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\View\View;

class UserController extends Controller
{
    public function __construct(private readonly UserService $users)
    {
    }

    public function index(): View
    {
        return view('supplier.users');
    }

    public function data(Request $request): JsonResponse
    {
        return response()->json($this->users->data($request->user()));
    }

    public function show(User $user, Request $request): JsonResponse
    {
        return response()->json(['data' => $this->users->row($user, $request->user())]);
    }

    public function store(StoreUserRequest $request): JsonResponse
    {
        $user = $this->users->store($request->validated(), $request->user());
        return response()->json([
            'message' => 'Account created. Share the temporary password privately; a password change is required at first login.',
            'data' => $this->users->row($user, $request->user()),
        ], 201);
    }

    public function update(User $user, UpdateUserRequest $request): JsonResponse
    {
        $updated = $this->users->update($user, $request->validated(), $request->user());
        if ((int) $updated->id === (int) $request->user()->id) {
            $request->session()->regenerate();
            AccountSession::remember($request, $updated);
        }
        return response()->json([
            'message' => 'Account updated. Changes to role, email or status revoke previous account sessions.',
            'data' => $this->users->row($updated, $request->user()),
        ]);
    }

    public function status(User $user, UserStatusRequest $request): JsonResponse
    {
        $updated = $this->users->status($user, $request->validated(), $request->user());
        return response()->json([
            'message' => 'Account status saved.',
            'data' => $this->users->row($updated, $request->user()),
        ]);
    }

    public function resetPassword(User $user, ResetUserPasswordRequest $request): JsonResponse
    {
        $updated = $this->users->resetPassword($user, $request->validated(), $request->user());
        return response()->json([
            'message' => 'Temporary password replaced. Previous sessions are revoked; the account must change password after signing in.',
            'data' => $this->users->row($updated, $request->user()),
        ]);
    }
}
```

## `app/Http/Middleware/EnsureAccountSessionIsCurrent.php`

```php
<?php

namespace App\Http\Middleware;

use App\Support\AccountSession;
use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Symfony\Component\HttpFoundation\Response;

class EnsureAccountSessionIsCurrent
{
    public function handle(Request $request, Closure $next): Response
    {
        $user = $request->user();
        $saved = $request->session()->get(AccountSession::KEY);

        if (!$user || !is_string($saved) || !hash_equals(AccountSession::signature($user), $saved)) {
            Auth::logout();
            $request->session()->invalidate();
            $request->session()->regenerateToken();

            if ($request->expectsJson()) {
                return response()->json(['message' => 'Account access changed. Please sign in again.'], 401);
            }

            return redirect()->route('login')->withErrors([
                'email' => 'Please sign in again to continue.',
            ]);
        }

        return $next($request);
    }
}
```

## `app/Http/Requests/DeliveryActionRequest.php`

```php
<?php

namespace App\Http\Requests;

use App\Models\User;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class DeliveryActionRequest extends FormRequest
{
    public function authorize(): bool
    {
        $user = $this->user();
        if (!$user || !$user->is_active || $user->must_change_password) return false;
        if (in_array($user->role, [User::ROLE_ADMIN, User::ROLE_SUPPLIER], true)) return true;
        $delivery = $this->route('delivery');
        return $user->role === User::ROLE_DELIVERY
            && $delivery instanceof \App\Models\Delivery
            && (int) $delivery->assigned_to === (int) $user->id
            && in_array($this->input('action'), ['deliver', 'fail'], true);
    }

    public function rules(): array
    {
        return [
            'request_key' => ['required', 'uuid'],
            'version' => ['required', 'string', 'size:64', 'regex:/\A[a-fA-F0-9]{64}\z/'],
            'action' => ['required', Rule::in(['schedule', 'assign', 'dispatch', 'deliver', 'fail', 'cancel'])],
            'scheduled_at' => ['exclude_unless:action,schedule', 'required', 'date_format:Y-m-d\TH:i:sP'],
            'assigned_to' => ['exclude_unless:action,assign', 'required', 'integer',
                Rule::exists('users', 'id')->where(fn ($query) => $query->where('role', User::ROLE_DELIVERY)->where('is_active', true))],
            'reason' => ['required_unless:action,dispatch', 'nullable', 'string', 'max:500'],
            'recipient_name' => ['exclude_unless:action,deliver', 'required', 'string', 'max:120'],
            'proof' => ['exclude_unless:action,deliver', 'required', 'file', 'image', 'mimes:jpg,jpeg,png,webp', 'max:5120'],
            'order_id' => ['prohibited'], 'status' => ['prohibited'],
            'proof_path' => ['prohibited'], 'delivered_at' => ['prohibited'], 'stocked_out' => ['prohibited'],
        ];
    }
}
```

## `app/Http/Requests/ResetUserPasswordRequest.php`

```php
<?php

namespace App\Http\Requests;

use App\Services\UserService;

class ResetUserPasswordRequest extends UserAccountRequest
{
    public function rules(): array
    {
        return UserService::rules('reset');
    }
}
```

## `app/Http/Requests/StoreUserRequest.php`

```php
<?php

namespace App\Http\Requests;

use App\Services\UserService;

class StoreUserRequest extends UserAccountRequest
{
    public function rules(): array
    {
        return UserService::rules('store');
    }
}
```

## `app/Http/Requests/UpdateUserRequest.php`

```php
<?php

namespace App\Http\Requests;

use App\Services\UserService;

class UpdateUserRequest extends UserAccountRequest
{
    public function rules(): array
    {
        return UserService::rules('update');
    }
}
```

## `app/Http/Requests/UserAccountRequest.php`

```php
<?php

namespace App\Http\Requests;

use App\Models\User;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Support\Str;

abstract class UserAccountRequest extends FormRequest
{
    public function authorize(): bool
    {
        $user = $this->user();
        return $user !== null && $user->role === User::ROLE_ADMIN
            && $user->is_active && !$user->must_change_password;
    }

    protected function prepareForValidation(): void
    {
        $values = [];
        if (is_string($this->input('name'))) $values['name'] = trim($this->input('name'));
        if (is_string($this->input('email'))) $values['email'] = Str::lower(trim($this->input('email')));
        if ($values !== []) $this->merge($values);
    }
}
```

## `app/Http/Requests/UserStatusRequest.php`

```php
<?php

namespace App\Http\Requests;

use App\Services\UserService;

class UserStatusRequest extends UserAccountRequest
{
    public function rules(): array
    {
        return UserService::rules('status');
    }
}
```

## `app/Rules/SafeAccountPassword.php`

```php
<?php

namespace App\Rules;

use Closure;
use Illuminate\Contracts\Validation\ValidationRule;

/** Bcrypt accepts at most 72 bytes; reject truncation and null bytes explicitly. */
class SafeAccountPassword implements ValidationRule
{
    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        if (is_string($value) && (strlen($value) > 72 || str_contains($value, "\0"))) {
            $fail('The :attribute must be at most 72 UTF-8 bytes and contain no null characters.');
        }
    }
}
```

## `app/Services/DeliveryService.php`

```php
<?php

namespace App\Services;

use App\Models\Delivery;
use App\Models\DeliveryHistory;
use App\Models\Flower;
use App\Models\InventoryReservation;
use App\Models\SalesRecord;
use App\Models\Order;
use App\Models\User;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Facades\Validator;
use Illuminate\Support\Str;
use Illuminate\Validation\ValidationException;
use Carbon\CarbonImmutable;

class DeliveryService
{
    /** Load real delivery records and active assignment options. */
    public function data(User $actor): array
    {
        $this->assertActor($actor, true);
        $query = Delivery::with(['order', 'personnel', 'histories']);
        if ($actor->role === User::ROLE_DELIVERY) {
            $query->where('assigned_to', $actor->id);
        }

        return [
            'data' => $query
                ->orderByDesc('id')
                ->get()
                ->map(fn (Delivery $delivery) => $this->row($delivery, $actor))
                ->all(),
            'personnel' => $actor->role === User::ROLE_DELIVERY ? [] : User::query()
                ->where('role', User::ROLE_DELIVERY)
                ->where('is_active', true)
                ->orderBy('name')
                ->get(['id', 'name'])
                ->map(fn (User $user) => [
                    'id' => (int) $user->id,
                    'name' => $user->name,
                ])
                ->all(),
        ];
    }

    /** Return browser fields without exposing the stored proof path. */
    public function row(Delivery $delivery, User $actor): array
    {
        $this->assertActor($actor, true);
        if ($actor->role === User::ROLE_DELIVERY) {
            abort_unless((int) $delivery->assigned_to === (int) $actor->id, 403);
        }
        $delivery->loadMissing(['order', 'personnel', 'histories']);

        return [
            'dbId' => $delivery->id,
            'id' => $delivery->code,
            'orderId' => $delivery->order->code,
            'client' => $delivery->order->client_name,
            'address' => $delivery->order->address,
            'method' => $delivery->order->method,
            'orderStatus' => $delivery->order->status,
            'status' => $delivery->status,
            'personId' => $delivery->assigned_to === null
                ? null : (int) $delivery->assigned_to,
            'personName' => $delivery->personnel?->name,
            'schedule' => $delivery->scheduled_at?->toIso8601String(),
            'recipient' => $delivery->recipient_name,
            'remarks' => $delivery->remarks,
            'hasProof' => filled($delivery->proof_path),
            'completedAt' => $delivery->delivered_at?->toIso8601String(),
            'version' => hash('sha256', json_encode([
                $delivery->getAttributes(),
                $delivery->order->getAttributes(),
                $delivery->histories->max('id'),
            ], JSON_THROW_ON_ERROR)),
            'history' => $delivery->histories
                ->sortBy('id')
                ->values()
                ->map(fn ($history) => [
                    'date' => $history->created_at->toIso8601String(),
                    'actor' => $history->actor_name,
                    'status' => $history->status,
                    'note' => $history->note,
                ])
                ->all(),
        ];
    }

    /** Limit service access to active operational accounts. */
    private function assertActor(User $actor, bool $allowDelivery = false): void
    {
        abort_unless(
            $actor->is_active
            && !$actor->must_change_password
            && in_array($actor->role, [
                User::ROLE_ADMIN,
                User::ROLE_SUPPLIER,
                ...($allowDelivery ? [User::ROLE_DELIVERY] : []),
            ], true),
            403
        );
    }

    /** Create one delivery record for a ready delivery order. */
    public function createForOrder(Order $order, User $actor): Delivery
    {
        $this->assertActor($actor);

        return DB::transaction(function () use ($order, $actor) {
            $locked = Order::query()
                ->lockForUpdate()
                ->findOrFail($order->id);

            if (
                $locked->method !== 'Delivery'
                || $locked->status !== 'Ready for Delivery'
            ) {
                throw ValidationException::withMessages([
                    'order' => 'The order must be Ready for Delivery.',
                ]);
            }

            $existing = Delivery::query()
                ->where('order_id', $locked->id)
                ->lockForUpdate()
                ->first();

            if ($existing) {
                return $existing;
            }

            $delivery = Delivery::create([
                'code' => 'TMP-' . Str::ulid(),
                'order_id' => $locked->id,
                'status' => 'Pending',
            ]);

            $delivery->code = 'DEL-' . str_pad(
                (string) $delivery->id,
                3,
                '0',
                STR_PAD_LEFT
            );
            $delivery->save();

            DeliveryHistory::create([
                'delivery_id' => $delivery->id,
                'request_key' => (string) Str::uuid(),
                'request_hash' => hash('sha256', 'create:' . $locked->id),
                'action' => 'create',
                'status' => 'Pending',
                'note' => 'Delivery created from ' . $locked->code . '.',
                'actor_id' => $actor->id,
                'actor_name' => $actor->name,
            ]);

            return $delivery;
        });
    }


    /** Apply delivery actions and actual completion with atomic stock fulfillment. */
    public function action(
        Delivery $delivery,
        array $input,
        User $actor
    ): Delivery {
        $this->assertActor($actor, true);

        $action = $input['action'];
        if ($actor->role === User::ROLE_DELIVERY) {
            abort_unless(in_array($action, ['deliver', 'fail'], true), 403);
        }

        if (!in_array($action, ['schedule', 'assign', 'dispatch', 'fail', 'deliver', 'cancel'], true)) {
            throw ValidationException::withMessages([
                'action' => 'This delivery action is not connected yet.',
            ]);
        }

        Validator::make($input, [
            'request_key' => ['required', 'uuid'],
            'version' => ['required', 'string', 'size:64', 'regex:/\A[a-fA-F0-9]{64}\z/'],
            'reason' => ['nullable', 'string', 'max:500'],
            'scheduled_at' => ['required_if:action,schedule', 'nullable', 'date_format:Y-m-d\TH:i:sP'],
            'assigned_to' => ['required_if:action,assign', 'nullable', 'integer', 'min:1'],
            'recipient_name' => ['required_if:action,deliver', 'nullable', 'string', 'max:120'],
            'proof' => ['required_if:action,deliver', 'nullable', 'file', 'image', 'mimes:jpg,jpeg,png,webp', 'max:5120'],
        ])->validate();

        $reason = trim($input['reason'] ?? '');

        if ($reason === '' && $action !== 'dispatch') {
            throw ValidationException::withMessages([
                'reason' => 'Enter the required remarks.',
            ]);
        }

        $proofHash = null;
        if ($action === 'deliver') {
            if (trim($input['recipient_name']) === '') {
                throw ValidationException::withMessages([
                    'recipient_name' => 'Enter the actual recipient name.',
                ]);
            }
            // Inspect actual bytes as well as the upload's reported MIME type.
            $proofPath = $input['proof']->getRealPath();
            $image = $proofPath ? @getimagesize($proofPath) : false;
            if (
                $image === false
                || !in_array($image['mime'] ?? '', [
                    'image/jpeg', 'image/png', 'image/webp',
                ], true)
                || ($image[0] ?? 0) < 1
                || ($image[1] ?? 0) < 1
            ) {
                throw ValidationException::withMessages([
                    'proof' => 'Select a valid JPEG, PNG or WebP image.',
                ]);
            }

            $proofHash = hash_file('sha256', $proofPath);
            if ($proofHash === false) {
                throw ValidationException::withMessages(['proof' => 'Unable to read proof image.']);
            }
        }

        $payload = [
            'delivery_id' => $delivery->id,
            'actor_id' => $actor->id,
            'action' => $action,
            'reason' => $reason,
            'scheduled_at' => $input['scheduled_at'] ?? null,
            'recipient_name' => $action === 'deliver' ? trim($input['recipient_name']) : null,
            'proof_hash' => $proofHash,
            'assigned_to' => isset($input['assigned_to'])
                ? (int) $input['assigned_to']
                : null,
        ];

        $hash = hash(
            'sha256',
            json_encode($payload, JSON_THROW_ON_ERROR)
        );

        // Keep one generated private file across transaction deadlock retries.
        $storedProof = null;

        try {
            $result = DB::transaction(function () use (
                $delivery, $input, $actor, $action, $reason, $hash, &$storedProof
            ) {
                // Lock the order first, consistently with Order Management.
                $order = Order::query()
                    ->lockForUpdate()
                    ->findOrFail($delivery->order_id);

                $locked = Delivery::query()
                    ->lockForUpdate()
                    ->findOrFail($delivery->id);

                if ($actor->role === User::ROLE_DELIVERY) {
                    abort_unless((int) $locked->assigned_to === (int) $actor->id, 403);
                }

                $previous = DeliveryHistory::query()
                    ->where('request_key', $input['request_key'])
                    ->lockForUpdate()
                    ->first();

                if ($previous) {
                    if (
                        (int) $previous->delivery_id !== (int) $locked->id
                        || !hash_equals($previous->request_hash, $hash)
                    ) {
                        throw ValidationException::withMessages([
                            'request_key' => 'This request key was already used.',
                        ]);
                    }

                    return $locked;
                }

                $current = $this->row($locked, $actor);

                if (!hash_equals($current['version'], $input['version'])) {
                    throw ValidationException::withMessages([
                        'version' => 'Delivery changed. Refresh and try again.',
                    ]);
                }

                $allowedStates = match ($action) {
                    'schedule', 'assign' => [
                        'Pending', 'Scheduled', 'Assigned', 'Rescheduled', 'Failed',
                    ],
                    'dispatch' => ['Assigned', 'Scheduled', 'Rescheduled'],
                    'cancel' => ['Pending', 'Scheduled', 'Assigned', 'Rescheduled', 'Failed'],
                    'fail', 'deliver' => ['Out for Delivery'],
                };

                $validOrderStatus = $order->status === 'Ready for Delivery';

                if (
                    $locked->status === 'Failed'
                    && in_array($action, ['schedule', 'assign', 'cancel'], true)
                ) {
                    $validOrderStatus = in_array(
                        $order->status,
                        ['Ready for Delivery', 'Out for Delivery'],
                        true
                    );
                }

                if (in_array($action, ['fail', 'deliver'], true)) {
                    $validOrderStatus = $order->status === 'Out for Delivery';
                }

                if (
                    $order->method !== 'Delivery'
                    || !in_array($locked->status, $allowedStates, true)
                    || !$validOrderStatus
                    || $order->stocked_out
                ) {
                    throw ValidationException::withMessages([
                        'action' => 'This action is unavailable for the current delivery.',
                    ]);
                }

                $before = [
                    'scheduled_at' => $locked->scheduled_at?->toIso8601String(),
                    'assigned_to' => $locked->assigned_to,
                ];

                if ($action === 'schedule') {
                    $schedule = CarbonImmutable::parse($input['scheduled_at'])
                        ->utc();

                    if (
                        !$schedule->isFuture()
                        || $locked->scheduled_at?->equalTo($schedule)
                    ) {
                        throw ValidationException::withMessages([
                            'scheduled_at' => 'Choose a different future schedule.',
                        ]);
                    }

                    $rescheduled = $locked->scheduled_at !== null
                        || $locked->status === 'Failed';

                    $locked->scheduled_at = $schedule;
                    $locked->status = $rescheduled
                        ? 'Rescheduled'
                        : ($locked->assigned_to ? 'Assigned' : 'Scheduled');

                    if ($order->status === 'Out for Delivery') {
                        $order->status = 'Ready for Delivery';
                        $order->save();

                        $order->histories()->create([
                            'status' => $order->status,
                            'note' => 'Failed delivery rescheduled. ' . $reason,
                            'actor_id' => $actor->id,
                            'actor_name' => $actor->name,
                        ]);
                    }
                } elseif ($action === 'assign') {
                    $person = User::query()
                        ->lockForUpdate()
                        ->find($input['assigned_to']);

                    if (
                        !$person
                        || !$person->is_active
                        || $person->role !== User::ROLE_DELIVERY
                        || (int) $locked->assigned_to === (int) $person->id
                    ) {
                        throw ValidationException::withMessages([
                            'assigned_to' => 'Choose different, active delivery personnel.',
                        ]);
                    }

                    $locked->assigned_to = $person->id;

                    if ($locked->status !== 'Failed') {
                        $locked->status = 'Assigned';
                    }
                }


                if ($action === 'dispatch') {
                    $person = User::query()
                        ->lockForUpdate()
                        ->find($locked->assigned_to);

                    if (
                        !in_array(
                            $locked->status,
                            ['Assigned', 'Scheduled', 'Rescheduled'],
                            true
                        )
                        || !$locked->scheduled_at
                        || !$person
                        || !$person->is_active
                        || $person->role !== User::ROLE_DELIVERY
                        || $order->status !== 'Ready for Delivery'
                        || $order->stocked_out
                    ) {
                        throw ValidationException::withMessages([
                            'action' => 'Dispatch requires a scheduled delivery, active personnel, and a ready order.',
                        ]);
                    }

                    $locked->status = 'Out for Delivery';
                    $locked->dispatched_at = now();

                    $order->status = 'Out for Delivery';
                    $order->save();

                    $reason = $reason !== ''
                        ? $reason
                        : 'Order handed to assigned delivery personnel.';

                    $order->histories()->create([
                        'status' => $order->status,
                        'note' => $reason,
                        'actor_id' => $actor->id,
                        'actor_name' => $actor->name,
                    ]);
                }


                if ($action === 'fail') {
                    $locked->status = 'Failed';

                    $order->histories()->create([
                        'status' => $order->status,
                        'note' => 'Delivery attempt failed. '.$reason,
                        'actor_id' => $actor->id,
                        'actor_name' => $actor->name,
                    ]);
                }

                if ($action === 'cancel') {
                    $locked->status = 'Cancelled';

                    $order->histories()->create([
                        'status' => $order->status,
                        'note' => 'Delivery cancelled; linked order and reservation retained. '.$reason,
                        'actor_id' => $actor->id,
                        'actor_name' => $actor->name,
                    ]);
                }

                if ($action === 'deliver') {
                    $this->completeDelivery($locked, $order, $input, $actor, $storedProof);
                }

                $locked->remarks = $reason;
                $locked->save();

                DeliveryHistory::create([
                    'delivery_id' => $locked->id,
                    'request_key' => $input['request_key'],
                    'request_hash' => $hash,
                    'action' => $action,
                    'status' => $locked->status,
                    'note' => $reason,
                    'actor_id' => $actor->id,
                    'actor_name' => $actor->name,
                    'metadata' => [
                        'before' => $before,
                        'after' => [
                            'scheduled_at' => $locked->scheduled_at?->toIso8601String(),
                            'assigned_to' => $locked->assigned_to,
                        ],
                    ],
                ]);

                return $locked->fresh();
            }, 3);
        } catch (\Throwable $exception) {
            $this->cleanupUnusedProof($storedProof);
            throw $exception;
        }

        // A matching replay can return the existing record without using this file.
        if ($storedProof !== null && $result->proof_path !== $storedProof) {
            $this->cleanupUnusedProof($storedProof);
        }

        return $result;
    }

    /** Complete the delivery, stock issue and sale in the caller's transaction. */
    private function completeDelivery(
        Delivery $delivery,
        Order $order,
        array $input,
        User $actor,
        ?string &$storedProof
    ): void {
        if (
            $delivery->status !== 'Out for Delivery'
            || $order->status !== 'Out for Delivery'
            || $order->method !== 'Delivery'
            || $order->stocked_out
        ) {
            throw ValidationException::withMessages([
                'action' => 'Only an outstanding dispatched delivery can be completed.',
            ]);
        }

        $items = $order->items()->orderBy('flower_id')->lockForUpdate()->get();
        if ($items->isEmpty()) {
            throw ValidationException::withMessages(['order' => 'This order has no items.']);
        }

        // Acquire every Flower lock before any reservation lock, matching OrderService.
        $flowers = Flower::query()
            ->whereIn('id', $items->pluck('flower_id'))
            ->orderBy('id')
            ->lockForUpdate()
            ->get();
        if ($flowers->count() !== $items->pluck('flower_id')->unique()->count()) {
            throw ValidationException::withMessages(['order' => 'An order item is missing.']);
        }

        if ($storedProof === null) {
            $path = $input['proof']->store('delivery-proofs', 'local');
            if (!is_string($path) || $path === '') {
                throw ValidationException::withMessages(['proof' => 'Proof upload failed.']);
            }
            $storedProof = $path;
        }

        foreach ($items as $item) {
            $reservation = InventoryReservation::query()
                ->where('flower_id', $item->flower_id)
                ->where('reference', $order->code)
                ->lockForUpdate()
                ->first();

            if (
                !$reservation
                || $reservation->status !== 'Active'
                || (int) $reservation->quantity !== (int) $item->quantity
            ) {
                throw ValidationException::withMessages([
                    'order' => 'The active stock reservation is missing or inconsistent.',
                ]);
            }

            app(InventoryService::class)->fulfillOrderReservation(
                $reservation,
                $item->fulfillment_key,
                $actor,
                $order->code
            );
        }

        $completedAt = now();
        $order->status = 'Completed';
        $order->stocked_out = true;
        $order->completed_at = $completedAt;
        $order->save();

        $sale = SalesRecord::create([
            'code' => 'TMP-'.Str::ulid(),
            'order_id' => $order->id,
            'client_id' => $order->client_id,
            'client_name' => $order->client_name,
            'client_code' => $order->client_code,
            'items' => $items->map(fn ($item) => [
                'flowerDbId' => (int) $item->flower_id,
                'flowerId' => $item->flower_code,
                'flowerName' => $item->name,
                'itemType' => $item->product_type,
                'baseUnit' => $item->base_unit,
                'quantity' => (int) $item->quantity,
                'unitPriceCents' => (int) $item->unit_price_cents,
                'subtotalCents' => (int) $item->subtotal_cents,
            ])->values()->all(),
            'subtotal_cents' => $order->subtotal_cents,
            'fee_cents' => $order->fee_cents,
            'total_cents' => $order->total_cents,
            'completed_at' => $completedAt,
        ]);
        $sale->code = 'SALE-'.str_pad((string) $sale->id, 3, '0', STR_PAD_LEFT);
        $sale->save();

        $delivery->status = 'Delivered';
        $delivery->recipient_name = trim($input['recipient_name']);
        $delivery->proof_path = $storedProof;
        $delivery->delivered_at = $completedAt;

        $order->histories()->create([
            'status' => 'Completed',
            'note' => 'Delivery received by '.$delivery->recipient_name
                .'; reserved stock issued and sale recorded.',
            'actor_id' => $actor->id,
            'actor_name' => $actor->name,
        ]);
    }

    /** Remove rolled-back uploads only after confirming no committed delivery uses them. */
    private function cleanupUnusedProof(?string $path): void
    {
        if ($path === null) {
            return;
        }

        try {
            if (!Delivery::query()->where('proof_path', $path)->exists()) {
                if (!Storage::disk('local')->delete($path)) {
                    Log::warning('Delivery proof cleanup did not complete.', ['path' => $path]);
                }
            }
        } catch (\Throwable $exception) {
            // Retain the file if commit state cannot be verified; preserve the original error.
            Log::warning('Delivery proof retained for reconciliation.', [
                'path' => $path,
                'error' => $exception->getMessage(),
            ]);
        }
    }
}
```

## `app/Services/UserService.php`

```php
<?php

namespace App\Services;

use App\Models\User;
use App\Rules\SafeAccountPassword;
use Illuminate\Database\QueryException;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Schema;
use Illuminate\Support\Facades\Validator;
use Illuminate\Support\Str;
use Illuminate\Validation\Rule;
use Illuminate\Validation\Rules\Password;
use Illuminate\Validation\ValidationException;

class UserService
{
    public const ROLES = [
        User::ROLE_ADMIN => 'Admin',
        User::ROLE_SUPPLIER => 'Supplier/Staff',
        User::ROLE_DELIVERY => 'Delivery Personnel',
    ];

    /** Share validation rules between HTTP requests and direct service callers. */
    public static function rules(string $operation): array
    {
        $version = ['required', 'string', 'size:64', 'regex:/\A[a-f0-9]{64}\z/'];
        $password = ['required', 'string', Password::min(12), new SafeAccountPassword()];
        $rules = match ($operation) {
            'store' => [
                'name' => ['required', 'string', 'max:120'],
                'email' => ['required', 'email', 'max:254'],
                'role' => ['required', Rule::in(array_keys(self::ROLES))],
                'status' => ['required', Rule::in(['Active', 'Inactive'])],
                'password' => $password,
            ],
            'update' => [
                'name' => ['required', 'string', 'max:120'],
                'email' => ['required', 'email', 'max:254'],
                'role' => ['required', Rule::in(array_keys(self::ROLES))],
                'status' => ['required', Rule::in(['Active', 'Inactive'])],
                'version' => $version,
            ],
            'status' => [
                'status' => ['required', Rule::in(['Active', 'Inactive'])],
                'version' => $version,
            ],
            'reset' => ['password' => $password, 'version' => $version],
            default => throw new \InvalidArgumentException('Unknown account operation.'),
        };

        foreach (['id', 'code', 'user_id', 'is_active', 'must_change_password',
            'remember_token', 'password_hash', 'email_verified_at', 'created_at', 'updated_at'] as $field) {
            $rules[$field] = ['prohibited'];
        }
        if (!in_array($operation, ['store', 'reset'], true)) {
            $rules['password'] = ['prohibited'];
        }

        return $rules;
    }

    /** List operational accounts only; Client identities stay in Client Management. */
    public function data(User $actor): array
    {
        $this->authorize($actor);
        $query = User::query()->whereIn('role', array_keys(self::ROLES));
        if (Schema::hasTable('clients')) {
            $query->whereNotIn('id', DB::table('clients')->select('user_id'));
        }
        $users = $query->orderBy('id')->get();
        $activeAdmins = $users->where('role', User::ROLE_ADMIN)->where('is_active', true)->count();

        return ['data' => $users->map(fn (User $user) => $this->row($user, $actor, $activeAdmins))->all()];
    }

    /** Expose a credential-free DTO and opaque stale-update version. */
    public function row(User $user, User $actor, ?int $activeAdmins = null): array
    {
        $this->authorize($actor);
        $this->operational($user);
        $activeAdmins ??= User::where('role', User::ROLE_ADMIN)->where('is_active', true)->count();
        $self = (int) $user->id === (int) $actor->id;
        $last = $user->role === User::ROLE_ADMIN && $user->is_active && $activeAdmins <= 1;

        return [
            'dbId' => (int) $user->id,
            'id' => 'USR-'.str_pad((string) $user->id, 3, '0', STR_PAD_LEFT),
            'name' => $user->name, 'email' => $user->email,
            'role' => self::ROLES[$user->role], 'roleValue' => $user->role,
            'status' => $user->is_active ? 'Active' : 'Inactive',
            'mustChangePassword' => (bool) $user->must_change_password,
            'created' => $user->created_at?->copy()->setTimezone('Asia/Manila')->toIso8601String(),
            'version' => $this->version($user),
            'isSelf' => $self, 'isLastAdmin' => $last,
            'canReset' => !$self, 'canChangeStatus' => !$self && !$last,
        ];
    }

    /** Create an account with a hashed temporary password and forced first-login change. */
    public function store(array $input, User $actor): User
    {
        $values = $this->validateInput('store', $input);
        return $this->write(function () use ($values, $actor) {
            $this->lockAdmins($actor);
            $this->uniqueEmail($values['email']);
            $user = new User();
            $user->name = $values['name'];
            $user->email = $values['email'];
            $user->role = $values['role'];
            $user->is_active = $values['status'] === 'Active';
            $user->password = $values['password'];
            $user->must_change_password = true;
            $user->save();
            $this->audit('created', $actor, $user, ['name', 'email', 'role', 'status']);
            return $user;
        });
    }

    /** Change only profile/access fields; passwords require the separate reset action. */
    public function update(User $target, array $input, User $actor): User
    {
        $values = $this->validateInput('update', $input);
        return $this->write(function () use ($target, $values, $actor) {
            $admins = $this->lockAdmins($actor);
            $user = User::query()->lockForUpdate()->findOrFail($target->id);
            $this->operational($user);
            $this->assertVersion($user, $values['version']);
            $active = $values['status'] === 'Active';
            $this->protectAccess($user, $actor, $values['role'], $active, $admins);
            $this->uniqueEmail($values['email'], (int) $user->id);
            $accessChanged = $user->email !== $values['email']
                || $user->role !== $values['role'] || (bool) $user->is_active !== $active;
            $fields = [];
            foreach (['name', 'email', 'role'] as $field) {
                if ($user->{$field} !== $values[$field]) $fields[] = $field;
            }
            if ((bool) $user->is_active !== $active) $fields[] = 'status';
            if ($user->email !== $values['email']) $user->email_verified_at = null;
            $user->name = $values['name'];
            $user->email = $values['email'];
            $user->role = $values['role'];
            $user->is_active = $active;
            if ($accessChanged) $user->remember_token = Str::random(60);
            $user->save();
            if ($fields !== []) $this->audit('updated', $actor, $user, $fields);
            return $user;
        });
    }

    /** Set an explicit desired status; a matching retry is a no-op, never a toggle. */
    public function status(User $target, array $input, User $actor): User
    {
        $values = $this->validateInput('status', $input);
        return $this->write(function () use ($target, $values, $actor) {
            $admins = $this->lockAdmins($actor);
            $user = User::query()->lockForUpdate()->findOrFail($target->id);
            $this->operational($user);
            $active = $values['status'] === 'Active';
            if ((bool) $user->is_active === $active) return $user;
            $this->assertVersion($user, $values['version']);
            $this->protectAccess($user, $actor, $user->role, $active, $admins);
            $user->is_active = $active;
            $user->remember_token = Str::random(60);
            $user->save();
            $this->audit('status_changed', $actor, $user, ['status']);
            return $user;
        });
    }

    /** Issue a fresh temporary password without returning or logging the credential. */
    public function resetPassword(User $target, array $input, User $actor): User
    {
        $values = $this->validateInput('reset', $input);
        return $this->write(function () use ($target, $values, $actor) {
            $this->lockAdmins($actor);
            $user = User::query()->lockForUpdate()->findOrFail($target->id);
            $this->operational($user);
            if ((int) $user->id === (int) $actor->id) {
                $this->invalid('password', 'Use your own Change Password screen instead of an Admin reset.');
            }
            $this->assertVersion($user, $values['version']);
            if (Hash::check($values['password'], $user->password)) {
                $this->invalid('password', 'Choose a different temporary password.');
            }
            $user->password = $values['password'];
            $user->must_change_password = true;
            $user->remember_token = Str::random(60);
            $user->save();
            $this->audit('temporary_password_reset', $actor, $user, ['password', 'must_change_password']);
            return $user;
        });
    }

    /** Lock Admin rows in ID order so concurrent changes cannot remove the last active Admin. */
    private function lockAdmins(User $actor)
    {
        $admins = User::query()->where('role', User::ROLE_ADMIN)->orderBy('id')->lockForUpdate()->get();
        $current = $admins->firstWhere('id', $actor->id);
        abort_unless($current && $current->is_active && !$current->must_change_password, 403);
        return $admins;
    }

    private function authorize(User $actor): void
    {
        abort_unless($actor->role === User::ROLE_ADMIN && $actor->is_active && !$actor->must_change_password, 403);
    }

    private function operational(User $user): void
    {
        abort_unless(isset(self::ROLES[$user->role]), 404);
        if (Schema::hasTable('clients')) {
            abort_if(DB::table('clients')->where('user_id', $user->id)->exists(), 404);
        }
    }

    private function protectAccess(User $user, User $actor, string $role, bool $active, $admins): void
    {
        if ((int) $user->id === (int) $actor->id && ($role !== User::ROLE_ADMIN || !$active)) {
            $this->invalid('role', 'You cannot deactivate or remove Admin access from your own account.');
        }
        if ($user->role === User::ROLE_ADMIN && $user->is_active
            && ($role !== User::ROLE_ADMIN || !$active)
            && $admins->where('is_active', true)->count() <= 1) {
            $this->invalid('role', 'The last active Admin must remain active and retain the Admin role.');
        }
    }

    private function version(User $user): string
    {
        return hash_hmac('sha256', json_encode([
            (int) $user->id, $user->name, $user->email, $user->role,
            (bool) $user->is_active, (bool) $user->must_change_password,
            $user->password, $user->remember_token,
            $user->email_verified_at?->format('Y-m-d H:i:s'),
            $user->updated_at?->format('Y-m-d H:i:s'),
        ], JSON_THROW_ON_ERROR), (string) config('app.key'));
    }

    private function assertVersion(User $user, string $expected): void
    {
        if (!hash_equals($this->version($user), $expected)) {
            $this->invalid('version', 'Account changed. Close and reopen this dialog.');
        }
    }

    private function uniqueEmail(string $email, ?int $ignore = null): void
    {
        $query = User::query()->whereRaw('LOWER(email) = ?', [$email]);
        if ($ignore !== null) $query->where('id', '!=', $ignore);
        if ($query->exists()) $this->invalid('email', 'This email already belongs to an account.');
    }

    private function validateInput(string $operation, array $input): array
    {
        if (isset($input['name']) && is_string($input['name'])) $input['name'] = trim($input['name']);
        if (isset($input['email']) && is_string($input['email'])) $input['email'] = Str::lower(trim($input['email']));
        return Validator::make($input, self::rules($operation))->validate();
    }

    /** Convert only email-uniqueness races into a validation response. */
    private function write(callable $callback): User
    {
        try {
            return DB::transaction($callback, 3);
        } catch (QueryException $exception) {
            if (in_array((int) ($exception->errorInfo[1] ?? 0), [1062, 19], true)
                && str_contains(strtolower($exception->getMessage()), 'email')) {
                $this->invalid('email', 'This email already belongs to an account.');
            }
            throw $exception;
        }
    }

    /** Record safe event metadata after commit; never include passwords, hashes or tokens. */
    private function audit(string $action, User $actor, User $target, array $fields): void
    {
        $context = ['actor_id' => (int) $actor->id, 'user_id' => (int) $target->id, 'changed_fields' => $fields];
        DB::afterCommit(static fn () => Log::info('Account '.$action, $context));
    }

    private function invalid(string $field, string $message): never
    {
        throw ValidationException::withMessages([$field => $message]);
    }
}
```

## `app/Support/AccountSession.php`

```php
<?php

namespace App\Support;

use App\Models\User;
use Illuminate\Http\Request;

class AccountSession
{
    public const KEY = 'account_security_signature';

    /** Changes to credentials, role, email or access flags revoke older sessions. */
    public static function signature(User $user): string
    {
        return hash_hmac('sha256', json_encode([
            (int) $user->id, $user->email, $user->password,
            $user->role, (bool) $user->is_active,
            (bool) $user->must_change_password, $user->remember_token,
        ], JSON_THROW_ON_ERROR), (string) config('app.key'));
    }

    public static function remember(Request $request, User $user): void
    {
        $request->session()->put(self::KEY, self::signature($user));
    }
}
```

## `resources/js/supplier/deliveries.js`

```javascript
import DataTable from 'datatables.net-bs5';
import { Modal } from 'bootstrap';

const escapeHtml = (value) =>
    String(value ?? '').replace(/[&<>"']/g, (character) => ({
        '&': '&amp;',
        '<': '&lt;',
        '>': '&gt;',
        '"': '&quot;',
        "'": '&#039;',
    })[character]);

function badge(value) {
    const className = value.toLowerCase().replaceAll(' ', '-');

    return `
        <span class="delivery-badge ${className}">
            ${escapeHtml(value)}
        </span>
    `;
}

function formatDateTime(value) {
    if (!value) return 'Not scheduled';

    return new Intl.DateTimeFormat('en-PH', {
        timeZone: 'Asia/Manila',
        dateStyle: 'medium',
        timeStyle: 'short',
    }).format(new Date(value));
}

/* Format the schedule input in the shop's timezone. */
function localInputValue(value) {
    if (!value) return '';
    const parts = Object.fromEntries(new Intl.DateTimeFormat('en-GB', {
        timeZone: 'Asia/Manila', year: 'numeric', month: '2-digit', day: '2-digit',
        hour: '2-digit', minute: '2-digit', hourCycle: 'h23',
    }).formatToParts(new Date(value)).map((part) => [part.type, part.value]));
    return `${parts.year}-${parts.month}-${parts.day}T${parts.hour}:${parts.minute}`;
}

/* Send an explicit Manila timezone offset accepted by the PHP request rules. */
function schedulePayload(value) {
    if (!/^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}$/.test(value)) {
        throw new Error('Choose a valid delivery schedule.');
    }
    const formatted = `${value}:00+08:00`;
    const time = new Date(formatted).getTime();
    if (!Number.isFinite(time) || time <= Date.now()) {
        throw new Error('Choose a future delivery date and time.');
    }
    return formatted;
}

class DeliveryManagementUI {
    /* Initialize the approved table layout using database records only. */
    constructor(element) {
        this.element = element;
        this.base = element.dataset.url.replace(/\/$/, '');
        this.csrf = element.dataset.csrf;
        this.isPersonnel = element.dataset.role === 'delivery';
        this.deliveries = [];
        this.personnel = [];
        this.selected = null;
        this.busy = false;
        this.loading = false;
        this.manageModal = Modal.getOrCreateInstance(this.el('deliveryManageModal'));
        this.viewModal = Modal.getOrCreateInstance(this.el('deliveryViewModal'));
        this.initializeTable();
        this.mountScroller();
        this.bindEvents();
        this.load().catch((error) => this.announce(error.message));
    }

    /* Find a page or modal element. */
    el(id) { return document.getElementById(id); }

    /* Announce loading, success or recovery messages. */
    announce(message) { this.el('deliveryAnnouncement').textContent = message; }

    /* Match numeric account IDs and retain inactive personnel names on old records. */
    personName(id) {
        return this.personnel.find((person) => String(person.id) === String(id))?.name
            || this.deliveries.find((row) => id != null && String(row.personId) === String(id))?.personName
            || 'Unassigned';
    }

    /* Only current active delivery accounts may receive a new assignment. */
    activePerson(id) {
        return this.personnel.some((person) => String(person.id) === String(id));
    }

    /* Terminal deliveries expose details only. */
    terminal(delivery) { return ['Delivered', 'Cancelled'].includes(delivery.status); }

    /* Show connected actions; the server rechecks every workflow transition. */
    actionsFor(delivery) {
        if (this.terminal(delivery) || delivery.method !== 'Delivery') return [];
        if (delivery.status === 'Out for Delivery') {
            return delivery.orderStatus === 'Out for Delivery'
                ? [['deliver', 'Mark Delivered'], ['fail', 'Record Failed Attempt']] : [];
        }
        if (this.isPersonnel) return [];
        const states = ['Pending', 'Scheduled', 'Assigned', 'Rescheduled', 'Failed'];
        const orderReady = delivery.orderStatus === 'Ready for Delivery';
        const failedAttempt = delivery.status === 'Failed' && delivery.orderStatus === 'Out for Delivery';
        if (!states.includes(delivery.status) || (!orderReady && !failedAttempt)) return [];
        const actions = [
            ['schedule', delivery.schedule ? 'Reschedule Delivery' : 'Schedule Delivery'],
            ['assign', delivery.personId ? 'Reassign Personnel' : 'Assign Personnel'],
        ];
        if (['Assigned', 'Scheduled', 'Rescheduled'].includes(delivery.status)
            && delivery.schedule && this.activePerson(delivery.personId) && orderReady) {
            actions.push(['dispatch', 'Dispatch / Out for Delivery']);
        }
        actions.push(['cancel', 'Cancel Delivery']);
        return actions;
    }

    /* Preserve column alignment, native sorting and compact pagination. */
    initializeTable() {
        this.table = new DataTable(this.element, {
            data: this.deliveries,
            autoWidth: false,
            pageLength: 5,
            order: [[0, 'asc']],
            layout: {
                topStart: null,
                topEnd: null,
                bottomStart: 'info',
                bottomEnd: 'paging',
            },
            language: {
                emptyTable: 'No deliveries available.',
                zeroRecords: 'No matching deliveries found.',
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
            columns: [
                {
                    data: 'id',
                    render: (value, type, row) =>
                        type === 'display'
                            ? `
                                <div class="delivery-cell">
                                    <strong>${escapeHtml(value)}</strong>
                                    <small>${escapeHtml(row.orderId)}</small>
                                </div>
                            `
                            : `${value} ${row.orderId}`,
                },
                {
                    data: 'client',
                    render: (value, type, row) =>
                        type === 'display'
                            ? `
                                <div class="delivery-cell">
                                    <strong>${escapeHtml(value)}</strong>
                                    <small>${escapeHtml(row.address)}</small>
                                </div>
                            `
                            : value,
                },
                {
                    data: 'personId',
                    render: (value, type) => {
                        const name = this.personName(value);
                        return type === 'display' ? escapeHtml(name) : name;
                    },
                },
                {
                    data: 'schedule',
                    render: (value, type) => {
                        if (type === 'sort' || type === 'type') {
                            return value ? new Date(value).getTime() : 0;
                        }

                        return type === 'display'
                            ? escapeHtml(formatDateTime(value))
                            : value;
                    },
                },
                {
                    data: 'status',
                    render: (value, type) =>
                        type === 'display' ? badge(value) : value,
                },
                {
                    data: null,
                    orderable: false,
                    searchable: false,
                    render: (_value, type, row) => {
                        if (type !== 'display') return '';

                        return `
                            <div class="delivery-actions">
                                <button
                                    type="button"
                                    class="delivery-action-btn view"
                                    data-delivery-action="view"
                                    data-delivery-id="${escapeHtml(row.id)}"
                                    title="View delivery"
                                    aria-label="View ${escapeHtml(row.id)}"
                                >
                                    <i class="bi bi-eye" aria-hidden="true"></i>
                                </button>

                                ${this.actionsFor(row).length ? `
                                    <button
                                        type="button"
                                        class="delivery-action-btn manage"
                                        data-delivery-action="manage"
                                        data-delivery-id="${escapeHtml(row.id)}"
                                        title="Manage delivery"
                                        aria-label="Manage ${escapeHtml(row.id)}"
                                    >
                                        <i class="bi bi-pencil-square" aria-hidden="true"></i>
                                    </button>
                                ` : ''}
                            </div>
                        `;
                    },
                },
            ],
        });
    }

    /* Keep horizontal scrolling inside the table only. */
    mountScroller() {
        const wrapper = document.createElement('div');

        wrapper.className = 'delivery-table-scroll';
        wrapper.tabIndex = 0;
        wrapper.setAttribute('role', 'region');
        wrapper.setAttribute(
            'aria-label',
            'Deliveries table. Scroll horizontally to see all columns.',
        );

        this.element.before(wrapper);
        wrapper.append(this.element);
    }

    /* Refresh displayed rows while retaining the current page. */
    refresh() {
        this.table.clear();
        this.table.rows.add(this.deliveries);
        this.table.draw(false);
    }

    /* Read JSON while preserving Laravel validation errors and session failures. */
    async request(url, options = {}) {
        const response = await fetch(url, {
            ...options, credentials: 'same-origin',
            headers: { Accept: 'application/json', 'X-Requested-With': 'XMLHttpRequest',
                'X-CSRF-TOKEN': this.csrf, ...options.headers },
        });
        const result = await response.json().catch(() => null);
        if (!response.ok || !result || response.redirected) {
            const defaults = {
                401: 'Please sign in again.', 403: 'Your account cannot perform this action.',
                404: 'Delivery not found. Refresh the list.', 419: 'Your session expired. Refresh the page.',
                413: 'The upload exceeds the server limit. Choose a smaller image.',
            };
            const error = new Error(defaults[response.status]
                || (result?.errors ? Object.values(result.errors).flat().join(' ') : null)
                || (response.status >= 500 ? 'Server error. Refresh the delivery before retrying.' : result?.message)
                || 'Unable to load the response. Refresh the page.');
            error.status = response.status;
            error.errors = result?.errors || {};
            throw error;
        }
        return result;
    }

    /* Load the canonical list and active personnel; never create records on a GET. */
    async load() {
        this.loading = true;
        try {
            const result = await this.request(`${this.base}/data`);
            if (!Array.isArray(result.data) || !Array.isArray(result.personnel)) {
                throw new Error('Unexpected delivery response. Refresh the page.');
            }
            this.deliveries = result.data;
            this.personnel = result.personnel;
            this.refresh();
            this.announce(this.deliveries.length ? '' : (this.isPersonnel ? 'No deliveries are assigned to your account.' : 'No delivery records yet. Mark a Delivery order Ready for Delivery.'));
        } finally { this.loading = false; }
    }

    /* Bind compact search, page size, modal actions and busy guards. */
    bindEvents() {
        this.el('deliverySearch').addEventListener('input', (event) => this.table.search(event.target.value).draw());
        this.el('deliveryPageLength').addEventListener('change', (event) => this.table.page.len(Number(event.target.value)).draw());
        this.element.addEventListener('click', (event) => {
            const button = event.target.closest('[data-delivery-action]');
            if (!button || this.busy || this.loading) return;
            const delivery = this.deliveries.find((row) => row.id === button.dataset.deliveryId);
            if (delivery) this.openRecord(delivery, button.dataset.deliveryAction);
        });
        this.el('deliveryAction').addEventListener('change', () => this.renderActionFields());
        this.el('deliveryManageForm').addEventListener('submit', (event) => {
            event.preventDefault(); this.saveAction();
        });
        this.el('deliveryManageModal').addEventListener('hide.bs.modal', (event) => {
            if (this.busy) event.preventDefault();
        });
        this.el('deliveryManageModal').addEventListener('hidden.bs.modal', () => {
            this.selected = null;
            this.el('deliveryManageForm').reset();
            this.el('deliveryActionFields').innerHTML = '';
        });
    }

    /* Retrieve current details and version before opening either modal. */
    async openRecord(delivery, action) {
        this.loading = true;
        try {
            const result = await this.request(`${this.base}/${Number(delivery.dbId)}`);
            if (!result.data?.version) throw new Error('Delivery details are unavailable.');
            this.deliveries = this.deliveries.map((row) => row.dbId === delivery.dbId ? result.data : row);
            this.refresh();
            if (action === 'view') this.showDetails(result.data);
            else this.openManage(result.data);
        } catch (error) { this.announce(error.message); }
        finally { this.loading = false; }
    }

    /* Show validation errors without discarding a retry's request identity. */
    showError(message = '') {
        this.el('deliveryManageError').textContent = message;
        this.el('deliveryManageError').hidden = !message;
    }

    /* Start an edit using the server version and supported actions. */
    openManage(delivery) {
        const actions = this.actionsFor(delivery);
        if (!actions.length) { this.announce('No actions are available for this delivery.'); return; }
        this.selected = { id: delivery.id, dbId: delivery.dbId, version: delivery.version,
            fingerprint: null, requestKey: null };
        this.showError();
        this.el('deliveryManageTitle').textContent = `Manage ${delivery.id}`;
        this.el('deliveryManageSummary').textContent =
            `${delivery.orderId} · ${delivery.client} · Current status: ${delivery.status}`;
        this.el('deliveryAction').innerHTML = actions.map(([value, label]) =>
            `<option value="${value}">${escapeHtml(label)}</option>`).join('');
        this.renderActionFields();
        this.manageModal.show();
    }

    /* Render inputs for the selected server-backed action. */
    renderActionFields() {
        this.showError();

        const delivery = this.deliveries.find(
            (item) => item.id === this.selected?.id,
        );

        if (!delivery) return;

        const action = this.el('deliveryAction').value;
        let fields = '';

        if (action === 'schedule') {
            fields = `
                <div class="delivery-field">
                    <label for="deliverySchedule">
                        Scheduled Date and Time <span>*</span>
                    </label>
                    <input
                        type="datetime-local"
                        id="deliverySchedule"
                        value="${localInputValue(delivery.schedule)}"
                        required
                    >
                </div>
                <div class="delivery-field">
                    <label for="deliveryReason">Schedule Remarks <span>*</span></label>
                    <textarea id="deliveryReason" rows="3" maxlength="500" required></textarea>
                </div>
                <p class="delivery-help">
                    Times use Philippine time (Asia/Manila). Rescheduling a failed attempt
                    returns the linked order to Ready for Delivery.
                </p>
            `;
        }

        if (action === 'assign') {
            fields = `
                <div class="delivery-field">
                    <label for="deliveryPersonnel">Delivery Personnel <span>*</span></label>
                    <select id="deliveryPersonnel" required>
                        <option value="">Select personnel</option>
                        ${this.personnel
                            .filter((person) =>
                                true,
                            )
                            .map((person) => `
                                <option
                                    value="${escapeHtml(person.id)}"
                                    ${String(person.id) === String(delivery.personId) ? 'selected' : ''}
                                >
                                    ${escapeHtml(person.name)}
                                </option>
                            `).join('')}
                    </select>
                </div>
                <div class="delivery-field">
                    <label for="deliveryReason">Assignment Remarks <span>*</span></label>
                    <textarea id="deliveryReason" rows="3" maxlength="500" required></textarea>
                </div>
            `;
        }

        if (action === 'dispatch') {
            fields = `
                <p class="delivery-help">
                    Confirm the order has been handed to the assigned personnel.
                    This changes the delivery and linked order to
                    Out for Delivery. Dispatch does not deduct inventory.
                </p>
                <div class="delivery-field">
                    <label for="deliveryReason">Dispatch Notes</label>
                    <textarea id="deliveryReason" rows="3" maxlength="500"></textarea>
                </div>
            `;
        }

        if (action === 'deliver') {
            fields = `
                <div class="delivery-field">
                    <label for="deliveryRecipient">Recipient Name <span>*</span></label>
                    <input type="text" id="deliveryRecipient" maxlength="120" required>
                </div>
                <div class="delivery-field">
                    <label for="deliveryReason">Delivery Remarks <span>*</span></label>
                    <textarea id="deliveryReason" rows="3" maxlength="500" required></textarea>
                </div>
                <div class="delivery-field">
                    <label for="deliveryProof">Photo Proof <span>*</span></label>
                    <input
                        type="file"
                        id="deliveryProof"
                        accept="image/jpeg,image/png,image/webp"
                        required
                    >
                </div>
                <p class="delivery-help">
                    JPEG, PNG or WebP; maximum 5 MB.
                    Photo is uploaded to private storage. Confirm only after actual receipt.
                    Completion time is recorded automatically.
                </p>
            `;
        }

        if (action === 'fail') {
            fields = `
                <div class="delivery-field">
                    <label for="deliveryReason">Failure Reason <span>*</span></label>
                    <textarea id="deliveryReason" rows="3" maxlength="500" required></textarea>
                </div>
                <p class="delivery-help">A failed attempt keeps the order reservation.
                    Reschedule before dispatching again.</p>`;
        }

        if (action === 'cancel') {
            fields = `
                <div class="delivery-field">
                    <label for="deliveryReason">Cancellation Reason <span>*</span></label>
                    <textarea id="deliveryReason" rows="3" maxlength="500" required></textarea>
                </div>
                <p class="delivery-help">This cancels only the delivery record.
                    The linked order and its stock reservation remain unchanged.
                    Cancel the order separately in Order Management if needed.</p>`;
        }

        this.el('deliveryActionFields').innerHTML = fields;
    }

    /* Check that the selected upload can be opened as an image. */
    async validatePhoto(file) {
        const supported = ['image/jpeg', 'image/png', 'image/webp'];

        if (!file || !supported.includes(file.type)) {
            throw new Error('Select a JPEG, PNG or WebP proof image.');
        }

        if (file.size > 5 * 1024 * 1024) {
            throw new Error('Proof image must be 5 MB or smaller.');
        }

        const url = URL.createObjectURL(file);

        try {
            await new Promise((resolve, reject) => {
                const image = new Image();
                image.onload = resolve;
                image.onerror = () => reject(new Error('The selected image could not be opened.'));
                image.src = url;
            });

            return;
        } finally {
            URL.revokeObjectURL(url);
        }
    }

    /* Submit only validated inputs; stock and workflow changes belong to the server. */
    async saveAction() {
        if (this.busy || !this.selected) return;
        this.showError();
        if (!this.el('deliveryManageForm').reportValidity()) return;
        const selected = this.selected;
        const delivery = this.deliveries.find((row) => row.id === selected.id);
        const action = this.el('deliveryAction').value;
        if (!delivery || !this.actionsFor(delivery).some(([value]) => value === action)) {
            this.showError('This action is unavailable. Close and refresh the delivery.'); return;
        }
        const values = { action, version: selected.version,
            reason: this.el('deliveryReason')?.value.trim() || '' };
        const file = this.el('deliveryProof')?.files[0];
        this.setBusy(true);
        try {
            if (action !== 'dispatch' && !values.reason) throw new Error('Enter the required remarks.');
            if (action === 'schedule') values.scheduled_at = schedulePayload(this.el('deliverySchedule').value);
            if (action === 'assign') {
                values.assigned_to = this.el('deliveryPersonnel').value;
                if (!this.activePerson(values.assigned_to)) throw new Error('Select active delivery personnel.');
            }
            let proofHash = null;
            if (action === 'deliver') {
                values.recipient_name = this.el('deliveryRecipient').value.trim();
                if (!values.recipient_name) throw new Error('Enter the recipient name.');
                await this.validatePhoto(file);
                proofHash = Array.from(new Uint8Array(await crypto.subtle.digest('SHA-256', await file.arrayBuffer())))
                    .map((byte) => byte.toString(16).padStart(2, '0')).join('');
            }
            const fingerprint = JSON.stringify([values, proofHash]);
            if (selected.fingerprint !== fingerprint) {
                selected.fingerprint = fingerprint;
                selected.requestKey = crypto.randomUUID();
            }
            const body = new FormData();
            Object.entries(values).forEach(([key, value]) => body.append(key, value));
            body.append('request_key', selected.requestKey);
            if (action === 'deliver') body.append('proof', file);
            const result = await this.request(`${this.base}/${Number(selected.dbId)}/action`, { method: 'POST', body });
            if (!result.data?.version) throw new Error('Update response is incomplete. Keep this dialog open and retry.');
            this.deliveries = this.deliveries.map((row) => row.dbId === result.data.dbId ? result.data : row);
            this.refresh();
            this.setBusy(false);
            this.manageModal.hide();
            try {
                await this.load();
                this.announce(`${result.data.id}: ${result.data.status}. ${result.message || 'Update saved.'}`);
            } catch {
                this.announce('Update saved. Refresh the page to reload the complete list.');
            }
        } catch (error) {
            this.showError(error.message || 'Could not verify the update. Keep this dialog open and retry.');
            if (error.errors?.version) {
                this.setBusy(false);
                this.manageModal.hide();
                await this.load().catch(() => {});
                this.announce('Delivery changed. Reopen it before applying another update.');
            }
        } finally { this.setBusy(false); }
    }

    /* Prevent edits and dismissal while an upload or write is in progress. */
    setBusy(value) {
        this.busy = value;
        this.el('deliveryManageModal').querySelectorAll('input, select, textarea, button')
            .forEach((control) => { control.disabled = value; });
    }

    /* Render canonical history and the protected proof endpoint. */
    showDetails(delivery) {
        this.el('deliveryViewTitle').textContent = `Delivery Details — ${delivery.id}`;
        const history = (delivery.history || []).map((entry) => `
            <li><strong>${escapeHtml(entry.status)}</strong> — ${escapeHtml(entry.note)}
                <small>${escapeHtml(formatDateTime(entry.date))} · ${escapeHtml(entry.actor)}</small></li>`).join('');
        const detail = (label, value, full = false) => `
            <div class="${full ? 'delivery-full' : ''}"><dt>${escapeHtml(label)}</dt><dd>${escapeHtml(value || '—')}</dd></div>`;
        const proof = delivery.hasProof ? `
            <dl class="delivery-detail-grid">
                ${detail('Recipient', delivery.recipient)}
                ${detail('Completed At', delivery.completedAt ? formatDateTime(delivery.completedAt) : '')}
                ${detail('Delivery Remarks', delivery.remarks, true)}
            </dl>
            <a href="${escapeHtml(`${this.base}/${Number(delivery.dbId)}/proof`)}" target="_blank" rel="noopener">Open delivery proof</a>
            <img class="delivery-proof" src="${escapeHtml(`${this.base}/${Number(delivery.dbId)}/proof`)}"
                alt="Proof of delivery for ${escapeHtml(delivery.id)}">`
            : '<p class="delivery-help">No completed delivery proof recorded.</p>';
        this.el('deliveryViewContent').innerHTML = `
            <dl class="delivery-detail-grid">
                ${detail('Delivery ID', delivery.id)}${detail('Order ID', delivery.orderId)}
                ${detail('Client', delivery.client)}${detail('Delivery Address', delivery.address, true)}
                ${detail('Assigned Personnel', this.personName(delivery.personId))}
                ${detail('Scheduled Delivery', formatDateTime(delivery.schedule))}
                <div><dt>Delivery Status</dt><dd>${badge(delivery.status)}</dd></div>
                ${detail('Linked Order Status', delivery.orderStatus)}
                ${detail('Inventory Coordination', delivery.status === 'Delivered' && delivery.orderStatus === 'Completed'
                    ? 'Completed delivery; reserved stock issued and sale recorded.'
                    : 'This delivery has not issued stock.', true)}
            </dl>
            <h6 class="delivery-section-title">Proof of Delivery</h6>${proof}
            <h6 class="delivery-section-title">Delivery History</h6>
            <ol class="delivery-history">${history}</ol>`;
        this.viewModal.show();
    }
}

/* Initialize once when the Delivery page exists. */
function initializeDeliveryManagement() {
    const table = document.getElementById('deliveryTable');
    if (!table || DataTable.isDataTable(table)) return;
    new DeliveryManagementUI(table);
}
if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', initializeDeliveryManagement, { once: true });
} else { initializeDeliveryManagement(); }
```

## `resources/js/supplier/users.js`

```javascript
import DataTable from 'datatables.net-bs5';
import { Modal } from 'bootstrap';

const escapeHTML = (value) => String(value ?? '').replace(/[&<>"']/g, (c) => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#039;' }[c]));
const roles = ['Admin', 'Supplier/Staff', 'Delivery Personnel'];
const rolePermissions = {
    Admin: ['All Supplier modules, including Reports, User Management and Audit Trail.', 'Create accounts, change roles/email/status, and issue temporary passwords.'],
    'Supplier/Staff': ['Operational Flower, Inventory, Batch, Prediction, Client, Order, Delivery and Sales/Payments access.', 'No Reports, User Management or Audit Trail access. Account administration remains Admin-only.'],
    'Delivery Personnel': ['Assigned deliveries only: view details, record failed attempts and submit actual delivery proof.', 'No order, inventory, payment, fee or assignment changes.'],
};
const formatTime = (value) => new Intl.DateTimeFormat('en-PH', { timeZone: 'Asia/Manila', dateStyle: 'medium', timeStyle: 'short' }).format(new Date(value));
const initials = (name) => name.trim().split(/\s+/).slice(0, 2).map((part) => Array.from(part)[0]).join('').toUpperCase();
const badge = (status) => `<span class="user-badge ${status.toLowerCase()}">${escapeHTML(status)}</span>`;

/* Keep public account data only; credentials never enter table records. */
class UserModel {
    constructor() { this.users = []; }
    find(id) { return this.users.find((user) => user.id === id); }
}

class UserManagementUI {
    /* Preserve the approved table layout and initialize an empty database-backed list. */
    constructor() {
        const page = this.el('userPage');
        this.base = page.dataset.url.replace(/\/$/, '');
        this.csrf = page.dataset.csrf;
        this.model = new UserModel();
        this.pending = null;
        this.editingVersion = null;
        this.busy = false;
        this.loading = false;
        this.formModal = Modal.getOrCreateInstance(this.el('userFormModal'));
        this.actionModal = Modal.getOrCreateInstance(this.el('userActionModal'));
        this.viewModal = Modal.getOrCreateInstance(this.el('userViewModal'));
        this.table = new DataTable('#userTable', {
            data: this.model.users, autoWidth: false, pageLength: 5, order: [[0, 'asc']],
            layout: { topStart: null, topEnd: null, bottomStart: 'info', bottomEnd: 'paging' },
            language: { emptyTable: 'No users available.', zeroRecords: 'No matching users found.', info: 'Showing _START_ to _END_ of _TOTAL_ entries', infoEmpty: 'Showing 0 to 0 of 0 entries', infoFiltered: '(filtered from _MAX_ total entries)', paginate: { first: '«', previous: '‹', next: '›', last: '»' } },
            columns: [
                { data: 'name', width: '27%', render: (value, type, row) => type === 'display' ? `<div class="user-identity"><span class="user-avatar" aria-hidden="true">${escapeHTML(initials(value))}</span><div><strong>${escapeHTML(value)}</strong><small>${escapeHTML(row.id)}</small></div></div>` : `${value} ${row.id}` },
                { data: 'email', width: '25%', render: (value, type) => type === 'display' ? escapeHTML(value) : value },
                { data: 'role', width: '18%', render: (value, type) => type === 'display' ? escapeHTML(value) : value },
                { data: 'status', width: '12%', render: (value, type) => type === 'display' ? badge(value) : value },
                { data: null, width: '18%', orderable: false, searchable: false, render: (_value, type, row) => type === 'display' ? this.actions(row) : '' },
            ],
        });
        const table = this.el('userTable'); const wrapper = document.createElement('div'); wrapper.className = 'user-table-scroll'; wrapper.tabIndex = 0; wrapper.setAttribute('role', 'region'); wrapper.setAttribute('aria-label', 'Users table. Scroll horizontally for all columns.'); table.before(wrapper); wrapper.append(table);
        this.bindEvents();
        this.load().catch((error) => this.announce(error.message));
    }

    /* Find page and modal controls. */
    el(id) { return document.getElementById(id); }

    /* Announce a canonical result or recovery instruction. */
    announce(message) { this.el('userAnnouncement').textContent = message; }

    /* Use server permissions for protected Admin and current-account actions. */
    actions(user) {
        const button = (action, icon, label, disabled = false) => `<button type="button" class="user-icon-btn ${action}"
            data-user-action="${action}" data-user-id="${escapeHTML(user.id)}" title="${escapeHTML(label)}"
            aria-label="${escapeHTML(label)} ${escapeHTML(user.name)}" ${disabled ? 'disabled' : ''}>
            <i class="bi bi-${icon}" aria-hidden="true"></i></button>`;
        const active = user.status === 'Active';
        return `<div class="user-actions">${button('view', 'eye', 'View user')}
            ${button('edit', 'pencil-square', 'Edit user')}
            ${button('reset', 'key', user.isSelf ? 'Use Change Password for your own account' : 'Reset temporary password', !user.canReset)}
            ${button(active ? 'deactivate' : 'activate', active ? 'person-dash' : 'person-check',
                user.isSelf ? 'Your own account is protected' : user.isLastAdmin ? 'Last active Admin is protected'
                : active ? 'Deactivate' : 'Activate', !user.canChangeStatus)}</div>`;
    }

    /* Send explicit JSON inputs and surface Laravel validation/session responses. */
    async request(url, body = null) {
        const response = await fetch(url, {
            method: body === null ? 'GET' : 'POST', credentials: 'same-origin',
            headers: { Accept: 'application/json', 'X-Requested-With': 'XMLHttpRequest',
                'X-CSRF-TOKEN': this.csrf, ...(body === null ? {} : { 'Content-Type': 'application/json' }) },
            ...(body === null ? {} : { body: JSON.stringify(body) }),
        });
        const result = await response.json().catch(() => null);
        if (!response.ok || !result || response.redirected) {
            const messages = { 401: 'Please sign in again.', 403: 'Only an active Admin can manage accounts.',
                404: 'Operational account not found.', 419: 'Session expired. Refresh the page.' };
            const error = new Error(messages[response.status]
                || (result?.errors ? Object.values(result.errors).flat().join(' ') : null)
                || (response.status >= 500 ? 'Server error. Refresh the account before retrying.' : result?.message)
                || 'Could not verify the response. Refresh the list before retrying.');
            error.errors = result?.errors || {};
            throw error;
        }
        return result;
    }

    /* Load accounts without sample rows or stored passwords. */
    async load() {
        this.loading = true;
        try {
            const result = await this.request(`${this.base}/data`);
            if (!Array.isArray(result.data)) throw new Error('Unexpected account response. Refresh the page.');
            this.model.users = result.data;
            this.refresh();
        } finally { this.loading = false; }
    }

    /* Retain table search and pagination while replacing rows from the server. */
    refresh(message = '') {
        this.table.clear(); this.table.rows.add(this.model.users); this.table.draw(false);
        this.announce(message);
    }

    /* Bind forms and prevent dismissal or duplicate submissions during writes. */
    bindEvents() {
        this.el('addUserBtn').addEventListener('click', () => { if (!this.busy && !this.loading) this.openForm(); });
        this.el('userSearch').addEventListener('input', (event) => this.table.search(event.target.value).draw());
        this.el('userLength').addEventListener('change', (event) => this.table.page.len(Number(event.target.value)).draw());
        this.el('userTable').addEventListener('click', (event) => {
            const button = event.target.closest('[data-user-action]');
            if (!button || button.disabled || this.busy || this.loading) return;
            const user = this.model.find(button.dataset.userId);
            if (user) this.openRecord(user, button.dataset.userAction);
        });
        this.el('userForm').addEventListener('submit', (event) => { event.preventDefault(); this.save(); });
        this.el('userActionForm').addEventListener('submit', (event) => { event.preventDefault(); this.performAction(); });
        ['userFormModal', 'userActionModal'].forEach((id) => this.el(id).addEventListener('hide.bs.modal', (event) => {
            if (this.busy) event.preventDefault();
        }));
        this.el('userFormModal').addEventListener('hidden.bs.modal', () => {
            this.el('userForm').reset(); this.el('userPassword').value = ''; this.editingVersion = null;
        });
        this.el('userActionModal').addEventListener('hidden.bs.modal', () => {
            this.el('userResetPassword').value = ''; this.pending = null;
        });
    }

    /* Fetch a fresh version before displaying details, edit or confirmation. */
    async openRecord(user, action) {
        this.loading = true;
        try {
            const result = await this.request(`${this.base}/${user.dbId}`);
            if (!result.data?.version) throw new Error('Account details are unavailable.');
            const fresh = result.data;
            this.model.users = this.model.users.map((row) => row.dbId === fresh.dbId ? fresh : row);
            this.refresh();
            if (action === 'view') this.view(fresh);
            else if (action === 'edit') this.openForm(fresh);
            else this.openAction(fresh, action);
        } catch (error) { this.announce(error.message); }
        finally { this.loading = false; }
    }

    /* Display an error without copying credentials into account data. */
    error(id, message = '') { this.el(id).textContent = message; this.el(id).hidden = !message; }

    /* Populate the form; passwords are accepted only when creating an account. */
    openForm(user = null) {
        this.el('userForm').reset(); this.error('userFormError');
        this.editingVersion = user?.version || null;
        this.el('userFormTitle').textContent = user ? 'Edit User' : 'Add New User';
        this.el('userEditingId').value = user?.dbId || '';
        this.el('userCode').value = user?.id || 'Generated after saving';
        this.el('userName').value = user?.name || '';
        this.el('userEmail').value = user?.email || '';
        this.el('userRole').value = user?.roleValue || 'supplier';
        this.el('userStatus').value = user?.status || 'Active';
        this.el('userPasswordField').hidden = Boolean(user);
        this.el('userPassword').disabled = Boolean(user);
        this.el('userPassword').required = !user;
        this.el('userPassword').value = '';
        this.formModal.show();
    }

    /* Check bcrypt's byte limit before sending a temporary password. */
    passwordValid(value) {
        if (Array.from(value).length < 12 || new TextEncoder().encode(value).length > 72 || value.includes('\0')) {
            throw new Error('Use at least 12 characters, at most 72 UTF-8 bytes, with no null characters.');
        }
    }

    /* Create or update using an allowlist; never merge credentials into the table model. */
    async save() {
        if (this.busy) return;
        this.error('userFormError');
        if (!this.el('userForm').reportValidity()) return;
        const id = this.el('userEditingId').value;
        const values = { name: this.el('userName').value.trim(), email: this.el('userEmail').value.trim().toLowerCase(),
            role: this.el('userRole').value, status: this.el('userStatus').value };
        try {
            if (id) values.version = this.editingVersion;
            else { values.password = this.el('userPassword').value; this.passwordValid(values.password); }
            this.setBusy(true);
            const result = await this.request(id ? `${this.base}/${Number(id)}/update` : this.base, values);
            this.el('userPassword').value = '';
            this.setBusy(false); this.formModal.hide();
            await this.reloadAfterWrite(result);
        } catch (error) { this.error('userFormError', error.message || 'Could not verify the update. Refresh before retrying.'); }
        finally { delete values.password; this.setBusy(false); }
    }

    /* Confirm an explicit status or a separate temporary-password reset. */
    openAction(user, action) {
        if ((action === 'reset' && !user.canReset) || (action !== 'reset' && !user.canChangeStatus)) return;
        const reset = action === 'reset';
        this.pending = { dbId: user.dbId, action, version: user.version };
        this.error('userActionError');
        this.el('userActionTitle').textContent = reset ? 'Reset Temporary Password' : action === 'activate' ? 'Activate User' : 'Deactivate User';
        this.el('userActionMessage').textContent = reset
            ? `Issue a new temporary password for ${user.name}? Previous sessions will be revoked and a password change will be required.`
            : `${action === 'activate' ? 'Activate' : 'Deactivate'} ${user.name}? Deactivation blocks login and revokes existing access.`;
        this.el('userResetField').hidden = !reset;
        this.el('userResetPassword').required = reset;
        this.el('userResetPassword').disabled = !reset;
        this.el('userResetPassword').value = '';
        this.actionModal.show();
    }

    /* Apply status/reset with the selected version; there is no unsafe status toggle. */
    async performAction() {
        if (this.busy || !this.pending) return;
        this.error('userActionError');
        if (!this.el('userActionForm').reportValidity()) return;
        const pending = { ...this.pending };
        const values = { version: pending.version };
        try {
            const reset = pending.action === 'reset';
            if (reset) { values.password = this.el('userResetPassword').value; this.passwordValid(values.password); }
            else values.status = pending.action === 'activate' ? 'Active' : 'Inactive';
            this.setBusy(true);
            const result = await this.request(`${this.base}/${pending.dbId}/${reset ? 'reset-password' : 'status'}`, values);
            this.el('userResetPassword').value = '';
            this.setBusy(false); this.actionModal.hide();
            await this.reloadAfterWrite(result);
        } catch (error) { this.error('userActionError', error.message || 'Could not verify the update. Refresh before retrying.'); }
        finally { delete values.password; this.setBusy(false); }
    }

    /* Preserve the returned saved row even if the subsequent list refresh fails. */
    async reloadAfterWrite(result) {
        if (!result.data?.version) throw new Error('Update response is incomplete. Refresh the page to verify.');
        const index = this.model.users.findIndex((row) => row.dbId === result.data.dbId);
        if (index < 0) this.model.users.push(result.data); else this.model.users[index] = result.data;
        this.refresh(result.message || 'Account saved.');
        try { await this.load(); this.announce(result.message || 'Account saved.'); }
        catch { this.announce('Account saved. Refresh the page to reload the complete list.'); }
    }

    /* Lock controls while writing; restore password-field states afterward. */
    setBusy(value) {
        this.busy = value;
        ['userFormModal', 'userActionModal'].forEach((id) => this.el(id)
            .querySelectorAll('input, select, button').forEach((control) => { control.disabled = value; }));
        this.el('addUserBtn').disabled = value;
        if (!value) {
            this.el('userPassword').disabled = Boolean(this.el('userEditingId').value);
            this.el('userResetPassword').disabled = this.pending?.action !== 'reset';
        }
    }

    /* Display saved account/security facts without fabricated session history or secrets. */
    view(user) {
        this.el('userViewTitle').textContent = `User Details — ${user.id}`;
        const fields = [['Name', user.name], ['Email', user.email], ['Role', user.role], ['Status', user.status],
            ['Registered', user.created ? formatTime(user.created) : '—'],
            ['Password Change Required', user.mustChangePassword ? 'Yes — at next login' : 'No']];
        this.el('userViewContent').innerHTML = `<dl class="user-detail-grid">${fields.map(([label, value]) =>
            `<div><dt>${escapeHTML(label)}</dt><dd>${escapeHTML(value)}</dd></div>`).join('')}</dl>
            <h6 class="user-section-title">Fixed Role Permissions</h6>
            <ul class="user-permissions">${(rolePermissions[user.role] || []).map((value) => `<li>${escapeHTML(value)}</li>`).join('')}</ul>
            <p class="user-help">Passwords are never displayed. Account administration is restricted to active Admin accounts.</p>`;
        this.viewModal.show();
    }
}

/* Initialize only on the User Management page. */
function initializeUsers() {
    if (document.getElementById('userPage') && !DataTable.isDataTable('#userTable')) new UserManagementUI();
}
if (document.readyState === 'loading') document.addEventListener('DOMContentLoaded', initializeUsers, { once: true });
else initializeUsers();
```

## `resources/views/layouts/delivery.blade.php`

```blade
<!DOCTYPE html>
<html lang="{{ str_replace('_', '-', app()->getLocale()) }}">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>@yield('title', 'Assigned Deliveries') - FlowerTrack IMS</title>
    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
<body>
    <div class="container-fluid px-3 px-lg-4 py-4">
        <header class="d-flex flex-wrap justify-content-between align-items-center gap-3 mb-4">
            <div class="d-flex align-items-center gap-3">
                <img src="{{ asset('images/gemmas-logo.png') }}" alt="Gemma's Flower Shop" width="56" height="56">
                <div><strong>FlowerTrack IMS</strong><div class="text-secondary">Assigned Deliveries</div></div>
            </div>
            <div class="d-flex flex-wrap align-items-center gap-3">
                <span>{{ auth()->user()->name }}</span>
                <a href="{{ route('password.change') }}">Change Password</a>
                <form method="POST" action="{{ route('logout') }}">
                    @csrf
                    <button type="submit" class="btn btn-outline-success">Sign Out</button>
                </form>
            </div>
        </header>
        <main>@yield('content')</main>
    </div>
</body>
</html>
```

## `resources/views/supplier/deliveries.blade.php`

```blade
@extends(auth()->user()->isDeliveryPersonnel() ? 'layouts.delivery' : 'layouts.supplier')

@section('title', 'Delivery Management')

@section('content')
    <div class="delivery-page">
        <div class="delivery-card">
            <div class="delivery-header">
                <div>
                    <h1>Delivery Management</h1>
                    <p>{{ auth()->user()->isDeliveryPersonnel() ? 'View your assigned deliveries and record actual delivery results.' : 'Manage delivery assignments, schedules, and fulfillment updates.' }}</p>
                </div>

            </div>

            <div class="delivery-notice">
                <i class="bi bi-info-circle" aria-hidden="true"></i>
                <span>
                    Deliveries come from orders marked Ready for Delivery.
                    Stock is deducted only after actual delivery is confirmed.
                </span>
            </div>

            <div class="delivery-toolbar">
                <div class="delivery-length">
                    <label for="deliveryPageLength">Show</label>
                    <select id="deliveryPageLength">
                        <option value="5">5</option>
                        <option value="10">10</option>
                        <option value="25">25</option>
                        <option value="50">50</option>
                    </select>
                    <span>entries</span>
                </div>

                <div class="delivery-search">
                    <label class="visually-hidden" for="deliverySearch">
                        Search deliveries
                    </label>
                    <i class="bi bi-search" aria-hidden="true"></i>
                    <input
                        type="search"
                        id="deliverySearch"
                        placeholder="Search delivery, order or client..."
                        autocomplete="off"
                    >
                </div>
            </div>

            <div class="delivery-table-container">
                <table id="deliveryTable"
                    data-url="{{ route(auth()->user()->isDeliveryPersonnel() ? 'delivery.deliveries' : 'supplier.deliveries') }}"
                    data-role="{{ auth()->user()->role }}"
                    data-csrf="{{ csrf_token() }}">
                    <thead>
                        <tr>
                            <th scope="col">Delivery / Order</th>
                            <th scope="col">Client / Address</th>
                            <th scope="col">Personnel</th>
                            <th scope="col">Schedule</th>
                            <th scope="col">Status</th>
                            <th scope="col">Actions</th>
                        </tr>
                    </thead>
                    <tbody></tbody>
                </table>
            </div>

            <p id="deliveryAnnouncement" class="delivery-announcement" role="status"></p>
        </div>
    </div>

    {{-- Manage Delivery --}}
    <div
        class="modal fade"
        id="deliveryManageModal"
        tabindex="-1"
        aria-labelledby="deliveryManageTitle"
        aria-hidden="true"
    >
        <div class="modal-dialog modal-dialog-centered modal-lg">
            <div class="modal-content delivery-modal">
                <div class="modal-header">
                    <div>
                        <span class="delivery-eyebrow">DELIVERY OPERATIONS</span>
                        <h5 class="modal-title" id="deliveryManageTitle">Manage Delivery</h5>
                    </div>
                    <button
                        type="button"
                        class="btn-close"
                        data-bs-dismiss="modal"
                        aria-label="Close"
                    ></button>
                </div>

                <form id="deliveryManageForm">
                    <div class="modal-body">
                        <div id="deliveryManageError" class="delivery-error" role="alert" hidden></div>

                        <p id="deliveryManageSummary" class="delivery-help"></p>

                        <div class="delivery-field">
                            <label for="deliveryAction">Action</label>
                            <select id="deliveryAction" required></select>
                        </div>

                        <div id="deliveryActionFields" class="delivery-action-fields"></div>
                    </div>

                    <div class="modal-footer">
                        <button
                            type="button"
                            class="delivery-secondary-btn"
                            data-bs-dismiss="modal"
                        >
                            Cancel
                        </button>
                        <button type="submit" class="delivery-primary-btn">
                            <i class="bi bi-check-lg" aria-hidden="true"></i>
                            Save Update
                        </button>
                    </div>
                </form>
            </div>
        </div>
    </div>

    {{-- Delivery Details --}}
    <div
        class="modal fade"
        id="deliveryViewModal"
        tabindex="-1"
        aria-labelledby="deliveryViewTitle"
        aria-hidden="true"
    >
        <div class="modal-dialog modal-dialog-centered modal-xl">
            <div class="modal-content delivery-modal">
                <div class="modal-header">
                    <div>
                        <span class="delivery-eyebrow">DELIVERY INFORMATION</span>
                        <h5 class="modal-title" id="deliveryViewTitle">Delivery Details</h5>
                    </div>
                    <button
                        type="button"
                        class="btn-close"
                        data-bs-dismiss="modal"
                        aria-label="Close"
                    ></button>
                </div>

                <div class="modal-body" id="deliveryViewContent"></div>

                <div class="modal-footer">
                    <button
                        type="button"
                        class="delivery-secondary-btn"
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

## `resources/views/supplier/users.blade.php`

```blade
@extends('layouts.supplier')
@section('title', 'User Management')
@section('content')
    <div class="user-page" id="userPage" data-url="{{ route('supplier.users') }}" data-csrf="{{ csrf_token() }}">
        <div class="user-card">
            <div class="user-header">
                <div>
                    <h1>User Management</h1>
                    <p>Manage staff accounts, roles, and account access.</p>
                </div>
                <button type="button" class="user-primary-btn" id="addUserBtn"><i class="bi bi-plus-lg"
                        aria-hidden="true"></i> Add New User</button>
            </div>
            <p class="user-notice">Manage Admin, Supplier/Staff and Delivery Personnel accounts. Client accounts remain in Client Management.
                <a href="{{ route('password.change') }}">Change your own password</a>.</p>
            <div class="user-toolbar">
                <div class="user-length"><label for="userLength">Show</label><select id="userLength">
                        <option>5</option>
                        <option>10</option>
                        <option>25</option>
                        <option>50</option>
                    </select><span>entries</span></div>
                <div class="user-search"><label class="visually-hidden" for="userSearch">Search users</label><i
                        class="bi bi-search" aria-hidden="true"></i><input id="userSearch" type="search"
                        placeholder="Search user ID, name or email..." autocomplete="off"></div>
            </div>
            <div class="user-table-container">
                <table id="userTable">
                    <thead>
                        <tr>
                            <th scope="col">User Details</th>
                            <th scope="col">Email</th>
                            <th scope="col">Role</th>
                            <th scope="col">Status</th>
                            <th scope="col">Actions</th>
                        </tr>
                    </thead>
                    <tbody></tbody>
                </table>
            </div>
            <p class="user-announcement" id="userAnnouncement" role="status"></p>
        </div>
    </div>

    <div class="modal fade" id="userFormModal" tabindex="-1" aria-labelledby="userFormTitle" aria-hidden="true">
        <div class="modal-dialog modal-dialog-centered modal-lg">
            <div class="modal-content user-modal">
                <div class="modal-header">
                    <div><span class="user-eyebrow">USER ACCOUNT</span>
                        <h5 class="modal-title" id="userFormTitle">Add New User</h5>
                    </div><button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close"></button>
                </div>
                <form id="userForm">
                    <div class="modal-body">
                        <div id="userFormError" class="user-error" role="alert" hidden></div><input type="hidden"
                            id="userEditingId">
                        <div class="user-form-grid">
                            <div class="user-field"><label for="userCode">User ID</label><input id="userCode" readonly>
                            </div>
                            <div class="user-field"><label for="userName">Full Name *</label><input id="userName"
                                    maxlength="120" autocomplete="name" required></div>
                            <div class="user-field"><label for="userEmail">Email Address *</label><input id="userEmail"
                                    type="email" maxlength="254" autocomplete="email" required></div>
                            <div class="user-field"><label for="userRole">Role *</label><select id="userRole" required>
                                    <option value="admin">Admin</option>
                                    <option value="supplier">Supplier/Staff</option>
                                    <option value="delivery">Delivery Personnel</option>
                                </select></div>
                            <div class="user-field"><label for="userStatus">Status</label><select id="userStatus">
                                    <option>Active</option>
                                    <option>Inactive</option>
                                </select></div>
                            <div class="user-field" id="userPasswordField"><label for="userPassword">Temporary Password
                                    *</label><input id="userPassword" type="password" minlength="12" maxlength="72"
                                    autocomplete="new-password"><small>At least 12 characters, maximum 72 UTF-8 bytes. Share privately; the user must change it after signing in.</small></div>
                        </div>
                        <p class="user-help">Roles use fixed permissions. New accounts and Admin password resets require a
                            password change at the next login.</p>
                    </div>
                    <div class="modal-footer"><button type="button" class="user-secondary-btn"
                            data-bs-dismiss="modal">Cancel</button><button type="submit" class="user-primary-btn">Save
                            User</button></div>
                </form>
            </div>
        </div>
    </div>

    <div class="modal fade" id="userActionModal" tabindex="-1" aria-labelledby="userActionTitle" aria-hidden="true">
        <div class="modal-dialog modal-dialog-centered">
            <div class="modal-content user-modal">
                <div class="modal-header">
                    <h5 class="modal-title" id="userActionTitle">Update Account</h5><button type="button" class="btn-close"
                        data-bs-dismiss="modal" aria-label="Close"></button>
                </div>
                <form id="userActionForm">
                    <div class="modal-body">
                        <div id="userActionError" class="user-error" role="alert" hidden></div>
                        <p id="userActionMessage" class="user-help"></p>
                        <div class="user-field" id="userResetField" hidden><label for="userResetPassword">New Temporary
                                Password *</label><input id="userResetPassword" type="password" minlength="12"
                                maxlength="72" autocomplete="new-password"><small>At least 12 characters, maximum 72 UTF-8 bytes. Previous sessions will be revoked.</small></div>
                    </div>
                    <div class="modal-footer"><button type="button" class="user-secondary-btn"
                            data-bs-dismiss="modal">Cancel</button><button type="submit"
                            class="user-primary-btn">Confirm</button></div>
                </form>
            </div>
        </div>
    </div>

    <div class="modal fade" id="userViewModal" tabindex="-1" aria-labelledby="userViewTitle" aria-hidden="true">
        <div class="modal-dialog modal-dialog-centered modal-lg">
            <div class="modal-content user-modal">
                <div class="modal-header">
                    <h5 class="modal-title" id="userViewTitle">User Details</h5><button type="button" class="btn-close"
                        data-bs-dismiss="modal" aria-label="Close"></button>
                </div>
                <div class="modal-body" id="userViewContent"></div>
                <div class="modal-footer"><button type="button" class="user-secondary-btn"
                        data-bs-dismiss="modal">Close</button></div>
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

                Route::view('/sales-payments', 'supplier.sales-payments')
                    ->name('sales-payments');

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

## `tests/Feature/DeliveryBackendTest.php`

```php
<?php

namespace Tests\Feature;

use App\Http\Controllers\Supplier\ClientController;
use App\Http\Requests\StoreOrderRequest;
use App\Models\Client;
use App\Models\Flower;
use App\Models\FlowerPrice;
use App\Models\InventoryBatch;
use App\Models\InventoryMovement;
use App\Models\InventoryOperation;
use App\Models\InventoryReservation;
use App\Models\Order;
use App\Models\OrderPayment;
use App\Models\SalesRecord;
use App\Models\User;
use App\Services\ClientService;
use App\Services\InventoryService;
use App\Services\OrderService;
use App\Support\Money;
use Illuminate\Auth\Access\AuthorizationException;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Schema;
use Illuminate\Support\Facades\Validator;
use Illuminate\Support\Str;
use Illuminate\Validation\ValidationException;
use Symfony\Component\HttpKernel\Exception\HttpException;
use Tests\TestCase;
use App\Models\Delivery;
use App\Models\DeliveryHistory;
use App\Services\DeliveryService;
use App\Support\AccountSession;
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;

class DeliveryBackendTest extends TestCase
{
    private OrderService $orders;
    private InventoryService $inventory;
    private User $admin;
    private User $supplier;
    private Client $client;
    private Flower $stem;
    private Flower $bouquet;
    private bool $testConnectionConfigured = false;

    protected function setUp(): void
    {
        parent::setUp();
        if (! extension_loaded('pdo_sqlite')) {
            $this->markTestSkipped('Isolated order tests require pdo_sqlite.');
        }
        config([
            'database.connections.order_backend_test' => [
                'driver' => 'sqlite', 'database' => ':memory:', 'prefix' => '',
                'foreign_key_constraints' => true,
            ],
            'database.default' => 'order_backend_test',
            'orders.delivery_fee' => null,
        ]);
        $this->testConnectionConfigured = true;
        DB::purge('order_backend_test');
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
        foreach (['create_clients_table', 'create_inventory_tables', 'create_order_tables', 'create_delivery_tables'] as $name) {
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
        $this->receive($this->stem, 20);
        $this->receive($this->bouquet, 5);
        Storage::fake('local');
        config(['orders.delivery_fee' => '150.00']);
    }


    /** Dispatch changes workflow only; physical stock and holds remain unchanged. */
    public function test_dispatch_preserves_stock_and_reservations(): void
    {
        [$order, $delivery] = $this->dispatched();
        $this->assertSame('Out for Delivery', $delivery->status);
        $this->assertSame('Out for Delivery', $order->fresh()->status);
        $this->assertSame(20, $this->remaining($this->stem));
        $this->assertSame(5, $this->remaining($this->bouquet));
        $this->assertSame(2, InventoryReservation::where('status', 'Active')->count());
        $this->assertSame(0, SalesRecord::count());
    }

    /** Actual receipt completes both records, consumes mixed stock and replays once. */
    public function test_actual_delivery_completes_once_with_private_proof_and_no_payment_gate(): void
    {
        [$order, $delivery] = $this->dispatched();
        $payload = $this->deliveryInput($delivery, [
            'action' => 'deliver', 'recipient_name' => 'Actual Recipient', 'proof' => $this->photo(),
        ]);
        $service = app(DeliveryService::class);
        $result = $service->action($delivery, $payload, $this->supplier);
        $movements = InventoryMovement::count();
        $service->action($delivery, $payload, $this->supplier);

        $this->assertSame('Delivered', $result->status);
        $this->assertSame('Actual Recipient', $result->recipient_name);
        $this->assertSame('Completed', $order->fresh()->status);
        $this->assertTrue($order->fresh()->stocked_out);
        $this->assertSame(16, $this->remaining($this->stem));
        $this->assertSame(3, $this->remaining($this->bouquet));
        $this->assertSame(2, InventoryReservation::where('status', 'Consumed')->count());
        $this->assertSame(0, InventoryReservation::where('status', 'Active')->count());
        $this->assertSame(1, SalesRecord::where('order_id', $order->id)->count());
        $this->assertSame((int) $order->total_cents, (int) SalesRecord::first()->total_cents);
        $this->assertSame($movements, InventoryMovement::count());
        $this->assertSame(1, DeliveryHistory::where('action', 'deliver')->count());
        $this->assertSame(0, OrderPayment::count());
        $this->assertNotNull($result->delivered_at);
        Storage::disk('local')->assertExists($result->proof_path);
        $this->assertCount(1, Storage::disk('local')->allFiles('delivery-proofs'));
        $this->assertArrayNotHasKey('proof_path', $service->row($result, $this->supplier));

        $payload['recipient_name'] = 'Changed Recipient';
        $this->rejected(fn () => $service->action($delivery, $payload, $this->supplier), 'request_key');
        $this->assertSame(1, SalesRecord::count());
    }

    /** A later bouquet shortage rolls back earlier stem fulfillment and the upload. */
    public function test_shortage_rolls_back_every_completion_effect_and_removes_unused_proof(): void
    {
        [$order, $delivery] = $this->dispatched();
        InventoryBatch::where('flower_id', $this->bouquet->id)->update(['remaining_quantity' => 1]);
        $movements = InventoryMovement::count();
        $operations = InventoryOperation::count();
        $history = DeliveryHistory::count();
        $payload = $this->deliveryInput($delivery, [
            'action' => 'deliver', 'recipient_name' => 'Recipient', 'proof' => $this->photo(),
        ]);
        $this->rejected(fn () => app(DeliveryService::class)->action($delivery, $payload, $this->supplier));
        $this->assertSame(20, $this->remaining($this->stem));
        $this->assertSame(1, $this->remaining($this->bouquet));
        $this->assertSame($movements, InventoryMovement::count());
        $this->assertSame($operations, InventoryOperation::count());
        $this->assertSame($history, DeliveryHistory::count());
        $this->assertSame(0, SalesRecord::count());
        $this->assertSame(2, InventoryReservation::where('status', 'Active')->count());
        $this->assertSame('Out for Delivery', $delivery->fresh()->status);
        $this->assertSame('Out for Delivery', $order->fresh()->status);
        $this->assertFalse($order->fresh()->stocked_out);
        $this->assertNull($delivery->fresh()->proof_path);
        $this->assertCount(0, Storage::disk('local')->allFiles('delivery-proofs'));
    }

    /** Failed attempts retain holds; rescheduling is mandatory before another dispatch. */
    public function test_failed_delivery_requires_rescheduling_and_does_not_issue_stock(): void
    {
        [$order, $delivery] = $this->dispatched();
        $delivery = $this->deliveryAction($delivery, ['action' => 'fail']);
        $this->assertSame('Failed', $delivery->status);
        $this->assertSame('Out for Delivery', $order->fresh()->status);
        $this->rejected(fn () => $this->deliveryAction($delivery, ['action' => 'dispatch']));
        $delivery = $this->deliveryAction($delivery, [
            'action' => 'schedule', 'scheduled_at' => now()->addDays(2)->toIso8601String(),
        ]);
        $this->assertSame('Rescheduled', $delivery->status);
        $this->assertSame('Ready for Delivery', $order->fresh()->status);
        $delivery = $this->deliveryAction($delivery, ['action' => 'dispatch']);
        $this->assertSame('Out for Delivery', $delivery->status);
        $this->assertSame(20, $this->remaining($this->stem));
        $this->assertSame(2, InventoryReservation::where('status', 'Active')->count());
        $this->assertSame(0, SalesRecord::count());
    }

    /** Completion requires a real image and an actual dispatched delivery. */
    public function test_completion_rejects_missing_or_invalid_photo_and_early_completion(): void
    {
        [, $delivery] = $this->ready();
        $this->rejected(fn () => $this->deliveryAction($delivery, [
            'action' => 'deliver', 'recipient_name' => 'Recipient', 'proof' => $this->photo(),
        ]));
        [, $delivery] = $this->dispatched();
        $this->rejected(fn () => $this->deliveryAction($delivery, [
            'action' => 'deliver', 'recipient_name' => 'Recipient',
        ]), 'proof');
        $this->rejected(fn () => $this->deliveryAction($delivery, [
            'action' => 'deliver', 'recipient_name' => 'Recipient',
            'proof' => UploadedFile::fake()->createWithContent('fake.jpg', 'not an image'),
        ]), 'proof');
        $this->assertSame(0, SalesRecord::count());
        $this->assertCount(0, Storage::disk('local')->allFiles('delivery-proofs'));
    }

    /** Reusing a request key and overwriting a stale schedule cannot change valid state. */
    public function test_schedule_replay_and_stale_changes_are_protected(): void
    {
        [, $delivery] = $this->ready();
        $payload = $this->deliveryInput($delivery, [
            'action' => 'schedule', 'scheduled_at' => now()->addDay()->toIso8601String(),
        ]);
        $service = app(DeliveryService::class);
        $service->action($delivery, $payload, $this->supplier);
        $service->action($delivery, $payload, $this->supplier);
        $this->assertSame(1, DeliveryHistory::where('action', 'schedule')->count());
        $stale = [...$payload, 'request_key' => (string) Str::uuid(), 'scheduled_at' => now()->addDays(3)->toIso8601String()];
        $this->rejected(fn () => $service->action($delivery, $stale, $this->supplier), 'version');
    }

    /** Cancelling only a delivery preserves its order, stock and reservation. */
    public function test_cancelling_a_ready_delivery_preserves_order_and_holds_and_replays_once(): void
    {
        [$order, $delivery] = $this->ready();
        $movements = InventoryMovement::count();
        $payload = $this->deliveryInput($delivery, [
            'action' => 'cancel', 'reason' => 'Delivery arrangement cancelled.',
        ]);
        $service = app(DeliveryService::class);
        $result = $service->action($delivery, $payload, $this->supplier);
        $service->action($delivery, $payload, $this->supplier);

        $this->assertSame('Cancelled', $result->status);
        $this->assertSame('Ready for Delivery', $order->fresh()->status);
        $this->assertFalse($order->fresh()->stocked_out);
        $this->assertSame(2, InventoryReservation::where('status', 'Active')->count());
        $this->assertSame(20, $this->remaining($this->stem));
        $this->assertSame(5, $this->remaining($this->bouquet));
        $this->assertSame($movements, InventoryMovement::count());
        $this->assertSame(0, SalesRecord::count());
        $this->assertSame(1, DeliveryHistory::where('action', 'cancel')->count());
        $this->rejected(fn () => $this->deliveryAction($result, [
            'action' => 'schedule', 'scheduled_at' => now()->addDay()->toIso8601String(),
        ]), 'action');
        $payload['reason'] = 'Changed reason';
        $this->rejected(fn () => $service->action($delivery, $payload, $this->supplier), 'request_key');
    }

    /** A dispatched delivery must first record a failed attempt before cancellation. */
    public function test_cancellation_requires_reason_and_cannot_cancel_an_active_dispatch(): void
    {
        [$order, $delivery] = $this->dispatched();
        $this->rejected(fn () => $this->deliveryAction($delivery, [
            'action' => 'cancel', 'reason' => 'Cancelled arrangement.',
        ]), 'action');
        $delivery = $this->deliveryAction($delivery, ['action' => 'fail']);
        $this->rejected(fn () => $this->deliveryAction($delivery, [
            'action' => 'cancel', 'reason' => '   ',
        ]), 'reason');
        $delivery = $this->deliveryAction($delivery, [
            'action' => 'cancel', 'reason' => 'No further delivery attempt.',
        ]);
        $this->assertSame('Cancelled', $delivery->status);
        $this->assertSame('Out for Delivery', $order->fresh()->status);
        $this->assertSame(2, InventoryReservation::where('status', 'Active')->count());
        $this->assertSame(20, $this->remaining($this->stem));
        $this->assertSame(0, SalesRecord::count());
    }

    /** Personnel lists, details and mutations are scoped to the locked assignment. */
    public function test_personnel_can_only_read_their_own_assigned_deliveries(): void
    {
        [, $delivery] = $this->dispatched();
        $person = User::findOrFail($delivery->assigned_to);
        $other = $this->user('other-courier', User::ROLE_DELIVERY);
        $service = app(DeliveryService::class);
        $this->assertCount(1, $service->data($person)['data']);
        $this->assertCount(0, $service->data($other)['data']);
        $this->assertSame([], $service->data($person)['personnel']);
        $this->assertSame($delivery->code, $service->row($delivery, $person)['id']);
        try {
            $service->row($delivery, $other);
            $this->fail('Another courier must not read this delivery.');
        } catch (HttpException $exception) {
            $this->assertSame(403, $exception->getStatusCode());
        }
        foreach (['schedule', 'assign', 'dispatch', 'cancel'] as $action) {
            try {
                $service->action($delivery, $this->deliveryInput($delivery, ['action' => $action]), $person);
                $this->fail('Personnel cannot manage assignment or dispatch.');
            } catch (HttpException $exception) {
                $this->assertSame(403, $exception->getStatusCode());
            }
        }
    }

    /** Actual assigned personnel may complete delivery; other personnel cannot access proof. */
    public function test_personnel_completion_and_private_proof_routes_enforce_ownership(): void
    {
        [$order, $delivery] = $this->dispatched();
        $person = User::findOrFail($delivery->assigned_to);
        $other = $this->user('other-courier', User::ROLE_DELIVERY);
        $service = app(DeliveryService::class);
        $payload = $this->deliveryInput($delivery, [
            'action' => 'deliver', 'recipient_name' => 'Recipient', 'proof' => $this->photo(),
        ]);
        try {
            $service->action($delivery, $payload, $other);
            $this->fail('A different courier must not complete this delivery.');
        } catch (HttpException $exception) {
            $this->assertSame(403, $exception->getStatusCode());
        }
        $this->assertSame(0, SalesRecord::count());
        $result = $service->action($delivery, $payload, $person);
        $this->assertSame('Delivered', $result->status);
        $this->assertSame('Completed', $order->fresh()->status);
        $this->assertSame(1, SalesRecord::count());
        $this->actingAs($person)->withSession([AccountSession::KEY => AccountSession::signature($person)])
            ->get('/delivery/deliveries/'.$delivery->id.'/proof')->assertOk();
        $this->actingAs($other)->withSession([AccountSession::KEY => AccountSession::signature($other)])
            ->getJson('/delivery/deliveries/'.$delivery->id.'/proof')->assertForbidden();
        $this->getJson('/supplier/deliveries/data')->assertForbidden();
    }

    /** Personnel may record a failure without consuming stock or releasing holds. */
    public function test_assigned_personnel_can_record_failed_attempt_without_stock_deduction(): void
    {
        [$order, $delivery] = $this->dispatched();
        $person = User::findOrFail($delivery->assigned_to);
        $result = app(DeliveryService::class)->action($delivery, $this->deliveryInput($delivery, [
            'action' => 'fail', 'reason' => 'Recipient unavailable.',
        ]), $person);
        $this->assertSame('Failed', $result->status);
        $this->assertSame('Out for Delivery', $order->fresh()->status);
        $this->assertSame(20, $this->remaining($this->stem));
        $this->assertSame(2, InventoryReservation::where('status', 'Active')->count());
        $this->assertSame(0, SalesRecord::count());
    }

    private function ready(): array
    {
        $order = $this->orders->store($this->input([
            'method' => 'Delivery', 'address' => 'Actual delivery address',
            'items' => [
                ['flower_id' => $this->stem->id, 'quantity' => 4],
                ['flower_id' => $this->bouquet->id, 'quantity' => 2],
            ],
        ]), $this->supplier);
        $this->action($order, 'Confirmed', $this->admin);
        $this->action($order, 'Processing', $this->supplier);
        $this->action($order, 'Ready for Delivery', $this->supplier);
        $delivery = app(DeliveryService::class)->createForOrder($order, $this->supplier);
        $this->assertSame(1, Delivery::where('order_id', $order->id)->count());

        return [$order->fresh(), $delivery];
    }

    private function dispatched(): array
    {
        [$order, $delivery] = $this->ready();
        $person = User::query()->where('role', User::ROLE_DELIVERY)->first()
            ?? $this->user('delivery', User::ROLE_DELIVERY);
        $delivery = $this->deliveryAction($delivery, [
            'action' => 'schedule', 'scheduled_at' => now()->addDay()->toIso8601String(),
        ]);
        $delivery = $this->deliveryAction($delivery, [
            'action' => 'assign', 'assigned_to' => $person->id,
        ]);
        $delivery = $this->deliveryAction($delivery, ['action' => 'dispatch']);

        return [$order->fresh(), $delivery];
    }

    private function deliveryInput(Delivery $delivery, array $overrides): array
    {
        return array_replace([
            'request_key' => (string) Str::uuid(),
            'version' => app(DeliveryService::class)->row($delivery->fresh(), $this->supplier)['version'],
            'reason' => 'Actual delivery operation test.',
        ], $overrides);
    }

    private function deliveryAction(Delivery $delivery, array $overrides): Delivery
    {
        return app(DeliveryService::class)->action(
            $delivery, $this->deliveryInput($delivery, $overrides), $this->supplier
        );
    }

    private function photo(): UploadedFile
    {
        return UploadedFile::fake()->createWithContent('proof.png', base64_decode(
            'iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mP8/x8AAwMCAO+jWZkAAAAASUVORK5CYII='
        ));
    }

    protected function tearDown(): void
    {
        if ($this->testConnectionConfigured) DB::disconnect('order_backend_test');
        parent::tearDown();
    }

    private function user(string $label, string $role): User
    {
        return User::forceCreate([
            'name' => 'Test '.$label, 'email' => $label.'@example.test', 'password' => 'Temporary-test-password!',
            'role' => $role, 'is_active' => true, 'must_change_password' => false,
        ]);
    }

    private function item(string $code, string $type, int $price): Flower
    {
        $flower = Flower::forceCreate([
            'code' => $code, 'name' => 'Test '.$type, 'product_type' => $type,
            'category' => 'Roses', 'minimum_stock' => 1, 'description' => 'Test item.', 'status' => 'Active',
        ]);
        FlowerPrice::create(['flower_id' => $flower->id, 'price_cents' => $price]);

        return $flower;
    }

    private function receive(Flower $flower, int $quantity): void
    {
        $this->inventory->stockIn([
            'request_key' => (string) Str::uuid(), 'flower_id' => $flower->id, 'quantity' => $quantity,
            'unit' => $flower->product_type, 'received' => '2026-01-02', 'harvest' => '2026-01-01',
            'expiry' => null, 'remarks' => 'Test receipt.',
        ], $this->admin);
    }

    private function input(array $overrides = []): array
    {
        return array_replace([
            'request_key' => (string) Str::uuid(), 'client_id' => $this->client->id,
            'method' => 'Customer Pickup', 'address' => null, 'notes' => null,
            'items' => [['flower_id' => $this->stem->id, 'quantity' => 4]],
        ], $overrides);
    }

    private function action(Order $order, string $status, User $actor, array $overrides = []): Order
    {
        return $this->orders->action($order, array_replace([
            'status' => $status, 'version' => $this->orders->row($order->fresh(), $actor)['version'],
        ], $overrides), $actor);
    }

    private function rejected(callable $action, ?string $field = null): void
    {
        try {
            $action(); $this->fail('Expected order validation to reject this operation.');
        } catch (ValidationException $exception) {
            if ($field !== null) $this->assertArrayHasKey($field, $exception->errors());
        }
    }

    private function remaining(Flower $flower): int
    {
        return (int) InventoryBatch::where('flower_id', $flower->id)
            ->sum('remaining_quantity');
    }
}
```

## `tests/Feature/UserBackendTest.php`

```php
<?php

namespace Tests\Feature;

use App\Models\User;
use App\Services\UserService;
use App\Support\AccountSession;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Schema;
use Illuminate\Support\Str;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Validation\ValidationException;
use Symfony\Component\HttpKernel\Exception\HttpException;
use Tests\TestCase;

class UserBackendTest extends TestCase
{
    private UserService $users;
    private User $admin;
    private bool $configured = false;

    /** Use an isolated SQLite database; never refresh the owner's application DB. */
    protected function setUp(): void
    {
        parent::setUp();
        if (!extension_loaded('pdo_sqlite')) $this->markTestSkipped('pdo_sqlite is required.');
        config([
            'database.connections.user_backend_test' => [
                'driver' => 'sqlite', 'database' => ':memory:', 'prefix' => '', 'foreign_key_constraints' => true,
            ],
            'database.default' => 'user_backend_test',
            'session.driver' => 'array',
            'cache.default' => 'array',
            'cache.limiter' => 'array',
            'hashing.bcrypt.rounds' => 4,
        ]);
        $this->configured = true;
        DB::purge('user_backend_test');
        $this->assertSame(':memory:', DB::connection()->getDatabaseName());
        Schema::create('users', function (Blueprint $table) {
            $table->id(); $table->string('name'); $table->string('email')->unique();
            $table->string('password'); $table->string('role');
            $table->boolean('is_active')->default(true);
            $table->boolean('must_change_password')->default(false);
            $table->timestamp('email_verified_at')->nullable(); $table->rememberToken(); $table->timestamps();
        });
        Schema::create('clients', function (Blueprint $table) {
            $table->id(); $table->unsignedBigInteger('user_id')->unique();
        });
        $this->admin = $this->account('Admin', User::ROLE_ADMIN);
        $this->users = app(UserService::class);
    }

    protected function tearDown(): void
    {
        if ($this->configured) DB::disconnect('user_backend_test');
        parent::tearDown();
    }

    public function test_creation_hashes_temporary_password_and_exposes_no_secrets(): void
    {
        $input = $this->input(['email' => ' Courier@Example.test ', 'name' => ' Courier ']);
        $user = $this->users->store($input, $this->admin);
        $this->assertSame('courier@example.test', $user->email);
        $this->assertSame('Courier', $user->name);
        $this->assertSame(User::ROLE_DELIVERY, $user->role);
        $this->assertTrue($user->must_change_password);
        $this->assertTrue($user->is_active);
        $this->assertTrue(Hash::check($input['password'], $user->password));
        $row = $this->users->row($user, $this->admin);
        $this->assertSame('Delivery Personnel', $row['role']);
        $this->assertSame('USR-002', $row['id']);
        foreach (['password', 'remember_token', 'password_hash'] as $key) $this->assertArrayNotHasKey($key, $row);
        $this->assertStringNotContainsString($input['password'], json_encode($row));
        $this->assertSame($row['version'], $this->users->row($user->fresh(), $this->admin)['version']);
    }

    public function test_global_case_insensitive_email_uniqueness_includes_client_accounts(): void
    {
        $client = $this->account('Client', User::ROLE_CLIENT);
        DB::table('clients')->insert(['user_id' => $client->id]);
        $this->rejected(fn () => $this->users->store($this->input(['email' => strtoupper($client->email)]), $this->admin), 'email');
        $this->assertSame(2, User::count());
        $this->rejected(fn () => $this->users->store($this->input(['email' => strtoupper($this->admin->email)]), $this->admin), 'email');
    }

    public function test_profile_update_preserves_password_and_clears_verification_on_email_change(): void
    {
        $user = $this->account('Staff', User::ROLE_SUPPLIER);
        $user->email_verified_at = now(); $user->save();
        $hash = $user->password;
        $signature = AccountSession::signature($user);
        $updated = $this->users->update($user, $this->edit($user, [
            'name' => 'Updated Staff', 'email' => 'changed@example.test', 'role' => User::ROLE_DELIVERY,
        ]), $this->admin);
        $this->assertSame($hash, $updated->password);
        $this->assertFalse($updated->must_change_password);
        $this->assertSame('Updated Staff', $updated->name);
        $this->assertNull($updated->email_verified_at);
        $this->assertNotSame($signature, AccountSession::signature($updated));
    }

    public function test_stale_profile_and_reset_requests_cannot_overwrite_newer_changes(): void
    {
        $user = $this->account('Staff', User::ROLE_SUPPLIER);
        $old = $this->edit($user);
        $this->users->update($user, [...$old, 'name' => 'New Name'], $this->admin);
        $this->rejected(fn () => $this->users->update($user, [...$old, 'name' => 'Stale Name'], $this->admin), 'version');
        $this->rejected(fn () => $this->users->resetPassword($user, [
            'version' => $old['version'], 'password' => 'New-temporary-password!',
        ], $this->admin), 'version');
        $this->assertSame('New Name', $user->fresh()->name);
    }

    public function test_status_retries_are_idempotent_and_revoke_old_sessions(): void
    {
        $user = $this->account('Staff', User::ROLE_SUPPLIER);
        $signature = AccountSession::signature($user);
        $input = ['status' => 'Inactive', 'version' => $this->version($user)];
        $first = $this->users->status($user, $input, $this->admin);
        $token = $first->remember_token;
        $second = $this->users->status($user, $input, $this->admin);
        $this->assertFalse($second->is_active);
        $this->assertSame($token, $second->remember_token);
        $this->assertNotSame($signature, AccountSession::signature($second));
        $this->actingAs($second)->withSession([AccountSession::KEY => $signature])
            ->getJson('/supplier/flowers/data')->assertForbidden();
    }

    public function test_password_reset_requires_change_revokes_sessions_and_does_not_log_credentials(): void
    {
        Log::spy();
        $user = $this->account('SecondAdmin', User::ROLE_ADMIN);
        $oldSignature = AccountSession::signature($user);
        $password = 'Brand-new-temporary-password!';
        $updated = $this->users->resetPassword($user, [
            'password' => $password, 'version' => $this->version($user),
        ], $this->admin);
        $this->assertTrue(Hash::check($password, $updated->password));
        $this->assertTrue($updated->must_change_password);
        $this->assertNotSame($oldSignature, AccountSession::signature($updated));
        Log::shouldHaveReceived('info')->withArgs(fn ($message, $context) =>
            $message === 'Account temporary_password_reset'
            && !str_contains(json_encode($context), $password)
            && !str_contains(json_encode($context), $updated->password)
        )->once();
        $this->actingAs($updated)->withSession([AccountSession::KEY => $oldSignature])
            ->getJson('/supplier/users/data')->assertUnauthorized();
    }

    public function test_self_access_and_last_active_admin_are_protected(): void
    {
        $this->rejected(fn () => $this->users->update($this->admin,
            $this->edit($this->admin, ['role' => User::ROLE_DELIVERY]), $this->admin), 'role');
        $this->rejected(fn () => $this->users->status($this->admin, [
            'status' => 'Inactive', 'version' => $this->version($this->admin),
        ], $this->admin), 'role');
        $this->rejected(fn () => $this->users->resetPassword($this->admin, [
            'password' => 'New-temporary-password!', 'version' => $this->version($this->admin),
        ], $this->admin), 'password');
        $this->assertTrue($this->admin->fresh()->is_active);
        $this->assertSame(User::ROLE_ADMIN, $this->admin->fresh()->role);
    }

    public function test_admin_can_demote_another_admin_when_one_active_admin_remains(): void
    {
        $other = $this->account('OtherAdmin', User::ROLE_ADMIN);
        $updated = $this->users->update($other, $this->edit($other, ['role' => User::ROLE_SUPPLIER]), $this->admin);
        $this->assertSame(User::ROLE_SUPPLIER, $updated->role);
        $this->assertSame(1, User::where('role', User::ROLE_ADMIN)->where('is_active', true)->count());
    }

    public function test_client_accounts_are_hidden_and_cannot_be_promoted_through_user_management(): void
    {
        $client = $this->account('Client', User::ROLE_CLIENT);
        DB::table('clients')->insert(['user_id' => $client->id]);
        $this->assertCount(1, $this->users->data($this->admin)['data']);
        try {
            $this->users->update($client, [
                'name' => 'Client', 'email' => $client->email, 'role' => User::ROLE_ADMIN,
                'status' => 'Active', 'version' => str_repeat('a', 64),
            ], $this->admin);
            $this->fail('Client must remain inaccessible.');
        } catch (HttpException $exception) {
            $this->assertSame(404, $exception->getStatusCode());
        }
        $this->assertSame(User::ROLE_CLIENT, $client->fresh()->role);
    }

    public function test_password_byte_limit_and_privilege_fields_are_enforced(): void
    {
        foreach ([str_repeat('x', 73), str_repeat('é', 37), 'Long-password-with-null'."\0"] as $password) {
            $this->rejected(fn () => $this->users->store($this->input(['password' => $password]), $this->admin), 'password');
        }
        foreach (['is_active' => false, 'must_change_password' => false, 'remember_token' => 'forged'] as $field => $value) {
            $this->rejected(fn () => $this->users->store($this->input([$field => $value]), $this->admin), $field);
        }
        $this->rejected(fn () => $this->users->store($this->input(['role' => 'client']), $this->admin), 'role');
        $this->assertSame(1, User::count());
    }

    public function test_http_routes_enforce_admin_access_and_return_public_dtos(): void
    {
        $staff = $this->account('Staff', User::ROLE_SUPPLIER);
        $this->actingAs($staff)->withSession([AccountSession::KEY => AccountSession::signature($staff)])
            ->postJson('/supplier/users', $this->input())->assertForbidden();
        $this->actingAs($this->admin)->withSession([AccountSession::KEY => AccountSession::signature($this->admin)])
            ->postJson('/supplier/users', $this->input())
            ->assertCreated()->assertJsonPath('data.roleValue', 'delivery')
            ->assertJsonMissingPath('data.password')->assertJsonMissingPath('data.remember_token');
    }

    public function test_delivery_login_forces_password_change_then_redirects_to_scoped_delivery_page(): void
    {
        $user = $this->users->store($this->input(), $this->admin);
        $this->post('/login', ['email' => $user->email, 'password' => 'Temporary-password!'])
            ->assertRedirect(route('password.change'));
        $this->put('/change-password', [
            'current_password' => 'Temporary-password!',
            'password' => 'Changed-personnel-password!', 'password_confirmation' => 'Changed-personnel-password!',
        ])->assertRedirect(route('delivery.deliveries'));
        $this->assertFalse($user->fresh()->must_change_password);
        $this->getJson('/supplier/users/data')->assertForbidden();
        $this->getJson('/supplier/deliveries/data')->assertForbidden();
    }

    private function account(string $name, string $role): User
    {
        return User::forceCreate([
            'name' => $name, 'email' => strtolower($name).'@example.test',
            'password' => 'Original-password!', 'role' => $role,
            'is_active' => true, 'must_change_password' => false,
        ]);
    }

    private function input(array $overrides = []): array
    {
        return array_replace([
            'name' => 'New Courier', 'email' => 'courier@example.test',
            'role' => User::ROLE_DELIVERY, 'status' => 'Active', 'password' => 'Temporary-password!',
        ], $overrides);
    }

    private function version(User $user): string
    {
        return $this->users->row($user->fresh(), $this->admin)['version'];
    }

    private function edit(User $user, array $overrides = []): array
    {
        $user = $user->fresh();
        return array_replace([
            'name' => $user->name, 'email' => $user->email, 'role' => $user->role,
            'status' => $user->is_active ? 'Active' : 'Inactive', 'version' => $this->version($user),
        ], $overrides);
    }

    private function rejected(callable $callback, ?string $field = null): void
    {
        try { $callback(); $this->fail('Expected account validation rejection.'); }
        catch (ValidationException $exception) {
            if ($field !== null) $this->assertArrayHasKey($field, $exception->errors());
        }
    }
}
```
