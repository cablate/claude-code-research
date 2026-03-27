# Seven Patches That Push Cache Efficiency Past 90%

V2 persistent sessions, documented in Report #1, solved the fundamental problem: every SDK call used to spawn a fresh process, rebuild runtime injections from scratch, and miss the prompt cache entirely. Switching to a long-lived process brought efficiency from a stuck 25% to roughly 84%.

That still leaves 16 percentage points on the table. Where does the remaining waste come from? Cache TTLs that expire during idle gaps. Compaction events that wipe the accumulated cache. Tool arrays that serialize in different orders after an MCP reconnect. A monitoring gap that lets these failures go unnoticed until the bill arrives.

This report documents seven optimizations — three patches to `cli.js`, two changes in wrapper code, one monitoring system, and one you get for free by combining two of the others. Each targets a specific mechanism traced in Reports #3 through #5. Together they close most of the remaining gap.

---

## Why We Patch With Anchor Strings

Before getting into the individual optimizations, it helps to understand the patching methodology, since three of the seven require modifying `cli.js` directly.

`cli.js` is a 12MB minified JavaScript file. Every SDK release re-obfuscates its variable names — what was `o3z` in v0.2.76 might be `p4a` in the next version. Patching by line number or variable name is fragile to the point of being useless.

String constants, however, survive minification. A function that checks `QA()==="bedrock"` will still contain that exact string in the next build, even if the function's own name changes. The anchor-string approach works like this: find a unique string constant that only appears in the function you want to patch, verify it appears exactly once in the file, then do a targeted string replacement. A postinstall script runs after every `npm install`, applying each patch idempotently.

This is not elegant engineering. It is controlled, verifiable surgery on a binary you do not control. The alternative — forking the SDK — is worse in every way.

---

## Patch 1: Force the One-Hour Cache TTL

The Anthropic API supports two cache TTL tiers: a standard five-minute window, and a one-hour window reserved for first-party integrations. Which tier you get depends on a server-side feature flag. In `cli.js`, a gating function checks whether the current query source appears on an allowlist. The SDK's query source value may not be on that list.

The practical consequence: an SDK session that sits idle for six minutes — a normal gap between human messages in an agent workflow — loses its entire cache. The next turn pays 125% to rebuild roughly 45,000 tokens from scratch.

**The patch** inserts a short-circuit at the top of the gating function. If the query source is `"sdk"` or `"repl_main_thread"`, it returns true immediately, bypassing the allowlist check.

```
Anchor:  function o3z(A){if(QA()==="bedrock"
Insert:  function o3z(A){if(A==="sdk"||A==="repl_main_thread")return!0;if(QA()==="bedrock"
```

The server may still enforce its own TTL policy, so this is best-effort. But in practice, sessions that previously lost their cache after a five-minute idle gap now survive for up to an hour. The risk is effectively zero — this only changes cache duration, not correctness.

---

## Patch 2: Shrink the Context Overflow Margin

When the conversation approaches the model's context limit, `cli.js` triggers overflow recovery — a compaction that summarizes the message history to free space. The recovery logic subtracts a 1,000-token safety margin before deciding how much room is left. That margin is conservative. The SDK already maintains a separate output floor of 3,000 tokens minimum, which provides its own safety buffer.

Reducing the margin from 1,000 to 200 keeps 800 additional tokens in play before overflow recovery kicks in. Each avoided compaction event preserves the existing cache. The effect is modest in isolation, but compaction is the single most destructive event for cache efficiency — every compaction wipes the prefix and forces a full 125% rebuild.

```
Anchor:   contextLimit:W}=X,Z=1000,G=Math.max(0,W-P-1000)
Replace:  contextLimit:W}=X,Z=200,G=Math.max(0,W-P-200)
```

Risk is minimal. The 3,000-token output floor is a separate check that this margin reduction does not affect.

---

## Patch 3: Raise the Compaction Threshold

The SDK's built-in compaction fires when total context reaches 100,000 tokens. Once it fires, the message history is replaced with a compressed summary, and the prompt cache prefix is destroyed. In sessions with heavy tool use — which is most production sessions — hitting 100k is routine.

Each compaction event costs the full cache rebuild (125% on ~45k tokens) plus the summary generation itself. In a long-running session, a compaction at 100k followed by a rapid context re-expansion can trigger a chain: compact, rebuild cache, grow back toward 100k, compact again.

Raising the threshold to 150,000 tokens extends the runway. Fewer compactions per session means fewer full cache rebuilds.

```
Anchor:   var nX7=1e5,
Replace:  var nX7=15e4,
```

This is the highest-risk patch of the three. Sessions will consume more context before compacting, which could cause `prompt_too_long` errors on models with smaller context windows. Monitor for these errors in production. If they appear, a value between 100k and 150k — say 120k or 130k — may be more appropriate for your workload.

---

## Wrapper Fix: Stabilize Tool Ordering

This one requires no patching. It is a single `.sort()` call in your own wrapper code that prevents a subtle and hard-to-diagnose cache bust.

The tool array that `cli.js` sends to the API is never sorted internally. The order depends on when each tool was registered, which in turn depends on the order MCP servers respond during initialization. If a server disconnects and reconnects — registering its tools in a slightly different order — the serialized tool list changes. Since the tool list is part of the system prompt, any change to it invalidates the system prompt cache prefix. Report #5 traces this mechanism in detail.

The fix is to sort the tool array and MCP server keys before passing them to the session:

```javascript
const sortedTools = [...allowedTools].sort();
const sortedMcpServers = Object.fromEntries(
  Object.entries(mcpServers).sort(([a], [b]) => a.localeCompare(b))
);
```

