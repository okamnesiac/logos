# Architecture

## Overview

Protos is a single-process personal AI assistant. Messages come in from messaging channels, get processed by an AI agent, and responses go back out through the same channels.

The workspace is organized into **six sibling domains**, each with a distinct role and lifecycle:

```
protos/                # workspace root
  spec/               # the blueprint — architecture, build, recipes, defaults
  agent/              # generated implementation — code that the bootstrap produces
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

`spec/` is **read-only at runtime**. The file-edit tools (`write_file`, `edit_file`) refuse any path under `spec/` regardless of other settings; `shell` carries a description nudge with the same rule. Spec changes happen out-of-band — via PR or via a coding agent invoked outside the running daemon. Instance-specific extensions to skills and cron go in `config/skills/` / `config/cron/`, which the loader merges with `spec/` at startup — config frontmatter wins, config body appends to spec body. See the **Layering** notes under Skills and Scheduler for details.

### `agent/` — generated implementation

The TypeScript code produced by the bootstrap: `index.ts`, `router.ts`, the channel implementation(s) the user chose, the built-in tool implementations, the wrapper script. Gitignored by this repo. Optionally a separate Git repo if you want to commit your specific implementation.

The agent edits its own source under `agent/src/` via the `self-edit` skill.

### `config/` — behavior

How this specific instance behaves: identity (`SOUL.md`), instance-specific overrides for tools/skills/channels/cron, environment variables. Per-user. Optionally a separate Git repo.

Behavior reads context, but context does not define behavior.

### `memory/` — durable state

What the agent knows and is committed to: facts, preferences, summaries, journal entries, durable task state. Per-user, private. Optionally a separate Git repo.

Granular files, not one consolidated file — Git history is more useful when changes are line-item rather than whole-file.

### `vendor/` — external tooling

Third-party tools the agent shells out to: cloned repos, editable installs, anything that lives outside the protos codebase but the agent uses regularly. One subdirectory per vendored tool (`vendor/browser-harness/`, …), each its own external Git repo with its own lifecycle.

Per-instance and gitignored by this repo. Skills that depend on a vendored tool name the canonical install path (`vendor/{name}/`) so the agent has one place to look. The agent may edit files inside `vendor/` (per the relevant skill — typically by delegating to a coding agent), but those edits are local to the vendored repo, not commits in `agent/` or `spec/`.

Empty by default — vendored tools are opt-in, installed when a skill walks the owner through setup.

### `runtime/` — ephemeral state

What is happening right now: message threads, logs, pid files, locks, caches. Machine-local, never versioned, safe to delete and rebuild.

Practical rule: if it works well with Git, it belongs in `memory/`. If it doesn't merge well, it belongs in `runtime/`.

## Tech stack

| Component | Choice |
|-----------|--------|
| Runtime | Node.js 18+ |
| Language | TypeScript |
| Storage | Plain files — no database |
| AI | Vercel AI SDK (`ai`) with `@ai-sdk/anthropic` and `@ai-sdk/openai` bundled |
| Hosting | Runs directly on the host — no containers |

## Components

### 1. Channels

Channels are messaging platform integrations. Each channel:

- Connects to a messaging platform (WhatsApp, Telegram, Discord, Slack, etc.)
- Receives inbound messages and normalizes them
- Sends outbound messages from the agent back to the platform
- Shows a typing indicator while the agent is thinking

**Owner-only:** Each channel knows the owner's ID on that platform (e.g., `TELEGRAM_OWNER_ID`). Messages from anyone else are silently ignored.

**Self-registering:** A channel only activates if its credentials are present in the environment. No credentials, no channel — no errors, no configuration needed.

**Main chat:** The `PRIMARY_CHANNEL` environment variable names the channel used for the owner's main conversation (e.g., `telegram`). The scheduler sends replies to the owner's conversation on this channel.

**Recipes vs implementations:** Channel **recipes** (`.md` setup guides) live in `spec/channels/` — they describe how to build each channel. Channel **implementations** (`.ts` code) live in `agent/src/channels/`. Both built-in channels (generated from spec recipes by the bootstrap) and custom channels (added by the user) live in the same directory — the `agent/` repo is the user's own implementation, so there's no need for a parallel code location in `config/`.

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

The agent is the brain. It uses the Vercel AI SDK to receive a message + history, decide how to respond, optionally use tools, and return a response.

**Tools** are typed capabilities defined in code. Each built-in tool has a recipe in `spec/tools/{name}.md` describing its inputs, outputs, behavior, and dependencies — same pattern as channels. Implementations live in `agent/src/tools/`.

The kernel ships eight tools:

- `read_file(path)` — read any file in the workspace
- `write_file(path, content, mode)` — `create` (fail if exists), `append`, or `replace` (overwrite)
- `edit_file(path, old_string, new_string)` — surgical find-and-replace; `old_string` must appear exactly once
- `find_memory(name)` — resolve a wiki-link-style name to every matching file, sorted shortest-path first (see [Tool return shapes](#tool-return-shapes))
- `add_memory(name, content)` — create a new memory file and preserve any `[[wiki-link]]` resolutions the new file would otherwise shadow
- `rename_memory(oldName, newName)` — move/rename a memory file and rewrite `[[wiki-links]]` so every reference resolves to the same file it did before
- `remember(text)` — sugar for appending to today's journal
- `shell(cmd)` — run a shell command (1 MB output limit)
- `delegate_task({ prompt, skills, tools, model? })` — spawn a sub-agent in an isolated context (see [Sub-agents](#sub-agents))
- `web_fetch(url)` — fetch a URL and return content as markdown (see `spec/tools/web_fetch.md` for backend choice and privacy)

Memory-aware tools (`find_memory`, `add_memory`, `rename_memory`) take names in **wiki-link form** — no `memory/` prefix, no `.md` extension. They return full workspace paths (e.g. `memory/preferences/coffee.md`) in their output for interop with `read_file`.

Users can add their own tools directly in `agent/src/tools/` — same location as the built-in tools. The AI SDK handles tool execution loops natively — limit the number of steps to prevent runaway tool use.

#### Tool return shapes

Tool return values get JSON-serialized and shown to the model. Two conventions:

- **Tools that succeed-or-fail** (`read_file`, `write_file`, `edit_file`, `remember`, `shell`) return their result on success. On failure they may throw; the agent's tool-dispatch layer catches the error and records it as the tool result so the model sees the error message.
- **Tools that succeed-or-miss** (anything that does lookup) return a **discriminated shape** — a list (empty = miss) or a union with a boolean tag — never a bare `null`. Makes the shape unambiguous to the model and leaves room for diagnostic fields later. See `spec/tools/find_memory.md` for the canonical example.

**Invariant: every tool call in the thread JSONL is paired with a tool result event.** The dispatch layer guarantees this regardless of whether the tool returned, threw, or timed out. A missing result orphans the `tool_calls` entry, and subsequent invocations can't replay the history — the LLM SDK rejects message histories with orphan tool calls, and the conversation stops responding.

The invariant is achieved by wrapping each tool's `execute` at registration time with an error-catching adapter. If the underlying call throws, the adapter returns `{ error: <message> }` as the result instead of propagating the throw. This keeps the LLM SDK's normal success path — including the entry in `step.toolResults` — so the dispatch layer's `onStep` callback writes a corresponding `tool` event to the thread.

**Skills** are markdown instruction files that teach the agent how to accomplish complex tasks using its tools. Bundled skills live in `spec/skills/{name}.md` — flat, one file per skill. Instance-specific skills live in `config/skills/`, which accepts both the same flat `{name}.md` form and the agentskills.io directory form (`{name}/SKILL.md` plus optional `scripts/`, `references/`, `assets/` sibling directories) so off-the-shelf skills drop in unmodified.

**Layering.** When a skill with the same name exists in both roots, the merged skill uses:

- **Frontmatter:** config wins on field collisions; missing config fields fall through to spec.
- **Body:** spec first, then config appended. Lets the user (or the agent itself) extend a default skill without restating it. Same rule as cron — agents tend to extend skills more often than replace them outright, so append-by-default is the right shape.

The skills summary in the system prompt shows both source paths for a merged skill (e.g. `spec/skills/X.md + config/skills/X.md`) so the agent knows where each part came from when re-reading. Within `config/`, if both flat (`{name}.md`) and directory (`{name}/SKILL.md`) forms exist for the same name, the flat form wins — these are not merged into each other, only the winning config form merges with the spec.

**Identity** comes from `config/SOUL.md`, written on first run. The agent reads it on every invocation.

**Long-term memory** comes from the `memory/` directory. The system prompt includes a manifest of **root-level files only** (name + summary); subfolder files are reached by following `[[wiki-links]]` from a root file. Full content is fetched on demand via `find_memory` and `read_file`. See **Memory format** below.

**Self-awareness.** The agent learns about its scheduled-job invocation mode via the bundled `scheduling` skill (`spec/skills/scheduling.md`), whose name and description appear in the skills summary at startup. Without this, the agent doesn't know cron exists and may tell the user it can't reach out proactively — which is wrong; cron firings are exactly the proactive-outreach mechanism.

#### Sub-agents

The agent can spawn focused sub-agents via the `delegate_task` tool. A sub-agent is a separate `generateText` invocation with:

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

The agent can edit its own source code (`agent/src/`) via its file-edit tools (`write_file`, `edit_file`). TypeScript runs directly with `tsx` — no build step. To apply code changes, the agent restarts itself using the `agent/protos restart` wrapper script.

**Safe-edit protocol:**

1. The agent commits the change with a clear message using the `git` skill (requires `agent/` to be a Git repo — see [Git repos per domain](#git-repos-per-domain)).
2. The wrapper type-checks the code (`tsc --noEmit`) before restarting. If the check fails, the restart is aborted and the old process keeps running.
3. The wrapper starts the new process, then waits a few seconds and confirms it's still alive. If the process crashed at runtime (e.g. bad import, missing env var, startup exception), the wrapper **automatically reverts the last commit in `agent/`** and restarts with the pre-edit code.

This closes the three failure modes for self-edit: compile errors (caught by typecheck), runtime startup errors (caught by post-start health check + auto-revert), and subtle logic bugs (can be manually reverted via `git` from the running agent).

**Disabling self-edit:** set `PROTOS_SELF_EDIT=false` in `config/.env`. When disabled:

- The `self-edit` skill is hidden from the agent (skills loader skips it).
- `write_file` and `edit_file` reject any path that resolves under `agent/` with a clear error.
- The `shell` tool gets a warning appended to its description: "self-edit is disabled; do not modify files under `agent/`." This is a nudge, not enforcement — the shell tool can still technically write anywhere the process user can.

This is **safety by convention plus tool guards** — enough to prevent accidental or unprompted self-edit. For guaranteed enforcement (e.g. against an adversarial or confused agent), run the process in an OS-level sandbox with `agent/` and `spec/` mounted read-only.

Default is `PROTOS_SELF_EDIT=true`. The capability exists and is well-guarded; users nervous about it flip one env var.

`spec/` is not edited by the running agent. Spec changes are made by humans (or coding agents like Claude Code) and applied by asking a coding agent to update `agent/` to match.

## Storage

Everything is plain files — no database. Files break down by domain:

### Message history (`runtime/threads/`)

One JSONL file per conversation, at `runtime/threads/{channelId}/{conversationId}.jsonl`. Each line is one **event** — a single LLM message in the AI SDK `CoreMessage` shape.

#### Event schema

```ts
type Event =
  | { role: "user", text: string, turn_id: string, timestamp: string }
  | { role: "assistant", text: string, tool_calls?: ToolCall[], turn_id: string, timestamp: string }
  | { role: "tool", tool_call_id: string, result: unknown, turn_id: string, timestamp: string }
  | { role: "audit", type: "cron_start", job: string, schedule: string, turn_id: string, timestamp: string }
  | { role: "audit", type: "cron_end", reply: string, duration_ms: number, turn_id: string, timestamp: string }

