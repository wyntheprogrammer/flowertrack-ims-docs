# Changelog

**Last updated:** 2026-11-01

All meaningful documentation and project-planning updates are recorded here.

## 2026-11-01

### Clarified — per-module documentation and Git workflow
- Confirmed the repeatable completion cycle: finish and review one module, update the relevant Markdown documentation immediately, provide a complete application Git build/check + stage + commit + push script, verify the owner's push output, and only then proceed to the next module.
- Clarified that the documentation repository and application repository are separate and their updates/pushes must be tracked independently.
- Required targeted staging and explicit protection of `.env`, `vendor/`, and `node_modules/`.


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
- Verified application repository tree snapshot and current frontend file baseline in `docs/AI_CONTEXT.md`.
- Repository-first inspection rule: inspect the live application repository before asking the owner to provide committed folder/file structure.

### Clarified
- JavaScript is the intended frontend scripting language.
- Python is a possible technology for spoilage prediction, pending requirements.
- The docs repository is separate from the application-code repository.
- The Supplier Side dashboard screenshot is an approved visual reference only; it is not the official FlowerTrack IMS logo and must not be copied as a branded template.
- The earlier exclusion of the external Supplier dashboard reference is superseded by the owner's later approval.
- The local development environment uses XAMPP's bundled MariaDB 10.11.16; do not refer to that server as Oracle MySQL.
- The public application repository `wyntheprogrammer/flowertrack-ims` exists on branch `main`; its initial Laravel skeleton is committed and pushed.

### Completed — initial environment setup
- Installed XAMPP under `C:\xampp` and resolved initial Apache/MySQL port conflicts so the local XAMPP dashboard/phpMyAdmin became accessible.
- Installed and verified PHP 8.3.36 and Composer 2.11.1.
- Enabled the PHP ZIP extension and verified the PHP extensions required by the Laravel application.
- Installed Laravel application skeleton 13.11.0 in `C:\xampp\htdocs\FlowerTrack`.
- Started the Laravel development server with `php artisan serve` and confirmed the default welcome page at `http://127.0.0.1:8000`.
- Opened the project in Visual Studio Code using `code .`.
- Installed and verified Node.js 26.13.0 (`node -v`) and npm 11.20.0 (`npm -v`).
- Verified Git 2.52.0.windows.1 with `git --version`.
- Configured Git global commit identity with user name `wyn` and email `wyntheprogrammer@gmail.com`.

### Documentation updates
- Recorded the actual local environment versions, verified setup, remaining checks, and next task in `DEVELOPMENT_PROGRESS.md`.
- Updated `AI_CONTEXT.md` with the local setup baseline and the instruction to proceed one step at a time.
- Updated `TECHNOLOGY_STACK_AND_REFERENCES.md` to distinguish XAMPP's MariaDB from Oracle MySQL and record installed versions.
- Updated `AI_CONTEXT.md` and `DEVELOPMENT_PROGRESS.md` with the verified application repository tree and current frontend entry-point status.
- Corrected the documentation README to identify the application repository as already created and pushed.

### Completed — application Git setup
- Created the public GitHub application repository `wyntheprogrammer/flowertrack-ims`.
- Initialized Git in `C:\\xampp\\htdocs\\FlowerTrack`, staged the Laravel starter project, and created initial commit `ca10822` (`Initial Laravel project setup`).
- Renamed the local branch to `main`, added the GitHub repository as `origin`, and successfully pushed `main` to GitHub. The local branch tracks `origin/main`.

### Not yet done
- No UI screens have been implemented.
- No database schema has been approved.
- No backend or database implementation has started.


### Completed — Bootstrap foundation (local working copy)
- Installed Bootstrap 5.3.8 and Popper 2.11.8 with npm; removed the Tailwind packages and Tailwind Vite plugin.
- Updated Vite configuration and frontend CSS/JavaScript entry points to load Bootstrap.
- Ran npm run build successfully.
- Replaced the Laravel starter welcome view with a simple FlowerTrack IMS Bootstrap verification page.
- Confirmed the page displays correctly in the browser at http://127.0.0.1:8000.
- Recorded that these application changes are verified locally; their push to the application GitHub repository has not been confirmed.

### Completed — Supplier/Admin dashboard UI first pass (local working copy)
- Created the Supplier/Admin Blade layout and dashboard view.
- Kept Gemma's Flower Shop logo in the sidebar and moved FlowerTrack IMS application name/subtitle to the top navbar to separate business branding from application identity.
- Removed the navbar search field at the owner's request.
- Refined sidebar navigation spacing and Quick Actions panel spacing.
- Added the two owner-supplied images as `public/images/flower-banner.jpg` and `public/images/bloom-business.jpg`.
- Used the wide flower image on the right side of the welcome banner without stretching it, and the pink rose image in the Bloom Your Business panel.
- Preserved the previously approved dashboard content/design after the owner requested reverting an alternative redesign.
- Owner confirmed the current dashboard is acceptable.
- These application changes are local; the system repository push is the next task and is not yet confirmed.


### Updated — Bootstrap Icons standard
- Adopted Bootstrap Icons as the standard icon library for interface icons across Supplier/Admin and Client panels.
- Installed the `bootstrap-icons` npm package and imported `bootstrap-icons/font/bootstrap-icons.css` through `resources/css/app.css`.
- Updated Supplier sidebar and Supplier Dashboard interface icons (summary cards, panel headings, Quick Actions, and Recent Activities) to Bootstrap Icons.
- Kept flower emojis in Top Selling Products as product visuals rather than interface icons.
- Preserved the approved Supplier Dashboard layout and existing content.
- Application changes are local; build verification and push to `wyntheprogrammer/flowertrack-ims` are pending.

### Still pending
- Remaining Supplier/Admin modules in the order documented in `MODULES.md`.
- Client ecommerce UI screens.
- Full UI review and approval before backend/database work.
