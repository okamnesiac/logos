# Self-Edit and Restart

Verify the agent can modify its own code and restart safely.

## Successful restart

1. Ask the agent to make a small change to its source code (e.g., "Add a comment to the top of src/agent.ts")
2. Expect: the agent explains what it changed, then runs `./logos restart`
3. Expect: the process restarts, the change is present in the file, and the agent responds to new messages

## Failed restart

1. Manually introduce a type error in a source file
2. Run `./logos restart`
3. Expect: the type check fails, the restart is aborted, and the old process keeps running
