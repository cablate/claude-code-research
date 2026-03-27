# Prompt Cache Architecture — How Claude Code Controls What Gets Cached and for How Long

> **SDK Version:** @anthropic-ai/claude-agent-sdk v0.2.76 (cli.js build 2026-03-14)
> **Date:** 2026-03-27
> **Status:** Draft

Every time Claude Code sends a request to the Anthropic API, it makes decisions about which parts of the payload to mark for caching, what TTL to assign, and which content is too dynamic to cache at all. These decisions are not user-configurable. They happen inside `cli.js` before the request leaves the machine.

This report maps the complete prompt cache architecture by tracing the call chain in the 12MB minified source: from the single factory function that produces every `cache_control` object, through the per-model enable/disable gates, to the server-side feature flag that controls whether 1-hour extended TTL is granted. It also documents how the system prompt is split into static and dynamic zones, how the message-level sliding window works, and why the underlying mechanism — byte-for-byte prefix matching — means even minor ordering changes can destroy an otherwise warm cache.

---

## Summary

`cli.js` manages prompt caching through a small set of coordinated mechanisms: a single `cache_control` factory (`Ml()`), a model-specific enable/disable gate (`IGq()`), a server-side 1h TTL gate (`o3z()`), a system prompt boundary that separates static from dynamic content (`_9z()`), and a message-level breakpoint placer that marks only the last content block (`z9z()`). Together these determine the exact bytes that go to the API and which positions are eligible for cache reads.

Understanding this architecture matters for anyone running the SDK programmatically: the same code paths that make CLI interactive mode so efficient also explain why certain SDK usage patterns produce persistent cache misses. The function names and string anchors in this report are stable across obfuscation — they can be used to locate the same logic in future versions.

---

## Methodology

All findings are from static analysis of `cli.js` extracted from the `@anthropic-ai/claude-agent-sdk` npm package at version 0.2.76. The file is a single 12MB minified JavaScript bundle with all variable names obfuscated to 2–4 character identifiers. Key functions were located by searching for string constants embedded in the source — these constants are not obfuscated and remain stable across version upgrades.

The primary search anchors used:

| String constant | Function located |
|---|---|
| `"DISABLE_PROMPT_CACHING"` | Cache enable/disable gate (`IGq()`) |
| `"tengu_prompt_cache_1h_config"` | 1h TTL feature flag gate (`o3z()`) |
| `"__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__"` | System prompt zone splitter (`_9z()` → `Jn8()`) |
| `"ephemeral"` | Cache control factory (`Ml()`) |
| `"skipCacheWrite"` | Message-level breakpoint placer (`z9z()`) |
| `"repl_main_thread"` | querySource routing and cache editing beta gate |

Simplified code snippets in this report preserve the logic structure but replace obfuscated names with descriptive ones. They are not verbatim copies of the source.

---

## Findings

### 1. The Cache Control Factory: `Ml()`

Every `cache_control` object in every API request produced by `cli.js` originates from a single factory function located at approximately line 6370. The function always produces a base object of `{type: "ephemeral"}` and conditionally extends it with two additional fields:

```js
// Simplified — actual code uses obfuscated names
function Ml(scope, ttl) {
  const ctrl = { type: "ephemeral" };
  if (ttl === "1h") ctrl.ttl = "1h";
  if (scope) ctrl.scope = scope;
  return ctrl;
}
```

The `type: "ephemeral"` value is hardcoded. Claude Code never produces `{type: "persistent"}` or any other variant. The `ttl` and `scope` fields appear only under conditions controlled by separate gate functions described below.

This is the sole factory for all cache breakpoints in the entire codebase. There is no code path that creates a `cache_control` object by any other means.

### 2. Cache Enable/Disable Gate: `IGq()`

Before any cache control is added to a request, `IGq()` checks whether caching is active for the model being used. It reads from environment variables in this order:

1. `DISABLE_PROMPT_CACHING` — disables caching for all models
2. `DISABLE_PROMPT_CACHING_HAIKU` — disables caching only when the model name contains `"haiku"`
3. `DISABLE_PROMPT_CACHING_SONNET` — disables caching only when the model name contains `"sonnet"`
4. `DISABLE_PROMPT_CACHING_OPUS` — disables caching only when the model name contains `"opus"`

If caching is disabled, `Ml()` is never called for that request and no `cache_control` fields appear in the payload. This is the only mechanism for fully suppressing cache markers at the request level.

The per-model env vars use substring matching against the model name string. A model named `claude-sonnet-4-5` would be matched by `DISABLE_PROMPT_CACHING_SONNET`.

