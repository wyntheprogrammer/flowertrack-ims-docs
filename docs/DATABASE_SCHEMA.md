# Database Schema

**Last updated:** 2026-11-01

**Status:** Not designed yet. This file is a placeholder for the approved database design.

The database schema will be designed after UI requirements and business workflows are sufficiently understood. UI-first does not mean inventing database tables before the required data and rules are known.

## Topics to resolve before schema design

- Flower/product identity, category, variety, units, and attributes.
- Batch identity, received quantity, remaining quantity, dates, supplier/source, and condition.
- Stock movement types and adjustment reasons.
- Client/customer account and contact details.
- Order items, order status, cancellation, and return handling.
- Delivery assignment, address, fee, and delivery status.
- Payments, payment method, verification, refunds, and references.
- User accounts, roles, permissions, and audit events.
- BatchDailyObservation retains batch-level source data. The nullable JSON flowers.spoilage_model stores per-flower order and evaluation metadata. BatchSpoilagePrediction stores per-batch results. See the [flower-level forecasting methodology](BUSINESS_LOGIC.md#flower-level-arima-forecasting).

## Schema documentation format

When approved, document each table with:
- Table name and purpose.
- Column name and data type.
- Primary key and foreign keys.
- Nullability and default values.
- Unique constraints and indexes.
- Relationships and deletion behavior.
- Business rules and example records.

No tables or columns are approved by this document yet.
