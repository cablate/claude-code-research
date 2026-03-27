# Prompt Cache Architecture — How Claude Code Controls What Gets Cached and for How Long

> **SDK Version:** @anthropic-ai/claude-agent-sdk v0.2.76 (cli.js build 2026-03-14)
> **Date:** 2026-03-27

Anthropic's API charges 10% of the base price for a cache read and 125% for a cache write. In a long conversation where the messages block alone runs 45,000 tokens, the difference between hitting and missing cache on every turn is roughly 12x in cost for that portion of the request. This is the single largest lever on per-message cost in any Claude Code deployment.

Claude Code has an elaborate internal system for controlling where cache breakpoints go, how long they live, and which users get the premium TTL tier. None of it is documented. None of it is configurable. This report maps the full mechanism by tracing the 12MB minified `cli.js` source.

---

## The Prefix Matching Rule — and Why It Makes Everything Else Matter

The Anthropic API caches by strict byte-for-byte prefix match. A request is a cache hit at position N only if every byte from position 0 through N-1 is identical to the cached version. No semantic matching, no hash-based deduplication, no partial credit.

The full payload sent to the API is structured as:

```
[system prompt blocks] → [tool definitions] → [messages array]
```

A single differing byte at position N invalidates the cache for everything from N onward. This has three immediate consequences for anyone building on the SDK:

- **Injection order matters.** If two system-reminder blocks swap positions between turns, the entire messages block misses cache.
- **Tool definition order matters.** Adding or reordering tools invalidates the tool section and everything after it.
- **Content stability matters.** A single changed timestamp in a dynamically injected memory block invalidates the full messages array.

This prefix rule is the reason Claude Code's cache architecture exists at all. Every mechanism described below is designed to keep bytes stable where they need to be stable, and to mark the right positions so the API knows where to look for cache boundaries.

---

## How Cache Breakpoints Are Placed

### The single factory

Every `cache_control` object in every API request originates from one factory function. It always produces a base object of `{type: "ephemeral"}` and conditionally adds two fields:

```js
// Simplified from the minified source
function createCacheControl(scope, ttl) {
  const ctrl = { type: "ephemeral" };
  if (ttl === "1h") ctrl.ttl = "1h";
  if (scope) ctrl.scope = scope;
  return ctrl;
}
```

The `type` is hardcoded. Claude Code never produces `{type: "persistent"}` or any other variant. This is the sole factory for all cache breakpoints in the entire codebase — there is no code path that creates a `cache_control` object by other means. In the minified source, searching for the string `"ephemeral"` used as a value (not in a comment) leads directly to this function (`Ml` in the current build).

### System prompt zones: the static/dynamic split

The system prompt is not sent as a single block. It is split into multiple content blocks at a boundary marker embedded in the assembly code: `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__`.

Everything assembled before that marker is static and shared — base system instructions, tool descriptions, organization-level context that does not change between sessions. Everything after is dynamic — the current working directory, loaded CLAUDE.md content, active skill configurations. The static blocks get `cache_control` with a `"global"` scope. The dynamic blocks get nothing.

```
  Block A: base instructions       → cache_control added (global scope)
  Block B: tool definitions        → cache_control added (global scope)
  Block C: org context             → cache_control added (global scope)
  ── __SYSTEM_PROMPT_DYNAMIC_BOUNDARY__ ──
  Block D: cwd, CLAUDE.md          → no cache_control
  Block E: active skills           → no cache_control
```

This split is why the system prompt portion achieves stable cache hits in both V1 and V2 SDK usage. The static zone is marked, the dynamic zone is not, and the static zone rarely changes. The boundary marker string is a reliable anchor for locating this logic in any version of the minified source.

### Message-level sliding window

After the system prompt and tools, the messages array is where most of the token volume lives. The breakpoint placement follows three rules:

**Only the last message gets a marker.** The assembler iterates the messages array and places `cache_control` on the last content block of the last message only. All earlier messages rely on stable-prefix matching — if the prefix is identical to the previous request, those positions hit cache without needing explicit markers.

