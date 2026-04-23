---
name: scheduling
description: How scheduled jobs invoke you, how to add new ones, and how to queue dynamic work for them.
---

# Scheduling

You can be invoked two ways: by a user message, or by a scheduled job firing. When a job fires, the scheduler injects its synthetic prompt and your reply goes to the user's primary channel — this is your mechanism for proactive outreach. Reply with exactly NO_REPLY if there's nothing worth saying.

## Seeing current jobs

Run `agent/protos cron` for a list, or read the files directly under `spec/cron/` (defaults) and `config/cron/` (overrides; config wins on name collision).

## Adding a new job

Write a markdown file to `config/cron/{name}.md` with frontmatter `schedule:` (cron expression) and a body describing what to do when it fires. The scheduler picks it up on next restart.

Optional frontmatter:

- `enabled: false` — disable the job without removing the file.
- `history: none` — run the job with no conversation history in context (only the cron body). Default is `history: primary`, which gives the agent the last 50 messages of the primary channel's owner conversation. Use `none` for internal/background jobs that don't need to comment on recent activity.

## Queueing dynamic work (opt-in pattern)

If a job needs to handle work that varies between firings, the convention is to have the job's instructions check `memory/tasks/{job}.md` and the user (or you) writes tasks there. This is opt-in — the cron file must explicitly say to check it. Example: `spec/cron/heartbeat.md` uses this pattern with one-shot and recurring sections.
