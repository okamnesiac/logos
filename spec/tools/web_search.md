# web_search

Canonical web-search tool. Native on three of four backends; Tavily Search wired in via `withImpls` on the Vercel backend (no native default).

## Description (shown to the model)

Search the web. Returns a list of results (title, URL, snippet) plus an optional synthesized answer summary on the backends that support one.

This is the canonical `webSearch` tool — agent-sdk handles the per-backend dispatch. Use it for any open-ended retrieval ("what's the current state of X?", "find documentation for Y"). For deep-content retrieval on a known URL, follow up with `webFetch` (HTTP only) or `browserFetch` (real browser engine).

## Input

```ts
{ query: string }
```

## Output

```ts
Array<{ title: string, url: string, content: string }>
```

The shape is whatever agent-sdk's canonical `webSearch` returns; the Tavily override (Vercel backend) maps Tavily's response to that same shape so the model sees the same surface across backends. When Tavily returns a synthesized `answer`, it's prepended as a synthetic first result with `title: "Tavily Answer"` and `url: ""` so the model gets both the summary and the underlying sources.

## Per-backend behavior

| Backend | Implementation |
|---|---|
| `claude` | Native (Anthropic-hosted `web_search`). |
| `codex` | Native (Codex AppServer's hosted search). |
| `openai` | Native (OpenAI Agents SDK's hosted `web_search`). |
| `vercel` | Tavily Search, wired via `withImpls`. See [Tavily wiring](#tavily-wiring) below. |

The native impls have their own provider stories (Anthropic curates, OpenAI does too, Codex piggybacks on its host); we don't try to harmonize them. The Tavily wiring on Vercel exists because Vercel has no native default — without `withImpls`, `webSearch` is a silent no-op on that backend.

## Tavily wiring

Lives in `agent/src/tools/impls/tavily.ts`, exported as `tavilySearchExecute` and registered via `withImpls(canonicalTools, { webSearch: tavilySearchExecute })` at `Agent` construction time on Vercel profiles.

- **Endpoint:** `POST https://api.tavily.com/search`
- **Auth:** `Authorization: Bearer $TAVILY_API_KEY`. The env var is referenced from `config/.env`. If unset and a Vercel profile is in use, the Backend construction surfaces the missing key as a clear startup error rather than failing per-call.
- **Body:** `{ query, max_results: 5, search_depth: 'basic', include_answer: true }`.
- **Response mapping:**
  - `results[].{title, url, content}` → canonical search-result shape.
  - If `answer` is non-empty, prepend it as the synthetic first result described under Output.
- **Failure modes — wrap-all rule:** every error thrown from `tavilySearchExecute` prefixes its message with `tavily:` and (where one is in scope) includes the query. This includes auth failures, rate limits, network/DNS errors, malformed responses (e.g. a maintenance-page HTML body returned with status 200, which crashes `res.json()`), and any unexpected condition. The model gets a consistent surface. Common cases:
  - 401 → `Error("tavily: HTTP 401 — check TAVILY_API_KEY in config/.env")`.
  - 429 → `Error("tavily: HTTP 429 — rate limit exceeded")`.
  - Network/DNS errors → `Error("tavily: network error for query {query}: {reason}")`.
  - Non-2xx status (other) → `Error("tavily: HTTP {status} for query {query}")`.
  - JSON parse / malformed response → `Error("tavily: malformed response for query {query}: {reason}")`.
- **Free tier:** 1000 queries/month, comfortably above personal volume. `search_depth: 'advanced'` costs more credits but returns deeper content; default to `basic`, revisit if results read thin.

## Dependencies

- Tavily wiring uses plain HTTP — no new npm dep.
- Native backends pull search through their respective SDKs; no protos-side dependency to track.

## Implementation notes

- **Why a single recipe for a per-backend split.** The model sees one `webSearch` tool regardless of backend. Documenting each backend's path here keeps the recipe co-located with the user-visible behavior. The override-vs-native distinction is an implementation detail of agent-sdk's dispatch layer; the spec describes both shapes so the build agent knows what code to write where.
- **No `withImpls` on `claude`/`codex`/`openai`.** agent-sdk's `withImpls` is consulted only when a backend doesn't have a native implementation. Wiring `tavilySearchExecute` for those backends would be a no-op — they ignore the execute and use their hosted search.
- **Caching.** Refetch every time for v1. Add a per-process LRU only if Tavily credit usage warrants it; see `plan/web-search-and-fetch.md` open question 1.

## Open question

Whether to also route `claude`/`codex` through Tavily for full vendor consolidation. Tradeoff: lose Anthropic/OpenAI's grounded/cited results. Deferred — see `plan/web-search-and-fetch.md` open question 4.