**`skipCacheWrite` shifts the breakpoint.** When a request is assembled with `skipCacheWrite: true`, the breakpoint moves to the second-to-last message instead. This is used for ephemeral sub-calls (short verification queries, classifiers) where the response is not worth caching — burning a cache write slot that will never be read again wastes money.

**Thinking blocks are always skipped.** Content blocks with type `"thinking"` or `"redacted_thinking"` are excluded from marker placement by explicit type checks. If the last content block in a message is a thinking block, the marker falls on the previous non-thinking block in the same message. The thinking output itself is never the anchor for a cache entry.

```
messages: [
  { role: "user",      content: [...] },          ← no marker (stable prefix)
  { role: "assistant", content: [...] },          ← no marker (stable prefix)
  { role: "user",      content: [...] },          ← no marker (stable prefix)
  { role: "assistant", content: [
      { type: "thinking", ... },                  ← skipped
      { type: "text", ..., cache_control: ... }   ← marker placed here
  ]}
]
```

---

## How TTL Is Determined — the 1-Hour Gate

Standard Anthropic API prompt caches expire in 5 minutes. There is a separate 1-hour extended TTL tier, and `cli.js` has a gate function that decides who gets it. The decision follows two paths:

**Path A — Bedrock with an environment variable:**
```
platform === "bedrock"
AND process.env.CLAUDE_CODE_ENABLE_1H_CACHE === "1"
```

**Path B — First-party users, not in overage, on the allowlist:**
```
isFirstParty
AND NOT isInOverage
AND tengu_prompt_cache_1h_config.allowlist includes userId
```

The `tengu_prompt_cache_1h_config` allowlist is a server-side feature flag, not part of the SDK package. It supports wildcard matching — an entry ending in `*` matches any userId with that prefix. This means the 1-hour TTL grant is server-controlled. The only user-accessible lever is the Bedrock environment variable.

When the gate returns false, the factory produces `{type: "ephemeral"}` — the 5-minute TTL. When it returns true, the factory produces `{type: "ephemeral", ttl: "1h"}`.

The practical difference: in a V2 persistent session that goes idle for 6 minutes, a 5-minute cache entry has expired and the next turn pays for a full rebuild. A user with 1-hour TTL can tolerate that idle period without cost. For V1 usage where every turn spawns a new process, this distinction is irrelevant — the cache breaks for structural reasons long before any TTL expires.

The gate function can be located in the minified source by searching for the string `"tengu_prompt_cache_1h_config"` (`o3z` in the current build).

---

## The Per-Model Disable Switches

Before any cache control is added to a request, a separate gate checks whether caching is active for the model being used. It reads four environment variables:

| Environment variable | Effect |
|---|---|
| `DISABLE_PROMPT_CACHING` | Disables caching for all models |
| `DISABLE_PROMPT_CACHING_HAIKU` | Disables when model name contains `"haiku"` |
| `DISABLE_PROMPT_CACHING_SONNET` | Disables when model name contains `"sonnet"` |
| `DISABLE_PROMPT_CACHING_OPUS` | Disables when model name contains `"opus"` |

If any matching variable is set, the cache control factory is never called for that request and no `cache_control` fields appear in the payload. The matching uses substring lookup against the model name — `claude-sonnet-4-5` would be caught by `DISABLE_PROMPT_CACHING_SONNET`.

This is the only mechanism for fully suppressing cache markers at the request level. The gate function can be found by searching for `"DISABLE_PROMPT_CACHING"` in the source (`IGq` in the current build).

---

## The 34 querySource Values — and Who Gets Special Treatment

Every request carries an internal `querySource` field identifying what part of the system initiated it. Thirty-four distinct values have been observed, grouping into six categories:

