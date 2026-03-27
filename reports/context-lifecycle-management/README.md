# Context Lifecycle Management — How Claude Code Decides When to Compress, What to Keep, and What It Costs

A long conversation runs until Claude Code silently compresses it. A cost spike appears in your dashboard. The next turn is slower. If you are running an automated agent loop, you may see the whole thing repeat two turns later.

This is autocompact. It is not documented. This report reverse-engineers exactly how it works, when it fires, what survives, and what it costs — including two mechanisms that, as far as we can find, have not been described publicly before.

SDK version analyzed: `@anthropic-ai/claude-agent-sdk` v0.2.76 (cli.js build 2026-03-14)

---

## Methodology

All findings are from direct source analysis of `cli.js` — the 12MB minified engine that backs both Claude Code CLI and the Agent SDK. Variable names are obfuscated; functions are located via string constant anchors.

Key anchors used:

- `"DISABLE_AUTO_COMPACT"` → threshold logic and trigger gate
- `"session_memory"` → recursion guard inside the compaction flow
- `"compact"` → query source tag used during compaction
- `"prompt_too_long"` → reactive compact entry point
- `"input length and max_tokens exceed context limit"` → context overflow recovery parser
- `"Today's date is"` → currentDate injection site

Each finding is followed by the constant name or call trace from the source.

---

## Findings

### 1. Context Window Size Is Calculated, Not Assumed

Claude Code does not assume a fixed context window. It computes the effective available space per-model at runtime.

**`uM(model, betas)`** determines the raw context window:

- Model name contains `[1m]` or `1M` beta feature flag → 1,000,000 tokens
- `sonnet-4-6` + HLA (high-latency-allowance) beta → 1,000,000 tokens
- All other models → 200,000 tokens

**`oa(model)`** determines max output tokens per model:

| Model | Max Output |
|---|---|
| claude-opus-4-6 | 64,000 |
| claude-opus-4-5 | 32,000 |
| claude-opus-4 | 32,000 |
| claude-3-opus | 4,096 |
| others | varies |

**`OF(model)`** computes the effective usable window:

```
effectiveWindow = contextWindow - maxOutputTokens - RmY(20,000)
```

The 20,000 token `RmY` constant is a permanent reservation subtracted from every window size calculation. This means a 200k-window model has an effective ceiling of roughly 164,000 tokens for input before any thresholds kick in.

---

### 2. Five Hardcoded Threshold Constants Govern All Decisions

```
RmY = 20,000   max output tokens reservation (subtracted from every window)
Jp8 = 13,000   autocompact safety margin
hmY = 20,000   warning threshold margin
SmY = 20,000   error threshold margin
Mp8 = 3,000    hard blocking limit margin
aqq = 3        circuit breaker: max consecutive compaction failures
```

These are baked into the minified source. There is no config file, no settings key, no UI control. The only overrides are environment variables (see section 13).

**`mz6(tokens, model)`** evaluates all thresholds together in a single pass and returns a state struct:

- `isAboveAutoCompactThreshold` — triggers soft compaction
- `isAtWarningLevel` — UI indicator only
- `isAtErrorLevel` — UI indicator only
- `isAtBlockingLimit` — hard stop, no new messages accepted

**`oc6(model)`** derives the autocompact trigger point:

```
autocompactThreshold = OF(model) - Jp8(13,000)
```

For a standard 200k model: `(200,000 - 4,096 - 20,000) - 13,000 = 162,904 tokens`.

---

### 3. The Gate Before the Gate

Before autocompact even checks token counts, three separate switches must all be clear.

**`Xh()`** checks:

1. `DISABLE_COMPACT` environment variable is not set
2. `DISABLE_AUTO_COMPACT` environment variable is not set
3. `settings.autoCompactEnabled` is not false

If any of the three is active, autocompact is skipped entirely for that turn.

**`CmY()`** is the pre-execution guard that runs immediately before the compaction attempt:

- If `querySource` is `"session_memory"` → skip (session memory refresh would recurse)
- If `querySource` is `"compact"` → skip (already compacting, would create infinite loop)

This is a recursion guard. Compaction is itself implemented as an LLM call with `querySource: "compact"`. Without this check, a large compaction summary could trigger another compaction.

---

