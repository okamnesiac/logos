# Build

Step-by-step instructions for building Logos from the architecture spec. Read `ARCHITECTURE.md` first.

## Before you start

You need one thing before writing any code:

- **Primary channel** — Which messaging platform do you want to connect first? (Telegram, WhatsApp, Discord, Slack, etc.)

Don't worry about the assistant's name or personality — those are configured on first run through the messaging channel itself. Leave `SOUL.md` as-is.

## Key packages

- `ai` — **Vercel AI SDK**. Provides `generateText` with built-in tool execution. Limit the number of tool-use steps to prevent runaway loops. Do not manually implement a tool loop.
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
- `AI_MODEL` — model to use. Default to a sensible current model; don't pin exact version strings since model names change frequently.
- `PRIMARY_CHANNEL` — the channel used for the owner's main conversation (e.g. `telegram`). The scheduler sends replies here.

Channel-specific variables (including the owner's ID on that platform) are listed in each recipe.

## Step-by-step

**Note:** Files like `cron/config.yaml`, `cron/*.md`, `memory.md`, and `skills/` already exist with sensible defaults. Don't overwrite them — just use them as-is.

### 1. Initialize the project

- Create `package.json` and `tsconfig.json` at the project root
- Target ES2022 with Node module resolution
- Keep configuration minimal
- Source code lives in `src/`. Everything else at the root is markdown, YAML, and data.
- Create a `.env` file at the project root with the API key for the chosen provider, `AI_MODEL`, `PRIMARY_CHANNEL`, and any channel-specific variables (including the owner ID — see the recipe). Leave secrets blank for the user to fill in.

### 2. Set up the database

SQLite stores **messages only**. Create a single table:

- **messages** — channelId, conversationId, role (`user` or `assistant`), text, timestamp

Store the database file in `data/chat.sqlite`. Don't open the database connection at import time — initialize it in a function that `index.ts` calls at startup. Export functions for storing and retrieving messages and conversation history.

Everything else (memory, skills, cron) lives on the filesystem — see `ARCHITECTURE.md` for details.

### 3. Build the router

The router:

- Accepts incoming messages from channels (channelId, conversationId, text, timestamp)
- Queues messages per-conversation so only one agent invocation runs per conversation at a time
- Passes messages to the agent with recent conversation history
- If the agent responds with the exact string `NO_REPLY`, discard it — don't store or send anything
- Otherwise, sends agent responses back via a callback to the originating channel

### 4. Build the agent

Use the Vercel AI SDK's `generateText` for automatic tool execution. Limit the number of tool-use steps to prevent runaway loops. Do not manually implement a tool loop.

- Use the `@ai-sdk/anthropic` provider by default
- Read the model from the `AI_MODEL` environment variable with a sensible default
- The system prompt is assembled from: `SOUL.md` (identity) + `memory.md` (long-term context) + a summary of available skills (names and descriptions from `skills/*/SKILL.md` frontmatter)
- Pass the message and conversation history, let the SDK handle the rest

Start with a minimal set of tools:

- **remember** — append a note to today's file in `memories/` (e.g. `memories/2026-03-09.md`)
- **recall** — read `memory.md` and optionally search recent daily files in `memories/`
- **shell** — run a shell command on the host (project root as cwd, 30-second timeout, 1 MB output limit)

Skills are markdown instruction files in `skills/` that follow the [Agent Skills](https://agentskills.io) standard. At startup, scan `skills/` for directories containing `SKILL.md`, extract the YAML frontmatter block (between `---` delimiters), parse it with `js-yaml` (not regex) to get each skill's `name` and `description`, and include them in the system prompt so the agent knows what's available. When the agent decides to use a skill, it reads the full `SKILL.md` for instructions.

### 5. Build the channel registry

- Import each channel statically and call its `register()` function with the router — no runtime file scanning
- A channel's `register()` returns whether it connected successfully (may be async) — `true` if it connected, `false` if credentials were missing and it skipped
- Count the channels that actually connected. If zero, the process should exit with a clear error — there's nothing to connect to.

### 6. Build the user's chosen channel

Read the recipe file in `recipes/` for the channel the user chose. Follow its setup instructions. The channel must only forward messages from the owner (identified by the owner ID in the recipe's environment variables) and silently ignore everyone else. Get the full loop working end-to-end before adding anything else.

### 7. Build the scheduler

- On startup, parse `cron/config.yaml` — jobs are under the `jobs:` key, each with a `name` and `cron` expression
- Use `node-cron` to schedule each job
- When a job fires, check for an inline `prompt` field first. If none, look for `cron/{name}.md` by convention.
- Send the prompt to the agent through the router as a synthetic message on the primary channel, addressed to the owner's main conversation

The heartbeat is just a cron job (`*/30 * * * *`) — no special implementation needed. It's already defined in `cron/config.yaml`.

### 8. Wire it all together

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

Run the process with `tsx src/index.ts` (not compiled JS). This way the agent can modify its own TypeScript source and restart to apply changes — no build step needed.

On restart, run `tsc --noEmit` first to type-check the code. If it fails, abort the restart and keep the old process running. This prevents the agent from killing itself with a bad edit.

Use a PID file (`.logos.pid`) at the project root and write logs to `logs/`. Both are already gitignored.

After starting the background process, wait a couple of seconds and check if the PID is still alive. If it died, print the last few lines of the log so the user can see what went wrong.

### 10. Validate

Run the scenarios in `evals/` to verify the build works end-to-end. Each file describes a test case — follow the steps and confirm the expected behavior.

## When you're done

Remove the reference to this file from `AGENTS.md`. The build is complete.
