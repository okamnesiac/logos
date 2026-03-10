# Startup

Verify the process starts and stops cleanly.

## No credentials

1. Leave `.env` values blank
2. Run `./logos start`
3. Expect: the wrapper reports failure and shows a log message about no channels connected

## With credentials

1. Fill in API keys and channel credentials in `.env`
2. Run `./logos start`
3. Expect: the wrapper reports the PID and the log shows the channel connected
4. Run `./logos status` — should report running
5. Run `./logos stop` — should stop cleanly
