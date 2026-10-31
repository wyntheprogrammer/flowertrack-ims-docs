# UI/UX Guidelines

**Last updated:** 2026-11-01

## 1. Overall design direction

- Platform: web application.
- Target: desktop-first and responsive.
- Build the UI first and complete/review it before backend development.
- Supplier/Admin and Client must have clearly different user experiences.
- Aim for a polished, professional, usable commercial product.
- Prioritize clarity, hierarchy, consistency, and readability over decoration.

## 2. Supplier/Admin Panel

- Management-oriented interface.
- Follow the owner-provided module order in `MODULES.md`.
- Use clear navigation, data tables, filters, forms, status indicators, and dashboard visualizations where relevant.
- Each module screen must be designed and reviewed individually.
- Exact visual identity, colors, logo, fonts, and components are still to be defined.

## 3. Client Panel

- Ecommerce / online flower shop experience.
- Distinct from the Supplier/Admin management interface.
- Product browsing and shopping should be intuitive.
- Product cards, catalog filters, cart, order views, and tracking should be designed as relevant.
- Exact storefront style and checkout details remain to be decided.

## 4. Styling technology rules

| Tool | Intended use |
|---|---|
| Bootstrap | Primary UI components, grid, utilities, and responsive behavior |
| Flexbox | Custom alignment and layout |
| Custom CSS | Colors, font sizes, spacing, dimensions, typography, and visual details |
| HTML | Semantic page structure |
| JavaScript | Interactive behavior |

Bootstrap is the primary UI framework preference. Do not add Tailwind or another competing UI framework without discussing the reason first.

## 5. Design system to establish during UI Foundation

- Brand color tokens (not yet selected).
- Neutral/background/surface colors.
- Typography family and type scale.
- Spacing scale.
- Border, radius, and shadow rules.
- Button variants and sizes.
- Form controls and validation states.
- Table styles and empty/loading/error states.
- Responsive breakpoints and navigation behavior.
- Reusable layout and component patterns.

## 6. Reference image status

The previously mentioned external Supplier dashboard reference image is **excluded for now** at the owner's request. Do not use it, derive colors from it, or include it as a required project asset unless the owner later reintroduces it.

## 7. UI completion criteria

A UI phase is complete only when:
- All planned screens for that phase have been created.
- Navigation between screens is coherent.
- Responsive behavior has been checked.
- Reusable elements are consistent.
- Empty, loading, validation, and error states are considered where relevant.
- The owner has reviewed and approved the visual direction.
- Progress and relevant design decisions are documented with a date.
