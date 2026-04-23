# Build

Step-by-step instructions for building Protos from the spec. Read `architecture.md` first — it defines the contracts this document tells you to implement.

All paths in this document are **relative to the workspace root** (the directory that contains `spec/`, `agent/`, `config/`, etc.). Run all commands from there.

The bootstrap reads from `spec/` (recipes, skills, cron defaults, this document) and generates code in `agent/`. After bootstrap, the running agent reads from `spec/` (skills, cron) and `agent/` (its own implementation), with `config/` for instance overrides.

## Before you start

Ask the user for:

- **Primary channel** (required) — Which messaging platform? (Telegram, WhatsApp, Discord, Slack, etc.)
- **AI model** (optional) — Which model to use. Defaults to the latest Claude Sonnet. Any Claude or GPT model works out of the box.

Don't worry about the assistant's name or personality — those are configured on first run through the messaging channel itself. The agent prompts the user and writes `config/SOUL.md` automatically.

## Key packages

- `ai` — **Vercel AI SDK**. Use the latest version. Provides `generateText` with built-in tool execution. Limit the number of tool-use steps to prevent runaway loops. Do not manually implement a tool loop.
- `@ai-sdk/anthropic` and `@ai-sdk/openai` — AI SDK providers, both installed by default so users can switch between Claude and GPT via `AI_MODEL` alone. Other providers (`@ai-sdk/google`, etc.) can be added as needed.
- `js-yaml` — YAML parsing for cron frontmatter and skill frontmatter
- `node-cron` — cron expression scheduling
- `tsx` — TypeScript execution without a build step. The agent can modify its own source and restart to apply changes.
- `zod` — schema validation, required by the AI SDK for tool parameter definitions
- Channel-specific libraries — see the chosen recipe in `spec/channels/`

**Do not use `dotenv` or `dotenvx`** to load `config/.env` — both packages print advertising lines to stdout on load (`tip: ⌁ auth for agents`…). Use Node's built-in `--env-file=config/.env` flag instead, or load the file manually (read, parse `KEY=value` lines, populate `process.env`). No noisy console output, one fewer dependency.

## Environment variables

All secrets (API keys, bot tokens) go in `config/.env`. This file is gitignored (the entire `config/` directory is gitignored by the spec repo). Load it at startup so the values land in `process.env` before anything else runs.

At minimum:

- `ANTHROPIC_API_KEY` and/or `OPENAI_API_KEY` — at least one must be set; which one depends on the `AI_MODEL` you choose.
- `AI_MODEL` — model to use. Default to a sensible current Claude model; don't pin exact version strings since model names change frequently. The provider is inferred from the model ID (`claude-*` → Anthropic, `gpt-*`/`o1*`/`o3*`/`o4*` → OpenAI). If inference is ambiguous, set `AI_PROVIDER` explicitly (`anthropic` or `openai`).
- `PRIMARY_CHANNEL` — the channel used for the owner's main conversation (e.g. `telegram`). The scheduler sends replies here.
- `PROTOS_SELF_EDIT` — `true` (default) or `false`. See `architecture.md` → Self-modification for the toggle's effects.
- `PROTOS_TERMINAL` — `true` (default) or `false`. When false, the terminal channel doesn't bind the socket and `agent/protos chat` has no daemon to connect to.
- `PROTOS_TERMINAL_SOCKET` — optional override for the terminal socket path. Defaults to `runtime/agent.sock`.
- `PROTOS_WEB_FETCH` — `true` (default) or `false`. When false, the `web_fetch` tool refuses every call.
- `PROTOS_WEB_FETCH_BACKEND` — `fetch` (default), `jina`, or `playwright`. See `spec/tools/web_fetch.md` for tradeoffs and privacy considerations.
- `PROTOS_WEB_FETCH_TIMEOUT_MS` — optional, default `15000`.
- `JINA_API_KEY` — optional; only consulted when backend is `jina`.