### 3. The 1h TTL Gate: `o3z()`

Standard Anthropic API prompt caches expire in 5 minutes. The 1h extended TTL is a separate tier that `cli.js` gates behind `o3z()`. This function evaluates two possible paths to a 1h TTL:

**Path A — Bedrock with env var:**
```
platform === "bedrock"
&& process.env.CLAUDE_CODE_ENABLE_1H_CACHE === "1"
```

**Path B — First-party + not in overage + feature flag:**
```
isFirstParty
&& !isInOverage
&& tengu_prompt_cache_1h_config.allowlist includes current userId
```

The `tengu_prompt_cache_1h_config` allowlist is read from a feature flag (not from the SDK package). It supports wildcard matching: an allowlist entry ending in `*` matches any userId with that prefix. This means the 1h TTL grant is server-controlled and user-configurable only through the Bedrock env var path.

When `o3z()` returns false, `Ml()` produces `{type: "ephemeral"}` — the 5-minute TTL. When it returns true, `Ml()` produces `{type: "ephemeral", ttl: "1h"}`.

The `scope: "global"` field is added separately, based on the `cacheScope` property of the content block being marked (see System Prompt Zones below).

### 4. querySource: 34 Observed Values and Their Effect on Caching

Every request has an internal `querySource` field that identifies what part of the system initiated it. This field influences caching behavior in at least one documented way and gates other features. The 34 observed values group into six categories:

| Category | querySource values |
|---|---|
| Main interactive loop | `repl_main_thread` |
| Subagent calls | `agent:custom`, `sdk`, `hook_agent`, `verification_agent`, and ~10 variants |
| Tooling | `bash_extract_prefix`, `web_fetch_apply`, and similar |
| Memory operations | `extract_memories`, `session_memory` |
| Compaction | `compact` |
| Side calls and classifiers | various |

`repl_main_thread` receives special treatment: it is the only `querySource` that enables the `cache editing beta` header, which activates experimental cached context surgery (the ability to modify cached content in place rather than invalidating it). This feature is gated additionally to first-party only — SDK-originated requests with `querySource: "sdk"` do not receive it.

The `sdk` value is assigned to V2 persistent session requests. Whether `sdk`-originated requests get 1h TTL depends entirely on the server-side allowlist in `o3z()`.

### 5. System Prompt Cache Zones: `_9z()` → `Jn8()`

The system prompt is not sent as a single block. It is split into multiple content blocks, each carrying a `cacheScope` property with one of three values: `null`, `"org"`, or `"global"`.

The split is controlled by a boundary marker embedded in the system prompt assembly: `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__`. Content assembled before this marker is static and shared — it includes the base system instructions, tool descriptions, and organization-level context that does not change between sessions. Content assembled after this marker is dynamic — it includes per-session context such as the current working directory, loaded CLAUDE.md content, and active skill configurations.

The `Jn8()` function maps `cacheScope` values to cache behavior:

| cacheScope | cache_control added | scope field |
|---|---|---|
| `null` | none | — |
| `"org"` | yes, via `Ml()` | `"global"` (org-scoped) |
| `"global"` | yes, via `Ml()` | `"global"` |

Blocks before the dynamic boundary receive non-null scope and get `cache_control`. Blocks after the boundary are dynamic and get `cacheScope: null` — no cache marker. The boundary ensures static content can be cached across sessions while dynamic content is reassembled fresh each time.

This is the mechanism behind why the system prompt portion of a request achieves stable cache hits across both V1 and V2 SDK usage: the static zone is marked, the dynamic zone is not, and the static zone rarely changes.

```
[system prompt assembly]

  Block A: base instructions       cacheScope: "global"  → cache_control added
  Block B: tool definitions        cacheScope: "global"  → cache_control added
  Block C: org context             cacheScope: "org"     → cache_control added
  ── __SYSTEM_PROMPT_DYNAMIC_BOUNDARY__ ──
  Block D: cwd, CLAUDE.md          cacheScope: null      → no cache_control
  Block E: active skills           cacheScope: null      → no cache_control
```

![System prompt split into static and dynamic zones at the dynamic boundary marker](./images/system-prompt-zones.webp)

### 6. Message-Level Cache Breakpoints: `z9z()` → `s3z()` / `t3z()`

After the system prompt, the messages array is where most of the token volume lives. `z9z()` places cache breakpoints in this array according to three rules:

**Rule 1 — Only the last message gets a marker.** The function iterates the assembled messages array and places `cache_control` on the last content block of the last message only. All prior messages rely on stable-prefix matching — if the prefix is identical to the previous request, those positions are cache hits without needing explicit markers.

**Rule 2 — `skipCacheWrite=true` shifts the breakpoint.** When a request is assembled with `skipCacheWrite: true`, the breakpoint moves to the second-to-last message instead of the last. This is used when the response is expected to be ephemeral and not worth caching — for example, in short verification sub-calls where caching the request would consume a write slot that won't be read again.

**Rule 3 — `thinking` blocks never get markers.** Content blocks with type `"thinking"` or `"redacted_thinking"` are excluded from cache marker placement by explicit type checks inside `s3z()` and `t3z()`. If the last content block in a message is a thinking block, the marker moves to the previous non-thinking block in the same message.

```
messages: [
  { role: "user",      content: [...] },        ← no marker (stable prefix)
  { role: "assistant", content: [...] },        ← no marker (stable prefix)
  { role: "user",      content: [...] },        ← no marker (stable prefix)
  { role: "assistant", content: [            ← last message
      { type: "thinking", ... },              ← skip: no marker
      { type: "text", ..., cache_control }    ← last non-thinking block: marker here
  ]}
]
```

### 7. The Prefix Matching Principle

The Anthropic API caches by byte-for-byte prefix match. A request is treated as a cache hit at position N only if every byte from position 0 through N-1 is identical to the cached version. There is no semantic matching, no hash-based deduplication, and no partial credit.

The full request sent to the API has this structure:

```
[system prompt blocks] [tool definitions] [messages array]
```

Any byte difference at position N invalidates the cache for everything from N onward. This means:

- **Injection order matters.** If two system-reminder blocks swap positions between turns, the entire messages block misses.
- **Tool definition order matters.** Adding or reordering tools invalidates the tool-section cache and everything after it.
- **Content stability matters.** A single mtime timestamp difference in a dynamically injected memory block invalidates the entire messages array.

