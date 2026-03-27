# Tool Serialization and Cache Stability — Why Your MCP Tools Might Be Silently Busting the Prompt Cache

You add an MCP server to Claude Code. Suddenly your cache efficiency drops. You add a second MCP server and it gets worse. Neither the SDK docs nor the Claude Code changelog mention anything about tool ordering and prompt caching. There is no warning.

This report documents how Claude Code's tool pipeline works internally, why MCP tool loading is non-deterministic by design, and which of those behaviors silently invalidate the prompt cache mid-conversation.

The analysis is based on reverse-engineering `cli.js` build `2026-03-14`, bundled in `@anthropic-ai/claude-agent-sdk` v0.2.76.

---

## Methodology

`cli.js` is a 12MB minified JavaScript file. Variable names are obfuscated on each build. The approach is to anchor on string constants that do not change between builds.

For this investigation the relevant anchors were:

- `"isMcp"` — locates MCP tool filtering and deferred loading logic
- `"available-deferred-tools"` — locates the system prompt block for deferred tools
- `"input_schema"` — locates `Sh1()`, the tool serialization function
- `"tengu_defer_all"` — locates the feature flag controlling full tool deferral
- `"disallowedTools"` — locates `_c()`, the tool filtering function
- `"K0"` (deduplication key) — locates `u66()`, the merge step

From each anchor, the surrounding function and its call graph were traced. The 4-stage pipeline was reconstructed by following the data flow from initial tool list construction through to the final API payload.

---

## The 4-Stage Tool Pipeline

Every API request Claude Code sends goes through four discrete stages to build the `tools` array.

### Stage 1: `ng()` — Built-in Tool Registry

`ng()` returns a hardcoded literal array containing every built-in tool (Read, Write, Edit, Bash, Grep, Glob, WebFetch, TodoWrite, etc.). The order of entries in this array is determined at bundle compile time and does not change between runs.

This is the only stage that is fully deterministic by construction.

### Stage 2: `FX()` — allowedTools / denyRules Filter

`FX()` takes the built-in array and applies `.filter()` using the `allowedTools` and `denyRules` configuration. Because `.filter()` preserves insertion order, the relative order of surviving entries is identical to Stage 1.

If you pass `allowedTools: ["Bash", "Read", "Write"]`, the resulting list will have those tools in their Stage 1 order, not in the order you specified. The `allowedTools` array is used as a membership test, not as an ordering instruction.

### Stage 3: `u66()` — Built-in + MCP Merge

`u66()` produces the final tool list by concatenating built-ins and MCP tools:

```javascript
[...builtins, ...mcpTools]
```

It then deduplicates by name using a key function `K0` (a Map-based dedup that keeps first occurrence). The result: built-ins always precede MCP tools, and if an MCP tool has the same name as a built-in, the built-in wins.

### Stage 4: `mGq()` → `Sh1()` — Serialization

`Sh1()` converts each tool to the API wire format:

```javascript
{
  name: tool.name,
  description: await A.prompt(),   // ASYNC
  input_schema: tool.inputJSONSchema ?? fU(tool.inputSchema)
}
```

Two details matter here:

**`description` is async.** Built-in tools return static strings synchronously. MCP tools may call the server for a dynamic description. Any non-determinism in the server response propagates directly into the serialized payload.

**Schema source differs by tool type.** MCP tools use `inputJSONSchema` (the schema as provided by the MCP server). Built-in tools derive their schema from Zod definitions via `fU()`, which is memoized by WeakMap — so built-in schemas are stable within a process.

`cache_control` is not applied at the individual tool level in the main query path. Tools are not individually cached; only the assembled system prompt gets cache markers.

---

## The Critical Finding: No `.sort()` Anywhere in the Tool Path

The entire `cli.js` source was searched for `.sort()` calls in proximity to tool-related code. Every `.sort()` call found belongs to one of these categories:

- Worktree path sorting
- Insights data sorting
- Help menu display
- Compact metadata output

There is no `.sort()` in `ng()`, `FX()`, `u66()`, `mGq()`, or `Sh1()`.

**Tool order is strictly insertion-order throughout all four stages.** This has a concrete consequence: if the insertion order of MCP tools varies between two runs, the serialized `tools` array is different, and the prompt cache misses.

