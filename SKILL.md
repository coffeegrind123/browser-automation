---
name: browser-automation
description: Browser automation through the zendriver-mcp-next server (browser MCP tools) for browsing, scraping, form filling, testing and security auditing. Drives a real, headed Chrome that pages cannot easily tell is automated (trusted input, no DOM markers, helper code in an isolated world). 19 tools are listed directly (the browse loop plus search_tools, describe_tool, call_tool, batch, jobs); about 80 more (cookies, storage, network logs and mocking, emulation, geolocation/timezone/locale, accessibility, tracing, Lighthouse, screencasts, Cloudflare challenges, security audit) are one search_tools hop away. Every result is structured; every error has a code and a hint. Use when asked to browse, scrape, fill a form, automate a website, check a page, open a browser, take a screenshot, monitor network requests, run a security audit, get past a Cloudflare challenge, or interact with any web page programmatically.
allowed-tools: mcp__browser__start_browser mcp__browser__stop_browser mcp__browser__get_browser_status mcp__browser__navigate mcp__browser__get_interaction_tree mcp__browser__click mcp__browser__type_text mcp__browser__press_key mcp__browser__get_text_content mcp__browser__screenshot mcp__browser__execute_javascript mcp__browser__wait_for_element mcp__browser__list_tabs mcp__browser__scroll mcp__browser__search_tools mcp__browser__describe_tool mcp__browser__call_tool mcp__browser__batch mcp__browser__jobs
when_to_use: "Use when the user wants to browse a website, scrape web content, fill out forms, automate browser interactions, take screenshots, run security audits, or interact with any web page. Examples: 'open this URL', 'scrape that page', 'fill out the form', 'take a screenshot', 'check this website', 'automate the login', 'run a security audit', 'monitor network requests'."
argument-hint: "[url or task description]"
context: fork
metadata:
  author: coffeegrind123
  version: "2.0"
  server: "zendriver-mcp-next >= 0.5.0"
---

# Browser Automation

