---
name: web-design
description: Josue's visual design system AND new-app workflow — high-end, award-winning aesthetic with mesh gradients, glassmorphism, WCAG 2.1 AA accessibility, and real copy (no lorem ipsum). Trigger this whenever Josue asks to generate, scaffold, or build a new app or page — even with no design direction given ("build me an app for X") — since the default workflow is to produce an aesthetic HTML/JS demo first, then implement the real thing informed by it. Also trigger for any styling task on existing Templ + HTMX + DaisyUI UI, landing pages, dashboards, or marketing sites, and for standalone single-file HTML demos. Consult this before defaulting to plain DaisyUI defaults, generic Tailwind boilerplate, or jumping straight to Templ components without a design pass first.
---

# Josue's Web Design System & New-App Workflow

Adapted from a prompt Josue previously ran manually through a separate tool (`deepsitepromptgen`) to produce a standalone HTML/JS demo, which he'd then hand to a coding agent as visual inspiration. This skill folds both steps into one: when asked to generate a new app, do the demo generation *and* the real build, without being asked twice.

## Workflow: generating a new app

When asked to build/scaffold a new app (not just add a feature to an existing one), follow this two-phase process rather than jumping straight to Templ components:

### Phase 1 — Aesthetic demo (design exploration)

Produce a **single-file HTML/JS demo** of the app's key pages, following the "One-off static HTML demo" spec below. This is throwaway — CDN-loaded Tailwind/DaisyUI, no backend, no real data — its only job is to nail the visual direction fast: layout, palette, typography, hero treatment, and page-to-page feel for the pages that matter most (landing/home, and 1-2 core app screens if known).

Show this to Josue (as an artifact/preview) before moving to Phase 2, unless he's explicitly said to skip straight to the real build.

### Phase 2 — Real implementation

Translate the demo into the actual stack: Templ components, HTMX interaction (per the "Interaction" section below), DaisyUI theme config matching the demo's palette/typography, wired to real routes/handlers/EzAuth/RiverQueue as needed. The demo is a visual reference, not code to reuse verbatim — rebuild it properly componentized.

Skip Phase 1 entirely for small changes to an existing app (a new form, a new endpoint, a style tweak) — this workflow is for net-new apps or substantial new page sets, not incremental work.

## Design philosophy

Build a "high-end," award-worthy look: generous whitespace, consistent spacing scale, confident typography, and a strong sense of depth. Avoid flat, generic SaaS-template looks — the goal is something that feels designed, not scaffolded.

## Visual direction

- **Color palette**: Derive a small, professional palette from the product's name/purpose rather than defaulting to DaisyUI's stock theme colors untouched. Configure a custom DaisyUI theme (via `@plugin "daisyui/theme"` or the theme config) rather than leaving `data-theme` on a built-in preset when brand identity matters.
- **Contrast (non-negotiable)**: All text/background combinations must meet **WCAG 2.1 AA — minimum 4.5:1 contrast ratio**. Avoid subtle grey-on-white text that fails this. Check custom theme colors against this before shipping.
- **Typography**: Import quality Google Fonts — a distinct display font for headings, a highly readable sans-serif for body text. Set both in the Tailwind/DaisyUI theme config, not inline per-component.
- **Imagery**: Use high-quality, relevant placeholder imagery during prototyping (e.g. Unsplash) — swap for real assets before ship.
- **Icons**: Use a consistent icon set throughout (Heroicons or Lucide pair well with DaisyUI/Tailwind).

## Depth and background treatment

Never ship flat solid-color backgrounds on hero/marketing sections. Use at least one of:

- **Mesh gradients / soft blobs**: layered radial gradients or a subtly animated blob shape behind hero content for depth and movement.
- **Glassmorphism**: `backdrop-filter: blur(12px)` with a semi-transparent background on cards, nav bars, and modals so color bleeds through elegantly. DaisyUI's `card` and `navbar` components are good candidates for this treatment.
- **Subtle texture**: low-opacity SVG patterns (topographic lines, grid, noise) behind content sections for tactile quality — never so strong it competes with text.
- **Section transitions**: angled `clip-path` dividers or SVG wave shapes between sections instead of hard horizontal lines.

## Structure & accessibility

- Semantic HTML5 throughout: `<header>`, `<main>`, `<article>`, `<footer>`, proper heading hierarchy (`h1 → h2 → h3`, never skipped).
- **Focus indicators**: never remove default focus outlines without replacing them with an equally visible custom focus state (thick ring, border color shift). Every interactive element must be clearly visible via keyboard navigation.
- **ARIA**: `aria-label`/`aria-labelledby` on any icon-only interactive element (buttons with no visible text).
- **JSON-LD**: include a `<script type="application/ld+json">` Schema.org block (`Organization`, `Product`, `LocalBusiness`, etc. as appropriate) describing the page/product for SEO.
- Mobile-first responsive layout.

## Interaction (HTMX-appropriate — Phase 2 real build only)

The original prompt targeted a fully client-rendered SPA with JS-driven section switching — that's fine for the throwaway Phase 1 demo, but the real app should use **server-rendered progressive enhancement** instead:

- Use `hx-boost` on nav links for SPA-like transitions without hand-written JS routing.
- Swap fragments with `hx-get`/`hx-post` + `hx-target`/`hx-swap` for in-page updates (forms, filters, tabs) rather than full-page JS state machines.
- Active nav state can be set server-side per request (Templ conditional class) rather than reproduced in client JS.
- Toast notifications: implement as a small Templ partial swapped via `hx-swap-oob`, styled with DaisyUI's `toast` + `alert` components, rather than a hand-rolled JS toast system.
- Keep transitions smooth (`transition: all 0.2s ease` equivalent via Tailwind transition utilities) on interactive elements.

## Content

- **No lorem ipsum, ever.** Generate realistic, persuasive, product-specific copy — even for prototypes and Phase 1 demos.

## One-off static HTML demo spec (Phase 1, or any explicit throwaway request)

Single HTML file, all CSS/JS inline, CDN-loaded CSS framework (Tailwind via script tag for rapid prototyping, or Bootstrap 5+ if requested). JS-based section show/hide for SPA-like nav (no page reloads, dynamic active nav state). Apply all of the Visual direction, Depth and background treatment, Structure & accessibility, and Content sections above. Include the JSON-LD block. Output just the complete HTML.
