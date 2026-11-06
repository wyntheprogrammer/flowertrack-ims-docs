# Supplier Client Management Backend Setup

**Last updated:** 2026-11-06

## Current decision and scope

Owner explicitly deferred Spoilage Prediction through the dataset-availability question response: "ay sorry wag pala muna tayo kay spoilage module next module na". Next module is Supplier-side Client Management. Do not build date-derived fake ML risk labels or resume Prediction without owner direction.

Client source package is prepared, not owner-installed/approved/pushed yet. Existing Batch commit 330b885 is the last confirmed application milestone. Owner completed the package-lock restore/install discussion before proceeding.

## Architecture

- users remains authoritative for name, email, hashed password, role, active flag and must-change-password flag.
- clients adds unique generated CLI code, unique user_id relationship, phone, address and automatic timestamps.
- Account and profile are created in one transaction; role is fixed to client, must_change_password true.
- Supplier/Admin may manage clients. Privilege/identity fields are rejected; User model fillable/hidden/casts remain unchanged.
- All email comparisons are normalized case-insensitively across ALL user roles, not just the visible client list.
- Updates lock User before Client; stale public versions are rejected. Explicit desired Active/Inactive status requests are safe to repeat after success.
- Password is creation-only: minimum 12 characters, maximum 72 UTF-8 bytes for bcrypt compatibility; no null bytes. The existing User hashed cast hashes it. Public DTOs never include password/hash/tokens/privilege fields.
- Editing profile does not reset password, alter role or clear must-change-password. A changed email clears prior email_verified_at.
- No delete or password-reset endpoint is introduced; records are retained and user FK restricts deletion.
- Client portal/login remains deferred. Current Supplier login authorization is unchanged and does not admit role client. Saved credentials and first-change flag are ready for later Client authentication integration; do not claim client sign-in works now.
- Order backend is not connected yet. Profiles return order_history_available=false, summary=null and orders=[]; UI displays unavailable totals instead of invented purchases or sample orders.

## Full replacement files

- database/migrations/2026_11_06_120000_create_clients_table.php
- app/Models/Client.php
- app/Http/Requests/ClientRequest.php
- app/Http/Requests/StoreClientRequest.php
- app/Http/Requests/UpdateClientRequest.php
- app/Http/Requests/UpdateClientStatusRequest.php
- app/Services/ClientService.php
- app/Http/Controllers/Supplier/ClientController.php
- resources/views/supplier/clients.blade.php
- resources/js/supplier/clients.js
- tests/Feature/ClientBackendTest.php

Keep existing Client CSS, User model and app.js Client import. New migration timestamps may differ when generated through Artisan; keep exactly one *_create_clients_table.php file. Tests discover by that suffix instead of a fixed timestamp.

## Routes

Import App\\Http\\Controllers\\Supplier\\ClientController once near the top of routes/web.php.

Replace only existing Client routes INSIDE the existing supplier group supplying prefix supplier, name supplier., authentication, active account, changed password and admin/supplier authorization:

```php
Route::get('/clients', [ClientController::class, 'index'])->name('clients');
Route::get('/clients/data', [ClientController::class, 'data'])->name('clients.data');
Route::get('/clients/{client}', [ClientController::class, 'show'])
    ->whereNumber('client')->name('clients.show');
Route::post('/clients', [ClientController::class, 'store'])->name('clients.store');
Route::post('/clients/{client}/update', [ClientController::class, 'update'])
    ->whereNumber('client')->name('clients.update');
Route::post('/clients/{client}/status', [ClientController::class, 'status'])
    ->whereNumber('client')->name('clients.status');
```

## CMD installation checks

Run in C:\\xampp\\htdocs\\FlowerTrack after copying files. Address syntax errors before migration.

```cmd
php -l app/Models/Client.php
php -l app/Services/ClientService.php
php -l app/Http/Controllers/Supplier/ClientController.php
php -l app/Http/Requests/ClientRequest.php
php -l app/Http/Requests/StoreClientRequest.php
php -l app/Http/Requests/UpdateClientRequest.php
php -l app/Http/Requests/UpdateClientStatusRequest.php
php -l tests/Feature/ClientBackendTest.php
php -l database/migrations/2026_11_06_120000_create_clients_table.php

php artisan config:clear
php artisan route:clear
php artisan view:clear
php artisan migrate
php artisan route:list --path=supplier/clients

npm.cmd run build
php artisan test --filter=ClientBackendTest
```

If the migration was generated with a different timestamp, lint that actual filename. There should be 6 Client routes. npm.cmd also avoids the PowerShell npm.ps1 execution-policy error encountered on the other device.

The migration creates clients only; it does not replace users or existing records. No fresh/rollback is required.

## Verification boundary

Database and Backend subagents reviewed architecture/schema and UI/server contracts. JavaScript syntax and 13 focused mocked-request checks passed. Eight isolated SQLite PHP tests were prepared, not executed here because PHP is unavailable. They create a named :memory: connection, create prerequisite users schema there, require exactly one Client migration and do not refresh/migrate the real application database.

Tests cover hashed fixed-role creation and atomic rollback, credential-free JSON, case-insensitive email conflicts, protected updates, stale versions, explicit status retry, forged-field/password/auth boundaries and inaccessible operational account profiles. They do not establish full HTTP/Client portal/browser/MariaDB concurrency behavior.

## Browser acceptance

1. Add a client with a unique email and temporary password of at least 12 characters.
2. Refresh: generated CLI code and profile persist.
3. Inspect users and clients in phpMyAdmin: correct link and hashed password; never paste credentials into chat.
4. Edit contact information/status and refresh; password/code/role remain unchanged.
5. Deactivate and reactivate through confirmation; profile remains visible.
6. Duplicate email, including an existing operational account email, is rejected.
7. View shows real profile and unavailable order metrics, not sample orders.

After owner approval, update MD progress and provide a targeted application commit/push script, then proceed to Order backend. Spoilage Prediction remains deferred.