type ToolCall = { id: string, tool: string, args: unknown }
```

Snake_case throughout the wire format. Tool calls are bundled with the assistant message they were part of via a flat `tool_calls` field (mirroring AI SDK and Anthropic API conventions). Tool results are separate events with `role: "tool"`. The optional `sub_agent_log` field on a tool result points to a nested log file; see [Sub-agents](#sub-agents).

`role: "audit"` is distinct from AI SDK's `role: "system"` — LLM-oriented roles (`user`, `assistant`, `tool`) feed the context reconstruction; `audit` events exist only for human/operator record-keeping and are skipped by `buildLlmMessages`. `assistant.text` may be an empty string when a step produced only tool calls; the render filter skips such events when searching for the last assistant text of a turn.

#### `turn_id`

Every event produced by a single agent invocation shares one `turn_id`. A user message is its own turn; the assistant invocation that responds (potentially producing multiple text + tool_call + tool events through the multi-step loop) shares a different `turn_id`.

`turn_id` exists for **rendering**: each channel decides what to show per turn (default: only the last assistant text of the turn — see channel recipes for specifics). It is NOT used by the LLM context builder, which preserves the full event sequence regardless.

#### LLM context (replay)

When the agent runs, the router reads the JSONL and reconstructs a `CoreMessage[]` to pass to `generateText`. **Each event becomes one CoreMessage in original order** — so the model sees exactly the sequence it generated against, including tool calls and results from prior turns. This is the correctness reason tool calls are persisted: an assistant turn that ended with a tool call must be followed by the tool result before the next assistant message, or the model has no record of what it did.

**Send-time annotations on user messages.** During reconstruction, every user event's `text` is prefixed with a `[sent YYYY-MM-DD HH:MM:SS ZZZ]` marker derived from its `timestamp` field, formatted in the owner's local timezone. Get the timezone from `Intl.DateTimeFormat().resolvedOptions().timeZone` and format the zone as a short abbreviation (e.g. `PDT`, `EST`, `UTC`). This gives the model a "when" for every user message, and because the latest user message is at the end of the prompt, that message's timestamp also serves as the model's "now" for any duration calculations the user asks about ("how long since I said X?"). Assistant events are NOT prefixed — the model can infer its own timing from the bounding user messages, and the asymmetric format discourages the model from mimicking the prefix in its own replies.

#### Append cadence and writers

- **Append-only writes.** The router serializes per-conversation, so there's never a concurrent writer on the same file.
- Each event is appended as it happens (user message → user event; assistant step → assistant event; tool call → tool event). One agent invocation typically appends multiple events.
- **Reads** load the whole file and parse it line-by-line; for a personal assistant this is fine even across years of conversation.
- **Backup** is just copying the directory.

### Cron logs (`runtime/logs/cron/`)

One JSONL file per cron run, at `runtime/logs/cron/{jobname}/{ISO-timestamp}.jsonl`. Uses the [same event schema](#event-schema) as threads, with `cron_start` and `cron_end` audit events bracketing the run. For `history: none` jobs (dream, nap) the cron log is the primary audit trail; for `history: primary` jobs (heartbeat) the cron log is an additional record — the same agent/tool events also land in the user's thread. See [Scheduler](#4-scheduler) for which events go where.

No retention policy — kept forever. Small text, useful for the agent to reflect on past runs via `read_file`.

### Identity, behavior, memory, ephemeral

All plain files spread across the domains:

- **`config/SOUL.md`** — identity. The agent writes this on first run after asking the user for a name and personality. Read on every invocation.
- **`config/cron/`** — instance-specific scheduled jobs (markdown).
- **`config/skills/`** — instance-specific skills. Accepts both the spec's flat `{name}.md` form and the agentskills.io directory form (`{name}/SKILL.md` + optional `scripts/`).
- **Custom channels and tools** live in `agent/src/channels/` and `agent/src/tools/` alongside the built-in ones — `config/` holds no code.
- **`memory/`** — granular markdown files of long-term knowledge. See **Memory format** below.
- **`memory/journal/`** — daily scratch pad files (e.g., `memory/journal/2026-03-09.md`). The agent jots notes throughout the day; the `dream` cron promotes important items into the rest of `memory/`.
- **`memory/new/`** — inbox folder. The agent writes new notes here when there's no obvious folder yet (use `add_memory("new/{name}", content)`). The `dream` cron sorts the inbox into appropriate folders. Prefer confident placement (`add_memory` directly into the right folder) when you know where it belongs; use the inbox only when genuinely unsure.
- **`memory/archive/`** — cold storage for content `dream` decided isn't currently useful but might be worth keeping. Exempt from the orphan check — things here don't need to be reachable from a root file. Use `rename_memory("{name}", "archive/{name}")` to move something into the archive.
- **`runtime/`** — `threads/` (conversation JSONLs + per-thread consolidation cursor sidecars), `logs/cron/{jobname}/{ISO}.jsonl` (one per cron run), `logs/cron/{jobname}/{ISO}/{call_id}.jsonl` (sub-agent logs), `logs/sub-agents/` (thread-originated sub-agent logs — see [Sub-agents](#sub-agents)), `*.pid`, `memory-graph.json` (backlink cache). See **Memory consolidation** below for cursor details.

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

**Resolution preservation on changes.** When a file is created, renamed, or moved, bare-name links (`[[coffee]]`) can silently change which file they resolve to — another file with the same basename might now win the shortest-path tiebreak, or the shadowed file might get un-shadowed. The `add_memory` and `rename_memory` tools handle this by snapshotting the full resolution map before the change and rewriting any reference whose resolved target would change after the change. Raw `write_file(mode: "create")` under `memory/` does NOT run this preservation step — prefer `add_memory` for any memory-file creation that could introduce name shadowing.

### Backlinks

Backlinks are not stored. They're computed by scanning all `memory/**/*.md` for `[[...]]` references and building a reverse index. The graph is cached at `runtime/memory-graph.json` and rebuilt when memory files change (mtime check at startup).

The `find_memory` tool returns a file's path and its backlinks ("files that reference this one") on hit — see the exact return shape under [Tool return shapes](#tool-return-shapes). This lets the agent traverse the graph naturally.

### Loading into context

Memory is **not loaded eagerly** into the system prompt. Instead, the system prompt includes a **manifest of root-level files only** — every `memory/*.md` file (not recursing into subfolders), with its name, aliases, tags, and a one-line summary. The summary comes from (in order of preference):

1. The frontmatter `description:` field, if present
2. The first H1 heading in the body
3. The first ~100 characters of body text

The manifest gives the agent enough context to know what high-level topics exist. The agent uses `find_memory` and `read_file` to fetch full content on demand, based on conversation context.

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

**Consolidation cursors.** A per-thread cursor records how many messages have been consolidated into memory. Stored as a sidecar file next to the thread JSONL: `runtime/threads/{channelId}/{conversationId}.cursor` containing a single integer (the count of consolidated messages from the start of the file). Both `nap` and `dream` read and update these cursors so neither re-consolidates content the other has already processed. Cursors live in `runtime/` intentionally — they share the lifecycle of the threads they describe. Wiping `runtime/` (the documented "rebuild from scratch" action) drops both the threads and their cursors together; there's nothing left to consolidate, and nothing gets re-consolidated by accident.

**Thread context at runtime is unchanged.** The agent always sees the most recent N messages of the active thread (capped at the token budget). Consolidated older content reaches the agent through the memory manifest, not through any thread-content rewriting. The thread is "what's happening now"; memory is "what I know."

## First-run flow

On startup, the agent checks for `config/SOUL.md`. If it doesn't exist:

1. The agent introduces itself as a blank Protos and asks the user for a name and personality.
2. The agent writes `config/SOUL.md` with the chosen identity (using `write_file` with `mode: "create"`).
3. Subsequent startups read the populated file.

The agent also creates `config/`, `memory/`, and `runtime/` directories on first run if they don't exist.

**Cron firings are suppressed while bootstrap is incomplete.** If a cron job's scheduled time arrives before `config/SOUL.md` exists, the scheduler skips that firing — otherwise the agent would respond with the bootstrap intro ("what should I be called?") to a heartbeat or dream prompt, polluting the cron log and confusing the user. Self-healing: the next firing after `config/SOUL.md` lands works normally.

## Startup flow

1. Ensure `runtime/` directory exists (`runtime/threads/` is created lazily on first message)
2. Discover skills (scan `spec/skills/*.md`; scan `config/skills/` for both `*.md` and `*/SKILL.md`; load names and descriptions; on name collision, merge — config frontmatter wins, config body appended to spec body)
3. Discover tools (scan `agent/src/tools/`)
4. Register channels (scan `agent/src/channels/`; each checks for credentials and connects if present). If no channels connect, exit with an error.
5. Start the scheduler (scan `spec/cron/` and `config/cron/`; merge by filename per the layering rules above)
6. If `config/SOUL.md` doesn't exist, run first-run flow
7. Begin processing incoming messages

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
  tools/              # tool recipes (markdown only)
    read_file.md
    write_file.md
    edit_file.md
    find_memory.md
    add_memory.md          # memory-aware create + resolution preservation
    rename_memory.md       # memory-aware rename + resolution preservation
    remember.md
    shell.md
    delegate_task.md       # sub-agent runner
    web_fetch.md           # privacy-aware web fetch
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
    tools/            # built-in + custom tools
      read_file.ts
      write_file.ts
      edit_file.ts
      find_memory.ts
      add_memory.ts
      rename_memory.ts
      remember.ts
      shell.ts
      delegate_task.ts
      web_fetch.ts
      list_threads.ts
      read_thread_tail.ts
      advance_thread_cursor.ts
      find_orphans.ts
    paths.ts          # shared path resolution + write guards
    agents/           # sub-agent runner (single generic file, no per-agent definitions)
      runner.ts

# Behavior — gitignored, optionally a separate repo. Markdown only — no code.
config/
  SOUL.md             # written on first run
  .env                # secrets
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
  ...                 # one subdir per vendored tool

# Ephemeral — gitignored, never a repo
runtime/
  threads/            # one JSONL file per conversation
    telegram/
      12345.jsonl
    terminal/
      cli.jsonl       # persistent CLI chat thread
  clients/            # per-client cursor files
    chat.cursor       # default terminal client
  logs/
  agent.sock          # Unix socket for terminal channel
  *.pid
  memory-graph.json   # backlink cache
```

