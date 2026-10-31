## 7. Verification & Parity Testing

Every migration phase requires before/after comparison proving no visual regression.

### What to capture

| Metric | How |
|--------|----- |
| `document.body.innerText` | Whitespace-normalised, sha1 hash |
| Element counts | `a`, `img`, `svg`, `li` totals |
| `scrollHeight` | Page height |
| Computed styles | fontSize, fontWeight, color, textAlign, paddingBlock, lineHeight on key elements |
| Compiled CSS md5 | `wp-content/uploads/elementor/css/post-<id>.css` |
| `debug.log` line count | Before and after |

### Parity probe pattern

Use `browser_exec` to capture metrics on every published URL:

```python
# Navigate to each URL, wait for load, then extract:
js('''
(() => {
    const text = document.body.innerText.replace(/\\s+/g, ' ').trim();
    const counts = {
        a: document.querySelectorAll('a').length,
        img: document.querySelectorAll('img').length,
        svg: document.querySelectorAll('svg').length,
    };
    return { text_len: text.length, scroll: document.body.scrollHeight, counts };
})()
''')
```

### Acceptable deltas

| Delta | Cause | Acceptable? |
|-------|-------|-------------|
| Wrapper elements (+1-2 per widget) | Elementor widget wrappers | Yes |
| Hidden nav copies (+5 `a`, +5 `li`) | `nav-menu` responsive dropdown | Yes |
| scrollHeight ±50px | Container padding rhythm differences | Yes |
| `svg` +2-3 per page | Icon widgets vs inline SVG | Yes |
| Text hash different | Content actually changed | NO — investigate |

### Debug log verification

```bash
wc -l wp-content/debug.log  # before
# ... run migration ...
wc -l wp-content/debug.log  # after — should be identical or +0-1
```

### Editor verification

Open each template in the Elementor editor (via admin cookie HTTP request):
- Response should be ~3.9 MB HTML with `window.elementor`
- No `Fatal error` or `Warning:` in response body
- `#elementor-panel` present

### Fallback verification

Draft all Theme Builder templates, flush caches, verify all URLs return 200
with the PHP partial markup (the "before" state).