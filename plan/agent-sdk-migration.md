# agent-sdk migration

Migrate the protos agent runtime from direct Vercel AI SDK calls to [`agent-sdk`](https://github.com/ExProtos/agent-sdk), with backend selection per `models.yaml` profile (Claude Agent SDK, Codex AppServer, OpenAI Agents SDK, or Vercel AI SDK Agent).

## Why

- **Subscription billing** via Pro/Max OAuth (`CLAUDE_CODE_OAUTH_TOKEN`) and ChatGPT OAuth (`~/.codex/auth.json`) — much better personal-use cost story than metered API.
- **Native tool integration** on Claude/Codex — sandboxed `Bash`, tuned `Read`/`Write`/`Edit`/`apply_patch`, native `Task`/`TodoWrite`.
- **OpenAI hosted tools** via the OpenAI Agents backend — `web_search`, `code_interpreter`, `image_generation`, `file_search`, `computer_use`, plus built-in tracing.
- **Local models** stay supported via the Vercel backend with `@ai-sdk/openai-compatible`.
- **Backend portability** falls out for free.

## What stays / what changes

**Unchanged:** channels, channel registry, scheduler, cron layering, memory module, skills loader, event schema (with the path layout change below), render filter, replay semantics, wrapper script, safe-restart, SOUL.md, cron logs, sub-agent log layout.

**Changes:** `agent.ts` (uses `Agent` from agent-sdk), `models.ts` (per-profile backend factory), `tools/` (mostly drops in favor of canonical), `agents/runner.ts` (sub-`Agent`), `router.ts` (event-stream consumer + gap detection), `threads.ts` (per-profile JSONL paths + merge for render), `paths.ts` (mostly goes away). Remove `PROTOS_SELF_EDIT` env var.

---

## 1. Backends

agent-sdk supports four: `claude`, `codex`, `openai-agents`, `vercel`. Each `models.yaml` profile names one.

`codex` and `openai-agents` both target OpenAI models but are not interchangeable:

- **`codex`** — wraps `codex app-server`. Supports the ChatGPT subscription via `codex login` (`~/.codex/auth.json`) or `OPENAI_API_KEY`.
- **`openai-agents`** — wraps `@openai/agents`. API-key only (`OPENAI_API_KEY`). Pulls in OpenAI's hosted tools (`web_search`, `code_interpreter`, `image_generation`, `file_search`, `computer_use`) and built-in tracing.

### `models.yaml` schema

Add `backend:` field, defaulting to `vercel` when omitted (backcompat).

```yaml
default:
  backend: claude
  model: claude-sonnet-4-5
  fallback: backup-via-api

fast:
  backend: claude
  model: claude-haiku-4-5

reasoning:
  backend: codex
  model: gpt-5

hosted-tools:
  backend: openai-agents
  model: gpt-5

local:
  backend: vercel
  provider: openai-compatible
  base_url: http://localhost:11434/v1
  model: llama3.1

backup-via-api:
  backend: vercel
  provider: anthropic
  api_key: $ANTHROPIC_API_KEY
  model: claude-sonnet-4-5

subagent: fast
```

| Field | `claude` | `codex` | `openai-agents` | `vercel` |
|---|---|---|---|---|
| `backend` | required | required | required | optional (default) |
| `model` | required | required | required | required |
| `provider` | n/a | n/a | n/a | required (`anthropic`/`openai`/`openai-compatible`) |
| `api_key` | n/a (env) | n/a (env or auth.json) | n/a (env) | required for hosted; optional for `openai-compatible` |
| `base_url` | n/a | n/a | n/a | required for `openai-compatible` |
| `temperature` | passes through | passes through | passes through | passes through |
| `fallback` | works | works | works | works |

Validation at load (existing rules) plus reject unknown `backend:` and reject fields that don't apply to the chosen backend.

### Auth

Per-process per backend, env-based. agent-sdk passes through whatever the backend SDK reads.

| Backend | Source |
|---|---|
| Claude (OAuth) | `CLAUDE_CODE_OAUTH_TOKEN` env (run `claude setup-token` once) |
| Claude (API) | `ANTHROPIC_API_KEY` env |
| Codex (OAuth) | `~/.codex/auth.json` (run `codex login` once) |
| Codex (API) | `OPENAI_API_KEY` env |
| OpenAI Agents | `OPENAI_API_KEY` env (no OAuth path) |
| Vercel | per-profile `api_key` in `models.yaml` |

Per-process caveat: two Claude profiles in one daemon share `CLAUDE_CODE_OAUTH_TOKEN` (same for Codex; same `OPENAI_API_KEY` shared by Codex API mode and OpenAI Agents). Per-profile API keys work only on Vercel. Fine for protos's single-user model; document.

### Fallback

Existing `fallback:` extends to cross-backend. Errors from `agent.run()` get classified by the existing rules (credit/rate/5xx/connection → fall back; auth → pass through). The fallback adapter wraps `agent.run` regardless of backend; on retry, build a new `Agent` from the fallback profile.

---

## 2. Tools

Native-first. Verified the Vercel backend has `withImpls` defaults for `bash`, `read`, `write`, `edit`, `glob`, `grep`, `webFetch` — adopting native gives full coverage on every backend without writing the file/shell plumbing.

### Catalog

| Tool | Source | Notes |
|---|---|---|
| `bash` | canonical | Native on Claude/Codex (sandboxed). Vercel/OpenAI Agents impl bundled. |
| `read` | canonical | Native on Claude/Codex. Vercel/OpenAI Agents impl bundled. |
| `write` | canonical | Native on Claude/Codex. Vercel/OpenAI Agents impl bundled. |
| `edit` | canonical | Native (Claude `Edit`, Codex `apply_patch`). Vercel/OpenAI Agents impl bundled. |
| `glob` | canonical | Native on Claude. Codex via shell. Vercel/OpenAI Agents impl bundled. |
| `grep` | canonical | Native on Claude. Codex via shell. Vercel/OpenAI Agents impl bundled. |
| `webFetch` | canonical | Native on Claude/Codex. Vercel/OpenAI Agents impl bundled. Privacy story shifts; document in recipe. |
| `webSearch` | canonical | Native on Claude/Codex/OpenAI Agents (hosted `web_search`). **Skipped on Vercel** (no `withImpls` provider) — silent no-op. |
| `todo` | canonical | Native on Claude (`TodoWrite`), Codex (`plan`); Vercel and OpenAI Agents special-case via in-process state + system-prompt re-injection. |
| `task` | canonical | Native (Claude `Task`, Codex `collabAgentToolCall`); Vercel and OpenAI Agents special-case via SDK sub-agent (`Agent.asTool` on OpenAI Agents). **Kept alongside `delegate_task`** (see below). |
| `find_memory` | custom | No native equivalent. |
| `add_memory` | custom | Memory-aware create + shadow-resolution preservation. |
| `rename_memory` | custom | Memory-aware rename + link rewriting. |
| `add_journal_entry` | custom | Renamed from `remember`. Date-stamped append to today's journal with conventions — more than `write`. |
| `delegate_task` | custom | Skill-list / tool-allowlist + sub-agent log layout. |

### Drops

- `shell` → `bash`
- `read_file` → `read`
- `write_file` → `write`
- `edit_file` → `edit`
- `web_fetch` → `webFetch`
- `remember` → renamed to `add_journal_entry`

Corresponding `spec/tools/*.md` recipes deleted (or renamed for the journal one).

### `task` vs `delegate_task`

Both kept. Different needs:

- **`task`** (canonical) — general-purpose Claude/Codex sub-agent. Tuned by the SDKs. Used when the model wants a focused worker without our skill-list/tool-allowlist conventions.
- **`delegate_task`** (custom) — protos-flavored. Caller provides explicit `skills: string[]`, `tools: string[]`, optional `model:` profile. Sub-agent log lands at `runtime/logs/sub-agents/...`. Skill `preferred_model` resolution applies here.

### Path safety

Convention only. The `paths.ts` wrapper layer was load-bearing because we owned `execute`; with native tools we have no interception point. Document in the spec, opt-in escape valves:

1. **`git revert spec/` after each turn** — cron-driven or post-dispatch. Cheap, catches stray writes.
2. **`chmod -R a-w spec/`** — OS-level. Recommended for production. Also `chmod -R a-w agent/` to disable self-edit.

System prompt reinforces convention. `PROTOS_SELF_EDIT` env var goes away — replaced by the chmod recommendation.

### `withImpls` for Vercel / OpenAI Agents

For v1, use the agent-sdk bundled defaults on both backends. Re-add custom impls only if a default proves inadequate (e.g. our existing `web_fetch` privacy/backend-choice gives way to the canonical for now; documented in the recipe; reinstate via `withImpls` later if it bites).

`webSearch` already routes natively on OpenAI Agents (hosted `web_search`); only Vercel still lacks a default provider.

---

## 3. Threads, continuations, and profile switches

### Per-profile thread layout

`runtime/threads/{channelId}/{conversationId}/{profile}.jsonl` — one JSONL per `(conversation, profile)` pair. Adjacent `{profile}.continuation` sidecar holds the active session token for that profile.

```
runtime/threads/telegram/12345/
  default.jsonl
  default.continuation
  fast.jsonl
  fast.continuation
```

Channel render reads ALL `*.jsonl` files in the directory and merges by `timestamp`. Users see one continuous conversation regardless of which profile served which turn.

Memory consolidation (nap/dream) reads the same merged view.

### Continuation strategy

One continuation **active** per dispatch, but each profile keeps its own continuation across switches. Resuming a profile uses its prior token; switching to a fresh profile starts a new session with a context recap.

### The five cases

1. **Cold start** — directory empty. `agent.run({ message })` no continuation. Save `session_start.continuation` to `{P}.continuation`. Append events to `{P}.jsonl`.
2. **Same-profile resume** — `{P}.continuation` exists. Pass token: `agent.run({ message, continuation })`. Backend resumes from native storage.
3. **Profile switch forward** (fresh into a not-yet-seen profile) — compute the recap from other profiles' tails (last N merged turns), prepend to the user message, call `agent.run({ message: <recap + actual> })`, save new continuation.
4. **Profile switch back** (`{P}.continuation` exists, but other profiles have run since) — compute the gap (events with `timestamp > T_P_last` from other profiles' JSONLs), prepend recap to user message, call `agent.run({ message, continuation })`. Resume the prior session AND inject the gap context.
5. **Continuation invalid** — `Backend.isContinuationInvalid(err)` triggers; clear sidecar, retry as case 3 (fresh session with full prior history as recap).

### Gap detection

For cases 3 and 4: read the last event timestamp from `{P}.jsonl` (`T_P_last`). Read events from every other profile's JSONL where `timestamp > T_P_last`. Merge by timestamp. That's the gap. Cap at last 10 events; verbatim text (no summarization for v1).

### Recap format

In-message prepend, never written to threads JSONL. Plain readable format using existing conventions; no new bracket-wrapper invented:

```
While you were inactive, conversation continued on profile `fast`. Recent turns:

  user (14:32 PDT): when does the train leave?
  fast (14:32 PDT): 4:15 PDT.
  user (14:48 PDT): thanks, how long is the ride?
  fast (14:48 PDT): about 2 hours.

[sent 2026-04-25 15:01:12 PDT] is there a dining car?
```

The actual user message keeps its existing `[sent ...]` prefix as the boundary cue. Indented gap turns use `role (time): text` — no fake `[sent ...]` headers competing with the convention. Profile names called out (`assistant (via fast)` style) explain tone shifts cheaply.

For case 3 (no prior session), reword the intro: "Recent conversation activity (other profiles):" — no "while you were inactive" framing since there's no prior session.

### Recap is runtime-only

The recap header gets prepended to the `message` field passed to `agent.run`. It does NOT get written to our threads JSONL. The user event in `{P}.jsonl` records only what the user actually typed. Otherwise:

- The recap would appear on render/replay as a fake user soliloquy.
- Future gap calculations would treat it as real conversation, compounding.

The augmentation lives in one place: between reading from threads/ and calling `agent.run`.

### Where each backend stores history

Our threads JSONL is durable / canonical. The runtime's session record is opaque to us:

| Profile | Backend reads on resume | Backend writes during turn |
|---|---|---|
| `claude` | `~/.claude/projects/.../{session-id}.jsonl` (Claude SDK's storage) | Same |
| `codex` | `~/.codex/sessions/...` + sqlite | Same |
| `openai-agents` | `runtime/sessions/openai-agents/{continuation}.jsonl` (`AgentInputItem` lines) | Appends `AgentInputItem` rows |
| `vercel` | `runtime/sessions/vercel/{continuation}.jsonl` (UIMessage) | Appends UIMessage rows |

Across all backends, we tee from `query.events` to our `{profile}.jsonl`. The SDK's session storage is never read or written by protos directly. For `openai-agents`, we configure `sessionsDir: runtime/sessions/openai-agents/` (not the default no-persistence mode, and not `useConversations` — the latter is mutually exclusive with `autoCompact`).

### Image events in the gap

If the gap includes user events with `attachments`, the recap text mentions them as `[user sent a photo]` rather than embedding the bytes. Cheaper. Image bytes remain in our threads/ for human replay, and the next round of conversation can re-share if it matters.

For the *current* turn (not the gap), image attachments pass through `QueryInput.attachments` — supported on all four backends. We pass `{type: 'image', source: {kind: 'path', path: 'runtime/blobs/...'}}` since the bytes already live on disk; agent-sdk handles the per-backend wire format.

---

## 4. Drop history caps and token guards

`MAX_HISTORY = 100` and `applyTokenGuard(...)` go away from protos. Made sense when we owned the message array; no longer applicable.

All four backends auto-compact natively:

- **Claude / Codex** — built into the upstream SDKs.
- **Vercel** — agent-sdk runs hysteresis-based compaction (`autoCompact` defaults to `true`; tunable via `contextThreshold`, `compactionModel`, `keepLastTurns`, `contextWindow`).
- **OpenAI Agents** — agent-sdk wraps the Session in `OpenAIResponsesCompactionSession` (also `autoCompact: true` by default).

protos sets explicit `contextWindow` per Vercel profile when the model isn't in agent-sdk's `MODEL_CONTEXT_WINDOWS` table — otherwise compaction silently disables for that profile. The other knobs can stay at defaults until tuning is needed.

PR #51's image-token accounting (~1500 tokens per image) lives in the Vercel backend's compaction math — protos no longer needs the local `applyTokenGuard` for images either.

---

## 5. Cold-start system prompt

Memory manifest + 24h journal already live in the system prompt and bridge across sessions cheaply. No changes there.

The recap mechanism handles "stuff that happened on other profiles in this conversation" via in-message prepend rather than `systemPromptAppend` — see the threads section above.

---

## 6. Upstream dependencies

Both former blockers shipped:

- **Vercel auto-compaction** — landed (commit `a56b383`). Hysteresis-based; on by default with tunable `contextThreshold` / `compactionModel` / `keepLastTurns` / `contextWindow`. Parity with Claude/Codex/OpenAI Agents.
- **Image support in `QueryInput`** — landed (commit `39406df`). `QueryInput.attachments?: Attachment[]` accepted on all four backends; image sources can be `url`, `base64`, or local `path`.

No upstream blockers remain for the migration. All four backends support attachments and auto-compact; both early caveats (Vercel-only images, manual Vercel thread pruning) are gone.

---

## Open questions (deferred)

1. **Gap recap summarization** — verbatim last-10 for v1. Switch to LLM-summarized if conversations get gappy enough that the verbatim version eats too much context.
2. **`webSearch` on Vercel** — skipped for now (native on Claude/Codex/OpenAI Agents). Add a `withImpls` provider (Brave, Tavily, etc.) when someone wants it.
3. **`webFetch` privacy** — bundled defaults for v1 on Vercel and OpenAI Agents. Reinstate our `PROTOS_WEB_FETCH_BACKEND` privacy controls via `withImpls` if it bites.
4. **Per-profile auth on Claude/Codex/OpenAI Agents** — out of scope; single-user assumption holds. Note that Codex API mode and OpenAI Agents share `OPENAI_API_KEY`.
5. **OpenAI Agents tracing** — auto-traces to OpenAI's tracing dashboard when this backend runs. No protos action; document where traces appear.
6. **Existing pre-migration thread history** — on first dispatch after migration, treat as case 3 (fresh session with full recap from existing threads JSONL). No conversion step needed; the gap mechanism handles it.

---

## Spec edit scope (for the eventual PR)

- `spec/architecture.md`: tech stack row, Agent section opener, Tools list rewrite, Sub-agents section (task + delegate_task complement), Storage section (per-profile thread paths + `runtime/sessions/{vercel,openai-agents}/`), new Backends subsection in Model selection (four backends), new Session continuity subsection, Self-modification simplification, file-structure tree.
- `spec/build.md`: Key packages, Configuration (Claude/Codex auth setup, OpenAI Agents env), step 2 (threads.ts per-profile + drop `buildLlmMessages`), step 4 (Agent runtime + drop `MAX_HISTORY`/`applyTokenGuard` + cold-start logic + Vercel `contextWindow` per profile), step 4d (sub-Agent), step 9 (drop `PROTOS_SELF_EDIT` validation; add per-backend auth-presence validation).
- `spec/tools/`: rename `remember.md` → `add_journal_entry.md`. Delete `shell.md`, `read_file.md`, `write_file.md`, `edit_file.md`, `web_fetch.md`. Keep `find_memory.md`, `add_memory.md`, `rename_memory.md`, `delegate_task.md`, plus the consolidation/orphan-finder recipes.
- Channel recipes: unchanged.
