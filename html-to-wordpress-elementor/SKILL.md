---
name: html-to-wordpress-elementor
description: "Convert HTML/React mockups to WordPress + Elementor."
version: 0.1.0
author: Hermes
metadata:
  hermes:
    tags:
      - WordPress
      - Elementor
      - Migration
      - Frontend
---

# HTML/React Mockup to WordPress + Elementor Conversion

End-to-end procedure for converting static HTML, React/Vite, or any website mockup
into a fully editable WordPress site built with Elementor widgets and Theme Builder
templates. Derived from the Villa Massis restaurant project — a real multi-phase
migration from Tailwind/PHP partials to native Elementor V4 atomic elements.

This skill does NOT cover WordPress installation or Elementor licensing.

## When to Use

- Converting an HTML prototype or React SPA to WordPress + Elementor
- Migrating a PHP theme's hardcoded markup into Elementor-editable widgets
- Replacing Tailwind utility classes with Elementor style controls
- Building Theme Builder templates (header/footer/single/archive) from PHP shells
- Creating custom Elementor widget plugins that delegate rendering to theme partials
- Using the Elementor MCP tools (`build-composition`, `manage-elements`, etc.)

## Prerequisites

- WordPress install with Elementor (+ Pro for Theme Builder features)
- Elementor MCP server connected (for programmatic page building)
- The mockup source files (HTML, CSS, JS, React components, images)
- `wp` CLI access for database operations and plugin management

## Architecture Overview

The conversion follows a layered architecture:

```
Mockup HTML/React
  ↓ decompose into sections
Theme partials (template-parts/blocks/*.php)   ← ALL markup lives here
  ↑ called by
Elementor custom widgets (vm_* plugin)         ← control panel only, no markup
  ↑ via trait: settings → args → vm_the_block()
Theme Builder templates (header/footer/single)  ← Elementor-managed shells
  ↑ fallback to PHP when no template matches
Elementor V4 atomic elements (native widgets)   ← final state for full editability
```

## Quick Reference

| Phase | What | Key files/tools |
|-------|------|-----------------|
| 1. Audit | Inventory all markup, classes, content | `search_files`, `read_file` |
| 2. Theme partials | Extract sections into `template-parts/blocks/` | `write_file` |
| 3. Widget plugin | Create Elementor widgets with controls | Plugin PHP files |
| 4. Component defaults | Seed repeater rows from mockup data | `inc/component-defaults.php` |
| 5. Style controls | Bind CSS to `{{WRAPPER}}` selectors | `trait-vm-widget-base.php` |
| 6. Theme Builder | Migrate header/footer/single/archive | `elementor/manage-site-parts` MCP |
| 7. Native rebuild | Replace shortcode bridges with V4 widgets | `elementor/build-composition` MCP |
| 8. Kit tokens | Bridge global colours/fonts to theme CSS | `kit-tokens.css` |
| 9. Verification | Parity check before vs after | Browser probes, `terminal` |

## Procedure

See reference files for detailed procedures per phase:

1. **Audit & decomposition** — `skill_view(file_path="references/01-audit-decomposition.md")`
2. **Widget plugin architecture** — `skill_view(file_path="references/02-widget-plugin.md")`
3. **Style controls & CSS binding** — `skill_view(file_path="references/03-style-controls.md")`
4. **Theme Builder migration** — `skill_view(file_path="references/04-theme-builder.md")`
5. **Elementor MCP tools** — `skill_view(file_path="references/05-elementor-mcp.md")`
6. **Kit tokens & global design** — `skill_view(file_path="references/06-kit-tokens.md")`
7. **Verification & parity** — `skill_view(file_path="references/07-verification.md")`
8. **Pitfalls & lessons** — `skill_view(file_path="references/08-pitfalls.md")`

## Pitfalls (Top-Level)

- Never put widget markup in the plugin — ALL markup belongs in theme partials
- Elementor style controls with non-empty defaults emit CSS rules; use empty/inert defaults
- Theme Builder `need_override_location` filter must return false or Elementor replaces your template hierarchy
- `post-8.css` md5 must stay identical after adding controls — verify byte-for-byte
- Always nest `section > column > widget` when probing widget CSS (not widget directly in section)
- Tailwind v4 utilities inside `@layer utilities` lose to unlayered Elementor resets — add `revert-layer` rules

## Verification

Run parity check: capture `document.body.innerText`, element counts, and `scrollHeight` before and after each migration phase. Text hash must be identical; structural differences (wrapper elements, hidden nav copies) must be accounted for.