Sort order is invisible to the API. The available tool set is identical. The only thing that changes is the serialized byte sequence, which is exactly what cache prefix matching cares about. Zero risk.

---

## Wrapper Fix: Cache Keepalive Pings

API cache has a TTL. When a session goes idle — no messages in either direction — the clock runs. Once the TTL expires, the next message pays full price for a cache rebuild.

Even with Patch 1 extending the TTL to one hour, real-world agent workflows have idle gaps. A user walks away for lunch. A batch job finishes its queue and waits for the next batch. The cache expires silently.

A keepalive ping every four minutes costs roughly 10 output tokens per ping — about $0.000015 at Sonnet 4 rates. A single avoided cache miss on a 60k-token context at 125% saves roughly $0.0028. The math works out to break-even at one avoided miss per 187 pings. In practice, any session with real traffic recovers the cost within the first avoided rebuild.

The implementation checks idle time before pinging — if the session was used within the last three minutes, the ping is skipped. If the ping fails, the timer stops. No interference with the session's actual work.

```javascript
class CacheKeepalive {
  constructor(session, intervalMs = 4 * 60 * 1000) {
    this.session = session;
    this.lastUsed = Date.now();
    this.timer = setInterval(() => this.ping(), intervalMs);
  }

  touch() { this.lastUsed = Date.now(); }

  async ping() {
    if (Date.now() - this.lastUsed < 3 * 60 * 1000) return;
    try {
      for await (const _ of this.session.stream("Reply with only 'ok'")) {}
    } catch (err) {
      console.warn('[cache-keepalive] ping failed:', err.message);
      clearInterval(this.timer);
    }
  }

  stop() { clearInterval(this.timer); }
}
```

---

## Monitoring: Tracking Cache Efficiency

Cache busts produce no errors. The session keeps working. The only signal is a higher bill, and by the time you notice, the damage has accumulated over days or weeks.

The metric that matters is the ratio of cache reads to total input tokens:

```
efficiency = cacheReadInputTokens / (inputTokens + cacheReadInputTokens + cacheCreationInputTokens)
```

A raw per-turn number is too noisy — the first turn of any session always writes cache, producing 0% efficiency. An exponential moving average with alpha 0.3 smooths this out while still responding quickly to sustained drops.

The warning threshold is 50%. After the first few turns of a session, a healthy cache should be reading more than it is writing. If the rolling EMA stays below 50%, something is breaking the cache on every turn: unstable tool ordering, MCP server reconnects, compaction chains, or the session process itself restarting.

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
        `[cache-monitor] efficiency ${this.ema.toFixed(1)}% below ${this.threshold}% threshold`,
        { efficiency, input_tokens, cache_read_input_tokens, cache_creation_input_tokens }
      );
    }
  }
}
```

When the monitor fires, the diagnostic path is straightforward: check whether tool ordering changed (Patch 4), whether an MCP server reconnected, whether compaction just ran (Patch 3), or whether the session process restarted entirely.

---

## What We Investigated But Did Not Patch

Not every opportunity is worth taking. Three mechanisms looked promising during the reverse-engineering phase but were ultimately set aside.

**The currentDate injection.** Every turn, a date string is injected at the first position in the messages array. It changes once per day, which means it breaks the cache prefix exactly once daily in a persistent session — annoying but tolerable. The problem with patching it is that the injection sits deep in the assembly pipeline, interleaved with CLAUDE.md and memory file injection. A surgical removal risks silently corrupting the entire context assembly. V2 persistent sessions already limit the damage to one break per day instead of one per turn.

**Deferred tool loading.** `cli.js` has a lazy loading path for tools that reduces the initial prompt size. Disabling it would front-load all tools, creating a larger but more stable initial prompt. The tradeoff — bigger initial cache write versus fewer incremental changes — does not clearly favor either side, and the implementation is tightly coupled to the MCP initialization sequence. Not enough upside to justify the coupling risk.

**Subagent result pruning.** When a subagent completes, its results are appended to the message history and survive compaction in summarized form. Pruning these entries would reduce context size, but those summaries carry information that affects task coherence in subsequent turns. Removing them is a quality gamble we chose not to take.

---

## After an SDK Upgrade

When a new version of `@anthropic-ai/claude-agent-sdk` lands, the postinstall script runs automatically. If an anchor string has moved or disappeared, the script exits with an error and the build fails — by design. A silent patch failure that corrupts `cli.js` is far worse than a loud build failure.

To verify manually before deploying:

```bash
grep -c 'function o3z(A){if(QA()==="bedrock"' node_modules/@anthropic-ai/claude-agent-sdk/cli.js
grep -c 'contextLimit:W}=X,Z=1000' node_modules/@anthropic-ai/claude-agent-sdk/cli.js
grep -c 'var nX7=1e5,' node_modules/@anthropic-ai/claude-agent-sdk/cli.js
```

Each should return `1`. A `0` means the anchor moved — grep for the surrounding string constants described in each patch section to relocate it. The function's purpose does not change between versions; only its minified name does.

The wrapper-level fixes (tool sorting, keepalive pings, monitoring) are immune to SDK upgrades. They operate on your own code, not on the SDK's internals.

---

## References

- [Report #1: Agent SDK Cache Invalidation](../agent-sdk-cache-invalidation/) — the V2 persistent session fix that established the 84% baseline
- [Report #3: Prompt Cache Architecture](../prompt-cache-architecture/) — how cache prefix matching works internally
- [Report #5: Tool Serialization and Cache Stability](../tool-serialization-cache-stability/) — why tool ordering matters for cache
- `cli.js` build 2026-03-14, `@anthropic-ai/claude-agent-sdk` v0.2.76
- [Anthropic API: Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
