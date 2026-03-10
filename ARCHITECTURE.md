# Architecture

## Overview

Logos is a single-process personal AI assistant. Messages come in from messaging channels, get processed by an AI agent, and responses go back out through the same channels.

```
┌─────────────────────────────────────────────────────┐
│                    Logos Process                     │
│                                                      │
│  ┌──────────┐   ┌──────────┐   ┌──────────────┐     │
│  │ Channels │──▶│  Router  │──▶│    Agent     │     │
│  │          │◀──│          │◀──│  (AI SDK)   │     │
│  └──────────┘   └──────────┘   └──────┬───────┘     │
│       │                               │              │
│       │     ┌───────────┐  ┌──────────┴──────────┐  │
│       │     ┌───────────┐  │    File System       │  │
│       │     │messages.db│  │ SOUL.md              │  │
│       │     └───────────┘  │ memory.md            │  │
│       │                    │ memories/            │  │
│  ┌──────────┐              │ conversations/       │  │
│  │Scheduler │              │ heartbeat.md         │  │
│  └──────────┘              │ cron.yaml + cron/    │  │
│                            └─────────────────────┘  │
└─────────────────────────────────────────────────────┘
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

**Tools** are capabilities the agent can use: reading files, running shell commands, searching the web, etc. The AI SDK handles tool execution loops natively via `maxSteps`.

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

**`SOUL.md`** — The agent's identity. Personality, voice, values, opinions. The agent reads this first, every time it wakes — it reads itself into being. Edit this file to change who your assistant is.

**`memory.md`** — The agent's consolidated long-term memory. Key facts, user preferences, important context that persists across all conversations. The agent reads this at the start of every invocation.

**`memories/`** — Daily scratch pad files (e.g. `memories/2026-03-09.md`). Throughout the day, the agent jots down notes, observations, and things worth remembering. These are append-only logs.

**`conversations/`** — One markdown file per conversation (e.g. `conversations/family-chat.md`). Contains the custom system prompt and any notes for how the agent should behave in that conversation. The agent reads the relevant file when handling a message.

**`heartbeat.md`** — A checklist the agent reviews on a regular interval. If nothing needs attention, it moves on. Used for lightweight, recurring monitoring (check inbox, check calendar, etc.).

**`cron.yaml`** — Scheduled tasks. Top-level `jobs:` key containing a list of jobs. Each job has a `name` and `cron` expression. Simple jobs include an inline `prompt`. Complex jobs omit the prompt — the scheduler automatically looks for a matching `cron/{name}.md` file.

**`cron/`** — Detailed instructions for complex scheduled tasks. Files are matched by convention: a job named `consolidate-memories` reads from `cron/consolidate-memories.md`.

### 5. Scheduler

The scheduler handles two types of recurring work:

**Heartbeat** — Runs on a fixed interval (e.g. every 30 minutes). Reads `heartbeat.md` and lets the agent check on things. If nothing needs attention, no tokens are wasted.

**Cron jobs** — Defined in `cron.yaml` under the `jobs:` key. Each job has a `name` and `cron` expression. When a job fires, the scheduler checks for an inline `prompt` first. If none, it looks for `cron/{name}.md` by convention. The prompt is sent to the agent through the router as a synthetic message.

### 6. Self-modification

The agent can edit its own source code via its shell tool. TypeScript is executed directly with `tsx` — there is no build step. To apply code changes, the agent restarts itself using the `./logos restart` wrapper script.

The wrapper script type-checks the code (`tsc --noEmit`) before restarting. If the check fails, the restart is aborted and the old process keeps running. This prevents the agent from killing itself with a bad edit.

The typical flow: the agent edits a file, sends a message explaining what it changed, then runs `./logos restart` as its last action.

## Startup flow

1. Initialize SQLite database (create tables if they don't exist)
2. Register channels (each channel checks for its credentials and connects if present)
3. Start the scheduler (load heartbeat interval and cron jobs)
4. Begin processing incoming messages

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
  scheduler.ts      # Heartbeat and cron runner
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
heartbeat.md        # Recurring checks
cron.yaml           # Scheduled tasks
cron/               # Detailed instructions for complex cron jobs
  consolidate-memories.md
logos               # Wrapper script (start/stop/restart/status)
logs/               # Runtime logs (gitignored)
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
export function register(router: Router): void
```

Inside `register`, the channel:

1. Checks if its credentials exist (e.g., `process.env.TELEGRAM_BOT_TOKEN`)
2. If not, returns silently
3. If yes, connects to the platform and starts forwarding messages to the router

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
