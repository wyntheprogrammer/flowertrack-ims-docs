# System Modules

**Last updated:** 2026-11-02

## A. Supplier / Admin Panel

The project owner supplied the following module order. This is also the initial Supplier/Admin UI implementation order after the shared UI foundation.

| UI order | Module | Purpose |
|---:|---|---|
| 1 | Dashboard | Overall business overview |
| 2 | Flower Management | Manage flower information |
| 3 | Inventory Management | Monitor stocks and movements |
| 4 | Batch Management | Track individual flower batches |
| 5 | Spoilage Prediction | Predict which flower batches are at risk of spoilage |
| 6 | Client Management | Manage customers |
| 7 | Order Management | Manage client orders |
| 8 | Delivery Management | Manage deliveries |
| 9 | Sales and Payment Management | Manage sales and payments |
| 10 | Reports | Generate business reports |
| 11 | User and Access Management | Manage accounts and permissions |
| 12 | Audit Trail | Track system activities |

### Supplier/Admin UI notes

- Management-oriented layout.
- Desktop-first and responsive.
- Bootstrap components, custom CSS, and Flexbox as appropriate.
#### Flower Management — approved scope (2026-11-02)

**Purpose:** Maintain the flower records used by inventory and the Client catalog.

**Fields**
- Flower Code — automatically generated.
- Flower Name — required.
- Category — required.
- Description — required.
- Primary Flower Image — optional; one primary image per flower.
- Status — Active or Inactive.

**Actions and controls**
- Add, view, and edit flower records.
- Deactivate a flower instead of permanently deleting it when related records exist.
- Search by flower name or flower code.
- Filter by category and status.

**Confirmed rules**
- A flower can be saved without an image, but not without its name, category, or description.
- Unit and selling price are not part of the approved Flower Management scope yet; decide them with the inventory and pricing workflows.

#### Inventory Management — decisions confirmed so far (2026-11-02)

**Purpose:** Show consolidated stock and track stock movements. Detailed scope is still being finalized.

**Confirmed rules**
- Inventory is batch-based. Every Stock-In must create a corresponding Batch Record.
- Overall inventory quantity is derived from the quantities of its batches; there is no separate independent stock quantity.
- FIFO (First In, First Out) is the approved default allocation method for Stock-Out: consume stock from the oldest received batch first.
- Stock movements must be recorded in Inventory History.

**Still to decide**
- Stock adjustment and physical inventory counting workflow.
- Other stock movement types and handling of damaged, missing, reserved, or returned quantities.
- Detailed fields, filters, table columns, permissions, and validation rules.

- For modules without an approved scope below, detailed fields, actions, filters, table columns, metrics, and workflows remain to be gathered.
- Do not invent operational rules from module names alone.

## B. Client Panel

The Client Panel must have a distinct ecommerce-style user experience.

| UI order | Navigation | Initial purpose |
|---:|---|---|
| 1 | Dashboard | Client landing/overview |
| 2 | Flower Catalog | Browse flowers |
| 3 | Shopping Cart | Review selected items |
| 4 | My Orders | View client orders |
| 5 | Order Tracking | Track order progress |
| 6 | My Payments | View payment information |
| 7 | My Profile | Manage client profile |

### Client UI notes

- Online-shop-style browsing and purchasing experience.
- Do not reuse the Supplier/Admin dashboard as the Client design.
- Product detail, checkout, registration, and guest-shopping rules need confirmation.

## C. Module dependencies to investigate

These are analysis topics, not finalized business rules:
- Flower Management supplies the flower/product information used by inventory and catalog.
- Batch Management records individual stock lots and supports batch-level tracking.
- Inventory Management presents overall stock and stock movements.
- Spoilage Prediction uses batch-related information, subject to the eventual prediction design.
- Orders, deliveries, sales, and payments need a confirmed lifecycle and relationship model.
