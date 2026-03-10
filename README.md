# Logos — the ProtoClaw

A zero-code blueprint for building your own personal AI assistant. Inspired by [OpenClaw](https://github.com/openclaw/openclaw) and [NanoClaw](https://github.com/qwibitai/nanoclaw).

## What is this?

Logos is not a framework, library, or application. It's an architecture specification — a set of documents precise enough that an AI coding agent can read them and generate a working personal AI assistant from scratch.

## How to use

1. **Fork this repository**
2. **Point your AI coding agent at it** (Claude Code, Cursor, etc.)
3. **Tell the agent to read `AGENTS.md` and build the project**
4. **Fill in your API keys** in `.env`
5. **Start it** — `./logos start`
6. **Send it a message** — on first run, it'll ask for a name and personality

## What you get

A personal AI assistant that:

- Runs as a single Node.js process on your own machine
- Connects to your messaging apps (WhatsApp, Telegram, Discord, Slack, etc.)
- Uses Claude as its brain (via the Vercel AI SDK — model-agnostic)
- Has a personality you define in `SOUL.md`
- Remembers things in markdown files you can read and edit
- Runs scheduled tasks and periodic check-ins on your behalf
- Is small enough to understand completely

## Guiding principles

- **As simple as necessary, no simpler.** No microservices, no container orchestration, no configuration sprawl.
- **Specify the what, not the how.** The architecture describes components and responsibilities. Implementation decisions are yours.
- **No speculative features.** Only what's needed for a working assistant.
- **Your machine, your rules.** Everything runs locally. You own your data.

## Default tech stack

| Component | Choice |
|-----------|--------|
| Runtime | Node.js 22+ |
| Language | TypeScript |
| Database | SQLite (messages only) |
| AI | Vercel AI SDK with Anthropic provider (model-agnostic) |
| Hosting | Runs directly on the host — no containers |

These are defaults, not requirements. Fork and change whatever you want.

## Repository structure

```
# Agent configuration
AGENTS.md           # Agent instructions (reads SOUL.md, memory, etc.)
SOUL.md             # Agent name, personality, voice, and values
ARCHITECTURE.md     # System design — how it all fits together
BUILD.md            # One-time build instructions (delete after use)

# Data
messages.db         # SQLite message history (gitignored)
memory.md           # Consolidated long-term memory
memories/           # Daily scratch pad files
conversations/      # Per-conversation context and prompts
cron/               # Scheduled tasks (config.yaml + job instructions)
skills/             # Agent skills (agentskills.io format)

# Recipes
recipes/            # Implementation guides for adding capabilities
  telegram.md       # Channel recipes
  whatsapp.md
  imessage.md
  discord.md
  slack.md
  signal.md
  matrix.md
  voice-input.md    # Capability recipes
  voice-output.md

# Code (generated during bootstrap)
src/                # All code and config lives here
```

## Architecture

See [ARCHITECTURE.md](ARCHITECTURE.md) for the full system design.
