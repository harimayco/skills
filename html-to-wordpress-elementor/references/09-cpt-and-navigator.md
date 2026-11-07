## 9. CPT/Taxonomy Data Model + Navigator Labels

Two post-migration extensions that build on the widget/partial architecture:
(1) move repeatable content (menu items, portfolio entries, testimonials) from
widget repeaters into custom post types so clients edit rows in wp-admin;
(2) give every built element a human-readable Navigator name so the editor
stays navigable after the tree grows.

### 9a. Custom post types + taxonomy + meta boxes

#### Model (in theme `inc/`)

```
inc/post-types.php    register_post_type + register_taxonomy + register_post_meta
inc/meta-boxes.php    add_meta_box (with wp.media frame for gallery field)
template-parts/content/*.php   single/archive card renderers for the CPT
```

Rules proven in production:

- **Own the meta keys.** Register every CPT meta key with `register_post_meta`
  (e.g. `price_bzd`, `price`, `price_unit`, `tags`, `description`, `popup_description`,
  `gallery_images`) AND read/write them consistently. Every meta field a box saves
  must be readable by the template part — a renamed or typo'd key renders as an
  empty string, not an error.
- **Meta key mapping traps.** A metabox may persist `vm_price_bzd` while the block
  reads `price_bzd` and `price`. Pick one canonical key per field and use it in
  the box input `name=`, the `register_post_meta` call, and the partial's
  `get_post_meta()` read. A mismatched pair silently drops the field.
- **The partial owns the query.** A block that lists CPT items runs `WP_Query`
  inside the partial when the post type exists, so it stays correct on the
  fallback path (no widget instance) and the from-CPT path alike.
- **Data-source switch (non-breaking).** Add a `data_source` control
  (`cpt` default / `manual`). When `manual`, the widget's repeater (`items`)
  wins as before; when `cpt`, the query wins. Never delete the repeater —
  a client mid-edit must not lose rows on the switch.
- **Gallery metabox = media frame + hidden comma list.** Save a hidden input
  of comma-separated attachment IDs; render thumbnails from `wp_get_attachment_image_url(id, 'thumbnail')`.
  The featured image is the cover; the list powers the popup/slider gallery.
- **Taxonomy lives on the CPT.** `menu_category`-style taxonomy registered with
  the CPT slug as object type; the partial reads the first term for the item's
  category slug.

### 9b. Human-readable Navigator names

Every built node must carry both:

```
settings._title
editor_settings.title
```

in its `_elementor_data` entry. Use descriptive, scoped names, not opaque IDs:

```
Good:  About · Image · "Aerial view of Villa Massis Waterfront"
Bad:   Div Block 12 / elementor-element-abc123 / ''
```

Pattern (produced by a traversal pass, not by hand):

1. Walk the full `_elementor_data` tree recursively.
2. Derive the label from context: section slug (Hero, About, …) + widget kind
   (Heading, Image, Paragraph, Button, Block) + a short excerpt of the control
   value (first ~30 chars of the heading/text). Quote the excerpt.
3. Write `settings._title` and `editor_settings.title` via `update_post_meta()`
   directly — never `$document->save()`, which re-runs `register_controls()` and
   regenerates CSS as a side effect. Backup `wp_json_encode($data)` first.
4. Verify: count labelled vs total nodes; assert 100% labelled, 0 machine names,
   0 empty. Native widget types (`e-heading`, `nav-menu`, `theme-post-content`,
   `archive-posts`) label themselves with their official widget name.

### 9c. Legacy saved-data drift

A repeater shipped earlier with rows saved empty (`items: []`) renders nothing
because `vm_cleared()` treats an explicitly-empty repeater as intentional.
Fix: drop the stale key from `_elementor_data` post meta so the widget defaults
and partial `wp_parse_args()` literals take over. Confirm with a front-end DOM
count of the section's items (e.g. 6 FAQ accordion items), not by reading meta.

### 9d. theme.json pairing

If the theme ships `theme.json`, it becomes the token source alongside kit
tokens: define palette colors (`primary`, `secondary`, `text`, `accent`), font
families, and `surface`/`border` input colors; disable duotone merging when it
would fight Elementor. Keep the same hexes as the kit fallbacks in
`kit-tokens.css` — two sources of one palette is the drift this prevents.
