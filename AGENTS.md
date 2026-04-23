# Agents

This workspace is a spec for building a personal AI assistant ("the agent"). If you're a coding agent (Claude Code, Codex, Cursor, …) that the user has opened here, you're here to generate or maintain that assistant's implementation.

The user may invoke you with:

- **`build <channel>`** — first-time generation. Read `spec/build.md` and generate the implementation in `agent/` for the named channel (telegram, slack, discord, …).
- **`update`** — sync an existing implementation with spec changes. Follow `spec/build.md` → Updating.
- **`test`** *(optional)* — generate or refresh tests in `agent/test/` that verify the implementation matches the spec, then run them. Follow `spec/test.md`. This is the **only** invocation that touches `agent/test/` — build and update never write, run, or mention tests.

Start with:

- `spec/build.md` — the build sequence
- `spec/architecture.md` — system design
- `spec/agent.md` — the base prompt loaded into the running assistant at runtime
