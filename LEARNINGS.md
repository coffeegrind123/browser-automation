# Browser Automation Learnings

Append-only file. Add entries as you discover non-obvious behaviors,
site-specific quirks, timing requirements, or selector patterns.

## Format

```
## YYYY-MM-DD: <brief title>
- **Context**: What you were doing
- **Learning**: What was non-obvious
- **Rule**: The new rule to follow
```

---

<!-- Add learnings below this line -->

## 2026-03-21: React/SPA forms need native value setter in Chrome
- **Context**: Filling a React-based login form — standard `type_text()` put both values in the same field
- **Learning**: React inputs don't respond to normal DOM value changes. Use `execute_js` with native setter: `Object.getOwnPropertyDescriptor(HTMLInputElement.prototype, 'value').set.call(input, value)` then `dispatchEvent(new Event('input', {bubbles: true}))`
- **Rule**: For React/Vue forms in Chrome, use native value setter via execute_js. For Firefox/Camoufox, use Playwright's `fill()` method instead.
- **Server 0.5.0**: `type_text` now focuses with a trusted click and inserts via CDP `Input.insertText`, which React sees as real input; the native-setter workaround is probably unnecessary. Unverified: use it only if a field still ignores `type_text`.

## 2026-03-21: Cookie consent banners block all interaction
- **Context**: A login page had a cookie consent overlay covering the input fields
- **Learning**: Cookie banners use fixed/absolute positioning and cover interactive elements. Clicks silently hit the banner instead of the target element. No error is thrown — it just doesn't work.
- **Rule**: Always check for and dismiss cookie banners immediately after navigation: `click(text="Accept All")` or similar. Do this before any other interaction.
- **Server 0.5.0**: no longer silent. A covered target fails with `element_obscured` and names the covering element.

## 2026-03-21: JWT tokens in localStorage, not cookies
- **Context**: After an OpenID login to a Vue SPA, `get_cookies()` returned empty but the app was authenticated
- **Learning**: The SPA stored a JWT in `localStorage.token` and sent it via `Authorization: Bearer` header. Many modern apps do this — cookies are only for server-rendered sites.
- **Rule**: When investigating auth, check both `get_cookies()` AND `get_local_storage()`. Look for `token`, `jwt`, `access_token` keys.

## 2026-03-21: Auth tokens can appear in redirect URL params
- **Context**: After completing an OpenID flow, the app redirected to `/return?token=eyJ...`
- **Learning**: Some apps pass the auth token in the URL query string on the OAuth redirect, before the SPA processes it into localStorage. If localStorage is empty, check `window.location.href` for `?token=` params.
- **Rule**: When extracting auth tokens, check URL params first, then localStorage, then cookies.

## 2026-03-21: get_interaction_tree() can return empty on SPAs
- **Context**: Vue.js SPA (Vuetify framework) — interaction tree returned `[]`
- **Learning**: The DOM walker may not find elements in Shadow DOM or dynamically-rendered SPA components. `find_buttons()` and `find_inputs()` use different detection logic and may succeed where the tree fails.
- **Rule**: If `get_interaction_tree()` returns empty, try `find_buttons()`, `find_inputs()`, or `execute_js` to query the DOM directly.

## 2026-07-11: low_memory launch flags are Cloudflare-detectable
- **Context**: Inspecting a site behind Cloudflare Turnstile. `bypass_cloudflare` timed out ("could not solve the checkbox") when the browser was started with `low_memory=true`.
- **Learning**: `low_memory` adds `--disable-gpu` / software-WebGL flags that anti-bot fingerprinting flags as automation, so the Turnstile never clears. Relaunching WITHOUT `low_memory` (plain `start_browser`) let the checkbox solve on the same site.
- **Rule**: For any Cloudflare/anti-bot site, launch `start_browser` WITHOUT `low_memory` (accept the higher memory use). Only use `low_memory` on constrained hosts for non-protected pages.

