# Cron and Scheduler

Verify scheduled jobs run and route correctly.

## Heartbeat

1. Start Logos and wait for the heartbeat to fire (every 30 minutes, or temporarily change the cron expression to `* * * * *` for testing)
2. If nothing needs attention, the agent should respond with `NO_REPLY` and no message should appear in the owner's chat
3. Check the database — no `NO_REPLY` messages should be stored

## Scheduler routing

1. Add a test cron job with an inline prompt that says "Say hello"
2. Wait for it to fire
3. Expect: the agent's reply appears in the owner's main chat on the primary channel
