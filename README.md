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
bootstrap telegram
```

The agent reads the spec and generates the assistant.

More specifically:

1. **Create your own copy** — GitHub doesn't allow private forks of public repos, so [import this repository](https://github.com/new/import) as a new private repo instead
2. **Point your AI coding agent at it** (Claude Code, Codex, etc.)
3. **Tell the agent:** `bootstrap telegram` (or whichever channel you want)
   - The coding agent reads `spec/` and generates the implementation in `agent/`. Approve its edits.
4. **Fill in your API keys** in `config/.env`
5. **Start it** — `agent/logos start`
6. **Send it a message** — on first run, it'll ask for a name and personality

The repository contains no running code. It contains the spec. The coding agent reads the spec and generates the implementation.

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

The workspace is split into five sibling domains. Only `spec/` is tracked by this repo; the others are gitignored and may optionally be backed by their own repos.

```
# Tracked by this repo
README.md, CLAUDE.md, AGENTS.md   # workspace entry-point docs
spec/                             # the blueprint
  ARCHITECTURE.md                 # system design
  BUILD.md                        # build instructions for coding agents
  channels/                       # channel recipes (markdown)
  skills/                         # bundled skills (markdown, agentskills.io format)
  cron/                           # default scheduled jobs (markdown)

# Gitignored (or own repo)
agent/                            # generated implementation — TypeScript code
config/                           # behavior — SOUL.md, instance overrides, .env
memory/                           # durable state — facts, preferences, journal
runtime/                          # ephemeral state — threads, logs, pid files
```

The repo is the **spec** — the design. The bootstrap reads `spec/` and generates `agent/`. Each user has their own `agent/`, `config/`, and `memory/`, optionally as their own Git repos.

See [spec/ARCHITECTURE.md](spec/ARCHITECTURE.md) for the full system design.

## Principles

- **Your machine, your data.** Everything runs locally. Nothing phones home.
- **Files over databases.** Everything is plain files: identity, memory, skills, cron, and message history. No database, no driver, no schema. `cat` is your inspector.
- **Specify the what, not the how.** The spec defines components and responsibilities. Implementation details are left to the coding agent.
- **No speculative features.** Only what's needed for a working assistant.
- **Four domains.** Engine, behavior, memory, and runtime are separate concerns with separate lifecycles.

## Inspired by

[NanoClaw](https://github.com/qwibitai/nanoclaw) — a brilliant minimal agent runtime. Logos asks: what if you went even more fundamental and didn't write any code at all?
