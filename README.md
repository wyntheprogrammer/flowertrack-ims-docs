# FlowerTrack IMS Documentation

**Last updated:** 2026-11-02  
**Project status:** Laravel and Bootstrap setup and the initial shared UI/dashboard foundation are complete. The project is currently in **requirements planning**, not module UI implementation. Flower Management, Inventory Management, Batch Management, Spoilage Prediction, Client Management, Order Management, Delivery Management, and Sales and Payment Management are approved. Order fulfillment Stock-Out timing is finalized: Delivery on successful Delivered status and Customer Pickup on Ready for Pickup. Orders are rejected when stock available after reservations is insufficient; no pending-stock queue is used. Sales and Payment Management includes manual flower pricing, sales creation on order completion, Cash/Bank Transfer/E-wallet, Admin payment verification, partial/multiple payments, Admin-managed refunds, completed-order sales reporting less refunds, and daily-to-yearly PDF/Excel reports; **Reports is the next Supplier/Admin module to plan**. Remaining Supplier/Admin and Client module requirements must be finalized before proceeding with module UI implementation.

This repository contains the planning, requirements, technical references, design rules, workflows, setup instructions, and development progress for **FlowerTrack IMS**.

> This repository is documentation-only. The application source code is maintained separately in the public [`flowertrack-ims`](https://github.com/wyntheprogrammer/flowertrack-ims) repository. The local working copy is `C:\xampp\htdocs\FlowerTrack`.

## Development workflow

Current workflow: finalize and approve the requirements for all modules first, following the order in `docs/MODULES.md`. Do not start a module's UI merely because its requirements have been approved; the owner will confirm when the planning phase is complete. After planning, implement and review modules one at a time. After each completed and owner-approved module, update the relevant Markdown documentation first, then provide a complete copy-ready script to build/check, stage, commit, and push that module to the application repository. Verify the push before proceeding. The documentation repository and application repository are separate and are updated independently.

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

The application repository is initialized on branch `main`. The initial Laravel skeleton and the first Supplier/Admin and Client starter UI files have been pushed; the initial UI commit is `488a4eb`. The owner also confirmed the later logo update and Bootstrap Icons refinement build/push. These foundation/dashboard milestones do not mean module UI implementation is currently underway. The current work is requirements planning, with Sales and Payment Management next. See [`docs/DEVELOPMENT_PROGRESS.md`](docs/DEVELOPMENT_PROGRESS.md) for the current phase and next task.

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
