# Browser MCP Tool Reference

## Browser Lifecycle

### start_browser
Start a new browser instance. Must be called before all other tools.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| headless | bool | false | Run without visible window |
| proxy | string | null | Proxy URL — supports `http://host:port`, `socks5://host:port`, and authenticated `http://user:pass@host:port` (credentials handled via CDP) |
| user_data_dir | string | null | Path to Chrome profile directory |

### stop_browser
Shut down the browser and release resources. Always call when done.

### get_browser_status
Check if browser is running and get its state. Returns: running/stopped, tabs.

---

## Navigation

### navigate
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| url | string | yes | Full URL to navigate to |
| settle | number | no | Seconds to wait for network idle after the page commits (default 10.0). `0` returns as soon as navigation commits. |

**Do NOT follow with a wait.** navigate settles the page itself and returns e.g.
`Navigated to <url> (network idle after 0.6s, 246 requests)`. `network still
active after 10.0s` is normal for a polling/streaming page and usually still
reads fine.

### go_back / go_forward
Navigate history. No parameters. These do NOT settle — follow with
`wait_for_network()` if you intend to read the page straight after.

### reload_page
Reload current page. Does NOT settle either; follow with `wait_for_network()`
before reading.

### get_page_info
Get current URL, title, and page metadata.

---

## Element Discovery

### get_interaction_tree
**PRIMARY DISCOVERY TOOL.** Returns token-optimized JSON of all interactive
elements with numeric IDs. 96% smaller than raw HTML. No parameters.

Returns array of:
```json
{"id": <int>, "t": "<type>", "l": "<label>", "r": "<region>", "v": "<value>"}
```

### find_element
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| selector | string | no | CSS selector |
| text | string | no | Text content to find |

Returns element info with suggestions if not found.

### find_all_elements
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| selector | string | yes | CSS selector |
| limit | int | no | Max results (default 20) |

### find_buttons
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| filter_text | string | no | Filter by button text |

Finds `<button>`, `input[type=submit]`, `[role=button]`, clickable links.
Returns smart CSS selectors for each.

### find_inputs
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| filter_type | string | no | Filter by type (text, email, password, etc.) |

Finds `<input>`, `<textarea>`, `[contenteditable]`, `[role=textbox]`.

### get_element_text
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| selector | string | yes | CSS selector or numeric ID |

### get_element_attribute
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| selector | string | yes | CSS selector or numeric ID |
| attribute | string | yes | Attribute name (href, src, class, etc.) |

---

## Element Interaction

### click
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| selector | string | no | CSS selector or numeric ID |
| text | string | no | Visible text to match |

Provide either selector OR text. Numeric IDs: `click(selector="5")`.

### type_text
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| text | string | yes | Text to type |
| selector | string | no | Target element (CSS or numeric ID) |

Uses CDP `Input.insertText` — real input, not JS simulation.

### clear_input
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| selector | string | yes | CSS selector or numeric ID |

### focus_element
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| selector | string | yes | CSS selector or numeric ID |

### select_option
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| selector | string | yes | CSS selector of select element |
| value | string | no | Option value attribute |
| text | string | no | Option visible text |

### upload_file
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| selector | string | yes | CSS selector of file input |
| file_path | string | yes | Absolute path to file |

### mouse_click
Last resort — coordinates are fragile.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| x | int | yes | X coordinate |
| y | int | yes | Y coordinate |

---

## Forms

### fill_form
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| json | object | yes | Map of CSS selector to value |

```
fill_form(json={"#email": "a@b.com", "#password": "secret"})
```

### submit_form
Submit the currently focused form.

### press_key
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| key | string | yes | Key name (Enter, Tab, Escape, ArrowDown, etc.) |

Full event simulation: keydown, keypress, keyup.

### press_enter
Convenience wrapper for `press_key("Enter")`.

---

## Page Content

### get_content
Get raw HTML of the page. **Use sparingly** — large token cost.

### get_text_content
Get all visible text. Cleaner than raw HTML for text extraction.

### scroll
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| direction | string | no | "up" or "down" |
| amount | int | no | Pixels to scroll |

### scroll_to_element
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| selector | string | yes | CSS selector or numeric ID |

---

## Tab Management

### new_tab
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| url | string | no | URL to open (blank if omitted) |

### list_tabs
List all open tabs with IDs and URLs.

### switch_tab
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| tab_id | int | yes | Tab index from list_tabs() |

### close_tab
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| tab_id | int | yes | Tab index to close |

---

## Storage

### get_cookies / set_cookies
Get all cookies or set cookies (array of cookie objects).

### get_localStorage / set_localStorage
Get all entries or set a key-value pair.

### clear_storage
Clear all cookies and localStorage.

---

## Network & Console Monitoring

### get_network_logs
Get captured network request/response logs. Call `clear_logs()` first.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| limit | int | no | Max entries to return |

### get_console_logs
Get captured browser console output.

### clear_logs
Clear all captured network and console logs. **Call before the action you want to monitor.**

### wait_for_network
Wait until all pending network requests complete.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| timeout | int | no | Max seconds to wait (default 30) |
| idle_time | float | no | Seconds of quiet to consider idle (default 0.5) |

### wait_for_request
Wait for a specific network request matching a URL pattern.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| url_pattern | string | yes | Substring to match against request URLs |
| timeout | int | no | Max seconds to wait |
| method | string | no | HTTP method filter (GET, POST, etc.) |

---

## Utilities

### screenshot
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| save_path | string | no | File path to save PNG |

Returns JPEG inline. Saves PNG if path provided.

### execute_js
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| script | string | yes | JavaScript to execute |

**Do NOT start with `return`**. Simple: `document.title`. Complex: `(function(){ return items; })()`

### wait
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| seconds | float | yes | Seconds to wait |

### wait_for_element
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| selector | string | yes | CSS selector |
| timeout | int | no | Max seconds (default 30) |
| visible | bool | no | Require visible (default false) |

### run_security_audit
Comprehensive security report: HTTPS, CSRF, mixed content, XSS vectors,
exposed secrets (AWS keys, JWT, private keys), missing SRI, dangerous JS
patterns. Returns formatted PASS/FAIL/WARN report.
