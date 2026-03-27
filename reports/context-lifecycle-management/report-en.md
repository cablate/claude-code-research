# Context Lifecycle Management — How Claude Code Decides When to Compress, What to Keep, and What It Costs

Long conversations grow. Every tool call, every code block, every back-and-forth adds tokens to the context window. Eventually the conversation has to shrink, or it hits a wall.

Claude Code has an automatic compression system called autocompact. It is not documented. It runs silently when your context gets too large, replaces most of the conversation with a summary, and lets you keep working as if nothing happened.

Except something did happen. The prompt cache was just destroyed. The next message costs 125% of normal. And under certain conditions, the compression itself triggers another compression, which destroys the cache again.

This report reverse-engineers the complete lifecycle: how context size is calculated, the five thresholds that govern decisions, when compression fires, what survives, and two previously undocumented mechanisms that silently inflate your costs.

SDK version analyzed: `@anthropic-ai/claude-agent-sdk` v0.2.76 (cli.js build 2026-03-14)

---

## How Context Size Is Calculated

Claude Code does not assume a fixed context window. It computes the effective capacity per model at runtime.

The first step determines the raw window. Models with `[1m]` in their name or the `1M` beta flag get 1,000,000 tokens. Sonnet 4.6 with the high-latency-allowance beta also gets 1,000,000. Everything else gets 200,000.

Then it subtracts two reservations. One is for the model's maximum output tokens — 64,000 for Opus 4.6, 32,000 for Opus 4.5 and Opus 4, 4,096 for Claude 3 Opus. The other is a fixed 20,000-token buffer that is permanently subtracted from every window, regardless of model.

The formula:

```
effective capacity = raw window - max output tokens - 20,000
```

For a standard 200k model with a 4,096-token output limit, the effective capacity is about 176,000 tokens. But this is not where compression triggers — there are additional margins on top of this.

---

## Five Thresholds, Five Behaviors

Claude Code evaluates five thresholds against the current token count. These are hardcoded constants with no UI, no config file, and no settings key. They can only be overridden through environment variables.

**The autocompact margin** subtracts another 13,000 tokens from the effective capacity. This is the trigger point for automatic compression. On a standard 200k model, this means compression fires at roughly 163,000 tokens — not 200,000, not 176,000, but 163,000. If you are planning around "it compresses near the limit," you are off by nearly 40,000 tokens.

**The warning threshold** adds a 20,000-token margin beyond the autocompact point. When context crosses this line, the UI shows a yellow indicator. No action is taken.

**The error threshold** adds another 20,000-token margin. The UI shows a red indicator. Still no action beyond the visual cue.

**The hard blocking limit** is 3,000 tokens from the absolute wall. At this point Claude Code refuses to accept new messages entirely. The session is frozen until context is reduced.

**The circuit breaker** counts consecutive compression failures. After 3 in a row, the system stops trying. This prevents infinite loops when compression cannot bring the context below threshold.

All five are evaluated together in a single pass on every turn. The function returns a state struct that the rest of the system reads to decide what to do next.

---

## The Decision Chain: When Autocompact Fires

Before checking whether the context is too large, the system runs through a series of gates.

First, three environment switches. If `DISABLE_COMPACT` is set, all compression is off — both proactive and reactive. If `DISABLE_AUTO_COMPACT` is set, only the proactive path is blocked; the emergency path still works. If the user settings have `autoCompactEnabled` set to false, same effect as the second switch. Any one of these being active means the system skips the threshold check entirely.

Second, a recursion guard. Compression is implemented as an LLM call — it sends the full conversation to Claude and asks for a summary. That means compression itself adds to the context. Without protection, a large summary could trigger another compression, which produces another summary, which triggers another. The guard checks the source of the current query: if it is already a compression call, or a session memory refresh, the threshold check is skipped.

Only after all these gates are clear does the system compare the current token count against the autocompact threshold. If the count is above, compression begins.

---

## What Happens During Compression

When the gates are clear and the threshold is exceeded, the compression executor kicks in. Here is the process from start to finish.

It starts by recording the current token count — a snapshot of how large the conversation is before anything changes. Then it runs any registered pre-compact hooks. These are extension points: if a hook exits with code 2, the entire compression is cancelled. This is the only external veto mechanism.

Next, it wipes the file tracking table. This table records which files Claude has read or written during the session and is used for detecting stale content. Wiping it forces a clean rebuild on the next turn, which prevents stale file diffs from accumulating in the compressed context.

Then it rebuilds memory files and tool definition attachments — making sure the fresh context has up-to-date references to project state.

The core of compression is building the summary prompt. The system constructs a 10-section request that asks the LLM to extract and preserve:

