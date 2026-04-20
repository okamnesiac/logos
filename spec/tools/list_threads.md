# list_threads

Enumerate conversation threads with pending consolidation work.

## Description (shown to the model)

List conversation threads that have unconsolidated messages waiting, with each thread's total message count, current consolidation cursor, and the number of unconsolidated messages. Use this to plan a consolidation pass. By default, returns only threads with `unconsolidated >= 1`; pass `min_unconsolidated` to raise the threshold (e.g. `50` for opportunistic per-thread consolidation) or `0` for a full listing.

## Input

```ts
{ min_unconsolidated?: number }
```

- `min_unconsolidated` — only return threads where `unconsolidated >= min_unconsolidated`. Defaults to `1` (returns only threads with pending work). Pass `0` to include fully-consolidated threads.

## Output

```ts
{
  threads: Array<{
    channelId: string,
    conversationId: string,
    total: number,         // total messages in the JSONL
    cursor: number,        // last consolidated index (0 if no cursor file)
    unconsolidated: number // total - cursor
  }>
}
```

`threads` is empty when no thread meets the threshold. Always returns the discriminated shape — never `null`.

## Behavior

- Walks `runtime/threads/{channelId}/{conversationId}.jsonl`. For each, counts non-blank lines to compute `total`, reads the sidecar `*.cursor` file (a single integer; missing means `0`), computes `unconsolidated = total - cursor`, filters by `min_unconsolidated`.
- Skips files that don't end in `.jsonl` and any directory entries that aren't conversation files.
- Cheap to call — counts lines without parsing JSON.

## Dependencies

Built-in, no external services. Reads `runtime/threads/`.

## Implementation notes

- Use the same path layout the threads module already enforces.
- Don't validate the cursor against the JSONL contents — if a cursor file says `999` but the JSONL has 5 lines, `unconsolidated` comes out negative. Negative values don't satisfy any positive `min_unconsolidated`, so a corrupt cursor naturally filters the thread out. (Caller can pass `min_unconsolidated: 0` and inspect if they want to see it.)