### 4. The 10-Step Compaction Flow

When all gates are clear and tokens exceed the threshold, **`mf6()`** executes. The steps in order:

1. Record `preCompactTokenCount` via `eW(messages)` — captures token count before state changes
2. Run PreCompact hooks — if any hook exits with code 2, compaction is cancelled entirely
3. Clear `readFileState` cache — file tracking table is wiped (forces fresh rebuild on next turn)
4. Rebuild memory files and tool definition attachments
5. Build a 10-section summary prompt using `C54(customInstructions)`
6. Call `Gqq()` with `querySource: "compact"` and `maxTurns: 1` — the actual LLM summarization call
7. Construct a continuation opener message `sF6(summary)` as the new conversation start
8. Measure `postCompactTokenCount` via `Ck()` — reads real token count from API response usage field
9. Estimate `truePostCompactTokenCount` via `GF6()` — character-length estimate for UI display
10. Run PostCompact hooks

The compaction call itself is a real API call. It costs tokens. The summary it produces replaces everything that came before it.

---

### 5. What the Summary Prompt Asks For

The 10-section prompt passed to the LLM during compaction:

1. **Environment/Setup** — OS, shell, tool versions
2. **Actual Changes Made** — files created, modified, deleted with specifics
3. **Files and Code Sections** — what was read, why it mattered
4. **Problem Solving** — errors encountered, hypotheses tried, solutions found
5. **Pending Work / Next Steps** — incomplete tasks, blockers
6. **User Decisions and Preferences** — explicit choices made during the session
7. **Context / Environment** — project structure, constraints
8. **Current Work** (marked most important) — the exact state of the work right now
9. **Optional Work** — low-priority items mentioned but not started
10. **Key Artifacts** — important code blocks, configs, file contents to preserve verbatim

The LLM response format uses two XML tags:

- `<analysis>` — private reasoning not included in the output (chain-of-thought)
- `<summary>` — the actual compacted context, used as the new conversation seed

---

### 6. What Survives: The Preserved Segment

Compaction does not replace the entire conversation history. The most recent messages are preserved verbatim and appended after the summary.

**`dE1`** defines the preservation parameters:

```javascript
dE1 = { minTokens: 10000, minTextBlockMessages: 5, maxTokens: 40000 }
```

**`EmY(messages, lastIdx)`** works backward from the most recent message:

- Accumulates tokens counting backward
- Stops when either: `tokens >= 40,000` OR (`tokens >= 10,000` AND `textBlockMessages >= 5`)
- Everything within these bounds is kept verbatim

**`Op8()`** ensures integrity: if a `tool_use` block is preserved, its corresponding `tool_result` block is also preserved (and vice versa). Orphaned tool call pairs are not allowed.

The result is a new conversation history: LLM-generated summary first, then 10k–40k tokens of the most recent raw messages intact.

---

### 7. ORIGINAL FINDING: The currentDate Cache-Kill Problem

Every turn, **`eE1()`** injects a `<system-reminder>` block containing:

```
Today's date is YYYY-MM-DD
```

This is not surprising on its own. The problem is *where* it is injected.

The `currentDate` string is concatenated into the **same text block** as:

- `claudeMd` (CLAUDE.md contents — static, changes rarely)
- `gitStatus` (changes per commit)
- Other static context

Prompt caching is a strict prefix match. The cache key is the byte sequence of the entire prefix up to and including each cached block. When `currentDate` changes — which happens once per day at midnight — the serialized text of this combined block changes. Because the date sits before the static content in the string, every byte after it is now at a different position in the prefix.

Result: every day at midnight, all cached context after this injection point is invalidated. The next session that day pays `cache_write` rates (125% base cost) on the full token count of the conversation history.

A change of roughly 10 tokens causes a cache miss on potentially thousands of tokens of unchanged static content.

This is not a bug in user code. It is a structural property of how `eE1()` assembles the block. As far as we can find, this is the first public documentation of this mechanism.

---

### 8. ORIGINAL FINDING: The Compact Chain Reaction

Compaction interacts with the prompt cache in a specific way that can cause it to fire twice in rapid succession.

**During compaction:**

`sqq()` (the main compaction executor) sets `skipCacheWrite: true` on the compaction call. This is correct — the summary is ephemeral and should not create a new cache breakpoint.