---

## MCP Tool Ordering: Stable in Theory, Fragile in Practice

MCP tools enter the pipeline via `Fr6()`, which loads tools from all registered MCP servers using concurrent server initialization (`ZL1`). The loading is concurrent — server responses arrive in non-deterministic order.

The tool list stored in the session is built from the client registration order at startup. Config file order determines registration order. For a stable config file read from disk, registration order is consistent across restarts. For programmatically constructed configs (e.g., injected via SDK options), insertion order depends on the caller.

In practice, MCP tool order is usually stable. The fragility appears in two scenarios:

1. **Programmatic config construction** where tool order is not explicitly controlled (e.g., iterating over an object whose key order is not guaranteed).
2. **MCP server reconnect events**, where a server goes down and comes back. On reconnect, the tool list is re-registered, potentially at a different position in the merged array.

Both scenarios produce a structurally different `tools` array. The Anthropic API treats the entire `tools` block as part of the cache prefix. A different array = a different prefix = a full cache miss.

---

## Deferred Tool Loading: The Mid-Conversation Cache Breaker

This is the most impactful finding.

`GX()` implements deferred tool loading. The rule is simple: every tool where `isMcp === true` is deferred by default.

What "deferred" means in practice:

- The tool's full schema is **not** sent in the initial API request
- Instead, only the tool's name is listed in a `<available-deferred-tools>` block inside the system prompt
- The system prompt looks like: `<available-deferred-tools>mcp__server__toolname, ...</available-deferred-tools>`

When Claude determines it needs a deferred tool, a tool-search step runs that loads the full schema. At this point the `tools` array gains a new entry with the full `{name, description, input_schema}` object.

**This means the first invocation of any MCP tool in a conversation is a guaranteed cache miss for that turn.** The `tools` array before the call differs structurally from the `tools` array after the call. The cache prefix has changed.

In a multi-turn session:

| Turn | MCP Tool Used | tools array state | Cache result |
|------|---------------|-------------------|--------------|
| 1 | none | deferred only | hit (if stable) |
| 2 | `mcp__files__read` (first use) | +1 full schema entry added | **miss** |
| 3 | `mcp__files__read` (cached) | same as turn 2 | hit |
| 4 | `mcp__search__query` (first use) | +1 more full schema entry added | **miss** |
| 5+ | same tools | stable | hit |

Each unique MCP tool invoked for the first time in a session costs one cache miss. The more distinct MCP tools you use, the more guaranteed misses you accumulate before the session reaches a stable tool array.

A feature flag `tengu_defer_all` extends deferral to all tools, including built-ins. When active, the initial tools array is nearly empty and every tool use triggers a deferred load. This flag appears to be used in specific internal scenarios.

---

## `disallowedTools` Filtering: `_c()` Preserves Order

`_c()` materializes the `disallowedTools` setting into a Set, then applies `.filter()` to remove matching entries. Order is preserved. The filtering is applied at the agent or subagent level, sourced from the agent definition frontmatter.

From the SDK side, `disallowedTools` are passed as `--disallowedTools ToolA,ToolB` CLI arguments. No sorting is applied at this stage either.

---

## Complete Cache Impact Analysis

Ranked by likelihood and severity:

### High impact: Deferred tool loading (first MCP use per session)

Guaranteed cache miss on the turn a new MCP tool is first loaded. Affects every session that uses MCP tools. The miss is structural — the `tools` array is literally different before and after the deferred load.

### High impact: Dynamic MCP tool descriptions

If an MCP server's tool description is generated dynamically (e.g., includes a timestamp, a database record count, or any runtime state), the serialized description changes between turns. Even a single character difference in any tool description invalidates the entire `tools` block.

The `description` field in `Sh1()` comes from `await A.prompt()`. For MCP tools, this resolves from the server. If the server returns anything that varies, the cache breaks.

### Medium impact: MCP server config ordering

If `mcpServers` is passed as a JavaScript object (common in SDK usage), the key iteration order depends on insertion order, which is not guaranteed to be consistent across all environments and versions. Two requests with nominally identical config but different key insertion order produce different MCP tool arrays.

### Medium impact: `extraToolSchemas` variance

