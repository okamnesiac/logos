---
name: memory
description: Save things you want to remember; organize, link, and persist your memory graph.
---

# Memory

Your long-term knowledge lives in `memory/` — markdown files connected by `[[wiki-links]]`. The system prompt only loads a manifest of root-level files; you reach everything else by following links from there. See `architecture.md` → Memory format for the full design.

## What's worth saving

- Facts about people (names, preferences, relationships, context)
- Decisions and commitments (yours or the user's)
- Recurring patterns or preferences
- Anything the user explicitly asks you to remember

What to skip:

- Routine exchanges with no new information
- Transient context that won't matter tomorrow (use the journal if you do want to jot it)
- Anything already captured (check `find_memory` first if unsure)

## Which tool to use

- **`remember(text)`** — quick journal jot. Sugar for appending to `memory/journal/{date}.md`. Use for transient observations or things you want to consolidate later.
- **`add_memory(name, content)`** — create a new structured memory file. Takes wiki-link form (no `memory/` prefix, no `.md`). Preserves existing `[[wiki-link]]` resolutions that the new file would otherwise shadow. Prefer this over `write_file` for new memory.
- **`find_memory(name)`** — look up by wiki-link name. Returns a list of matches sorted shortest-path first; first entry is what bare `[[name]]` would resolve to.
- **`rename_memory(oldName, newName)`** — move/rename a memory file. Rewrites every `[[wiki-link]]` so each reference still resolves to the same file.
- **`edit_file` / `write_file({mode: "replace"})`** — update an existing memory file. Don't use `write_file({mode: "create"})` for new memory; it bypasses link preservation.

## Linking and hierarchy

The graph is your map. A few rules of the road:

- **Root files are the front page.** The system prompt manifest only includes `memory/*.md` (no subfolders). Anything important must be transitively reachable from a root file via `[[wiki-links]]`.
- **Use links liberally.** `[[name]]` resolves by basename anywhere in the tree. Cross-link related notes so the graph stays navigable.
- **Each file's `description:` frontmatter is what surfaces in the manifest.** Make it clear and retrievable.
- **Reserved folders** (exempt from the orphan check):
  - `memory/journal/` — daily scratch pad. The `dream` cron promotes notable items into structured files.
  - `memory/new/` — inbox for notes without a clear home. `dream` sorts these.
  - `memory/archive/` — cold storage. Move stale orphans here with `rename_memory("name", "archive/name")` instead of deleting.

## Persistence

**After any write under `memory/`, commit and push from inside the `memory/` directory using the `git` skill.** Push directly to `main` — no branch, no PR. Memory commits should be small, granular, and immediate. A clear commit message describing what you saved is enough.

```
git -C memory add -A
git -C memory commit -m "remember justin's coffee order"
git -C memory push
```

(Or run the commands from inside `memory/` without `-C`.) Don't ask for confirmation; this is your durable state and yours to maintain.

If a push fails (auth, network), surface it to the user briefly — don't silently drop the change. The next successful push will catch up.