## Capability layout

Channels and tools are **code** — they live in `agent/src/`, which is the user's own implementation repo. Their recipes live in `spec/`. Skills and cron are **behavior configuration** — markdown-only, layered between `spec/` (defaults) and `config/` (user overrides).

| | Recipes | Code | Layered? |
|---|---|---|---|
| **Channels** | `spec/channels/{name}.md` | `agent/src/channels/{name}.ts` | No — single code root |
| **Tools** | `spec/tools/{name}.md` | `agent/src/tools/{name}.ts` | No — single code root |
| **Skills** | `spec/skills/{name}.md` (built-in) + `config/skills/{name}.md` *or* `config/skills/{name}/SKILL.md` (user) | none | Yes — two-root, frontmatter override + body append |
| **Cron** | `spec/cron/{name}.md` (default) + `config/cron/{name}.md` (override) | none | Yes — two-root, frontmatter override + body append |

The asymmetry is intentional:

- **Code lives in `agent/` because `agent/` is the user's repo.** Built-in channels (generated from spec recipes during bootstrap) and custom channels (added by the user) live in the same directory. There's no need for a separate "user extension" location because the user already owns `agent/`.
- **Skills and cron live partly in `spec/` because they're behavior the spec ships with defaults for** (e.g., the heartbeat job, the `self-edit` skill). A user can override a default by dropping a same-named file in `config/`. No code means no node_modules or compilation — pure markdown layering.

