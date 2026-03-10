# Architecture

## Overview

Logos is a single-process personal AI assistant. Messages come in from messaging channels, get processed by an AI agent, and responses go back out through the same channels.

```
┌──────────────────────────────────────────────────────┐
│                     Logos Process                    │
│                                                      │
│  ┌──────────┐   ┌──────────┐   ┌──────────────┐      │
│  │ Channels │──▶│  Router  │──▶│    Agent     │      │
│  │          │◀──│          │◀──│   (AI SDK)   │      │
│  └──────────┘   └──────────┘   └──────┬───────┘      │
│       │                               │              │
│       │     ┌───────────┐  ┌──────────┴───────────┐  │
│       │     │  data/    │  │     File System      │  │
│       │     └───────────┘  │  SOUL.md             │  │
│       │                    │  memory.md           │  │
│  ┌──────────┐              │  memories/           │  │
│  │Scheduler │              │  cron/               │  │
│                            │  skills/             │  │
│                            └──────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

## Tech stack

| Component | Choice |
|-----------|--------|
| Runtime | Node.js 22+ |
| Language | TypeScript |
| Database | SQLite (messages only) |
| AI | Vercel AI SDK (`ai`) with `@ai-sdk/anthropic` (default provider) |
| Hosting | Runs directly on the host — no containers |

## Components

### 1. Channels

Channels are messaging platform integrations. Each channel is responsible for:

- Connecting to a messaging platform (WhatsApp, Telegram, Discord, Slack, etc.)
- Receiving inbound messages and normalizing them into a common format
- Sending outbound messages from the agent back to the platform

**Owner-only:** Each channel knows the owner's ID on that platform (e.g., `TELEGRAM_OWNER_CHAT_ID`). Messages from anyone else are silently ignored. Multi-user support can be added later.

**Self-registering:** A channel only activates if its credentials are present in the environment. No credentials, no channel — no errors, no configuration needed.

**Main chat:** The `PRIMARY_CHANNEL` environment variable names the channel used for the owner's main conversation (e.g., `telegram`). The scheduler sends replies to the owner's conversation on this channel instead of logging them silently.

**Common message format:**

```
{
  channelId: string       // e.g. "whatsapp", "telegram"
  conversationId: string  // unique ID for the conversation
  text: string            // message content
  timestamp: Date
}
```

Each channel lives in its own file under `src/channels/`. Adding a new channel means editing the channel registry to import and register it.

### 2. Router

The router sits between channels and the agent. It:

- Receives normalized messages from channels
- Maintains a per-conversation message queue (one agent invocation at a time per conversation)
- Passes messages to the agent with conversation context
- Routes agent responses back to the originating channel

If the agent's response is the exact string `NO_REPLY`, the router discards it — nothing is stored or sent. This lets cron jobs and heartbeats run without generating a message when there's nothing to report.

The router is simple glue code. It does not make decisions — it just connects channels to the agent and manages concurrency.

### 3. Agent

The agent is the brain. It uses the Vercel AI SDK to:

- Receive a message and conversation history
- Decide how to respond, optionally using tools
- Return a response

**Tools** are typed capabilities defined in code: `remember`, `recall`, and `shell`. The AI SDK handles tool execution loops natively — limit the number of steps to prevent runaway tool use.

**Skills** teach the agent how to do more complex things using its tools. Skills are markdown files following the [Agent Skills](https://agentskills.io) open standard — see the Skills section below for details.

**Model-agnostic:** The default provider is Anthropic (Claude), but switching to OpenAI, Google, or any other provider is a one-line change.

### 4. Storage

Logos uses two storage mechanisms — SQLite for structured data and the filesystem for everything the user or agent should be able to inspect and edit directly.

#### SQLite (messages only)

SQLite stores message history:

- **messages** — channelId, conversationId, role (`user` or `assistant`), text, timestamp

`chat.sqlite` lives in `data/` at the project root. Back it up by copying it. Inspect it with any SQLite client.

#### Filesystem

Everything else lives in plain files:

**`SOUL.md`** — The agent's identity: name, personality, voice, values. The agent reads this first, every time it wakes — it reads itself into being. On first run, the agent asks the user for a name and personality, then writes this file. Edit it anytime to change who your assistant is.

**`memory.md`** — The agent's consolidated long-term memory. Key facts, user preferences, important context that persists across all conversations. The agent reads this at the start of every invocation.

**`memories/`** — Daily scratch pad files (e.g. `memories/2026-03-09.md`). Throughout the day, the agent jots down notes, observations, and things worth remembering. These are append-only logs.

**`cron/`** — Everything related to scheduled tasks lives here. `cron/config.yaml` defines the jobs — a top-level `jobs:` key containing a list of jobs, each with a `name` and `cron` expression. Simple jobs include an inline `prompt`. Complex jobs omit the prompt — the scheduler automatically looks for a matching `cron/{name}.md` file. The default heartbeat (every 30 minutes) is just another cron job.

### 5. Scheduler

The scheduler runs cron jobs defined in `cron/config.yaml` under the `jobs:` key. Each job has a `name` and `cron` expression. When a job fires, the scheduler checks for an inline `prompt` first. If none, it looks for `cron/{name}.md` by convention. The prompt is sent to the agent through the router as a synthetic message on the primary channel, addressed to the owner's main conversation.

The heartbeat is just a cron job that runs every 30 minutes — no special implementation needed.

### 6. Skills

Skills are markdown instruction files that teach the agent how to accomplish tasks using its tools. Logos follows the [Agent Skills](https://agentskills.io) open standard — the same format used by Claude Code, Cursor, Gemini CLI, and others. This means skills built for those platforms can work in Logos too.

Each skill is a directory containing a `SKILL.md` file with YAML frontmatter (name, description, requirements) and a markdown body with instructions. See the [specification](https://agentskills.io/specification) for the full format.

Skills live in `skills/` at the project root. The agent discovers available skills by reading their names and descriptions at startup. When a skill is relevant to a conversation, the agent reads the full `SKILL.md` and follows its instructions.

A skill can declare requirements in its frontmatter — environment variables, CLI tools, etc. If the requirements aren't met, the skill is unavailable.

### 7. Self-modification

The agent can edit its own source code via its shell tool. TypeScript is executed directly with `tsx` — there is no build step. To apply code changes, the agent restarts itself using the `./logos restart` wrapper script.

The wrapper script type-checks the code (`tsc --noEmit`) before restarting. If the check fails, the restart is aborted and the old process keeps running. This prevents the agent from killing itself with a bad edit.

The typical flow: the agent edits a file, sends a message explaining what it changed, then runs `./logos restart` as its last action.

## Startup flow

1. Initialize SQLite database (create tables if they don't exist)
2. Discover skills (scan `skills/` for `SKILL.md` files, load names and descriptions)
3. Register channels (each channel checks for its credentials and connects if present). If no channels connect, exit with an error — there's nothing to do without at least one channel.
4. Start the scheduler (load cron jobs from `cron/config.yaml`)
5. Begin processing incoming messages

## File structure

```
# Project root
package.json        # Dependencies
tsconfig.json       # TypeScript config

