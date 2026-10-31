## 8. Pitfalls & Lessons Learned

Hard-won lessons from real migration work. Each is a rule that prevents a known failure.

### Widget architecture

- **Never put markup in the plugin.** The widget is a control panel; `vm_render()` always delegates to a theme partial via `vm_the_block()`.
- **`vm_args()` drops empty values.** An empty control falls through to the partial's `wp_parse_args()` literal. That literal is the real default content.
- **`vm_cleared()` keeps explicitly-emptied repeaters empty.** Without it, deleting all FAQ rows would re-show the seed rows.
- **Two copies = drift.** Never maintain `plugin/` AND `wp-content/plugins/` copies. Use a symlink.

### Style controls

- **Inert defaults are load-bearing.** `selectors_dictionary['full'] = ''` means "full width emits no CSS rule". Changing that default to a non-empty value changes every existing widget's compiled CSS.
- **`Post::create(8)->update()` writes stale CSS** after a revert. Reliable regeneration = `delete_post_meta` + `unlink` the CSS file + one real front-end request.
- **Always nest `section > column > widget`** when probing widget CSS. A widget directly inside a section compiles as a Column (Elementor instantiates section children as columns), making `padding_block` silently disappear while `heading_color` looks correct.

### Theme Builder

- **`need_override_location` must return false.** Without it, Elementor replaces your template hierarchy with `elementor_header_footer.php`, losing body classes, skip link, and the `#vm-page` wrapper.
- **Elementor's `.elementor img{height:auto}` and `.elementor a{text-decoration:none}` are unlayered.** They beat Tailwind v4 utilities inside `@layer utilities`. Fix with `revert-layer` rules.
- **JS enqueue for header location.** The Elementor header doesn't load the PHP partial that enqueues `nav.js`. Add the enqueue to `vm_theme_location()` when the header location renders.
- **Content scripts must be re-enqueued.** When the header template renders via Elementor (not via `get_template_part()`), any scripts the PHP partial would have enqueued are missing.

### Elementor MCP

- **Write conflict after build-composition.** `manage-elements` does a read-modify-write. Using element IDs from a prior `get-page-structure` after `build-composition` silently overwrites the new content.
- **`dry_run` does not assign IDs.** Only a non-dry_run call returns `resolved_xml` with server-assigned element IDs.
- **V3 vs V4 elements.** Only V4 (atomic) elements can be modified via MCP tools. V3 elements are returned for context but must be edited in the Elementor editor.
- **SVG widgets require uploaded assets.** External URLs on `e-svg` render an empty div. Use `elementor_list_assets` with `type: "svg"` to find attachment IDs.

### CSS & Tailwind

- **Tailwind v4 puts utilities in `@layer utilities`.** Unlayered CSS always wins regardless of specificity. This is why Elementor's resets beat Tailwind.
- **Kit colours live on `<body>`, not `<html>`.** `--e-global-color-*` is set on `.elementor-kit-6` (body class). Measuring on `<html>` gets the fallback value.
- **`color-mix()` in `var()` chains.** Intermediate colours (jungle-800) need `color-mix(in srgb, var(--vm-color-primary) 62%, var(--vm-color-secondary))` in Tailwind.

### Data migration

- **Empty repeater key vs absent key.** `faqs = []` in `_elementor_data` means "editor deleted all rows" (render nothing). Absent key means "use the default" (render 6 rows). Removing the key restores the default.
- **`$document->save()` re-runs `register_controls()` on every widget** and regenerates CSS. For metadata-only changes (titles), write `update_post_meta()` directly.
- **Backup before migration.** Always `wp_json_encode($data)` the original `_elementor_data` to a fixture file before modifying.