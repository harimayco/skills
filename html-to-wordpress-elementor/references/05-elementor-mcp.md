## 5. Elementor MCP Tools — Programmatic Page Building

The Elementor MCP server provides tools for building pages without the visual editor.
This is the primary mechanism for converting mockup HTML into Elementor elements.

### Tool inventory

| Tool | Purpose |
|------|---------|
| `elementor_list_widget_schemas` | Discover available widget types (use `summary=true` first) |
| `elementor_get_widget_schema` | Full JSON Schema for one widget type's settings |
| `elementor_get_page_structure` | Element tree skeleton for a post (id, type, version) |
| `elementor_build_composition` | Build new element trees via XML + config |
| `elementor_manage_elements` | Surgical edits on existing V4 elements |
| `elementor_manage_global_variable` | Create/update/delete design tokens (color, font, size) |
| `elementor_manage_classes` | Create/update/delete global CSS classes |
| `elementor_manage_default_styles` | Site-wide default styles per HTML tag |
| `elementor_manage_site_parts` | Create/manage Theme Builder parts (header, footer, etc.) |
| `elementor_manage_component` | Create/manage reusable widget compositions |
| `elementor_create_page` | Create new draft page |
| `elementor_create_preview_link` | Generate preview URL for validation |
| `elementor_list_assets` | Images/SVGs in Media Library |
| `elementor_list_posts` | WordPress posts/pages |
| `elementor_list_site_parts` | Theme Builder documents |
| `elementor_list_components` | Reusable compositions |
| `elementor_publish_document` | Publish a draft |

### build-composition workflow

1. **Discover widgets**: `elementor_list_widget_schemas(summary=true)`
2. **Read schemas**: `elementor_get_widget_schema(widget_type='e-heading')` for each type you'll use
3. **Check globals**: Read `elementor://global-variables` and `elementor://global-classes`
4. **Build XML**: Every element gets a unique `configuration-id` attribute
5. **Configure**: Map `configuration-id` → `element_config` (props) + `style` (CSS) + `classes` (global class labels)
6. **Execute**: `elementor_build_composition(post_id=..., xml_structure=..., element_config=..., style=...)`
7. **Use returned IDs**: `resolved_xml` contains server-assigned element `id` attributes

### XML structure rules

- Widget tags: `<e-button configuration-id="btn1"></e-button>`
- Containers: `e-flexbox`, `e-div-block`, `e-tabs`
- Every element needs unique `configuration-id` — NO other attributes
- Multiple root elements allowed
- No `<![CDATA[]]>`, code fences, or wrappers — raw XML in JSON string

### element_config format (plain JSON values)

| Type | Format |
|------|--------|
| string/enum | `"h2"`, `"https://example.com"` |
| number | `42` |
| boolean | `true` |
| text (heading/paragraph/button) | `"Welcome"` (may contain inline HTML) |
| image (library) | `{"src": {"id": 123}, "size": "full"}` |
| image (external) | `{"src": {"url": "https://..."}, "size": "full"}` |
| link | `{"destination": "https://...", "isTargetBlank": true, "tag": "a"}` |
| dynamic | `{"name": "post-title", "settings": {}}` |

### Style string format

Plain CSS, supports pseudo-states and breakpoints:
```css
font-size: 3.5rem; color: #2d2a26;
&:hover { color: var(--gold); }
@media(--mobile) { font-size: 2.25rem; }
@media(--tablet) { font-size: 2.75rem; }
```

### Write conflict prevention (CRITICAL)

`manage-elements` does a read → modify → write. If you call it after
`build-composition` using element IDs from a prior `get-page-structure`,
it restores the old tree and silently overwrites `build-composition`'s changes.

**Rule**: Only use element IDs from the LATEST `resolved_xml` response.

### Mode parameter

- `append` (default): Insert new elements as children of `parent_id`
- `replace_children`: Remove all children first, then insert new ones
- `parent_id: 'document'` + `mode: 'replace_children'` = redesign whole page

### Sizing philosophy (CRITICAL)

- DO NOT specify height/width unless you have a specific visual reason
- Flexbox handles sizing automatically
- Use `min-height` on root sections for viewport-spanning heroes
- Use `max-width` for contained content areas
- Never nest 100vh inside 100vh (200vh overflow)