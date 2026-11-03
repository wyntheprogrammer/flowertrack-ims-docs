# Workflows

**Last updated:** 2026-11-03

**Status:** The Inventory-owned spoilage workflow below is confirmed as of 2026-11-03. Other sections retain their previously documented topic/proposal status.

## Supplier/Admin workflows to map

1. Dashboard review.
2. Flower/product creation and maintenance.
3. Stock receiving and inventory movement.
4. Batch creation and batch monitoring.
5. Spoilage risk review and action.
6. Client record management.
7. Order review and processing.
8. Delivery preparation and tracking.
9. Sales and payment handling.
10. Report generation.
11. User account and permission administration.
12. Audit activity review.

## Client workflows to map

1. Visit client dashboard.
2. Browse flower catalog.
3. View product details.
4. Add items to shopping cart.
5. Review cart and proceed to checkout.
6. Submit order.
7. View order status and tracking.
8. View payment records.
9. Manage profile.

The checkout and order submission steps are candidate flow details inferred from the online-shop requirement; exact behavior, payment rules, and guest/account requirements remain to be confirmed.


### Updated — spoilage workflow and Batch controls (2026-11-03)

This owner-approved change supersedes the earlier separate manual Mark as Spoiled action in Batch Management.

- The user records a Stock-Out with purpose Spoilage and required explanatory remarks in Inventory Management.
- For partial spoilage, deduct only the spoiled quantity; retain the usable remainder and its quantity-derived batch status.
- When a spoilage Stock-Out consumes a batch's entire remaining quantity, the backend transaction must also set that batch's status to Spoiled, without a second stock deduction.
- A batch depleted by ordinary consumption remains Depleted; a passed freshness date or ML prediction does not mark it Spoiled.
- Batch Management displays the resulting status and supports viewing/history and metadata editing. It has no separate Mark as Spoiled button.
- Keep depleted/spoiled batches and their transaction history.
- Batch listing uses search, Show Entries, native column sorting and pagination. Flower/status/received-date filter controls were removed from the approved UI.
- These cross-module status updates are the agreed backend behavior; current separate sample-data prototypes do not implement shared persistence.
