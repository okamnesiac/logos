# Web search + browser fetch

Replace `webSearch`'s bundled stub with **Tavily Search** on backends where protos owns the execution path, and add a new canonical-style tool **`browserFetch`** (headless Chromium via Playwright + Mozilla Readability) available on all four backends.

Lands after the [agent-sdk migration](./agent-sdk-migration.md).

## Why

Search and fetch are different problems with different right answers.

**Search — Tavily.** Brave runs its own ~20B-page index; Tavily aggregates over Google/Bing. For a personal-assistant query mix (news, recipes, programming, random lookups) the long-tail recall advantage matters more than for a developer-only tool. Tavily's snippets are also tuned for LLM consumption (cleaned passages, deduped, optional synthesized `answer` line) where Brave returns raw SERP descriptions written for humans deciding which link to click. The privacy difference is real but marginal — queries hit a third party either way, and protos already trusts Anthropic/OpenAI with conversation content.

**Fetch — new `browserFetch` tool, not an override.** Originally we considered swapping `webFetch` to Playwright. Two reasons not to:

1. We can't replace native `webFetch` on Claude or Codex anyway — agent-sdk's `withImpls` only intercepts where the backend doesn't have native handling. So an override would only land on `openai`/`vercel`.
2. The simple `webFetch` is the right tool when JS doesn't matter (JSON APIs, plain HTML, raw markdown). Forcing every fetch through Chromium burns ~1–2s and ~150MB of memory for content the model could've grabbed in 200ms. The model should pick.

So: `webFetch` stays as-is everywhere. `browserFetch` is a new custom tool the model invokes when it actually needs a real engine. Empirically it handles JS-heavy SPAs, paywalled previews, and bot-blocked sites (X/Twitter, Cloudflare-gated articles) that Tavily Extract refused.

This also gives protos a clean three-tier ladder for web access:

| Tool | Mechanism | When |
|---|---|---|
| `webFetch` | HTTP only | Static pages, docs, JSON, anything where a node fetch would work |
| `browserFetch` | Single-shot render in headless Chromium | JS-heavy pages, bot-blocked URLs, paywalled article previews |
| `browser-use` (skill) | Interactive multi-step session via the [browser-use](../spec/skills/browser-use.md) toolkit | Multi-page flows, form fill, login, anything stateful |

## Scope

| Backend | `webSearch` | `webFetch` | `browserFetch` |
|---|---|---|---|
| `claude` | native — untouched | native — untouched | **add** (custom tool) |
| `codex` | native — untouched | native — untouched | **add** (custom tool) |
| `openai` | native (`web_search` hosted) — untouched | native node-fetch — untouched | **add** (custom tool) |
| `vercel` | **add** Tavily Search (no current default) | bundled node-fetch — untouched | **add** (custom tool) |

`webSearch` Tavily is wired via `withImpls`. `browserFetch` is added to the custom-tool catalog and ships on every backend.

## Tavily Search

- Endpoint: `POST https://api.tavily.com/search`
- Body: `{ query, max_results: 5, search_depth: 'basic', include_answer: true }`
- Auth: `Authorization: Bearer $TAVILY_API_KEY`
- Free tier: 1000 queries/mo — comfortably above personal volume.
- Map response: `results[].{title, url, content}` to canonical `webSearch` output. If `answer` is non-empty, prepend it as a synthesized summary line (the model gets both the answer and the sources).
- Errors: 429 (rate limit) and 401 (auth) surface as tool-level errors, no crash.

`search_depth: 'advanced'` costs more credits but returns deeper content. Default to `basic`; revisit if the agent reports thin results.

## browserFetch

Custom tool, single argument `url: string`. Returns `{ url, title, content }` where `content` is the Readability-cleaned article text.

Pipeline: `chromium.launch({ headless: true })` → fresh `BrowserContext` → `page.goto(url, { waitUntil: 'networkidle', timeout: 30000 })` → `page.content()` → Mozilla Readability over the rendered HTML → return `article.textContent` plus `article.title`.

