## 6. Kit Tokens & Global Design System

Bridge Elementor's Site Settings (global colors/fonts) into the theme CSS
so both layers share a single source of truth for the brand palette.

### The problem

Elementor prints `--e-global-color-*` on `.elementor-kit-<id>` (a class on `<body>`),
not on `:root`. Tailwind prints its `--color-*` defaults on `:root` / `:host`.
A `var()` chain evaluated on `<html>` resolves to its fallback, not the kit value.

### Solution: kit-tokens.css

Create `assets/css/kit-tokens.css`, enqueued BEFORE `main.css`:

```css
html, body, .elementor-kit-6 {
    --vm-color-primary:   var(--e-global-color-primary,   #2e6647);
    --vm-color-secondary: var(--e-global-color-secondary, #0f2419);
    --vm-color-text:      var(--e-global-color-text,      #242a26);
    --vm-color-accent:    var(--e-global-color-accent,    #d4af37);
}
```

Each alias keeps the pre-kit hex as its `var()` fallback, so a site without
globals renders as before.

### Tailwind integration

In `tailwind.src.css`, point the colour tokens at the bridge variables:

```css
@theme {
    --color-jungle-900: var(--vm-color-secondary, #0f2419);
    --color-jungle-700: var(--vm-color-primary,   #2e6647);
    --color-gold-500:   var(--vm-color-accent,    #d4af37);
}
```

Every compiled utility therefore emits the token, not the hex:
`.text-jungle-900{color:var(--color-jungle-900)}`

### Propagation verification

Change a kit colour, measure computed styles on `<body>` (not `<html>`):

| State | `--vm-color-primary` | Gallery chip colour |
|-------|---------------------|--------------------|
| Before | `#2e6647` | `rgb(15, 36, 25)` |
| Kit primary `#ff00aa` | `#ff00aa` | still secondary |
| Kit secondary `#ff00aa` | `#ff00aa` | `rgb(255, 0, 170)` |
| Reverted | `#2e6647` | `rgb(15, 36, 25)` |

**Read tokens off `<body>`, not `<html>`.** Tailwind's defaults resolve on `:root`,
but kit values only exist on `<body class="elementor-kit-6">`.

### Remaining hardcoded colours

~330 literal hexes in markup as arbitrary-value Tailwind classes:
`bg-[#F5F0E6]`, `text-[#242A26]`, `border-[#0F2419]/30`

These compile to literal `oklab(...)` that the kit cannot reach.
Rewriting them to palette utilities (`bg-sand-100`, `text-jungle-900`) puts
those surfaces under Site Settings control.

### Elementor MCP global variables

Use `elementor_manage_global_variable` to create design tokens:
```json
{"action": "create", "type": "global-color-variable", "label": "wc26-gold", "value": "#C6A15B"}
```

Reference in `style` strings as `var(--wc26-gold)` (NOT `var(--e-gv-wc26-gold)`).