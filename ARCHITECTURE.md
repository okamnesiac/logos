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
│       │     │messages.db│  │     File System      │  │
│       │     └───────────┘  │  SOUL.md             │  │
│       │                    │  memory.md           │  │
│  ┌──────────┐              │  memories/           │  │
│  │Scheduler │              │  conversations/      │  │
│  └──────────┘              │  cron/               │  │
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

**Self-registering:** A channel only activates if its credentials are present in the environment. No credentials, no channel — no errors, no configuration needed.

**Common message format:**

```
{
  channelId: string       // e.g. "whatsapp", "telegram"
  conversationId: string  // unique ID for the conversation/group
  senderId: string        // who sent the message
  senderName: string      // display name
  text: string            // message content
  timestamp: Date
  replyTo?: string        // optional message ID being replied to
  attachments?: []        // optional media attachments
}
```

Each channel lives in its own file under `src/channels/`. Adding a new channel means adding a single file that exports a `register` function.

### 2. Router

The router sits between channels and the agent. It:

- Receives normalized messages from channels
- Maintains a per-conversation message queue (one agent invocation at a time per conversation)
- Passes messages to the agent with conversation context
- Routes agent responses back to the originating channel

The router is simple glue code. It does not make decisions — it just connects channels to the agent and manages concurrency.

### 3. Agent

The agent is the brain. It uses the Vercel AI SDK to:

- Receive a message and conversation history
- Decide how to respond, optionally using tools
- Return a response

**Tools** are typed capabilities defined in code: `remember`, `recall`, and `shell`. The AI SDK handles tool execution loops natively via `maxSteps`.

**Skills** teach the agent how to do more complex things using its tools. Skills are markdown files following the [Agent Skills](https://agentskills.io) open standard — see the Skills section below for details.

**Model-agnostic:** The default provider is Anthropic (Claude), but switching to OpenAI, Google, or any other provider is a one-line change.

**Per-conversation context:** Each conversation gets its own system prompt and history. A conversation with your family group chat behaves differently than a 1:1 with your work colleague because the context is different.

### 4. Storage

Logos uses two storage mechanisms — SQLite for structured data and the filesystem for everything the user or agent should be able to inspect and edit directly.

#### SQLite (messages only)

SQLite stores message history:

- **messages** — channelId, conversationId, senderId, senderName, text, timestamp, direction (inbound/outbound)

`messages.db` lives at the project root alongside the other data files. Back it up by copying it. Inspect it with any SQLite client.

#### Filesystem

Everything else lives in plain files:

**`SOUL.md`** — The agent's identity: name, personality, voice, values. The agent reads this first, every time it wakes — it reads itself into being. On first run, the agent asks the user for a name and personality, then writes this file. Edit it anytime to change who your assistant is.

**`memory.md`** — The agent's consolidated long-term memory. Key facts, user preferences, important context that persists across all conversations. The agent reads this at the start of every invocation.

**`memories/`** — Daily scratch pad files (e.g. `memories/2026-03-09.md`). Throughout the day, the agent jots down notes, observations, and things worth remembering. These are append-only logs.

**`conversations/`** — One markdown file per conversation (e.g. `conversations/family-chat.md`). Contains the custom system prompt and any notes for how the agent should behave in that conversation. The agent reads the relevant file when handling a message.

**`cron/`** — Everything related to scheduled tasks lives here. `cron/config.yaml` defines the jobs — a top-level `jobs:` key containing a list of jobs, each with a `name` and `cron` expression. Simple jobs include an inline `prompt`. Complex jobs omit the prompt — the scheduler automatically looks for a matching `cron/{name}.md` file. The default heartbeat (every 30 minutes) is just another cron job.

### 5. Scheduler

The scheduler runs cron jobs defined in `cron/config.yaml` under the `jobs:` key. Each job has a `name` and `cron` expression. When a job fires, the scheduler checks for an inline `prompt` first. If none, it looks for `cron/{name}.md` by convention. The prompt is sent to the agent through the router as a synthetic message.

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
3. Register channels (each channel checks for its credentials and connects if present)
4. Start the scheduler (load cron jobs from `cron/config.yaml`)
5. Begin processing incoming messages

## File structure

```
# Code (all code and config lives here)
src/
  package.json      # Dependencies
  tsconfig.json     # TypeScript config
  index.ts          # Entry point — starts everything
  router.ts         # Message routing and conversation queue
  agent.ts          # AI SDK wrapper
  db.ts             # SQLite operations (messages only)
  scheduler.ts      # Cron scheduler
  channels/
    registry.ts     # Channel auto-discovery and registration
    whatsapp.ts     # WhatsApp channel
    telegram.ts     # Telegram channel
    discord.ts      # Discord channel
    slack.ts        # Slack channel
    ...             # One file per channel

# Everything outside src/ is data — human-readable files and the database
messages.db         # SQLite database (messages only)
SOUL.md             # Agent identity — personality, voice, values
memory.md           # Consolidated long-term memory
memories/           # Daily scratch pad files
  2026-03-09.md
  2026-03-10.md
conversations/      # Per-conversation context and prompts
  family-chat.md
  work-dm.md
cron/               # Scheduled tasks — all in one place
  config.yaml       # Job schedules (name + cron expression)
  heartbeat.md      # Instructions for the heartbeat job
  consolidate-memories.md
logos               # Wrapper script (start/stop/restart/status)
logs/               # Runtime logs (gitignored)
skills/             # Agent skills (agentskills.io format)
  self-edit/
    SKILL.md
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

A channel is a single file that exports:

```typescript
export function register(router: Router): boolean
```

Inside `register`, the channel:

1. Checks if its credentials exist (e.g., `process.env.TELEGRAM_BOT_TOKEN`)
2. If not, returns `false`
3. If yes, connects to the platform, starts forwarding messages to the router, and returns `true`

That's it. No configuration files, no plugin manifests.

### Recipes

Implementation details for specific channels and capabilities live in `recipes/`. Each recipe is a self-contained guide — the library to use, environment variables, setup steps, and any gotchas. See `recipes/` for available options.

## Guidelines

- **Keep it simple.** One file per component. Minimal abstractions.
- **No over-engineering.** Don't build plugin systems, middleware chains, or event buses. Direct function calls are fine.
- **Fail gracefully.** If a channel can't connect, log it and move on. Don't crash the process.
- **Log usefully.** Log when channels connect, when messages are received/sent, and when errors occur.

## Security considerations

- The agent runs directly on the host with the same permissions as the user
- Be thoughtful about which tools you give the agent (shell access, file access, etc.)
- Messaging credentials are stored as environment variables
- The SQLite database contains all your messages — protect it accordingly
