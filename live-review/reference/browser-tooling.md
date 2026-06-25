# Browser tooling

This skill is **tool-agnostic by design** — Step 2 is "find the browser tool." It needs a
driver that can: navigate, screenshot, resize the viewport, click/type/scroll, run JS in the
page, and ideally capture console + network. Below are the drivers it knows, best first, with
copy-ready recipes. Discover what's present, pick the best one, and use it consistently for
the whole walk.

To use a driver not listed here, add its tools to the skill's `allowed-tools` and follow the
same recipe shape (navigate → screenshot → set viewport → interact → inspect).

---

## 1. qa-master MCP (preferred)

Purpose-built headless-browser QA, exposed as MCP tools (`mcp__qa-master__*`). Two patterns —
use both:

### Pattern A — drive it yourself (precise, step-by-step)

| Need | Tool |
| --- | --- |
| Open a URL in a fresh tab | `qa_open(url)` |
| Navigate the same tab (keeps viewport override) | `qa_goto(url)` |
| Screenshot now | `qa_screenshot(label, full_page=False)` |
| Responsive resize | `qa_set_viewport(width, height, mobile=)` |
| Click / type / scroll / key | `qa_click`, `qa_type`, `qa_scroll`, `qa_press_key` |
| Inspect or script the page | `qa_js(expression, await_promise=)` |
| Wait for load / a condition | `qa_wait`, `qa_wait_for` |
| Several actions in one round-trip | `qa_batch` |
| Release the tab when done | `qa_close()` |

Responsive sweep — the viewports the skill defaults to:

```
qa_set_viewport(375, 812, mobile=True)    # phone
qa_set_viewport(768, 1024, mobile=True)   # tablet (when layout is in doubt)
qa_set_viewport(1280, 800)                # desktop
```

`qa_screenshot` and most actions return an image, so you *see* the result of every step —
screenshot, read the pixels, decide the next click.

### Gotcha: some MCP clients don't surface `qa_js` return values

Field-tested: in some clients (e.g. Claude Code), a `qa_js` call comes back with only the
**screenshot + page metadata** (`URL` / `Title` / `Viewport` / `Page`) — the expression's
*return value* is silently dropped. So `qa_js: JSON.stringify(window.__qa)` runs but you never
see the JSON, which quietly breaks every "read a value back" recipe below.

The reliable workaround is to route the value through a field the client **does** echo —
`document.title` is perfect, because the metadata line always shows `Title`:

```js
// Instead of `return X` (may not surface), write X into the title and read it off the line:
qa_js: `document.title = '[QA] overflow=' + (document.documentElement.scrollWidth > document.documentElement.clientWidth)
  + ' h1=' + document.querySelectorAll('h1').length
  + ' errs=' + (window.__qa ? window.__qa.errors.length : 'NC')`
// → output shows:  Title: [QA] overflow=false h1=1 errs=0
```

Keep payloads short (a title line, not a JSON blob), prefix with a marker like `[QA] ` so it's
easy to spot, and remember the title is per-session DOM only — it doesn't touch the real page.
Confirm your client first: run a trivial `qa_js` that returns a string; if you don't see it,
switch to the title channel for all reads. (Native console/network capture in a Playwright MCP
sidesteps this entirely — see below.)

### Pattern B — hand off a checklist (cheap on context)

`qa_request(target, checklist)` spawns an in-container agent that opens the URL, works the
checklist on its own, and returns a pass/fail verdict plus screenshots — its intermediate
shots stay in the container, so a 20-step walk costs you one verdict, not 20 images. Use it
to exercise a whole journey at once, then drill into failures with Pattern A.

```
qa_request(
  target="https://staging.example.com/get-started",
  checklist=[
    "Fill the signup form with a test email and a valid password.",
    "[screenshot: before-submit]",
    "Click 'Create account'.",
    "Verify navigation to /app (check the URL with JS).",
    "Verify there are no console errors (check via JS).",
    "[screenshot: after-submit | full]",
  ],
)
```

