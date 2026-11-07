## 4. Theme Builder Migration

Converts hardcoded PHP template shells (header.php, footer.php, single.php,
index.php, page.php) into Elementor Theme Builder templates while keeping
the PHP as fallback.

### Architecture

New module `inc/theme-builder.php` loaded from `functions.php`:

| Function | Purpose |
|----------|--------|
| `register_elementor_locations()` | Registers core locations on `elementor/theme/register_locations` |
| `keep_theme_template()` | Filters `elementor/theme/need_override_location` to `false` |
| `theme_location( $location, $fallback )` | `elementor_theme_do_location()` when available, else PHP fallback |
| `[vm_partial name="..."]` shortcode | Renders theme partial from Theme Builder template |

### The override problem (CRITICAL)

Without `need_override_location` returning false, Elementor Pro replaces
your page template with its own `elementor_header_footer.php`:
- Loses body classes, skip link, `#vm-page` wrapper
- Renders title shell twice
- Breaks the CSS cascade

### Shell changes

| File | Before | After |
|------|--------|-------|
| `header.php` | `get_template_part('template-parts/header/navbar')` | `vm_theme_location('header', fn() => ...)` |
| `footer.php` | `get_template_part('template-parts/footer/footer')` | `vm_theme_location('footer', fn() => ...)` |
| `single.php` | inline `while(have_posts())` | `vm_theme_location('single', fn() => ...)` |
| `index.php` | inline loop + pagination | `vm_theme_location('archive', fn() => ...)` |
| `page.php` | inline loop → content/page | `vm_theme_location('single', fn() => ...)` |

### Phase 1: Shortcode bridge

Create templates containing `[vm_partial name="header/navbar"]` — same markup,
now editable in Elementor as a Shortcode widget inside a full-width container.

### Phase 2: Native widget rebuild

Replace the Shortcode widget with native Elementor V4 widgets:
- `e-heading`, `e-paragraph`, `e-button`, `e-image` for text/media
- `e-flexbox`, `e-div-block` for layout containers
- `nav-menu` (Pro) for navigation
- `theme-post-content`, `theme-post-featured-image` for dynamic content
- `archive-posts` for archive listings

Dynamic values use atomic dynamic tags:
```json
{"$$type": "dynamic", "value": {"name": "post-title"}}
```

### Template conditions

| Template | Conditions |
|----------|------------|
| Header | `include/general` |
| Footer | `include/general` |
| Single Page | `include/singular/page`, `exclude/front_page` |
| Single Post | `include/singular/post` |
| Archive | `include/archive` |

### Header location script enqueue (post-hindsight)

The Elementor header template does not run the PHP partial that enqueues the
mobile-nav script, so the drawer toggle is dead on any page rendered through
Theme Builder. Enqueue the header script from inside the bridge, not the partial:

```php
function vm_theme_location( $location, callable $fallback ) {
    if ( 'header' === $location && wp_script_is( 'vm-nav', 'registered' ) ) {
        wp_enqueue_script( 'vm-nav' );
    }
    if ( ! function_exists( 'elementor_theme_do_location' ) || ! elementor_theme_do_location( $location ) ) {
        $fallback();
    }
}
```

### Shared content partials (single source for both paths)

When a template body is loop-sensitive, extract the inner markup into a
loop-free partial so the PHP fallback and the Theme Builder path print
identical HTML. Two patterns proven in production:

- **Single post:** `template-parts/content/single-body.php` — calls
  `get_post()` + `setup_postdata()` + `wp_reset_postdata()` itself, because the
  Theme Builder render may consume the main loop before the partial runs.
  The Theme Builder template then renders `[vm_partial name="content/single-body"]`.
- **Archive:** `template-parts/content/archive-grouped.php` renders the
  grouped list via a custom widget (`vm_archive_grouped`), while
  `template-parts/content/archive.php` is the fallback; both delegate each
  card to a shared `template-parts/content/blog-card.php` so grouped and
  fallback paths render the same premium card.

Do not duplicate the card/body markup in both paths; extract once.

### Elementor CSS reset conflict

`.elementor img{height:auto}` and `.elementor a{text-decoration:none}` are
unlayered, so they beat Tailwind v4 utilities (which are inside `@layer utilities`).

Fix: add `revert-layer` rules scoped to elements with classes:
```css
.elementor img[class] { height: revert-layer; }
.elementor a[class] { text-decoration: revert-layer; }
```
