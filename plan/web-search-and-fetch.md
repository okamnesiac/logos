# Web search + fetch impls

Replace the bundled stubs with **Tavily Search** for `webSearch` and **Tavily Extract** for `webFetch`, on the backends where protos owns the execution path.

Lands after the [agent-sdk migration](./agent-sdk-migration.md).

## Why

- Vercel-backed profiles have no `webSearch` impl today (silent no-op) and a JS-bad `webFetch` (basic node fetch + html2md). `openai` has a JS-bad `webFetch` too.
- **Tavily** is purpose-built for AI agents: search returns clean LLM-friendly snippets plus an optional synthesized `answer`; extract renders JS server-side and returns cleaned content. Result quality is consistently better than Brave's raw SERP + custom extraction in early evaluation.
- One vendor for both gaps means one API key, one rate-limit budget, no browser binary, no extraction code to maintain.
- Privacy tradeoff: Tavily is an aggregator (your query also reaches the underlying engines via Tavily's account), and it's an AI-tooling vendor whose default incentive favors retention. Brave + self-hosted Playwright was the privacy-maximal alternative — declined for v1 because result quality ranked above the incremental privacy gain.

## Scope per backend

| Backend | `webSearch` | `webFetch` |
|---|---|---|
| `claude` | native (Anthropic hosted) — untouched | native (Anthropic hosted) — untouched |
| `codex` | native — untouched | native — untouched |
| `openai` | native (`web_search` hosted) — untouched | **override** with Tavily Extract |
| `vercel` | **add** Tavily Search (no current default) | **override** with Tavily Extract |

Wired via a single `withImpls({ webSearch: tavilySearchExecute, webFetch: tavilyExtractExecute })` call. Backends that prefer native markers (Claude/Codex everywhere; `openai` for `webSearch`) ignore the execute; Vercel and `openai` `webFetch` fall through to it.

## Tavily Search

- Endpoint: `POST https://api.tavily.com/search`
- Body: `{ query, max_results: 5, search_depth: 'basic', include_answer: true }`
- Auth: `Authorization: Bearer $TAVILY_API_KEY`
- Free tier: 1000 queries/mo — comfortably above personal volume.
- Map response: `results[].{title, url, content}` to canonical `webSearch` output. If `answer` is non-empty, prepend it as a synthesized summary line (the model gets both the answer and the sources).
- Errors: 429 (rate limit) and 401 (auth) surface as tool-level errors, no crash.

`search_depth: 'advanced'` costs more credits but returns deeper content. Default to `basic`; revisit if the agent reports thin results.

## Tavily Extract

- Endpoint: `POST https://api.tavily.com/extract`
- Body: `{ urls: [url], extract_depth: 'basic' }`
- Same `TAVILY_API_KEY` — no separate auth.
- Response: `results[0].raw_content` — already rendered (JS executed) and cleaned. Pass through directly to the canonical `webFetch` output shape.
- `failed_results[]` non-empty → tool-level error with the failure reason.
- `extract_depth: 'advanced'` costs more, returns more content. Default basic.

No browser binary. No persistent process state. Stateless HTTP call per fetch.

## Wiring

`withImpls` runs once at agent construction in `src/agent.ts`:

```ts
const tools = withImpls(canonicalTools, {
  webSearch: tavilySearchExecute,
  webFetch: tavilyExtractExecute,
});
```

Both execute functions live in `src/tools/impls/tavily.ts` (single file — they share the API key and a small `tavilyFetch` helper).

## Spec edit scope

- `spec/architecture.md`: short subsection naming Tavily as the canonical impl for protos's owned-execution backends. File-structure tree gains `src/tools/impls/`.
- `spec/build.md`: Key packages line — no new deps (Tavily is plain HTTP). Step 9 env validation adds `TAVILY_API_KEY`.
- `spec/tools/web_fetch.md` and new `spec/tools/web_search.md`: short recipes describing behavior per backend (native vs Tavily).

## Open questions (deferred)

1. **Result caching** — refetch every time for v1. Add a per-process LRU if it bites credits.
2. **`search_depth` / `extract_depth` default** — `basic` for v1; flip to `advanced` per-call if the agent surfaces it as a tool argument later.
3. **`include_answer` framing** — including the synthesized answer is convenient but conflates Tavily's summary with the model's reasoning. Keep for v1; revisit if it leads to over-anchoring on Tavily's framing.
4. **Replace native on Claude/Codex** — could route those through Tavily for full vendor consolidation. Tradeoff: lose Anthropic/OpenAI's grounded/cited results. Defer; no clear win.
5. **Privacy escape hatch** — if the Tavily-as-aggregator privacy story becomes a problem, the Brave + Playwright pairing remains a viable swap (no change to interface, just different `withImpls` execute bodies).
