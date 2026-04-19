# Build

Step-by-step instructions for building Logos from the spec. Read `ARCHITECTURE.md` first.

All paths in this document are **relative to the workspace root** (the directory that contains `spec/`, `agent/`, `config/`, etc.). Run all commands from there.

The bootstrap reads from `spec/` (recipes, skills, cron defaults, this document) and generates code in `agent/`. After bootstrap, the running agent reads from `spec/` (skills, cron) and `agent/` (its own implementation), with `config/` for instance overrides.

## Before you start

Ask the user for:

- **Primary channel** (required) — Which messaging platform? (Telegram, WhatsApp, Discord, Slack, etc.)
- **AI model** (optional) — Which model to use. Defaults to the latest Claude Sonnet if not specified.

Don't worry about the assistant's name or personality — those are configured on first run through the messaging channel itself. The agent prompts the user and writes `config/SOUL.md` automatically.

## Key packages

- `ai` — **Vercel AI SDK**. Use the latest version. Provides `generateText` with built-in tool execution. Limit the number of tool-use steps to prevent runaway loops. Do not manually implement a tool loop.
- `@ai-sdk/anthropic` — Anthropic provider for the AI SDK (default). Can be swapped for `@ai-sdk/openai`, `@ai-sdk/google`, etc.
- `js-yaml` — YAML parsing for cron frontmatter and skill frontmatter
- `node-cron` — cron expression scheduling
- `tsx` — TypeScript execution without a build step. The agent can modify its own source and restart to apply changes.
- `zod` — schema validation, required by the AI SDK for tool parameter definitions
- `dotenv` — load environment variables from `config/.env`
- Channel-specific libraries — see the chosen recipe in `spec/channels/`

## Environment variables

All secrets (API keys, bot tokens) go in `config/.env`. This file is gitignored (the entire `config/` directory is gitignored by the spec repo). Load it at startup with `dotenv` pointed explicitly at `config/.env`.

At minimum:

- The API key for whichever AI provider you're using (e.g. `ANTHROPIC_API_KEY` for Anthropic)
- `AI_MODEL` — model to use. Default to a sensible current model; don't pin exact version strings since model names change frequently.
- `PRIMARY_CHANNEL` — the channel used for the owner's main conversation (e.g. `telegram`). The scheduler sends replies here.