## 2026-07-11: execute_js does NOT await Promises (async returns {})
- **Context**: Ran an `async` IIFE in `execute_js` (a `fetch()` to grab a favicon) — the tool returned `{}` with no data.
- **Learning**: The wrapper serializes the return value synchronously; an `async` function returns a Promise that serializes to `{}`. `await`/`.then` results never come back.
- **Rule**: Keep `execute_js` synchronous. For network, either navigate directly to the resource URL and read it from the resulting document, or use a synchronous `XMLHttpRequest` (`xhr.open(..., false)`). Never rely on `async`/`await` inside `execute_js`.
- **Superseded (server 0.5.0)**: `execute_js` awaits promises and accepts statement blocks with `return` (run as an async body), so `return await fetch(u).then(r => r.text())` works.

## 2026-07-11: site-wrapped Turnstile needs a manual form submit after bypass
- **Context**: `bypass_cloudflare` returned "solved" and the widget showed a green "Success!", but the page stayed on "Verification required" — a site-custom challenge, not the standard CF interstitial.
- **Learning**: Some sites embed Turnstile in their OWN `<form action="/verify_captcha" method="post">` with a populated hidden `<input name="cf-turnstile-response">` (the solved token, ~700 chars) and a manual "Verify" submit button. `bypass_cloudflare` only solves the checkbox; it does not submit the form, and `click(text="Verify")` may not trigger the submit.
- **Rule**: After `bypass_cloudflare` shows "Success!" but the page doesn't advance, inspect `document.forms` for a challenge form with a filled `cf-turnstile-response`, then submit it directly: `execute_js("(function(){document.forms[0].submit();return true})()")`.

## 2026-07-11: static assets often bypass the JS challenge — fetch with curl
- **Context**: Needed each site's favicon; some sites were Cloudflare-gated on the HTML.
- **Learning**: CF challenges HTML *navigations*, not always static assets. `curl -A "<a real Chrome UA>" --compressed <favicon-url>` returned the real PNG (HTTP 200) even on a challenged domain. Without `--compressed` you get raw gzip bytes (file says "gzip compressed data", not "PNG").
- **Rule**: To pull static assets (favicons/images/css) from a CF-protected site, try `curl -A "<browser UA>" --compressed` on the host first — it's far simpler than driving the browser. Always include `--compressed`.

## 2026-07-11: cross-origin image → base64 needs a same-origin document
- **Context**: Tried to canvas a favicon into base64 from a page on a different origin — `canvas.toDataURL()` throws a taint SecurityError.
- **Learning**: Canvas read-back is blocked for cross-origin images without CORS. But navigating the browser DIRECTLY to the image URL makes the generated document same-origin with the image, so `document.images[0]` canvases cleanly. `https://www.google.com/s2/favicons?domain=<host>&sz=128` is also same-origin (google.com) and canvas-readable — but it often returns a generic/greyscale icon, so prefer the site's real favicon URL.
- **Rule**: To get a cross-origin image as base64, `navigate` to the image URL itself, then canvas `document.images[0]`. Prefer downloading the raw bytes with curl (previous entry) when you just need the file.

## 2026-07-16: Turnstile `/auto/failure` = hard-fail; bypass_cloudflare can never solve it
- **Context**: Reading another site's public SEO markup (robots.txt/sitemap.xml/HTML). `is_cloudflare_challenge_present()` returned true, but `bypass_cloudflare(timeout=30)` timed out on "Could not solve the checkbox".
- **Learning**: Two things were non-obvious. (1) It was NOT the standard CF edge challenge — the site self-hosted a Turnstile widget in its own `<form action="/verify_captcha">` with a `data-sitekey`. (2) `get_content()` showed the widget iframe at `.../challenge-platform/h/b/fr/<id>/en-us/auto/failure` — Turnstile had already fingerprinted the automated browser and REJECTED it, rendering a feedback report instead of a checkbox. `bypass_cloudflare()` polls for a checkbox, so with no checkbox in the DOM it can only ever time out. Retrying / longer timeouts / re-clicking cannot help.
- **Rule**: When `bypass_cloudflare()` times out, `get_content()` and look for `/auto/failure` or a `cf-turnstile-feedback` wrapper before retrying anything — that's a hard-fail, so change approach instead of clicking. And when the goal is PUBLIC content (markup/robots/sitemap), try a crawler UA with plain curl FIRST: many sites allowlist crawlers by UA string alone with no reverse-DNS check, so `curl -sA "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)"` walks past the gate and returns real HTML with no browser at all. Restrict this to content the site already serves to crawlers by design — never anything gated, paid or private. Also: don't pass `low_memory=true` when stealth matters (software WebGL / `--disable-gpu` are bot signals).

