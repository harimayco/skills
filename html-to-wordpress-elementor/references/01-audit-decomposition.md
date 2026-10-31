## 1. Audit & Decomposition

Before writing any code, inventory every visual section, its markup origin,
its styling approach, and its content source. This determines the migration
order and the widget/partial boundaries.

### Step 1: Map every renderable section

For each visible section of the mockup, record:

| Field | What to capture |
|-------|-----------------|
| Section name | Hero, About, Menu, Gallery, Footer, etc. |
| Markup source | Which file(s) produce this HTML |
| Content source | Hardcoded? CMS? API? JS-rendered? |
| Styling | Tailwind classes, inline styles, CSS modules, styled-components |
| Interactivity | Accordion, slider, modal, filter, form |
| Images/media | Paths, sizes, formats, lazy loading |

Use `search_files` to find all template/component files.

### Step 2: Identify content layers (3 possible sources)

1. **Theme partial defaults** — `wp_parse_args()` literals in PHP partials
2. **Elementor widget settings** — repeater rows, text controls
3. **Theme options** — site-wide constants (phone, address, hours)

Rule: the partial literal is the REAL default. Elementor control defaults
must match the partial defaults exactly, or they drift.

### Step 3: Classify each section for migration path

| Type | Migration path |
|------|---------------|
| Content section (hero, about, menu) | Custom widget + theme partial |
| Chrome (header, footer) | Theme Builder template |
| Template shell (single, archive, page) | Theme Builder template |
| Utility (404, search) | Theme Builder or leave as PHP |

### Step 4: Decompose mockup HTML into partial boundaries

Each `<section>` (or logical block) becomes one theme partial:
- `template-parts/blocks/<name>.php` for content sections
- `template-parts/header/navbar.php` for the site header
- `template-parts/footer/footer.php` for the site footer
- `template-parts/content/single.php` for single post layout

Partials must render with ZERO args (theme fallback path):
```php
$args = wp_parse_args( $args ?? array(), array(
    'heading'   => 'Default Heading',
    'items'     => vm_component_default( 'items' ),
));
```

### Step 5: Extract hardcoded content into component-defaults

Create `inc/component-defaults.php` — single source feeding both the partial's
`wp_parse_args()` fallback and the widget's repeater `default` via
`vm_component_default_control_rows()`.

### Step 6: Adapt content shapes for Elementor controls

Two control types need reshaping from the partial's native format:
- `MEDIA` control wants `array('url' => ..., 'id' => ...)` where partial holds a URL string
- Specials `offers` field is a textarea of pipe-delimited lines where partial holds row arrays

Build `vm_component_default_control_rows($name)` to handle these conversions.

### Step 7: Inventory CSS approach

Classify every CSS rule:
- **Tailwind utilities** → keep as theme substrate
- **Element-ID selectors** (`.elementor-element-<id>`) → replace with stable selectors (`:has()`, anchors, structural)
- **Hex literals in markup** (`bg-[#F5F0E6]`) → convert to palette utilities or Elementor controls
- **`!important`** → document why each survives; eliminate where possible
