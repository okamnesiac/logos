# Architecture

## Overview

Logos is a single-process personal AI assistant. Messages come in from messaging channels, get processed by an AI agent, and responses go back out through the same channels.

The workspace is organized into four sibling domains, each with a distinct role and lifecycle:

```
logos/                # workspace root
  agent/              # the engine — code, default capabilities, engine docs
  config/             # behavior — identity, instance-specific tools/skills/cron
  memory/             # durable state — facts, preferences, journal
  runtime/            # ephemeral state — db, logs, pid files
```

Only the agent repo lives here. `config/`, `memory/`, and `runtime/` are sibling directories — gitignored by this repo, optionally backed by their own repos, never tracked here.

## The four domains

### `agent/` — the engine

Everything the engine ships: source code, default capabilities (channels, tools, skills, cron jobs), engine documentation. Shared across all Logos users — what's in this directory is what every instance gets out of the box.

Read-only at runtime from the agent's perspective (the agent edits its own source via the `self-edit` skill, but treats `agent/` as engine code, not personal state).

### `config/` — behavior

How this specific instance behaves: identity (`SOUL.md`), instance-specific tools/skills/channels, instance-specific cron jobs, environment variables. Per-user. Optionally a separate Git repo.

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

**Bundled vs. instance-specific:** Channel implementations ship in `agent/channels/` (one `.ts` file per channel, plus a colocated `.md` recipe). Users can add their own channels in `config/channels/`; the loader scans both directories at startup.

### 2. Router

The router sits between channels and the agent. It:

- Receives normalized messages from channels
- Maintains a per-conversation message queue (one agent invocation at a time per conversation)
- Passes messages to the agent with conversation context
- Routes agent responses back to the originating channel

If the agent's response is the exact string `NO_REPLY`, the router discards it. This lets cron jobs and heartbeats run without generating a message when there's nothing to report.

### 3. Agent

The agent is the brain. It uses the Vercel AI SDK to receive a message + history, decide how to respond, optionally use tools, and return a response.

**Tools** are typed capabilities defined in code. The kernel ships four primitives in `agent/tools/`: `read_file`, `remember`, `recall`, `shell`. Users can add their own in `config/tools/`; both directories are loaded at startup. The AI SDK handles tool execution loops natively — limit the number of steps to prevent runaway tool use.

**Skills** are markdown instruction files that teach the agent how to accomplish complex tasks using its tools. Bundled skills live in `agent/skills/`, instance-specific skills in `config/skills/`. Both directories are scanned at startup; on name collision, `config/` wins.

**Identity** comes from `config/SOUL.md`, written on first run. The agent reads it on every invocation.

**Long-term memory** comes from the `memory/` directory. The agent reads granular files (facts, preferences, summaries) into context as needed.

**Model-agnostic:** The default provider is Anthropic (Claude), but switching to OpenAI, Google, or any other provider is a one-line change.

### 4. Scheduler

The scheduler runs cron jobs from two roots: `agent/cron/` (defaults shipped with the engine) and `config/cron/` (instance-specific jobs).

**One file per job.** Each job is a single `.md` file. The filename (minus extension) is the job name. Frontmatter holds the schedule and any options. The body holds the prompt sent to the agent when the job fires.

```markdown
---
schedule: "*/30 * * * *"
---

Check for unread messages that need follow-up. NO_REPLY if nothing to report.
```

**Layering.** When a file with the same name appears in both roots, the merged job uses:

