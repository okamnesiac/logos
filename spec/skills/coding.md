---
name: coding
description: Delegate complex coding tasks to a coding agent (Claude Code or Codex).
---

# Coding

For complex, multi-file, or unfamiliar coding tasks, delegate to a coding agent rather than hand-editing with shell commands.

## When to use this

- Building something new (a website, a script, a tool)
- Multi-file refactors or large changes
- Tasks where you'd benefit from a coding agent's tools (file search, LSP, testing)

## Available agents

Use whichever is installed on the host. Check with `which claude` or `which codex`.

### Claude Code

```
claude --dangerously-skip-permissions -p "your prompt here"
```

- `--dangerously-skip-permissions` lets it run without interactive approval (use only for tasks you've already confirmed with the owner)
- `-p` passes a one-shot prompt
- For tasks in a specific directory, `cd` there first

### Codex

```
codex --approval-mode full-auto "your prompt here"
```

- `--approval-mode full-auto` lets it run without interactive approval
- For tasks in a specific directory, `cd` there first

## Steps

1. Confirm the task with the owner — what to build, where it should go, any constraints
2. Craft a clear, detailed prompt describing the task
3. Run the coding agent via your shell tool
4. Review the output and report back to the owner what was built or changed

## Rules

- Always confirm the task before running a coding agent
- Set a working directory appropriate to the task — don't run everything from the project root
- If the coding agent fails or produces bad output, tell the owner rather than retrying blindly