**After compaction:**

`gl()` resets session state. This invalidates all existing prompt cache breakpoints accumulated over the session. The next turn starts with no cached context.

The first turn after compaction therefore behaves like the first turn of a new session:

- All tokens are sent as `cache_write` (125% rate)
- A new cache breakpoint is written

**If the summary is still large:**

`mz6()` evaluates the new `postCompactTokenCount`. If it still exceeds the autocompact threshold, `willRetriggerNextTurn = true`. On the next user turn, the threshold check fires again. A second compaction runs. `gl()` resets state again. Another full cache rebuild follows.

Each link in this chain:

1. Fires a full LLM compaction call (API cost)
2. Rebuilds the prompt cache from zero (125% write on all tokens)
3. Potentially triggers another link

The chain stops when the token count finally drops below threshold or when the circuit breaker at `aqq = 3` consecutive failures trips.

This is only possible when the context is genuinely very large — a 1M-token session where even the summarized form exceeds the 200k threshold. But in automated agent loops running extended tasks, this is not a hypothetical edge case.

---

### 9. Reactive Compact: The Emergency Path

Autocompact is proactive — it fires before the API call when token estimates cross the threshold.

Reactive compact fires *after* the API call fails.

If the Anthropic API returns a `prompt_too_long` error, **`Bi6.tryReactiveCompact()`** is called:

- Maximum one attempt per turn
- If already attempted and the attempt failed → returns `{reason: "prompt_too_long"}` and gives up
- The same `mf6()` flow runs, but triggered by API rejection rather than threshold estimate

The distinction matters for cost: reactive compact means you already paid for the failed API call at full `cache_write` rate before the compaction began.

---

### 10. Context Overflow Recovery: Automatic max_tokens Reduction

Separate from compaction, Claude Code handles the case where `input + max_tokens > context limit` at the API level.

**`$54()`** parses the error message:

```
input length and max_tokens exceed context limit: X + Y > Z
```

It extracts the numbers, computes a reduced `max_tokens`:

```
reducedMaxTokens = contextLimit - inputLength - 1000
```

The 1000-token margin is hardcoded. The floor is **`fN8 = 3,000 tokens`** — it will not reduce `max_tokens` below 3,000.

The request is retried automatically with the reduced value. No user intervention, no visible error.

---

### 11. Microcompact: A Stub

**`pg()`** is called in the main loop before autocompact runs:

```
microcompact → autocompact
```

The current implementation of `pg()` returns the original messages array unchanged. It is a placeholder. Future versions of Claude Code may implement partial context trimming (targeting specific message types or tool outputs) before falling back to full summarization-based autocompact.

---

### 12. Token Estimation: Three Methods, One Real

**Text estimation** — `j5(text, charsPerToken=4)`:

```javascript
Math.round(text.length / 4)
```

**JSON estimation** — same function, `charsPerToken=2`:

JSON is denser than prose, so the divisor is halved.

**Real token count** — from the API response `usage` field:

```javascript
fF6(usage): input + cache_creation + cache_read + output
```

`mz6()` and the threshold checks use the estimated counts before the API call. `postCompactTokenCount` and circuit breaker decisions use real counts from the API response after the call. There is a consistent gap between the two — estimated counts are used to decide whether to compact, and real counts are used to decide whether it worked.

---

### 13. Environment Variable Controls (Complete List)

| Variable | Effect |
|---|---|
| `DISABLE_COMPACT` | Disables all compaction (autocompact and reactive) |
| `DISABLE_AUTO_COMPACT` | Disables proactive autocompact only; reactive compact still runs |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | Sets threshold as percentage of context window: `floor(contextWindow * N / 100)` |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | Override for the context window size used in threshold calculation |
| `CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE` | Override for the hard blocking limit margin (`Mp8`) |
| `CLAUDE_CODE_DISABLE_1M_CONTEXT` | Forces 200k window even for 1M-eligible models |
| `ENABLE_CLAUDE_CODE_SM_COMPACT` | Enables an alternate session-memory-based compact path (disabled by default) |

---

## Impact

### Cost Model

A standard autocompact event has the following cost structure:

| Event | Cost |
|---|---|
| Compaction LLM call | Full API call at `cache_write` rate for all input tokens |
| Post-compaction first turn | Full `cache_write` on all tokens (no cache exists yet) |
| Each subsequent turn | Cache rebuilds normally — `cache_read` after the second turn |
| Chain reaction (N links) | N × (compaction call + cache rebuild) |

For a 160k-token session, a single compaction plus one post-compaction turn can cost the equivalent of 5–8 normal turns.

### Operational Impact for Agent Loops

Automated agent loops running extended tasks are the most affected:

1. **Threshold surprise** — the 13,000-token `Jp8` margin means compaction fires 13k tokens before the window is actually full. Planning for "it compacts at 200k" is wrong; it compacts at ~163k.

2. **Midnight invalidation** — any session running across midnight will pay a full cache rebuild on the next turn due to the `currentDate` injection mechanism. Agents running overnight tasks should account for this in cost projections.

3. **Chain reaction risk** — in 1M-context sessions with very large working sets, even the compacted summary may exceed the 200k default threshold. This triggers the chain reaction. Monitoring `willRetriggerNextTurn` (if surfaced) or watching for two consecutive compaction events is the only early warning.

4. **Reactive compact double cost** — if proactive autocompact is disabled but context grows large, the first `prompt_too_long` error triggers reactive compact. You paid once for the failed call, then again for the compaction.

---

## Mitigation

### Do Not Disable Compaction Blindly

`DISABLE_COMPACT` prevents both autocompact and reactive compact. If the context genuinely exceeds the window, the session will hang on API errors. Use `DISABLE_AUTO_COMPACT` only if you are managing context size externally.

### Use `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` to Compact Earlier

If you want to avoid the chain reaction, compact at a lower percentage of the window. Example: `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=70` triggers compaction at 70% of the context window, leaving headroom to absorb a larger-than-expected summary.

### Account for the Midnight Cache Invalidation

If running overnight agents, budget for one full cache rebuild per session per day. The cost is proportional to the total conversation token count at midnight. Sessions that run long enough to cross midnight multiple times pay this cost each time.

### Monitor `postCompactTokenCount` vs Threshold

After compaction, compare the post-compaction token count against the autocompact threshold. If it is still above threshold, a chain reaction is guaranteed on the next user turn. The only recovery at that point is to reduce `max_tokens` output limits, split the task, or compact again manually.

### The Preserved Segment Is Not Configurable

The 10k–40k preserved segment parameters are hardcoded in `dE1`. There is no way to expand or shrink the preservation window through configuration. Work that must survive compaction verbatim should be front-loaded into tool results or structured outputs that land near the end of the conversation, where they are within the preservation window.

---

## References

### Source Locations (v0.2.76 cli.js)

| Symbol | Anchor String |
|---|---|
| `uM(model, betas)` | `"CLAUDE_CODE_DISABLE_1M_CONTEXT"` |
| `OF(model)` | context window minus `RmY` and `oa(model)` calls |
| `mz6(tokens, model)` | `"isAboveAutoCompactThreshold"` |
| `oc6(model)` | `"CLAUDE_AUTOCOMPACT_PCT_OVERRIDE"` |
| `Xh()` | `"DISABLE_AUTO_COMPACT"` |
| `CmY()` | `"session_memory"` adjacent to `"compact"` guard |
| `mf6()` | `"preCompactTokenCount"` |
| `sqq()` | `"skipCacheWrite"` adjacent to compact logic |
| `eE1()` | `"Today's date is"` |
| `dE1` | `"minTextBlockMessages"` |
| `EmY()` | `"minTokens"` in preservation scan |
| `$54()` | `"input length and max_tokens exceed context limit"` |
| `Bi6.tryReactiveCompact()` | `"prompt_too_long"` |
| `pg()` | appears before autocompact in main loop, returns messages unchanged |

### Related Reports in This Repository

- [Reverse-Engineering the Claude Agent SDK: Root Cause and Fix for the 2–3% Credit Burn Per Message](../agent-sdk-cache-invalidation/README.md) — cache invalidation mechanics at the session level
- [Prompt Cache Architecture](../prompt-cache-architecture/) — how cache breakpoints are placed and maintained
- [system-reminder Injection](../system-reminder-injection/) — full injection taxonomy and token costs