1. Environment setup — OS, shell, tool versions
2. Actual changes made — specific files created, modified, deleted
3. Files and code sections read — what was examined and why
4. Problem-solving history — errors, hypotheses, solutions
5. Pending work and blockers — what is not finished yet
6. User decisions and preferences — explicit choices made during the session
7. Project context and constraints
8. Current work state (marked as the most important section)
9. Optional or low-priority items mentioned but not started
10. Key artifacts — code blocks, configs, or file contents to preserve verbatim

This prompt is sent as a real API call with `maxTurns: 1`. The LLM responds with two XML sections: an `<analysis>` block for private reasoning (discarded) and a `<summary>` block that becomes the new conversation seed.

After the summary returns, the system measures the actual post-compression token count from the API response usage field — not an estimate, but the real number. It also computes a character-length estimate for UI display. Finally, it runs any registered post-compact hooks and the process is complete.

The old conversation history is gone. In its place: a summary, plus a preserved tail of recent messages.

---

## What Survives Compression

Compression does not replace the entire history. The most recent messages are preserved verbatim and appended after the summary.

The preservation logic works backward from the last message, accumulating tokens as it goes. It stops when it has collected either 40,000 tokens, or at least 10,000 tokens with at least 5 text-block messages — whichever comes first. Everything within this window is kept exactly as-is.

There is an integrity check: if a tool call is preserved, its corresponding tool result must also be preserved, and vice versa. Orphaned pairs are not allowed. This prevents the post-compression context from containing a tool invocation without its output, which would confuse the model.

The result is a new conversation: the LLM-generated summary at the top, then 10,000 to 40,000 tokens of the most recent raw exchange intact at the bottom. The preservation parameters are hardcoded — there is no way to expand or shrink this window through configuration.

---

## Original Finding: The Daily Cache Wipe Nobody Sees

We found that a 10-token date string silently invalidates thousands of tokens of cached context every day. Here is why.

Every turn, Claude Code injects a `<system-reminder>` block containing the current date:

```
Today's date is 2026-03-27
```

This is not surprising on its own — the model needs to know the date. The problem is where this injection sits in the message assembly.

The date string is concatenated into the same text block as the CLAUDE.md contents, git status, and other context that rarely changes. These are all joined into a single serialized unit before being sent to the API.

Prompt caching works on strict prefix matching. The cache key is the byte sequence of the entire prefix up to each cached block. When the date changes at midnight, the serialized content of this combined block changes. Because the date appears before the static CLAUDE.md content in the string, every byte after it shifts position in the prefix.

The consequence: every day at midnight, all cached context after this injection point is invalidated. The first session of the new day pays `cache_write` rates — 125% of base cost — on the full token count of the conversation history. Thousands of tokens of completely unchanged content are rewritten because a 10-character date string moved the prefix boundary.

This is not a bug in user code. It is a structural consequence of how the injection block is assembled — the date is embedded in the same serialized unit as static content instead of being placed in a separate block. As far as we can find, this mechanism has not been publicly described before.

For most interactive users, this is invisible — sessions rarely span midnight. But automated agents running overnight tasks pay this cost every 24 hours, and it compounds with conversation length.

---

## Original Finding: The Compression Chain Reaction

Compression is supposed to save costs by shrinking the context. But under certain conditions, it triggers a chain reaction that multiplies them.

Here is how.

During compression, the system correctly sets a flag to skip writing a new cache breakpoint — the summary is ephemeral and should not seed a new cache. After compression completes, the session state is reset. This invalidates all existing prompt cache breakpoints that had accumulated over the session.

The next turn after compression therefore behaves like the first turn of a brand-new session: every token is sent at `cache_write` rate (125%), and a new cache breakpoint is established from scratch.

Now the critical question: what if the compressed summary is still too large?

After compression, the system evaluates the post-compression token count against the threshold. If it still exceeds the autocompact trigger point, it flags that compression will retrigger on the next turn. When the user sends their next message, the threshold check fires again. A second compression runs. The session state resets again. Another full cache rebuild follows.

Each link in this chain costs three things: a full LLM compression call (the API cost of sending the entire context for summarization), a complete cache rebuild at 125% write rate on all tokens, and the potential to trigger the next link.

The chain stops when one of two things happens: the token count finally drops below threshold, or the circuit breaker trips after 3 consecutive failures.

This scenario is only possible when the context is genuinely enormous — a 1M-token session where even the summarized form exceeds the autocompact threshold. In interactive use, this is rare. But in automated agent loops running extended multi-hour tasks, it is a realistic failure mode. A single chain reaction of 2-3 links can cost the equivalent of 10-15 normal conversation turns.

---

## The Emergency Path: Reactive Compression

Autocompact is proactive — it fires before the API call based on token estimates. But estimates can be wrong.

