---
name: update
description: Check for spec updates, summarize them for the owner, and apply via a coding agent on approval.
---

# Update

When the owner asks you to "check for updates", "sync with spec", or similar, show them what's new in the spec and only delegate to a coding agent after they approve.

The coding agent runs non-interactively (no human in its loop), so the review has to happen in the chat with the owner — before the coding agent runs.

## Steps

1. **Pull the spec.** `git -C spec pull` to fetch the latest.
2. **Find the delta.** Use `git -C agent log -1 --format=%aI` to get the timestamp of the agent's last commit (the last sync point), then `git -C spec log --since="<that-timestamp>" --oneline` to list spec commits newer than that. If nothing comes back, tell the owner the spec is already current and stop.
3. **Summarize for the owner.** List the new spec commits with their subject lines and a one-sentence description of what each changes (group by theme when helpful). Ask whether to proceed.
4. **On approval, delegate via the coding skill.** Use the **coding** skill to spawn Claude Code or Codex from the workspace root with the prompt `update agent`. It follows `spec/build.md` → Updating: diffs the spec against `agent/`, applies the edits, and commits inside `agent/`.
5. **Report completion.** Summarize what the coding agent committed (use `git -C agent show --stat HEAD`). Remind the owner to run `agent/protos restart` if `agent/src/` was touched.

## Rules

- Never skip the review step — the owner wants to see the spec delta before anything is applied.
- Don't attempt the diff or edits yourself. Delegation is the point.
- Never touch `config/`, `memory/`, or `runtime/`.
- Never run `agent/protos restart` automatically — the owner decides when to pick up changes.