Tips: be specific per item; for hard guarantees ("no 404s", "image naturalWidth > 0", "exact
count") say **"verify with JS"** or the agent may only eyeball it. `[screenshot: NAME]` items
are capture points, not checks — place one right after the step it should picture; append
`| full` for a full-page shot.

### Capturing console errors & failed requests with qa-master

qa-master doesn't stream the console, so install collectors with `qa_js` and read them back:

```js
// Right after qa_open / qa_goto, install collectors:
qa_js: `
  window.__qa = { errors: [], rejections: [] };
  window.addEventListener('error', e => window.__qa.errors.push(String(e.message)));
  window.addEventListener('unhandledrejection', e => window.__qa.rejections.push(String(e.reason)));
  const _err = console.error; console.error = (...a) => { window.__qa.errors.push(a.join(' ')); _err(...a); };
  'installed'
`
// …interact…
// Then read them back. If your client surfaces qa_js return values:
qa_js: `JSON.stringify(window.__qa)`
// If it doesn't (see the gotcha above), route the count through the title instead:
qa_js: `document.title = '[QA] errs=' + window.__qa.errors.length + ' rej=' + window.__qa.rejections.length`
```

Collectors are wiped on navigation — re-install after each `qa_goto`, and re-load the page
*after* installing if you need to catch load-time errors. For failed network requests, read
the Resource Timing buffer (status codes aren't there, but failures and slow assets are):

```js
qa_js: `JSON.stringify(
  performance.getEntriesByType('resource')
    .filter(r => r.transferSize === 0 && r.duration > 0 || r.duration > 1000)
    .map(r => ({ url: r.name, ms: Math.round(r.duration), size: r.transferSize }))
)`
```

For real status codes, either probe a suspect endpoint directly (`fetch(url).then(r=>r.status)`
via `qa_js` with `await_promise=True`) or use a driver with native network capture (below).

---

## 2. Playwright MCP (if present)

If a Playwright MCP is connected, prefer it when you need **native console and network
capture** — it surfaces console messages and request/response (with status codes) without the
collector dance above. Map the same recipe: navigate → `browser_resize` → screenshot →
click/type → `browser_console_messages` / `browser_network_requests` to read errors. Use it
especially when chasing a 4xx/5xx or a noisy console.

---

## 3. browser-harness (CDP, via Bash)

If `browser-harness` is on `$PATH`, it's a CDP driver invoked through Bash with a heredoc:

```bash
browser-harness <<'PY'
new_tab("https://staging.example.com")   # first nav is new_tab, not goto_url
wait_for_load()
print(page_info())
capture_screenshot()
PY
```

Coordinate clicks are the default (`capture_screenshot()` → read the pixel → `click_at_xy(x,y)`
→ re-screenshot). For network capture it has a dedicated interaction skill
(`interaction-skills/network-requests.md`) and for viewport work `interaction-skills/viewport.md`.
Note: browser-harness connects to the **user's running Chrome** — don't drive it on a site the
user is actively using, and prefer a fresh tab.

---

## Simulating slow / failed networks

Part of the live review is the unhappy network path (Step 3). Options, by driver:

- **qa-master:** no first-class throttle; provoke failures by exercising flows while a backend
  is down, or probe endpoints with `qa_js` + `fetch`. Note the limitation rather than faking it.
- **Playwright MCP:** route-based request interception/abort where available.
- **browser-harness:** raw CDP — `cdp("Network.emulateNetworkConditions", {...})` for throttle
  and `cdp("Network.enable")` + request interception for forced failures.

Whichever you use, when you *can't* simulate a condition, say so in the report instead of
guessing — an unverified perf/error claim is worse than a noted gap.

---

## What "good enough" looks like

You don't need all the fancy capture to deliver value. The minimum viable loop is:
**open → screenshot → resize to mobile → screenshot → click the primary CTA → screenshot →
read `window.__qa` for errors.** That alone catches dead buttons, broken mobile layout, and
JS errors — the three most common live failures. Everything else is depth on top.
