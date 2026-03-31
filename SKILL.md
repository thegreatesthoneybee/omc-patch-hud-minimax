---
name: patch-hud-minimax
description: Patch OMC HUD to support MiniMax API usage display. Use this skill when the user mentions MiniMax API usage not showing in HUD, or when HUD shows API errors for MiniMax users. This skill patches the OMC plugin to add MiniMax support for the usage/quota display feature.
---

# Patch HUD for MiniMax API Support

This skill patches the OMC HUD to display usage information when using MiniMax API.

## When to Use

- User says HUD doesn't show usage for MiniMax API
- User sees "API error" or "no credentials" in HUD when using MiniMax
- User wants to display MiniMax usage quota in the HUD status line

## Prerequisites

1. Check if OMC plugin is installed:
   ```bash
   ls ~/.claude/plugins/cache/omc/oh-my-claudecode/
   ```

2. Find the latest version directory (e.g., `4.9.3`)

## Steps

### 1. Find the usage-api.ts file

```bash
PLUGIN_DIR="$HOME/.claude/plugins/cache/omc/oh-my-claudecode"
LATEST_VERSION=$(ls "$PLUGIN_DIR" | grep -E '^[0-9]+\.[0-9]+\.[0-9]+$' | sort -V | tail -1)
USAGE_API="$PLUGIN_DIR/$LATEST_VERSION/src/hud/usage-api.ts"
echo "Patching: $USAGE_API"
```

### 2. Add MiniMax types (if not already present)

Add this after the `ZaiQuotaResponse` interface definition (around line 93):

```typescript
interface MiniMaxModelRemain {
  start_time: number;          // Unix timestamp in milliseconds
  end_time: number;            // Unix timestamp in milliseconds
  remains_time: number;         // Remaining time in ms
  current_interval_total_count: number;
  current_interval_usage_count: number;
  model_name: string;
  current_weekly_total_count: number;
  current_weekly_usage_count: number;
  weekly_start_time: number;
  weekly_end_time: number;
  weekly_remains_time: number;
}

interface MiniMaxQuotaResponse {
  model_remains?: MiniMaxModelRemain[];
}
```

### 3. Update isZaiHost function

Change the `isZaiHost` function to recognize MiniMax:

```typescript
export function isZaiHost(urlString: string): boolean {
  try {
    const url = new URL(urlString);
    const hostname = url.hostname.toLowerCase();
    return hostname === 'z.ai' || hostname.endsWith('.z.ai') ||
           hostname === 'minimax.io' || hostname.endsWith('.minimax.io');
  } catch {
    return false;
  }
}
```

### 4. Add source type to UsageCache interface

Find `source?: 'anthropic' | 'zai';` and change to:
```typescript
source?: 'anthropic' | 'zai' | 'minimax';
```

This appears in two places in the file (lines ~55 and ~177).

### 5. Update createRateLimitedCacheEntry function signature

Change:
```typescript
source: 'anthropic' | 'zai',
```
to:
```typescript
source: 'anthropic' | 'zai' | 'minimax',
```

### 6. Add fetchUsageFromMiniMax function

Add this after the `fetchUsageFromZai` function:

```typescript
/**
 * Fetch usage from MiniMax API
 */
function fetchUsageFromMiniMax(apiKey: string): Promise<FetchResult<MiniMaxQuotaResponse>> {
  return new Promise((resolve) => {
    // MiniMax API base is always https://api.minimax.io regardless of ANTHROPIC_BASE_URL
    const baseUrl = 'https://api.minimax.io';

    try {
      const quotaLimitUrl = `${baseUrl}/v1/api/openplatform/coding_plan/remains`;
      const urlObj = new URL(quotaLimitUrl);

      const req = https.request(
        {
          hostname: urlObj.hostname,
          path: urlObj.pathname,
          method: 'GET',
          headers: {
            'Authorization': `Bearer ${apiKey}`,
            'Content-Type': 'application/json',
            'Accept-Language': 'en-US,en',
          },
          timeout: API_TIMEOUT_MS,
        },
        (res) => {
          let data = '';
          res.on('data', (chunk) => { data += chunk; });
          res.on('end', () => {
            if (res.statusCode === 200) {
              try {
                resolve({ data: JSON.parse(data) });
              } catch {
                resolve({ data: null });
              }
            } else if (res.statusCode === 429) {
              if (process.env.OMC_DEBUG) {
                console.error(`[usage-api] MiniMax API returned 429 (rate limited)`);
              }
              resolve({ data: null, rateLimited: true });
            } else {
              resolve({ data: null });
            }
          });
        }
      );

      req.on('error', () => resolve({ data: null }));
      req.on('timeout', () => { req.destroy(); resolve({ data: null }); });
      req.end();
    } catch {
      resolve({ data: null });
    }
  });
}
```

### 7. Add parseMiniMaxResponse function

Add this after `parseZaiResponse`:

