# Bootstrap

Step-by-step instructions for building Logos from the architecture spec. Read `ARCHITECTURE.md` first.

## Before you start

You need three things before writing any code. The user may provide them inline (e.g. "bootstrap Diff telegram witty") or you can ask:

1. **Name** — What should your assistant be called?
2. **Primary channel** — Which messaging platform do you want to connect first? (Telegram, WhatsApp, Discord, Slack, etc.)
3. **Personality** — Describe your assistant's personality and tone. (casual, formal, terse, witty, etc.) Use the answers to populate `SOUL.md`.

The chosen name should be set as `ASSISTANT_NAME` in `.env`. The code should read it from there rather than hardcoding it.

## Key packages

- `ai` — **Vercel AI SDK**. Provides `generateText` with built-in tool execution via `maxSteps`. Do not manually implement a tool loop.
- `@ai-sdk/anthropic` — Anthropic provider for the AI SDK (default). Can be swapped for `@ai-sdk/openai`, `@ai-sdk/google`, etc.
- `better-sqlite3` — SQLite driver
- `js-yaml` — YAML parsing for cron config and skill frontmatter
- `node-cron` — cron expression scheduling
- `tsx` — TypeScript execution without a build step. The agent can modify its own source and restart to apply changes.
- `dotenv` — load environment variables from `.env`
- Channel-specific libraries — see the chosen recipe in `recipes/`

## Environment variables

All secrets (API keys, bot tokens) go in `.env` at the project root. This file is gitignored. Load it at startup with `dotenv`.

At minimum:

- The API key for whichever AI provider you're using (e.g. `ANTHROPIC_API_KEY` for Anthropic)
- `AI_MODEL` — model to use (default: latest Claude Sonnet)
- `ASSISTANT_NAME` — your assistant's name (default: `Logos`)

Channel-specific variables are listed in each recipe.

## Step-by-step

**Note:** Files like `cron/config.yaml`, `cron/*.md`, `memory.md`, and `skills/` already exist with sensible defaults. Don't overwrite them — just use them as-is.

### 1. Initialize the project

- Create `package.json` and `tsconfig.json` inside `src/`
- Target ES2022 with Node module resolution
- Keep configuration minimal
- **All code and TypeScript/JavaScript configuration belongs in `src/`.** Everything outside `src/` is markdown, YAML, and data — human-readable files the agent and user interact with directly.
- Create a `.env` file at the project root with the API key for the chosen provider, `AI_MODEL`, `ASSISTANT_NAME`, and any channel-specific variables. Leave the API key blank for the user to fill in.

### 2. Set up the database (`src/db.ts`)

SQLite stores **messages only**. Create a single table:

- **messages** — channelId, conversationId, senderId, senderName, text, timestamp, direction (inbound/outbound)

Export functions for storing and retrieving messages and conversation history.

Everything else (memory, tasks, conversation context) lives on the filesystem — see `ARCHITECTURE.md` for details.

### 3. Build the router (`src/router.ts`)

The router:

- Accepts incoming messages from channels in the common message format (see ARCHITECTURE.md)
- Queues messages per-conversation so only one agent invocation runs per conversation at a time
- Passes messages to the agent with recent conversation history
- Sends agent responses back via a callback to the originating channel

### 4. Build the agent (`src/agent.ts`)

Use the Vercel AI SDK's `generateText` with `maxSteps` for automatic tool execution. Do not manually implement a tool loop.

- Use the Anthropic provider by default: `import { anthropic } from "@ai-sdk/anthropic"`
- Read the model from `process.env.AI_MODEL` with a sensible default
- The system prompt is assembled from: `SOUL.md` (identity) + `memory.md` (long-term context) + the conversation's file from `conversations/` if one exists + a summary of available skills (names and descriptions from `skills/*/SKILL.md` frontmatter)
- Pass the message and conversation history, let the SDK handle the rest

Start with a minimal set of tools:

- **remember** — append a note to today's file in `memories/` (e.g. `memories/2026-03-09.md`)
- **recall** — read `memory.md` and optionally search recent daily files in `memories/`
- **shell** — run a shell command on the host

Skills are markdown instruction files in `skills/` that follow the [Agent Skills](https://agentskills.io) standard. At startup, scan `skills/` for directories containing `SKILL.md`, parse the YAML frontmatter with `js-yaml` to extract each skill's `name` and `description`, and include them in the system prompt so the agent knows what's available. When the agent decides to use a skill, it reads the full `SKILL.md` for instructions.

### 5. Build the channel registry (`src/channels/registry.ts`)

- Scan the `src/channels/` directory for channel files
- Call each channel's `register()` function with the router
- Channels that lack credentials simply don't activate
- Return the number of channels that registered. If zero, the process should exit with a clear error — there's nothing to connect to.

### 6. Build the user's chosen channel

Read the recipe file in `recipes/` for the channel the user chose. Follow its setup instructions. Get the full loop working end-to-end before adding anything else.

### 7. Build the scheduler (`src/scheduler.ts`)

- On startup, parse `cron/config.yaml` — jobs are under the `jobs:` key, each with a `name` and `cron` expression
- Use `node-cron` to schedule each job
- When a job fires, check for an inline `prompt` field first. If none, look for `cron/{name}.md` by convention.
- Send the prompt to the agent through the router as a synthetic message

The heartbeat is just a cron job (`*/30 * * * *`) — no special implementation needed. It's already defined in `cron/config.yaml`.

### 8. Wire it all together (`src/index.ts`)

The entry point:

1. Initializes the database
2. Registers channels
3. Starts the scheduler
4. Logs that it's running

That's it. No HTTP server needed unless a channel requires a webhook.

### 9. Create the `./logos` wrapper script

Create a simple bash script at the project root called `logos` that supports:

- `./logos` or `./logos start` — start the process in the background
- `./logos stop` — stop it
- `./logos restart` — restart it
- `./logos status` — check if it's running

Run the process with `tsx index.ts` (not compiled JS). This way the agent can modify its own TypeScript source and restart to apply changes — no build step needed.

On restart, run `tsc --noEmit` first to type-check the code. If it fails, abort the restart and keep the old process running. This prevents the agent from killing itself with a bad edit.

Use a PID file (`.logos.pid`) at the project root and write logs to `logs/`. Both are already gitignored.

## When you're done

Remove the reference to this file from `AGENTS.md` and delete `BOOTSTRAP.md`. It has served its purpose.
