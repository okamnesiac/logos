---
name: consolidation
description: The per-thread and full-sweep consolidation pattern used by the nap and dream cron jobs — read unconsolidated thread tails, distill them into long-term memory, advance the cursors.
---

# Consolidation

The shared pattern for turning conversation thread content into structured long-term memory. Used by both `nap` (per-thread, hourly, threshold-gated) and `dream` (full sweep, daily). The mechanics are identical; the cron files layer on different scope.

## The core loop

1. **`list_threads({ min_unconsolidated? })`** — get threads with pending work. Defaults to `min_unconsolidated: 1` (any unconsolidated message). Raise the threshold to match your scope (e.g. `50` for `nap`-style opportunistic consolidation).
2. For each thread you want to consolidate:
   1. **`read_thread_tail(channelId, conversationId)`** → `{ messages, newCursor }`. The messages are everything since the cursor.
   2. **Distill** the messages into memory. Update existing files (`edit_file` or `write_file` with `mode: "replace"`), create new ones (`add_memory`), use `[[wiki-links]]` to connect notes, drop transient or routine content.
   3. **`advance_thread_cursor(channelId, conversationId, cursor: newCursor)`** — only after the distillation succeeded. Skip on failure; the messages will reappear in the next pass.

That's it. The cursor mechanics are mindless once you use the tools.

## What's worth keeping

There's no rigid schema, but useful things to capture:

- Facts about people (names, preferences, relationships)
- Decisions and commitments
- Recurring patterns or preferences
- Cross-references — "X mentioned this also relates to Y"

What to skip:

- Routine exchanges with no new information
- Transient context that won't matter tomorrow
- Anything already in memory (check first via `find_memory` or the manifest)

## Where to put things

Memory is an Obsidian-compatible graph: markdown files, `[[wiki-links]]` to connect notes, folders for whatever taxonomy makes sense. There's no rigid schema. A few rules of the road:

- The system prompt only sees **root-level** files (`memory/*.md`). Anything important must be transitively reachable from a root file via `[[wiki-links]]`.
- Each file's frontmatter `description:` is what surfaces in the manifest. Make it clear and retrievable.
- When new information contradicts old, update or remove the stale entry rather than letting both coexist.
- When creating a new memory file, use `add_memory` rather than raw `write_file(mode: "create")` — `add_memory` preserves `[[wiki-link]]` resolutions the new file would otherwise shadow.
- When reorganizing, use `rename_memory` instead of `write_file` + `read_file` + delete — it moves the file AND rewrites `[[wiki-links]]` so every reference resolves to the same file it did before.

## Cross-thread correlation

When consolidating multiple threads in one pass (the `dream` use case), look for patterns *across* threads as well as within them — the same person mentioned in two channels, a decision in one thread that affects work tracked in another, recurring themes worth a top-level note. Per-thread consolidation (`nap`) doesn't get this benefit.

## Hygiene (dream only)

After updating memory at the end of a full sweep, run `find_orphans()` to find non-root files no longer reachable from any root file. For each orphan, decide:

- Still relevant? Add a `[[wiki-link]]` from a relevant root file (or a file that's transitively reachable from root).
- Stale? Move to `archive/` or delete.

Then consider promotions and demotions:

- A subfolder file being read or referenced often → promote to root via `rename_memory("{folder}/{name}", "{name}")`.
- A root file with narrow scope or rare reference → demote into a subfolder, leave a `[[wiki-link]]` from a broader root file.
