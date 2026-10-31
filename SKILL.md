---
name: browser-automation
description: Browser automation using browser MCP tools for web scraping, form filling, testing, and security auditing. Controls a real Chromium through a 10-tool gateway that reaches ~98 underlying tools — navigation, element interaction, DOM querying, tab management, network monitoring, and JavaScript execution. Use when asked to browse, scrape, fill a form, automate a website, check a page, open a browser, take a screenshot, monitor network requests, run a security audit, bypass a Cloudflare challenge, or interact with any web page programmatically. Features a token-optimized DOM walker that reduces HTML to compact JSON with numeric IDs, plus Cloudflare Turnstile solving and fingerprint (user-agent/locale/timezone/geo) overrides.
allowed-tools: mcp__browser__start_browser mcp__browser__stop_browser mcp__browser__navigate mcp__browser__click mcp__browser__type_text mcp__browser__get_text_content mcp__browser__get_interaction_tree mcp__browser__search_tools mcp__browser__describe_tool mcp__browser__call_tool
when_to_use: "Use when the user wants to browse a website, scrape web content, fill out forms, automate browser interactions, take screenshots, run security audits, or interact with any web page. Examples: 'open this URL', 'scrape that page', 'fill out the form', 'take a screenshot', 'check this website', 'automate the login', 'run a security audit', 'monitor network requests'."
argument-hint: "[url or task description]"
context: fork
metadata:
  author: coffeegrind123
  version: "1.4"
---

# Browser Automation

## The server runs in GATEWAY mode — only 10 tools are directly callable

Exactly these ten appear as `mcp__browser__*`:

```
start_browser  stop_browser  navigate  click  type_text
get_text_content  get_interaction_tree
search_tools  describe_tool  call_tool
```

**Every other capability** — screenshots, forms, cookies, localStorage, tabs,
network/console logs, JS execution, scrolling, stealth/user-agent, geolocation,
security audit, Cloudflare — exists on the server but is *hidden* to keep the
context small. Reach it in one hop:

```
search_tools(query="read cookies")        → ranked 'name(params) — summary' lines
describe_tool(name="set_cookie")          → full schema, when the summary is not enough
call_tool(name="set_cookie", arguments={...})   → actually run it
```

`call_tool` validates `arguments` against the real schema, so a mistyped field
returns a clear error rather than failing silently. Pass `{}` for a no-arg tool.

Throughout this document, any tool NOT in the ten above is written by its bare
name for readability — invoke it as `call_tool(name="<that name>", arguments={…})`.
If you are unsure a tool exists, `search_tools` is cheaper than guessing.

## Lifecycle: Every Session Follows This

```
start_browser() → navigate(url) → [work] → stop_browser()
```

No tool works until `start_browser()` completes. No page tools work until
`navigate()` completes. Always `stop_browser()` when finished.

**`navigate()` settles the page before it returns — do not add a wait after it.**
It waits for the network to go quiet and reports how that ended, e.g.
`Navigated to https://x/ (network idle after 0.6s, 246 requests)`. A read
straight afterwards sees rendered content. `network still active after 10.0s` is
normal for a page that polls or streams and usually still reads fine. Pass
`settle=0` to return the instant navigation commits, when you will not read the
page.

| Scenario | Call |
|----------|------|
| Default (headful) | `start_browser()` |
| Headless scraping | `start_browser(headless=true)` |
| With proxy | `start_browser(proxy="socks5://host:port")` |
| Authenticated proxy | `start_browser(proxy="http://user:pass@host:port")` |
| Persist session | `start_browser(user_data_dir="/path/to/profile")` |
| Check if running | `get_browser_status()` |

## Primary Discovery: get_interaction_tree()

**ALWAYS use `get_interaction_tree()` as the FIRST step to understand a page.**

This is the single most important tool. It returns a token-optimized JSON
of all interactive elements — 96% smaller than raw HTML. Each element gets
a numeric ID you can click/type into directly.