- **Frontmatter:** config overrides agent (e.g., `schedule:` in config replaces agent's).
- **Body:** agent first, then config appended. Lets you extend a default prompt without restating it.

To disable an agent default, drop a config file with the same name and `enabled: false` in frontmatter.

There is no central registry. Adding a job means dropping a file. The merged view (with source annotations) is available via `agent/logos cron list`.

When a job fires, the scheduler looks up the primary channel and sends the merged prompt to the agent through the router as a synthetic message addressed to the owner's main conversation. A reminder is appended: "If you have nothing to say to the owner, respond with NO_REPLY."

### 5. Self-modification

The agent can edit its own source code via its shell tool. TypeScript runs directly with `tsx` — no build step. To apply code changes, the agent restarts itself using the `agent/logos restart` wrapper script.

The wrapper type-checks the code (`tsc --noEmit`) before restarting. If the check fails, the restart is aborted and the old process keeps running. This prevents the agent from killing itself with a bad edit.

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

All plain files spread across the four domains:

- **`config/SOUL.md`** — identity. The agent writes this on first run after asking the user for a name and personality. Read on every invocation.
- **`config/cron/`** — instance-specific scheduled jobs.
- **`config/tools/`, `config/skills/`, `config/channels/`** — instance-specific capability extensions.
- **`memory/`** — granular markdown files of long-term knowledge. See **Memory format** below for conventions.
- **`memory/journal/`** — daily scratch pad files (e.g., `memory/journal/2026-03-09.md`). The agent jots notes throughout the day; the consolidate-memories job promotes important items into the rest of `memory/`.
- **`memory/new/`** — landing folder for files lazily created by `[[wiki-link]]` references. The consolidate-memories job sorts these into appropriate folders.
- **`runtime/`** — `threads/`, `logs/`, `*.pid`, `memory-graph.json` (backlink cache).

## Memory format

Memory files are standard markdown with optional YAML frontmatter. The format is **Obsidian-compatible** — you can open `memory/` as an Obsidian vault and get a free GUI for browsing and editing.

### File anatomy

```markdown
---
created: 2026-04-18T10:30:00Z
updated: 2026-04-18T11:00:00Z
tags: [coffee, preferences]
aliases: [coffee-order, my-coffee]
---

I like my coffee black, no sugar. Usually from [[people/justin]]'s
favorite place, [[blue-bottle]].

See also: ![[preferences/morning-routine]]
```

- **Frontmatter** holds intrinsic metadata about the note. Standard keys (`tags`, `aliases`, `created`, `updated`) match Obsidian conventions. Engine-specific fields are namespaced — e.g., `logos.confidence`, `logos.source` — to avoid collision with Obsidian or its plugins.
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
4. If no match: **lazy-create** at `memory/new/{name}.md` with empty frontmatter and body

### Backlinks

Backlinks are not stored. They're computed by scanning all `memory/**/*.md` for `[[...]]` references and building a reverse index. The graph is cached at `runtime/memory-graph.json` and rebuilt when memory files change (mtime check at startup).

The `recall` tool returns both a file's content and its backlinks ("files that reference this one"), letting the agent traverse the graph naturally.

### Obsidian compatibility

If `memory/` is opened as an Obsidian vault, Obsidian creates a `.obsidian/` directory for vault settings (workspace layout, plugin config). This directory is machine-local and should be in `memory/.gitignore`. Configure Obsidian's "Default location for new notes" → `new/` to match the lazy-create destination.

## First-run flow

On startup, the agent checks for `config/SOUL.md`. If it doesn't exist:

1. The agent introduces itself as a blank Logos and asks the user for a name and personality.
2. The agent writes `config/SOUL.md` with the chosen identity.
3. Subsequent startups read the populated file.

The agent also creates `config/`, `memory/`, and `runtime/` directories on first run if they don't exist.

## Startup flow

1. Ensure `runtime/` directory exists (`runtime/threads/` is created lazily on first message)
2. Discover skills (scan `agent/skills/` and `config/skills/` for `SKILL.md` files; load names and descriptions; config overrides agent on name collision)
3. Discover tools (scan `agent/tools/` and `config/tools/`; same merge rules)
4. Register channels (scan `agent/channels/` and `config/channels/`; each checks for credentials and connects if present). If no channels connect, exit with an error.
5. Start the scheduler (scan `agent/cron/` and `config/cron/`; merge by filename per the layering rules above)
6. If `config/SOUL.md` doesn't exist, run first-run flow
7. Begin processing incoming messages

## File structure

```
# Workspace root — only these are tracked by the agent repo
README.md
CLAUDE.md
AGENTS.md
.gitignore
agent/

# Engine — tracked
agent/
  README.md           # optional engine README
  ARCHITECTURE.md     # this file
  BUILD.md            # build instructions for coding agents
  package.json
  tsconfig.json
  logos               # wrapper script (start/stop/restart/status)
  src/
    index.ts          # entry point
    router.ts
    agent.ts
    db.ts
    scheduler.ts
    channels/
      registry.ts
      telegram.ts     # one .ts per channel
      ...
    tools/
      read_file.ts
      remember.ts
      recall.ts
      shell.ts
  channels/           # channel recipes (colocate with src/channels/ when implementations land)
    telegram.md
    whatsapp.md
    ...
  voice/              # voice capability recipes
    voice-input.md
    voice-output.md
  skills/             # bundled skills
    self-edit/
      SKILL.md
    git/
      SKILL.md
    coding/
      SKILL.md
  cron/               # default cron jobs
    heartbeat.md
    consolidate-memories.md

# Behavior — gitignored, optionally a separate repo
config/
  SOUL.md             # written on first run
  .env                # secrets
  cron/               # instance-specific or override jobs
  skills/             # instance-specific skills
  tools/              # instance-specific tools
  channels/           # instance-specific channels

# Durable state — gitignored, optionally a separate repo
memory/
  .obsidian/          # vault settings (machine-local, gitignored)
  facts/
  people/
  preferences/
  summaries/
  journal/
    2026-03-10.md
  new/                # lazy-created files (sorted by consolidate-memories)

# Ephemeral — gitignored, never a repo
runtime/
  threads/            # one JSONL file per conversation
    telegram/
      12345.jsonl
  logs/
  *.pid
```

## Adding a channel

A channel is a single `.ts` file plus a colocated `.md` recipe. The `.ts` file exports a `register` function that takes the router and returns the channel's ID, owner conversation ID, and send function if it connected — or nothing if credentials were missing.

`register()`:

1. Checks if its credentials exist (e.g., `TELEGRAM_BOT_TOKEN`)
2. If not, returns nothing
3. If yes, connects, starts forwarding owner messages to the router (ignoring all others), and returns

The channel registry scans `agent/channels/` and `config/channels/` at startup for `*.ts` files and registers each one. No central registry edit needed when adding a channel.

The colocated `.md` recipe documents the library to use, environment variables, setup steps, and any gotchas.

## Permissions model

| Domain | Agent (running) | Coding agents (Claude/Codex) | Human |
|--------|-----------------|------------------------------|-------|
| `agent/` | read + self-edit via skill | read/write | read/write |
| `config/` | read/write | read/write | read/write |
| `memory/` | read/write | read/write | read/write |
| `runtime/` | read/write | read-only | read/write |

Read `runtime/` to debug. Modify the source domains (`agent/`, `config/`, `memory/`) to fix.

## Multi-machine model

The same `memory/` repo can be shared across multiple machines running different `agent/` versions and different `config/` layers. Read access can be shared freely; write authority should be explicit. Typical model: one primary writer, or multiple writers resolving via Git.

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
- **Submodules** — `config/` and `memory/` are independent repos when promoted, never submodules
