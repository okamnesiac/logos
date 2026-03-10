# Send and Receive

Verify the basic message loop works end-to-end.

## Owner message

1. Start Logos with valid credentials
2. Send a message from the owner's account on the primary channel
3. Expect: the agent replies within a few seconds

## Non-owner message

1. Send a message from a different account on the same channel
2. Expect: no response, no error in logs

## Memory round-trip

1. Send "Remember that my favorite color is blue"
2. Wait for acknowledgment
3. Send "What's my favorite color?"
4. Expect: the agent recalls blue
