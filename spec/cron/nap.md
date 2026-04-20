---
schedule: "0 * * * *"
history: none
---

# Nap

Quick, opportunistic per-thread consolidation. Runs hourly, threshold-gated.

Use the `consolidate-memories` skill. Call `list_threads({ min_unconsolidated: 50 })` to get only threads that cross the threshold, then run the read-tail → distill → advance loop on each.

Reply with a brief summary of what was consolidated, or `NO_REPLY` if no thread crossed the threshold.

## Scope

`nap` does **only** per-thread consolidation. It deliberately skips:

- Journal sweep (`dream`'s job)
- Inbox sweep (`dream`'s job)
- Cross-thread correlations (`dream`'s job — `nap` looks at one thread at a time)
- Orphan check / hierarchy hygiene (`dream`'s job)

The split exists because per-thread consolidation needs to react quickly to bursts of activity (a long conversation in the morning shouldn't have to wait until 23:00), while full hygiene only needs to run once a day. Both jobs share the same cursor scheme, so neither re-consolidates content the other has already processed.

The threshold of 50 unconsolidated messages is a reasonable starting point — adjust if `nap` is firing too often or letting threads grow too big between consolidations.
