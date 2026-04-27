# browser_fetch

Fetch a URL through headless Chromium with article extraction. The "heavy" tier of the three-tier web ladder (`webFetch` → `browserFetch` → `browser-use` skill — see `architecture.md` → Components → Agent).

## Description (shown to the model)

Fetch a URL through a real browser engine (headless Chromium) and return the extracted article content. Use this when:

- The page is JS-heavy and `webFetch` returns empty or shell-only HTML (single-page apps, infinite-scroll articles, dynamic content).
- `webFetch` reports the URL is bot-blocked (X/Twitter, Cloudflare-gated articles, sites with bot detection).
- You need the rendered page, not the raw HTML.

For static pages, JSON APIs, plain HTML, anything where a plain HTTP fetch would work, prefer `webFetch` — it's an order of magnitude faster (200ms vs ~1–2s) and doesn't burn ~150MB of memory. For multi-step interactions (login, form fill, navigation across pages), use the `browser-use` skill instead.

**This tool is anonymous.** Each call gets a fresh, logged-out browser context. It does **not** share sessions with the user's Chrome or with `browser-use` (which connects to the user's logged-in Chrome via CDP — different mechanism, different blast radius). If a page requires authentication or a captured session, use `browser-use`, not `browser_fetch`.

## Input

```ts
{
  url: string,
  session?: string,  // Optional. Reuse a persisted browser context across calls.
}
```

`session` is a freeform name (e.g. `"nyt"`, `"reddit"`). When omitted, the call gets a fresh ephemeral context — no cookies, no localStorage, nothing carried in. When provided, the call uses a **persistent context** stored under `runtime/browser-sessions/{session}/` (gitignored, machine-local). Cookies, localStorage, and IndexedDB persist across calls **and across daemon restarts**, so a CAPTCHA solved once stays solved, a free-tier login the model performed (via `bash` + `webFetch` cookie injection or similar) keeps working, and a metered paywall counts the same browser as one visitor.

Sessions are isolated from each other and from the ephemeral default — using `session: "nyt"` and `session: "reddit"` gets you two separate cookie jars. Sessions are also isolated from the user's Chrome and from `browser-use`; only `browser_fetch` calls reading the same session see the same state.

Sanitize `session` to a safe path segment — no `/`, `..`, or other path-escape characters. Reject the call with a clear error otherwise.

## Output

```ts
{ url: string, title: string, content: string }
```

- `url` — the final URL after any redirects.
- `title` — the article title from Mozilla Readability, or the page's `<title>` if Readability couldn't extract.
- `content` — the cleaned article text. Extracted via Mozilla Readability over the rendered HTML, so it strips nav, footer, ads, and comment sections.

## Behavior

Pipeline: `chromium.launch({ headless: true })` → fresh `BrowserContext` (cookie isolation) → **two-phase wait** (see below) → `page.content()` → Mozilla Readability over the rendered HTML → return `{ url: page.url(), title: article.title, content: article.textContent }`.

**Two-phase wait — load + best-effort networkidle.** A single `waitUntil: 'networkidle'` with a 30s ceiling burns 30 full seconds on continuously-polling SPAs (X/Twitter, dashboards, news live-blogs) where analytics, websocket-ish polling, and ad telemetry keep the network busy indefinitely. Use two stages instead:

```ts
await page.goto(url, { waitUntil: "load", timeout: 15_000 });
await page.waitForLoadState("networkidle", { timeout: 5_000 }).catch(() => {});
```

- **Phase 1 (`load`, 15s ceiling):** waits for the `window.load` event — DOM parsed, all subresources loaded. Most pages reach this in 1–3s.
- **Phase 2 (`networkidle`, 5s ceiling, best-effort):** gives JS frameworks (React/Vue/etc) time to finish hydration after `load`. The `.catch(() => {})` swallows the timeout — if the page never goes idle (polling SPAs, live blogs), proceed with whatever's rendered. Five seconds is enough for late hydration without burning latency on sites that won't settle.