Channel-specific variables (including the owner's ID on that platform) are listed in each channel recipe under `spec/channels/`. Tool-specific variables are listed in each tool recipe under `spec/tools/`.

## Step-by-step

**Note:** Default cron jobs (`spec/cron/heartbeat.md`, `spec/cron/nap.md`, `spec/cron/dream.md`) and bundled skills (`spec/skills/`) ship with the spec. The running agent reads these directly. Don't copy them into `agent/`.

### 1. Initialize the project

- Create `agent/package.json` and `agent/tsconfig.json`
- Target ES2022 with Node module resolution
- Keep configuration minimal
- Install packages with `npm install <package-name>` from inside `agent/` rather than writing `package.json` by hand — this ensures you get the latest versions and only lists direct dependencies. **Never manually edit the `dependencies` or `devDependencies` objects in `package.json`.**
- Source code goes in `agent/src/` per the file structure in `architecture.md`.
- Use `process.cwd()` for the workspace root path, not `import.meta.dirname` — tsx runs in CJS mode where `import.meta.dirname` is undefined. The wrapper script ensures the process runs with the workspace root as cwd.
- Create `config/` if it doesn't exist (the agent should do this on first run, but the build can pre-create it). Create a `config/.env` template with `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `AI_MODEL`, `PRIMARY_CHANNEL`, and any channel-specific variables. Leave secrets blank for the user to fill in.

### 1a. Initialize Git repos for agent/, config/, memory/

For each of `agent/`, `config/`, `memory/`: if the directory does not already contain a `.git/`, run `git init` and make an initial commit. This is **required for `agent/`** — the safe-restart auto-revert relies on `HEAD` existing. Without this, a self-edit that crashes on startup can't be rolled back. See `architecture.md` → Git repos per domain for the rationale.

Per-domain specifics:

- **`agent/`** — write `agent/.gitignore` (ignore `node_modules/`, `*.log`, `*.pid`). Then `git add -A && git commit -m "bootstrap"`.
- **`config/`** — write `config/.gitignore` that ignores `.env`, `.env.*` (and allows `.env.example` if used). **Never commit `.env`** — it contains secrets. After writing the gitignore and the `.env` template, `git add .gitignore && git commit -m "initial config"`. The `.env` file stays untracked.
- **`memory/`** — write `memory/.gitignore` (ignore `.obsidian/` for Obsidian vault settings). `git add -A && git commit -m "initial state"`. If `memory/` is empty, commit just the gitignore.

If any of these directories already has a `.git/`, leave it alone — the user is managing it.

**Commit additional changes as you make them.** If the bootstrap process edits generated files after the initial commit (e.g., you notice a bug and fix it), make a follow-up commit. End the bootstrap with a clean working tree in each repo.

### 2. Set up message storage

Implement `agent/src/threads.ts`. Event schema, `turn_id` grouping, replay semantics, and the render filter are defined in `architecture.md` → Storage / Message history (Event schema).

Functions to export:

- `appendEvent(channelId, conversationId, event)` — append one event to the conversation's JSONL (creating the directory if missing). Sanitize `channelId` and `conversationId` to safe path segments — reject anything with `/`, `..`, or other path-escape characters. Accepts any variant of the event discriminated union.
- `readEvents(channelId, conversationId, limit?)` — read the file (return `[]` if missing), parse each non-blank line as JSON, drop malformed lines, optionally slice to the last `limit` events with **turn-boundary backward-alignment**: if the naïve cut point `P = events.length - limit` lands mid-turn (i.e. `events[P-1].turn_id === events[P].turn_id`), decrement `P` until that's no longer true, so the returned window starts at the first event of its turn. The returned window may be slightly larger than `limit` (by up to one turn), never smaller. This preserves the tool-call/tool-result pairing invariant (see `architecture.md` → Tool return shapes) — a forward-aligned slice could leave orphan tool results at the head, which the LLM SDK rejects with `Each tool_result block must have a corresponding tool_use block in the previous message`.
- `buildLlmMessages(events)` — given an ordered event list, produce the AI SDK `CoreMessage[]` per the rules in `architecture.md` → LLM context (replay): each event becomes one `CoreMessage` in order, user events get the send-time annotation, audit events are skipped, adjacent same-role messages are collapsed. This is the **correctness fix** that lets the model see the tool calls it actually made.
- `renderForChannel(events)` — apply the render filter: for each `turn_id`, emit user events as-is and the **last** assistant text of the turn (skipping assistant events whose `text` is empty OR equals the literal `NO_REPLY` lifecycle marker); drop intermediate assistant texts, assistant events' `tool_calls`, `role: "tool"` events, and `role: "audit"` events. Used by watch-based channels (e.g. terminal). Push-based channels don't call this.

The router serializes per-conversation, so `appendEvent` never has a concurrent writer on the same file. No locking needed.

**Cron logs.** The same `appendEvent` / `readEvents` primitives work for cron logs at `runtime/logs/cron/{jobname}/{ISO}.jsonl` — expose a variant (e.g. `appendCronEvent(jobname, timestamp, event)`) that routes to the right path. Sub-agent logs work the same way at the nested paths defined in `architecture.md` → Sub-agents.

### 3. Build the router

Implement `agent/src/router.ts`. Behavior contract is in `architecture.md` → Router (especially the `NO_REPLY` semantics and the [Channel `send()` contract](architecture.md)).

Implementation:

- Per-conversation queue keyed on the composite `(channelId, conversationId)` so each conversation runs serially. Don't key on `conversationId` alone — two channels could happen to share the same conversation-id string and end up sharing a queue.
- **`turn_id` assignment.** User events and the assistant invocation that responds get **different** `turn_id`s. Every event from the same invocation (the user message, or the assistant's multi-step response) shares one `turn_id`. Generate a fresh id at each invocation boundary (uuid or a timestamp-based id).
- For each inbound message: generate a `turn_id`, append the user event via `threads.appendEvent`, read recent events via `threads.readEvents`, build LLM messages via `threads.buildLlmMessages`, call the agent (passing a fresh `turn_id` for its invocation).
- The agent runs its multi-step loop. As it produces assistant steps (text + tool calls) and tool results, append each as its own event with the agent's `turn_id`.
- After the agent returns, the final assistant text has already been appended. Extract it and call `channel.send(conversationId, reply)`.
- The reply is stored and sent **as-is** — including the literal string `NO_REPLY`. No translation. Channels detect `NO_REPLY` themselves per the contract.

### 4. Build the agent

Use the Vercel AI SDK's `generateText` for automatic tool execution. Cap the multi-step tool loop at **`stepCountIs(25)`** to prevent runaway loops. Do not manually implement a tool loop. Sub-agents inherit the same cap.

- Select the provider from `AI_MODEL` per the mapping in **Environment variables** above; honor `AI_PROVIDER` as the override.
- Read the model from the `AI_MODEL` environment variable with a sensible default.
- The agent receives conversation history (the current message is already the last entry). Pass it directly to the SDK as the messages array.
- Cap conversation history at the 100 most recent events (backward-aligned to a turn boundary — see `readEvents` above) to avoid blowing past token limits. Apply the cap when retrieving history, not in the agent. 100 gives comfortable slack over `nap`'s 50-event consolidation threshold so unconsolidated events stay in the agent's working context between `nap` runs.
- Guard against oversized prompts. Estimate tokens using a 4:1 character-to-token ratio. First, truncate any individual message over 10,000 tokens. Then, if the total (system prompt + messages) exceeds 150,000 tokens, drop the oldest messages until it fits.

#### System prompt assembly

The system prompt is concatenated from these sections, in order:

1. **`spec/agent.md`** — the base prompt (role, behavior, tool/skill conventions, security).
2. **`config/SOUL.md`** (identity) — if missing, run the first-run flow (see step 4a).
3. **Memory manifest** — see `architecture.md` → Memory format → Loading into context. Build it from the memory module's manifest output (step 4b).
4. **Last 24 hours of `memory/journal/` entries inline** — recent agent-authored notes likely to be relevant; older journal entries appear in the manifest only.
5. **Skills summary** — pulled from `spec/skills/*.md`, `config/skills/*.md`, and `config/skills/*/SKILL.md` frontmatter; config merges with spec on name collision (see step 4b). Each entry is formatted as `- <name> (<paths joined with " + ">) — <description>` so the agent can open the full skill body via `read_file` — for a merged skill, both source files together.

#### 4a. First-run flow

If `config/SOUL.md` doesn't exist when the agent assembles its system prompt:

- Use a minimal bootstrap system prompt: "You are a new personal AI assistant named Protos. You haven't been configured yet. On your next reply, introduce yourself and ask the user (1) what to call yourself and (2) how you should act — personality, tone, style. Once they answer, use `write_file` with `mode: \"create\"` to create `config/SOUL.md` with the chosen name and personality. Nothing else belongs in that file — memory, skills, and other context are loaded separately."
- After the user answers, the agent writes `config/SOUL.md`. Subsequent invocations read it normally.
- Also ensure `config/`, `memory/`, and `runtime/` directories exist; create them if not.

#### Tools

Implement one tool per recipe under `spec/tools/`. Each recipe is the contract for that tool — inputs, outputs, behavior, dependencies.

Bundled tools to build (each has a recipe):

- Core file/shell: `read_file`, `write_file`, `edit_file`, `remember`, `shell`
- Memory-aware create/find/rename (wiki-link-form input, resolution-preserving): `find_memory`, `add_memory`, `rename_memory`
- Agent control: `delegate_task`, `web_fetch`
- Consolidation (used by `nap`/`dream` crons): `list_threads`, `read_thread_tail`, `advance_thread_cursor`
- Memory hygiene: `find_orphans`

Implementation conventions:

- **Paths helper** — share a single module (`agent/src/paths.ts`) used by every path-taking tool. The helper resolves to absolute, rejects paths escaping the workspace root, and enforces the `spec/` and `agent/` write guards (see `architecture.md` → Self-modification for what those guards are and when they apply).
- **Memory graph cache invalidation** — any tool that writes under `memory/` (`write_file`, `edit_file`, `remember`, `add_memory`, `rename_memory`) must delete `runtime/memory-graph.json` so the next graph operation rebuilds.
- **Resolution-preservation helper** — `add_memory` and `rename_memory` share the same pre/post snapshot → rewrite-changed-resolutions algorithm. Extract it as a single helper in `agent/src/memory.ts` and call it from both tools.
- **Tool return shapes** follow `architecture.md` → Tool return shapes. Never return bare `null` from a tool, and honor the call/result pairing invariant (error-catching adapter at registration).

Custom tools are added by dropping `.ts` files directly into `agent/src/tools/` alongside the built-in ones. Loader scans that single directory.

#### 4b. Memory module

Implement `agent/src/memory.ts`. Resolution rules, manifest format, backlink semantics, and Obsidian compatibility are defined in `architecture.md` → Memory format.

The module exports:

- **Link resolver** — given a name (optionally path-qualified), return the matching file path or `null` per the resolution rules. Does NOT auto-create.
- **Graph builder** — at startup, scan `memory/**/*.md`, parse frontmatter and `[[...]]` links, build a name index and a backlink index covering the full tree. Build manifest entries for **root-level files only** (`memory/*.md`, not recursing into subfolders) — subfolder files are reachable via `[[wiki-links]]` and don't appear in the system prompt.
- **Cache** — write the graph to `runtime/memory-graph.json`. On startup, check the cache: if all source file mtimes are ≤ the cache's mtime, use it; otherwise rebuild.
- **Frontmatter parser** — use `js-yaml` to parse the YAML block between `---` delimiters. Tolerate missing or malformed frontmatter (treat as empty).

Skills loader:

- `spec/skills/`: scan for `*.md` files. The filename (minus `.md`) is the skill name.
- `config/skills/`: scan for both `*.md` files (filename = skill name) and subdirectories containing `SKILL.md` (directory name = skill name). The directory form is for off-the-shelf agentskills.io skills the user drops in unmodified.
- Extract YAML frontmatter with `js-yaml` (not regex).
- **Within `config/`**, if both flat (`{name}.md`) and directory (`{name}/SKILL.md`) forms exist for the same name, the flat form wins. Within-config collisions are not merged.
- **Across roots**, when the same name exists in spec and config, merge:
  - **Frontmatter** — per-field. Config wins on collision; spec fills in keys not present in config. No deep merge — array and map fields replace wholesale on collision.
  - **Body** — spec body + `\n\n` + config body.
  - Spec-only or config-only skills pass through unchanged.
- Each skill entry tracks the source paths it was assembled from (one for unmerged, two for merged) so the system prompt summary can show both.

#### 4c. Self-edit enforcement

Read `PROTOS_SELF_EDIT` once at startup and thread the boolean through the tool loader and skills loader. Behavior is defined in `architecture.md` → Self-modification.

What you must implement:

- **Always-on `spec/` write guard** in the paths helper: any path under `{workspace}/spec/` throws `spec/ is read-only at runtime; instance-specific changes belong in config/`. Independent of `PROTOS_SELF_EDIT`.
- **Conditional `agent/` write guard** when `PROTOS_SELF_EDIT=false`: paths under `{workspace}/agent/` throw `self-edit is disabled; refusing to write under agent/`.
- **Skills loader filter** when `PROTOS_SELF_EDIT=false`: skip `spec/skills/self-edit.md` so the skill is hidden from the agent's prompt.
- **Shell tool description nudges**: always include the `spec/` warning; conditionally append the `agent/` warning when `PROTOS_SELF_EDIT=false`. These are conventions, not enforcement.

#### 4d. Sub-agent runner

Implement `agent/src/agents/runner.ts`. Tool contract is in `spec/tools/delegate_task.md`; design rationale and constraints are in `architecture.md` → Sub-agents.

The runner takes the `delegate_task` inputs (prompt, skills, tools, optional model) plus enough context to write its log to the right place under the caller's log directory: a stable id for this particular call, and a parent-context discriminator describing whether the call originated from a cron run (carrying the job name and run timestamp) or a thread turn (carrying the channel id, conversation id, and turn id). The `delegate_task` tool wrapper knows which one applies and plumbs it through.

Behavior:

- **Resolve skills** by name from `spec/skills/` then `config/skills/` (config wins). Read the FULL skill body — the `{name}.md` file or the `{name}/SKILL.md` body, whichever the loader registered. On any missing skill, fail the call.
- **Resolve tools** from the loaded tools map (the same map `agent.ts` builds). **Always strip `delegate_task`** — sub-agents never recurse, regardless of caller request.
- **Build the system prompt:** framing line + today's date + concatenated skill bodies (NO SOUL.md, NO memory manifest, NO conversation history).
- **Compute the sub-agent log path** from the parent context and call id:
  - Cron parent → `runtime/logs/cron/{jobname}/{timestamp}/{callId}.jsonl`
  - Thread parent → `runtime/logs/sub-agents/{channelId}/{conversationId}/{turn_id}-{callId}.jsonl`
- **Call `generateText`** with the assembled system prompt, the prompt as the user message, the resolved tool subset, the model from the argument or `AI_MODEL`, and the same `stepCountIs(25)` cap as the main agent. Append every event the sub-agent produces (user prompt, assistant steps, tool calls, tool results) to the sub-agent log using the same event schema as threads and cron logs.
- **Return** the final assistant text together with the workspace-relative log path. The tool-loop machinery will include this return value in the parent's `tool_result` event automatically — see `spec/tools/delegate_task.md` for the exact return shape.

Framing line for the system prompt:

> "You are a focused sub-agent invoked by the main Protos agent. Complete the task described in the user message and return your final response. You have no access to the main agent's identity, memory, or conversation history beyond what's stated below."

### 5. Build the channel registry

Implement `agent/src/channels/index.ts`. Channels are defined in `architecture.md` → Channels (especially the [`send()` contract](architecture.md)).

- Scan `agent/src/channels/` for `*.ts` files at startup. For each, dynamically import and call its `register()` function with the router. No manual registration list.
- A channel's `register()` returns `{ channelId, ownerConversationId, send }` if it connected, or nothing if credentials were missing. When skipping, log which environment variables are missing and how to get them (see the recipe in `spec/channels/{name}.md`).
- The registry collects connected channels into a map by channel ID so the scheduler can look up any channel's send function and owner conversation ID.
- If no channels connected, exit with a clear error.

### 6. Build the user's chosen channel

Read the recipe at `spec/channels/{name}.md` and follow its setup instructions. The implementation goes at `agent/src/channels/{name}.ts`. The channel must only forward messages from the owner (identified by the owner ID in the recipe's environment variables) and silently ignore everyone else. Get the full loop working end-to-end before adding anything else.

### 6b. Build the terminal channel (always included)

Terminal is a zero-dependency, bundled channel shipped with every bootstrap. Follow `spec/channels/terminal.md` for the protocol, replay semantics, and rendering rules. Implementation lives at `agent/src/channels/terminal.ts` (server) and `agent/src/cli/chat.ts` (client). Wired into the wrapper as `agent/protos chat` (see step 9).

The `PROTOS_TERMINAL=false` env var disables the channel: the server doesn't bind the socket, and the channel registry skips it.

### 7. Build the scheduler

Implement `agent/src/scheduler.ts`. Cron format, merge rules, the merged-job table, and the per-run log layout are defined in `architecture.md` → Scheduler.

- Scan both `spec/cron/` and `config/cron/`, parse frontmatter with `js-yaml`, merge per the layering rules (frontmatter override, body append; skip when merged `enabled: false`).
- Use `node-cron` to schedule each enabled job using its merged `schedule` field.
- When a job fires:
   1. **Bootstrap guard.** If `config/SOUL.md` doesn't exist, skip the firing entirely — log a one-line note and return. The agent isn't ready to respond to cron prompts; the next firing after bootstrap works normally.
   2. Generate a run timestamp `ISO` and open the cron log at `runtime/logs/cron/{jobname}/{ISO}.jsonl`.
   3. Append a `cron_start` audit event (job name, schedule, `turn_id`, timestamp).
   4. Append the cron reminder (see `architecture.md` → Scheduler) to the merged body. Then dispatch through the router as a synthetic message. Pass the cron log file handle so the router can append each agent/tool event to it.
   5. After the agent returns, append a `cron_end` audit event (reply text, `duration_ms`, timestamp).
- **Honor the `history:` frontmatter field** (default `primary`). The field controls only what the agent **reads** as context: `primary` reads recent primary-thread events; `none` runs with only the synthetic prompt. **Writes are the same in both cases**: the full event stream goes to the cron log; only the final assistant reply goes to the user thread (skipped on `NO_REPLY`). The synthetic prompt and intermediate steps are NOT written to the user thread.
- Add a CLI subcommand `agent/protos cron` that prints the merged job table with source annotations (`[spec]`, `[config]`, `[spec → overridden by config]`, `[disabled]`).

### 8. Wire it all together

The entry point (`agent/src/index.ts`):

1. Loads `config/.env`
2. Ensures `runtime/` exists
3. Registers channels
4. Starts the scheduler
5. Logs that it's running

That's it. No HTTP server needed unless a channel requires a webhook.

### 9. Create the `agent/protos` wrapper script

Create a bash script at `agent/protos` that supports:

- `agent/protos` (no args) — print a usage message listing the subcommands and exit.
- `agent/protos start` — start the process in the background. Before launching, validate that `AI_MODEL` is set in `config/.env` and that the API key for its provider (see **Environment variables**) is non-empty. If not, print a clear error naming the variable and `config/.env`, then exit non-zero without starting. The check lives in the wrapper so errors surface directly in the user's terminal — the daemon is backgrounded and its stderr only reaches the log file.
- `agent/protos stop` — stop it
- `agent/protos restart` — restart it (with safe-restart protocol below)
- `agent/protos status` — check if it's running
- `agent/protos cron` — show the merged cron job table
- `agent/protos chat [flags]` — connect a terminal client to the running daemon (see step 6b). Passes flags through to `npx tsx agent/src/cli/chat.ts`. Fails fast with "daemon not running; start it with `agent/protos start`" if the daemon PID isn't alive.

The script must be invoked from the workspace root (the parent of `agent/`). It should `cd` to the workspace root if invoked from elsewhere by resolving its own location.

Run the process with `npx tsx agent/src/index.ts` (not compiled JS). This way the agent can modify its own TypeScript source and restart to apply changes — no build step needed.

Use a PID file at `runtime/agent.pid` and write logs to `runtime/logs/`. Both are gitignored. Append to the log file — don't truncate it on restart.

**`stop` must kill the entire process tree, not just the PID file's PID.** `npx tsx agent/src/index.ts` produces a multi-process tree (npm wrapper + tsx node child). If `stop` only kills the root PID, the children get re-parented to init and survive — invisible to the wrapper, still polling channels, still firing scheduled jobs. Each `restart` then leaks a zombie daemon. The wrapper must walk the process tree (e.g. via `pgrep -P` recursive descent — available on both macOS and Linux) and signal every descendant. Send SIGTERM first, give the tree a few seconds to shut down gracefully, then SIGKILL anything still alive.

**Safe-restart protocol for `restart`** (architecture: see `architecture.md` → Self-modification):

1. **Snapshot the current `agent/` HEAD.** If `agent/` is a Git repo (`agent/.git` exists), record the current commit SHA with `git -C agent rev-parse HEAD`. If it's not a Git repo, skip the rollback steps below and warn the user that self-edit rollback won't work.
2. **Typecheck.** Invoke TypeScript from inside `agent/` (where its `node_modules` lives) — e.g. `npm --prefix agent run typecheck` or `(cd agent && npx tsc --noEmit)`. Don't `npx tsc` from the workspace root: there's no `node_modules/.bin/tsc` there, and `npx` will print "This is not the tsc command you are looking for" and fail. If the typecheck fails, abort the restart, keep the old process running, and print the error.
3. **Stop the old process** (if running) and **start the new one** in the background.
4. **Health check.** Wait a few seconds (e.g. 5s) and check if the new PID is still alive. If dead:
   - Print the last ~30 lines of the log so the user can see what crashed.
   - If `agent/` is a Git repo and HEAD has moved since the snapshot: run `git -C agent reset --hard <snapshot-sha>` to revert the self-edit, then start the process again with the pre-edit code. Log the auto-revert clearly (`[agent] post-start check failed; auto-reverted agent to <sha> and restarted`).
   - If no Git repo or HEAD is unchanged: leave the process stopped and tell the user to investigate.

Auto-revert only affects commits the agent itself made between the snapshot and the failed restart. Changes in `config/` or `memory/` are not part of self-edit.

## Before you're done

Verify the build before handing it off:

- `tsc --noEmit` (against `agent/tsconfig.json`) passes with no errors
- `agent/protos start` with blank credentials fails and shows the error in the terminal
- `agent/protos start` with blank credentials then `agent/protos status` reports not running
- `config/SOUL.md` does not exist yet (it's written on first run, not by the build)
- `spec/cron/heartbeat.md`, `spec/cron/nap.md`, and `spec/cron/dream.md` are unchanged
- `spec/` is untouched by the build (the bootstrap only writes to `agent/` and optionally creates `config/`)
- The wrapper script is executable (`chmod +x agent/protos`)
- `git -C agent log --oneline` shows at least one commit (the `bootstrap` anchor — critical for safe-restart auto-revert)
- `git -C agent status` is clean (no uncommitted changes)
- `git -C config log --oneline` shows at least one commit; `config/.env` is NOT in the commit
- `git -C memory log --oneline` shows at least one commit

## When you're done

The build is complete. `AGENTS.md` stays as-is — it's the coding-agent entry point and is needed again for future `update agent` invocations.

## Updating

For the `update agent` command — sync an existing implementation when the spec evolves.

1. **Identify the delta.** Read `spec/` as-is and compare against the current `agent/` tree. `git log` on the spec repo is useful for surfacing recent changes.
2. **Propose the changes** to the user — a per-file summary is more useful than a raw diff. Apply once they approve.
3. **Commit inside `agent/`** with a message summarizing what changed. The commit is required: the safe-restart auto-revert depends on `agent/` having a clean `HEAD` (same reason the bootstrap commits — see step 1a).
4. **Leave `config/`, `memory/`, and `runtime/` alone.** Spec changes regenerate `agent/` only.

End with a clean working tree in `agent/`, same invariant as bootstrap.
