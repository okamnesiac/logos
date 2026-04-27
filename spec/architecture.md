# Architecture

## Overview

Protos is a single-process personal AI assistant. Messages come in from messaging channels, get processed by an AI agent, and responses go back out through the same channels.

The workspace is organized into **six sibling domains**, each with a distinct role and lifecycle:

```
protos/                # workspace root
  spec/               # the blueprint — architecture, build, recipes, defaults
  agent/              # generated implementation — code that the build produces
  config/             # behavior — identity, instance overrides, .env
  memory/             # durable state — facts, preferences, journal
  vendor/             # external tooling — third-party clones the agent uses
  runtime/            # ephemeral state — message threads, logs, pid files
```

Only `spec/` (and the workspace entry-point docs) is tracked by this repo. `agent/`, `config/`, and `memory/` are sibling directories — gitignored here and **strongly recommended** to be their own Git repos (see [Git repos per domain](#git-repos-per-domain) below). `vendor/` is also gitignored here; each of its subdirectories is its own external repo with its own upstream. `runtime/` is local-only.

## The six domains

### `spec/` — the blueprint

The architecture documents (`architecture.md`, `build.md`), channel recipes, bundled skills, and default cron jobs. Tracked by this repo. Shared across all Protos users — what's in `spec/` is what every instance gets out of the box.

The running agent reads from `spec/` directly for skills and cron defaults. Spec updates take effect on the next agent restart — no copy step, no drift.

`spec/` is **read-only at runtime by convention**. The system prompt reinforces it; production deployments harden it via `chmod -R a-w spec/` (see [Self-modification](#5-self-modification) for path discipline). Spec changes happen out-of-band — via PR or via a coding agent invoked outside the running daemon. Instance-specific extensions to skills and cron go in `config/skills/` / `config/cron/`, which the loader merges with `spec/` at startup — config frontmatter wins, config body appends to spec body. See the **Layering** notes under Skills and Scheduler for details.

### `agent/` — generated implementation

The TypeScript code produced by the build: `index.ts`, `router.ts`, the channel implementation(s) the user chose, the built-in tool implementations, the wrapper script. Gitignored by this repo. Optionally a separate Git repo if you want to commit your specific implementation.

The agent edits its own source under `agent/src/` via the `self-edit` skill.

### `config/` — behavior

How this specific instance behaves: identity (`SOUL.md`), instance-specific overrides for tools/skills/channels/cron, environment variables. Per-user. Optionally a separate Git repo.

Behavior reads context, but context does not define behavior.

### `memory/` — durable state

What the agent knows and is committed to: facts, preferences, summaries, journal entries, durable task state. Per-user, private. Optionally a separate Git repo.

Granular files, not one consolidated file — Git history is more useful when changes are line-item rather than whole-file.

### `vendor/` — external tooling

Third-party tooling installed at known paths under `vendor/`. Two flavors today:

- **Cloned source repos** (`vendor/browser-harness/`, …) — each its own external Git repo with its own lifecycle. Set up on demand by the relevant skill.
- **Downloaded binaries** (`vendor/chromium/`) — tooling fetched by an external installer (e.g. Playwright's `npx playwright install chromium` writes here when `PLAYWRIGHT_BROWSERS_PATH=vendor/chromium`). Pinned by the Node-side package version (`agent/package.json`); the binary directory is the manifestation of that pin on disk.

Per-instance and gitignored. Skills, tools, and the build script name the canonical install path (`vendor/{name}/`) so the agent and its tooling have one place to look. The agent may edit files inside cloned-repo vendors per the relevant skill (typically via a coding agent), but those edits are local to the vendored repo, not commits in `agent/` or `spec/`. Binary vendors are read-only — the installer manages them.

Empty by default — vendored tooling is opt-in. `vendor/chromium/` ships during `build` because `browserFetch` is bundled; cloned-repo vendors install when their skill walks the owner through setup.

### `runtime/` — ephemeral state

What is happening right now: message threads, logs, pid files, locks, caches. Machine-local, never versioned, safe to delete and rebuild.

Practical rule: if it works well with Git, it belongs in `memory/`. If it doesn't merge well, it belongs in `runtime/`.

## Tech stack

| Component | Choice |
|-----------|--------|
| Runtime | Node.js 18+ |
| Language | TypeScript |
| Storage | Plain files — no database |
| AI | [`@exprotos/agent-sdk`](https://github.com/ExProtos/agent-sdk) — one API over four runtimes (Claude Agent SDK, Codex AppServer, OpenAI Agents SDK, Vercel AI SDK) |
| Hosting | Runs directly on the host — no containers |

## Components

### 1. Channels

Channels are messaging platform integrations. Each channel:

- Connects to a messaging platform (WhatsApp, Telegram, Discord, Slack, etc.)
- Receives inbound messages and normalizes them
- Sends outbound messages from the agent back to the platform
- Shows a typing indicator while the agent is thinking

**Owner-only:** Each channel knows the owner's ID on that platform. Messages from anyone else are silently ignored.

**Config-driven registration.** Channels are defined in `config/channels.yaml`. Each top-level entry is a channel name; its `provider:` field names the implementation module under `agent/src/channels/` and defaults to the channel name if omitted. Modules in `agent/src/channels/` are **providers**, not auto-activating channels — a module is only instantiated when a `channels.yaml` entry references it. Two entries with the same `provider:` but different names are allowed (e.g., `work-telegram:` and `personal-telegram:` both using `provider: telegram`); each gets its own `channelId` and thread directory.

```yaml
# config/channels.yaml
primary: telegram           # string value = pointer to another entry
telegram:
  provider: telegram        # optional — defaults to the channel name
  bot_token: $TELEGRAM_BOT_TOKEN
  owner_id: 456789
  model: reasoning          # optional — names a profile in models.yaml
terminal:
  enabled: true             # no credentials needed
```

Like `models.yaml`, any top-level entry is either a full config (an object) or a pointer (a string). `primary:` is a conventional entry naming the channel used for the owner's main conversation — the scheduler sends cron replies here.

**Per-entry fields.** `provider:` selects the module; `model:` names a model profile (falls through to `default` if omitted); credentials and platform-specific fields (e.g. `bot_token`, `owner_id`) are consumed by the provider module. `enabled: false` disables a channel without deleting its block.

**Recipes vs implementations:** Channel **recipes** (`.md` setup guides) live in `spec/channels/` — they describe how to build each provider module. Channel **implementations** (`.ts` code) live in `agent/src/channels/`. Both built-in channels (generated from spec recipes by the build) and custom channels (added by the user) live in the same directory — the `agent/` repo is the user's own implementation, so there's no need for a parallel code location in `config/`.

#### Channel `send()` contract

Every channel exports a `send(text)` function via its `register()` return. The router calls `send` exactly once per agent invocation, with one of:

- **Normal text** — a reply to display. Send it on the platform however the platform expects.
- **The literal string `"NO_REPLY"`** — a **lifecycle marker**, NOT a message. The agent decided to stay silent. The channel must NOT show anything to the user, but it MUST clean up any turn-scoped state — typing indicators, refresh loops, "thinking" animations. Without this, channels that show "the bot is thinking" while the agent works hang on those indicators when the agent stays silent.

The router stores `NO_REPLY` literally in the JSONL and passes it through to `send()` unchanged — no translation layer. The same string is used by the agent (its output convention), in storage (the JSONL record), and in the API view sent back to the model on subsequent invocations. This means past silent turns reinforce the convention by example, rather than introducing a separate placeholder the model could imitate.

The literal `NO_REPLY` token is matched exactly (`text.trim() === "NO_REPLY"`) — substring matches don't count. If the model wants to discuss the convention (e.g., "I respond with NO_REPLY when…"), the surrounding content makes it a normal reply.

This contract is general — any new channel MUST handle the `NO_REPLY` marker correctly. Don't repeat this rule in per-channel recipes; recipes mention only platform-specific cleanup details (e.g., "clear the typing-refresh interval" for Telegram).

#### Dispatch-error surfacing

If a `router.dispatch` call throws, the channel's error handler replies to the owner with the actual error message (e.g. `dispatch error: Your credit balance is too low…`). No generic placeholder — the audience is one person and the real error is what tells them what to fix.

#### Cursor-based replay

Because thread history is stored as append-only JSONL files, any channel with disconnecting clients can support replay with no server-side state. The pattern:

1. Client stores its last-seen line index locally.
2. On reconnect, client sends `{"from": N}`.
3. Server reads lines `N..end` from `runtime/threads/{channelId}/{conversationId}.jsonl`, **applies a render filter** (see below), and streams the resulting messages as replay.
4. Server watches the JSONL file (`fs.watch`) for new appends, applies the same filter, and streams those live.
5. Client updates its cursor as messages arrive.

**Render filter.** The JSONL contains the full event stream (user messages, assistant steps with tool calls, tool results — see [Storage → Event schema](#event-schema)). The user-facing conversation is a collapsed view of that stream: for each `turn_id`, only the last non-empty, non-`NO_REPLY` assistant text is shown, and intermediate tool-call / tool-result events are hidden. (`NO_REPLY` is a lifecycle marker per the [Channel `send()` contract](#channel-send-contract), not a message — render the same way: skip it.) Watch-based channels (e.g. terminal) must apply this filter server-side before emitting to clients; push-based channels (e.g. telegram) receive the final reply from the router's `send()` call and don't need to filter.

User events carry their `attachments` field through the filter unchanged; each channel decides how to display each attachment per its capability (render thumbnails inline, show `[image]` placeholders, skip non-text-capable parts). Channels with no media-display path should still render the surrounding text.

This gives any client-server channel "pick up where I left off" semantics across reconnects — messages queued while no client was connected (e.g., cron firing at night) are delivered on next connect. See the terminal channel for the reference implementation.

### 2. Router

The router sits between channels and the agent. It:

- Receives normalized messages from channels
- Maintains a per-conversation message queue (one agent invocation at a time per conversation)
- Passes messages to the agent with conversation context
- Routes agent responses back to the originating channel

If the agent's response is `NO_REPLY` (exact match after trim), the router stores it **as-is** — the JSONL gets `{role: "assistant", text: "NO_REPLY"}` and `channel.send("NO_REPLY")` is called. Channels recognize the marker per the [Channel `send()` contract](#channel-send-contract): no display, but clean up turn-scoped UI (typing indicators, thinking dots).

This applies to all invocations — direct user messages and cron-originated heartbeats both. The agent retains latitude to stay silent; the channels still get clean turn lifecycles. Storage, API view, and the agent's output convention all use the same string — no translation layer.

### 3. Agent

The agent is the brain. It uses [agent-sdk](https://github.com/ExProtos/agent-sdk) to receive a message + history, decide how to respond, optionally use tools, and return a response. agent-sdk wraps four runtimes (Claude Agent SDK, Codex AppServer, OpenAI Agents SDK, Vercel AI SDK) behind one API; each model profile picks one as its **backend** (see [Model selection](#model-selection) below).

**Tools** are typed capabilities. agent-sdk ships a canonical catalog with backend-specific dispatch — native implementations on Claude/Codex (sandboxed `Bash`, tuned `Read`/`Write`/`Edit`/`apply_patch`, native `Task`/`TodoWrite`), hosted tools on the `openai` backend (`web_search`, `code_interpreter`, etc., via the OpenAI Agents SDK), and bundled in-process implementations on Vercel and `openai` fallbacks. Custom protos-specific tools live in `agent/src/tools/` and are passed alongside the canonical set.

**Canonical tools** (provided by agent-sdk):

- `bash(command)` — run a shell command
- `read(path)` — read any file in the workspace
- `write(path, content)` — create or overwrite a file
- `edit(path, old_string, new_string)` — surgical find-and-replace
- `glob(pattern)` — file pattern matching
- `grep(pattern, path?)` — content search
- `webFetch(url)` — fetch a URL and return content as markdown. Native on Claude/Codex/`openai`; bundled node-fetch on Vercel. HTTP only — no JS execution. Pair with `browserFetch` (custom tool below) for JS-heavy pages.
- `webSearch(query)` — web search. Native on Claude/Codex/`openai` (hosted `web_search`); Tavily Search on Vercel via `withImpls` (see `spec/tools/web_search.md`).
- `todo(todos)` — task list (native on Claude/Codex; in-memory state with system-prompt re-injection on Vercel/`openai`)
- `task({prompt, subagent_type})` — spawn a focused SDK-level sub-agent (see [Sub-agents](#sub-agents))

**Custom tools** (protos-specific, in `agent/src/tools/`):

- `find_memory(name)` — resolve a wiki-link-style name to every matching file, sorted shortest-path first (see [Tool return shapes](#tool-return-shapes))
- `add_memory(name, content)` — create a new memory file and preserve any `[[wiki-link]]` resolutions the new file would otherwise shadow
- `rename_memory(oldName, newName)` — move/rename a memory file and rewrite `[[wiki-links]]` so every reference resolves to the same file it did before
- `add_journal_entry(text)` — append a timestamped line to today's journal
- `delegate_task({ prompt, skills, tools, model? })` — protos's skill-aware sub-agent (see [Sub-agents](#sub-agents))
- `browserFetch(url)` — fetch a URL through headless Chromium (Playwright) with Mozilla Readability cleanup. Use for JS-heavy pages, paywalled article previews, and sites that bot-block plain `webFetch`. Available on every backend. See `spec/tools/browser_fetch.md`.
- Plus consolidation/orphan-finder helpers used by cron jobs

Memory-aware tools (`find_memory`, `add_memory`, `rename_memory`) take names in **wiki-link form** — no `memory/` prefix, no `.md` extension. They return full workspace paths (e.g. `memory/preferences/coffee.md`) in their output for interop with `read`.

**Three-tier web access ladder.** The agent has three increasingly heavy ways to reach a URL — pick the lightest one that works:

| Tool | Mechanism | When |
|---|---|---|
| `webFetch` | HTTP only | Static pages, docs, JSON, anything where a plain fetch would work |
| `browserFetch` | Single-shot render in **headless Chromium**, anonymous (no shared session with the user's Chrome) | JS-heavy pages, bot-blocked URLs, paywalled article previews where a logged-out browser still gets useful content |
| `browser-use` (skill) | Attaches to the user's **real Chrome** via CDP (`127.0.0.1:9222`); inherits the logged-in session | Multi-page flows, form fill, login, anything requiring the user's session or stateful navigation |

`browserFetch` and `browser-use` are deliberately separate — one is anonymous-and-cheap, the other is signed-in-and-expensive. They don't share session state.

Users can add their own tools directly in `agent/src/tools/` alongside the bundled custom set. agent-sdk handles the tool execution loop natively — the per-backend turn cap (default 25) prevents runaway tool use.

#### Tool return shapes

Tool return values get JSON-serialized and shown to the model. Two conventions:

- **Tools that succeed-or-fail** (`bash`, `read`, `write`, `edit`, `add_journal_entry`, etc.) return their result on success. On failure they may throw; the SDK's tool-dispatch layer catches the error and records it as the tool result so the model sees the error message.
- **Tools that succeed-or-miss** (anything that does lookup) return a **discriminated shape** — a list (empty = miss) or a union with a boolean tag — never a bare `null`. Makes the shape unambiguous to the model and leaves room for diagnostic fields later. See `spec/tools/find_memory.md` for the canonical example.

**Invariant: every tool call in the thread JSONL is paired with a tool result event.** agent-sdk guarantees this — its event stream emits `tool_result` for every `tool_call_end`, including errors. We tee both into our threads JSONL and the pairing falls out automatically. The orphan-tool-call problem (which used to break replay against models that reject orphans) is now handled inside agent-sdk; we don't reconstruct the message array ourselves.

**Skills** are markdown instruction files that teach the agent how to accomplish complex tasks using its tools. Bundled skills live in `spec/skills/{name}.md` — flat, one file per skill. Instance-specific skills live in `config/skills/`, which accepts both the same flat `{name}.md` form and the agentskills.io directory form (`{name}/SKILL.md` plus optional `scripts/`, `references/`, `assets/` sibling directories) so off-the-shelf skills drop in unmodified.

**Layering.** When a skill with the same name exists in both roots, the merged skill uses:

- **Frontmatter:** config wins on field collisions; missing config fields fall through to spec.
- **Body:** spec first, then config appended. Lets the user (or the agent itself) extend a default skill without restating it. Same rule as cron — agents tend to extend skills more often than replace them outright, so append-by-default is the right shape.

The skills summary in the system prompt shows both source paths for a merged skill (e.g. `spec/skills/X.md + config/skills/X.md`) so the agent knows where each part came from when re-reading. Within `config/`, if both flat (`{name}.md`) and directory (`{name}/SKILL.md`) forms exist for the same name, the flat form wins — these are not merged into each other, only the winning config form merges with the spec.

**Identity** comes from `config/SOUL.md`, written on first run. The agent reads it on every invocation.

**Long-term memory** comes from the `memory/` directory. The system prompt includes a manifest of **root-level files only** (name + summary); subfolder files are reached by following `[[wiki-links]]` from a root file. Full content is fetched on demand via `find_memory` and `read`. See **Memory format** below.

**Self-awareness.** The agent learns about its scheduled-job invocation mode via the bundled `scheduling` skill (`spec/skills/scheduling.md`), whose name and description appear in the skills summary at startup. Without this, the agent doesn't know cron exists and may tell the user it can't reach out proactively — which is wrong; cron firings are exactly the proactive-outreach mechanism.

#### Model selection

The agent supports multiple LLMs through named **profiles** in `config/models.yaml`. Each profile names a backend, a model ID, and any backend-specific config; callers (channels, cron jobs, `delegate_task`) reference profiles by name.

```yaml
# config/models.yaml
default:
  backend: claude
  model: claude-sonnet-4-6
  fallback: backup-via-api

fast:
  backend: claude
  model: claude-haiku-4-5

reasoning:
  backend: codex
  model: gpt-5

hosted-tools:
  backend: openai
  model: gpt-5
  api_key: $OPENAI_API_KEY

local:
  backend: vercel
  provider: openai-compatible
  base_url: http://localhost:11434/v1
  model: llama3.1

backup-via-api:
  backend: vercel
  provider: anthropic
  api_key: $ANTHROPIC_API_KEY
  model: claude-sonnet-4-6

# Multi-account example: same backend, different credentials
work:
  backend: claude
  model: claude-sonnet-4-6
  oauth_token: $CLAUDE_WORK_TOKEN

work-codex:
  backend: codex
  model: gpt-5
  codex_home: runtime/codex-auth/work

subagent: fast              # string value = pointer to another profile
```

**Pointers and inline definitions.** Every top-level entry is either a full profile (an object) or a pointer (a string naming another entry). `default` and `subagent` are conventional names — `default` is the fallback choice when nothing more specific is requested; `subagent` is the default for `delegate_task`. Either may be defined inline or as a pointer.

**Backends.** Each profile names one of four backends. agent-sdk handles dispatch; protos cares about which one is named because that determines auth and tool availability.

- **`claude`** — Claude Agent SDK. Pro/Max subscription via `oauth_token:` (or ambient `CLAUDE_CODE_OAUTH_TOKEN` from `claude setup-token`); API key via `api_key:` (or ambient `ANTHROPIC_API_KEY`). The two are mutually exclusive per profile. Native sandboxed `Bash`, tuned `Read`/`Write`/`Edit`, native `Task`/`TodoWrite`/`webSearch`/`webFetch`.
- **`codex`** — Codex AppServer (`codex app-server`). Auth lives in `auth.json` under a `CODEX_HOME` directory; pass `codex_home:` to point at a specific dir, or omit and use ambient `~/.codex/` (populated by `codex login`). The caller is responsible for populating `auth.json` (run `CODEX_HOME=<dir> codex login` for OAuth or `codex login --with-api-key` for API key). Native `apply_patch` editing, native `plan` (todo) and `collabAgentToolCall` (task).
- **`openai`** — OpenAI Agents SDK (`@openai/agents`). API-key only — `api_key:` typed field, falls back to ambient `OPENAI_API_KEY`. Optional `base_url:`, `organization:`, `project:` for non-default OpenAI endpoints. Adds OpenAI's hosted tools (`web_search`, `code_interpreter`, `image_generation`, `file_search`, `computer_use`) and built-in tracing. **Don't confuse `backend: openai` with `provider: openai`** — the former is a sibling of `claude`/`codex`/`vercel` and uses `@openai/agents`; the latter is a Vercel sub-option that uses `@ai-sdk/openai`. They reach the same models via different paths.
- **`vercel`** — Vercel AI SDK Agent. Provider-portable: `anthropic`, `openai`, or `openai-compatible` (Ollama, LM Studio, llama.cpp server, vLLM). Per-profile `api_key:`. Bundled in-process tool implementations; `webSearch` is a silent no-op until a `withImpls` provider is wired.

| Field | `claude` | `codex` | `openai` | `vercel` |
|---|---|---|---|---|
| `backend` | required | required | required | optional (default if `provider:` is set) |
| `model` | required | required | required | required |
| `provider` | n/a | n/a | n/a | required (`anthropic`/`openai`/`openai-compatible`) |
| `oauth_token` | optional (or ambient env) | n/a | n/a | n/a |
| `api_key` | optional (or ambient env) | n/a (in `auth.json`) | optional (or ambient env) | required for hosted; optional for `openai-compatible` |
| `codex_home` | n/a | optional path (default ambient `~/.codex/`) | n/a | n/a |
| `base_url` | n/a | n/a | optional | required for `openai-compatible` |
| `organization` | n/a | n/a | optional | n/a |
| `project` | n/a | n/a | optional | n/a |
| `permission_mode` | optional — `default` / `acceptEdits` / `plan` / `bypassPermissions` (Claude SDK enum) | n/a | n/a | n/a |
| `ask_for_approval` | n/a | optional — `never` / `untrusted` / `on-request` / `granular` (Codex enum) | n/a | n/a |
| `sandbox_mode` | n/a | optional — `read-only` / `workspace-write` / `danger-full-access` (Codex enum) | n/a | n/a |
| `temperature` | passes through | passes through | passes through | passes through |
| `context_window` | n/a | n/a | n/a | optional override for agent-sdk's auto-compaction threshold (only needed when the model isn't in agent-sdk's `MODEL_CONTEXT_WINDOWS` table; otherwise compaction silently disables for that profile) |
| `fallback` | works | works | works | works |

The permission/sandbox fields use each backend's native enum verbatim — no harmonization across backends. `claude` uses Claude Agent SDK's `permissionMode`; `codex` uses Codex's `AskForApproval` and `SandboxMode`. `openai` and `vercel` have no permission layer in their tool dispatch — bundled in-process tools just execute.

**Unattended defaults.** Protos runs as a daemon with no human in the loop to answer permission prompts, so `agent.ts` constructs each Backend with these defaults when the YAML doesn't override:

| Backend | Default for unattended use |
|---|---|
| `claude` | `permission_mode: bypassPermissions` |
| `codex` | `ask_for_approval: never`, `sandbox_mode: workspace-write` |
| `openai` / `vercel` | n/a — no permission layer; bundled tools run unrestricted |

A user who wants stricter behavior on a specific profile sets a non-default value in `models.yaml`. The trade-off is that any policy that prompts (Claude `default`/`acceptEdits`, Codex `on-request`/`untrusted`/`granular`) will hang the agent waiting for a response — agent-sdk currently auto-declines codex approval requests, and Claude's prompt has no programmatic accept path either. The unattended defaults are the only fully-supported mode today.

For `claude`, setting both `oauth_token:` and `api_key:` on the same profile errors at load (mutually exclusive — agent-sdk's `ClaudeBackend` constructor throws).

`codex` and `openai` both target OpenAI models but are not interchangeable: `codex` is the subscription path (auth in `~/.codex/auth.json` via `codex login`), `openai` is the API-key path with hosted tools and tracing.

Tool-use support varies across local models; if an `openai-compatible` profile struggles with tool calling, route the specific calls that need it (via channel/cron `model:` or `delegate_task` `model:`) to a hosted profile and keep local for simpler text tasks.

**Env-var substitution.** Any string field accepts `$NAME` as a whole-string value; the loader replaces it with `process.env.NAME`. Missing env vars error at load, naming the file path and variable. This lets users who want to commit `models.yaml` keep credentials in `.env`; users who don't mind can inline values.

**Multi-account.** Each profile's auth is independent — agent-sdk constructs a fresh Backend instance per profile and passes the typed auth fields directly. Two `claude` profiles with different `oauth_token:` values are isolated; same for two `openai` profiles with different `api_key:` values. For `codex` multi-account, give each profile its own `codex_home:` directory and bootstrap each one once with `CODEX_HOME=<dir> codex login`.

**Fallback.** A profile's `fallback:` names another profile. When a call against a profile errors with a credit-exhausted, rate-limit, provider 5xx, or connection error (ECONNREFUSED, timeout), the runtime retries the same call against the fallback — even if the fallback is a different backend. Connection errors are included so a "local-first, cloud-as-backup" setup — a `vercel` openai-compatible default with a `claude` profile as its `fallback:` — works when the local server is down. **Auth errors do NOT fall back** — they're config bugs and silent fallback hides them. One hop only; cycles detected at load.

**Selection precedence.**

- **User messages through a channel:** channel's `model:` → `default`.
- **Cron firings:** cron frontmatter `model:` (merged per layering rules — config overrides spec) → `default`.
- **`delegate_task`:** explicit `model:` arg → first skill in `skills:` with a resolvable `preferred_model:` → `subagent`.

**Hints vs. directives.** Channel `model:`, cron `model:`, explicit `delegate_task` `model:`, and profile `fallback:` are **directives** — an unknown profile errors (at load for static references, at call time for dynamic). Skill `preferred_model:` is a **hint** — if the named profile doesn't exist, the preference is silently skipped and resolution continues (next skill's preference, else `subagent`). This lets skills ship with reasonable preferences without forcing every user to define a matching profile.

**Validation at load.** The loader validates every profile resolves (no unknown pointers, no cycles); every resolved profile has a valid backend + model after env substitution, with backend-required fields present (e.g. `provider:` and `base_url:` for `vercel` openai-compatible) and unknown-for-this-backend fields rejected; mutually exclusive fields don't both appear (Claude `oauth_token:` + `api_key:`); for each profile that doesn't set its backend's typed auth field, the corresponding ambient env var is present so the SDK has a fallback (`CLAUDE_CODE_OAUTH_TOKEN` or `ANTHROPIC_API_KEY` for `claude`; `~/.codex/auth.json` exists for `codex` with no `codex_home:`; `OPENAI_API_KEY` for `openai`); and every directive reference (channel `model:` in `channels.yaml`, cron `model:` in merged frontmatter, profile `fallback:`) points at an existing profile. Skill `preferred_model:` is a hint and is NOT validated at load. Today "load" = startup; validation failures exit the process non-zero with a message naming the file and the offending reference.

**Temperature** is a per-profile field. Other knobs (max tokens, thinking budget, context-window override, …) can be added later — keep the schema minimal until there's demand.

#### Sub-agents

The agent can spawn focused sub-agents two ways:

- **`task`** (canonical) — agent-sdk's tuned sub-agent primitive: `Task` on Claude, `collabAgentToolCall` on Codex, `Agent.asTool` on the `openai` backend, special-cased child runner on Vercel. Use when the model wants a focused worker without protos's skill-list/tool-allowlist conventions.
- **`delegate_task`** (custom) — protos's skill-aware sub-agent. Caller provides explicit `skills: string[]`, `tools: string[]`, and optional `model:`. The skill bodies are inlined; the tool allowlist is enforced; the sub-agent log lands at the protos path layout (below). Skill `preferred_model:` resolution applies here.

A `delegate_task` sub-agent is a separate `agent.run()` invocation with:

- A task-specific prompt (written by the main agent at call time)
- An explicit list of skills (full skill bodies inlined into its system prompt)
- An explicit allowlist of tools (a subset of what the main agent has)
- An isolated context — no SOUL.md, no memory manifest, no conversation history

Sub-agents are not pre-defined as separate entities. There is no `spec/agents/` directory. The skills system already provides reusable behavior templates; `delegate_task` just lets the main agent compose them on demand into a focused sub-task. To make a sub-agent specialty reusable, write a regular skill in `spec/skills/{name}.md`.

**When to delegate:**

- Tasks that require many tool calls or large intermediate outputs (web research, multi-file analysis)
- Sub-tasks that benefit from a fresh focused context
- Anything where intermediate data shouldn't pollute the main context

**Constraints:**

- Sub-agents cannot spawn sub-agents. The runner strips `delegate_task` from the tool allowlist regardless of what the caller asks for. Single-level delegation, period.
- **Cost info is NOT surfaced to the calling agent.** Token usage and wall-clock duration can be recorded in the sub-agent's log (for operator visibility) but never appear in the `delegate_task` return value. Rationale: the calling agent should decide whether to delegate based on the task, not on running-total cost.

**Sub-agent logs.** Every `delegate_task` invocation writes its full event stream (same [event schema](#event-schema) as threads and cron logs) to a log file **nested under the caller's log**:

- Called from a cron run: `runtime/logs/cron/{jobname}/{ISO-timestamp}/{call_id}.jsonl` — sibling directory to the parent's `{ISO-timestamp}.jsonl` file.
- Called from a thread turn: `runtime/logs/sub-agents/{channelId}/{conversationId}/{turn_id}-{call_id}.jsonl`.

The parent's `tool_result` event for the `delegate_task` call carries an additional `sub_agent_log` field with the workspace-relative path of the sub-agent log file, so the full trace is recursively navigable.

See `spec/tools/delegate_task.md` for the full tool contract and build.md → step 4d for the runner implementation.

### 4. Scheduler

The scheduler runs cron jobs from two roots: `spec/cron/` (defaults shipped with the spec) and `config/cron/` (instance-specific jobs).

**One file per job.** Each job is a single `.md` file. The filename (minus extension) is the job name. Frontmatter holds the schedule and any options. The body holds the prompt sent to the agent when the job fires.

```markdown
---
schedule: "*/30 * * * *"
history: primary  # or: none
---

Check for unread messages that need follow-up. NO_REPLY if nothing to report.
```

Frontmatter fields:

- **`schedule:`** — cron expression. Required to fire.
- **`enabled:`** — defaults to `true`. Set to `false` to disable.
- **`model:`** — optional model profile name (see Model selection). Defaults to `default`.
- **`history:`** — what conversation context the agent sees when the job fires. Defaults to `primary`.
  - `primary` — the agent runs with the primary channel's owner conversation as context, capped at the most recent events (see `build.md` → step 4 for the exact cap and the turn-alignment rule). Use for outreach-style jobs that may want to reference recent activity (e.g. heartbeat).
  - `none` — the agent runs with no thread history; only the cron body is in context. Use for internal/background jobs that don't depend on a conversation (e.g. consolidation jobs that delegate to a sub-agent and write to memory).

**Layering.** When a file with the same name appears in both roots, the merged job uses:

- **Frontmatter:** config overrides spec (e.g., `schedule:` in config replaces spec's).
- **Body:** spec first, then config appended. Lets you extend a default prompt without restating it.

To disable a spec default, drop a config file with the same name and `enabled: false` in frontmatter.

There is no central registry. Adding a job means dropping a file. The merged view (with source annotations) is available via `agent/protos cron`.

When a job fires, the scheduler looks up the primary channel and sends the merged prompt to the agent through the router as a synthetic message addressed to the owner's main conversation. A reminder is appended: "If you have nothing to say to the owner, respond with exactly NO_REPLY and nothing else."

**Router rules for cron dispatch.** When the scheduler dispatches a cron job, it passes the `history` mode alongside the synthetic prompt. The cron log always receives the full run; the user thread always receives only the final assistant reply (or nothing on `NO_REPLY`). The `history` mode controls only what the agent reads on the way in:

- `history: primary` — agent reads the recent primary-thread events as context, then the synthetic prompt. Use for proactive jobs that may want to comment on recent activity (heartbeat).
- `history: none` — agent reads no prior history; only the synthetic prompt is in context. Use for internal jobs whose work is independent of the conversation (dream, nap).

In both cases, intermediate events (multi-step assistant text, tool calls, tool results) are suppressed from the user thread — the user only sees the final reply. The synthetic prompt itself is never written to the user thread either (it would otherwise show up on replay as if the owner had typed "check for unread messages…"). The full event stream lives in the cron log.

**Where the events land** (see [Storage → Message history](#message-history-runtimethreads) for the event schema):

| `history:` | Cron log (`runtime/logs/cron/…`) | User thread (primary channel) |
|---|---|---|
| `primary` | full run: `cron_start`, synthetic user prompt, all agent/tool events, `cron_end` | final assistant reply only (skipped on `NO_REPLY`) |
| `none` | full run: `cron_start`, synthetic user prompt, all agent/tool events, `cron_end` | final assistant reply only (skipped on `NO_REPLY`) |

`cron_start` / `cron_end` events exist only in cron logs. They carry the job name, schedule, duration, and final reply for audit purposes. The two files are independent records — matching entries line up by timestamp.

### 5. Self-modification

The agent can edit its own source code (`agent/src/`) via the canonical `write` and `edit` tools. TypeScript runs directly with `tsx` — no build step. To apply code changes, the agent restarts itself using the `agent/protos restart` wrapper script.

**Safe-edit protocol:**

1. The agent commits the change with a clear message using the `git` skill (requires `agent/` to be a Git repo — see [Git repos per domain](#git-repos-per-domain)).
2. The wrapper type-checks the code (`tsc --noEmit`) before restarting. If the check fails, the restart is aborted and the old process keeps running.
3. The wrapper starts the new process, then waits a few seconds and confirms it's still alive. If the process crashed at runtime (e.g. bad import, missing env var, startup exception), the wrapper **automatically reverts the last commit in `agent/`** and restarts with the pre-edit code.

This closes the three failure modes for self-edit: compile errors (caught by typecheck), runtime startup errors (caught by post-start health check + auto-revert), and subtle logic bugs (can be manually reverted via `git` from the running agent).

**Path discipline.** With native tools, protos no longer wraps `execute` to enforce path guards — the canonical tools come directly from agent-sdk. Discipline is convention plus OS-level enforcement:

- **System prompt reinforces convention** — `spec/` is the design (shared, read-only at runtime); `agent/` is the implementation (editable via the self-edit skill).
- **`chmod -R a-w spec/`** — recommended for production. The kernel refuses writes regardless of which tool issues them.
- **`chmod -R a-w agent/`** — to disable self-edit entirely. Pair with hiding the `self-edit` skill (the skills loader can be configured to skip a named skill via `config/skills/self-edit.md` with `enabled: false`).
- **Optional `git revert spec/` after each turn** — cron-driven or post-dispatch. Cheap, catches stray writes that landed before chmod could have stopped them.

For guaranteed enforcement against an adversarial agent, run the process in an OS-level sandbox with `agent/` and `spec/` mounted read-only.

`spec/` is not edited by the running agent. Spec changes are made by humans (or coding agents like Claude Code) and applied by asking a coding agent to update `agent/` to match.

## Storage

Everything is plain files — no database. Files break down by domain:

### Message history (`runtime/threads/`)

One JSONL file per `(conversation, profile)` pair, at `runtime/threads/{channelId}/{conversationId}/{profile}.jsonl`. Each line is one **event** teed from agent-sdk's `query.events` stream. A `{profile}.continuation` sidecar in the same directory holds the active backend session token.

```
runtime/threads/telegram/12345/
  default.jsonl          # canonical event log for the default profile
  default.continuation   # backend session token for resuming default
  fast.jsonl
  fast.continuation
```

The user-facing conversation is the merged view of all `*.jsonl` files in the directory, ordered by `timestamp`. See [Session continuity](#session-continuity) below for how we resume across profile switches.

#### Event schema

```ts
type Event =
  | { role: "user", text: string, attachments?: Attachment[], turn_id: string, timestamp: string }
  | { role: "assistant", text: string, tool_calls?: ToolCall[], turn_id: string, timestamp: string }
  | { role: "tool", tool_call_id: string, result: unknown, turn_id: string, timestamp: string }
  | { role: "audit", type: "cron_start", job: string, schedule: string, turn_id: string, timestamp: string }
  | { role: "audit", type: "cron_end", reply: string, duration_ms: number, turn_id: string, timestamp: string }

type ToolCall = { id: string, tool: string, args: unknown }
type Attachment = { type: "image", path: string, media_type: string }
```

Snake_case throughout. Events are derived from agent-sdk's typed `AgentEvent` stream by a tee layer in `agent/src/threads.ts` — every `text_end`, `tool_call_end`, `tool_result`, and inbound user message becomes a row. Tool calls are bundled with the assistant message they were part of via a flat `tool_calls` field. Tool results are separate events with `role: "tool"`. The optional `sub_agent_log` field on a tool result points to a nested log file; see [Sub-agents](#sub-agents). The optional `attachments` field on user events points to media blobs in `runtime/blobs/`; see [Attachments](#attachments-runtimeblobs).

`role: "audit"` events (`cron_start`/`cron_end`) exist for human/operator record-keeping and never round-trip through agent-sdk. `assistant.text` may be an empty string when a step produced only tool calls; the render filter skips such events when searching for the last assistant text of a turn.

#### `turn_id`

Every event produced by a single agent invocation shares one `turn_id`. A user message is its own turn; the assistant invocation that responds (potentially producing multiple text + tool_call + tool events through the multi-step loop) shares a different `turn_id`.

`turn_id` exists for **rendering**: each channel decides what to show per turn (default: only the last assistant text of the turn — see channel recipes for specifics). It is NOT used by the LLM context builder, which preserves the full event sequence regardless.

#### Session continuity

Each backend owns conversation-context reconstruction via its own continuation tokens. Protos owns the durable thread record at the path above and tees events from `agent.events` into JSONL. We never reconstruct a message array ourselves.

**One continuation active per dispatch, one continuation kept per profile across switches.** The `{profile}.continuation` sidecar holds the most recent `session_start.continuation` token for that profile; it's read on resume and rewritten whenever a fresh session is started.

**The five cases** the dispatcher handles:

1. **Cold start** — directory empty. `agent.run({ message })` no continuation. Save the resulting token to `{profile}.continuation`. Append events to `{profile}.jsonl`.
2. **Same-profile resume** — `{profile}.continuation` exists. Pass token: `agent.run({ message, continuation })`. Backend resumes from native storage (Claude/Codex SDK files; agent-sdk's session JSONL on Vercel/`openai`).
3. **Profile switch forward** (fresh into a not-yet-seen profile) — compute the recap from other profiles' tails, prepend to the user message, call `agent.run({ message: <recap + actual> })`, save new continuation.
4. **Profile switch back** (`{profile}.continuation` exists, but other profiles have run since) — compute the gap (events with `timestamp > T_{profile}_last` from other profiles' JSONLs), prepend recap to user message, call `agent.run({ message, continuation })`. Resume the prior session AND inject the gap context.
5. **Continuation invalid** — backend signals `isContinuationInvalid(err)`; clear the sidecar and retry as case 3.

**Gap detection.** Read the last event timestamp from `{profile}.jsonl` (`T_last`). Read events from every other profile's JSONL where `timestamp > T_last`. Merge by timestamp. Cap at last 10 events; verbatim text (no summarization for v1).

**Recap format.** In-message prepend, never written to threads JSONL. Plain readable format:

```
While you were inactive, conversation continued on profile `fast`. Recent turns:

  user (14:32 PDT): when does the train leave?
  fast (14:32 PDT): 4:15 PDT.
  user (14:48 PDT): thanks, how long is the ride?
  fast (14:48 PDT): about 2 hours.

[sent 2026-04-25 15:01:12 PDT] is there a dining car?
```

The actual user message keeps its existing `[sent …]` prefix as the boundary cue. Indented gap turns use `role (time): text` — no fake `[sent …]` headers competing with the convention. For case 3 (no prior session), reword the intro to "Recent conversation activity (other profiles):" — no "while you were inactive" framing since there's no prior session.

**Recap is runtime-only.** The recap header gets prepended to the `message` field passed to `agent.run`. It is NOT written to `{profile}.jsonl`. The user event in the JSONL records only what the user actually typed. Otherwise the recap would appear on render/replay as a fake user soliloquy, and future gap calculations would treat it as real conversation, compounding.

**Render across profiles.** Channel render reads ALL `*.jsonl` files in the directory and merges by `timestamp`. Users see one continuous conversation regardless of which profile served which turn. Memory consolidation (nap/dream) reads the same merged view.

**Send-time annotations on user messages.** Every user message we hand to `agent.run` gets prefixed with a `[sent YYYY-MM-DD HH:MM:SS ZZZ]` marker derived from the current timestamp, formatted in the owner's local timezone. Get the timezone from `Intl.DateTimeFormat().resolvedOptions().timeZone` and format the zone as a short abbreviation (e.g. `PDT`, `EST`, `UTC`). This gives the model a "when" for the latest user message, which doubles as its "now" for any duration calculations ("how long since I said X?"). Assistant events are NOT prefixed — the model can infer its own timing from the bounding user messages, and the asymmetric format discourages the model from mimicking the prefix in its own replies.

**Attachments on user messages.** If a user event has `attachments`, we pass them via `QueryInput.attachments` (agent-sdk handles the per-backend wire format). For local blobs we use `{type: 'image', source: {kind: 'path', path: '<runtime/blobs/...>'}}`.

**Replay safety.** When passing an attachment to agent-sdk, verify the resolved absolute path is under `runtime/blobs/` — reject anything that escapes (e.g. a crafted `path: "../etc/passwd"` would otherwise read host files). **Missing blobs** (cleanup, partial sync, machine swap) get replaced with a `[image unavailable: <path>]` text part and a logged warning; the rest of the conversation continues. Don't fail the whole call.

**Image events in the gap.** If the gap recap includes user events with `attachments`, the recap text mentions them as `[user sent a photo]` rather than re-embedding the bytes. Image bytes remain in our JSONL for human replay; the next turn can re-share if needed.

**Where each backend stores its own session record:**

| Profile | Backend reads on resume | Backend writes during turn |
|---|---|---|
| `claude` | `~/.claude/projects/.../{session-id}.jsonl` (Claude SDK's storage) | Same |
| `codex` | `~/.codex/sessions/...` + sqlite | Same |
| `openai` | `runtime/sessions/openai/{continuation}.jsonl` (`AgentInputItem` lines) | Appends `AgentInputItem` rows |
| `vercel` | `runtime/sessions/vercel/{continuation}.jsonl` (UIMessage) | Appends UIMessage rows |

The SDK's session storage is opaque to protos — never read or written directly. Our threads JSONL is the canonical durable record.

#### Append cadence and writers

- **Append-only writes.** The router serializes per-conversation, so there's never a concurrent writer on the same `(channelId, conversationId)` directory.
- Each event is appended to its profile's JSONL as it streams from `agent.events`. One agent invocation typically appends multiple events to a single `{profile}.jsonl`.
- **Reads** load the whole directory and parse line-by-line, merging across profiles by timestamp. For a personal assistant this is fine even across years of conversation.
- **Backup** is copying both `runtime/threads/` and `runtime/blobs/` together. The threads JSONL references blobs by path; without the bytes, replay produces missing-blob warnings rather than the original images. `runtime/sessions/` is also worth backing up if you want the Vercel/`openai` backends to truly resume sessions across machines — without it, the next turn falls into case 5 (continuation invalid) and runs as a fresh session with a recap.

### Attachments (`runtime/blobs/`)

Inbound media (images today; audio and documents stay as placeholders for v1) is stored as content-addressed blobs at `runtime/blobs/{sha256}.{ext}`. Thread events reference blobs by workspace-relative path in their `attachments` field; content addressing deduplicates repeat-sent media automatically.

The channel adapter is responsible for downloading bytes, hashing them, writing under `runtime/blobs/`, and attaching `{ type, path, media_type }` to the dispatched message. Channels without attachment support never touch this directory.

Blobs are kept indefinitely — there is no automatic retention. Modern phone photos are MB-scale per file, so the directory grows over time; clean up manually or add a cleanup cron once it matters.

**Cron exposure.** Cron jobs with `history: primary` reconstruct recent thread events into context on every firing — including any attachments those events carry. Image-heavy threads multiply the per-firing token cost. If a job (e.g. heartbeat) doesn't actually need recent conversation context, switch it to `history: none`.

### Cron logs (`runtime/logs/cron/`)

One JSONL file per cron run, at `runtime/logs/cron/{jobname}/{ISO-timestamp}.jsonl`. Uses the [same event schema](#event-schema) as threads, with `cron_start` and `cron_end` audit events bracketing the run. For `history: none` jobs (dream, nap) the cron log is the primary audit trail; for `history: primary` jobs (heartbeat) the cron log is an additional record — the same agent/tool events also land in the user's thread. See [Scheduler](#4-scheduler) for which events go where.

No retention policy — kept forever. Small text, useful for the agent to reflect on past runs via `read`.

### Identity, behavior, memory, ephemeral

All plain files spread across the domains:

- **`config/SOUL.md`** — identity. The agent writes this on first run after asking the user for a name and personality. Read on every invocation.
- **`config/models.yaml`** — LLM profiles and credentials. Read at startup. See Model selection.
- **`config/channels.yaml`** — messaging channels and credentials. Read at startup. See Channels.
- **`config/cron/`** — instance-specific scheduled jobs (markdown).
- **`config/skills/`** — instance-specific skills. Accepts both the spec's flat `{name}.md` form and the agentskills.io directory form (`{name}/SKILL.md` + optional `scripts/`).
- **Custom channels and tools** live in `agent/src/channels/` and `agent/src/tools/` alongside the built-in ones — `config/` holds no code.
- **`memory/`** — granular markdown files of long-term knowledge. See **Memory format** below.
- **`memory/journal/`** — daily scratch pad files (e.g., `memory/journal/2026-03-09.md`). The agent jots notes throughout the day; the `dream` cron promotes important items into the rest of `memory/`.
- **`memory/new/`** — inbox folder. The agent writes new notes here when there's no obvious folder yet (use `add_memory("new/{name}", content)`). The `dream` cron sorts the inbox into appropriate folders. Prefer confident placement (`add_memory` directly into the right folder) when you know where it belongs; use the inbox only when genuinely unsure.
- **`memory/archive/`** — cold storage for content `dream` decided isn't currently useful but might be worth keeping. Exempt from the orphan check — things here don't need to be reachable from a root file. Use `rename_memory("{name}", "archive/{name}")` to move something into the archive.
- **`runtime/`** — `threads/` (per-conversation directories of `{profile}.jsonl` + `{profile}.continuation` sidecars + per-profile consolidation cursors), `sessions/{vercel,openai}/{continuation}.jsonl` (agent-sdk session storage for backends without their own), `logs/cron/{jobname}/{ISO}.jsonl` (one per cron run), `logs/cron/{jobname}/{ISO}/{call_id}.jsonl` (sub-agent logs), `logs/sub-agents/` (thread-originated sub-agent logs — see [Sub-agents](#sub-agents)), `*.pid`, `memory-graph.json` (backlink cache). See **Memory consolidation** below for cursor details.

## Memory format

Memory files are standard markdown with optional YAML frontmatter. The format is **Obsidian-compatible** — you can open `memory/` as an Obsidian vault and get a free GUI for browsing and editing.

### File anatomy

```markdown
---
created: 2026-04-18T10:30:00Z
updated: 2026-04-18T11:00:00Z
description: My coffee preferences
tags: [coffee, preferences]
aliases: [coffee-order, my-coffee]
---

I like my coffee black, no sugar. Usually from [[people/justin]]'s
favorite place, [[blue-bottle]].

See also: ![[preferences/morning-routine]]
```

- **Frontmatter** holds intrinsic metadata about the note. Standard keys (`tags`, `aliases`, `created`, `updated`, `description`) match Obsidian conventions. Engine-specific fields are namespaced — e.g., `protos.confidence`, `protos.source` — to avoid collision with Obsidian or its plugins.
- **Body** is markdown. Tasks (`- [ ]`) work with Obsidian's Tasks plugin.

### Linking

Forward links use the wiki-link syntax:

- `[[note-name]]` — link by name (no `.md` extension)
- `[[note-name|display text]]` — link with custom display text
- `![[note-name]]` — embed (inlines the linked file's content)
- `[[path/to/note]]` — disambiguate when multiple files share a name

**Resolution rules** (matching Obsidian):

1. Find all files in `memory/` whose filename (without `.md`) equals the link target, or whose `aliases:` list contains it
2. If exactly one match: that's the link target
3. If multiple matches: pick the **shortest path**, then alphabetical
4. If no match: return no target (no auto-creation). The agent decides whether to create a file via `add_memory` and where to put it. This matches Obsidian's spirit — Obsidian only creates a target file when the user *clicks* a missing link, not when one is parsed.

**Resolution preservation on changes.** When a file is created, renamed, or moved, bare-name links (`[[coffee]]`) can silently change which file they resolve to — another file with the same basename might now win the shortest-path tiebreak, or the shadowed file might get un-shadowed. The `add_memory` and `rename_memory` tools handle this by snapshotting the full resolution map before the change and rewriting any reference whose resolved target would change after the change. The canonical `write` tool under `memory/` does NOT run this preservation step — prefer `add_memory` for any memory-file creation that could introduce name shadowing.

### Backlinks

Backlinks are not stored. They're computed by scanning all `memory/**/*.md` for `[[...]]` references and building a reverse index. The graph is cached at `runtime/memory-graph.json` and rebuilt when memory files change (mtime check at startup).

The `find_memory` tool returns a file's path and its backlinks ("files that reference this one") on hit — see the exact return shape under [Tool return shapes](#tool-return-shapes). This lets the agent traverse the graph naturally.

### Loading into context

Memory is **not loaded eagerly** into the system prompt. Instead, the system prompt includes a **manifest of root-level files only** — every `memory/*.md` file (not recursing into subfolders), with its name, aliases, tags, and a one-line summary. The summary comes from (in order of preference):

1. The frontmatter `description:` field, if present
2. The first H1 heading in the body
3. The first ~100 characters of body text

The manifest gives the agent enough context to know what high-level topics exist. The agent uses `find_memory` and `read` to fetch full content on demand, based on conversation context.

**Root-level files act as entry points.** Memory is organized so that anything important is reachable from a root file via `[[wiki-links]]` (transitively). Subfolder files exist for organization but are never surfaced in the manifest — the agent finds them by following links from root files. This is the standard "Map of Content" pattern: root files are the front page; deep files live in folders but are linked from above.

The reachability invariant — every non-root file is transitively linked from at least one root file — is maintained by the `dream` cron job, which runs an orphan check during its daily sweep and either links orphans in or archives them.

**Journal exception:** the last 24 hours of `memory/journal/` entries are included in the system prompt directly, since they're recent agent-authored notes likely to be relevant. Journal entries are NOT in the root manifest (they're in the `journal/` subfolder); older entries are reachable only by date.

### Obsidian compatibility

If `memory/` is opened as an Obsidian vault, Obsidian creates a `.obsidian/` directory for vault settings (workspace layout, plugin config). This directory is machine-local and should be in `memory/.gitignore`. Configure Obsidian's "Default location for new notes" → `new/` so notes you create in Obsidian land in the same inbox the agent uses.

## Memory consolidation

Conversation threads are append-only and grow forever. To keep the agent's working knowledge useful as threads grow, two scheduled jobs distill thread content into long-term memory.

**Two-tier consolidation:**

- **`nap`** (hourly, `history: none`) — quick per-thread pass. Walks `runtime/threads/`; for any thread whose unconsolidated message count exceeds a threshold, reads the unconsolidated tail, writes summaries to memory, and advances the cursor. Scoped to one thread at a time. Doesn't touch journal or inbox.
- **`dream`** (daily, `history: none`) — deep full sweep. Reads all threads (allowing cross-thread correlations), the day's journal, and the `memory/new/` inbox. Updates memory files, runs the orphan check (every non-root file reachable from a root file via `[[wiki-links]]`), promotes hot content to root level, archives or deletes cold content. Posts a brief summary to the user.

Both jobs run with `history: none`: the agent's context for the invocation starts clean (no prior conversation history is loaded), so the consolidation work doesn't need to be isolated in a sub-agent — the cron invocation itself is already an isolated context. Only the final summary reply lands in the user's thread.

**Consolidation cursors.** Per-thread, per-profile cursors record how many messages have been consolidated into memory. Stored as sidecar files in the conversation directory: `runtime/threads/{channelId}/{conversationId}/{profile}.cursor`, each containing a single integer (the count of consolidated messages from the start of that profile's JSONL). Consolidation walks the merged event view (across all profiles in the directory), tracks which profile each event came from, and advances each profile's cursor as it processes. Both `nap` and `dream` read and update these cursors so neither re-consolidates content the other has already processed. Cursors live in `runtime/` intentionally — they share the lifecycle of the threads they describe. Wiping `runtime/` (the documented "rebuild from scratch" action) drops the threads and their cursors together.

**Thread context at runtime is unchanged.** The agent always sees the most recent N messages of the active thread (capped at the token budget). Consolidated older content reaches the agent through the memory manifest, not through any thread-content rewriting. The thread is "what's happening now"; memory is "what I know."

## First-run flow

On startup, the agent checks for `config/SOUL.md`. If it doesn't exist:

1. The agent introduces itself as a blank Protos and asks the user for a name and personality.
2. The agent writes `config/SOUL.md` with the chosen identity (using `write`).
3. Subsequent startups read the populated file.

The agent also creates `config/`, `memory/`, and `runtime/` directories on first run if they don't exist.

**Cron firings are suppressed while first-run setup is incomplete.** If a cron job's scheduled time arrives before `config/SOUL.md` exists, the scheduler skips that firing — otherwise the agent would respond with the first-run intro ("what should I be called?") to a heartbeat or dream prompt, polluting the cron log and confusing the user. Self-healing: the next firing after `config/SOUL.md` lands works normally.

## Startup flow

1. Ensure `runtime/` directory exists (`runtime/threads/` is created lazily on first message)
2. Load `config/models.yaml`: parse, run `$NAME` substitution, resolve pointers, validate (see Model selection → Validation at load). Error out on any invalid entry.
3. Discover skills (scan `spec/skills/*.md`; scan `config/skills/` for both `*.md` and `*/SKILL.md`; load names and descriptions; on name collision, merge — config frontmatter wins, config body appended to spec body)
4. Discover tools (scan `agent/src/tools/`)
5. Load `config/channels.yaml` and register channels: for each entry, look up its `provider:` (default = entry name) in `agent/src/channels/` and instantiate the provider module with the entry's config. Skip entries with `enabled: false`. If no channels register, exit with an error.
6. Start the scheduler (scan `spec/cron/` and `config/cron/`; merge by filename per the layering rules above)
7. If `config/SOUL.md` doesn't exist, run first-run flow
8. Begin processing incoming messages

## File structure

```
# Workspace root — only these are tracked by the spec repo
README.md
CLAUDE.md
AGENTS.md
.gitignore
spec/

# Spec — tracked
spec/
  architecture.md     # this file
  agent.md            # base prompt loaded into the running assistant
  build.md            # build instructions for coding agents
  channels/           # channel recipes (markdown only)
    telegram.md
    terminal.md       # local client-server chat over Unix socket
    whatsapp.md
    ...
  tools/              # tool recipes for the protos-specific custom tools (canonical tools live in agent-sdk)
    find_memory.md
    add_memory.md          # memory-aware create + resolution preservation
    rename_memory.md       # memory-aware rename + resolution preservation
    add_journal_entry.md   # append timestamped line to today's journal
    delegate_task.md       # protos's skill-aware sub-agent
    list_threads.md        # consolidation: enumerate threads with cursor state
    read_thread_tail.md    # consolidation: read messages since cursor
    advance_thread_cursor.md  # consolidation: mark messages consolidated
    find_orphans.md        # memory hygiene: unreachable non-root files
  skills/             # bundled skills (one .md per skill — flat, not the agentskills.io directory format)
    self-edit.md
    git.md
    coding.md
    scheduling.md
    consolidation.md
    memory.md
    update.md
    browser-use.md
  cron/               # default cron jobs (markdown with frontmatter)
    heartbeat.md
    nap.md
    dream.md

# Generated implementation — gitignored, optionally a separate repo
agent/
  package.json
  tsconfig.json
  protos               # wrapper script (start/stop/restart/status/chat)
  src/
    index.ts          # entry point
    router.ts
    agent.ts
    scheduler.ts
    threads.ts
    memory.ts
    cli/              # isolated client code — does not import router/agent/memory
      chat.ts         # terminal client: connects to runtime/agent.sock
    channels/         # built-in + custom channels (built-in generated from spec/channels/ recipes)
      terminal.ts     # bundled — zero-config client-server channel
      terminal-protocol.ts  # shared JSON message shapes (server + cli/chat.ts)
      telegram.ts
      ...
    tools/            # protos-specific custom tools (canonical tools come from agent-sdk)
      find_memory.ts
      add_memory.ts
      rename_memory.ts
      add_journal_entry.ts
      delegate_task.ts
      list_threads.ts
      read_thread_tail.ts
      advance_thread_cursor.ts
      find_orphans.ts
      browser_fetch.ts
      impls/                  # withImpls executes for canonical tools (not standalone tools)
        tavily.ts             # webSearch override on Vercel
    agents/           # delegate_task sub-agent runner (single generic file, no per-agent definitions)
      runner.ts

# Behavior — gitignored, optionally a separate repo.
config/
  SOUL.md             # written on first run
  models.yaml         # LLM profiles + credentials (see Model selection)
  channels.yaml       # messaging channels + credentials (see Channels)
  .env                # optional — holds secrets referenced from YAML via $NAME
  cron/               # instance-specific or override jobs (markdown)
  skills/             # instance-specific skills — accepts both flat {name}.md and agentskills.io {name}/SKILL.md form

# Durable state — gitignored, optionally a separate repo
memory/
  .obsidian/          # vault settings (machine-local, gitignored)
  facts/
  people/
  preferences/
  summaries/
  journal/
    2026-03-10.md
  new/                # inbox — agent writes here when no obvious folder yet
  archive/            # cold storage — exempt from the orphan check

# External tooling — gitignored, each subdir is its own external repo
vendor/
  browser-harness/    # cloned + installed by the browser-use skill on demand
  chromium/           # Playwright's Chromium binary (browserFetch); installed at build via PLAYWRIGHT_BROWSERS_PATH
  ...                 # one subdir per vendored tool

# Ephemeral — gitignored, never a repo
runtime/
  threads/            # one directory per conversation, one JSONL per (conversation, profile)
    telegram/
      12345/
        default.jsonl
        default.continuation
        default.cursor          # consolidation cursor for this profile
        fast.jsonl
        fast.continuation
        fast.cursor
    terminal/
      cli/
        default.jsonl
        default.continuation
        default.cursor
  sessions/           # agent-sdk session storage (Vercel + openai own these; Claude/Codex don't)
    vercel/
      {continuation}.jsonl
    openai/
      {continuation}.jsonl
  blobs/              # content-addressed media (sha256-named) referenced by thread events
  browser-sessions/   # browserFetch persistent contexts, keyed by `session` arg (one user-data-dir per name)
    nyt/
    reddit/
  clients/            # per-client cursor files
    chat.cursor       # default terminal client
  logs/
  agent.sock          # Unix socket for terminal channel
  *.pid
  memory-graph.json   # backlink cache
```

## Capability layout

Channels and tools are **code** — they live in `agent/src/`, which is the user's own implementation repo. Their recipes live in `spec/`. Skills and cron are **behavior configuration** — markdown-only, layered between `spec/` (defaults) and `config/` (user overrides).

| | Recipes | Code | Instance config | Layered? |
|---|---|---|---|---|
| **Channels** | `spec/channels/{provider}.md` | `agent/src/channels/{provider}.ts` | `config/channels.yaml` entries | No — single code root; channel names from YAML |
| **Tools** | `spec/tools/{name}.md` | `agent/src/tools/{name}.ts` | — | No — single code root |
| **Skills** | `spec/skills/{name}.md` (built-in) + `config/skills/{name}.md` *or* `config/skills/{name}/SKILL.md` (user) | none | — | Yes — two-root, frontmatter override + body append |
| **Cron** | `spec/cron/{name}.md` (default) + `config/cron/{name}.md` (override) | none | — | Yes — two-root, frontmatter override + body append |

The asymmetry is intentional:

- **Code lives in `agent/` because `agent/` is the user's repo.** Built-in providers (generated from spec recipes during build) and custom providers (added by the user) live in the same directory. There's no need for a separate "user extension" location because the user already owns `agent/`.
- **Skills and cron live partly in `spec/` because they're behavior the spec ships with defaults for** (e.g., the heartbeat job, the `self-edit` skill). A user can override a default by dropping a same-named file in `config/`. No code means no node_modules or compilation — pure markdown layering.

Rules:

- **Channels: filename = provider name.** `agent/src/channels/telegram.ts` implements the `telegram` provider. Channel *names* are arbitrary and come from `config/channels.yaml`; they reference a provider by file-basename via their `provider:` field (or by matching the entry's name). The channel loader scans `agent/src/channels/` to build the provider map but only instantiates providers referenced from `channels.yaml`.
- **Tools / skills / cron: filename = capability name.** No central registry — the loader scans the directory and registers every file.
- **Recipes describe; implementations execute.** A recipe in `spec/channels/telegram.md` tells the build how to generate `agent/src/channels/telegram.ts`. Recipes never name the implementation path explicitly — the path is determined by their own location.
- **Custom channels don't need spec recipes.** A user adding a provider directly to `agent/src/channels/` can colocate the optional `.md` alongside it, or skip the recipe entirely.

### Adding a channel

A provider module in `agent/src/channels/` exports a factory function that the registry calls once per matching `channels.yaml` entry:

```ts
export function createChannel(name: string, config: ChannelConfig, router: Router):
  { channelId: string, ownerConversationId: string, send: (text: string) => void }
```

- `name` is the top-level key from `channels.yaml`; the factory uses it as `channelId` so thread storage and cron routing can target this specific channel.
- `config` is the entry's resolved config (credentials and platform-specific fields), post `$NAME` substitution and pointer resolution.
- The factory connects, starts forwarding owner messages to the router (ignoring all others), and returns. If connection fails (bad credentials, network), it throws — the registry logs the failure and continues with the remaining channels.

To add a channel: write `agent/src/channels/{provider}.ts` and add an entry to `config/channels.yaml` naming it. If the provider is reusable, also write a recipe at `spec/channels/{provider}.md` so future users can build it.

### Applying spec updates

**Re-running build is not idempotent.** Build is a one-time operation that initializes `agent/`. When the spec updates and you want the changes in your existing `agent/`, point a coding agent at the updated spec and ask it to apply the delta: "Here's my current `agent/`, here's the updated `spec/`. Update the code to match the new spec, preserving my custom additions."

This keeps the build simple (no manifest tracking, no merge logic) and makes spec updates explicit — you see the diff, you approve the changes.

## Permissions model

| Domain | Agent (running) | Coding agents (Claude/Codex) | Human |
|--------|-----------------|------------------------------|-------|
| `spec/` | read | read/write | read/write |
| `agent/` | read + self-edit via skill | read/write | read/write |
| `config/` | read/write | read/write | read/write |
| `memory/` | read/write | read/write | read/write |
| `vendor/` | read + exec; edits via the relevant skill (typically delegated to a coding agent) | read/write | read/write |
| `runtime/` | read/write | read-only | read/write |

Read `runtime/` to debug. Modify the source domains (`spec/`, `agent/`, `config/`, `memory/`) to fix.

## Git repos per domain

Each domain has its own lifecycle, so each should have its own Git repo:

| Domain | Recommended? | Purpose of the repo |
|--------|--------------|---------------------|
| `spec/` | **Required** (this repo) | The canonical design, shared across all Protos users |
| `agent/` | **Strongly recommended** | History and rollback for self-edits; portable across machines |
| `config/` | **Recommended** | Sync behavior/identity across machines |
| `memory/` | **Recommended** (private) | Durable knowledge with line-item history |
| `vendor/` | Per subdir (each is its own external repo) | Each vendored tool already has its own upstream Git history; nothing extra to manage at the `vendor/` level |
| `runtime/` | Never | Ephemeral state, not versioned |

Why `agent/` matters most: self-edit depends on it. If `agent/` is a Git repo, the wrapper can auto-revert a bad edit when the new process crashes on startup. Without a Git repo, a bad edit that passes `tsc --noEmit` but crashes at runtime becomes a manual recovery problem.

**Setup** (run once, from inside each directory):

```bash
cd agent && git init && git add -A && git commit -m "build"
cd config && git init && git add -A && git commit -m "initial config"
cd memory && git init  # commits happen as the agent learns
```

`.env` and any other secret files should be gitignored inside `config/`.

**Multi-machine model:** The same `memory/` repo can be shared across multiple machines running different `agent/` versions and different `config/` layers. Read access can be shared freely; write authority should be explicit. Typical model: one primary writer, or multiple writers resolving via Git.

`runtime/` is never portable — it represents current embodiment, not identity.

## Guidelines

- **Keep it simple.** One file per component. Minimal abstractions.
- **No over-engineering.** Don't build plugin systems, middleware chains, or event buses. Direct function calls are fine.
- **Fail gracefully.** If a channel can't connect, log it and move on. Don't crash the process. The entry point installs top-level `unhandledRejection` and `uncaughtException` handlers that log the error and keep the daemon running — a single failure (API billing, transient network, a bug in one handler) must not take down every channel and cron job.
- **Log usefully.** Log when channels connect, when messages are received/sent, and when errors occur.

## Security considerations

- **OS sandbox is the safety boundary.** Protos runs unattended, so all four backends are configured to skip in-process permission prompts (see Model selection → Unattended defaults). Tool execution effectively has the same authority as the host user across every backend. Run protos on a dedicated machine, in a container, or under a per-user macOS sandbox profile.
- The canonical **`bash` tool** runs commands using bash from the workspace root. On Claude with `permission_mode: bypassPermissions` and Codex with `ask_for_approval: never`, the SDKs themselves don't gate execution — they just dispatch. On Vercel and `openai`, agent-sdk's bundled in-process impl runs the command directly. The agent should confirm destructive commands with the user before running them.
- **Never log credentials.** When catching errors, log only the error message — not full objects, which may contain API tokens or bot secrets.
- Secrets live in `config/models.yaml` and `config/channels.yaml` (inline, or referenced via `$NAME` substitution from `config/.env`).
- The thread files in `runtime/threads/` contain all your messages — protect that directory accordingly.

## Non-goals

These are intentionally left out. They may be added later.

- **Multi-user conversations** — all messages are assumed to be from the owner
- **Web UI or dashboard** — the messaging app is the interface
- **Plugin system** — channels, tools, and skills are just files, not a formal plugin API
- **Submodules** — `agent/`, `config/`, and `memory/` are independent repos when promoted, never submodules
