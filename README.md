# Logos

A blueprint for personal AI assistants that build themselves.

## Why

[OpenClaw](https://github.com/openclaw/openclaw) demonstrates that local AI assistants are incredibly useful. But it is also a large and complex system.

Logos explores the opposite extreme: a minimal assistant architecture built from readable files. It's an architecture specification — a set of documents precise enough that an AI coding agent can read them and generate a working personal assistant from scratch.

## How it works

1. **Fork this repository**
2. **Point your AI coding agent at it** (Claude Code, Codex, etc.)
3. **Tell the agent:** `bootstrap telegram`
   - The agent will create files and run commands. Approve its edits — it's building the whole codebase for you.
4. **Fill in your API keys** in `.env`
5. **Start it** — `./logos start`
6. **Send it a message** — on first run, it'll ask for a name and personality

The repository contains no running code. It contains the spec. The coding agent reads the spec and generates the implementation.

## What you get

A personal AI assistant that:

- Runs as a single Node.js process on your machine
- Connects to your messaging apps (Telegram, WhatsApp, Discord, Slack, etc.)
- Uses Claude as its brain (model-agnostic via the Vercel AI SDK)
- Has a personality you define in `SOUL.md`
- Remembers things in markdown files you can read and edit
- Runs scheduled tasks on your behalf
- Can modify its own source code and restart to apply changes
- Is small enough to understand completely

## What's in the repo

```
# The spec
ARCHITECTURE.md     # System design — components, contracts, data flow
BUILD.md            # Step-by-step build instructions for coding agents
AGENTS.md           # Runtime behavior contract for the assistant
SOUL.md             # Identity template — name, personality, voice

# Data (used at runtime)
memory.md           # Long-term memory (read every invocation)
memories/           # Daily scratch pad files
cron/               # Scheduled tasks (config + prompts)
skills/             # Reusable capabilities (agentskills.io format)
recipes/            # Implementation guides for channels and features

# Generated during bootstrap
src/                # All source code
```

## Principles

- **Your machine, your data.** Everything runs locally. Nothing phones home.
- **Files over databases.** Memory, skills, and config are readable markdown and YAML. SQLite is only for message history.
- **Specify the what, not the how.** The spec defines components and responsibilities. Implementation details are left to the coding agent.
- **No speculative features.** Only what's needed for a working assistant.

## Architecture

See [ARCHITECTURE.md](ARCHITECTURE.md) for the full system design.

## Inspired by

[OpenClaw](https://github.com/openclaw/openclaw) and [NanoClaw](https://github.com/qwibitai/nanoclaw).
