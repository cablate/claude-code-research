# Tool Serialization and Cache Stability — Why Your MCP Tools Might Be Silently Busting the Prompt Cache

The `tools` array is part of every API request's cache prefix. If the serialized tools change between turns — even by a single byte — the Anthropic API treats the entire prefix as new and rewrites the cache at 125% cost. The system prompt, the messages, everything after the divergence point: all invalidated.

Claude Code never sorts its tools. Stability depends entirely on insertion consistency. And the deferred tool loading mechanism silently changes the tools array mid-conversation, guaranteeing cache misses in ways that are invisible to the developer.

This report traces the full pipeline from tool list construction to API serialization, identifies every point where instability can enter, and explains what SDK users can do about it.

Analysis based on reverse-engineering `cli.js` build `2026-03-14`, bundled in `@anthropic-ai/claude-agent-sdk` v0.2.76.

---

## How the Tool List Gets Built

Every API request that Claude Code sends assembles its `tools` array through four stages. Understanding these stages is necessary to understand where cache instability enters.

### Stage 1: The Built-in Registry

The first stage is a hardcoded literal array containing every built-in tool — Read, Write, Edit, Bash, Grep, Glob, WebFetch, TodoWrite, and others. This array is determined at bundle compile time. The tool list builder function (`ng` in source) returns this same array on every call, in the same order. This is the only stage that is deterministic by construction.

### Stage 2: Allowed/Denied Filtering

The second stage takes the built-in array and applies JavaScript's `.filter()` using the `allowedTools` and `denyRules` configuration (`FX` in source). Because `.filter()` preserves insertion order, the relative ordering of surviving entries is identical to Stage 1.

One subtlety: if you pass `allowedTools: ["Bash", "Read", "Write"]`, the resulting list has those tools in their Stage 1 order (Bash before Read before Write based on compile-time position), not in the order you specified. The `allowedTools` array is a membership test, not an ordering instruction.

### Stage 3: Merging Built-ins with MCP Tools

The merge step (`u66` in source) concatenates built-ins and MCP tools:

```javascript
// Built-in tools always come first, MCP tools appended after
[...builtins, ...mcpTools]
```

It then deduplicates by name, keeping the first occurrence. Built-ins always precede MCP tools. If an MCP tool shares a name with a built-in, the built-in wins.

### Stage 4: Serialization to Wire Format

The serializer (`Sh1` in source) converts each tool into the shape the API expects:

```javascript
{
  name: tool.name,
  description: await tool.prompt(),  // async — this matters
  input_schema: tool.jsonSchema ?? deriveFromZod(tool.inputSchema)
}
```

Two details matter for cache stability:

**Descriptions are resolved asynchronously.** Built-in tools return static strings — always the same content. MCP tools may call the server for a dynamic description. Any non-determinism in the server response propagates directly into the serialized payload.

**Schema sources differ by tool type.** MCP tools use the JSON schema provided by the MCP server at registration. Built-in tools derive their schemas from Zod definitions via a memoized converter — stable within a process lifetime.

The `cache_control` marker is not applied at the individual tool level. Tools are not individually cached; only the assembled system prompt gets cache markers.

---

## The Critical Finding: No Sorting Anywhere

I searched the entire `cli.js` source for `.sort()` calls near tool-related code. Every `.sort()` found belongs to unrelated functionality — worktree path sorting, insights data, help menu display, compact metadata output.

The tool pipeline has no `.sort()` at any stage. Tool order is strictly insertion-order from start to finish.

This means the cache prefix depends on something no one is explicitly controlling. For built-in tools it works out fine — the compile-time array is the same every run. For MCP tools, it means the order depends on how and when tools are registered, and that is where things get fragile.

---

## MCP Tool Ordering: Usually Stable, Occasionally Not

MCP tools enter the pipeline through a loader (`Fr6` in source) that initializes all registered MCP servers concurrently. The responses arrive in non-deterministic order — but the tool list stored in the session is built from the client registration order at startup, not from response arrival order. So for a config file read from disk, registration order is consistent across restarts.

The fragility appears in two scenarios:

**Programmatic config construction.** If your code builds the `mcpServers` object by iterating over a data structure whose key order isn't guaranteed, the registration order varies between runs. Two requests with nominally identical config but different key insertion order produce different MCP tool arrays.

**MCP server reconnection.** If a server goes down and comes back, the tool list is re-registered, potentially at a different position in the merged array.

