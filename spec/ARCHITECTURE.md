# Architecture

## Overview

Logos is a single-process personal AI assistant. Messages come in from messaging channels, get processed by an AI agent, and responses go back out through the same channels.

The workspace is organized into **five sibling domains**, each with a distinct role and lifecycle:

```
logos/                # workspace root
  spec/               # the blueprint — ARCHITECTURE, BUILD, recipes, defaults
  agent/              # generated implementation — code that the bootstrap produces
  config/             # behavior — identity, instance overrides, .env
  memory/             # durable state — facts, preferences, journal
  runtime/            # ephemeral state — message threads, logs, pid files
```

Only `spec/` (and the workspace entry-point docs) is tracked by this repo. `agent/`, `config/`, and `memory/` are sibling directories — gitignored here and **strongly recommended** to be their own Git repos (see [Git repos per domain](#git-repos-per-domain) below). `runtime/` is local-only.

## The five domains

### `spec/` — the blueprint

The architecture documents (`ARCHITECTURE.md`, `BUILD.md`), channel recipes, bundled skills, and default cron jobs. Tracked by this repo. Shared across all Logos users — what's in `spec/` is what every instance gets out of the box.

The running agent reads from `spec/` directly for skills and cron defaults. Spec updates take effect on the next agent restart — no copy step, no drift.

### `agent/` — generated implementation

The TypeScript code produced by the bootstrap: `index.ts`, `router.ts`, the channel implementation(s) the user chose, the built-in tool implementations, the wrapper script. Gitignored by this repo. Optionally a separate Git repo if you want to commit your specific implementation.

The agent edits its own source under `agent/src/` via the `self-edit` skill.

### `config/` — behavior

How this specific instance behaves: identity (`SOUL.md`), instance-specific overrides for tools/skills/channels/cron, environment variables. Per-user. Optionally a separate Git repo.

Behavior reads context, but context does not define behavior.

### `memory/` — durable state

What the agent knows and is committed to: facts, preferences, summaries, journal entries, durable task state. Per-user, private. Optionally a separate Git repo.

Granular files, not one consolidated file — Git history is more useful when changes are line-item rather than whole-file.

### `runtime/` — ephemeral state

What is happening right now: message threads, logs, pid files, locks, caches. Machine-local, never versioned, safe to delete and rebuild.

Practical rule: if it works well with Git, it belongs in `memory/`. If it doesn't merge well, it belongs in `runtime/`.

## Tech stack

| Component | Choice |
|-----------|--------|
| Runtime | Node.js 18+ |
| Language | TypeScript |
| Storage | Plain files — no database |
| AI | Vercel AI SDK (`ai`) with `@ai-sdk/anthropic` (default provider) |
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

#### Cursor-based replay

Because thread history is stored as append-only JSONL files, any channel with disconnecting clients can support replay with no server-side state. The pattern:

1. Client stores its last-seen line index locally.
2. On reconnect, client sends `{"from": N}`.
3. Server reads lines `N..end` from `runtime/threads/{channelId}/{conversationId}.jsonl` and streams them as replay.
4. Server watches the JSONL file (`fs.watch`) for new appends and streams those live.
5. Client updates its cursor as messages arrive.

This gives any client-server channel "pick up where I left off" semantics across reconnects — messages queued while no client was connected (e.g., cron firing at night) are delivered on next connect. See the terminal channel for the reference implementation.

### 2. Router

The router sits between channels and the agent. It:

- Receives normalized messages from channels
- Maintains a per-conversation message queue (one agent invocation at a time per conversation)
- Passes messages to the agent with conversation context
- Routes agent responses back to the originating channel

If the agent's response is the exact string `NO_REPLY`, the router discards it. This lets cron jobs and heartbeats run without generating a message when there's nothing to report.

### 3. Agent

The agent is the brain. It uses the Vercel AI SDK to receive a message + history, decide how to respond, optionally use tools, and return a response.

**Tools** are typed capabilities defined in code. Each built-in tool has a recipe in `spec/tools/{name}.md` describing its inputs, outputs, behavior, and dependencies — same pattern as channels. Implementations live in `agent/src/tools/`.

The kernel ships eight tools:

- `read_file(path)` — read any file in the workspace
- `write_file(path, content, mode)` — `create` (fail if exists), `append`, or `replace` (overwrite)
- `edit_file(path, old_string, new_string)` — surgical find-and-replace; `old_string` must appear exactly once
- `find_memory(name)` — resolve a wiki-link-style name to a path. Returns `{ found: true, path, backlinks }` or `{ found: false }` (see [Tool return shapes](#tool-return-shapes)). Does NOT lazy-create.
- `remember(text)` — sugar for appending to today's journal
- `shell(cmd)` — run a shell command (1 MB output limit)
- `delegate_task({ prompt, skills, tools, model? })` — spawn a sub-agent in an isolated context (see [Sub-agents](#sub-agents))
- `web_fetch(url)` — fetch a URL and return content as markdown (see `spec/tools/web_fetch.md` for backend choice and privacy)

Users can add their own tools directly in `agent/src/tools/` — same location as the built-in tools. The AI SDK handles tool execution loops natively — limit the number of steps to prevent runaway tool use.

#### Tool return shapes

Tool return values get JSON-serialized and shown to the model. Two conventions:

- **Tools that succeed-or-throw** (`read_file`, `write_file`, `edit_file`, `remember`, `shell`) return their result on success and throw on failure. The AI SDK reports the throw to the model as an error.
- **Tools that succeed-or-miss** (anything that does lookup, including `find_memory`) return a **discriminated union** with a boolean tag — never a bare `null`. The tag makes the shape unambiguous to the model and leaves room to add diagnostic fields later (e.g., suggestions for fuzzy matches).

`find_memory` specifically:

```ts
type FindMemoryResult =
  | { found: true; path: string; backlinks: string[] }
  | { found: false };
```

Do not return `null`, `undefined`, or `{ path: null }`. The shape above is the contract.

**Skills** are markdown instruction files that teach the agent how to accomplish complex tasks using its tools. Bundled skills live in `spec/skills/`, instance-specific skills in `config/skills/`. Both directories are scanned at startup; on name collision, `config/` wins.

**Identity** comes from `config/SOUL.md`, written on first run. The agent reads it on every invocation.

**Long-term memory** comes from the `memory/` directory. The system prompt includes a manifest (name + summary) for every memory file; full content is fetched on demand via `find_memory` and `read_file`. See **Memory format** below.

**Self-awareness.** The system prompt includes a fixed "How you operate" block that explains the agent's two invocation modes (user messages vs. scheduled cron jobs) and a list of active cron jobs (name + schedule + summary). Without this, the agent doesn't know cron exists and may tell the user it can't reach out proactively — which is wrong; cron heartbeats are exactly the proactive-outreach mechanism. See `BUILD.md` step 4 for the exact wording.

**Model-agnostic:** The default provider is Anthropic (Claude), but switching to OpenAI, Google, or any other provider is a one-line change.

#### Sub-agents

The agent can spawn focused sub-agents via the `delegate_task` tool. A sub-agent is a separate `generateText` invocation with:

- A task-specific prompt (written by the main agent at call time)
- An explicit list of skills (full SKILL.md bodies inlined into its system prompt)
- An explicit allowlist of tools (a subset of what the main agent has)
- An isolated context — no SOUL.md, no memory manifest, no conversation history

Sub-agents are not pre-defined as separate entities. There is no `spec/agents/` directory. The skills system already provides reusable behavior templates; `delegate_task` just lets the main agent compose them on demand into a focused sub-task. To make a sub-agent specialty reusable, write a regular skill in `spec/skills/{name}/`.

**When to delegate:**

- Tasks that require many tool calls or large intermediate outputs (web research, multi-file analysis)
- Sub-tasks that benefit from a fresh focused context
- Anything where intermediate data shouldn't pollute the main context

**Constraints:**

- Sub-agents cannot spawn sub-agents. The runner strips `delegate_task` from the tool allowlist regardless of what the caller asks for. Single-level delegation, period.
- The runner logs each invocation (skills, tools, model, duration, tokens) to `runtime/logs/` for operator visibility. Cost info is NOT surfaced to the calling agent.

See `spec/tools/delegate_task.md` for the full tool contract and BUILD.md → step 4d for the runner implementation.

### 4. Scheduler

The scheduler runs cron jobs from two roots: `spec/cron/` (defaults shipped with the spec) and `config/cron/` (instance-specific jobs).

**One file per job.** Each job is a single `.md` file. The filename (minus extension) is the job name. Frontmatter holds the schedule and any options. The body holds the prompt sent to the agent when the job fires.

```markdown
---
schedule: "*/30 * * * *"
---

Check for unread messages that need follow-up. NO_REPLY if nothing to report.
```

**Layering.** When a file with the same name appears in both roots, the merged job uses:

- **Frontmatter:** config overrides spec (e.g., `schedule:` in config replaces spec's).
- **Body:** spec first, then config appended. Lets you extend a default prompt without restating it.

To disable a spec default, drop a config file with the same name and `enabled: false` in frontmatter.

There is no central registry. Adding a job means dropping a file. The merged view (with source annotations) is available via `agent/logos cron`.

When a job fires, the scheduler looks up the primary channel and sends the merged prompt to the agent through the router as a synthetic message addressed to the owner's main conversation. A reminder is appended: "If you have nothing to say to the owner, respond with NO_REPLY."

### 5. Self-modification

The agent can edit its own source code (`agent/src/`) via its file-edit tools (`write_file`, `edit_file`). TypeScript runs directly with `tsx` — no build step. To apply code changes, the agent restarts itself using the `agent/logos restart` wrapper script.

**Safe-edit protocol:**

1. The agent commits the change with a clear message using the `git` skill (requires `agent/` to be a Git repo — see [Git repos per domain](#git-repos-per-domain)).
2. The wrapper type-checks the code (`tsc --noEmit`) before restarting. If the check fails, the restart is aborted and the old process keeps running.
3. The wrapper starts the new process, then waits a few seconds and confirms it's still alive. If the process crashed at runtime (e.g. bad import, missing env var, startup exception), the wrapper **automatically reverts the last commit in `agent/`** and restarts with the pre-edit code.

This closes the three failure modes for self-edit: compile errors (caught by typecheck), runtime startup errors (caught by post-start health check + auto-revert), and subtle logic bugs (can be manually reverted via `git` from the running agent).

**Disabling self-edit:** set `LOGOS_SELF_EDIT=false` in `config/.env`. When disabled:

- The `self-edit` skill is hidden from the agent (skills loader skips it).
- `write_file` and `edit_file` reject any path that resolves under `agent/` with a clear error.
- The `shell` tool gets a warning appended to its description: "self-edit is disabled; do not modify files under `agent/`." This is a nudge, not enforcement — the shell tool can still technically write anywhere the process user can.

This is **safety by convention plus tool guards** — enough to prevent accidental or unprompted self-edit. For guaranteed enforcement (e.g. against an adversarial or confused agent), run the process in a sandbox with `agent/` mounted read-only. See `BUILD.md` → Sandboxing for OS-specific notes.

Default is `LOGOS_SELF_EDIT=true`. The capability exists and is well-guarded; users nervous about it flip one env var.

`spec/` is not edited by the running agent. Spec changes are made by humans (or coding agents like Claude Code) and applied by asking a coding agent to update `agent/` to match.

## Storage

Everything is plain files — no database. Files break down by domain:

### Message history (`runtime/threads/`)

One JSONL file per conversation, at `runtime/threads/{channelId}/{conversationId}.jsonl`. Each line is one message:

```jsonl
{"role":"user","text":"hi","timestamp":"2026-04-18T10:30:00.000Z"}
{"role":"assistant","text":"hello!","timestamp":"2026-04-18T10:30:02.000Z"}
```

- **Append-only writes.** The router serializes per-conversation, so there's never a concurrent writer on the same file.
- **Reads** load the whole file and parse it; for a personal assistant this is fine even across years of conversation.
- **Backup** is just copying the directory.

### Identity, behavior, memory, ephemeral

All plain files spread across the domains:

- **`config/SOUL.md`** — identity. The agent writes this on first run after asking the user for a name and personality. Read on every invocation.
- **`config/cron/`** — instance-specific scheduled jobs (markdown).
- **`config/skills/`** — instance-specific skills (markdown, agentskills.io directory format).
- **Custom channels and tools** live in `agent/src/channels/` and `agent/src/tools/` alongside the built-in ones — `config/` holds no code.
- **`memory/`** — granular markdown files of long-term knowledge. See **Memory format** below.
- **`memory/journal/`** — daily scratch pad files (e.g., `memory/journal/2026-03-09.md`). The agent jots notes throughout the day; the consolidate-memories job promotes important items into the rest of `memory/`.
- **`memory/new/`** — inbox folder. The agent writes new notes here when there's no obvious folder yet (use `write_file` with `mode: "create"`). The consolidate-memories job sorts the inbox into appropriate folders.
- **`runtime/`** — `threads/`, `logs/`, `*.pid`, `memory-graph.json` (backlink cache).

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

- **Frontmatter** holds intrinsic metadata about the note. Standard keys (`tags`, `aliases`, `created`, `updated`, `description`) match Obsidian conventions. Engine-specific fields are namespaced — e.g., `logos.confidence`, `logos.source` — to avoid collision with Obsidian or its plugins.
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
4. If no match: return null (no auto-creation). The agent decides whether to create a file via `write_file` and where to put it. This matches Obsidian's spirit — Obsidian only creates a target file when the user *clicks* a missing link, not when one is parsed.

### Backlinks

Backlinks are not stored. They're computed by scanning all `memory/**/*.md` for `[[...]]` references and building a reverse index. The graph is cached at `runtime/memory-graph.json` and rebuilt when memory files change (mtime check at startup).

The `find_memory` tool returns a file's path and its backlinks ("files that reference this one") on hit — see the exact return shape under [Tool return shapes](#tool-return-shapes). This lets the agent traverse the graph naturally.

### Loading into context

Memory is **not loaded eagerly** into the system prompt. Instead, the system prompt includes a **manifest**: a flat list of every memory file with its name, aliases, tags, and a one-line summary. The summary comes from (in order of preference):

1. The frontmatter `description:` field, if present
2. The first H1 heading in the body
3. The first ~100 characters of body text

The manifest gives the agent enough context to know what's available. The agent uses `find_memory` and `read_file` to fetch full content on demand, based on conversation context.

**Journal exception:** the last 24 hours of `memory/journal/` entries are included in the system prompt directly, since they're recent agent-authored notes likely to be relevant. Older journal entries appear in the manifest but not inline.

### Obsidian compatibility

If `memory/` is opened as an Obsidian vault, Obsidian creates a `.obsidian/` directory for vault settings (workspace layout, plugin config). This directory is machine-local and should be in `memory/.gitignore`. Configure Obsidian's "Default location for new notes" → `new/` so notes you create in Obsidian land in the same inbox the agent uses.

## First-run flow

On startup, the agent checks for `config/SOUL.md`. If it doesn't exist:

1. The agent introduces itself as a blank Logos and asks the user for a name and personality.
2. The agent writes `config/SOUL.md` with the chosen identity (using `write_file` with `mode: "create"`).
3. Subsequent startups read the populated file.

The agent also creates `config/`, `memory/`, and `runtime/` directories on first run if they don't exist.

## Startup flow

1. Ensure `runtime/` directory exists (`runtime/threads/` is created lazily on first message)
2. Discover skills (scan `spec/skills/` and `config/skills/` for `SKILL.md` files; load names and descriptions; config overrides spec on name collision)
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
  ARCHITECTURE.md     # this file
  BUILD.md            # build instructions for coding agents
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
    remember.md
    shell.md
    delegate_task.md  # sub-agent runner
    web_fetch.md      # privacy-aware web fetch
  skills/             # bundled skills (agentskills.io directory format)
    self-edit/
      SKILL.md
    git/
      SKILL.md
    coding/
      SKILL.md
  cron/               # default cron jobs (markdown with frontmatter)
    heartbeat.md
    consolidate-memories.md

# Generated implementation — gitignored, optionally a separate repo
agent/
  package.json
  tsconfig.json
  logos               # wrapper script (start/stop/restart/status/chat)
  src/
    index.ts          # entry point
    router.ts
    agent.ts
    scheduler.ts
    threads.ts
    memory.ts
    cli/              # isolated client code — does not import router/agent/memory
      chat.ts         # terminal client: connects to runtime/logos.sock
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
      remember.ts
      shell.ts
      delegate_task.ts
      web_fetch.ts
      _paths.ts       # shared path-safety + self-edit-guard helper
    agents/           # sub-agent runner (single generic file, no per-agent definitions)
      runner.ts

# Behavior — gitignored, optionally a separate repo. Markdown only — no code.
config/
  SOUL.md             # written on first run
  .env                # secrets
  cron/               # instance-specific or override jobs (markdown)
  skills/             # instance-specific skills (markdown)

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
  logos.sock          # Unix socket for terminal channel
  *.pid
  memory-graph.json   # backlink cache
```

## Capability layout

Channels and tools are **code** — they live in `agent/src/`, which is the user's own implementation repo. Their recipes live in `spec/`. Skills and cron are **behavior configuration** — markdown-only, layered between `spec/` (defaults) and `config/` (user overrides).

| | Recipes | Code | Layered? |
|---|---|---|---|
| **Channels** | `spec/channels/{name}.md` | `agent/src/channels/{name}.ts` | No — single code root |
| **Tools** | `spec/tools/{name}.md` | `agent/src/tools/{name}.ts` | No — single code root |
| **Skills** | `spec/skills/{name}/SKILL.md` (built-in) + `config/skills/{name}/SKILL.md` (user) | none | Yes — two-root, config wins |
| **Cron** | `spec/cron/{name}.md` (default) + `config/cron/{name}.md` (override) | none | Yes — two-root, frontmatter override + body append |

The asymmetry is intentional:

- **Code lives in `agent/` because `agent/` is the user's repo.** Built-in channels (generated from spec recipes during bootstrap) and custom channels (added by the user) live in the same directory. There's no need for a separate "user extension" location because the user already owns `agent/`.
- **Skills and cron live partly in `spec/` because they're behavior the spec ships with defaults for** (e.g., the heartbeat job, the `self-edit` skill). A user can override a default by dropping a same-named file in `config/`. No code means no node_modules or compilation — pure markdown layering.

Rules:

- **Filename = capability name.** No central registry. The loader scans the directory for `*.ts` files (channels, tools) or `*.md` files / `{name}/SKILL.md` (cron, skills) and registers each one.
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
| `runtime/` | read/write | read-only | read/write |

Read `runtime/` to debug. Modify the source domains (`spec/`, `agent/`, `config/`, `memory/`) to fix.

## Git repos per domain

Each domain has its own lifecycle, so each should have its own Git repo:

| Domain | Recommended? | Purpose of the repo |
|--------|--------------|---------------------|
| `spec/` | **Required** (this repo) | The canonical design, shared across all Logos users |
| `agent/` | **Strongly recommended** | History and rollback for self-edits; portable across machines |
| `config/` | **Recommended** | Sync behavior/identity across machines |
| `memory/` | **Recommended** (private) | Durable knowledge with line-item history |
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
- **Fail gracefully.** If a channel can't connect, log it and move on. Don't crash the process.
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
