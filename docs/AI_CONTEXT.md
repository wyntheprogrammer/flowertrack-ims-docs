# AI Context — FlowerTrack IMS

**Last updated:** 2026-11-01

## Role and working style

Act as a patient senior software engineer, system analyst, and UI/UX design mentor. Teach the project owner step by step. Explain commands, code, design choices, and technical concepts in straightforward language. Do not simply dump a complete application without teaching the implementation.

## Project identity

- Project name: **FlowerTrack IMS**
- Documentation repository: `flowertrack-ims-docs`
- Application repository: [`wyntheprogrammer/flowertrack-ims`](https://github.com/wyntheprogrammer/flowertrack-ims) (public; initialized and pushed to `main`)
- Application type: Web application
- Main interfaces: Supplier/Admin Panel and Client Panel
- Supplier/Admin and Client must have distinct user experiences.

## Confirmed technology direction

- Laravel and PHP for server-side application development.
- MySQL for the relational database.
- HTML for page structure.
- JavaScript for browser-side interactions.
- Bootstrap for UI components and responsive layout.
- Flexbox for layout and alignment when appropriate.
- Custom CSS for brand colors, typography, sizes, spacing, dimensions, and visual refinements.
- Bootstrap Icons for interface icons across Supplier/Admin and Client panels; use the package through the Vite CSS pipeline.
- Python is being considered for spoilage prediction; its final role and implementation are not yet decided.

Do not interpret the user's earlier mention of Java as a requirement for Java programming. The listed frontend category and intended use refer to **JavaScript**.

## Local development environment — installed and verified

The owner has completed the initial local setup on Windows:

| Component | Installed / selected version | Location or verification |
|---|---|---|
| XAMPP | Installed; exact XAMPP package version not recorded | `C:\xampp` |
| PHP | 8.3.36 | `C:\xampp\php\php.exe`; `php -v` verified |
| Composer | 2.11.1 | `composer -V` verified |
| Laravel application skeleton | 13.11.0 | Installed in `C:\xampp\htdocs\FlowerTrack` |
| MariaDB bundled with XAMPP | 10.11.16 | Version shown in XAMPP MariaDB startup log |
| Visual Studio Code | Installed and project opened | `code .` from project directory |
| Node.js | 26.13.0 | `node -v` verified |
| npm | 11.20.0 | `npm -v` verified |
| Git | 2.52.0.windows.1 | `git --version` verified; global identity configured as `wyn` / `wyntheprogrammer@gmail.com` |
| PHP ZIP extension | Enabled | `php -m` lists `zip` |
| Required Laravel PHP extensions | Verified present | Includes BCMath, Ctype, cURL, DOM, Fileinfo, Mbstring, OpenSSL, PDO, pdo_mysql, Tokenizer, and XML |

XAMPP's local dashboard/phpMyAdmin became accessible after resolving port conflicts with pre-existing MySQL and Apache processes. Do not assume the old services were uninstalled; only their conflicts were resolved during setup. The Laravel development server was started with `php artisan serve`, and the default Laravel welcome page at `http://127.0.0.1:8000` was confirmed working.

Database connectivity from Laravel will be tested when database work begins; the UI-first phase does not require database configuration yet.

## Bootstrap and Vite foundation — locally verified

The owner completed and browser-tested the initial Bootstrap integration locally on 2026-11-01:

- Installed bootstrap@5.3.8 and @popperjs/core@2.11.8 through npm.
- Installed the `bootstrap-icons` npm package; exact package version has not been recorded.
- Removed the Tailwind packages tailwindcss and @tailwindcss/vite.
- Updated vite.config.js to use the Laravel Vite plugin without the Tailwind Vite plugin.
- Updated resources/css/app.css to import Bootstrap CSS and `bootstrap-icons/font/bootstrap-icons.css`.
- Updated resources/js/app.js to import Bootstrap JavaScript.
- Ran npm run build successfully; Vite generated the production manifest and CSS/JS assets.
- Replaced the Laravel starter welcome view with a simple FlowerTrack IMS Bootstrap verification page.
- Confirmed in the browser at http://127.0.0.1:8000 that the page displays the FlowerTrack IMS heading and Bootstrap 5.3.8 Ready badge.

The Bootstrap foundation and initial Supplier/Admin UI were committed and pushed to the application repository. The initial Supplier/Admin and Client starter UI commit was `488a4eb` (`Build initial Supplier and Client UI layouts`). The owner subsequently reported completing a logo-only update push. The newer Bootstrap Icons changes described below are local and still require a build check and Git push. Client screens and backend/database work remain future work.

## Verified application repository structure

The following snapshot was inspected directly from the public application repository's `main` branch on 2026-11-01 (initial tree commit `ca10822eeea300e23042c5910697f8bc86210e9b`):

```text
flowertrack-ims/
├── app/
│   ├── Http/
│   │   └── Controllers/
│   │       └── Controller.php
│   ├── Models/
│   │   └── User.php
│   └── Providers/
│       └── AppServiceProvider.php
├── bootstrap/
│   ├── app.php
│   ├── cache/
│   └── providers.php
├── config/
│   ├── app.php
│   ├── auth.php
│   ├── cache.php
│   ├── database.php
│   ├── filesystems.php
│   ├── logging.php
│   ├── mail.php
│   ├── queue.php
│   ├── services.php
│   └── session.php
├── database/
│   ├── factories/
│   │   └── UserFactory.php
│   ├── migrations/
│   │   ├── 0001_01_01_000000_create_users_table.php
│   │   ├── 0001_01_01_000001_create_cache_table.php
│   │   └── 0001_01_01_000002_create_jobs_table.php
│   └── seeders/
│       └── DatabaseSeeder.php
├── public/
├── resources/
│   ├── css/
│   │   └── app.css
│   ├── js/
│   │   └── app.js
│   └── views/
│       └── welcome.blade.php
├── routes/
│   ├── console.php
│   └── web.php
├── storage/
├── tests/
│   ├── Feature/
│   │   └── ExampleTest.php
│   ├── Unit/
│   │   └── ExampleTest.php
│   └── TestCase.php
├── .editorconfig
├── .env.example
├── .gitattributes
├── .gitignore
├── .npmrc
├── AGENTS.md
├── CLAUDE.md
├── artisan
├── composer.json
├── composer.lock
├── package.json
├── phpunit.xml
├── vite.config.js
└── README.md
```

Current UI-relevant baseline:
- `resources/views/welcome.blade.php` is now a simple local Bootstrap verification page; reusable application layouts and feature screens have not yet been created.
- `resources/css/app.css` imports Bootstrap CSS in the owner's locally updated working copy; the public GitHub snapshot may still show the earlier starter version until pushed.
- `resources/js/app.js` imports Bootstrap in the owner's locally updated working copy; verify GitHub before assuming the change has been pushed.
- `routes/web.php` currently maps `/` to the `welcome` view.
- `app/Providers/AppServiceProvider.php` is still the default empty provider.
- No custom Supplier/Admin or Client views/layouts have been created yet.
- This is a snapshot of the inspected `main` branch, not a permanent inventory; re-check GitHub when the structure may have changed. The Bootstrap changes above are confirmed locally, not confirmed as pushed to GitHub.

### Repository inspection rule for AI assistance

Before asking the owner to paste or list application folders/files:
1. Inspect the latest application repository tree on GitHub (`main`, unless the owner specifies another branch).
2. Fetch the relevant source file directly from GitHub when its contents are needed.
3. Use the verified repository structure and the documentation as the baseline; do not repeatedly ask the owner to provide structure that can be inspected from the repository.
4. Ask the owner to paste code or output only when the relevant content is not available in the repository, local uncommitted changes may matter, or a command/test must be run on the owner's machine.
5. Clearly distinguish the committed GitHub version from any local changes that have not been pushed.

## Current UI implementation — Supplier/Admin dashboard (2026-11-01)

The owner has completed the first Supplier/Admin dashboard UI pass locally in `C:\\xampp\\htdocs\\FlowerTrack`. The following implementation details are confirmed from the working session:

- Created separate Blade layouts and views under `resources/views/layouts/` and `resources/views/supplier/`, including `supplier.blade.php` and `dashboard.blade.php`.
- Supplier sidebar uses Gemma's Flower Shop logo as the business/client branding; the logo is displayed without a white background.
- FlowerTrack IMS and the subtitle “Inventory Management System” were moved to the top navbar so the client business logo and application identity are visually distinct.
- Removed the navbar search field at the owner's request.
- Refined sidebar navigation spacing to be more compact.
- Kept the existing dashboard content/design after the owner requested reverting an alternative dashboard redesign; do not replace the current dashboard with the discarded redesign.
- Added two owner-supplied local image assets under `public/images/`: `flower-banner.jpg` for the welcome banner and `bloom-business.jpg` for the “Bloom Your Business” promotional panel. These are existing supplied images, not generated assets.
- The welcome banner uses the flower image on the right side, with the welcome text on the left; the image should preserve its aspect ratio and must not be stretched.
- Inventory Overview and Quick Actions panel spacing was adjusted to reduce excess empty space under Quick Actions.
- The owner confirmed the current dashboard design is acceptable as of 2026-11-01.

Relevant local files include `resources/css/app.css`, `resources/views/layouts/supplier.blade.php`, `resources/views/supplier/dashboard.blade.php`, and the two image assets in `public/images/`. These changes are local and must not be described as pushed until the application repository is checked and a push is confirmed.

### UI implementation rules learned

- Preserve approved UI and make targeted changes only; do not redesign unrelated dashboard sections when the owner asks to adjust one component.
- Keep Gemma's Flower Shop (business identity) separate from FlowerTrack IMS (application identity).
- Use the user's supplied images directly when requested; do not generate replacement images.
- When positioning banner imagery, preserve the original image proportions and use CSS background sizing/positioning intentionally.


### Bootstrap Icons standard — 2026-11-01

- **Confirmed icon standard:** Use Bootstrap Icons for interface/navigation icons throughout FlowerTrack IMS, including Supplier/Admin sidebar, Client sidebar, navbar, dashboard cards, panel headings, quick actions, and activity indicators.
- Installed the `bootstrap-icons` npm package and imported `bootstrap-icons/font/bootstrap-icons.css` in `resources/css/app.css` after the Bootstrap CSS import.
- Use the standard Bootstrap Icons markup pattern: `<i class="bi bi-icon-name"></i>`, typically inside the existing icon wrapper span so current layout styling is preserved.
- Do not use Unicode symbols or emoji as substitutes for interface icons. Emoji or images may remain when they are actual product imagery/content (for example, flower illustrations in the Top Selling Products list), not interface controls.
- The Supplier sidebar and Supplier Dashboard icon markup have been updated locally to use Bootstrap Icons. Preserve the approved dashboard layout, colors, chart, sample values, and spacing; only change the icon presentation unless the owner requests a broader redesign.
- Run `npm run build` after the changes. The Bootstrap Icons changes are pending application-repository commit/push until the owner runs and confirms the Git commands.

## UI-first rule

The user wants to finish and review the UI before starting backend and database implementation.

Agreed high-level sequence:
1. Environment setup.
2. Shared UI foundation and design system.
3. Supplier/Admin UI, following the supplied module order.
4. Client ecommerce-style UI.
5. UI review and finalization.
6. Backend and database.
7. Integration, algorithms, testing, and deployment.

Do not begin backend implementation while the UI-first phase is still in progress unless the user explicitly changes this decision.

## UI design rules

- Supplier/Admin Panel is a business-management interface.
- Client Panel must look and behave like an online flower shop.
- Use Bootstrap as the primary component/layout framework.
- Use Flexbox for custom alignment and layout.
- Use custom CSS for colors, font sizes, spacing, sizing, and brand-specific styling.
- Prefer reusable components and consistent patterns.
- Build one screen or component at a time, explain it, and review it before moving on.
- Do not invent or alter the official FlowerTrack IMS logo.
- The uploaded flower-shop dashboard screenshot is an approved **visual reference for the Supplier Side only**. Follow the visual direction and design prompt in `SUPPLIER_UI_DESIGN_REFERENCE.md`.
- Do not copy the reference business name/logo or treat that screenshot as the official FlowerTrack IMS logo. If the official FlowerTrack logo is supplied separately, use it as the primary brand reference.
- Keep the Client Panel distinct and ecommerce-oriented.

## Learning and implementation rules

1. Start each task by stating the goal and expected result.
2. Give the user the exact next action or code change.
3. Explain important code and commands.
4. Avoid skipping setup steps.
5. Do not silently add features, libraries, or architecture.
6. Ask for a decision when a business rule is unknown and materially affects implementation.
7. Mark assumptions clearly; do not present them as confirmed requirements.
8. Test each meaningful UI milestone before proceeding.
9. Keep the documentation synchronized with actual decisions and completed work; after every completed module, update the relevant Markdown before providing the application Git push script.
10. Work in small steps. Give only the current implementation step, wait for the owner's result, and then continue. Do not dump the entire setup or implementation plan as a sequence of actions to execute at once. At module completion, however, provide one complete, ready-to-run Git script for that module's build/check, staging, commit, push, and final status verification.

## Per-module completion, documentation, and Git workflow

This is the required workflow after **every completed module or independently reviewable feature**. Follow it consistently; do not batch several completed modules before documenting or pushing them.

1. **Complete and review the module.** Finish the requested UI/feature, preserve approved designs, and let the owner review it. Run the relevant build, test, or manual checks.
2. **Update documentation immediately after approval.** Update the relevant Markdown files to record what was completed, important decisions, files/areas changed, checks performed, remaining issues, and the next exact task. Update `DEVELOPMENT_PROGRESS.md` for progress/status changes and add a dated entry to `CHANGELOG.md`. Update `AI_CONTEXT.md`, `UI_UX_GUIDELINES.md`, or other requirement/design documents when the module establishes a reusable rule or changes a confirmed decision. Keep `README.md` current when the project status or workflow summary changes.
3. **Provide the complete application Git script.** Give the owner the exact commands to run from `C:\xampp\htdocs\FlowerTrack`: relevant build/test command(s), `git status`, targeted `git add` for the completed module, another `git status` review, a descriptive `git commit`, `git push origin main`, and a final `git status`. Use `git add -A` only when its scope is safe and ignored files have been verified. Never stage `.env`, `vendor/`, or `node_modules/`.
4. **Wait for the owner's terminal output and verify.** Inspect staged files before commit when needed. Do not claim commit/push success until the owner provides successful output or the remote state is independently verified. If build/tests fail, stop the Git sequence and resolve the issue first.
5. **Proceed only after confirmation.** Once the application push is confirmed and the final working tree is clean (or remaining changes are understood), identify the next module from the documented module order and begin it one step at a time.

The documentation repository and application repository are separate. Update and push the documentation repository when documenting the approved module, then provide the owner the application-repository script for the code changes. Keep both repository statuses distinct and never imply that updating one repository pushed the other.

## Application Git repository

- Local project: `C:\xampp\htdocs\FlowerTrack`.
- Git repository initialized locally; current branch: `main`.
- Initial commit: `ca10822` (`Initial Laravel project setup`).
- Remote `origin`: `https://github.com/wyntheprogrammer/flowertrack-ims.git`.
- Initial push succeeded; local `main` tracks `origin/main`.
- `.env`, `vendor`, and `node_modules` were not included in the initial commit based on the staged-file review.
- Git global identity configured by the owner: `wyn` / `wyntheprogrammer@gmail.com`.

## GitHub documentation update rule

Whenever a meaningful milestone is completed, a new technical concept is learned, or a project decision/requirement changes:
- Update the relevant Markdown document(s).
- Update `DEVELOPMENT_PROGRESS.md` only when project progress or status changes.
- Add an entry to `CHANGELOG.md`.
- Set the **Last updated** date using `YYYY-MM-DD`.
- Push the documentation changes to `flowertrack-ims-docs` after the user authorizes the push and repository access is available.

Never claim a GitHub push succeeded unless the host confirms it. If direct repository access is unavailable, prepare the changed files and provide the user with clear commit/push instructions.

## Continuity protocol

At the start of each resumed work session:
1. Read this file and `DEVELOPMENT_PROGRESS.md`.
2. Check the latest requirements and relevant module/design documents.
3. Identify the last completed task and current task.
4. Continue from the recorded next step rather than restarting or assuming progress.

At the end of each session, record:
- What was completed.
- What was learned or decided.
- Files changed.
- Tests or checks performed.
- Remaining issues.
- The next exact task.

## Accuracy rule

Only treat items marked **Confirmed** in the documentation as approved requirements. Items marked **To decide**, **Proposed**, or **Assumption** must not be implemented as settled business rules.
