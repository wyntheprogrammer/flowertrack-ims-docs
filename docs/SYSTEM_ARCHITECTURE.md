# System Architecture

**Last updated:** 2026-11-01

## Current architectural direction

FlowerTrack IMS is planned as a Laravel-based web application backed by MySQL, with HTML, JavaScript, Bootstrap, and custom CSS for the browser interface.

The application has two distinct presentation areas:
- Supplier/Admin Panel — operational and management interface.
- Client Panel — ecommerce-style shopping interface.

## Proposed logical layers

This is a conceptual starting point, not a finalized implementation decision.

1. **Presentation layer:** Blade-rendered pages, HTML, Bootstrap, custom CSS, JavaScript.
2. **Application layer:** Laravel routes, controllers, validation, services, and authorization.
3. **Data layer:** Laravel models and migrations using MySQL.
4. **Optional prediction component:** Python service/library, only if requirements confirm it is appropriate.

## Architecture decisions still pending

- Blade-only frontend versus a separate frontend framework.
- Authentication approach and session strategy.
- Role/permission package or custom authorization.
- Whether Python prediction runs inside a separate service or another integration pattern.
- File/image storage approach.
- Hosting and deployment architecture.

## Guideline

Prefer a maintainable architecture that the project owner can understand and operate. Do not add a separate frontend framework, microservice, or extra infrastructure without a demonstrated need and explicit discussion.
