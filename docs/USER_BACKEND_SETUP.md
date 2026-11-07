# User Management backend — installation

Prepared 2026-11-07 against application checkpoint `43695bf`.

Copy the supplied complete files to the matching project paths. No new migration, npm dependency, User model change, or database reset is needed.

## Included behavior

- Admin-only create, view, profile/role/status update, activate/deactivate and temporary-password reset for Admin, Supplier/Staff and Delivery Personnel accounts.
- Client identities remain managed by Client Management. Global normalized email uniqueness includes all roles.
- At least 12 password characters, maximum 72 UTF-8 bytes; null bytes rejected. User's existing hashed cast stores the hash. Passwords never enter public DTOs or event logs.
- New accounts and Admin resets force a password change. Own role removal/deactivation and own Admin reset are blocked; use Change Password for your own credential.
- Active-Admin rows are locked in ID order before account writes. Last active Admin remains protected. Opaque versions reject stale edits/resets, and status writes use explicit desired state rather than toggles.
- Session security signatures revoke previous access after credentials, role, email, active flag or forced-change flag changes. Existing unsigned sessions must sign in again after installation. Remember-token rotation invalidates old remembered credentials.
- Safe account-event metadata is written to the Laravel server log after commit. No fabricated browser session history is shown. The separate Audit Trail backend is still a later integration.
- Delivery personnel sign in, change their temporary password, and reach `/delivery/deliveries`. They can list/view only assigned deliveries, view their own private delivery proof, record failed attempts and confirm actual delivery with proof. They cannot schedule, assign, dispatch, cancel, or access Supplier inventory/orders/payments/users.
- Admin/Supplier operational Delivery Management stays at `/supplier/deliveries`. Completion's FIFO issue, stable fulfillment UUIDs, private proof storage and atomic sale creation remain unchanged.
- The full routes replacement also restores the Supplier proof route missing from checkpoint 43695bf.

## Check installation

Run inside `C:\xampp\htdocs\FlowerTrack`:

```text
composer dump-autoload
php artisan optimize:clear
php artisan route:list --path=supplier/users
php artisan route:list --path=delivery/deliveries
npm.cmd run build
php artisan test --filter=UserBackendTest
php artisan test --filter=DeliveryBackendTest
php artisan test --filter=OrderBackendTest
```

Expected route inventory: seven Supplier User routes and five personnel Delivery routes. Twelve User tests and eleven Delivery tests are supplied; results must be verified on the owner's PHP runtime. Existing Order regression tests also remain required. Tests use isolated named SQLite memory databases, not RefreshDatabase against application data.

Assistant verification: both JS syntax checks, nine focused User UI checks, five focused personnel UI checks. PHP is not installed in the preparation environment; PHP tests, full Vite build and browser checks are not claimed as run.

## Browser acceptance

1. Sign in again as Administrator and open User Management.
2. Create a Delivery Personnel account with Active status and a temporary password. Privately share the actual credential with that person; no password email is sent automatically.
3. As Admin/Supplier, schedule and assign a Ready for Delivery order, then dispatch only when actually handed to personnel.
4. In a separate browser/private window, sign in as the delivery person, change the temporary password and verify only assigned records are visible.
5. Confirm actual receipt with recipient, remarks and valid proof. Verify Delivery Delivered, Order Completed, exactly-once stock issue, one sale, and private proof viewing. No stock deduction during scheduling/assignment/dispatch.
6. Verify status changes, failed attempt/reschedule/cancel, stale versions, Admin reset/re-login and own/last Admin protection.

After owner approval: update progress docs with actual results, inspect targeted Git status, commit/push, and return to pending Delivery browser acceptance before calling Delivery fully verified.
