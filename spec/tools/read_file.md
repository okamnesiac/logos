# read_file

Read a file from the workspace and return its contents.

## Description (shown to the model)

Read any file in the workspace by relative path. Returns the file's text content.

## Input

```ts
{ path: string }
```

`path` is relative to the workspace root. Examples: `config/SOUL.md`, `memory/people/justin.md`, `agent/src/router.ts`.

## Output

Plain string — the file's content as UTF-8 text.

On error: throws. The AI SDK reports the throw to the model as a tool error.

## Behavior

- **Reject path escapes.** Resolve to absolute, then check that the resolved path is inside the workspace root. Reject `..`, absolute paths outside the workspace, and paths that would escape via symlinks.
- **No size cap** — the agent's per-message token guard truncates oversize content downstream.
- Return `ENOENT` errors with a clear message: `file not found: {path}`.

## Dependencies

Node standard library (`node:fs/promises`).

## Implementation notes

- Share path-safety helpers with `write_file` and `edit_file` (e.g., `agent/src/tools/path-safety.ts`).
- This tool does NOT participate in the `LOGOS_SELF_EDIT` guard — it's read-only.
