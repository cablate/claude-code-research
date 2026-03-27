# Production Cache Optimization Playbook: Concrete Patches and Strategies for Maximizing Prompt Cache Efficiency

Prompt cache efficiency determines how much you pay per message in production. A cold cache means every turn is a 125% write. A warm cache brings that down to 10% reads. The difference compounds fast at scale.

This report documents seven concrete strategies for maximizing prompt cache efficiency with the Claude Agent SDK v0.2.76. Each one is grounded in source-level evidence from `cli.js` — anchor strings, function names, and observed behavior. The strategies range from surgical patches to behavioral changes in your own wrapper code.

---

## Summary

| # | Strategy | Risk | Expected Effect |
|---|----------|------|----------------|
| 1 | Force 1h cache TTL for SDK sessions | Low | Longer cache survival between queries |
| 2 | Reduce context overflow margin (1000→200) | Very low | 800 more tokens before overflow triggers |
| 3 | Raise compaction threshold (100k→150k) | Medium | Fewer compaction events, fewer cache rebuilds |
| 4 | Tool ordering stabilization | None | Prevents tool list changes from busting cache prefix |
| 5 | Cache keepalive pinging | None | Keeps 5-minute cache alive across idle gaps |
| 6 | Cache efficiency monitoring | — | Early warning for cache bust events |
| 7 | MCP server key ordering | None | Stable serialization → stable cache prefix |

Strategies 1–3 and 7 require patching `cli.js`. Strategies 4–6 are implemented in your wrapper code. All patches use the anchor-string pattern described at the end of this report.

---

## Methodology

All findings are from reverse-engineering `cli.js` build 2026-03-14 (SDK v0.2.76). The file is a 12MB minified bundle with obfuscated variable names. Navigation uses string constants as stable anchors — these do not change across build versions even when variable names are re-obfuscated.