- One browser process per agent run — `chromium.launch()` lazily on first use, reuse across calls; close on process exit. Cold start is the dominant cost (~1–2s); cached launches drop subsequent fetches to ~500ms–1.5s depending on the page.
- Per-fetch: a fresh `BrowserContext` (cookie isolation) and a single `Page`. Close both after extraction.
- User agent: a current desktop Chrome string. Default Playwright UA gets flagged as a bot more often than the realistic one.
- Failure modes surface as tool errors: navigation timeout, non-2xx HTTP status, Readability returning null (page has no extractable article — pass through `body.innerText` truncated to a sane size as a fallback rather than failing entirely).
- No browser-side persistent state. No download interception. The browser sees only what the URL renders to a fresh visitor.

Reference probe: `~/bin/playwright-probe` (locally installed during the Tavily-vs-Playwright evaluation) wraps the same Chromium + Readability pipeline; the production impl mirrors it.

### Browser binary

`npx playwright install chromium` runs once at build time per `spec/build.md` step 1. ~165MB on disk; downloads to `~/Library/Caches/ms-playwright/` on macOS and `~/.cache/ms-playwright/` on Linux. The build script should run this and verify the binary is present before declaring success.

## Wiring

`withImpls` runs once at agent construction in `src/agent.ts`, just for search:

```ts
const tools = withImpls(canonicalTools, {
  webSearch: tavilySearchExecute,
});
```

`browserFetch` is a new file under `src/tools/` (the custom-tool convention from `architecture.md` → File structure):

- `src/tools/browser_fetch.ts` — exports the tool definition (name, schema, execute) plus a singleton `getBrowser()` that lazy-launches Chromium and returns the cached instance. The tool loader picks it up automatically.
- `src/tools/impls/tavily.ts` — `tavilySearchExecute` plus a small `tavilyFetch` helper. Lives under `impls/` because it's a `withImpls` execute, not a standalone tool.

## Spec edit scope

- `spec/architecture.md`: short subsection naming Tavily (search) and Playwright + Readability (`browserFetch`) as canonical impls. File-structure tree gains `src/tools/impls/`.
- `spec/build.md`:
  - Key packages line — `playwright`, `@mozilla/readability`, `jsdom` (Readability needs a DOM to parse against). No new deps for Tavily (plain HTTP).
  - Step 1 gains `npx playwright install chromium` after `npm install`, with a note that the binary lands outside `node_modules/` (cache directory varies by OS) so it survives `node_modules/` rebuilds.
  - Step 9 env validation adds `TAVILY_API_KEY`. Also a "browser binary present" check (Playwright errors clearly if it's missing, but a build-time probe gives a better message).
- New `spec/tools/browser_fetch.md`: tool recipe — schema, behavior, when to use it vs `webFetch` vs `browser-use` skill, the failure-mode contract, the singleton browser lifecycle.
- New `spec/tools/web_search.md`: short recipe describing native vs Tavily behavior per backend. (`spec/tools/web_fetch.md` is unchanged.)

## Open questions (deferred)

1. **Result caching** — refetch every time for v1. Add a per-process LRU if it bites credits or page load latency.
2. **`search_depth` default** — `basic` for v1; flip to `advanced` per-call if the agent surfaces it as a tool argument later.
3. **`include_answer` framing** — including the synthesized answer is convenient but conflates Tavily's summary with the model's reasoning. Keep for v1; revisit if it leads to over-anchoring on Tavily's framing.
4. **Replace native search on Claude/Codex** — could route those through Tavily for full vendor consolidation on search. Tradeoff: lose Anthropic/OpenAI's grounded/cited results. Defer; no clear win.
5. **Browser pool / concurrency** — single browser process serializes `browserFetch` calls per agent. If cron-driven workflows surface concurrent fetches as a bottleneck, swap to a small pool. v1 ships with one browser.
6. **Browser process lifecycle on long-running agents** — Chromium leaks memory on multi-hour runs. v1 just keeps the process up; if memory grows, recycle the browser every N fetches or on a timer.
7. **Tool naming when models surface choices to the user** — does the model pick `webFetch` vs `browserFetch` correctly on its own, or does it need explicit guidance in `spec/agent.md`? v1 ships with the tool descriptions only; revisit if the model defaults wrong (e.g. always reaches for `browserFetch` even on plain JSON).
