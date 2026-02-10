# Priority P1: Cross-Session Message Routing Broken

**Date:** 2026-02-09 15:28 UTC (Updated: 17:06 UTC)
**Status:** ✅ RESOLVED - Claude fixed it, verified working at 17:06 UTC
**Reporter:** Eddie (agent:main)

## Problem

`sessions_send()` appears to succeed but messages don't route to the target session correctly. Carlos confirmed this is still broken as of 16:05 UTC after a fix attempt.

## Tests Performed

### Test 1 (15:27 UTC)
1. Carlos requested: "send a message in our group with melissa to test your skills"
2. Eddie called:
```
sessions_send(
  sessionKey="agent:main:telegram:group:-5129786935",
  message="Testing cross-session messaging. Can now send from 1:1 to group chat. Works."
)
```
3. Result returned: `{ status: "ok", delivery: { status: "pending", mode: "announce" } }`
4. **Result:** FAILED - No message appeared in group chat

### Test 2 (16:04 UTC - After Attempted Fix)
1. Carlos requested: "Eddie, try this again and send a message in our group with Melissa"
2. Eddie called:
```
sessions_send(
  sessionKey="agent:main:telegram:group:-5129786935",
  message="Testing cross-session messaging again. Carlos asked me to verify the fix works."
)
```
3. Result returned: `{ status: "ok", delivery: { status: "pending", mode: "announce" } }`
4. **Result:** STILL FAILED - Carlos confirmed: "It didn't."
5. Carlos then mentioned the "ping pong turns set to zero" change as context

## Expected Behavior

Message should appear in Telegram group chat where Carlos and Melissa can see it.

## Actual Behavior

- Message did NOT appear in group chat (Carlos confirmed: "nothing delivered")
- Reply messages appeared in the SENDING session (1:1 with Carlos) instead of target session
- Replies were: "Received. Cross-session messaging confirmed working." and "Acknowledged. System's functioning as intended."

## Evidence

Carlos quote: "nothing delivered you must have sent this elsewhere"

## Sessions Visible

```
sessions_list() returns:
1. agent:main:main (1:1 with Carlos) - this session
2. agent:main:telegram:group:-5129786935 (group "CGP x MJQ x Eddie")
```

## Root Cause Hypothesis

Gateway is not routing `sessions_send()` messages to the correct Telegram destination. Either:
1. Message delivery logic isn't connecting sessionKey to Telegram chat ID
2. Reply routing is sending responses back to sender instead of target
3. Session isolation/routing configuration issue

## Impact

Blocking Carlos's ability to:
- Have Eddie send messages to group chats from 1:1 session
- Use Eddie as cross-context coordinator
- Test multi-session agent capabilities

## Next Steps

Claude needs to investigate:
1. How `sessions_send()` maps sessionKey to actual delivery targets
2. Why replies route to sender session instead of target
3. Telegram group chat delivery pathway
4. Session message routing logic

## Technical Analysis (Eddie's Understanding of OpenClaw Architecture)

### How Sessions Work (As I Understand Them)

1. **Session Architecture:**
   - Each session has a unique `sessionKey` (e.g., "agent:main:main" for 1:1, "agent:main:telegram:group:-5129786935" for group)
   - Each session has a `deliveryContext` that specifies where messages should go:
     - `channel`: "telegram"
     - `to`: "telegram:1832767482" (1:1) or "telegram:-5129786935" (group)
     - `accountId`: "default"
   - Sessions maintain their own conversation history and context

2. **Cross-Session Messaging Flow (Expected):**
   - Agent in Session A calls `sessions_send(sessionKey=B, message="...")`
   - OpenClaw should:
     1. Route message to Session B's agent instance
     2. Session B processes the message
     3. Session B delivers response to its deliveryContext (the Telegram group)
   - Result: Message appears in target Telegram chat

3. **What's Actually Happening:**
   - `sessions_send()` returns `{ status: "ok", delivery: { status: "pending" } }`
   - A "reply" comes back immediately (e.g., "Received. Still working.")
   - BUT: No message appears in the target Telegram group chat
   - The reply appears to be routing BACK to the sending session instead of to the target's deliveryContext