Some features inject additional tool schemas into the array only when active. Web search is one example: its schema is present in the tools array only when the web search feature is enabled for that turn. Toggling the feature mid-conversation changes the tools array structure.

### Low impact: `isEnabled()` state changes

Each tool has an `isEnabled()` predicate. MCP tools can become disabled if the MCP server connection drops. A reconnect that changes which tools are `isEnabled()` changes the filtered tool list. Rare in stable setups, but worth noting.

### No impact: Built-in tool descriptions

Built-in tools return static strings from `A.prompt()`. These strings are not async in practice and do not vary between runs. Built-in tool descriptions are safe from a cache perspective.

### No impact: Built-in tool schemas

Built-in schemas are derived from Zod definitions via `fU()`, memoized by WeakMap within the process lifetime. They do not change between turns and are safe.

---

## Mitigation Strategies

### 1. Sort `allowedTools` before passing to the SDK

```javascript
// Before (unstable — depends on object insertion order)
const options = {
  allowedTools: getEnabledTools()
};

// After (stable — alphabetical order)
const options = {
  allowedTools: getEnabledTools().sort()
};
```

One line. Eliminates one source of ordering instability.

### 2. Sort `mcpServers` keys for stable serialization

```javascript
// Before
const mcpServers = buildMcpConfig();

// After
const mcpServers = Object.fromEntries(
  Object.entries(buildMcpConfig()).sort(([a], [b]) => a.localeCompare(b))
);
```

Ensures that programmatically constructed MCP configs produce a consistent key order regardless of insertion order.

### 3. Make MCP tool descriptions completely static

If you control the MCP server, ensure its tool descriptions are hardcoded constants. No timestamps, no database lookups, no environment-dependent content in the `description` field.

```python
# Bad: dynamic description
@mcp.tool()
def search(query: str) -> str:
    """Search across {len(self.documents)} documents."""
    ...

# Good: static description
@mcp.tool()
def search(query: str) -> str:
    """Search across the document index."""
    ...
```

### 4. Pre-warm MCP tools at session start

If you know which MCP tools a session will use, trigger them early to force deferred loading before cache-sensitive turns. A no-op call to each tool at session initialization incurs the guaranteed miss upfront rather than mid-conversation.

```javascript
// Session initialization
await session.query("list the available mcp tools", { maxTurns: 1 });
// Now all MCP tools are loaded; subsequent turns can cache
```

### 5. Consider disabling deferred loading for cache-critical sessions

If your session uses a known, fixed set of MCP tools, disabling deferral means all schemas are serialized upfront. No mid-conversation cache busting from new tool loads. This is a tradeoff: larger initial request, but stable `tools` array from turn 1.

This requires patching or a flag — there is no current public API option to disable deferred loading directly.

### 6. Monitor the tools array across turns

Add logging to capture the serialized tools array for the first few turns of a session. If you see new entries appearing in turns 2, 3, 4, those are deferred loads causing cache misses. Quantify the pattern before deciding on mitigation.

---

## Impact on Real Systems

For a system using 3 MCP servers with 4–6 tools each:

- **Baseline**: 15–18 deferred tools at session start
- **First 15–18 turns**: at least one guaranteed miss per unique MCP tool invoked
- **After all tools loaded**: `tools` array stabilizes, cache can accumulate

If your average session is 10 turns and you have 15 MCP tools, every session potentially stays in a cache-miss state for its entire duration.

For a system with no MCP tools (built-ins only):

- Stage 1 output is compile-time stable
- Stage 2 output is stable if `allowedTools` is sorted or omitted
- No deferred loading
- Cache stability is essentially guaranteed from turn 1

---

## References

- [Reverse-Engineering the Claude Agent SDK: Root Cause and Fix for the 2–3% Credit Burn Per Message](../agent-sdk-cache-invalidation/README.md) — prerequisite reading; covers how the prompt cache works and why cache misses are expensive
- SDK version: `@anthropic-ai/claude-agent-sdk` v0.2.76, `cli.js` build `2026-03-14`
- Anthropic API prompt caching: cache reads at 10% of base input cost; cache writes at 125% of base input cost
- [Model Context Protocol specification](https://modelcontextprotocol.io) — MCP tool schema format referenced in `Sh1()`
