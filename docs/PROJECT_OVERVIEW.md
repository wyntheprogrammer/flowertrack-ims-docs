# Project Overview

**Last updated:** 2026-11-01

## Project name

**FlowerTrack IMS** — Flower Shop Inventory Management System.

## Purpose

FlowerTrack IMS is planned as a web application for managing flower-shop operations through a Supplier/Admin management interface and a separate Client ecommerce-style interface.

The system is expected to support flower information, inventory, batch tracking, spoilage-risk prediction, customers, orders, deliveries, sales, payments, reports, user access, and audit activities.

## Main user-facing areas

### Supplier/Admin Panel

A management-oriented interface for staff and administrators. It contains the 12 modules listed in `MODULES.md`.

### Client Panel

A separate online-shop-style interface for customers. Its planned navigation is:
- Dashboard
- Flower Catalog
- Shopping Cart
- My Orders
- Order Tracking
- My Payments
- My Profile

The Client Panel must not simply reuse the Supplier/Admin dashboard layout.

## Platform

- Web application.
- Desktop-first, responsive design direction.
- Exact supported browsers, minimum screen sizes, hosting, and deployment target are still to be decided.

## Development philosophy

- Teach and implement step by step.
- Complete and review the UI before backend development.
- Maintain project knowledge in this documentation repository.
- Keep the actual application source code in a separate repository.
- Record meaningful decisions and progress with dates.

## Scope boundaries

This overview records the currently known scope only. Detailed flower attributes, batch rules, stock calculations, order lifecycle, payment methods, delivery rules, and spoilage prediction method remain subject to requirements gathering.