Rules:

- **Filename = capability name.** No central registry. The loader scans the directory for `*.ts` files (channels, tools) or `*.md` files (cron, skills, with `config/skills/` additionally accepting `{name}/SKILL.md` for off-the-shelf agentskills.io drops) and registers each one.
- **Recipes describe; implementations execute.** A recipe in `spec/channels/telegram.md` tells the bootstrap how to build `agent/src/channels/telegram.ts`. Recipes never name the implementation path explicitly — the path is determined by their own location.
- **Custom channels don't need spec recipes.** A user adding a channel directly to `agent/src/channels/` can colocate the optional `.md` alongside it, or skip the recipe entirely.

### Adding a channel

A channel's `.ts` file exports a `register` function that takes the router and returns the channel's ID, owner conversation ID, and send function if it connected — or nothing if credentials were missing.

`register()`:

1. Checks if its credentials exist (e.g., `TELEGRAM_BOT_TOKEN`)
2. If not, returns nothing
3. If yes, connects, starts forwarding owner messages to the router (ignoring all others), and returns

To add a channel: write `agent/src/channels/{name}.ts`. If it's reusable, also write a recipe at `spec/channels/{name}.md` so future users can bootstrap it.

### Applying spec updates

**Re-bootstrap is not idempotent.** Bootstrap is a one-time operation that initializes `agent/`. When the spec updates and you want the changes in your existing `agent/`, point a coding agent at the updated spec and ask it to apply the delta: "Here's my current `agent/`, here's the updated `spec/`. Update the code to match the new spec, preserving my custom additions."

This keeps the bootstrap simple (no manifest tracking, no merge logic) and makes spec updates explicit — you see the diff, you approve the changes.

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
cd agent && git init && git add -A && git commit -m "bootstrap"
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

- The agent runs with shell access and the same permissions as the host user. Consider running it on a dedicated machine or in a container.
- The **shell tool** runs commands using bash from the workspace root with a 1 MB output limit. The agent should confirm destructive commands with the user before running them.
- **Never log credentials.** When catching errors, log only the error message — not full objects, which may contain API tokens or bot secrets.
- Secrets live in `config/.env`.
- The thread files in `runtime/threads/` contain all your messages — protect that directory accordingly.

## Non-goals

These are intentionally left out. They may be added later.

- **Multi-user conversations** — all messages are assumed to be from the owner
- **Multiple AI providers at once** — one provider per deployment
- **Web UI or dashboard** — the messaging app is the interface
- **Plugin system** — channels, tools, and skills are just files, not a formal plugin API
- **Submodules** — `agent/`, `config/`, and `memory/` are independent repos when promoted, never submodules