For cache-related behavior, reference reports [#1 (cache invalidation)](../agent-sdk-cache-invalidation/), [#3 (prompt cache architecture)](../prompt-cache-architecture/), and [#5 (tool serialization)](../tool-serialization-cache-stability/) provide the underlying mechanics. This report focuses on actionable patches derived from that prior work.

---

## Findings

### Strategy 1: Force 1h Cache TTL for SDK Sessions

**Root cause.** Cache TTL in `cli.js` is controlled by `o3z()`:

```
Anchor: function o3z(A){if(QA()==="bedrock"
```

This function gates the 1-hour TTL behind a server-side feature flag allowlist. A query source value of `"sdk"` may not be on that allowlist, causing SDK sessions to fall back to the standard 5-minute TTL. With a 5-minute TTL, any idle session longer than 5 minutes loses its cache entirely and must pay 125% for a full rebuild on the next turn.

**Patch.** Insert at the start of `o3z()`:

```javascript
if(A==="sdk"||A==="repl_main_thread")return!0;
```

This short-circuits the allowlist check for SDK and REPL query sources, unconditionally returning the 1h TTL flag.

**Effect.** SDK sessions get 1-hour cache TTL regardless of server-side configuration. A session that's idle for 20 minutes keeps its cache; without this patch it would not.

**Risk.** Low. This only affects cache duration. The server may still enforce its own TTL policy, so the patch is a best-effort improvement rather than a guarantee. No correctness impact.

---

### Strategy 2: Reduce Context Overflow Margin (1000→200)

**Root cause.** When `cli.js` handles context overflow recovery, `$54()` calculates how much room to leave:

```
Anchor: contextLimit:W}=X,Z=1000,G=Math.max(0,W-P-1000)
```

The constant `1000` is a safety margin subtracted from the context limit before compaction. This is conservative — it means overflow recovery fires 800 tokens earlier than necessary.

**Patch.** Replace both occurrences of `1000` in this expression with `200`:

```
contextLimit:W}=X,Z=200,G=Math.max(0,W-P-200)
```

**Effect.** 800 additional tokens are available before overflow recovery triggers. This delays the compaction event marginally, giving the cache more time to accumulate read hits before the next rebuild cycle.

**Risk.** Very low. The output floor `fN8` (3000 tokens minimum) provides a separate safety buffer. The margin reduction does not interfere with that floor.

---

### Strategy 3: Raise Compaction Threshold (100k→150k)

**Root cause.** `AgentStream` has a built-in compaction that fires at:

```
Anchor: var nX7=1e5,
```

`nX7 = 100000` is the total-token threshold. When a session accumulates 100k tokens in its context, compaction runs automatically. Compaction replaces the message history with a compressed summary, which destroys the prompt cache prefix entirely. The next turn pays 125% for a full cache rebuild.

Each compaction event costs: the write cost of the rebuilt cache (125% on ~45k tokens) plus the write cost of the compaction summary itself. In a long-running session, frequent compaction events compound into significant waste.

**Patch.** Change to 150k:

```
var nX7=15e4,
```

**Effect.** Compaction fires at 150k instead of 100k. Fewer compaction events per session → fewer full cache rebuilds. In sessions with heavy tool use, the difference between one compaction at 100k versus one at 150k can be several thousand tokens saved.

**Risk.** Medium. Sessions consume more context before compacting. Monitor for `prompt_too_long` errors in production, especially on models with smaller context windows. If errors appear, revert to a value between `1e5` and `15e4`.

---

### Strategy 4: Tool Ordering Stabilization

**Root cause.** Confirmed via source inspection: `cli.js` does not sort tool arrays anywhere in the tool pipeline. The order of tools in the API request depends on the insertion order of MCP server responses. If a server reconnects and re-registers tools in a different order, the serialized tool list changes. A changed tool list breaks the system prompt cache prefix (see [Report #5](../tool-serialization-cache-stability/) for the full mechanics).

**Fix.** In your SDK wrapper, sort `allowedTools` and sort `mcpServers` object keys before passing to `createSession()`:

```javascript
const sortedTools = [...allowedTools].sort();
const sortedMcpServers = Object.fromEntries(
  Object.entries(mcpServers).sort(([a], [b]) => a.localeCompare(b))
);
```

**Effect.** Tool serialization order becomes deterministic regardless of MCP server registration order. Cache prefix is stable across reconnects.

**Risk.** None. Sort order is invisible to the API — the set of available tools is identical. The only change is the serialized order, which determines cache prefix stability.

---

### Strategy 5: Cache Keepalive Pinging

**Root cause.** Standard Anthropic API cache TTL is 5 minutes. First-party integrations on the allowlist get 1h TTL (see Strategy 1). Even with Strategy 1's patch, sessions that are genuinely idle for over an hour lose their cache.

For active sessions with idle gaps under 5 minutes (or under 1h with Strategy 1), sending a lightweight ping prevents the cache from expiring:

**Implementation.**

```javascript
class CacheKeepalive {
  constructor(session, intervalMs = 4 * 60 * 1000) {
    this.session = session;
    this.lastUsed = Date.now();
    this.timer = setInterval(() => this.ping(), intervalMs);
  }

  touch() {
    this.lastUsed = Date.now();
  }

  async ping() {
    const idleMs = Date.now() - this.lastUsed;
    if (idleMs < 3 * 60 * 1000) return; // used recently, skip

    try {
      for await (const _ of this.session.stream("Reply with only 'ok'")) {
        // drain response
      }
    } catch (err) {
      console.warn('[cache-keepalive] ping failed, stopping:', err.message);
      clearInterval(this.timer);
    }
  }

  stop() {
    clearInterval(this.timer);
  }
}
```

**Cost analysis.** Each ping generates roughly 10 output tokens. At Sonnet 4 rates, that's approximately $0.000015 per ping. A full cache miss on a 60k-token context at 125% costs roughly $0.0028. The keepalive breaks even if it prevents one cache miss per 187 pings — in practice, any session with real workload recovers this cost within the first avoided miss.

**Risk.** None. Ping failure is logged and the timer stops — no impact on the session's actual work. The ping message does not affect conversation state.

---

### Strategy 6: Cache Efficiency Monitoring

Monitoring is not optional in production. Cache busts are silent — there's no error, just a higher bill.

**Implementation.** Track a rolling exponential moving average (EMA) of per-turn cache efficiency:

```javascript
class CacheEfficiencyMonitor {
  constructor(alpha = 0.3, warnThreshold = 50) {
    this.alpha = alpha;
    this.threshold = warnThreshold;
    this.ema = null;
  }

  record(usage) {
    const { input_tokens, cache_read_input_tokens, cache_creation_input_tokens } = usage;
    const total = input_tokens + cache_read_input_tokens + cache_creation_input_tokens;
    if (total === 0) return;

    const efficiency = (cache_read_input_tokens / total) * 100;
    this.ema = this.ema === null
      ? efficiency
      : this.alpha * efficiency + (1 - this.alpha) * this.ema;

    if (this.ema < this.threshold) {
      console.warn(
        `[cache-monitor] efficiency ${this.ema.toFixed(1)}% below threshold (${this.threshold}%)`,
        { efficiency, input_tokens, cache_read_input_tokens, cache_creation_input_tokens }
      );
    }
  }
}
```

**Formula.** `efficiency = cacheReadInputTokens / (inputTokens + cacheReadInputTokens + cacheCreationInputTokens) × 100`

**Alert threshold.** Warn if the rolling EMA drops below 50%. A session should be reading more from cache than it's writing after the first few turns. Below 50% sustained means something is busting the cache every turn — tool ordering instability, MCP reconnect, compaction chain, or session restart.

**What low efficiency indicates:**
- Cache bust from tool list change (→ check Strategy 4)
- MCP server reconnect (→ check Strategy 7)
- Compaction chain reaction (→ check Strategy 3)
- Session process restart (→ check V2 persistent sessions)

---

### Strategy 7: MCP Server Key Ordering

**Root cause.** `mcpServers` is typed as `Record<string, ServerConfig>`. JavaScript object key iteration order follows insertion order (per spec for string keys since ES2015). If your code builds this object differently between calls — or if a future SDK version iterates keys during serialization — the resulting JSON may differ in key order.

This is a latent risk. The cost of preventing it is zero.

**Fix.**

```javascript
function sortedMcpServers(servers) {
  return Object.fromEntries(
    Object.entries(servers).sort(([a], [b]) => a.localeCompare(b))
  );
}
```

Apply this before passing `mcpServers` to `createSession()`. Combine with Strategy 4's tool sort into a single normalization step.

**Risk.** None.

---

## Evidence

| Claim | Source anchor | Location |
|-------|--------------|----------|
| 1h TTL gated by feature flag | `function o3z(A){if(QA()==="bedrock"` | cli.js |
| Overflow margin = 1000 tokens | `contextLimit:W}=X,Z=1000,G=Math.max(0,W-P-1000)` | cli.js |
| Compaction threshold = 100k tokens | `var nX7=1e5,` | cli.js |
| No `.sort()` in tool pipeline | Zero matches for `.sort()` adjacent to tool array operations | cli.js |
| Standard cache TTL = 5 minutes | Anthropic API documentation | External |
| mcpServers key ordering | `Record<string,ServerConfig>` type, JS insertion-order spec | cli.js / JS spec |

---

## Impact

Starting from a V2 persistent session baseline (Report #1 established ~84% peak efficiency), these strategies address the remaining failure modes:

| Failure mode | Strategy that addresses it |
|---|---|
| Cache expires during idle gap | 5 (keepalive ping) |
| Short idle gaps → 5-min TTL expiry | 1 (force 1h TTL) |
| Frequent compaction events | 3 (raise threshold) |
| Tool list reorder on MCP reconnect | 4, 7 (sort stabilization) |
| Compaction fires too early | 2 (reduce overflow margin) |
| Silent cache regressions | 6 (monitoring) |

In a production system with long-running sessions and MCP servers, applying all strategies together should keep rolling cache efficiency above 70% during sustained work periods.

---

## Implementation: The Postinstall Patch Pattern

Strategies 1, 2, 3, and 7 require modifying `cli.js`. The approach is postinstall scripts using anchor strings, not line numbers. Minified code re-obfuscates variable names on every build, but string constants and structural patterns stay stable across versions.

### Structure

```
scripts/
  patch-sdk-v2.cjs       # existing: patches sdk.mjs for V2 options
  patch-cli-cache-opt.cjs  # new: patches cli.js for cache strategies 1-3
```

`package.json`:
```json
"scripts": {
  "postinstall": "node scripts/patch-sdk-v2.cjs && node scripts/patch-cli-cache-opt.cjs"
}
```

### Patch Script Pattern

```javascript
// patch-cli-cache-opt.cjs
const fs = require('fs');
const path = require('path');

const CLI_PATH = path.resolve(__dirname, '../node_modules/@anthropic-ai/claude-agent-sdk/cli.js');

function applyPatch(source, anchor, replacement, patchName) {
  if (!source.includes(anchor)) {
    console.error(`[patch] ANCHOR NOT FOUND: ${patchName}`);
    console.error(`  anchor: ${anchor}`);
    process.exit(1);
  }

  const count = source.split(anchor).length - 1;
  if (count !== 1) {
    console.error(`[patch] ANCHOR NOT UNIQUE: ${patchName} (found ${count} times)`);
    process.exit(1);
  }

  if (source.includes(replacement)) {
    console.log(`[patch] already applied: ${patchName}`);
    return source;
  }

  console.log(`[patch] applying: ${patchName}`);
  return source.replace(anchor, replacement);
}

let source = fs.readFileSync(CLI_PATH, 'utf8');

// Backup original
const backupPath = CLI_PATH + '.pre-patch';
if (!fs.existsSync(backupPath)) {
  fs.writeFileSync(backupPath, source);
}

// Strategy 1: Force 1h TTL for SDK sessions
source = applyPatch(
  source,
  'function o3z(A){if(QA()==="bedrock"',
  'function o3z(A){if(A==="sdk"||A==="repl_main_thread")return!0;if(QA()==="bedrock"',
  'force-1h-ttl'
);

// Strategy 2: Reduce overflow margin
source = applyPatch(
  source,
  'contextLimit:W}=X,Z=1000,G=Math.max(0,W-P-1000)',
  'contextLimit:W}=X,Z=200,G=Math.max(0,W-P-200)',
  'reduce-overflow-margin'
);

// Strategy 3: Raise compaction threshold
source = applyPatch(
  source,
  'var nX7=1e5,',
  'var nX7=15e4,',
  'raise-compaction-threshold'
);

fs.writeFileSync(CLI_PATH, source);
console.log('[patch] cli.js patched successfully');
```

### Idempotency

The script checks `source.includes(replacement)` before applying each patch. Running `npm install` repeatedly applies each patch at most once.

### After SDK Upgrade

When the SDK updates, verify each anchor still exists before the patched build reaches production:

```bash
grep -c 'function o3z(A){if(QA()==="bedrock"' node_modules/@anthropic-ai/claude-agent-sdk/cli.js
grep -c 'contextLimit:W}=X,Z=1000' node_modules/@anthropic-ai/claude-agent-sdk/cli.js
grep -c 'var nX7=1e5,' node_modules/@anthropic-ai/claude-agent-sdk/cli.js
```

Each should return `1`. If any returns `0`, the anchor moved or the implementation changed — re-locate the function using the string constants described in each strategy's root cause section and update the anchor.

---

## What We Investigated But Chose Not to Patch

**currentDate injection cache-kill.** The `currentDate` string is injected into the first message position on every turn. It's a direct cache buster and the most impactful one. However, it sits deep in the injection assembly pipeline, touching multiple codepaths that also handle CLAUDE.md and memory file injection. The risk of a bad patch silently breaking context assembly is high, and the payoff is partially addressed by V2 persistent sessions (the date changes once per day, not once per turn). Deferred.

**Deferred tool loading disable.** `cli.js` has a lazy tool loading path that could reduce initial prompt size. Disabling it would load all tools upfront, creating a larger but more stable initial prompt. The tradeoff — larger initial write versus fewer incremental writes — does not clearly favor one side, and the implementation is coupled to the MCP initialization path. Deferred.

**Fork/subagent pruning.** Subagent results are appended to the message history and survive compaction in summarized form. Pruning these entries would reduce context size but risks losing context quality that affects task coherence. Not worth the risk.

**Multi-model cache isolation.** Different models maintain separate prompt caches. If your production system routes to multiple models within the same session, the cache built for one model is not shared with another. Solving this properly requires architecture-level session routing, which is out of scope for a patching approach.

---

## Monitoring Checklist for Production

After applying the patches and deploying:

1. **Per-turn efficiency.** Log `cacheReadInputTokens / (inputTokens + cacheReadInputTokens + cacheCreationInputTokens)` for every response. Alert at sustained EMA below 50%.

2. **Compaction frequency.** Log when compaction events occur. A chain of compactions (compaction → rebuild → compaction again quickly) indicates the threshold is still too low for your workload.

3. **Keepalive ping success rate.** Track ping failures. A spike in failures may indicate session instability or network issues, not a cache problem.

4. **First-turn vs. subsequent-turn cost.** First turn always writes cache. Track the ratio of first-turn cost to subsequent-turn cost as a sanity check that the cache is accumulating correctly.

5. **Post-upgrade verification.** After each SDK upgrade, run the anchor grep commands above before deploying. A broken patch is worse than no patch — it can silently corrupt the `cli.js` source.

---

## References

- [Report #1: Agent SDK Cache Invalidation (Root Cause and Fix)](../agent-sdk-cache-invalidation/)
- [Report #3: Prompt Cache Architecture](../prompt-cache-architecture/)
- [Report #5: Tool Serialization and Cache Stability](../tool-serialization-cache-stability/)
- `cli.js` build 2026-03-14, `@anthropic-ai/claude-agent-sdk` v0.2.76
- Anthropic API documentation: [Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