```json
{"id": 1, "t": "btn", "l": "Submit", "r": "main"}
{"id": 2, "t": "in",  "l": "Email",  "r": "main", "v": ""}
{"id": 3, "t": "link", "l": "Sign Up", "r": "nav"}
```

- `t` = type (`btn`, `link`, `in`, `sel`, `chk`, `rad`, `tab`, `mnu`, `el`)
- `l` = label (inferred from aria-label/placeholder/text/title/alt)
- `r` = region (`hdr`, `nav`, `main`, `side`, `ftr`, `dlg`)
- `v` = current value (inputs only)

**Use numeric IDs directly:**
- `click(selector="1")` — clicks element with id 1
- `type_text(text="hello", selector="2")` — types into element with id 2

Read `reference/dom-walker.md` if you need the full type/region code reference or label inference rules.

## Tool Selection Decision Tree

### "I need to find something on the page"

```
Interactive element (button, link, input)?
  YES → get_interaction_tree()
    Only buttons? → find_buttons(filter_text="optional")
    Only inputs?  → find_inputs(filter_type="text|email|...")
  NO → Visible text content?
    YES → get_text_content()
    NO  → get_content() [raw HTML — use sparingly, large tokens]
```

### "I need to interact with something"

```
Click → get_interaction_tree() → click(selector="<id>")
      → OR click(text="Button Text") if you know exact text
Fill form → fill_form(form_data="{\"#email\": \"a@b.com\", \"#pass\": \"123\"}")
            NOTE: form_data is a JSON *string*, not an object. Serialize it.
          → OR type_text(text="value", selector="<id>")   [core tool, direct]
Dropdown  → select_option(selector="<css>", value="option_value")
Upload    → upload_file(selector="input[type=file]", file_path="/path")
Submit    → submit_form() or press_enter()
```

### "I need to wait for something"

```
Page load        → nothing. navigate() already settled it.
Element appear   → wait_for_element(selector="css", timeout=10, visible=true)
Network idle     → wait_for_network(timeout=5)     # AFTER a click, not after navigate
Specific request → wait_for_request(url_pattern="api/data")
```

All three real waits are hidden tools — `call_tool(name="wait_for_element", …)`.
That extra hop is exactly why you should not spend one on a page load that
`navigate()` has already handled.

A blind `wait(seconds=N)` is the last resort: slower than settling on a fast
page, too short on a slow one. Reach for it only when the thing you are waiting
for is neither an element nor a request — an animation, or a rate limit.

### "I need to extract data"

```
DOM data     → execute_js("document.querySelector('.price').textContent")
Visible text → get_text_content()
API response → clear_logs() → [action] → wait_for_request(url_pattern="api/")
Screenshot   → screenshot(save_path="/tmp/page.png")
```

### "The page is blocked by Cloudflare"

```
Probe   → is_cloudflare_challenge_present()   # fast, no clicking
Solve   → bypass_cloudflare(timeout=20)       # clicks the Turnstile checkbox
```

