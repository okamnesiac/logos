---
schedule: "0 23 * * *"
history: none
---

# Dream

Daily deep consolidation. Distill every thread's unconsolidated tail, the day's journal, and the inbox into long-term memory; keep the memory graph tidy.

## Steps

### 1. Threads — full sweep with cross-thread correlation

Use the `consolidate-memories` skill. Call `list_threads()` (the default `min_unconsolidated: 1` picks up every thread with pending work) and run the read-tail → distill → advance loop on each. Look for cross-thread patterns as you go (the same person mentioned in two channels, a decision in one thread that affects work in another).

### 2. Journal — promote anything worth keeping

Read `memory/journal/{today}.md`. Promote anything that should outlive the day into structured memory files. Routine exchanges and one-off context can stay in the journal (or be discarded — the journal is a scratch pad, not a permanent record).

### 3. Inbox — sort and merge

Files under `memory/new/` are notes the agent (or a human in Obsidian) wrote without a clear home. Move each to a sensible folder with `rename_memory`, or merge into an existing file and delete the original. The inbox should be empty (or close to it) when you're done.

### 4. Hygiene — keep the graph navigable

After the updates above, do the orphan check and the promotion/demotion pass described in the `consolidate-memories` skill:

- `find_orphans()` to surface unreachable non-root files. For each, link in or move/delete.
- Promote subfolder files that are read or referenced often to root level.
- Demote root files that have narrow scope or rare references into subfolders.

## Reply

Post a brief summary of what changed (threads consolidated, items promoted, orphans dealt with), or `NO_REPLY` if it was a quiet day with nothing to report.
