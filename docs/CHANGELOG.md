# Changelog

**Last updated:** 2026-11-01

All meaningful documentation and project-planning updates are recorded here.

## 2026-11-01

### Added
- Initial FlowerTrack IMS documentation structure.
- Project overview and requirements baseline.
- Supplier/Admin module list in the owner's specified order.
- Client Panel navigation and ecommerce-style requirement.
- Technology stack and supplied repository reference catalog.
- System architecture direction and open architecture decisions.
- UI/UX guidelines for Bootstrap, Flexbox, and custom CSS.
- UI-first development plan.
- Development progress tracker and AI continuity instructions.
- Date requirement for meaningful Markdown updates.
- Supplier Side UI design-reference document with the approved dashboard screenshot and supplied design-prompt requirements.
- Supplier Side dashboard screenshot stored at `assets/ui-references/supplier-dashboard-reference.jpg`.

### Clarified
- JavaScript is the intended frontend scripting language.
- Python is a possible technology for spoilage prediction, pending requirements.
- The docs repository is separate from the planned application-code repository.
- The Supplier Side dashboard screenshot is an approved visual reference only; it is not the official FlowerTrack IMS logo and must not be copied as a branded template.
- The earlier exclusion of the external Supplier dashboard reference is superseded by the owner's later approval.
- The local development environment uses XAMPP's bundled MariaDB 10.11.16; do not refer to that server as Oracle MySQL.

### Completed — initial environment setup
- Installed XAMPP under `C:\xampp` and resolved initial Apache/MySQL port conflicts so the local XAMPP dashboard/phpMyAdmin became accessible.
- Installed and verified PHP 8.3.36 and Composer 2.11.1.
- Enabled the PHP ZIP extension and verified the PHP extensions required by the Laravel application.
- Installed Laravel application skeleton 13.11.0 in `C:\xampp\htdocs\FlowerTrack`.
- Started the Laravel development server with `php artisan serve` and confirmed the default welcome page at `http://127.0.0.1:8000`.
- Opened the project in Visual Studio Code using `code .`.
- Installed and verified Node.js 26.13.0 (`node -v`) and npm 11.20.0 (`npm -v`).

### Documentation updates
- Recorded the actual local environment versions, verified setup, remaining checks, and next task in `DEVELOPMENT_PROGRESS.md`.
- Updated `AI_CONTEXT.md` with the local setup baseline and the instruction to proceed one step at a time.
- Updated `TECHNOLOGY_STACK_AND_REFERENCES.md` to distinguish XAMPP's MariaDB from Oracle MySQL and record installed versions.

### Not yet done
- Git availability/version still needs to be checked.
- The separate `flowertrack-ims` application repository has not yet been created.
- No UI screens have been implemented.
- No database schema has been approved.
- No backend or database implementation has started.
