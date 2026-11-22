# Live Daily Batch Observations

**Last updated:** 2026-11-17
**Status:** approved by Wyn and published in application commit [`bb8d9bb`](https://github.com/wyntheprogrammer/flowertrack-ims/commit/bb8d9bbbc989e8069587e78fefc364286c4fe468).

This revises implementation steps 2 (daily observations) and 3 (closing/catch-up). Stock-In creates the daily row immediately. Successful eligible Admin/Supplier login and logout verify past days. There is no scheduled, midnight, or periodic closing trigger. Dates still identify observations and prevent completing an unfinished day; they do not start processing by themselves.

## Step 2: live observations

1. Stock-In creates the batch and signed receipt movement, then creates its observation for the movement's **Asia/Manila calendar date**, all in the inventory transaction.
2. Every new spoilage Stock-Out, other-use Stock-Out, completed pickup/delivery issue, receipt correction or batch-metadata movement invokes the same synchronous observer.
3. The observer replays that batch's signed movement history and rebuilds the affected day's draft. It never adds a movement quantity to an existing observation counter.
4. The existing unique batch/date constraint and parent Flower/batch locks protect one observation per day. A matching request-key retry returns the existing inventory operation and creates neither another movement nor another observation.
5. Drafts have is_complete=false and verified_at=null. No scheduled command or page refresh is required for Stock-In or subsequent movements to populate them.
6. On a new day, the first movement creates that day's draft when enough ledger information is available. A day with no movements is created by closing.

Quantities are always in the item's own base unit: loose Stem or supplier-made finished Bouquet. Receiving conversions are snapshotted; there is no bouquet assembly/BOM deduction.

Receipt-day opening includes the initial receipt. On other ordinary days, opening is the preceding closing balance. For eligible rows:

    opening_quantity - spoiled_quantity - released_quantity = closing_quantity

Example: receive 100 stems, record 5 spoiled, and fulfill 20 stems: the same draft becomes opening 100, spoiled 5, released 20, closing 75. It remains incomplete until closing. Replaying any request does not change those totals again.

## Step 3: catch-up on login/logout

- A successful active Admin/Supplier login or logout invokes the shared day-closing service. Login is the primary opportunity; logout is an additional opportunity. Temporary-password accounts, inactive accounts, failed authentication, Client accounts and Delivery accounts do not trigger it.
- Catch-up checks dates strictly before today in Asia/Manila. A login just after midnight can close yesterday. Logout never closes today's draft early.
- A persistent checkpoint starts on the date the closing-state migration was activated. Processing does not manufacture earlier preactivation history.
- Each eligible login/logout processes up to seven pending dates by default. A large backlog is continued by the next eligible login/logout; repeated triggers do not repeat completed work. Bounded work limits the delay to an authentication request, but a large day's inventory can still take time.
- Closing replays and reconciles movements against current batch stock, verifies eligible drafts, and inserts verified zero-movement rows for batches that still had stock that day.
- An already depleted batch does not accumulate endless zero-opening observations. Its actual depletion day can still be recorded.
- Matching source rows are updated, not duplicated. An unchanged completed row retains its verification timestamp and does not produce another collection audit event.
- Exceptions leave the unfinished date pending. Partially saved batch rows are safe to process again.
- Activity-triggered verification records the triggering account in collected_by and the audit actor, with source User Activity. This identifies who triggered reconciliation; it does not claim the user manually confirmed every zero-movement day. Explicit CLI recovery is a Console action.
- Catch-up failures never replace a successful login/logout response. They are reported/logged and leave the unfinished checkpoint pending for the next eligible event. Missing migrations also leave authentication usable.
- The previous FlowerTrackIMS-Scheduler Windows task is disabled. The periodic Laravel schedule and scheduler installer/runner scripts have been removed. No queue worker, Windows task or daily command is required for normal catch-up.
- Without eligible login/logout, historical completion waits. Simply staying signed in, viewing pages, closing a browser window, or an inactive period does not itself close a day. Browser close and network loss are not reliable logout events.

Example: stock arrives November 17 and immediately creates a draft. Nobody signs in or out on November 18. An Admin/Supplier logs in November 19: catch-up verifies November 17 and inserts an eligible zero-movement November 18 row when the ledger supports it. November 19 stays open. Movements continue to maintain that day's draft independently.

## Corrections, invalidation and concurrency

The observer runs before the inventory transaction commits. A failure later in Stock-In, pickup/delivery completion or draft recording rolls back the operation, movements, stock and observations together.

Late movements invalidate affected collector-owned history. The changed day's row is rebuilt as a draft, and an existing current-day draft is refreshed when affected. Past dependent rows are reverified by catch-up. The closing checkpoint is rewound to the earliest affected eligible date, bounded by activation.

The checkpoint has a revision counter. Closing advances only if both its pending date and revision are unchanged. A movement arriving during closing therefore leaves the date pending for retry, rather than silently advancing over invalidated history.

Receipt/date corrections remain conservative: affected historical days and the correction day are excluded from forecasting when their history cannot be verified. Correction deltas are never labeled spoilage or sales. Diagnostic quantities may be present on an excluded draft, but that is not eligible model history. Storage changes within a day also exclude that day; later stable days can become eligible.

Backdated received/harvest dates do not fabricate outcomes before the first recorded receipt movement. Unknown storage is not inferred. Incomplete, unsupported or inconsistent ledgers remain excluded. Existing imported observations with another/null source are preserved rather than overwritten by this collector.

## Prediction eligibility

Python does not connect to the database directly. Laravel supplies eligible history from batch_daily_observations to Python.

The forecasting query pools completed, unexcluded observations with valid opening and spoiled quantities before the forecast date. Storage is not a grouping key or filter in that query. Corrections change the input fingerprint. See the [flower-level forecasting methodology](BUSINESS_LOGIC.md#flower-level-arima-forecasting).

## Preserved business rules

- Order confirmation reserves stock; payments and Ready for Pickup/Delivery do not issue it.
- Actual collected pickup and successful delivered completion issue reservations once, using the existing inventory flow and FIFO rules.
- Manual spoilage is recorded through Inventory Stock-Out, with the existing remarks and partial/full depletion behavior.
- Sales, payments/refunds, reservations, proof requirements, authorization and physical stock accounting are unchanged.
- Observations are derived records. They do not make additional stock deductions.

## One-time installation and review

Run from the application root:

~~~powershell
php artisan migrate
php artisan test --compact tests/Feature/BatchObservationTest.php tests/Feature/OrderBackendTest.php tests/Feature/DeliveryBackendTest.php
php artisan spoilage:close-days --preview
~~~

The additional revision migration preserves the existing activation date and checkpoint; it only adds the concurrency counter. The live workflow needs the provenance migration from the collector setup.

No scheduler installation is needed on another device. If the old task was installed there, disable it once:

~~~powershell
Disable-ScheduledTask -TaskName FlowerTrackIMS-Scheduler
~~~

The existing local Windows task has already been disabled. Keep the application/database available during login/logout. SPOILAGE_ACTIVITY_COLLECTION=false disables activity-based collection; live movement drafts remain independent. Previous SPOILAGE_AUTOMATIC_COLLECTION settings are no longer used. Do not copy Python virtual environments between devices.

For manual inspection or recovery, the existing collector remains available:

~~~powershell
php artisan spoilage:collect-observations 2026-11-16
php artisan spoilage:collect-observations 2026-11-16 --confirm-complete --actor=YOUR_EMAIL
~~~

The first command only previews and cannot mark a draft complete. The second requires an active Admin/Supplier with an updated password and only accepts completed dates. The spoilage:close-days command is an explicit recovery/preview tool, not scheduled work. Daily operation uses Stock-In/Stock-Out/order fulfillment and eligible login/logout normally; no manual daily insertion command is required.

To review without fabricated production movements: receive an actual batch, inspect its draft, record a real spoilage/release, and verify that the row ID stays the same and its quantities update. On a later Philippine date, sign in or explicitly sign out as an active Admin/Supplier and inspect completion of past observations. Today must remain incomplete. Use isolated tests to simulate time changes, authentication, interruptions, retries, rollback and late-entry races.

## Verification

Agent-run full regression: **137 passed / 1,760 assertions**. Activity-triggered observation suite: **28 passed / 239 assertions**. Pint and application/documentation whitespace checks passed. Laravel schedule inspection confirms no periodic tasks, and the real-data catch-up preview preserves the existing November 17 activation checkpoint. Tests include real login/logout requests, role/account exclusions, catch-up failure isolation, bounded backlog processing, duplicate protection, live drafts, rollback and completed-only prediction inputs. Isolated SQLite tests are not a claim of simultaneous MySQL stress testing or completed browser acceptance. No synthetic production stock transactions were created merely for testing.

Wyn approved publication of both repositories. The application checkpoint above is verified on main. The documentation records the reviewed implementation and test evidence; deployment to other devices still requires the migrations.