```typescript
/**
 * Parse MiniMax API response into RateLimits
 */
export function parseMiniMaxResponse(response: MiniMaxQuotaResponse): RateLimits | null {
  const modelRemains = response.model_remains;
  if (!modelRemains || modelRemains.length === 0) return null;

  // Find the text model (MiniMax-M*) - it has 5-hour and weekly limits
  const textModel = modelRemains.find(m => m.model_name.startsWith('MiniMax-M'));
  if (!textModel) return null;

  // Parse Unix timestamp (milliseconds) to Date
  const parseResetTime = (timestamp: number | undefined): Date | null => {
    if (!timestamp) return null;
    try {
      const date = new Date(timestamp);
      return isNaN(date.getTime()) ? null : date;
    } catch {
      return null;
    }
  };

  // Calculate 5-hour remaining percentage (current_interval)
  // Remaining = (total - usage) / total * 100
  const fiveHourPercent = textModel.current_interval_total_count > 0
    ? clamp(Math.floor(((textModel.current_interval_total_count - textModel.current_interval_usage_count) / textModel.current_interval_total_count) * 100))
    : 0;

  // Calculate weekly remaining percentage
  const weeklyPercent = textModel.current_weekly_total_count > 0
    ? clamp(Math.floor(((textModel.current_weekly_total_count - textModel.current_weekly_usage_count) / textModel.current_weekly_total_count) * 100))
    : 0;

  return {
    fiveHourPercent,
    weeklyPercent,
    fiveHourResetsAt: parseResetTime(textModel.end_time),
    weeklyResetsAt: parseResetTime(textModel.weekly_end_time),
  };
}
```

### 8. Update getUsage function

Find the `getUsage` function and add MiniMax detection and handling:

At the beginning of `getUsage`, replace the source detection logic:

```typescript
export async function getUsage(): Promise<UsageResult> {
  const baseUrl = process.env.ANTHROPIC_BASE_URL;
  const apiKey = process.env.ANTHROPIC_API_KEY;
  const authToken = process.env.ANTHROPIC_AUTH_TOKEN;
  const isZaiHostUrl = baseUrl != null && isZaiHost(baseUrl);
  // Differentiate between z.ai and minimax.io
  const isMiniMax = baseUrl?.includes('minimax.io') ?? false;
  const isZai = isZaiHostUrl && !isMiniMax;

  let currentSource: 'anthropic' | 'zai' | 'minimax';
  if (isMiniMax && apiKey) {
    currentSource = 'minimax';
  } else if (isZai && authToken) {
    currentSource = 'zai';
  } else {
    currentSource = 'anthropic';
  }
  // ... rest of function
```

Then add the MiniMax API call block after the z.ai path check. The MiniMax block should look like:

```typescript
  // MiniMax path (uses ANTHROPIC_API_KEY)
  if (isMiniMax && apiKey) {
    const result = await fetchUsageFromMiniMax(apiKey);
    const cachedMiniMax = cache?.source === 'minimax' ? cache : null;

    if (result.rateLimited) {
      const prevLastSuccess = cachedMiniMax?.lastSuccessAt;
      const rateLimitedCache = createRateLimitedCacheEntry('minimax', cachedMiniMax?.data || null, pollIntervalMs, cachedMiniMax?.rateLimitedCount || 0, prevLastSuccess);
      writeCache({
        data: rateLimitedCache.data,
        error: rateLimitedCache.error,
        source: rateLimitedCache.source,
        rateLimited: true,
        rateLimitedCount: rateLimitedCache.rateLimitedCount,
        rateLimitedUntil: rateLimitedCache.rateLimitedUntil,
        errorReason: 'rate_limited',
        lastSuccessAt: rateLimitedCache.lastSuccessAt,
      });
      if (rateLimitedCache.data) {
        if (prevLastSuccess && Date.now() - prevLastSuccess > MAX_STALE_DATA_MS) {
          return { rateLimits: null, error: 'rate_limited' };
        }
        return { rateLimits: rateLimitedCache.data, error: 'rate_limited', stale: true };
      }
      return { rateLimits: null, error: 'rate_limited' };
    }

    if (!result.data) {
      const fallbackData = hasUsableStaleData(cachedMiniMax) ? cachedMiniMax.data : null;
      writeCache({
        data: fallbackData,
        error: true,
        source: 'minimax',
        errorReason: 'network',
        lastSuccessAt: cachedMiniMax?.lastSuccessAt,
      });
      if (fallbackData) {
        return { rateLimits: fallbackData, error: 'network', stale: true };
      }
      return { rateLimits: null, error: 'network' };
    }

    const usage = parseMiniMaxResponse(result.data);
    writeCache({ data: usage, error: !usage, source: 'minimax', lastSuccessAt: Date.now() });
    return { rateLimits: usage };
  }
```

### 9. Build the plugin

```bash
cd "$PLUGIN_DIR/$LATEST_VERSION"
npm run build
```

### 10. Clear cache (if needed)

```bash
rm -f ~/.claude/plugins/oh-my-claudecode/.usage-cache.json
```

## Verification

Test the API directly:
```bash
curl -s -H "Authorization: Bearer $ANTHROPIC_API_KEY" \
  "https://api.minimax.io/v1/api/openplatform/coding_plan/remains" | \
  python3 -m json.tool
```

Should return JSON with `model_remains` array.

## Notes

- MiniMax API endpoint: `https://api.minimax.io/v1/api/openplatform/coding_plan/remains`
- Authentication: Uses `ANTHROPIC_API_KEY` environment variable
- The HUD displays **remaining** percentage (not used percentage)
- MiniMax has 5-hour and weekly limits
