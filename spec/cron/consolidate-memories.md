---
schedule: "0 23 * * *"
---

# Consolidate Memories

Two sources to sweep:

1. Today's scratch pad in `memory/journal/` — promote anything important into structured memory files.
2. Files in `memory/new/` — the inbox where new notes land when there's no obvious folder yet. Sort them into appropriate folders (or merge them into existing files).

## What to promote

- Facts about people (names, preferences, relationships)
- Decisions that were made
- Commitments or promises (things the user said they'd do, or asked you to do)
- Recurring patterns or preferences

## What to skip

- Routine exchanges that don't contain new information
- Anything already captured in `memory/`
- Temporary context that won't matter tomorrow

## How to update memory

- Add granular entries to relevant files under `memory/` (create files and subdirectories as needed)
- Use `[[wiki-link]]` syntax to reference other notes by name
- For files in `memory/new/`: either move them to a sensible folder (e.g., `mv memory/new/coffee.md memory/preferences/`) or merge their content into an existing file and delete the original
- Remove entries that are outdated or contradicted by new information
- Keep entries concise — the system prompt loads a manifest of all memory (name + summary), so a clear `description:` field in each file's frontmatter helps future-you retrieve the right note
