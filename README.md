# FlowerTrack IMS Documentation

**Last updated:** 2026-11-17  
**Payment policy:** One exact full payment per order; no partial payments or installments.  
**Communication:** Address the owner as **Wyn** in every response; see [AI Context](docs/AI_CONTEXT.md#communication-preference--wyn-2026-11-16).

**Project status:** Approved storage-condition capture and dropdowns: [`3db137b`](https://github.com/wyntheprogrammer/flowertrack-ims/commit/3db137b3a12194d9ba8a56c7f76c5426492f39e9) (published and verified on main). Agent full suite: 109 passed / 1,467 assertions. Completed-day observation collection is next.

This repository contains the planning, requirements, technical references, design rules, workflows, setup instructions, and development progress for **FlowerTrack IMS**.

> This repository is documentation-only. The application source code is maintained separately in the public [`flowertrack-ims`](https://github.com/wyntheprogrammer/flowertrack-ims) repository. The local working copy is `C:\xampp\htdocs\FlowerTrack`.


## Database and backend specialists — 2026-11-05

- [Database Engineer](docs/DATABASE_ENGINEER_SUBAGENT.md): MariaDB/MySQL schema, migrations, constraints/indexes, quantity/money integrity and transaction design.
- [Backend Engineer](docs/BACKEND_ENGINEER_SUBAGENT.md): Laravel/PHP authentication, authorization, validation, persistence, secure uploads and feature tests.
- Owner explicitly requested both agents; they were spawned as `database_engineer` and `backend_engineer`, received project rules, acknowledged their roles and are awaiting implementation assignments.
- Primary agent coordinates schema/API contracts and reviews combined work. No application or database changes were performed merely by creating these roles.
- These files preserve reusable instructions; they do not persist a running agent across chats. Resume/recreate agents explicitly when appropriate.


## Development workflow

Current workflow: implement and review Supplier/Admin modules one at a time in the order in `docs/MODULES.md`. Flower Management is the approved module-page visual reference; preserve the approved dashboard/shared shell. After each owner-approved module, update the relevant Markdown first, provide the complete application build/check + staging + commit + push script, and verify push success before starting the next module. Finish Supplier UI before Client UI; backend/database implementation follows full UI review. Documentation and application repositories are updated independently.

## Start here

1. [`docs/SETUP_GUIDE.md`](docs/SETUP_GUIDE.md) — complete Windows setup manual with official installer download links, installation/configuration steps, PHP extensions and PATH checks, compatibility notes, and verification commands.
2. [`docs/AI_CONTEXT.md`](docs/AI_CONTEXT.md) — instructions for AI-assisted development, verified application structure, and continuity.
3. [`docs/PROJECT_OVERVIEW.md`](docs/PROJECT_OVERVIEW.md) — project purpose and scope.
4. [`docs/REQUIREMENTS.md`](docs/REQUIREMENTS.md) — confirmed requirements and open questions.
5. [`docs/DEVELOPMENT_PLAN.md`](docs/DEVELOPMENT_PLAN.md) — agreed implementation order.
6. [`docs/DEVELOPMENT_PROGRESS.md`](docs/DEVELOPMENT_PROGRESS.md) — current progress, installed environment, and next task.
7. [`docs/UI_UX_GUIDELINES.md`](docs/UI_UX_GUIDELINES.md) — interface and styling rules.
8. [`docs/SUPPLIER_UI_DESIGN_REFERENCE.md`](docs/SUPPLIER_UI_DESIGN_REFERENCE.md) — approved Supplier Side dashboard visual reference and design prompt.
9. [`docs/TECHNOLOGY_STACK_AND_REFERENCES.md`](docs/TECHNOLOGY_STACK_AND_REFERENCES.md) — planned technologies, actual local setup, and useful repositories.

10. [`docs/UI_UX_DESIGNER_SUBAGENT.md`](docs/UI_UX_DESIGNER_SUBAGENT.md) — reusable UI/UX designer subagent role, project constraints, current Flower Management layout brief, and reuse prompt.

For the reusable approved module-page pattern, see [UI/UX Guidelines](docs/UI_UX_GUIDELINES.md#approved-module-ui-reference--flower-management-2026-11-03).

## Current remaining work — reviewed 2026-11-17

Storage capture is complete. Next: completed-day observation collection, followed by day closing, Dashboard prediction alerts and final browser integration verification. The separate Client storefront and real-data ML/deployment validation remain later work. Full evidence and priorities: [Current Review](docs/DEVELOPMENT_PROGRESS.md#current-review-and-remaining-work--2026-11-16).

Older dated UI-only sections are historical. Do not restart already saved Supplier backends or treat placeholder Client pages as a working storefront.

## Storage-condition capture approved — 2026-11-17

- Wyn reviewed the storage workflow, requested dropdowns, and approved committing/pushing the application and updating this documentation.
- Stock-In requires actual storage condition. Stock-In and Batch Edit use Ambient/Chilled dropdowns; existing saved custom values are retained as an option when editing. Batch Edit permits Unknown (null). Backend validation remains required string, maximum 30 characters for new receipts; optional/nullable string, maximum 30 characters for metadata edits. The dropdown change does not add an enum restriction to the API.
- Inventory and Batch detail responses/display include storage. Receipt corrections display storage without editing it; changes to existing storage go through Batch Management.
- Storage is saved on the existing inventory_batches column and included in receipt/metadata snapshots, batch version checks and audit safe fields. Edits append zero-quantity metadata history and do not alter stock, reservations or historical daily observations. No migration or inferred legacy storage was added.
- Required repository setup installed Laravel Boost v2.10.4 and generated AGENTS.md/CLAUDE.md guidelines plus boost.json; dependency lock changes are included.
- Agent-run verification: full suite **109 passed / 1,467 assertions**; dedicated Batch suite **9 passed / 146 assertions**; Pint, Composer validation, production build and diff whitespace checks passed. Build retains the known image-reference and large-chunk warnings. Wyn's acceptance is for this storage UI/workflow, not a new claim of completed forecasting browser acceptance or real-world accuracy.
- Application checkpoint: [`3db137b`](https://github.com/wyntheprogrammer/flowertrack-ims/commit/3db137b3a12194d9ba8a56c7f76c5426492f39e9) (published and verified on main).
- **Next:** trustworthy completed-day observation collection and reconciliation, then reviewed day closing/catch-up. Dashboard current-risk alerts, final automatic-refresh browser acceptance, Client storefront and real-data/deployment validation remain separate work.

## Previous checkpoint — 2026-11-16

- Application: [`46e4145`](https://github.com/wyntheprogrammer/flowertrack-ims/commit/46e4145), automatic Spoilage page refresh with one Python batch process, input-based cache reuse, concurrency protection and corrected SQLite date lookup.
- Owner full tests: **107 passed / 1,439 assertions**; dedicated Spoilage tests: **9 passed / 62 assertions**. Build passed with known image/chunk warnings.
- Python environment is recreated per device using committed requirements.txt; .venv is ignored (`f213583`). See [Setup Guide](docs/SETUP_GUIDE.md#automatic-spoilage-page-refresh--2026-11-16).
- Refresh is asynchronous and browser-triggered; it is not a scheduled worker. Daily observation collection still needs implementation/review. Final automatic-flow browser acceptance is not inferred from passing service tests.
- Older sections below are historical; current scope and evidence are in [Development Progress](docs/DEVELOPMENT_PROGRESS.md).

## Local setup snapshot

- XAMPP: `C:\xampp`
- PHP: 8.3.36
- Composer: 2.11.1
- Laravel application skeleton: 13.11.0
- Bundled database: MariaDB 10.11.16
- Node.js: 26.13.0
- npm: 11.20.0
- Git: 2.52.0.windows.1
- Local project: `C:\xampp\htdocs\FlowerTrack`
- Laravel development URL (when `php artisan serve` is running): `http://127.0.0.1:8000`

The application repository is initialized on branch `main`. The initial Laravel skeleton and the first Supplier/Admin and Client starter UI files have been pushed; the initial UI commit is `488a4eb`. The owner also confirmed the later logo update and Bootstrap Icons refinement build/push. Flower Management commit `3f580e5` has been verified on GitHub. Inventory Management commit `e758bf8` is also verified on GitHub. Batch Management UI is owner-approved and its application push is verified (`4472a2a8f471b81ec91be03db11bb67e3f8d861f`). Supplier navigation setup is accepted; its application push still needs verification. Spoilage Prediction UI is approved with sample advisory data and typography matched to Flower Management. Its application build/commit/push remains unverified; Supplier Client Management UI is next after push verification. See [`docs/DEVELOPMENT_PROGRESS.md`](docs/DEVELOPMENT_PROGRESS.md) for the current phase and next task.

## Repository structure

This is the **documentation repository** structure:

```text
flowertrack-ims-docs/
├── README.md
├── docs/
│   ├── SETUP_GUIDE.md
│   ├── AI_CONTEXT.md
│   ├── PROJECT_OVERVIEW.md
│   ├── REQUIREMENTS.md
│   ├── SYSTEM_ARCHITECTURE.md
│   ├── TECHNOLOGY_STACK_AND_REFERENCES.md
│   ├── MODULES.md
│   ├── DATABASE_SCHEMA.md
│   ├── BUSINESS_LOGIC.md
│   ├── WORKFLOWS.md
│   ├── UI_UX_DESIGNER_SUBAGENT.md
│   ├── UI_UX_GUIDELINES.md
│   ├── SUPPLIER_UI_DESIGN_REFERENCE.md
│   ├── DEVELOPMENT_PLAN.md
│   ├── DEVELOPMENT_PROGRESS.md
│   └── CHANGELOG.md
└── assets/
    └── ui-references/
        └── README.md
```

The separate application repository is [`wyntheprogrammer/flowertrack-ims`](https://github.com/wyntheprogrammer/flowertrack-ims). Its verified current Laravel starter structure, including the existing frontend files, is recorded in [`docs/AI_CONTEXT.md`](docs/AI_CONTEXT.md). The live application repository should be checked again when its structure may have changed; AI assistance should inspect it before asking the owner to list committed folders or files.

The Supplier Side dashboard reference and the owner's design instructions are documented in `docs/SUPPLIER_UI_DESIGN_REFERENCE.md`. The screenshot is a visual reference, not the official FlowerTrack IMS logo.
