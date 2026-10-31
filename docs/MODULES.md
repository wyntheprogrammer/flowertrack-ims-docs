# System Modules

**Last updated:** 2026-11-01

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
- Detailed fields, actions, filters, table columns, dashboard metrics, and workflows are still to be gathered.
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
