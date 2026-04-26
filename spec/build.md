# Build

Step-by-step instructions for building Protos from the spec. Read `architecture.md` first — it defines the contracts this document tells you to implement.

All paths in this document are **relative to the workspace root** (the directory that contains `spec/`, `agent/`, `config/`, etc.). Run all commands from there.

The build reads from `spec/` (recipes, skills, cron defaults, this document) and generates code in `agent/`. After build, the running agent reads from `spec/` (skills, cron) and `agent/` (its own implementation), with `config/` for instance overrides.

## Before you start

Ask the user for:

- **Primary channel** (required) — Which messaging platform? (Telegram, WhatsApp, Discord, Slack, etc.)
- **Default model** (optional) — Which model to use as the `default` profile in `config/models.yaml`. Defaults to a current Claude Sonnet. Any Claude or GPT model works out of the box; more providers can be added as needed.

Don't worry about the assistant's name or personality — those are configured on first run through the messaging channel itself. The agent prompts the user and writes `config/SOUL.md` automatically.

## Key packages

- `agent-sdk` — **agent-sdk** ([github.com/ExProtos/agent-sdk](https://github.com/ExProtos/agent-sdk)). One API over four runtimes (Claude Agent SDK, Codex AppServer, OpenAI Agents SDK, Vercel AI SDK). Provides `Agent`, `agent.run()`, the canonical tool catalog (`bash`, `read`, `write`, `edit`, `glob`, `grep`, `webFetch`, `webSearch`, `todo`, `task`), and `withImpls()` for plugging in-process executors on backends that consult `execute`. agent-sdk handles the tool execution loop, automatic compaction, and per-backend session storage; do not implement a tool loop or token guard.
  - **Not on npm yet.** Install via the GitHub URL: `npm install github:ExProtos/agent-sdk` (npm clones the repo and runs the upstream `prepare` script to build `dist/` automatically). The resulting `agent/package.json` entry will look like `"agent-sdk": "github:ExProtos/agent-sdk"`. Pin to a specific SHA or tag (`github:ExProtos/agent-sdk#abc1234`) once a stable release exists. **GitHub-URL deps pin the resolved commit in `package-lock.json` forever** — plain `npm install` won't pick up new agent-sdk commits. See Updating → step 1 for the refresh recipe.
- `js-yaml` — YAML parsing for cron frontmatter and skill frontmatter
- `node-cron` — cron expression scheduling
- `tsx` — TypeScript execution without a compile step. The agent can modify its own source and restart to apply changes.
- `zod` — schema validation, used by agent-sdk for tool parameter definitions
- Channel-specific libraries — see the chosen recipe in `spec/channels/`

agent-sdk pulls in the underlying SDKs (`@anthropic-ai/claude-agent-sdk`, the Codex app-server bridge, `@openai/agents`, `ai` + `@ai-sdk/anthropic` / `@ai-sdk/openai` / `@ai-sdk/openai-compatible`) as transitive dependencies — they install with agent-sdk. There's nothing to install separately for the four backends.

**Do not use `dotenv` or `dotenvx`** to load `config/.env` — both packages print advertising lines to stdout on load (`tip: ⌁ auth for agents`…). Use Node's built-in `--env-file=config/.env` flag instead, or load the file manually (read, parse `KEY=value` lines, populate `process.env`). No noisy console output, one fewer dependency.

## Configuration

Per-instance configuration lives in three files under `config/`. See `architecture.md` → Model selection and Channels for the full semantics; this section is the build-time view.

### `config/models.yaml`

LLM profiles. At minimum, a `default` profile must resolve to a concrete backend + model, with backend-specific fields and required env vars set per `architecture.md` → Model selection → Backends.

```yaml
default:
  backend: claude
  model: claude-sonnet-4-6
```

Put a current Claude Sonnet model ID in the template. Don't try to pin a specific version across builds — model names change frequently. `backend:` is required for `claude` / `codex` / `openai`; the `vercel` backend's profile must additionally set `provider:` (`anthropic` / `openai` / `openai-compatible`) and the corresponding fields (`api_key:` for hosted; `base_url:` for openai-compatible).

**Auth setup before first run.** Each backend accepts typed auth fields directly on the profile (preferred — discoverable, IDE-completable, supports multi-account) and falls back to ambient env vars (simpler — fine for the common single-account case).

- **`claude` backend** —
  - *Subscription:* run `claude setup-token` once. Either set `CLAUDE_CODE_OAUTH_TOKEN` in `.env` (ambient fallback) or reference it from a profile's `oauth_token: $CLAUDE_CODE_OAUTH_TOKEN`.
  - *API key:* set `ANTHROPIC_API_KEY` in `.env`, or use `api_key: $ANTHROPIC_API_KEY` per profile.
  - The two are mutually exclusive on a single profile (agent-sdk's `ClaudeBackend` constructor throws).
- **`codex` backend** —
  - *Single account (typical):* run `codex login` once; ambient `~/.codex/auth.json` is read by default.
  - *Multi-account:* per-profile `codex_home: runtime/codex-auth/<name>` in YAML; bootstrap each one with `CODEX_HOME=runtime/codex-auth/<name> codex login` (or `codex login --with-api-key` for an API-key-based profile).
- **`openai` backend** — set `OPENAI_API_KEY` in `.env`, or per-profile `api_key: $OPENAI_API_KEY` (typed). Optional `base_url:`, `organization:`, `project:` for non-default endpoints. No OAuth path; subscription auth requires `codex` instead.
- **`vercel` backend** — per-profile `api_key:` in YAML (typically referenced via `$NAME` from `.env`).

Document this in the user-facing setup notes the build emits.

### `config/channels.yaml`

Messaging channels. One entry per channel the user wants active; `primary:` names the channel used for the owner's main conversation.

```yaml
primary: telegram
telegram:
  bot_token: $TELEGRAM_BOT_TOKEN
  owner_id: 456789
terminal:
  enabled: true
```

`provider:` defaults to the entry name, so a single-instance `telegram:` entry doesn't need it spelled out. Multiple entries of the same provider (`work-telegram:`, `personal-telegram:`) each need `provider: telegram` explicitly.

### `config/.env`

Holds secrets referenced from the YAML files via `$NAME` substitution. Gitignored by the build.

Auth env vars (used either as ambient fallbacks when no typed YAML field is set, or as values referenced by typed fields via `$NAME`):

- `CLAUDE_CODE_OAUTH_TOKEN` — populated by `claude setup-token`. Used by the `claude` backend (subscription).
- `ANTHROPIC_API_KEY` — used by the `claude` backend (metered API) and `vercel` profiles using `provider: anthropic`.
- `OPENAI_API_KEY` — used by the `openai` backend and `vercel` profiles using `provider: openai`. (Codex auth lives in `~/.codex/auth.json`, populated by `codex login`; this env var is not consulted by the codex app-server.)

Self-edit toggling and `spec/` write protection are now OS-level — see `architecture.md` → Self-modification. No `PROTOS_SELF_EDIT` env var.

Channel-specific credentials (including the owner's ID on that platform) are listed in each channel recipe under `spec/channels/` as fields to put in `channels.yaml`. Tool-specific variables are listed in each tool recipe under `spec/tools/`.

### Migrating from legacy env-only deployments

If the user is updating from a pre-YAML deployment (where `AI_MODEL` / `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `PRIMARY_CHANNEL` / per-channel env vars were read directly by the runtime), synthesize `config/models.yaml` and `config/channels.yaml` from those env vars. Default migration target depends on `AI_MODEL`:

- Anthropic model → `backend: claude` with `api_key: $ANTHROPIC_API_KEY` (preserves the existing API-key auth; no behavior change). The user can later flip to `oauth_token: $CLAUDE_CODE_OAUTH_TOKEN` after running `claude setup-token` if they want subscription billing — same backend, just different auth field.
- OpenAI model → `backend: openai` with `api_key: $OPENAI_API_KEY`. Or `backend: codex` if the user wants subscription billing (requires `codex login` once).
- Local / OpenAI-compatible model → `backend: vercel` with `provider: openai-compatible` and `base_url:` / `api_key:` fields.

Reference each secret by `$NAME` rather than inlining, so the existing `.env` keeps holding the credentials. Show the user the generated YAML for review before writing; don't touch the original `.env`. The runtime does NOT read the legacy model/channel env vars — the YAML files do, via substitution.

## Step-by-step

**Note:** Default cron jobs (`spec/cron/heartbeat.md`, `spec/cron/nap.md`, `spec/cron/dream.md`) and bundled skills (`spec/skills/`) ship with the spec. The running agent reads these directly. Don't copy them into `agent/`.

### 1. Initialize the project

- Create `agent/package.json` and `agent/tsconfig.json`
- Target ES2022 with Node module resolution
- Keep configuration minimal
- Install packages with `npm install <package-name>` from inside `agent/` rather than writing `package.json` by hand — this ensures you get the latest versions and only lists direct dependencies. **Never manually edit the `dependencies` or `devDependencies` objects in `package.json`.**
- Source code goes in `agent/src/` per the file structure in `architecture.md`.
- Use `process.cwd()` for the workspace root path, not `import.meta.dirname` — tsx runs in CJS mode where `import.meta.dirname` is undefined. The wrapper script ensures the process runs with the workspace root as cwd.
- Create `config/` if it doesn't exist (the agent should do this on first run, but the build can pre-create it). Write three templates:
  - `config/models.yaml` — a single `default:` profile with `backend: claude` and a current Claude Sonnet model ID. The user runs `claude setup-token` once to populate `CLAUDE_CODE_OAUTH_TOKEN` (subscription billing). The user can edit to pick a different backend (`codex`, `openai`, `vercel`), add providers, or add more profiles. Drop a comment in the template pointing at `architecture.md` → Model selection → Backends.
  - `config/channels.yaml` — `primary:` pointing at the user's chosen channel, one block for that channel with credentials referenced via `$NAME`, and a `terminal: { enabled: true }` block.
  - `config/.env` — template containing the `$NAME` variables the YAML references (e.g. `TELEGRAM_BOT_TOKEN=`) plus stubs for the per-backend auth env vars from the Configuration section (`CLAUDE_CODE_OAUTH_TOKEN=`, `ANTHROPIC_API_KEY=`, `OPENAI_API_KEY=`), blank for the user to fill in.

### 1a. Initialize Git repos for agent/, config/, memory/

For each of `agent/`, `config/`, `memory/`: if the directory does not already contain a `.git/`, run `git init` and make an initial commit. This is **required for `agent/`** — the safe-restart auto-revert relies on `HEAD` existing. Without this, a self-edit that crashes on startup can't be rolled back. See `architecture.md` → Git repos per domain for the rationale.

Per-domain specifics:

- **`agent/`** — write `agent/.gitignore` (ignore `node_modules/`, `*.log`, `*.pid`). Then `git add -A && git commit -m "build"`.
- **`config/`** — write `config/.gitignore` that ignores `.env`, `.env.*` (and allows `.env.example` if used). **Never commit `.env`** — it contains the secrets that `models.yaml` and `channels.yaml` reference via `$NAME`. The two YAML files are NOT auto-gitignored; the user decides whether to commit them based on whether they reference secrets by `$NAME` (safe to commit) or inline values (unsafe unless the repo is private). Default behavior during build: track the YAML files with their `$NAME`-referenced templates. After writing the gitignore and the templates, `git add .gitignore models.yaml channels.yaml && git commit -m "initial config"`. The `.env` file stays untracked.
- **`memory/`** — write `memory/.gitignore` (ignore `.obsidian/` for Obsidian vault settings). `git add -A && git commit -m "initial state"`. If `memory/` is empty, commit just the gitignore.

If any of these directories already has a `.git/`, leave it alone — the user is managing it.

**Commit additional changes as you make them.** If the build process edits generated files after the initial commit (e.g., you notice a bug and fix it), make a follow-up commit. End the build with a clean working tree in each repo.

### 2. Set up message storage

Implement `agent/src/threads.ts`. Event schema, `turn_id` grouping, the per-profile JSONL layout, the continuation sidecar, and the render filter are defined in `architecture.md` → Storage / Message history and Session continuity.

Functions to export:

- `appendEvent(channelId, conversationId, profile, event)` — append one event to `runtime/threads/{channelId}/{conversationId}/{profile}.jsonl` (creating the directory if missing). Sanitize all three id segments to safe path segments — reject anything with `/`, `..`, or other path-escape characters. Accepts any variant of the event discriminated union.
- `readMergedEvents(channelId, conversationId, limit?)` — read every `*.jsonl` file in the conversation directory, parse each non-blank line as JSON, drop malformed lines, merge by `timestamp`. Optional `limit` slices to the last N events. This is the canonical "what happened in this conversation, across profiles" view used by render filters and consolidation.
- `readProfileEvents(channelId, conversationId, profile, limit?)` — same shape as the merged version but scoped to one profile's JSONL. Used by the gap-detection logic (we need each profile's last-event timestamp independently).
- `readContinuation(channelId, conversationId, profile)` / `writeContinuation(...)` / `clearContinuation(...)` — read/write the `{profile}.continuation` sidecar holding the active backend session token.
- `renderForChannel(events)` — apply the render filter to a merged event list: for each `turn_id`, emit user events as-is and the **last** assistant text of the turn (skipping assistant events whose `text` is empty OR equals the literal `NO_REPLY` lifecycle marker); drop intermediate assistant texts, assistant events' `tool_calls`, `role: "tool"` events, and `role: "audit"` events. Used by watch-based channels (e.g. terminal). Push-based channels don't call this.

There is no `buildLlmMessages` — agent-sdk owns conversation reconstruction via continuation tokens. Our threads JSONL is the durable record we control by teeing from `agent.events`; the backend's session storage handles its own model-input format.

The router serializes per-conversation, so `appendEvent` never has a concurrent writer on the same `(channelId, conversationId)` directory. No locking needed.

**Tee from agent-sdk.** Wrap `agent.run({...})` and consume the returned `query.events` AsyncIterable: for every `text_end`, `tool_call_end`, and `tool_result` event, build a thread `Event` and call `appendEvent`. Capture the first `session_start.continuation` token and persist it via `writeContinuation`.

**Cron logs.** The same `appendEvent` primitive can target cron log paths via a variant (`appendCronEvent(jobname, timestamp, event)`); cron logs are not per-profile (each run is a single backend invocation, so one JSONL per run is enough). Sub-agent logs are similar — one log per call, no per-profile fan-out.

### 3. Build the router

Implement `agent/src/router.ts`. Behavior contract is in `architecture.md` → Router (especially the `NO_REPLY` semantics and the [Channel `send()` contract](architecture.md)).

Implementation:

- Per-conversation queue keyed on the composite `(channelId, conversationId)` so each conversation runs serially. Don't key on `conversationId` alone — two channels could happen to share the same conversation-id string and end up sharing a queue.
- **`turn_id` assignment.** User events and the assistant invocation that responds get **different** `turn_id`s. Every event from the same invocation shares one `turn_id`. Generate a fresh id at each invocation boundary (uuid or a timestamp-based id).
- **Profile resolution.** Resolve the active model profile per the rules in `architecture.md` → Model selection (channel `model:` → `default`; cron `model:` for cron firings; explicit override for sub-agent paths). The chosen profile selects which `{profile}.jsonl` and `{profile}.continuation` to use.
- **For each inbound message:**
  1. Generate a `turn_id`, append the user event to `{profile}.jsonl` via `threads.appendEvent` (timestamp included).
  2. Read `{profile}.continuation` (if any). Compute the gap-recap per the five cases in `architecture.md` → Session continuity (using `readProfileEvents` and `readMergedEvents` to detect what other profiles have done since this one last ran).
  3. Build the user message string: optional gap-recap + the `[sent …]`-prefixed actual text. Build `QueryInput.attachments` from any user-event attachments.
  4. Call `agent.run({ message, continuation?, attachments? })` on the resolved profile's cached `Agent`. Tee `query.events` into `{profile}.jsonl`. Persist the first `session_start.continuation` to `{profile}.continuation`. On `isContinuationInvalid` errors, clear the sidecar and retry as case 3.
- After the run completes, the final assistant text has already been appended (via the tee). Extract it and call `channel.send(conversationId, reply)`.
- The reply is stored and sent **as-is** — including the literal string `NO_REPLY`. No translation. Channels detect `NO_REPLY` themselves per the contract.

### 4. Build the agent

Use agent-sdk's `Agent` + `agent.run()` for automatic tool execution. Cap each `Agent` at the SDK default (~25 turns). Do not manually implement a tool loop, manage history, or guard tokens — agent-sdk owns all three.

- Build a **profile-to-Agent resolver** from `config/models.yaml` at startup per `architecture.md` → Model selection. Expose a helper like `agentForProfile(profileName)` → `Agent` that the router/scheduler/sub-agent runner call with the resolved profile name. Construct each profile's `Agent` once on first use and cache it (one `Agent` per profile); reuse across dispatches.
- **Apply unattended defaults** at Backend construction (per `architecture.md` → Model selection → Unattended defaults). For each `claude` profile that doesn't set `permission_mode:` in YAML, pass `permissionMode: 'bypassPermissions'` to the agent-sdk Backend. For each `codex` profile that doesn't set `ask_for_approval:` / `sandbox_mode:`, pass `askForApproval: 'never'` and `sandboxMode: 'workspace-write'`. YAML values, when present, override the defaults.
- Wrap every `agent.run()` call through a **fallback-aware adapter**: on credit-exhausted, rate-limit, or provider 5xx, or connection errors, look up the resolved profile's `fallback:` (if any), build the fallback profile's `Agent`, and retry once with the fallback. Cross-backend fallback works — the adapter wraps `agent.run` regardless of backend. Auth and 4xx-other errors pass through. Continuation tokens are profile-scoped (per `architecture.md` → Session continuity), so a fallback retry uses the fallback profile's continuation, not the failed profile's.
- The dispatcher reads continuation + computes any gap-recap per the five cases in `architecture.md` → Session continuity, then calls `agent.run({ message, continuation?, attachments? })`. Tee `query.events` into the appropriate `{profile}.jsonl`; persist the first `session_start.continuation` to the sidecar.
- **No per-message history cap.** agent-sdk's per-backend auto-compaction (Claude/Codex native; Vercel and `openai` wrap their session in a compaction-aware decorator) keeps context within the model's window. Set per-profile `contextWindow:` only when the model isn't in agent-sdk's `MODEL_CONTEXT_WINDOWS` table — otherwise compaction silently disables for that profile.
- **No per-image token guard.** PR #51's `~1500 tokens per image` accounting now lives upstream in agent-sdk's compaction math. Just pass `QueryInput.attachments` and let the backend handle it.

#### System prompt assembly

The system prompt is concatenated from these sections, in order:

1. **`spec/agent.md`** — the base prompt (role, behavior, tool/skill conventions, security).
2. **`config/SOUL.md`** (identity) — if missing, run the first-run flow (see step 4a).
3. **Memory manifest** — see `architecture.md` → Memory format → Loading into context. Build it from the memory module's manifest output (step 4b).
4. **Last 24 hours of `memory/journal/` entries inline** — recent agent-authored notes likely to be relevant; older journal entries appear in the manifest only.
5. **Skills summary** — pulled from `spec/skills/*.md`, `config/skills/*.md`, and `config/skills/*/SKILL.md` frontmatter; config merges with spec on name collision (see step 4b). Each entry is formatted as `- <name> (<paths joined with " + ">) — <description>` so the agent can open the full skill body via the canonical `read` tool — for a merged skill, both source files together.

#### 4a. First-run flow

If `config/SOUL.md` doesn't exist when the agent assembles its system prompt:

- Use a minimal first-run system prompt: "You are a new personal AI assistant named Protos. You haven't been configured yet. On your next reply, introduce yourself and ask the user (1) what to call yourself and (2) how you should act — personality, tone, style. Once they answer, use the `write` tool to create `config/SOUL.md` with the chosen name and personality. Nothing else belongs in that file — memory, skills, and other context are loaded separately."
- After the user answers, the agent writes `config/SOUL.md`. Subsequent invocations read it normally.
- Also ensure `config/`, `memory/`, and `runtime/` directories exist; create them if not.

#### Tools

The canonical tools (`bash`, `read`, `write`, `edit`, `glob`, `grep`, `webFetch`, `webSearch`, `todo`, `task`) come from agent-sdk — import from the package, no implementation needed. Each `Agent` is constructed with the canonical catalog plus the protos-specific custom tools below.

**Custom tools** (one recipe each in `spec/tools/`):

- Memory-aware create/find/rename (wiki-link-form input, resolution-preserving): `find_memory`, `add_memory`, `rename_memory`
- Journaling: `add_journal_entry` (sugar for appending to today's journal entry)
- Agent control: `delegate_task` (protos's skill-aware sub-agent — see step 4d)
- Consolidation (used by `nap`/`dream` crons): `list_threads`, `read_thread_tail`, `advance_thread_cursor`
- Memory hygiene: `find_orphans`

Implementation conventions:

- **Memory graph cache invalidation** — any tool that writes under `memory/` (`add_memory`, `rename_memory`, `add_journal_entry`, plus the canonical `write` / `edit` when the path lands under `memory/`) must delete `runtime/memory-graph.json` so the next graph operation rebuilds. For the canonical tools, hook this via `withImpls` wrappers on `write` and `edit` that call the original execute then invalidate the cache when the path is under `memory/`. (Claude/Codex use native tools that bypass `execute` — for those backends, the cache rebuild falls back to mtime detection at next graph operation, which is sufficient.)
- **Resolution-preservation helper** — `add_memory` and `rename_memory` share the same pre/post snapshot → rewrite-changed-resolutions algorithm. Extract it as a single helper in `agent/src/memory.ts` and call it from both tools.
- **Tool return shapes** follow `architecture.md` → Tool return shapes. Never return bare `null` from a tool. Pairing of `tool_call_end` / `tool_result` events is owned by agent-sdk; the tee layer just records both sides.
- **Workspace path safety** — custom tools that take user-supplied paths should resolve under the workspace root and reject path-traversal escapes. There's no shared paths.ts helper anymore (the spec/agent guards are now OS-level via chmod); the small handful of remaining tools that need path validation can do it inline or share a tiny helper module.

Custom tools are added by dropping `.ts` files directly into `agent/src/tools/`. Pass them to each `Agent` alongside the canonical catalog. Loader scans the directory and registers everything.

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

#### 4c. Self-edit posture

There's no in-process write guard or env-var toggle anymore. Path discipline is convention (system prompt) plus OS-level (`chmod -R a-w spec/` in production; `chmod -R a-w agent/` to disable self-edit). See `architecture.md` → Self-modification.

What you must implement:

- **Skills loader honors `enabled: false` in frontmatter** — if `config/skills/self-edit.md` (or any skill) sets `enabled: false`, the loader skips it and the skill is hidden from the agent's system prompt. This is the user-facing way to disable self-edit even when chmod isn't applied.
- **No paths.ts helper.** The build sequence drops the shared path-guard module that previously enforced `spec/` and `agent/` rules.

#### 4d. Sub-agent runner (delegate_task)

Implement `agent/src/agents/runner.ts`. Tool contract is in `spec/tools/delegate_task.md`; design rationale and constraints are in `architecture.md` → Sub-agents.

The runner takes the `delegate_task` inputs (prompt, skills, tools, optional model) plus enough context to write its log to the right place under the caller's log directory: a stable id for this particular call, and a parent-context discriminator describing whether the call originated from a cron run (carrying the job name and run timestamp) or a thread turn (carrying the channel id, conversation id, and turn id). The `delegate_task` tool wrapper knows which one applies and plumbs it through.

Behavior:

- **Resolve skills** by name from `spec/skills/` then `config/skills/` (config wins). Read the FULL skill body — the `{name}.md` file or the `{name}/SKILL.md` body, whichever the loader registered. On any missing skill, fail the call.
- **Resolve tools** from the canonical agent-sdk catalog plus the custom tools map. **Always strip `delegate_task`** and the canonical `task` — sub-agents never spawn further sub-agents, regardless of caller request.
- **Build the system prompt:** framing line + today's date + concatenated skill bodies (NO SOUL.md, NO memory manifest, NO conversation history).
- **Compute the sub-agent log path** from the parent context and call id:
  - Cron parent → `runtime/logs/cron/{jobname}/{timestamp}/{callId}.jsonl`
  - Thread parent → `runtime/logs/sub-agents/{channelId}/{conversationId}/{turn_id}-{callId}.jsonl`
- **Resolve the model** per `architecture.md` → Model selection (`delegate_task` rule): explicit `model:` arg → first skill in the `skills:` list with `preferred_model:` frontmatter → the `subagent` profile. Call the same `agentForProfile` resolver described in step 4 so fallback is applied consistently.
- **Construct a one-shot `Agent`** (or reuse a cached profile-keyed one) configured with the assembled `instructions` (system prompt), the resolved tool subset, and the same per-backend turn cap as the main agent. Call `agent.run({ message: prompt })` with no continuation — sub-agents are stateless; each invocation is a fresh session. Tee `query.events` into the sub-agent log file using the same event schema as threads and cron logs.
- **Return** the final assistant text together with the workspace-relative log path. agent-sdk surfaces the assistant final text via the `text_end` event sequence — accumulate the last assistant turn's text and pass it back. The parent's `tool_result` event records both the text and the log path — see `spec/tools/delegate_task.md` for the exact return shape.

Framing line for the system prompt:

> "You are a focused sub-agent invoked by the main Protos agent. Complete the task described in the user message and return your final response. You have no access to the main agent's identity, memory, or conversation history beyond what's stated below."

### 5. Build the channel registry

Implement `agent/src/channels/index.ts`. Channels are defined in `architecture.md` → Channels (especially the [`send()` contract](architecture.md)).

- At startup, scan `agent/src/channels/` for `*.ts` files to build a **provider map** (filename without extension → imported module). Providers are not activated yet — this map is just a lookup.
- Load `config/channels.yaml`, run `$NAME` substitution, resolve top-level pointers.
- **Validate at load, before activating any channel.** For each entry with `enabled !== false`:
  - Look up `provider:` (default: the entry name) in the provider map. **If the provider is missing, exit the process non-zero** naming the channel and the expected module path (e.g. `agent/src/channels/{provider}.ts`). This is a config error, not a runtime one — silently skipping would mask a typo until first use.
  - If the entry has a `model:` field, check it names an existing profile in `config/models.yaml`. Unknown profile → exit non-zero naming the channel and the bad reference. Same rationale as cron `model:` validation below: directive references validate at load, not on first firing.
- Only after validation passes, call each provider module's `createChannel(name, config, router)` factory with the entry's resolved config. On factory throw, log the channel name + error message (but not the full config, which may contain secrets) and continue with the remaining channels. A runtime failure to connect is different from a config typo — the latter should prevent startup, the former should let other channels come up.
- The registry collects successfully created channels into a map keyed by `channelId` (the entry name from the YAML) so the scheduler can look up any channel's send function and owner conversation ID.
- Resolve `primary:` last. If the named channel isn't in the registered map (disabled, failed to connect, or the name doesn't match any entry), exit non-zero — the scheduler depends on the primary channel.
- If no channels registered successfully, exit non-zero with a clear error.

### 6. Build the user's chosen channel

Read the recipe at `spec/channels/{provider}.md` and follow its setup instructions. The provider implementation goes at `agent/src/channels/{provider}.ts`. The factory (`createChannel(name, config, router)`) must only forward messages from the owner (identified by `owner_id` in the channel config) and silently ignore everyone else. Get the full loop working end-to-end before adding anything else. Add a corresponding entry for this channel in the `config/channels.yaml` template from step 1.

### 6b. Build the terminal channel (always included)

Terminal is a zero-dependency, bundled channel shipped with every build. Follow `spec/channels/terminal.md` for the protocol, replay semantics, and rendering rules. Implementation lives at `agent/src/channels/terminal.ts` (server) and `agent/src/cli/chat.ts` (client). Wired into the wrapper as `agent/protos chat` (see step 9).

Setting `enabled: false` on the `terminal:` entry in `config/channels.yaml` (or omitting the entry entirely) disables the channel: the server doesn't bind the socket, and the channel registry skips it.

### 7. Build the scheduler

Implement `agent/src/scheduler.ts`. Cron format, merge rules, the merged-job table, and the per-run log layout are defined in `architecture.md` → Scheduler.

- Scan both `spec/cron/` and `config/cron/`, parse frontmatter with `js-yaml`, merge per the layering rules (frontmatter override, body append; skip when merged `enabled: false`).
- **Validate cron `model:` at load.** After merging, for every enabled job whose merged frontmatter sets `model:`, check it names an existing profile in `config/models.yaml`. Unknown profile → exit non-zero naming the job file and the bad reference. Symmetric with channel `model:` validation in step 5: directive references validate at load, not on first firing.
- Use `node-cron` to schedule each enabled job using its merged `schedule` field.
- When a job fires:
   1. **First-run guard.** If `config/SOUL.md` doesn't exist, skip the firing entirely — log a one-line note and return. The agent isn't ready to respond to cron prompts; the next firing after first-run setup works normally.
   2. Generate a run timestamp `ISO` and open the cron log at `runtime/logs/cron/{jobname}/{ISO}.jsonl`.
   3. Append a `cron_start` audit event (job name, schedule, `turn_id`, timestamp).
   4. Append the cron reminder (see `architecture.md` → Scheduler) to the merged body. Then dispatch through the router as a synthetic message. Pass the cron log file handle so the router can append each agent/tool event to it.
   5. After the agent returns, append a `cron_end` audit event (reply text, `duration_ms`, timestamp).
- **Honor the `history:` frontmatter field** (default `primary`). The field controls only what the agent **reads** as context: `primary` reads recent primary-thread events; `none` runs with only the synthetic prompt. **Writes are the same in both cases**: the full event stream goes to the cron log; only the final assistant reply goes to the user thread (skipped on `NO_REPLY`). The synthetic prompt and intermediate steps are NOT written to the user thread.
- **Honor the `model:` frontmatter field** (default: `default` profile). Validated at load (see above — unknown profile exits non-zero at startup, so firing time never sees an unresolved name). At firing time, resolve the profile through the model resolver from step 4 and pass the resulting client through to the agent invocation.
- Add a CLI subcommand `agent/protos cron` that prints the merged job table with source annotations (`[spec]`, `[config]`, `[spec → overridden by config]`, `[disabled]`).

### 8. Wire it all together

The entry point (`agent/src/index.ts`):

1. Loads `config/.env`
2. Ensures `runtime/` exists
3. Loads + validates `config/models.yaml` (see `architecture.md` → Model selection → Validation at load)
4. Loads + validates `config/channels.yaml` and registers channels
5. Starts the scheduler
6. Logs that it's running

On any config-load failure (missing file, invalid YAML, unresolved pointer, missing `$NAME` env var, missing directive reference, **missing auth source** for any profile that doesn't set a typed auth field — i.e. `claude` with no `oauth_token:`/`api_key:` AND no ambient `CLAUDE_CODE_OAUTH_TOKEN`/`ANTHROPIC_API_KEY`; `codex` with no `codex_home:` AND no `~/.codex/auth.json`; `openai` with no `api_key:` AND no ambient `OPENAI_API_KEY`), the process exits non-zero with a clear error message to stdout/stderr. The bash wrapper can't reasonably replicate this validation (YAML parsing, pointer resolution, env substitution), so it relies on a post-start health check (step 9) to surface early-exit failures by tailing the log.

That's it. No HTTP server needed unless a channel requires a webhook.

### 9. Create the `agent/protos` wrapper script

Create a bash script at `agent/protos` that supports:

- `agent/protos` (no args) — print a usage message listing the subcommands and exit.
- `agent/protos start` — start the process in the background, then run the post-start health check (same shape as restart's, below): wait ~5s, check if the PID is still alive; if dead, print the last ~30 lines of the current log file and exit non-zero. The Node process validates its config at startup and exits with a clear error on failure (see step 8), so a fast early exit almost always means a config error — the tailed log will show it directly.
- `agent/protos stop` — stop it
- `agent/protos restart` — restart it (with safe-restart protocol below)
- `agent/protos status` — check if it's running
- `agent/protos cron` — show the merged cron job table
- `agent/protos chat [flags]` — connect a terminal client to the running daemon (see step 6b). Passes flags through to `npx tsx agent/src/cli/chat.ts`. Fails fast with "daemon not running; start it with `agent/protos start`" if the daemon PID isn't alive.

The script must be invoked from the workspace root (the parent of `agent/`). It should `cd` to the workspace root if invoked from elsewhere by resolving its own location.

Run the process with `npx tsx agent/src/index.ts` (not compiled JS). This way the agent can modify its own TypeScript source and restart to apply changes — no compile step needed.

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
- `config/models.yaml` and `config/channels.yaml` exist with the build's templates; `config/.env` exists with empty stubs for every `$NAME` the YAML references
- `spec/cron/heartbeat.md`, `spec/cron/nap.md`, and `spec/cron/dream.md` are unchanged
- `spec/` is untouched by the build (the build only writes to `agent/` and optionally creates `config/`)
- The wrapper script is executable (`chmod +x agent/protos`)
- `git -C agent log --oneline` shows at least one commit (the `build` anchor — critical for safe-restart auto-revert)
- `git -C agent status` is clean (no uncommitted changes)
- `git -C config log --oneline` shows at least one commit; `config/.env` is NOT in the commit; `config/models.yaml` and `config/channels.yaml` ARE in the commit (their `$NAME` references don't carry secrets)
- `git -C memory log --oneline` shows at least one commit

## When you're done

The build is complete. `AGENTS.md` stays as-is — it's the coding-agent entry point and is needed again for future `update` invocations.

## Updating

For the `update` command — sync an existing implementation when the spec evolves.

1. **Refresh agent-sdk first.** Run `npm install github:ExProtos/agent-sdk@latest` (or `npm update agent-sdk`) inside `agent/` and stage the resulting `package-lock.json` change. agent-sdk is on a GitHub URL, not a registry version range, so `npm install` alone reuses the previously-resolved commit forever. Skipping this step is the most common reason a spec change fails to reach the running agent — symptom is the generated code reaching for a CLI workaround (e.g. `-c approval_policy=never` on the codex backend's `args:`) when the spec describes a typed agent-sdk field (e.g. `askForApproval`). That's stale agent-sdk; refresh first, regenerate the code path second.
2. **Identify the delta.** Read `spec/` as-is and compare against the current `agent/` tree. `git log` on the spec repo is useful for surfacing recent changes.
3. **Propose the changes** to the user — a per-file summary is more useful than a raw diff. Apply once they approve.
4. **Commit inside `agent/`** with a message summarizing what changed (include the agent-sdk refresh from step 1 if its lockfile moved). The commit is required: the safe-restart auto-revert depends on `agent/` having a clean `HEAD` (same reason the build commits — see step 1a).
5. **Leave `config/`, `memory/`, and `runtime/` alone by default.** Spec changes regenerate `agent/` only.
6. **Migrate data layouts when the spec changes them.** When a spec change alters where data lives (file paths, directory shape, JSONL schema), `update` should migrate the existing data automatically if there's a deterministic mapping. Show the user the planned migration before applying, and commit inside the affected domain repo (`config/` or `memory/`); `runtime/` is unversioned, so just log what moved. Known migrations the `update` command should detect and run:
   - **Legacy `.env` → YAML config** — synthesize `config/models.yaml` and `config/channels.yaml` from existing env vars per the "Migrating from legacy env-only deployments" section. Commit inside `config/`.
   - **Flat threads JSONL → per-profile threads directory** *(introduced by the agent-sdk migration)* — for each existing `runtime/threads/{channelId}/{conversationId}.jsonl`, mkdir `runtime/threads/{channelId}/{conversationId}/` and move the JSONL to `{conversationId}/{default-profile}.jsonl` (where `{default-profile}` is the resolved name of the `default:` profile in `models.yaml`). Move any sibling `{conversationId}.cursor` to `{conversationId}/{default-profile}.cursor` the same way. No `.continuation` sidecar is created — the first dispatch after migration falls into the case-3 / case-5 path (fresh session with full prior history as recap), which is the right behavior since pre-migration threads have no backend session token to resume.

End with a clean working tree in `agent/`, same invariant as build.

## Testing

See `spec/test.md`.
