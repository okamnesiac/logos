# Logos

A blueprint for personal AI assistants that build themselves.

## Why

[OpenClaw](https://github.com/openclaw/openclaw) demonstrates that local AI assistants are incredibly useful. But it is also a large and complex system.

Logos explores the opposite extreme. It lets you build a powerful assistant that is:

- **simple enough to understand**
- **built from readable files**
- **local-first**
- **entirely yours**

Logos is an architecture specification — a set of documents precise enough that an AI coding agent can read them and generate a working personal assistant from scratch.

## How it works

You don't install Logos. You point an AI coding agent at it and say:

```
bootstrap <channel>
```

…where `<channel>` is whichever messaging platform you want (Telegram, WhatsApp, Discord, Slack, …). The agent reads the spec and generates the assistant.

More specifically:

1. **Clone the repo** — `git clone https://github.com/ninjudd/logos.git`. No fork needed; your personal state lives in nested repos (`agent/`, `config/`, `memory/`) created during bootstrap, which are gitignored here.
2. **Point your AI coding agent at it** (Claude Code, Codex, etc.)
3. **Tell the agent:** `bootstrap telegram` (or whichever channel recipe under `spec/channels/` you prefer)
   - The coding agent reads `spec/` and generates the implementation in `agent/`. Approve its edits.
4. **Fill in your API keys** in `config/.env`
5. **Start it** — `agent/logos start`
6. **Talk to it** — DM the bot on your chosen channel, or run `agent/logos chat` for a local terminal chat over a Unix socket (always available, no messaging platform setup required). On first run, the agent asks for a name and personality.

The repository contains no running code. It contains the spec. The coding agent reads the spec and generates the implementation.

## Updating

When the spec evolves, sync your implementation the same way you bootstrapped it. Point your coding agent at the workspace and say:

```
update agent
```

The agent pulls the latest `spec/`, diffs it against your `agent/` tree, shows what would change, and applies the edits once you approve. Restart with `agent/logos restart` to pick them up.

Only `agent/` is regenerated. Your `config/`, `memory/`, and `runtime/` are left alone.

## What you get

A personal AI assistant that:

- Runs as a single Node.js process on your machine
- Connects to your messaging apps (Telegram, WhatsApp, Discord, Slack, etc.)
- Uses any LLM you choose (model-agnostic via the Vercel AI SDK)
- Has a personality you define in `config/SOUL.md`
- Remembers things in markdown files you can read and edit
- Runs scheduled tasks on your behalf
- Can modify its own source code and restart to apply changes
- Is small enough to understand completely

## Workspace layout

The workspace is split into five sibling domains. Only `spec/` is tracked by this repo. `agent/` is a nested Git repo (required — the self-edit auto-revert depends on `git` in that directory). `config/` and `memory/` are strongly recommended to be Git repos too; `runtime/` is never versioned.

```
# Tracked by this repo
README.md, CLAUDE.md, AGENTS.md   # workspace entry-point docs
spec/                             # the blueprint
  architecture.md                 # system design
  agent.md                        # base prompt for the running assistant
  build.md                        # build instructions for coding agents
  channels/                       # channel recipes (markdown)
  tools/                          # tool recipes (markdown)
  skills/                         # bundled skills (markdown, agentskills.io format)
  cron/                           # default scheduled jobs (markdown)

# Gitignored (own repo)
agent/                            # generated implementation — TypeScript code
config/                           # behavior — SOUL.md, instance overrides, .env
memory/                           # durable state — facts, preferences, journal
runtime/                          # ephemeral state — threads, logs, pid files
```

The repo is the **spec** — the design. The bootstrap reads `spec/` and generates `agent/`. Each user has their own `agent/`, `config/`, and `memory/` as their own Git repos.

See [spec/architecture.md](spec/architecture.md) for the full system design.

## Principles

- **Your machine, your data.** Everything runs locally. Nothing phones home.
- **Files over databases.** Everything is plain files: identity, memory, skills, cron, and message history. No database, no driver, no schema. `cat` is your inspector.
- **Specify the what, not the how.** The spec defines components and responsibilities. Implementation details are left to the coding agent.
- **No speculative features.** Only what's needed for a working assistant.
- **Five domains.** `spec/` (blueprint), `agent/` (engine), `config/` (behavior), `memory/` (knowledge), and `runtime/` (ephemeral state) are separate concerns with separate lifecycles.