## 2026-07-21: bypass_cloudflare aborted on Turnstile iframe re-render — now resilient (headed only)
- **Context**: A community download site behind a real CF Turnstile. `bypass_cloudflare()` errored *immediately* (not a timeout): "Node with given id does not belong to the document" (`DOM.resolveNode`).
- **Learning**: zendriver's `verify_cf` captures the Turnstile checkbox/input nodes ONCE, up front — but the challenge iframe re-renders mid-solve, so those `backendNodeId`s go stale and the whole solve aborts on the first re-render. Separately, it only works HEADED (`start_browser(headless=false)`, on Xvfb `:99` here); in headless the click is rejected — I wasted two tries in headless.
- **Rule**: Fixed in the server (old fork build ≥ `5a5654e`): `bypass_cloudflare()` now retries `verify_cf` from scratch each round so every attempt re-finds FRESH nodes, swallows the stale-node/transient error, polls for auto-clear (a headed stealth browser passes many managed challenges with no click), and reloads once. If you still see the stale-node error, the MCP is on an old build — restart it. And always launch HEADED for CF sites. Confirmed: clears the gate that the old build died on.
- **Server 0.5.0**: the resilient solver is in zendriver-mcp-next; every launch is headed (headless is refused), and the solver reports progress per attempt.

## 2026-08-17: navigate did not wait, and wait_for_network went blind past 100 requests
- **Context**: A pi session (2026-08-17T05-31) was asked for the news. It did the obvious correct thing — `navigate("https://www.bbc.com/news")` then `get_text_content(max_chars=5000)` — and got `[chars 0-0 of 0]`. Four round trips went into recovering: a `setTimeout` that does not exist in the caller's sandbox, a tool search for a wait, then an explicit `wait_for_network`, before the same read returned 8,029 characters.
- **Learning**: Three separate defects, all in the server. (1) `navigate`'s docstring promised it would "wait for the page to load" and it did not — `session.navigate()` calls zendriver's `get()`, which returns when the navigation COMMITS, not when anything has rendered. A believed-but-false promise is worse than no promise: the caller reads too early and blames the site. (2) `wait_for_network` compared `len(get_network_logs(100))` against its previous value, and that list is CAPPED at 100 entries — so once a page had made 100 requests the count could never change and it declared idle after one `idle_time` regardless. The failing session shows the tell exactly: "Network idle after 0.6s (**100** requests captured)" — the cap, not a measurement. It defeated the wait precisely on the busy pages that need one. (3) `get_text_content` returning a bare `[chars 0-0 of 0]` is indistinguishable from a page that genuinely has no text, which is why that session went hunting for a consent wall.
- **Rule**: Fixed in the server (old fork build ≥ `f29626c`). **Do not add a wait after `navigate()`** — it now settles on network idle itself and reports how that ended (`network idle after 0.6s, 246 requests` / `network still active after 10.0s`); `settle=0` restores return-on-commit. Waits belong AFTER a click or an action that starts new traffic, not after a navigation. An empty `get_text_content` now appends the reason (`still loading` / `no body element` / `has markup` → iframe or shadow root / `really is blank`) — read that line before concluding you were blocked. If you still see a bare `[chars 0-0 of 0]` or a request count pinned at exactly 100, the MCP is on an old build: restart it.
- **Server 0.5.0**: still true, and `navigate` now returns the outcome as fields (`settled`, `document_ready`, `elapsed`, `requests`).
