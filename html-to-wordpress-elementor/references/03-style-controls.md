## 3. Style Controls & CSS Binding

The key insight: Elementor style controls with `selectors` emit CSS rules only
when their value is non-empty. The design rule is: **an untouched widget must
compile to byte-identical CSS** — no control should gain a non-empty default.

### Binding pattern

Every style control needs a `selectors` entry keyed on `{{WRAPPER}}`:

```php
$this->add_control('heading_color', array(
    'label'     => 'Heading colour',
    'type'      => Controls_Manager::COLOR,
    'default'   => '',   // EMPTY = inert
    'selectors' => array(
        '{{WRAPPER}} h1, {{WRAPPER}} h2, {{WRAPPER}} h3' => 'color: {{VALUE}};',
    ),
));
```

### Inert defaults — zero rules until editor sets a value

| Control | Inert default | How |
|---------|---------------|-----|
| `content_width` | `'full'` | `selectors_dictionary['full'] = ''` → empty = no rule |
| `padding_block` | `'none'` | `selectors_dictionary['none'] = ''` |
| `text_align` | `''` | Empty string maps to `''` via dictionary |
| `heading_font_size` | `array()` | Empty slider = no size/unit = no rule |
| `bg_overlay_opacity` | `array()` | Empty slider |
| Colors | `''` | Empty string |
| `bg_image` | `array()` | Empty media |

### Specificity strategy (no `!important`)

The widget wrapper prefix makes rules (0,1,1) or (0,2,0), which outranks
the theme's single Tailwind class (0,1,0) without `!important`:

```
.elementor-8 .elementor-element.elementor-element-vxw1a06 h1 { color: red; }
   (0,3,1) beats
.text-jungle-900 { color: #0f2419; }
   (0,1,0)
```

### Theme hints (editor metadata, not compiled)

Document the theme's effective value in the control's `description`:
```php
protected function vm_theme_hints() {
    return array(
        'vm_hero' => array(
            'heading_color' => 'Theme default: #f5f0e6 (sand-50) over the hero scrim.',
            'heading_font_size' => 'Theme default: 3.5rem / 56px, fluid via clamp().',
        ),
    );
}
```
These are editor-only — never compiled into CSS.

### Verification

After adding style controls, the page's compiled CSS must be byte-identical:
```bash
md5sum wp-content/uploads/elementor/css/post-8.css
# Must match the md5 from BEFORE adding controls
```

Forced regeneration:
```php
delete_post_meta( $post_id, '_elementor_css' );
unlink( "wp-content/uploads/elementor/css/post-{$post_id}.css" );
// Then request the page in a browser to regenerate
```
