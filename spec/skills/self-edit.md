---
name: self-edit
description: Modify the agent's own source code and safely restart to apply changes.
---

# Self-Edit

You can modify your own source code in `agent/src/`. The process is designed to be safe: compile errors are caught before restart, runtime startup crashes auto-revert the edit.

If this skill is visible to you at all, self-edit is enabled (the skill is hidden when `PROTOS_SELF_EDIT=false`).

## Steps

1. **Make your code changes** using `edit_file` (surgical find-and-replace) or `write_file` (for whole-file changes or new files). Prefer `edit_file` when possible — smaller, safer.
2. **Commit the change** using the **git** skill. One commit per edit, with a clear message explaining what changed and why. This enables rollback if the restart fails.
3. **Send a message to the conversation** explaining what you changed and why.
4. **Run `agent/protos restart`** as your **last action**.

## Safety

The wrapper protects you from self-inflicted outages:

- **Compile errors:** the wrapper runs `tsc --noEmit` before restarting. If it fails, the restart is aborted and the old process keeps running.
- **Runtime crashes:** after starting the new process, the wrapper waits a few seconds and checks if it's alive. If it crashed on startup (bad import, missing env var, startup exception), the wrapper **auto-reverts your last commit in `agent/`** and restarts with the pre-edit code.
- **Subtle logic bugs:** if you notice the change is wrong after the fact, revert it with the **git** skill — `git revert HEAD` from inside `agent/`, then restart.

This assumes `agent/` is a Git repo. If it isn't, auto-revert won't work — the wrapper will leave the broken code in place for manual recovery.

## Reference

Before making changes, read the relevant documentation:

- `spec/architecture.md` — system design, component responsibilities, how pieces fit together
- `spec/channels/*.md` — implementation recipes for each messaging channel
- `spec/build.md` — how the engine was built (useful when adding new components)

## Rules

- Always commit before restarting — the auto-revert depends on it
- Always explain what you changed in the conversation
- Make small, focused changes — one thing at a time
- For complex or multi-file changes, use the **coding** skill instead
- If you're unsure about a change, describe it first and ask for confirmation
- Never modify `config/.env` or files outside the workspace
- Edit files in `agent/src/` (your implementation). Don't edit `spec/` — that's the design, shared with all Protos users; if the design needs to change, raise it with the human owner.
- Don't put instance-specific behavior in `agent/` — that belongs in `config/`
