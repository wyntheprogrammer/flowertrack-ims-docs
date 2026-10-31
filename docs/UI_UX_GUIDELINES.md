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
- Follow the approved visual direction and design prompt in `SUPPLIER_UI_DESIGN_REFERENCE.md`.
- Use a distinctive, polished floral/botanical identity while keeping the interface practical and readable.
- Do not copy the reference shop's name, logo, business identity, or sample content.
- The reference screenshot is not the official FlowerTrack IMS logo. If the owner supplies the official logo separately, use it as the primary brand reference and do not alter it.

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

- Brand color tokens informed by the official FlowerTrack logo when available.
- Neutral/background/surface colors.
- Typography family and type scale.
- Spacing scale.
- Border, radius, and shadow rules.
- Button variants and sizes.
- Form controls and validation states.
- Table styles and empty/loading/error states.
- Responsive breakpoints and navigation behavior.
- Reusable layout and component patterns.
- Consistent chart, status, and data-visualization treatments.

## 6. Supplier Side reference image and prompt

The owner has approved the uploaded flower-shop dashboard screenshot and supplied design prompt as a **visual reference for the Supplier Side**. This newer approval supersedes the earlier exclusion.

- Refer to `SUPPLIER_UI_DESIGN_REFERENCE.md` for the image, prompt requirements, visual cues, brand safeguards, and application scope.
- Use the image as inspiration for the botanical atmosphere, green/dark navigation, light content surfaces, floral imagery, dashboard hierarchy, and professional finish.
- Aim for an original FlowerTrack IMS interface, not a pixel-for-pixel recreation.
- The screenshot's shop name and logo are not FlowerTrack IMS branding.
- If the official FlowerTrack logo is supplied separately, analyze its colors and personality to establish the primary brand palette. Do not redesign, distort, or change that logo.
- When a task specifically requests a UI image, the supplied prompt calls for one complete high-fidelity image, without variants or accompanying explanation.
- This reference does not change the Client Panel direction, module order, or any operational business rule.

## 7. UI completion criteria

A UI phase is complete only when:
- All planned screens for that phase have been created.
- Navigation between screens is coherent.
- Responsive behavior has been checked.
- Reusable elements are consistent.
- Empty, loading, validation, and error states are considered where relevant.
- The owner has reviewed and approved the visual direction.
- Progress and relevant design decisions are documented with a date.