# Code
src/
  index.ts          # Entry point — starts everything
  router.ts         # Message routing and conversation queue
  agent.ts          # AI SDK wrapper
  db.ts             # SQLite operations (messages only)
  scheduler.ts      # Cron scheduler
  channels/
    registry.ts     # Static channel registration
    telegram.ts     # Telegram channel
    ...             # One file per channel

# Data
data/
  chat.sqlite       # SQLite database
SOUL.md             # Agent identity — personality, voice, values
memory.md           # Consolidated long-term memory
memories/           # Daily scratch pad files
  2026-03-09.md
  2026-03-10.md
cron/               # Scheduled tasks — all in one place
  config.yaml       # Job schedules (name + cron expression)
  heartbeat.md      # Instructions for the heartbeat job
  consolidate-memories.md
logos               # Wrapper script (start/stop/restart/status)
logs/               # Runtime logs (gitignored)
skills/             # Agent skills (agentskills.io format)
  self-edit/
    SKILL.md
evals/              # Validation scenarios for testing the build
recipes/            # Implementation guides for channels and capabilities
  telegram.md
  whatsapp.md
  imessage.md
  discord.md
  slack.md
  signal.md
  matrix.md
  voice-input.md
  voice-output.md
```

## Adding a channel

A channel is a single file that exports a `register` function. The function takes the router, and returns whether it connected successfully (may be async). Inside `register`, the channel:

1. Checks if its credentials exist (e.g., the `TELEGRAM_BOT_TOKEN` environment variable)
2. If not, returns `false`
3. If yes, connects to the platform, starts forwarding messages from the owner to the router (ignoring all others), and returns `true`

The channel registry imports and registers each channel statically — no runtime file scanning. Adding a new channel means adding the channel file and importing it in the registry.

### Recipes

Implementation details for specific channels and capabilities live in `recipes/`. Each recipe is a self-contained guide — the library to use, environment variables, setup steps, and any gotchas. See `recipes/` for available options.

## Guidelines

- **Keep it simple.** One file per component. Minimal abstractions.
- **No over-engineering.** Don't build plugin systems, middleware chains, or event buses. Direct function calls are fine.
- **Fail gracefully.** If a channel can't connect, log it and move on. Don't crash the process.
- **Log usefully.** Log when channels connect, when messages are received/sent, and when errors occur.

## Security considerations

- The agent runs with shell access and the same permissions as the host user. Consider running it on a dedicated machine or in a container.
- The **shell tool** runs commands from the project root with a 30-second timeout and a 1 MB output limit. The agent should confirm destructive commands with the user before running them.
- Messaging credentials are stored as environment variables
- The SQLite database contains all your messages — protect it accordingly

## Non-goals

These are intentionally left out of the spec. They may be added later.

- **Multi-user conversations** — all messages are assumed to be from the owner
- **Multiple AI providers at once** — one provider per deployment
- **Web UI or dashboard** — the messaging app is the interface
- **Plugin system** — channels and skills are just files, not a formal plugin API
