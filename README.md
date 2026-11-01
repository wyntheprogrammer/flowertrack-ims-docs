# FlowerTrack IMS Documentation

**Last updated:** 2026-11-01  
**Project status:** Initial Laravel application setup committed and pushed to the public GitHub application repository. UI development has not started.

This repository contains the planning, requirements, technical references, design rules, workflows, and development progress for **FlowerTrack IMS**.

> This repository is documentation-only. The application source code is maintained separately in the public [`flowertrack-ims`](https://github.com/wyntheprogrammer/flowertrack-ims) repository. The local working copy is `C:\xampp\htdocs\FlowerTrack`.

## Start here

1. [`docs/AI_CONTEXT.md`](docs/AI_CONTEXT.md) — instructions for AI-assisted development, verified application structure, and continuity.
2. [`docs/PROJECT_OVERVIEW.md`](docs/PROJECT_OVERVIEW.md) — project purpose and scope.
3. [`docs/REQUIREMENTS.md`](docs/REQUIREMENTS.md) — confirmed requirements and open questions.
4. [`docs/DEVELOPMENT_PLAN.md`](docs/DEVELOPMENT_PLAN.md) — agreed implementation order.
5. [`docs/DEVELOPMENT_PROGRESS.md`](docs/DEVELOPMENT_PROGRESS.md) — current progress, installed environment, and next task.
6. [`docs/UI_UX_GUIDELINES.md`](docs/UI_UX_GUIDELINES.md) — interface and styling rules.
7. [`docs/SUPPLIER_UI_DESIGN_REFERENCE.md`](docs/SUPPLIER_UI_DESIGN_REFERENCE.md) — approved Supplier Side dashboard visual reference and design prompt.
8. [`docs/TECHNOLOGY_STACK_AND_REFERENCES.md`](docs/TECHNOLOGY_STACK_AND_REFERENCES.md) — planned technologies, actual local setup, and useful repositories.

## Local setup snapshot

- XAMPP: `C:\xampp`
- PHP: 8.3.36
- Composer: 2.11.1
- Laravel application skeleton: 13.11.0
- Bundled database: MariaDB 10.11.16
- Local project: `C:\xampp\htdocs\FlowerTrack`
- Laravel development URL (when `php artisan serve` is running): `http://127.0.0.1:8000`

The application repository is initialized on branch `main`; its initial Laravel skeleton commit (`ca10822`) has been pushed successfully. See [`docs/DEVELOPMENT_PROGRESS.md`](docs/DEVELOPMENT_PROGRESS.md) for the current phase and next task. Node.js 26.13.0, npm 11.20.0, and Git 2.52.0.windows.1 are installed and verified. Git identity is configured as `wyn` (`wyntheprogrammer@gmail.com`).

## Repository structure

This is the **documentation repository** structure:

```text
flowertrack-ims-docs/
├── README.md
├── docs/
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