Both scenarios produce a structurally different `tools` array. The Anthropic API treats the entire `tools` block as part of the cache prefix. Different array = different prefix = full cache miss.

---

## Deferred Tool Loading: The Hidden Cache Breaker

This is the most impactful finding in the report.

Claude Code defers all MCP tools by default. The deferred loading controller (`GX` in source) applies a simple rule: every tool where `isMcp === true` is deferred. Here's what that means in practice:

When a session starts, MCP tools are not included in the `tools` array at all. Instead, only their names are listed in a text block inside the system prompt: `<available-deferred-tools>mcp__server__toolname, ...</available-deferred-tools>`. The model sees the names and knows it can request them, but the full schemas are absent from the request.

When Claude decides it needs a deferred tool, a search step runs that loads the full schema. At that point, the `tools` array gains a new entry with the complete `{name, description, input_schema}` object.

**The first invocation of any MCP tool in a conversation is a guaranteed cache miss.** The `tools` array before the call is structurally different from the `tools` array after. The cache prefix has changed. There is no way around this.

### How It Works Internally: The Discovery Scanner

We traced this to the exact mechanism. On every turn, the main query function (`mGq` in source) rebuilds the `tools` array from scratch. The decision of which deferred tools to include is driven by a discovery scanner (`zF` in source) that scans the full message history for previously loaded tools:

```javascript
// Simplified from mGq — the main query generator
// Runs on EVERY turn, not just the first

let discoveredTools = scanMessageHistory(messages);  // zF(A)
let J;
if (dynamicToolLoading) {
  J = allTools.filter(tool => {
    if (!isDeferred(tool)) return true;      // built-ins: always include
    if (isToolSearch(tool)) return true;      // tool-search itself: always
    return discoveredTools.has(tool.name);    // MCP tools: ONLY if discovered
  });
}
```

This means the `tools` array is not a fixed list — it grows over the course of a conversation. Each time Claude uses tool-search to load a deferred MCP tool, that tool's name enters the message history. On the next turn, the scanner finds it, and the filter includes it. The serialized `tools` array gains one more entry.

**The cost compounds with conversation length.** Each cache miss forces a full cache rebuild covering the system prompt, all tool definitions, and the entire message history up to that point. A tool loaded on turn 10 causes a miss that rewrites far more tokens than one loaded on turn 2.

Here's what a multi-turn session looks like:

| Turn | What Happens | tools Array | Cache | Rebuild Size |
|------|-------------|-------------|-------|-------------|
| 1 | No MCP tools used | Built-ins only | Hit | — |
| 2 | Claude uses tool-search to load `mcp__files__read` | Same as turn 1 (tool-search result is a message, not a tool entry) | Hit | — |
| 3 | Claude calls `mcp__files__read` | Discovery scanner finds it in history → +1 full schema in array | **Miss** | ~20k tokens |
| 4 | `mcp__files__read` again | Same as turn 3 | Hit | — |
| 5 | Claude loads `mcp__search__query` via tool-search | Same as turn 3 | Hit | — |
| 6 | Claude calls `mcp__search__query` | Scanner finds it → +1 more schema | **Miss** | ~40k tokens |
| 7 | Claude loads `mcp__browser__click` via tool-search | Same as turn 6 | Hit | — |
| 8 | Claude calls `mcp__browser__click` | Scanner finds it → +1 more schema | **Miss** | ~60k tokens |
| 9+ | Same tools repeated | Stable | Hit | — |

Each unique MCP tool invoked for the first time costs one guaranteed cache miss. The misses do not happen when the tool is loaded via tool-search — they happen on the **next turn**, when the discovery scanner rebuilds the tools array and includes the newly discovered tool.

**The cost compounds.** The cache miss on turn 8 rebuilds ~60k tokens (all prior messages), while the miss on turn 3 rebuilds only ~20k. Using three MCP tools across eight turns costs roughly three times more than a single close-and-resume would.

There is also a feature flag (`tengu_defer_all` in source) that extends deferral to all tools including built-ins. When active, the initial tools array is nearly empty and every tool use triggers a deferred load. This appears to be used in specific internal scenarios.

---

## Tool Descriptions: Safe for Built-ins, Risky for MCP

The serializer resolves each tool's description by calling an async method. For built-in tools, this returns a static string — hardcoded in the bundle, identical every time. Built-in descriptions are cache-safe.

