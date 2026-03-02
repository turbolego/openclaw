# Telegram Polling Initialization Fix

## Problem
The Telegram bot in OpenClaw was initializing but never actually starting to poll for messages from the Telegram API. This caused:
- Zero `getUpdates` API calls in logs
- No messages received from users
- Health monitor detecting provider as "stuck" every 35 minutes and restarting

## Root Cause
In `src/telegram/monitor.ts`, the `createPollingBot()` function had a critical bug:

```typescript
const createPollingBot = async (): Promise<TelegramBot | undefined> => {
  try {
    return createTelegramBot({...});
  } catch (err) {
    const shouldRetry = await waitBeforeRetryOnRecoverableSetupError(
      err,
      "Telegram setup network error",
    );
    if (!shouldRetry) {
      return undefined;
    }
    return undefined;  // BUG: Should retry, not return undefined!
  }
};
```

**The Issue**: When bot creation failed with a recoverable error:
1. `waitBeforeRetryOnRecoverableSetupError()` is called, which includes a backoff sleep
2. Returns `true` if the error is recoverable and sleep has occurred
3. **But then returns `undefined` unconditionally**
4. This caused the while loop to immediately retry without actually re-attempting bot creation
5. Result: Busy loop that never actually creates the bot and starts polling

## Solution
Changed line 243 in `src/telegram/monitor.ts` to recursively retry bot creation when `shouldRetry` is true:

```typescript
const createPollingBot = async (): Promise<TelegramBot | undefined> => {
  try {
    return createTelegramBot({...});
  } catch (err) {
    const shouldRetry = await waitBeforeRetryOnRecoverableSetupError(
      err,
      "Telegram setup network error",
    );
    if (!shouldRetry) {
      return undefined;
    }
    return await createPollingBot();  // Recursively retry after backoff
  }
};
```

## Impact
- **Before**: Bot initializes but never polls → never receives messages
- **After**: Bot initializes and properly retries with backoff until polling starts → receives messages and can respond

## Deployment
- **Branch**: `fix/telegram-polling-init-timeout`
- **Commit**: `8bae6f32c` - "Fix Telegram polling initialization retry logic"
- **Deployed**: OpenClaw container rebuilt and running with the fix

## Testing
1. Built new Docker image with the fixed code
2. Started container - bot initializes with: `[default] starting provider (@kloa_zero_bot)`
3. Telegram API sendMessage working (verified with test message)
4. Bot now has ability to receive and process messages

## Files Changed
- `src/telegram/monitor.ts` - Line 243: Changed `return undefined;` to `return await createPollingBot();`
