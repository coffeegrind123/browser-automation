# DOM Walker: Interaction Tree Internals

## What get_interaction_tree() Does

Traverses the entire page DOM and extracts every interactive element into
compact JSON. This is the primary way to understand what you can interact
with on a page. Skips hidden elements, SVG internals, and redundant nested
interactive children.

## Output Format

```json
{
  "id": 1,        // Numeric ID — use with click(selector="1")
  "t": "btn",     // Type code
  "l": "Submit",  // Human-readable label
  "r": "main",    // Page region
  "v": "",        // Current value (inputs only)
  "it": "email",  // Input type (only if non-text)
  "ck": true,     // Checked state (checkboxes/radios only)
  "dis": true,    // Disabled flag (only if disabled)
  "off": true     // Off-screen flag (only if off-screen)
}
```

Only `id`, `t`, and `l` are always present. Other fields appear only when relevant.

## Type Codes

| Code | Meaning | HTML Elements |
|------|---------|---------------|
| `btn` | Button | `<button>`, `<input type="submit">`, `[role="button"]` |
| `link` | Link | `<a>`, `[role="link"]` |
| `in` | Text input | `<input>`, `<textarea>`, `[contenteditable]` |
| `sel` | Select/dropdown | `<select>`, `[role="combobox"]`, `[role="listbox"]` |
| `chk` | Checkbox | `<input type="checkbox">`, `[role="checkbox"]` |
| `rad` | Radio button | `<input type="radio">`, `[role="radio"]` |
| `tab` | Tab control | `[role="tab"]` |
| `mnu` | Menu item | `[role="menuitem"]` |
| `el` | Generic interactive | Any focusable/clickable not matching above |

## Region Codes

| Code | Meaning | Detection |
|------|---------|-----------|
| `hdr` | Header | `<header>`, `[role="banner"]`, top < 80px |
| `nav` | Navigation | `<nav>`, `[role="navigation"]` |
| `main` | Main content | `<main>`, `[role="main"]`, largest content area |
| `side` | Sidebar | `<aside>`, `[role="complementary"]`, left < 200px |
| `ftr` | Footer | `<footer>`, `[role="contentinfo"]`, bottom > vh-80 |
| `dlg` | Dialog/modal | `<dialog>`, `[role="dialog"]`, `[role="alertdialog"]` |

## Label Inference Priority

The walker infers a human-readable label using this priority:

1. `aria-label` attribute
2. `aria-labelledby` → text of referenced element
3. Associated `<label>` element (via `for` attribute)
4. `placeholder` attribute
5. Visible text content (`innerText`, max 60 chars)
6. `title` attribute
7. `alt` attribute (images)

Even unlabeled elements usually get a useful label through this chain.

## Using Numeric IDs

```
tree = get_interaction_tree()
# Returns: {"id": 7, "t": "btn", "l": "Add to Cart", "r": "main"}

click(selector="7")              # click by numeric ID
type_text(text="hello", selector="7")  # type by numeric ID
get_element_text(selector="7")   # read text by numeric ID
```

### IDs Are Ephemeral

Numeric IDs are assigned fresh on every `get_interaction_tree()` call.
They are NOT stable across:
- Page navigations
- DOM mutations (SPA route changes, AJAX updates)
- Multiple `get_interaction_tree()` calls

**Rule**: Call `get_interaction_tree()`, use the IDs immediately, never
store them for later.

## When Interaction Tree Is Not Enough

| Need | Use instead |
|------|-------------|
| Static text | `get_text_content()` or `execute_js()` |
| HTML structure | `get_content()` (sparingly) |
| Element attributes | `get_element_attribute(selector, attr)` |
| Computed styles | `execute_js("getComputedStyle(el).property")` |

## Token Size Comparison

| Method | Typical size (medium page) |
|--------|---------------------------|
| `get_content()` | 50,000 - 200,000 tokens |
| `get_text_content()` | 2,000 - 10,000 tokens |
| `get_interaction_tree()` | 500 - 3,000 tokens |

The 96% reduction is real and consistent. Always start with the tree.

## Troubleshooting

**Empty tree**: Page not loaded. Did you `wait()` after `navigate()`?

**Missing elements**: Element may be inside an iframe, created after
initial load (use `wait_for_element()` first), or hidden with `display:none`.

**Wrong labels**: Label inference picked up adjacent text. Use
`get_element_attribute(selector="<id>", attribute="outerHTML")` to inspect.

**Too many elements**: On complex pages, the tree can still be large.
Use `find_buttons()` or `find_inputs()` to narrow the scope.