For MCP tools, the description comes from the MCP server. If the server generates descriptions dynamically — including a timestamp, a record count, an environment variable, anything that varies between calls — the serialized description changes between turns. Even a single character difference in any tool description invalidates the entire `tools` block in the cache prefix.

This is especially insidious because the tool might function identically. The search results are the same, the behavior is the same, but the description says "Search across 1,847 documents" instead of "Search across 1,846 documents" and the entire cache is gone.

---

## disallowedTools and allowedTools: Order-Preserving Filters

The disallowed tools filter (`_c` in source) materializes the `disallowedTools` setting into a Set, then applies `.filter()` to remove matching entries. Order is preserved. The filtering is applied at the agent or subagent level, sourced from the agent definition frontmatter.

From the SDK side, `disallowedTools` are passed as `--disallowedTools ToolA,ToolB` CLI arguments. No sorting is applied at this stage either.

Neither `allowedTools` nor `disallowedTools` introduces ordering instability on their own — they preserve whatever order existed before filtering. The risk is only if the input to these filters is itself unstable.

---

## What SDK Users Can Do

### Sort `allowedTools` before passing to the SDK

```javascript
// Unstable — depends on however getEnabledTools() builds the list
const options = { allowedTools: getEnabledTools() };

// Stable — alphabetical order, always the same
const options = { allowedTools: getEnabledTools().sort() };
```

One line. Eliminates one source of ordering instability.

### Sort `mcpServers` keys

```javascript
// Ensures consistent key order regardless of how the config was built
const mcpServers = Object.fromEntries(
  Object.entries(buildMcpConfig()).sort(([a], [b]) => a.localeCompare(b))
);
```

### Make MCP tool descriptions completely static

If you control the MCP server, hardcode the descriptions. No timestamps, no database lookups, no runtime state.

```python
# Dynamic description — cache breaker
@mcp.tool()
def search(query: str) -> str:
    """Search across {len(self.documents)} documents."""
    ...

# Static description — cache safe
@mcp.tool()
def search(query: str) -> str:
    """Search across the document index."""
    ...
```

### Pre-warm MCP tools at session start

If you know which MCP tools a session will use, trigger them early. A throwaway prompt at session initialization forces deferred loading before cache-sensitive turns begin:

```javascript
// Force all MCP tools to load upfront
await session.query("list the available mcp tools", { maxTurns: 1 });
// Subsequent turns benefit from a stable tools array
```

This shifts the guaranteed cache misses to the beginning of the session rather than scattering them across productive turns.

### Consider disabling deferred loading entirely

If your session uses a known, fixed set of MCP tools, disabling deferral means all schemas are serialized upfront. Larger initial request, but stable `tools` array from turn 1. No mid-conversation surprises.

There is no public API option for this — it requires patching or a feature flag.

### Monitor the tools array across turns

Log the serialized tools array for the first few turns of a session. If new entries appear in turns 2, 3, 4, those are deferred loads causing cache misses. Quantify the pattern before deciding on mitigation.

---

## The Real-World Cost

For a system with 3 MCP servers and 4-6 tools each:

- Session starts with 15-18 deferred tools
- Each unique MCP tool invoked for the first time triggers a guaranteed cache miss **on the following turn**
- The tools array does not stabilize until all needed tools have been used at least once
- The cost of each miss scales with conversation length — later misses are more expensive than earlier ones
- If your average session is 10 turns and you use 5 distinct MCP tools across those turns, each miss rebuilds an increasingly large context. At Opus 4 rates ($37.50/M for cache writes), 5 sequential misses across a growing conversation can cost more than 5 separate close-and-resume cycles

For a system with built-in tools only:

- The tool list is compile-time stable
- No deferred loading occurs
- Cache stability is essentially guaranteed from turn 1

The gap between these two scenarios is the cost of MCP tool integration that no one talks about.

---

## References

- [Reverse-Engineering the Claude Agent SDK: Root Cause and Fix for the 2-3% Credit Burn Per Message](../agent-sdk-cache-invalidation/README.md) — covers how the prompt cache works and why cache misses are expensive
- SDK version: `@anthropic-ai/claude-agent-sdk` v0.2.76, `cli.js` build `2026-03-14`
- Anthropic API prompt caching: cache reads at 10% of base input cost, cache writes at 125%
- [Model Context Protocol specification](https://modelcontextprotocol.io)
