---
schedule: "*/30 * * * *"
---

# Heartbeat

Check `memory/tasks/heartbeat.md` for pending work. If the file doesn't exist, there's nothing to do — reply with exactly NO_REPLY.

When the file exists, it can have two sections (both optional):

## One-shot

Tasks to do once. Complete them, then remove the line. Report what you did.

## Recurring

Tasks to do every heartbeat. Complete them, leave the line. Mention them only if something noteworthy happened.

If nothing was completed and nothing noteworthy happened, reply with exactly NO_REPLY.

When you (or the user) need to add a heartbeat task, create `memory/tasks/heartbeat.md` with the two headers and add the task under the appropriate section.
