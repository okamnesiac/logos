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
- `LOGOS_SELF_EDIT` — `true` (default) or `false`. When false, the `self-edit` skill is hidden and the file-edit tools refuse writes under `agent/`. See step 4c below for the enforcement details.
- `LOGOS_TERMINAL` — `true` (default) or `false`. When false, the terminal channel doesn't bind the socket and `agent/logos chat` has no daemon to connect to.
- `LOGOS_TERMINAL_SOCKET` — optional override for the terminal socket path. Defaults to `runtime/logos.sock`.
- `LOGOS_WEB_FETCH` — `true` (default) or `false`. When false, the `web_fetch` tool refuses every call.
- `LOGOS_WEB_FETCH_BACKEND` — `fetch` (default), `jina`, or `playwright`. See `spec/tools/web_fetch.md` for tradeoffs and privacy considerations.
- `LOGOS_WEB_FETCH_TIMEOUT_MS` — optional, default `15000`.
- `JINA_API_KEY` — optional; only consulted when backend is `jina`.

Channel-specific variables (including the owner's ID on that platform) are listed in each channel recipe under `spec/channels/`. Tool-specific variables are listed in each tool recipe under `spec/tools/`.

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

### 1a. Initialize Git repos for agent/, config/, memory/

For each of `agent/`, `config/`, `memory/`: if the directory does not already contain a `.git/`, run `git init` and make an initial commit. This is **required for `agent/`** — the safe-restart auto-revert relies on `HEAD` existing. Without this, a self-edit that crashes on startup can't be rolled back.

Per-domain specifics:

- **`agent/`** — write `agent/.gitignore` (ignore `node_modules/`, `*.log`, `*.pid`). Then `git add -A && git commit -m "bootstrap"`.
- **`config/`** — write `config/.gitignore` that ignores `.env`, `.env.*` (and allows `.env.example` if used). **Never commit `.env`** — it contains secrets. After writing the gitignore and the `.env` template, `git add .gitignore && git commit -m "initial config"`. The `.env` file stays untracked.
- **`memory/`** — write `memory/.gitignore` (ignore `.obsidian/` for Obsidian vault settings). `git add -A && git commit -m "initial state"`. If `memory/` is empty, commit just the gitignore.

If any of these directories already has a `.git/`, leave it alone — the user is managing it.

**Commit additional changes as you make them.** If the bootstrap process edits generated files after the initial commit (e.g., you notice a bug and fix it), make a follow-up commit. End the bootstrap with a clean working tree in each repo.

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
  6. **A "How you operate" block** — a fixed paragraph explaining the agent's invocation model so it understands its own architecture. Include verbatim:

     ```
     You run continuously and can be invoked in two ways:

     - **User messages** from connected channels (e.g. Telegram, terminal chat). When the user sends a message, you receive it as the latest entry in the conversation history and respond.

     - **Scheduled cron jobs.** The scheduler periodically injects a synthetic prompt from a job definition (see "Scheduled tasks" below). Cron is your mechanism for proactive outreach — when a heartbeat fires, that's your chance to reach out to the user. To send a proactive message, just respond normally; your reply is delivered to the user's primary messaging channel. Reply with exactly `NO_REPLY` if there's nothing worth saying.
     ```

     This block is static — the same text every invocation. Without it the agent doesn't know that cron exists, doesn't realize heartbeats are the proactive-outreach mechanism, and may incorrectly tell users it can't reach out unprompted.

  7. **Active cron jobs** — a flat list of `name • schedule • summary` for each enabled merged job (the same merged set the scheduler loaded from `spec/cron/` + `config/cron/`). Summary is the frontmatter `description:` field if present, else the first H1 heading in the body, else the first ~100 chars of body. Skip disabled jobs. This makes the agent aware of what's scheduled to fire and what each job is for, so it can recognize cron-originated prompts when they arrive.
- The agent receives conversation history (the current message is already the last entry). Pass it directly to the SDK as the messages array.
- Cap conversation history at 50 messages (most recent) to avoid blowing past token limits. Apply the cap when retrieving history, not in the agent.
- Guard against oversized prompts. Estimate tokens using a 4:1 character-to-token ratio. First, truncate any individual message over 10,000 tokens. Then, if the total (system prompt + messages) exceeds 150,000 tokens, drop the oldest messages until it fits.

#### 4a. First-run flow

If `config/SOUL.md` doesn't exist when the agent assembles its system prompt:

- Use a minimal bootstrap system prompt: "You are a new personal AI assistant named Logos. You haven't been configured yet. On your next reply, introduce yourself and ask the user (1) what to call yourself and (2) how you should act — personality, tone, style. Once they answer, use `write_file` with `mode: \"create\"` to create `config/SOUL.md` with the chosen name and personality. Nothing else belongs in that file — memory, skills, and other context are loaded separately."
- After the user answers, the agent writes `config/SOUL.md`. Subsequent invocations read it normally.
- Also ensure `config/`, `memory/`, and `runtime/` directories exist; create them if not.

Tools live in `agent/src/tools/`. Each built-in tool has a recipe in `spec/tools/{name}.md` describing its inputs, outputs, behavior, and dependencies. Implement one tool per recipe.

Bundled tools to build:

- `read_file` — see `spec/tools/read_file.md`
- `write_file` — see `spec/tools/write_file.md`
- `edit_file` — see `spec/tools/edit_file.md`
- `find_memory` — see `spec/tools/find_memory.md`
- `remember` — see `spec/tools/remember.md`
- `shell` — see `spec/tools/shell.md`
- `delegate_task` — see `spec/tools/delegate_task.md` (sub-agent runner)
- `web_fetch` — see `spec/tools/web_fetch.md` (uses backend specified by `LOGOS_WEB_FETCH_BACKEND`)

**Conventions all tools follow:**

- **Path-safety helper** — share a single `agent/src/tools/_paths.ts` (or similar) used by `read_file`, `write_file`, `edit_file`, and any other path-taking tool. It resolves to absolute and rejects anything escaping the workspace root.
- **Self-edit guards** — `write_file` and `edit_file` route through that helper; when `LOGOS_SELF_EDIT=false` and the resolved path is under `agent/`, throw `self-edit is disabled; refusing to write under agent/`. The `shell` tool gets a description nudge under the same env var.
- **Discriminated unions for succeed-or-miss results** (see `find_memory`, `web_fetch`). Never return `null` from a tool. See `ARCHITECTURE.md` → Tool return shapes.
- **Memory graph cache invalidation** — any tool that writes under `memory/` (`write_file`, `edit_file`, `remember`) must invalidate `runtime/memory-graph.json`.

Custom tools are added by dropping `.ts` files directly into `agent/src/tools/` alongside the built-in ones. Loader scans that single directory. Custom tools that warrant documentation can have a colocated `.md` (same pattern as channels), but it's not required.

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

#### 4c. Self-edit enforcement

Self-edit is controlled by the `LOGOS_SELF_EDIT` env var (default `true`). Read it once at startup and thread the boolean through the tool loader and skills loader.

When `LOGOS_SELF_EDIT=false`:

- **Skills loader skips `self-edit`.** When walking `spec/skills/`, filter out the `self-edit` directory before registering. The agent never sees the skill in its system prompt.
- **`write_file` and `edit_file` refuse writes under `agent/`.** In the path-safety helper that both tools share, add a check: resolve the target path to an absolute path; if it lives under `{workspace}/agent/`, throw with a clear message ("self-edit is disabled; refusing to write under agent/"). The tools can still write to `config/`, `memory/`, and `runtime/`.
- **`shell` tool description gets a nudge.** When disabled, the `shell` tool's description (what the model sees) includes: "NOTE: self-edit is disabled. Do not modify files under `agent/`." This is best-effort — shell can still technically write anywhere the process user can. Document this limitation in the tool description itself so the model knows it's convention, not enforcement.

**These are application-level guards, not OS-level enforcement.** A determined agent with `shell` can bypass them. For guaranteed enforcement, sandbox the process — see the Sandboxing section at the end of this document.

#### 4d. Sub-agent runner

The `delegate_task` tool (see `spec/tools/delegate_task.md`) lets the main agent spawn a focused sub-agent in an isolated context. Implement the runner at `agent/src/agents/runner.ts`:

- **`runSubAgent({ prompt, skills, tools, model })`** — single exported function.
- **Resolve skills:** for each name in `skills`, find the matching `SKILL.md` (search `spec/skills/` then `config/skills/`; config wins on collision). Read the FULL body. Return `{ ok: false, error }` if any skill is missing.
- **Resolve tools:** look up each tool name in the loaded tools map (the same map `agent.ts` builds for the main agent). **Always strip `delegate_task` from the resolved set** — sub-agents don't get to spawn sub-agents (single-level delegation, period). Return `{ ok: false, error }` if any other tool is missing.
- **Build the system prompt:** concatenate, in order:
  1. A framing line: `"You are a focused sub-agent invoked by the main Logos agent. Complete the task described in the user message and return your final response. You have no access to the main agent's identity, memory, or conversation history beyond what's stated below."`
  2. Today's date.
  3. The full body of each loaded skill (separated by blank lines).
- **Call `generateText`** with the system prompt, the `prompt` argument as the user message, the resolved tool subset, the model from the argument or `AI_MODEL` env, and a step limit equal to the main agent's (e.g. `stepCountIs(25)` to start).
- **Log the invocation** to `runtime/logs/` (or a sub-agents-specific logfile) with: timestamp, skills, tools, model, duration, token usage. Do NOT include the prompt or response text in the log (could contain sensitive content). Operator-only visibility.
- **Return** `{ ok: true, response: text }` on success or `{ ok: false, error: message }` on any thrown error during resolution or execution.

The sub-agent does **NOT** receive: SOUL.md, the memory manifest, recent journal entries, or the main agent's conversation history. Isolation is the point.

### 5. Build the channel registry

- Scan `agent/src/channels/` for `*.ts` files at startup. For each, dynamically import and call its `register()` function with the router. No manual registration list — channels are discovered. Both built-in channels (generated from spec recipes) and custom channels live in the same directory.
- A channel's `register()` returns the channel's ID, owner conversation ID, and send function if it connected successfully — or nothing if credentials were missing and it skipped. When skipping, log which environment variables are missing and how to get them (see the recipe in `spec/channels/{name}.md`).
- The registry collects connected channels into a map by channel ID so the scheduler can look up any channel's send function and owner conversation ID
- If no channels connected, the process should exit with a clear error — there's nothing to connect to.

### 6. Build the user's chosen channel

Read the recipe at `spec/channels/{name}.md` for the channel the user chose and follow its setup instructions. The implementation goes at `agent/src/channels/{name}.ts`. The channel must only forward messages from the owner (identified by the owner ID in the recipe's environment variables) and silently ignore everyone else. Get the full loop working end-to-end before adding anything else.

### 6b. Build the terminal channel (always included)

Terminal is a zero-dependency, bundled channel shipped with every bootstrap. It enables local CLI access to the running daemon via `agent/logos chat`. Follow the recipe at `spec/channels/terminal.md` for the full protocol. Key implementation points:

- **Server side (`agent/src/channels/terminal.ts`):** Binds a Unix domain socket at `runtime/logos.sock` (configurable via `LOGOS_TERMINAL_SOCKET`). Accepts multiple simultaneous client connections. Each client negotiates a starting cursor via `{"from": N}` then receives replay messages, a `live-start` marker, and live `message` events as the JSONL file grows. Implement cursor-driven replay by reading the thread JSONL with line numbers and watching the file via `fs.watch` for live updates.
- **Channel ID `terminal`, conversation ID `cli`** (single persistent thread). Owner filtering is by filesystem access to the socket — no env var needed.
- **Client side (`agent/src/cli/chat.ts`):** Isolated module. Must NOT import from `agent/src/router.ts`, `agent.ts`, `memory.ts`, or any other engine code. It should only import `node:net`, `node:fs/promises`, `node:readline`, and the shared protocol types from a new `agent/src/channels/terminal-protocol.ts` file. The client reads stdin lines, sends `{"type": "message", "text": ...}` to the socket, and prints replies from the socket to stdout. Handles `/quit`, `/exit`, and `Ctrl+D` as clean disconnects.
- **Cursor persistence:** `runtime/clients/chat.cursor` stores the last-seen index. On start, client reads the cursor file, sends `{"from": N}`; when no cursor file exists, default behavior replays the last 20 messages (client sends `{"from": max(0, total - 20)}`).
- **Flags:** `--from-start` (replay all), `--last N` (replay last N), `--new` (advance cursor to end without replay), `--session NAME` (use a different cursor file, still writing to the same `cli` thread).
- **Daemon-not-running check:** `agent/logos chat` should check if `runtime/logos.pid` is alive. If not, exit with a clear message. Don't auto-start.

The `LOGOS_TERMINAL=false` env var disables the channel (same pattern as self-edit toggle): the server doesn't bind the socket, and the channel registry skips it.

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

Add a CLI subcommand `agent/logos cron` that prints the merged job table with source annotations (`[spec]`, `[config]`, `[spec → overridden by config]`, `[disabled]`).

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
- `agent/logos cron` — show the merged cron job table
- `agent/logos chat [flags]` — connect a terminal client to the running daemon (see step 6b). Passes flags through to `npx tsx agent/src/cli/chat.ts`. Fails fast with "daemon not running; start it with `agent/logos start`" if the daemon PID isn't alive.

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
- `git -C agent log --oneline` shows at least one commit (the `bootstrap` anchor — critical for safe-restart auto-revert)
- `git -C agent status` is clean (no uncommitted changes)
- `git -C config log --oneline` shows at least one commit; `config/.env` is NOT in the commit
- `git -C memory log --oneline` shows at least one commit

## Sandboxing (optional, for hard self-edit enforcement)

The `LOGOS_SELF_EDIT=false` mode (step 4c) uses tool-level guards — soft enforcement. A determined agent with `shell` access can bypass them. For guaranteed enforcement, run the process in a sandbox with `agent/` mounted or marked read-only. Pick the approach that fits your OS:

**Linux (read-only bind mount):**

```bash
# Make agent/ read-only for the process via a bind mount.
sudo mount --bind agent/ agent/
sudo mount -o remount,ro,bind agent/
```

Or use `bwrap` / `firejail` to namespace-sandbox the process with a read-only view of `agent/`.

**macOS (`sandbox-exec`):**

```bash
# sandbox.sb denies all writes under agent/src/
# (partial profile — you'd build on the default policy)
(version 1)
(allow default)
(deny file-write*
  (subpath "/absolute/path/to/workspace/agent/src"))

sandbox-exec -f sandbox.sb agent/logos start
```

**Either platform (separate user):**

Run the agent as a user that only has read access to `agent/` on the filesystem. Owner owns `agent/`; agent user can read + execute but not write.

None of these are wired into the stock build — they're deployment choices for users who need hard enforcement. Document in your own operations notes.

## When you're done

Remove the reference to this file from `AGENTS.md` and `CLAUDE.md`. The build is complete.