Read `page.content()` after both phases regardless of whether either timed out. The page is almost always fully rendered by then.

**Failure handling.** Phase 1's timeout is a real failure — if `load` doesn't fire in 15s, the page is genuinely broken (DNS error, connection refused, server hung). Wrap that as `browser_fetch: navigation timeout for {url}`. Phase 2's timeout is **not** a failure — `.catch(() => {})` swallows it and the pipeline continues. The reference probe at `~/bin/playwright-probe` shipped with a single-phase 30s wait and worked because the probe always reads the page after the timeout; production caps the wait tighter so the model isn't sitting through 30s tool calls on sites that never settle.

- **Browser singleton — kept running for the daemon's lifetime.** Lazy-launched on the **first** `browser_fetch` call after `agent/protos start`; reused for **every** subsequent ephemeral call across channels, sub-agents, and cron jobs; closed only when the daemon exits or restarts. Cold start (~1–2s) pays once per daemon, not per fetch. Warm calls land at ~500ms–1.5s depending on the page. Do not relaunch the browser per dispatch — that defeats the entire reason this tool exists.
- **Ephemeral path** (no `session`): a fresh `BrowserContext` and `Page` per call off the singleton browser, both closed in a `finally` block. The `Browser` stays up.
- **Session path** (`session: "name"` provided): use `chromium.launchPersistentContext({ userDataDir: 'runtime/browser-sessions/{name}/', headless: true, ... })`. Persistent contexts can't share a `Browser` with each other or with the ephemeral singleton — Chromium locks each `userDataDir` to its launching process — so each named session gets its own cached `BrowserContext` (which transitively owns its own headless Chromium). Cache them in a module-scope `Map<string, BrowserContext>`; reuse on subsequent calls with the same `session`. A fresh `Page` per call inside the cached context (cookies persist, but each call starts on a blank tab). Close every cached persistent context on daemon shutdown alongside the ephemeral singleton.
- **Realistic user agent.** A current desktop Chrome string. Default Playwright UA gets flagged as a bot more often than the realistic one.
- **Failure modes** surface as tool errors (the SDK's tool-dispatch layer turns thrown errors into tool results the model sees). **Wrap-all rule:** every error thrown from this tool prefixes its message with `browser_fetch:` and includes the URL where one is in scope — including failures from `chromium.launch`, `launchPersistentContext`, `newContext`, `newPage`, `safeSegment` on the session name, and the navigation/render path below. The model gets a consistent surface and can reliably detect "this is a browser_fetch failure, fall back to webFetch." Common cases (the message shapes for the most frequent paths):
  - Phase-1 (`load`) timeout (>15s) → `Error("browser_fetch: navigation timeout for {url}")`. The page genuinely failed to load. Phase-2 (`networkidle`) timeout is **not** an error and never reaches the wrap — `.catch(() => {})` swallows it inline so the pipeline proceeds to `page.content()`.
  - Non-2xx HTTP status → `Error("browser_fetch: HTTP {status} for {url}")`.
  - Readability returning `null` (no extractable article) → **don't** error; fall back to `body.innerText` truncated to ~50KB. Many useful pages aren't articles, and a partial result beats a hard fail.
  - Network/DNS errors from `page.goto` → wrap and re-throw with the URL in the message.
  - Invalid `session` name (path-escape characters) → `Error("browser_fetch: invalid session name {value}")`.
- **No persistent state.** No cookies, no localStorage, no service workers carried across calls. The browser sees only what the URL renders to a fresh visitor — same blast radius as `webFetch`, just on a real engine.

## Dependencies

- `playwright` — the browser automation library. Pinned via `agent/package.json`.
- `@mozilla/readability` — article extraction.
- `jsdom` — DOM Readability parses against (Readability needs a real DOM, not a regex parser).
- Chromium binary at `vendor/chromium/`, installed at build time via `PLAYWRIGHT_BROWSERS_PATH=$WORKSPACE_ROOT/vendor/chromium npx playwright install chromium` (see `spec/build.md` → step 1). The wrapper script (`agent/protos`) exports the same `PLAYWRIGHT_BROWSERS_PATH` before launching the daemon so the runtime finds the binary.

## Implementation notes

- **Single-file tool.** `agent/src/tools/browser_fetch.ts` exports the tool definition (name, schema, execute) plus two module-local helpers: `getBrowser()` for the ephemeral singleton, and `getSessionContext(name)` for the per-session persistent contexts. Caches live at module scope:
  ```ts
  let browser: Browser | null = null;                          // ephemeral
  const sessions = new Map<string, BrowserContext>();          // persistent
  ```
  Module scope (not factory-scope) so every importer in the same process — main agent, sub-agent runner, cron scheduler — sees the same instances. **Do not** put `chromium.launch()` or `launchPersistentContext()` inside the `execute` function or inside an `Agent` factory closure; both construct fresh state per dispatch and would relaunch on every call.
- **Browser lifecycle.** Register a `process.on('exit')` (or `SIGTERM`/`SIGINT`) handler that calls `browser?.close()` and `ctx.close()` for every entry in the `sessions` Map. Don't fight the process exit; if a close fails or hangs, the process exiting cleans up anyway.
- **Session disk layout.** `runtime/browser-sessions/{name}/` holds Chromium's user-data-dir (Cookies SQLite, IndexedDB, localStorage, cache). Gitignored alongside the rest of `runtime/`. The `update` workflow leaves these alone — they're machine-local state.
- **Session sanitization.** Reject any `session` that contains `/`, `..`, leading `.`, or characters that would resolve outside `runtime/browser-sessions/{name}/`. Use the same workspace-segment safety helper threads.ts uses for channel/conversation IDs.
- **Concurrent callers — guard the launch.** The router serializes per-conversation, *not* globally; two channels (or a channel + a cron) can call `browser_fetch` simultaneously. Cache the **in-flight Promise**, not just the resolved `Browser` / `BrowserContext`, so concurrent first-callers `await` the same launch:
  ```ts
  let browserPromise: Promise<Browser> | null = null;
  const sessionPromises = new Map<string, Promise<BrowserContext>>();
  ```
  The naive nullable-and-assign pattern (`if (!browser) browser = await chromium.launch(...)`) silently leaks an orphaned Chromium process per concurrent first-caller — invisible until you inspect `ps`. On launch failure, clear the cached Promise so the next call retries instead of locking onto the rejected promise forever.
- **Concurrent throughput.** Once the singleton is up, individual `page.goto`s on its `BrowserContext`s run concurrently inside Chromium without further coordination. Persistent contexts each own their own Chromium process, so per-session calls are also parallel relative to other sessions and to ephemeral fetches. If high concurrency on the *ephemeral* singleton becomes a bottleneck, swap to a small pool — see `plan/web-search-and-fetch.md` open question 5.
- **Memory leaks.** Chromium leaks memory on multi-hour runs. v1 keeps the process up; if memory grows, recycle the browser every N fetches or on a timer — `plan/web-search-and-fetch.md` open question 6.
- **No download interception, no file uploads.** This tool reads pages; it doesn't drive interactions. Anything stateful belongs in the `browser-use` skill.
- **Don't catch the SDK's turn-cap.** `browser_fetch` doesn't iterate or recurse — it's one round-trip per call. Turn-cap concerns belong upstream in agent-sdk, not here.

## Testing

A reference probe lives at `~/bin/playwright-probe` (installed locally during the Tavily-vs-Playwright evaluation; see `plan/web-search-and-fetch.md`). It runs the same Chromium + Readability pipeline as this tool, so testing whether `browser_fetch` will work on a given URL is `playwright-probe <url>`. The production tool's only difference is the singleton browser cache; cold-start latency on the probe is the upper bound on `browser_fetch` latency.