Server: [zendriver-mcp-next](https://github.com/coffeegrind123/zendriver-mcp-next), registered
as `browser`, so its tools are `mcp__browser__*`. One headed Chrome per server process.

## What you can call

**Listed directly** (19):

```
start_browser  stop_browser  get_browser_status  navigate  get_interaction_tree
click  type_text  press_key  get_text_content  screenshot  execute_javascript
wait_for_element  list_tabs  scroll
search_tools  describe_tool  call_tool  batch  jobs
```

**Everything else** exists on the server but is not listed: tabs (`new_tab`, `switch_tab`,
`close_tab`), history, element queries, forms, cookies, storage, logs, network control,
interception, proxy, emulation, permissions, accessibility, tracing, Lighthouse, screencasts,
Cloudflare, security audit, human-like input. Reach it in one hop:

```
search_tools(query="block requests")              → ranked matches with signature, effect
describe_tool(name="mock_response")               → input AND output schema
call_tool(name="mock_response", arguments={...})  → same validation and result as a direct call
```

Below, an unlisted tool is written by its bare name; invoke it with `call_tool`. If a name
returns `unknown_tool`, the error's hint names the closest real tool.

## Lifecycle

```
start_browser() → navigate(url) → [work] → stop_browser()
```

- **Always headed.** `headless=true` is accepted and ignored (headless is what bot detection
  looks for). On a host without a screen the server needs an X display (Xvfb).
- **`navigate()` settles the page.** It waits for `document.readyState` complete, then for the
  network to go quiet, and returns `{settled, document_ready, elapsed, requests, final_url}`.
  Do not add a wait after it. `settled: false` is normal for pages that poll or stream.
  `settle=0` returns as soon as the navigation commits.
- **Options:** `proxy="http://user:pass@host:port"` or `socks5://`, `user_data_dir=` for a
  persistent profile, `window_size="1440x900"`. Leave `low_memory` false on protected sites
  (its `--disable-gpu`/software-WebGL flags are a bot signal).

## Results and errors

Every tool returns structured fields plus a one-line `summary`. Read the fields, not prose.

Every failure is an error result `{error, code, hint}`. **Follow the hint.** Codes you will see:

| code | meaning | do |
|---|---|---|
| `browser_not_started` | no browser | `start_browser()` |
| `page_not_loaded` | no page in this tab | `navigate(url)` |
| `element_not_found` | selector matched nothing | `find_element` (lists what exists) or `get_interaction_tree` |
| `stale_id` | the id's element is gone (navigation or re-render) | re-run `get_interaction_tree` |
| `element_obscured` | something covers the element; the error names it | dismiss it (cookie banner, modal), or `force=true` |
| `element_hidden` | no visible box | `scroll_to_element`, or open what reveals it |
| `tab_not_found` / `tab_forbidden` | bad tab id / another client's tab | `list_tabs` |
| `invalid_argument` | wrong arguments | fix them as the hint says |
| `js_error` | your `execute_javascript` script threw | fix the script |
| `timeout` / `operation_timeout` | the tool's budget ran out / something inside it gave up | check `get_browser_status`, retry once |
| `browser_unreachable` | Chrome wedged and was discarded | `start_browser()` |
| `unknown_tool` | not a tool name | the hint suggests the closest |

## Seeing the page: get_interaction_tree()

The first step on any page. Compact JSON of everything interactive, each with a numeric id:

```json
{"id": 12, "t": "btn", "l": "Submit", "r": "main"}
{"id": 13, "t": "in",  "l": "Email",  "r": "main", "it": "email"}
{"id": 14, "t": "link", "l": "Pricing", "r": "nav", "h": "/pricing"}
```

`t` type (`btn link in sel chk rad tab mnu el`), `l` label, `r` region
(`hdr nav main side ftr dlg`), `v` value, `ck` checked, `dis` disabled, `off` off-screen,
`h` link target (only with `links=true`). Shadow DOM is included.

- **Ids are stable for the element's life.** Re-walking keeps an element's id; a new
  element gets a new number; numbers are never reused. After a navigation every id is
  stale, and using one fails with `stale_id` instead of hitting a different element.
- Pass an id as a string anywhere a selector goes: `click(selector="12")`.
- The tree is truncated at `limit` (150); the result says so. Raise it on dense pages.
- Empty tree on an SPA? `wait_for_element` for something you expect, then re-walk;
  `find_buttons` / `find_inputs` use different detection.

Details: `reference/dom-walker.md`.

## Acting

| Need | Call |
|---|---|
| Click | `click(selector="12")`, `click(selector="#buy")` or `click(text="Add to cart")`. A real mouse click with pointer, mouse and click events (`isTrusted: true`). Fails with `element_obscured` if something covers it. |
| Type | `type_text(text="a@b.c", selector="13")`: clicks to focus, then inserts. It appends, so `clear_input` first to replace. |
| Keys | `press_key(key="Enter")`, `press_key(key="a", modifiers=["Control"])`, `press_key(key="Tab", selector="13")`. Real key events. |
| Human-paced | `human_click` (curved mouse path), `human_type(wpm=220)` for sites that time input |
| Many fields | `fill_form(form_data={"#email": "a@b.c", "#pass": "x"})` (an object or a JSON string); returns `missing` selectors |
| Dropdown | `select_option(selector="#country", value="FI")` matches value, then label. Its change event is untrusted; for trusted input focus the select and use `press_key` |
| Submit | `submit_form()` uses `requestSubmit()`, so the page's validation and handlers run |
| Upload | `upload_file(selector="input[type=file]", file_path="/abs/path")` |
| Scroll | `scroll(direction="down", amount=800)`, `scroll_to_element(selector=...)` |
| Shadow DOM | `click_shadow(selector="nes-button")`, `describe_shadow` |

## Reading

| Need | Call |
|---|---|
| Visible text | `get_text_content()`, paged: `offset`, `max_chars`; result has `next_offset` |
| HTML | `get_content()` (large; use sparingly) |
| One element | `find_element`, `get_element_text`, `get_element_attribute` |
| Data in one pass | `execute_javascript(script="return [...document.querySelectorAll('h2')].map(h => h.textContent)")`; `timeout=` (up to 115 s) for scripts that poll or retry |
| Look at it | `screenshot()` returns the image (≤1024px wide unless `full_resolution=true`); `save_path=` also writes the file and returns a `browser://artifacts/` link |
| Where am I | `get_page_info()` reads the live document title and URL |

**`execute_javascript`** takes an expression (`document.title`) or a statement block with `return`
(run as an async function body, so `await fetch(...)` works). Promises are awaited. It
runs in the page's own JavaScript world, which the page can in principle observe; the
dedicated tools run in an isolated world the page cannot see, so prefer them. A script that
outlives its `timeout` (default 20 s) keeps running in the page: if it clicks or submits,
check the page before running it again.

## Waiting

```
page load        → nothing; navigate() already settled
element appears  → wait_for_element(selector, timeout=10, visible=true)   (listed)
traffic settles  → wait_for_network(timeout=5)   AFTER a click or submit
one API call     → wait_for_request(url_pattern="api/orders")
```

`wait(seconds)` is the last resort, for animations and rate limits.

## Tabs

- `new_tab(url)` opens a tab and makes it active; `list_tabs` also shows tabs the page
  opened itself (popups, `target=_blank`).
- **Act on another tab without switching:** every page tool takes `tab="tab_2"`:
  `get_text_content(tab="tab_2")`. The active tab stays where it was.
- `switch_tab(tab_id)`, `close_tab(tab_id)`. Close what you open.
- Network and console log entries carry their `tab`; pass `tab=` to filter them.

## Several steps at once: batch

For a sequence you already know, one call instead of five. Later steps read earlier results:
`"$N.field"` (whole value) or `"${N.field}"` (inside a string), 0-based, list items by index.

```
batch(steps=[
  {"tool": "navigate",        "arguments": {"url": "https://example.com/login"}},
  {"tool": "type_text",       "arguments": {"selector": "#email", "text": "a@b.c"}},
  {"tool": "type_text",       "arguments": {"selector": "#pass",  "text": "secret"}},
  {"tool": "press_key",       "arguments": {"key": "Enter", "selector": "#pass"}},
  {"tool": "wait_for_network","arguments": {"timeout": 5}},
  {"tool": "get_page_info",   "arguments": {}}
])
```

At most 50 steps per call. It stops at the first failure (`failed_at`, with that step's
error) unless `stop_on_error=false`. Nothing is rolled back.

## Slow work: jobs

Lighthouse, heap snapshots, screencast export, long waits, Cloudflare:

```
jobs(action="start", tool="run_lighthouse", arguments={"url": "...", "form_factor": "mobile"})
jobs(action="wait",  job_id="job_…", timeout=120)   → state, progress, result
```

Also `get`, `cancel` and `list`. Finished jobs are kept for an hour.

## Network work

```
Watch traffic   → clear_logs() → [action] → wait_for_request(url_pattern=...) / get_network_logs()
Mock a response → mock_response(url_pattern="*/api/orders", status=200, body="[]",
                                headers={"Content-Type": "application/json"})
Fail requests   → fail_requests(url_pattern="*/analytics/*", error_reason="BlockedByClient")
Rules           → list_interceptions() (with match_count), stop_interception(rule_id?)
Block / headers → block_urls(patterns=[...]), set_extra_headers(headers={...})
```

Interception rules belong to the tab they were created on, and they work together with an
authenticated proxy.

## Auth and state

- `get_cookies()` (this page's cookies, HttpOnly included), `set_cookie(name, value, ...)`;
  whole browser: `list_all_cookies`, `export_cookies(file_path)`, `import_cookies(file_path)`.
- SPAs often keep tokens in `get_local_storage()`, not cookies. Check both, and the URL
  after an OAuth redirect.
- A persistent login: `start_browser(user_data_dir="/path/profile")`, or log in once and
  `export_cookies` / `import_cookies`. Recipe: `reference/patterns.md`.

## Cloudflare

```
is_cloudflare_challenge_present(timeout=5)   → {present}
bypass_cloudflare(timeout=30)                → {solved, attempts, elapsed}
```

- A headed browser with trusted input passes many challenges with no click; check before
  solving.
- **When `bypass_cloudflare` times out, look before retrying.** `get_content()` showing a
  Turnstile iframe at `.../challenge-platform/.../auto/failure` (or a
  `cf-turnstile-feedback` wrapper) is a hard fail: the widget rejected the browser and will
  never render a checkbox. Retrying cannot help; change approach (another IP, or the
  options below).
- A site-wrapped Turnstile (its own `<form action="/verify...">` with a filled
  `cf-turnstile-response`) may need the form submitted after the solve: `submit_form()`.
- For **public** markup only (robots.txt, sitemaps, HTML served to crawlers by design), a
  crawler user agent with plain `curl` often skips the gate entirely. Static assets
  (favicons, images) are often not challenged at all:
  `curl -A "<browser UA>" --compressed <url>`. Never use this for anything gated, paid or
  private.
- `set_user_agent` / `set_locale` / `set_timezone` / `set_geolocation` align the
  fingerprint with a proxy's location.

## Rules

1. **Work autonomously.** Run the whole workflow; ask only when the task is genuinely
   ambiguous.
2. **Use the tree, not HTML,** for discovery. `get_content` costs many times more tokens.
3. **Don't wait after `navigate`.** Wait after clicks and submits that start traffic.
4. **Dismiss overlays first.** `element_obscured` names the cover. Click its
   accept/close control, then retry.
5. **An empty read says why.** `get_text_content` on an empty page reports
   `still loading`, `no body element`, `has markup` (iframe or shadow root) or
   `really is blank`. Read that before concluding you were blocked.
6. **Clear logs before** the action you want to observe.
7. **Diagnose with a screenshot** when something does not behave as expected.
8. **Close tabs you opened**, and `stop_browser()` at the end.

## Self-refinement

When you learn something non-obvious (a site quirk, a timing need, a server behaviour),
append it to `LEARNINGS.md` in the format given there. Keep entries generic: describe the
kind of site, never its name. When a learning is about the server itself, also say which
server version it applies to.

## References

- `reference/tools.md`: every tool, with effect, tab support, parameters and summary
  (generated from the server).
- `reference/patterns.md`: recipes (login, pagination, wizards, iframes, infinite scroll,
  polling).
- `reference/dom-walker.md`: interaction tree internals.