Channel-specific variables (including the owner's ID on that platform) are listed in each channel recipe under `spec/channels/`.

## Step-by-step

**Note:** Default cron jobs (`spec/cron/heartbeat.md`, `spec/cron/consolidate-memories.md`) and bundled skills (`spec/skills/`) ship with the spec. The running agent reads these directly. Don't copy them into `agent/`.

### 1. Initialize the project

- Create `agent/package.json` and `agent/tsconfig.json`
- Target ES2022 with Node module resolution
- Keep configuration minimal
- Install packages with `npm install <package-name>` from inside `agent/` rather than writing `package.json` by hand — this ensures you get the latest versions and only lists direct dependencies. **Never manually edit the `dependencies` or `devDependencies` objects in `package.json`.**
- Source code lives in `agent/src/`. Top-level engine modules (`index.ts`, `router.ts`, `agent.ts`, `scheduler.ts`, `threads.ts`, `memory.ts`) sit at `agent/src/` root. Capability code lives in `agent/src/channels/` and `agent/src/tools/`.
- Use `process.cwd()` for the workspace root path, not `import.meta.dirname` — tsx runs in CJS mode where `import.meta.dirname` is undefined. The wrapper script ensures the process runs with the workspace root as cwd.
- Create `config/` if it doesn't exist (the agent should do this on first run, but the build can pre-create it). Create a `config/.env` template with the API key for the chosen provider, `AI_MODEL`, `PRIMARY_CHANNEL`, and any channel-specific variables. Leave secrets blank for the user to fill in.

### 2. Set up message storage

No database. Each conversation is an append-only JSONL file at `runtime/threads/{channelId}/{conversationId}.jsonl`. Each line is one message:

```jsonl
{"role":"user","text":"hi","timestamp":"2026-04-18T10:30:00.000Z"}
{"role":"assistant","text":"hello!","timestamp":"2026-04-18T10:30:02.000Z"}
```

Create a small module (e.g. `agent/src/threads.ts`) with two functions:

- `appendMessage(channelId, conversationId, role, text, timestamp)` — ensure the parent directory exists, then append one JSON line with a trailing `\n`. Use `fs.promises.appendFile` (which creates the file if missing) or write a thin wrapper. Sanitize `channelId` and `conversationId` to safe path segments — reject anything with `/`, `..`, or other path-escape characters.
- `getHistory(channelId, conversationId, limit)` — read the file (return empty if it doesn't exist), split on newlines, parse each line as JSON, drop empty/malformed lines, and return the last `limit` entries.

The router serializes per-conversation, so `appendMessage` never has a concurrent writer on the same file. No locking needed.

Everything else (memory, skills, cron, identity) lives on the filesystem in the appropriate domain — see `ARCHITECTURE.md`.

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
- The system prompt is assembled from:
  1. `config/SOUL.md` (identity) — if missing, run the first-run flow (see step 4a)
  2. A **memory manifest** — a flat list of every memory file with its name, aliases, tags, and a one-line summary. **Do NOT load full file contents.** The summary comes from (in order): the frontmatter `description:` field, the first H1 heading in the body, or the first ~100 chars of body text. The agent uses `find_memory` and `read_file` to fetch full content on demand.
  3. The last 24 hours of `memory/journal/` entries inline (these are recent agent-authored notes likely to be relevant; older journal entries appear in the manifest only)
  4. A summary of available skills (names and descriptions from `spec/skills/*/SKILL.md` and `config/skills/*/SKILL.md` frontmatter; config wins on name collision)
  5. Today's date — so `remember` and other date-aware behavior work without a tool call
- The agent receives conversation history (the current message is already the last entry). Pass it directly to the SDK as the messages array.
- Cap conversation history at 50 messages (most recent) to avoid blowing past token limits. Apply the cap when retrieving history, not in the agent.
- Guard against oversized prompts. Estimate tokens using a 4:1 character-to-token ratio. First, truncate any individual message over 10,000 tokens. Then, if the total (system prompt + messages) exceeds 150,000 tokens, drop the oldest messages until it fits.

#### 4a. First-run flow

If `config/SOUL.md` doesn't exist when the agent assembles its system prompt:

- Use a minimal bootstrap system prompt: "You are a new personal AI assistant named Logos. You haven't been configured yet. On your next reply, introduce yourself and ask the user (1) what to call yourself and (2) how you should act — personality, tone, style. Once they answer, use `write_file` with `mode: \"create\"` to create `config/SOUL.md` with the chosen name and personality. Nothing else belongs in that file — memory, skills, and other context are loaded separately."
- After the user answers, the agent writes `config/SOUL.md`. Subsequent invocations read it normally.
- Also ensure `config/`, `memory/`, and `runtime/` directories exist; create them if not.

Six tools live in `agent/src/tools/`:

- **read_file** `(path)` — read a file from the workspace. Takes a relative path, returns the file contents. Reject paths that escape the workspace root (e.g. `../` or absolute paths).
- **write_file** `(path, content, mode)` — write to a file. Modes:
  - `create` — fail if the file already exists
  - `append` — append to the end (insert a `\n` separator if the existing file doesn't end with one)
  - `replace` — overwrite the whole file
- **edit_file** `(path, old_string, new_string)` — surgical find-and-replace. `old_string` must appear exactly once in the file, otherwise the call fails (forces the agent to add surrounding context for uniqueness). Cheaper than full rewrites for small updates.
- **find_memory** `(name)` — resolve a wiki-link-style name (or alias) to a path under `memory/` using the link resolver (see step 4b). Returns a discriminated union — **never `null`**:
  - On hit: `{ found: true, path: "memory/...", backlinks: ["memory/..."] }`
  - On miss: `{ found: false }`

  The `found` boolean makes the shape unambiguous to the model and leaves room to add diagnostic fields (e.g. fuzzy-match suggestions) later. **Does not lazy-create.** When the agent wants a new note, it picks a path and uses `write_file` with `mode: "create"`.
- **remember** `(text)` — sugar for appending to today's journal at `memory/journal/{YYYY-MM-DD}.md`. Equivalent to `write_file` with that path in `append` mode; kept as a separate tool because journaling is the most common write pattern.
- **shell** `(cmd)` — run a shell command asynchronously using bash on the host (workspace root as cwd, 1 MB output limit). Don't block the event loop. The tool description should tell the agent to let the user know before running long commands, since the conversation pauses during execution.

Custom tools are added by dropping `.ts` files directly into `agent/src/tools/` alongside the built-in ones. The loader scans that single directory.

#### 4b. Memory graph

Memory uses Obsidian-compatible markdown with `[[wiki-link]]` syntax (see `ARCHITECTURE.md` → Memory format for the conventions).

Build a small memory module (e.g. `agent/src/memory.ts`) with:

- **Link resolver** — given a link target (a name, optionally with path hints), find the matching file under `memory/` using these rules in order:
  1. Filename match (without `.md`) or alias match (from frontmatter `aliases:`)
  2. If multiple, pick shortest path, then alphabetical
  3. If none, return null. Do NOT auto-create. The agent decides whether to create a missing note (via `write_file` with `mode: "create"`).
- **Graph builder** — at startup, scan `memory/**/*.md`, parse frontmatter and `[[...]]` links from each file, build:
  - A name index (filename + aliases → file path)
  - A backlink index (file path → list of files that link to it)
  - A **manifest** for the system prompt: a flat list of `{ relPath, name, aliases, tags, summary }` for every file. `summary` is the frontmatter `description:` field if present, else the first H1 heading, else the first ~100 chars of body.
- **Cache** — write the graph to `runtime/memory-graph.json` after building. On startup, check the cache: if all source file mtimes are <= the cache's mtime, use it; otherwise rebuild.
- **Frontmatter parser** — use `js-yaml` to parse the YAML block between `---` delimiters at the top of each file. Tolerate missing or malformed frontmatter (treat as empty).

The `find_memory` tool wraps the resolver. The internal resolver function may return `null` on miss, but the tool wrapper MUST translate that to `{ found: false }` (see the tool return shape above). The agent then checks `result.found`, and if true, uses `read_file(result.path)` to fetch contents.

Skills are markdown instruction files following the [Agent Skills](https://agentskills.io) standard. At startup, scan both `spec/skills/` and `config/skills/` for directories containing `SKILL.md`. Extract the YAML frontmatter block (between `---` delimiters), parse it with `js-yaml` (not regex) to get each skill's `name` and `description`, and include them in the system prompt. On name collision, `config/` wins. When the agent decides to use a skill, it reads the full `SKILL.md` for instructions.

### 5. Build the channel registry

- Scan `agent/src/channels/` for `*.ts` files at startup. For each, dynamically import and call its `register()` function with the router. No manual registration list — channels are discovered. Both built-in channels (generated from spec recipes) and custom channels live in the same directory.
- A channel's `register()` returns the channel's ID, owner conversation ID, and send function if it connected successfully — or nothing if credentials were missing and it skipped. When skipping, log which environment variables are missing and how to get them (see the recipe in `spec/channels/{name}.md`).
- The registry collects connected channels into a map by channel ID so the scheduler can look up any channel's send function and owner conversation ID
- If no channels connected, the process should exit with a clear error — there's nothing to connect to.

### 6. Build the user's chosen channel

Read the recipe at `spec/channels/{name}.md` for the channel the user chose and follow its setup instructions. The implementation goes at `agent/src/channels/{name}.ts`. The channel must only forward messages from the owner (identified by the owner ID in the recipe's environment variables) and silently ignore everyone else. Get the full loop working end-to-end before adding anything else.

### 7. Build the scheduler

- On startup, scan both `spec/cron/` and `config/cron/` for `*.md` files. The filename (minus extension) is the job name.
- For each unique job name, parse frontmatter from the spec and config versions and merge:
  - **Frontmatter:** start with spec, override with config keys
  - **Body:** spec body first, config body appended (separated by a blank line)
  - If `enabled: false` appears in the merged frontmatter, skip the job
- Use `node-cron` to schedule each enabled job using its merged `schedule` field
- When a job fires, append a reminder to the merged body: "If you have nothing to say to the owner, respond with NO_REPLY."
- Look up the primary channel in the registry to get its send function and owner conversation ID. Send the prompt to the agent through the router as a synthetic message addressed to that conversation.

The default heartbeat (`spec/cron/heartbeat.md`, every 30 minutes) and consolidate-memories job (`spec/cron/consolidate-memories.md`, daily at 23:00) ship with the spec.

Add a CLI subcommand `agent/logos cron list` that prints the merged job table with source annotations (`[spec]`, `[config]`, `[spec → overridden by config]`, `[disabled]`).

### 8. Wire it all together

The entry point (`agent/src/index.ts`):

1. Loads `config/.env` via dotenv
2. Ensures `runtime/` exists
3. Registers channels
4. Starts the scheduler
5. Logs that it's running

That's it. No HTTP server needed unless a channel requires a webhook.

### 9. Create the `agent/logos` wrapper script

Create a bash script at `agent/logos` that supports:

- `agent/logos` or `agent/logos start` — start the process in the background
- `agent/logos stop` — stop it
- `agent/logos restart` — restart it
- `agent/logos status` — check if it's running
- `agent/logos cron list` — show the merged cron job table

The script must be invoked from the workspace root (the parent of `agent/`). It should `cd` to the workspace root if invoked from elsewhere by resolving its own location, then `cd ..` from `agent/`.

Run the process with `npx tsx agent/src/index.ts` (not compiled JS). This way the agent can modify its own TypeScript source and restart to apply changes — no build step needed.

Use a PID file at `runtime/logos.pid` and write logs to `runtime/logs/`. Both are gitignored. Append to the log file — don't truncate it on restart.

**Safe-restart protocol for `restart`:**

1. **Snapshot the current `agent/` HEAD.** If `agent/` is a Git repo (`agent/.git` exists), record the current commit SHA with `git -C agent rev-parse HEAD`. If it's not a Git repo, skip the rollback steps below and warn the user that self-edit rollback won't work.
2. **Typecheck.** Run `tsc --noEmit -p agent/tsconfig.json`. If it fails, abort the restart, keep the old process running, and print the error.
3. **Stop the old process** (if running) and **start the new one** in the background.
4. **Health check.** Wait a few seconds (e.g. 5s) and check if the new PID is still alive. If dead:
   - Print the last ~30 lines of the log so the user can see what crashed.
   - If `agent/` is a Git repo and HEAD has moved since the snapshot: run `git -C agent reset --hard <snapshot-sha>` to revert the self-edit, then start the process again with the pre-edit code. Log the auto-revert clearly (`[logos] post-start check failed; auto-reverted agent to <sha> and restarted`).
   - If no Git repo or HEAD is unchanged: leave the process stopped and tell the user to investigate.

This closes the runtime-crash hole in self-edit. `tsc --noEmit` catches compile errors; the post-start health check catches runtime startup errors; Git auto-revert recovers without human intervention.

Note: The auto-revert only affects the last commit (or last few commits, if the agent committed more than once since the snapshot). It does NOT revert committed changes in `config/` or `memory/` — those aren't part of self-edit.

## Before you're done

Verify the build before handing it off. Run these checks outside any sandbox so that tools like `tsx` work normally:

- `tsc --noEmit` (against `agent/tsconfig.json`) passes with no errors
- `agent/logos start` with blank credentials fails and shows the error in the terminal
- `agent/logos start` with blank credentials then `agent/logos status` reports not running
- `config/SOUL.md` does not exist yet (it's written on first run, not by the build)
- `spec/cron/heartbeat.md` and `spec/cron/consolidate-memories.md` are unchanged
- `spec/` is untouched by the build (the bootstrap only writes to `agent/` and optionally creates `config/`)
- The wrapper script is executable (`chmod +x agent/logos`)

## When you're done

Remove the reference to this file from `AGENTS.md` and `CLAUDE.md`. The build is complete.
