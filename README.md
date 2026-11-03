# FlowerTrack IMS Documentation

**Last updated:** 2026-11-03  
**Project status:** Flower Management, Inventory Management and Batch Management UI are owner-approved. Flower Management remains the visual reference for Supplier module screens. Batch Management application push is verified (`4472a2a`). Supplier sidebar links and named routes are accepted, with Coming Soon pages for upcoming modules; navigation runtime checks and application push remain unverified. Spoilage Prediction UI is next. Batch Management has View/Edit actions, search without the removed filter row, and table-only horizontal scrolling. Inventory spoilage Stock-Out owns the agreed future batch Spoiled-status update. Client UI and backend/database work remain deferred under the UI-first process.

This repository contains the planning, requirements, technical references, design rules, workflows, setup instructions, and development progress for **FlowerTrack IMS**.

> This repository is documentation-only. The application source code is maintained separately in the public [`flowertrack-ims`](https://github.com/wyntheprogrammer/flowertrack-ims) repository. The local working copy is `C:\xampp\htdocs\FlowerTrack`.

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

The application repository is initialized on branch `main`. The initial Laravel skeleton and the first Supplier/Admin and Client starter UI files have been pushed; the initial UI commit is `488a4eb`. The owner also confirmed the later logo update and Bootstrap Icons refinement build/push. Flower Management commit `3f580e5` has been verified on GitHub. Inventory Management commit `e758bf8` is also verified on GitHub. Batch Management UI is owner-approved and its application push is verified (`4472a2a8f471b81ec91be03db11bb67e3f8d861f`). Supplier navigation setup is accepted; its application push still needs verification. Spoilage Prediction UI is next. See [`docs/DEVELOPMENT_PROGRESS.md`](docs/DEVELOPMENT_PROGRESS.md) for the current phase and next task.

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
