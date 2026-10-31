# Requirements

**Last updated:** 2026-11-01

Status labels:
- **Confirmed** — explicitly provided or approved by the project owner.
- **To decide** — information still needed before implementation.
- **Proposed** — a suggestion, not an approved requirement.

## 1. General requirements

| ID | Requirement | Status |
|---|---|---|
| GEN-01 | The system is named FlowerTrack IMS. | Confirmed |
| GEN-02 | The system is a web application. | Confirmed |
| GEN-03 | The design direction is desktop-first and responsive. | Confirmed |
| GEN-04 | Supplier/Admin and Client interfaces must be different. | Confirmed |
| GEN-05 | The Client Panel should provide an online-shop-style experience. | Confirmed |
| GEN-06 | UI must be completed and reviewed before backend development begins. | Confirmed |
| GEN-07 | Development must be taught and completed step by step. | Confirmed |
| GEN-08 | Meaningful updates and newly learned technical knowledge must be recorded in Markdown and pushed to the docs repository, subject to authorization/access. | Confirmed |

## 2. Supplier/Admin modules

The initial module order supplied by the project owner is the intended UI implementation order:

1. Dashboard
2. Flower Management
3. Inventory Management
4. Batch Management
5. Spoilage Prediction
6. Client Management
7. Order Management
8. Delivery Management
9. Sales and Payment Management
10. Reports
11. User and Access Management
12. Audit Trail

Descriptions are in `MODULES.md`.

## 3. Client navigation

The confirmed initial navigation order is:
1. Dashboard
2. Flower Catalog
3. Shopping Cart
4. My Orders
5. Order Tracking
6. My Payments
7. My Profile

## 4. Technology requirements

| Technology | Intended role | Status |
|---|---|---|
| Laravel | PHP web application framework | Confirmed direction |
| PHP | Server-side language | Confirmed |
| MySQL | Database | Confirmed direction |
| HTML | Page structure | Confirmed |
| JavaScript | Frontend interactions | Confirmed |
| Bootstrap | UI components and responsive layout | Confirmed preference |
| Flexbox | Layout and alignment | Confirmed preference |
| Custom CSS | Colors, sizes, spacing, typography and visual styling | Confirmed preference |
| Python | Possible spoilage prediction/algorithm support | To decide |

## 5. UI requirements

- Bootstrap is the primary UI framework.
- Flexbox may be used for layout and alignment.
- Custom CSS controls the color scheme, font sizes, spacing, dimensions, and other visual details.
- Build reusable UI patterns where appropriate.
- Supplier/Admin is management-oriented.
- Client is ecommerce-oriented.
- No external dashboard reference image or palette is currently approved.

## 6. Business requirements still to gather

- Flower product categories, varieties, units, and product attributes.
- Whether products include bouquets, plants, vases, wrappers, gifts, or other items.
- Batch creation and receiving process.
- Stock unit and stock movement rules.
- Batch aging, shelf-life, and spoilage definition.
- Spoilage prediction data, method, and intended output.
- Supplier/vendor management needs (the current module list does not include a dedicated supplier directory; confirm whether one is needed).
- Customer registration and account approval rules.
- Order statuses and cancellation/return rules.
- Delivery assignment, fees, and delivery statuses.
- Payment methods, proof of payment, verification, refunds, and reconciliation.
- Reports and export formats.
- Roles, permissions, and account lifecycle.
- Audit trail events and retention.
- Single-store or multi-branch operation.
- Security, backup, hosting, and deployment requirements.

## 7. Open decisions

Do not implement these as settled decisions until confirmed:
- Exact Laravel/PHP/Node versions.
- Laravel starter/authentication approach.
- Whether to use a UI admin template or build the Supplier/Admin UI from Bootstrap components.
- Whether Python is needed and how it communicates with Laravel.
- Exact UI colors, fonts, logo assets, and design tokens.
- Exact database schema and business rules.
