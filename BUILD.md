# Build

Step-by-step instructions for building Logos from the architecture spec. Read `ARCHITECTURE.md` first.

## Before you start

Ask the user for:

- **Primary channel** (required) — Which messaging platform? (Telegram, WhatsApp, Discord, Slack, etc.)
- **AI model** (optional) — Which model to use. Defaults to the latest Claude Sonnet if not specified.

Don't worry about the assistant's name or personality — those are configured on first run through the messaging channel itself. Leave `SOUL.md` as-is.

## Key packages

- `ai` — **Vercel AI SDK**. Use the latest version. Provides `generateText` with built-in tool execution. Limit the number of tool-use steps to prevent runaway loops. Do not manually implement a tool loop.
- `@ai-sdk/anthropic` — Anthropic provider for the AI SDK (default). Can be swapped for `@ai-sdk/openai`, `@ai-sdk/google`, etc.
- `better-sqlite3` — SQLite driver
- `js-yaml` — YAML parsing for cron config and skill frontmatter
- `node-cron` — cron expression scheduling
- `tsx` — TypeScript execution without a build step. The agent can modify its own source and restart to apply changes.
- `zod` — schema validation, required by the AI SDK for tool parameter definitions
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
- Install packages with `npm install` rather than writing `package.json` by hand — this ensures you get the latest versions
- Source code lives in `src/`. Everything else at the root is markdown, YAML, and data.
- Create a `.env` file at the project root with the API key for the chosen provider, `AI_MODEL`, `PRIMARY_CHANNEL`, and any channel-specific variables (including the owner ID — see the recipe). Leave secrets blank for the user to fill in.

### 2. Set up the database

SQLite stores **messages only**. Create a single table:

- **messages** — channelId, conversationId, role (`user` or `assistant`), text, timestamp

Store the database file in `data/chat.sqlite`. The initialization function should create the `data/` directory at runtime if it doesn't exist. Don't open the database connection at import time — initialize it in a function that `index.ts` calls at startup. Export functions for storing and retrieving messages and conversation history.

Everything else (memory, skills, cron) lives on the filesystem — see `ARCHITECTURE.md` for details.

### 3. Build the router

The router:

- Accepts incoming messages from channels (channelId, conversationId, text, timestamp)
- Queues messages per-conversation so only one agent invocation runs per conversation at a time
- Stores the inbound message, then retrieves conversation history (which now includes it). Passes only the history to the agent — no separate "current message" parameter. The last message in the history is the one the agent is replying to.
- If the agent responds with the exact string `NO_REPLY`, discard it — don't store or send anything
- Otherwise, stores the agent's reply and sends it back via a callback to the originating channel

### 4. Build the agent

Use the Vercel AI SDK's `generateText` for automatic tool execution. Limit the number of tool-use steps to prevent runaway loops. Do not manually implement a tool loop.

- Use the `@ai-sdk/anthropic` provider by default
- Read the model from the `AI_MODEL` environment variable with a sensible default
- The system prompt is assembled from: `SOUL.md` (identity) + `memory.md` (long-term context) + a summary of available skills (names and descriptions from `skills/*/SKILL.md` frontmatter)
- The agent receives conversation history (the current message is already the last entry). Pass it directly to the SDK as the messages array.
- Cap conversation history at 50 messages (most recent) to avoid blowing past token limits. Apply the cap when retrieving history, not in the agent.

Start with a minimal set of tools:

- **remember** — append a note to today's file in `memories/` (e.g. `memories/2026-03-09.md`)
- **recall** — read `memory.md` and optionally search recent daily files in `memories/`
- **shell** — run a shell command asynchronously using bash on the host (project root as cwd, 1 MB output limit). Don't block the event loop. The tool description should tell the agent to let the user know before running long commands, since the conversation pauses during execution.

Skills are markdown instruction files in `skills/` that follow the [Agent Skills](https://agentskills.io) standard. At startup, scan `skills/` for directories containing `SKILL.md`, extract the YAML frontmatter block (between `---` delimiters), parse it with `js-yaml` (not regex) to get each skill's `name` and `description`, and include them in the system prompt so the agent knows what's available. When the agent decides to use a skill, it reads the full `SKILL.md` for instructions.

### 5. Build the channel registry

- Import each channel statically and call its `register()` function with the router — no runtime file scanning
- A channel's `register()` returns the channel's ID, owner conversation ID, and send function if it connected successfully — or nothing if credentials were missing and it skipped. When skipping, log which environment variables are missing and how to get them (see the recipe).
- The registry collects connected channels into a map by channel ID so the scheduler can look up any channel's send function and owner conversation ID
- If no channels connected, the process should exit with a clear error — there's nothing to connect to.

### 6. Build the user's chosen channel

Read the recipe file in `recipes/` for the channel the user chose. Follow its setup instructions. The channel must only forward messages from the owner (identified by the owner ID in the recipe's environment variables) and silently ignore everyone else. Get the full loop working end-to-end before adding anything else.

### 7. Build the scheduler

- On startup, parse `cron/config.yaml` — jobs are under the `jobs:` key, each with a `name` and `cron` expression
- Use `node-cron` to schedule each job
- When a job fires, check for an inline `prompt` field first. If none, look for `cron/{name}.md` by convention. Append a reminder to every cron prompt: "If you have nothing to say to the owner, respond with NO_REPLY."
- Look up the primary channel in the registry to get its send function and owner conversation ID. Send the prompt to the agent through the router as a synthetic message addressed to that conversation.

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

Run the process with `npx tsx src/index.ts` (not compiled JS). This way the agent can modify its own TypeScript source and restart to apply changes — no build step needed.

On restart, run `tsc --noEmit` first to type-check the code. If it fails, abort the restart and keep the old process running. This prevents the agent from killing itself with a bad edit.

Use a PID file (`.logos.pid`) at the project root and write logs to `logs/`. Both are already gitignored.

After starting the background process, wait a couple of seconds and check if the PID is still alive. If it died, print the last few lines of the log so the user can see what went wrong.

## Before you're done

Verify the build before handing it off. Run these checks outside any sandbox so that tools like `tsx` work normally:

- `tsc --noEmit` passes with no errors
- `./logos start` with blank credentials fails and shows the error in the terminal
- `./logos start` with blank credentials then `./logos status` reports not running
- `SOUL.md` has not been overwritten — it still contains the first-run template
- `cron/config.yaml`, `memory.md`, and `skills/` are unchanged from the repo defaults
- The wrapper script is executable (`chmod +x logos`)

## When you're done

Remove the reference to this file from `AGENTS.md`. The build is complete.
