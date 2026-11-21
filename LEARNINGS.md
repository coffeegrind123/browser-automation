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
- **Learning**: React inputs don't respond to normal DOM value changes. Use `execute_javascript` with native setter: `Object.getOwnPropertyDescriptor(HTMLInputElement.prototype, 'value').set.call(input, value)` then `dispatchEvent(new Event('input', {bubbles: true}))`
- **Rule**: Use `type_text` first. Fall back to the native setter via `execute_javascript` only if a React/Vue field still ignores it.
- **Server 0.5.0**: `type_text` now focuses with a trusted click and inserts via CDP `Input.insertText`, which React sees as real input; the native-setter workaround is probably unnecessary. Unverified: use it only if a field still ignores `type_text`.

## 2026-07-11: cross-origin image → base64 needs a same-origin document
- **Context**: Tried to canvas a favicon into base64 from a page on a different origin — `canvas.toDataURL()` throws a taint SecurityError.
- **Learning**: Canvas read-back is blocked for cross-origin images without CORS. But navigating the browser DIRECTLY to the image URL makes the generated document same-origin with the image, so `document.images[0]` canvases cleanly. `https://www.google.com/s2/favicons?domain=<host>&sz=128` is also same-origin (google.com) and canvas-readable — but it often returns a generic/greyscale icon, so prefer the site's real favicon URL.
- **Rule**: To get a cross-origin image as base64, `navigate` to the image URL itself, then canvas `document.images[0]`. Prefer downloading the raw bytes with `curl --compressed` (SKILL.md, Cloudflare) when you just need the file.

## 2026-11-12: execute_js is now execute_javascript
- **Context**: An agent-usability lint flagged `execute_js` as a generic verb with a name its description never echoed.
- **Rule**: Call `execute_javascript`; arguments and behaviour are unchanged. Older transcripts and notes that say `execute_js` mean this tool. Through `call_tool`, `batch` or `jobs` the old name fails with `unknown_tool` and a hint naming the new one. Direct `mcp__browser__execute_js` calls fail at the client because the tool is no longer listed, so restart the MCP server and update any saved permission allowlist that names the old tool.