If the system underestimates the context size and sends a request that is too large, the Anthropic API returns a `prompt_too_long` error. At this point, reactive compression kicks in.

It runs the same compression flow as autocompact, but with one important difference: you already paid for the failed API call. The tokens were sent, processed, and billed at `cache_write` rate before the rejection came back. Reactive compression then runs as a second API call. So you pay twice — once for the rejected request, once for the compression.

There is a safety limit: reactive compression is attempted at most once per turn. If the attempt itself fails, the system gives up and surfaces the error.

---

## Automatic Output Reduction

Separate from compression, Claude Code handles a different overflow scenario at the API level: when the input tokens plus the requested output tokens exceed the context limit.

If this happens, the system parses the error message to extract the actual numbers — input length, requested output, and context limit. It then computes a reduced output allocation:

```
reduced output = context limit - input length - 1,000
```

The 1,000-token margin is hardcoded. The floor is 3,000 tokens — it will not reduce the output allocation below that, to ensure the model can still produce a meaningful response.

The request is retried automatically with the reduced value. No user intervention, no visible error. The model just produces a shorter response than it would have otherwise.

---

## What You Can Tune

Claude Code exposes seven environment variables that affect context lifecycle behavior:

| Variable | What it does |
|---|---|
| `DISABLE_COMPACT` | Turns off all compression. If context genuinely exceeds the window, the session will hang on API errors. Use only if you manage context externally. |
| `DISABLE_AUTO_COMPACT` | Turns off proactive compression only. The emergency path still works, but you pay double when it triggers. |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | Sets the compression trigger as a percentage of the context window. `70` means compress at 70% capacity — useful for avoiding chain reactions by leaving headroom. |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | Overrides the context window size used in threshold calculations. |
| `CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE` | Overrides the hard blocking margin (default 3,000 tokens). |
| `CLAUDE_CODE_DISABLE_1M_CONTEXT` | Forces 200k window even for models eligible for 1M. Useful for cost control. |
| `ENABLE_CLAUDE_CODE_SM_COMPACT` | Enables an alternate compression path based on session memory. Disabled by default. |

The most useful for cost control is `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`. Setting it to 60-70% means compression fires earlier, when the context is smaller, producing a smaller summary that is less likely to still exceed the threshold. This is the simplest way to prevent chain reactions.

---

## The Full Cost Picture

A single compression event is not just one API call. Here is what actually happens to your bill:

| Event | Cost impact |
|---|---|
| The compression call itself | Full API call — entire conversation sent for summarization |
| First turn after compression | All tokens at 125% `cache_write` rate (cache was destroyed) |
| Second turn onward | Cache rebuilds normally, back to `cache_read` rates |
| Chain reaction (N links) | N times the above, compounding |
| Midnight cache wipe | One full `cache_write` rebuild, proportional to total conversation size |

For a 160,000-token session, one compression plus the post-compression cache rebuild costs roughly the same as 5-8 normal conversation turns. A chain reaction of 2-3 links doubles or triples that.

The practical guidance: if you are running agent loops, monitor the post-compression token count. If it is still above the trigger threshold, a chain reaction is already in progress. Lower your `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`, split the task into smaller sessions, or accept the cost spike and let the circuit breaker stop it after 3 iterations.

---

## Source Locations

All findings from `cli.js` in `@anthropic-ai/claude-agent-sdk` v0.2.76. Functions are located via string constant anchors — these survive minification across versions.

| Anchor string | What it locates |
|---|---|
| `"CLAUDE_CODE_DISABLE_1M_CONTEXT"` | Raw context window determination |
| `"isAboveAutoCompactThreshold"` | Threshold evaluation function |
| `"CLAUDE_AUTOCOMPACT_PCT_OVERRIDE"` | Autocompact trigger point derivation |
| `"DISABLE_AUTO_COMPACT"` | Environment switch gate |
| `"session_memory"` / `"compact"` | Recursion guard |
| `"preCompactTokenCount"` | Main compression executor |
| `"skipCacheWrite"` | Cache flag during compression |
| `"Today's date is"` | Date injection site |
| `"minTextBlockMessages"` | Preservation parameters |
| `"input length and max_tokens exceed context limit"` | Overflow recovery parser |
| `"prompt_too_long"` | Reactive compression entry point |

---

## Related Reports

- [Reverse-Engineering the Claude Agent SDK: Root Cause and Fix for the 2–3% Credit Burn Per Message](../agent-sdk-cache-invalidation/README.md) — how spawning a new process every call destroys the prompt cache
- [Prompt Cache Architecture](../prompt-cache-architecture/) — how cache breakpoints are placed and maintained
- [system-reminder Injection](../system-reminder-injection/) — full injection taxonomy and token costs