Most Cloudflare-gated sites pass automatically when the browser is **headed**
(started with `start_browser(headless=false)`); under `--headless=new` the
Turnstile is often unsolvable. For an interactive challenge that survives headed
mode, `bypass_cloudflare()` solves it (wraps zendriver's built-in `verify_cf`).
`set_user_agent` / `set_locale` / `set_timezone` / `set_geolocation` align the
fingerprint with a proxy's geo when needed.

Do **not** pass `low_memory=true` when stealth matters: its flags (software WebGL,
`--disable-gpu`) are themselves a bot signal.

#### When `bypass_cloudflare()` times out — check for a hard-fail before retrying

`bypass_cloudflare()` polls for a Turnstile **checkbox**. If Turnstile has already
fingerprinted the browser and rejected it, it never renders one — it renders a
*feedback report* instead, and the call can only ever time out. Retrying, raising
the timeout, or re-clicking cannot fix this. Confirm with `get_content()`:

```
iframe src=".../challenge-platform/h/b/fr/<id>/en-us/auto/failure#..."
                                                        ^^^^^^^ hard-fail
```

`.../auto/failure` (and a visible `cf-turnstile-feedback` wrapper) = fingerprint
rejection. Stop clicking and change approach.

**Try a crawler User-Agent first when the goal is public SEO/markup.** Many sites
allowlist search crawlers by UA string alone, with no reverse-DNS verification, so
plain `curl` walks straight past the challenge and returns the real HTML — no
browser, no solver, far faster and more reliable:

```
curl -sA "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" https://site/robots.txt
```

Worked on a self-hosted Turnstile gate (2026-07) that hard-failed the automated
browser but served `robots.txt` / `sitemap.xml` / full HTML to a Googlebot UA.
Note a self-hosted widget (its own `<form action="/verify_captcha">` with a
`data-sitekey`) is NOT the standard CF edge challenge — the edge-challenge advice
above may not apply to it at all.

Reach for this only for **publicly published** content (markup, robots, sitemaps)
that the site already serves to crawlers by design — not to get at anything gated,
paid, or private.

## General Rules

1. **Autonomy**: Execute the full browsing workflow without asking for
   confirmation at each step. Ask only when genuinely ambiguous.

2. **Token discipline**: Prefer `get_interaction_tree()` over `get_content()`.
   Raw HTML burns context fast. Only use `get_content()` for HTML structure
   the interaction tree doesn't expose.

3. **Do not wait after navigate**: `navigate()` settles the page itself and says
   how it ended. Add a wait only after a *click* or an action that starts new
   traffic — `wait_for_element()` for content an SPA hydrates in late, or
   `wait_for_network()` after a submit.

3a. **An empty read tells you which problem you have.** `get_text_content()` on a
   page with no text returns `[chars 0-0 of 0]` followed by the reason:
   `still loading` (read again, or navigate with a bigger `settle`), `no body
   element`, `has markup` (the text is in an iframe or shadow root — try
   `get_content` or `get_interaction_tree`), or `really is blank`. Read that line
   before concluding the site blocked you.

4. **Numeric IDs are ephemeral**: They change on every `get_interaction_tree()`
   call. Get the tree, use IDs immediately. Never cache across navigations.

5. **CSS selectors for stability**: When referencing the same element across
   page changes, use a CSS selector instead of a numeric ID.

6. **execute_js syntax**: Scripts must NOT start with `return`. Simple
   expressions directly. Complex logic: `(function(){ var x = 1; return x; })()`

7. **Network monitoring**: Call `clear_logs()` BEFORE the action you want to
   monitor, then `get_network_logs()` or `wait_for_request()` AFTER.

8. **Tab hygiene**: Close tabs opened with `new_tab()` when done.

9. **Error recovery**: If a click or navigation fails, `screenshot()` to
   visually diagnose. The page may have changed or a modal may block.

10. **Auth persistence**: Use `get_cookies()` / `set_cookies()` for cookie-based auth.
    For JWT-based SPAs, use `get_local_storage()` / `set_local_storage()` — many
    modern apps store tokens in localStorage, not cookies.

## Common Workflows

Core tools are called directly; everything else goes through `call_tool`.

### Web Scraping
```
start_browser(headless=true) → navigate(url)      # settled on return
→ get_text_content()                              # or get_interaction_tree()
→ call_tool(name="execute_js", arguments={"script": "..."})
→ stop_browser()
```

### Form Filling
```
start_browser() → navigate(url)
→ get_interaction_tree()
→ call_tool(name="fill_form", arguments={"form_data": "{\"#email\": \"user@ex.com\", \"#password\": \"secret\"}"})
→ click(selector="<submit_id>")
→ call_tool(name="wait_for_network", arguments={"timeout": 5})    # the click, not the navigate
→ call_tool(name="screenshot", arguments={"save_path": "/tmp/after.png"})
→ stop_browser()
```

### Multi-Tab
```
start_browser() → navigate("https://site-a.com") → [work]
→ call_tool(name="new_tab", arguments={"url": "https://site-b.com"}) → [work]
→ call_tool(name="list_tabs")
→ call_tool(name="switch_tab", arguments={"tab_id": "tab_1"})
→ call_tool(name="close_tab", arguments={"tab_id": "tab_2"})
→ stop_browser()
```

### API Response Interception
```
start_browser() → navigate(url)
→ call_tool(name="clear_logs")                    # clear BEFORE the action
→ click(selector="<trigger_id>")
→ call_tool(name="wait_for_request", arguments={"url_pattern": "api/v2/data"})
→ stop_browser()
```

### Security Audit
```
start_browser() → navigate(url)
→ call_tool(name="run_security_audit") → stop_browser()
```

## Error Handling

| Symptom | Diagnosis | Fix |
|---------|-----------|-----|
| "Browser not started" | Forgot lifecycle | `start_browser()` first |
| "No page loaded" | Forgot navigation | `navigate(url)` — no wait needed |
| `[chars 0-0 of 0]` | Read the reason on the next line | It names the fix: still loading / no body / markup-but-no-text / blank |
| Unknown tool name | It is hidden, not absent | `search_tools(query="…")` then `call_tool` |
| Click does nothing | Element not visible | `scroll_to_element()` then retry |
| Element not found | Page not loaded yet | `wait_for_element()` then re-query |
| Stale numeric ID | Page mutated | Re-call `get_interaction_tree()` |
| execute_js fails | Started with `return` | Remove `return`, use IIFE |
| Network logs empty | Didn't clear first | `clear_logs()` before action |
| Form submit no effect | Need explicit submit | `submit_form()` or `press_enter()` |
| Interaction tree empty | SPA hydrates after the network settles | `wait_for_element()` on something you expect, or re-navigate with a larger `settle`; `find_inputs()`/`find_buttons()` as a fallback |
| Clicks blocked by overlay | Cookie banner or modal | Dismiss banner first: `click(text="Accept All")` |

## Critical Pitfalls

- ❌ Do NOT call any tool before `start_browser()` — everything will fail
- ❌ Do NOT use `get_content()` as primary discovery — wastes tokens massively
- ❌ Do NOT cache numeric IDs across navigations — they regenerate each call
- ❌ Do NOT start `execute_js` scripts with `return` — use bare expression or IIFE
- ❌ Do NOT add a `wait()` after `navigate()` — it already settled the page, and
  under the gateway that wasted wait costs a whole `call_tool` hop
- ❌ Do NOT leave tabs open — close with `close_tab()` when done
- ❌ Do NOT read network logs without clearing first — stale entries mislead
- ❌ Do NOT use `mouse_click(x,y)` unless no other option — coordinates are fragile
- ❌ Do NOT call `get_interaction_tree()` then ignore the IDs — that's the whole point
- ❌ Do NOT interact with elements before dismissing cookie consent banners — they block clicks
- ❌ Do NOT assume auth is in cookies — check `get_local_storage()` for JWT tokens too
- ❌ Do NOT pass `timeout` to `screenshot` — it has no such parameter (only
  `save_path` and `full_resolution`), and `call_tool` validates arguments, so it
  is a hard error. Screenshots really can hang on an unresponsive page; guard
  against that by settling the page first (which `navigate` now does) rather than
  by inventing an argument

## Self-Refinement Protocol

After completing a browser automation task, if you discovered something
non-obvious (a selector pattern, a timing requirement, a site-specific
quirk), append it to `LEARNINGS.md`:

```
## YYYY-MM-DD: <brief title>
- **Context**: What you were doing
- **Learning**: What was non-obvious
- **Rule**: The new rule to follow
```

Read `reference/tools.md` if you need full parameter documentation for any tool.
Read `reference/patterns.md` if you need automation recipes beyond the common workflows above.
Read `reference/dom-walker.md` if the interaction tree output is unclear or you need type/region code details.