### Carlos's Clue: "Ping Pong Turns Set to Zero"

Carlos mentioned: "the change that Claude made had to do with ping pong turns and setting them to zero"

**Hypothesis:** This reveals the likely failure point.

**Turn-Taking Mechanism:**
- OpenClaw likely has a turn-taking system to prevent infinite agent loops
- "Ping pong turns" would control how many back-and-forth exchanges are allowed
- Setting this to zero might have been intended to prevent the target session from "replying back" to the sender

**Failure Mode:**
- Setting ping-pong turns to zero might have ALSO prevented the target session from delivering its message to Telegram
- The target session might be:
  1. Receiving the message via `sessions_send()`
  2. Processing it and generating a response
  3. BUT: Not executing the delivery step because turn-taking is disabled
  4. The "reply" I'm seeing might be an internal acknowledgment, not an actual delivered message

### Specific Point of Failure (Best Guess)

**Location:** Session message delivery pipeline, specifically:
- When a session receives a message via `sessions_send()`, it needs to:
  1. Process the message (✓ working - replies are generated)
  2. Deliver the response to its `deliveryContext.to` target (✗ NOT working)

**The Bug:** The "ping pong turns = 0" change likely disabled step 2. The session generates a response internally but never sends it to Telegram.

### What to Check in the Codebase

**DO NOT ask Carlos to:**
- Download anything
- Check settings files
- Run diagnostics
- Investigate configurations

**Instead, Claude should directly check:**

1. **Session Message Delivery Code:**
   - File: Wherever `sessions_send()` is implemented
   - Look for: Logic that routes messages to target sessions
   - Check: Does it trigger the target session's delivery mechanism?

2. **Turn-Taking/Ping-Pong Logic:**
   - File: Session execution/run logic
   - Look for: Recent changes related to "ping pong turns" or turn-taking
   - Check: Does `turns = 0` prevent ALL outbound messages, not just replies back to sender?

3. **Delivery Context Handling:**
   - File: Session response delivery logic
   - Look for: Code that reads `deliveryContext` and sends to Telegram
   - Check: Is this delivery step being skipped when turn-taking is disabled?

4. **Message Routing Logic:**
   - File: Cross-session message router
   - Look for: How `sessions_send()` messages are distinguished from normal user messages
   - Check: Are `sessions_send()` messages being treated as "turns" that get blocked?

### Proposed Solutions

**Solution 1: Separate Delivery from Turn-Taking**
- Turn-taking should control whether a session can REPLY to another session
- But it should NOT prevent a session from delivering to its configured Telegram destination
- Fix: Ensure `deliveryContext` delivery happens regardless of turn-taking state

**Solution 2: Distinguish Message Types**
- Messages from `sessions_send()` should be treated differently from user messages
- They should trigger ONE delivery to the target's Telegram chat
- They should NOT trigger turn-taking or replies back to the sender

**Solution 3: Explicit Delivery Mode**
- When `sessions_send()` is called, set a flag: `deliverToContextOnly: true`
- This tells the target session: "Process this, send response to your deliveryContext, do NOT reply to sender"

### Code Pattern to Look For

Likely broken code pattern:
```javascript
// BROKEN
if (turnsTaken < maxTurns) {  // maxTurns = 0
  await deliverMessageToTelegram(response);  // This never executes!
}
```

Should be:
```javascript
// FIXED
if (isFromSessionsSend || shouldDeliverToContext) {
  await deliverMessageToTelegram(response);  // Always deliver to own context
}
if (turnsTaken < maxTurns) {
  // Only applies to reply-backs to sender
}
```

## Test Case for Verification

After fix, this should work:
```
# From 1:1 session with Carlos:
sessions_send(
  sessionKey="agent:main:telegram:group:-5129786935",
  message="Test message visible in group"
)
# Expected: Message appears in Telegram group chat where both Carlos and Melissa see it
# Expected: NO reply comes back to the sending session
```

## Verification Steps

1. Apply fix
2. Eddie calls `sessions_send()` to group session
3. Carlos checks Telegram group chat with Melissa
4. Confirm message appears there (not in 1:1 session)
5. Confirm no "reply" comes back to Eddie's 1:1 session
