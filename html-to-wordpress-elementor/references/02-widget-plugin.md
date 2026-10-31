## 2. Custom Elementor Widget Plugin Architecture

The widget plugin is ONLY a control panel. All markup lives in theme partials.

### Plugin structure

```
wp-content/plugins/<name>-elements/
├── <name>-elements.php          # Plugin header + boot
├── includes/
│   ├── class-<name>-plugin.php  # Category + widget registration
│   ├── trait-<name>-widget-base.php  # Shared controls + render path
│   ├── trait-<name>-reservation-controls.php  # Domain-specific shared controls
│   └── widgets/
│       ├── class-<name>-widget-hero.php
│       ├── class-<name>-widget-menu-list.php
│       └── ... (one per section)
└── languages/
```

### Boot sequence

```php
// Plugin main file
define( '<PREFIX>_ELEMENTS_FILE', __FILE__ );
define( '<PREFIX>_ELEMENTS_DIR', plugin_dir_path( __FILE__ ) );
require_once <PREFIX>_ELEMENTS_DIR . 'includes/trait-<name>-widget-base.php';
require_once <PREFIX>_ELEMENTS_DIR . 'includes/class-<name>-plugin.php';
<Plugin_Class>::boot();
```

Boot hooks:
- `elementor/elements/categories_registered` → register custom category
- `elementor/widgets/register` → register each widget class

### Widget class pattern

```php
class Widget_Hero extends \Elementor\Widget_Base {
    use Widget_Base_Trait;
    
    public function get_name()    { return 'vm_hero'; }
    public function get_title()   { return 'Hero'; }
    public function get_icon()    { return 'eicon-banner'; }
    
    protected function register_controls() {
        $this->vm_group_header( array('heading' => 'Welcome') );
        $this->vm_group_anchor( 'hero' );
        $this->vm_group_layout();
        $this->vm_group_background();
        $this->vm_group_style();
        $this->vm_group_actions();
        $this->vm_group_motion();
    }
    
    protected function render() {
        $s = $this->get_settings_for_display();
        $this->vm_render( 'hero', $s );
    }
}
```

### The render path (critical)

```
Widget::render()
  → vm_render( $block, $settings, $extra )
    → vm_args( $settings )        # drop empty values, skip internals
    → apply_filters( 'vm_widget_block_args', $args )
    → vm_the_block( $block, $args )
      → get_template_part( 'template-parts/blocks/' . $block, null, $args )
```

`vm_args()` drops empty values so the partial's `wp_parse_args()` default wins.
`vm_cleared()` keeps explicitly-emptied repeaters empty (editor deleted all rows).

### Shared control groups (via trait)

| Group | Controls | Tab |
|-------|----------|-----|
| `vm_group_header()` | badge_text, badge_icon, heading, subheading, heading_tag | Content |
| `vm_group_anchor()` | section_id, section_class | Content |
| `vm_group_layout()` | content_width, padding_block, text_align | Content |
| `vm_group_background()` | bg_type, bg_color, bg_image, bg_overlay_opacity | Content |
| `vm_group_style()` | heading_color, heading_font_size/weight, letter_spacing, text_transform, subheading_color | Style |
| `vm_group_actions()` | cta_text, cta_action, cta_anchor, cta_url | Content |
| `vm_group_motion()` | reveal, reveal_delay, respect_reduced_motion | Content |

### Repeater pattern

```php
$this->vm_repeater_control(
    'faqs',
    __( 'FAQ items' ),
    array(
        'question' => array( 'label' => 'Question', 'type' => Controls_Manager::TEXT, 'default' => '' ),
        'answer'   => array( 'label' => 'Answer',   'type' => Controls_Manager::WYSIWYG, 'default' => '' ),
    ),
    array( 'default' => $this->vm_repeater_default( 'faqs' ) )
);
```

`vm_repeater_default('faqs')` calls the theme's `vm_component_default_control_rows('faqs')`,
so the seed rows live in one place (the theme) and are reshaped for Elementor's control types.