| Category | Example values |
|---|---|
| Main interactive loop | `repl_main_thread` |
| Subagent calls | `agent:custom`, `sdk`, `hook_agent`, `verification_agent`, ~10 more |
| Tooling | `bash_extract_prefix`, `web_fetch_apply`, and similar |
| Memory operations | `extract_memories`, `session_memory` |
| Compaction | `compact` |
| Side calls and classifiers | various |

The main interactive loop source (`repl_main_thread`) receives special treatment: it is the only value that enables the `cache editing beta` HTTP header, which activates experimental cached-context surgery — the ability to modify cached content in place rather than invalidating it. This feature is additionally gated to first-party only. Requests originating from the SDK path (`querySource: "sdk"`) do not receive it.

The `sdk` value is assigned to V2 persistent session requests. Whether those requests get 1-hour TTL depends entirely on the server-side allowlist in the TTL gate, not on the querySource itself.

---

## What This Means for SDK Users

### V1 `query()` — structural cache misses on every resumed turn

V1 creates a new process per call. The system prompt static zone caches reliably because it rarely changes. The messages array does not — each spawn reassembles dynamic injections from scratch and cannot guarantee byte-level consistency. The result is that the message block (~45k tokens) is re-written at 125% rate on every V1 call with `resume`.

The 1-hour TTL is irrelevant here. The cache breaks for structural reasons, not because it expired.

### V2 persistent sessions — accumulating cache hits

V2 keeps the process alive. The assembler runs in the same process across turns, making the byte-level output deterministic. Messages accumulate in the cache across turns. The 1-hour TTL becomes relevant: idle sessions with 5-minute TTL lose their cache after 5 minutes of inactivity, while 1-hour users can tolerate longer gaps.

### Tool definition ordering

Tool definitions sit between the system prompt and the messages array in the request prefix. Any change to tool definitions — adding, removing, or reordering — invalidates the cache for the entire messages portion. Deployments that dynamically load MCP servers or tool sets can produce cache misses that look unrelated to message content.

The fix is straightforward: load the full tool set at session start and keep it stable across turns. Do not add or remove tools mid-session.

### Monitoring

Add token counting to every SDK call. Log `cacheReadTokens`, `cacheCreationTokens`, and `inputTokens` from the usage field of each response. Compute:

```
efficiency = cacheRead / (input + cacheRead + cacheCreation)
```

Below 50%: structural cache break — something is changing between turns that should not be. Above 70%: healthy. Degradation between turns (not just on turn #1) usually points to dynamic injection instability.

---

## Methodology

All findings are from static analysis of `cli.js` extracted from the `@anthropic-ai/claude-agent-sdk` npm package at version 0.2.76. The file is a single 12MB minified JavaScript bundle with all variable names obfuscated to 2-4 character identifiers. Key functions were located by searching for string constants — these are not obfuscated and remain stable across upgrades.

The primary anchors used and what they locate:

| String constant | What it locates |
|---|---|
| `"ephemeral"` | The cache control factory |
| `"DISABLE_PROMPT_CACHING"` | The per-model enable/disable gate |
| `"tengu_prompt_cache_1h_config"` | The 1-hour TTL feature flag gate |
| `"__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__"` | The system prompt zone splitter |
| `"skipCacheWrite"` | The message-level breakpoint placer |
| `"repl_main_thread"` | The querySource routing and cache editing beta gate |

Code snippets in this report preserve the logic structure but use descriptive names instead of the obfuscated originals. They are not verbatim copies. All source references point to: `node_modules/@anthropic-ai/claude-agent-sdk/dist/cli.js`, approximately lines 6340-6500 for the cache-related functions.

---

## References

- Report #1: [Reverse-Engineering the Claude Agent SDK: Root Cause and Fix for the 2-3% Credit Burn Per Message](../agent-sdk-cache-invalidation/README.md)
- Anthropic prompt caching documentation: https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
- GitHub Issue #9769 — request for per-system-reminder toggles: https://github.com/anthropics/claude-code/issues/9769
- GitHub Issue #16021 — token waste from file-modification injections: https://github.com/anthropics/claude-code/issues/16021