This is why eliminating injection sources one by one (as documented in Report #1) had limited effect: even with most injections removed, any remaining dynamic content that changes between turns breaks the prefix, and the 45k-token messages block has to be re-written at 125%.

The only reliable solution is to keep the assembler deterministic — same content, same order, same bytes — across turns. V2 persistent sessions achieve this because the same process assembles the request each time. V1 per-turn spawning reassembles from scratch and cannot guarantee byte-level stability.

### 8. Cache Cost Structure

The Anthropic API bills cache operations at three rates:

| Operation | Rate |
|---|---|
| Cache read (hit) | 10% of base input price |
| Cache write (miss, new entry) | 125% of base input price |
| Normal input (no cache marker) | 100% of base input price |

A cache miss costs 25% more than a normal request. A cache hit costs 90% less. In a conversation where the messages block (~45k tokens) misses every turn, the effective cost is roughly 1.25x the base rate for that portion of the input. When the same block hits, it drops to 0.10x.

The efficiency formula is:

```
cache_efficiency = cacheReadTokens / (inputTokens + cacheReadTokens + cacheCreationTokens)
```

This metric should be monitored in any production SDK deployment. Consistent values below 50% indicate either a misconfigured deployment or a structural cache break similar to the V1 SDK issue documented in Report #1.

---

## Evidence

### `Ml()` factory — anchor: `"ephemeral"`

Searching `"ephemeral"` in `cli.js` yields one location where the string is used as a value (not in a comment or condition). That location is the `Ml()` function body. The function is called from three places in the codebase: `Jn8()` (system prompt zone handler), `s3z()` (message last-block marker), and `t3z()` (message second-to-last marker with `skipCacheWrite`).

### `IGq()` gate — anchor: `"DISABLE_PROMPT_CACHING"`

The string `"DISABLE_PROMPT_CACHING"` appears as a `process.env` key access four times in close proximity, corresponding to the four env var checks (global, haiku, sonnet, opus). All four reads are inside the same gate function.

### `o3z()` 1h gate — anchor: `"tengu_prompt_cache_1h_config"`

The feature flag name `"tengu_prompt_cache_1h_config"` appears once. The surrounding code reads the flag's allowlist field and performs a loop that checks whether a userId string satisfies a match condition. The wildcard logic uses `endsWith("*")` to detect prefix patterns.

### `_9z()` / `Jn8()` boundary — anchor: `"__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__"`

The boundary string appears once as a literal in the system prompt assembly function. The code splits the assembled blocks at this marker and assigns `cacheScope` values based on which side of the split each block falls on.

### `z9z()` message breakpoints — anchor: `"skipCacheWrite"`

The string `"skipCacheWrite"` appears as a property access on the request options object. The containing function iterates the messages array in reverse and places `cache_control` via `Ml()` on the located block. The thinking-type exclusion checks are visible as string comparisons against `"thinking"` and `"redacted_thinking"` immediately before the `Ml()` call.

### `repl_main_thread` cache editing beta — anchor: `"repl_main_thread"`

The string `"repl_main_thread"` is compared against the `querySource` field in a conditional block. The true branch sets a header corresponding to the `cache editing beta` feature. A second condition in the same block checks for first-party status before the header is applied.

All source references are in: `node_modules/@anthropic-ai/claude-agent-sdk/dist/cli.js`, approximately lines 6340–6500 for the cache-related functions.

---

## Impact

### For SDK users (V1 `query()`)

V1 creates a new process per call. The system prompt static zone is cached reliably because it rarely changes. The messages array is not — each spawn reassembles dynamic injections and cannot guarantee byte-level consistency across calls. The practical result is that message-block cache writes happen on every V1 call with `resume`, at 125% rate.

The 1h TTL gate (`o3z()`) is server-controlled. SDK users without an approved `userId` in the `tengu_prompt_cache_1h_config` allowlist receive 5-minute TTLs. For V1 usage where each turn spawns a new process, this distinction is irrelevant — the 5-minute window is never the bottleneck.

### For V2 persistent session users

V2 keeps the process alive. The assembler runs in the same process across turns, which makes the byte-level output deterministic. Messages accumulate in the cache across turns. The 1h TTL becomes relevant here: if the session is idle for more than 5 minutes, a 5-minute TTL cache entry expires and must be rebuilt on the next turn. Users with 1h TTL (via first-party allowlist or Bedrock env var) can tolerate longer idle periods without paying the rebuild cost.

### For the `thinking` block exclusion

The explicit exclusion of `thinking` and `redacted_thinking` blocks from cache marker placement has a concrete consequence: if an extended thinking response is the last content block in an assistant turn, no cache marker is placed on it. The marker instead falls on the preceding text block. This means the thinking output itself is never the anchor for a cache entry — only text content is.

### For tool definition ordering

Tool definitions appear between the system prompt blocks and the messages array in the request structure. Any change to tool definitions (adding tools, removing tools, reordering tools) invalidates the cache for the entire messages portion. In deployments that dynamically load MCP servers or tool sets based on context, this can produce cache misses that look unrelated to message content.

---

## Mitigation / Workaround

### Stable cache in SDK deployments

The root requirement is byte-level consistency in the messages block across turns. The only approach that reliably delivers this is V2 persistent sessions (same process, same assembler state). Patching the V2 API for full option support is documented in Report #1.

For deployments that cannot use V2, the next best approach is to minimize dynamic injection sources. The JSONL sanitizer technique from Report #1 eliminates file-modification diff injections. Setting `CLAUDE_CODE_REMOTE=1` suppresses git status injection. Neither fully solves the problem, but both reduce the magnitude of the cache waste.

### Monitoring cache efficiency

Add token counting to every SDK call. Log `cacheReadTokens`, `cacheCreationTokens`, and `inputTokens` from the usage field of each response. Compute:

```
efficiency = cacheRead / (input + cacheRead + cacheCreation)
```

Below 50%: structural cache break. Above 70%: healthy. Degradation between turns (not just on turn #1) usually indicates dynamic injection instability.

### 1h TTL access

There is no public mechanism to request inclusion in the `tengu_prompt_cache_1h_config` allowlist. The Bedrock path (`CLAUDE_CODE_ENABLE_1H_CACHE=1`) is the only user-accessible gate. Bedrock deployments can enable it unconditionally via that env var.

### Tool definition stability

If tools are loaded dynamically, load the full set at session start and keep it stable across turns. Avoid adding or removing tools mid-session. Tool definitions sit before the messages block in the request prefix — a change there invalidates everything after it.

---

## References

- Report #1: [Reverse-Engineering the Claude Agent SDK: Root Cause and Fix for the 2–3% Credit Burn Per Message](../agent-sdk-cache-invalidation/README.md)
- Anthropic prompt caching documentation: https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
- GitHub Issue #9769 — request for per-system-reminder toggles: https://github.com/anthropics/claude-code/issues/9769
- GitHub Issue #16021 — token waste from file-modification injections: https://github.com/anthropics/claude-code/issues/16021
