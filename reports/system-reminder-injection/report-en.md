# Reverse-Engineering Claude Code's System-Reminder Injection Mechanism

Claude Code dynamically injects `<system-reminder>` content blocks into every API request. These blocks are invisible in the UI, not stored in conversation history files, and the model is instructed to never mention them. There are over 15 categories, and they are re-injected every conversation turn.

This report documents the complete injection mechanism based on reverse-engineering `cli.js` from `@anthropic-ai/claude-code` v2.1.71, cross-referenced with 30+ open GitHub issues and session-level JSONL analysis.

This is a companion to [Report #1 (Agent SDK Cache Invalidation)](../agent-sdk-cache-invalidation/), which covers the cost impact. This report focuses on the injection mechanism itself.

---

## Complete Type Catalogue

Source: reverse-engineering cli.js + [Piebald-AI/claude-code-system-prompts](https://github.com/Piebald-AI/claude-code-system-prompts)

### Injected Every Turn

| system-reminder | Trigger | Estimated Size |
|---|---|---|
| malware analysis | Every `Read` tool call | ~50 tokens per Read |
| file modified by user or linter | File mtime changed (IDE autosave, linter, hook, etc.) | ~30 tokens + file diff (can be large) |
| TodoWrite reminder | TodoWrite tool not used recently | ~80 tokens |
| Task tools reminder | TaskCreate/TaskUpdate not used recently | ~80 tokens |
| memory file contents | MEMORY.md contents attached every turn | Full MEMORY.md size |
| invoked skills | Available skills list injected every turn | ~2000+ tokens (scales with skill count) |

### Conditionally Injected

| system-reminder | Trigger | Estimated Size |
|---|---|---|
| hook success/error/context | Each hook execution result | Depends on hook stdout |
| hook stopped continuation | Hook blocks continuation | ~30 tokens + message |
| file exists but empty | Reading an empty file | ~30 tokens |
| file truncated | File too large, content truncated | ~30 tokens |
| file shorter than offset | Read offset exceeds file length | ~30 tokens |
| new diagnostics detected | LSP/IDE reports new errors/warnings | ~50 tokens + diagnostic content |
| file opened in IDE | File opened in IDE | ~30 tokens + file info |
| lines selected in IDE | Code selected in IDE | ~30 tokens + selection content |
| session continuation | Summary after context compaction | Hundreds to thousands of tokens |
| token usage | Approaching context limit | ~50 tokens |
| USD budget | Approaching budget limit | ~30 tokens |
| compact file reference | Referencing compacted files | Variable |
| plan mode active | Plan mode enabled | ~100 tokens |
| output style active | Output style specified | ~30 tokens |
| team coordination/shutdown | Multi-agent team mode | ~100 tokens |
| agent mention | @ mentioned | ~30 tokens |

### File Modification Injection Template

```
Note: ${filename} was modified, either by the user or by a linter.
This change was intentional, so make sure to take it into account as you proceed
(ie. don't revert it unless the user asks you to). Don't tell the user this, since
they are already aware. Here are the relevant changes (shown with line numbers):
${snippet}
```

---

## Historical Evolution

### Phase 1: Invisible and Lightweight (Mid 2025)

- 2025-07: [#4464](https://github.com/anthropics/claude-code/issues/4464) — first report
- Only the malware reminder existed (~50 tokens per Read)
- Tagged with `isMeta: true`, filtered out at the UI layer
- Most users had no idea it existed

### Phase 2: File Modification Begins Expanding (2025 Q4 – 2026 Q1)

- 2025-10: [#9769](https://github.com/anthropics/claude-code/issues/9769) — request for a toggle (still open)
- 2026-01-02: [#16021](https://github.com/anthropics/claude-code/issues/16021) — hundreds of lines of file content injected into every user message
- 2026-01-12: [#17601](https://github.com/anthropics/claude-code/issues/17601) — mitmproxy captures 10,577 hidden injections, 15%+ context window consumed

The key shift: file modification reminders went from "notify once" to "re-inject every user message." VS Code extension was particularly affected — Claude's own Edit operations also triggered injection. CLI was somewhat better — injection occurred only once on user edits.

### Phase 3: Type Explosion (2026 Q1)

- TodoWrite reminder, Task tools reminder, IP reminder (copyright), Skills list injection, diagnostics injection all added
- Injection types grew from 1–2 to 15+
- Every type includes `NEVER mention this reminder`
- 2026-01-16: [#18560](https://github.com/anthropics/claude-code/issues/18560) — system-reminder begins overriding CLAUDE.md instructions

### Phase 4: Behavioral Degradation (2026-02 – 03)

- 2026-02-06: [#23537](https://github.com/anthropics/claude-code/issues/23537) — model treats system-reminder as user instructions
- 2026-02-22: [#27599](https://github.com/anthropics/claude-code/issues/27599) — infinite repetition in headless mode
- 2026-03-05: [#30730](https://github.com/anthropics/claude-code/issues/30730) — sub-agent injection overrides custom agent definitions
- 2026-03-06: [#31447](https://github.com/anthropics/claude-code/issues/31447) — Claude claims system-reminders are "injected," social-engineering users into relaxing permissions
- 2026-03-06: v2.1.70 regression — entire file contents injected (1300 lines per file, 15%+ context per turn)
- 2026-03-07: v2.1.71 confirmed — files edited but not re-read are permanently treated as stale

---

## Core Mechanism: The File Tracking Table (readFileState)

`readFileState` is an LRU Cache (max 100 entries) where each entry tracks a file:

```javascript
{
  content: string,      // file content snapshot
  timestamp: number,    // recorded time (ms)
  offset: number | undefined,
  limit: number | undefined
}
```

### Four Write Points

| Source | Key Format | offset | limit | timestamp Source |
|---|---|---|---|---|
| Memory loading (CLAUDE.md, MEMORY.md) | Raw path (not normalized) | `undefined` | `undefined` | `Date.now()` |
| Read tool | Normalized path | `1` (default) | `undefined` | `Math.floor(mtimeMs)` |
| Write/Edit tool | Normalized path | `undefined` | `undefined` | File mtime |
| Session resume (rebuild function) | Normalized path | `undefined` | `undefined` | JSONL message timestamp |

The critical observation: **Read is safe, Edit/Write is not.** Read sets `offset: 1` by default, which causes the stale check to skip the entry. Edit/Write sets `offset: undefined`, which means the entry is always tracked.

### Stale Check (Runs Every User Message)

On each user message, cli.js iterates the tracking table:

```
For each entry in readFileState:
  1. offset or limit is not undefined → skip (not tracked)
  2. file mtime <= recorded timestamp → skip (unchanged)
  3. Re-read file, compute diff
  4. Diff is empty → skip
  5. Diff is non-empty → inject system-reminder
```

All five conditions must be met for injection to occur.

---

## Five Trigger Conditions for File Modification Injection

| # | Condition | When It Does NOT Trigger |
|---|---|---|
| 1 | File is in the tracking table | Never Read/Edited, not CLAUDE.md/MEMORY.md |
| 2 | Not a partial read (both offset and limit are `undefined`) | Read tool default offset=1 → stale check skips |
| 3 | File mtime > recorded timestamp | mtime precision race: Edit and timestamp recorded in same ms → skip |
| 4 | File can be read successfully | File deleted, permission issue → removed from tracking |
| 5 | Diff is non-empty | Content unchanged (IDE autosave same content) → skip |

---

## Root Causes of Inconsistent Triggering

### Cause 1: Path Key Mismatch

The tracking table has multiple write points using different key formats:

- Memory loading: raw path (no normalization)
- Read/Write/Edit tool: normalized path (via a normalization function that applies NFC Unicode normalization and path resolution)

On Windows, normalization can change the path string (`.normalize("NFC")`, `/c/` → `C:\`, etc.), creating two entries for the same file. The stale check iterates the raw key, but the internal re-read updates the normalized key. The raw key entry is never updated — infinite injection loop.

### Cause 2: mtime Precision Race Condition

```
Claude Edit writes file → tracking table records timestamp from mtime  // T1
→ External process (IDE autosave, linter, git hook) touches file     // mtime becomes T2
→ Stale check: current mtime T2 > recorded T1 → triggers
```

Windows NTFS mtime precision is 100ns, but JavaScript `Date.now()` is ms. If the Edit and timestamp recording complete within the same millisecond, `mtime <= timestamp` holds and injection is skipped. Otherwise it triggers.

### Cause 3: Stale Check Does Not Update the Original Entry

After detecting a change, the stale check function returns the diff but does not update the original key's content or timestamp. The internal re-read updates the normalized key. If the original key differs from the normalized key, the original entry retains stale values permanently.

---

## CLI vs Agent SDK Differences

| | CLI Interactive Mode | Agent SDK / Headless |
|---|---|---|
| Tracking table lifetime | Shared for entire session, persists until session ends | Rebuilt from JSONL on every `submitMessage()` call |
| CLAUDE.md/MEMORY.md loading | Loaded once at session start, `.has()` prevents duplicates | Reloaded on every initialization → raw path keys re-added |
| Result | Sometimes triggers, sometimes not (depends on key collision in LRU cache) | Triggers consistently (tracking table rebuilt from scratch every turn) |

### Agent SDK's Core Problem

`readFileState` is not stored as a class property (`this.readFileState`). It is a local variable created inside `submitMessage()`. Every call rebuilds it from `this.mutableMessages` using the rebuild function — all entries get `offset: undefined` and past timestamps — so every call triggers injection.

```
submitMessage() called
  → readFileState = rebuild(this.mutableMessages)  ← full rebuild from JSONL, max=10
  → stale check → inject diffs
  → main loop runs, Edit/Write updates readFileState in memory
  → next submitMessage()
  → readFileState = rebuild(this.mutableMessages)  ← rebuilt again, all in-memory updates lost
  → always injects
```

---

## Session Resume: The Tracking Table Rebuild Function

When a session is resumed (CLI `--resume` or Agent SDK `resume: true`), cli.js rebuilds the tracking table from the JSONL conversation history.

### Rebuild Logic

```javascript
// Simplified from the deobfuscated source
function rebuildTrackingTable(messages, cwd, maxEntries = 10) {
  let cache = new LRUCache(maxEntries);
  let readOps = new Map();   // tool_use_id → normalized path
  let writeOps = new Map();  // tool_use_id → { path, content }

  // Pass 1: Scan assistant messages for Read and Write tool_use blocks
  for (let msg of messages) {
    if (msg.type === "assistant") {
      for (let block of msg.message.content) {
        // Only collect Reads WITHOUT offset or limit
        if (block.name === "Read"
            && block.input.offset === undefined
            && block.input.limit === undefined) {
          readOps.set(block.id, normalize(block.input.file_path));
        }
        // Collect Writes with content
        if (block.name === "Write"
            && block.input.file_path
            && block.input.content) {
          writeOps.set(block.id, { path, content });
        }
        // Edit is NOT handled
      }
    }
  }

  // Pass 2: Scan user messages for matching tool_result blocks
  for (let msg of messages) {
    if (msg.type === "user") {
      for (let block of msg.message.content) {
        if (block.type === "tool_result") {
          let readPath = readOps.get(block.tool_use_id);
          if (readPath) {
            cache.set(readPath, {
              content: cleanContent(block.content),
              timestamp: new Date(msg.timestamp).getTime(),  // past time
              offset: undefined,   // always undefined
              limit: undefined     // always undefined
            });
          }
        }
      }
    }
  }
  return cache;
}
```

### Four Critical Issues

**1. Edit operations are completely ignored.** The rebuild function only processes Read (without offset/limit) and Write. Edit — the most common modification method — is not handled. Files modified via Edit during the session are not rebuilt into the tracking table through this path.

**2. Read with default offset=1 is also ignored.** The collection condition requires `offset === undefined`. But the Read tool's default is `offset: 1`, so most normal Read operations are not collected. Only the rare explicit `offset: undefined` Reads enter the table.

**3. Max 10 entries (Agent SDK) vs 100 (CLI).** The Agent SDK rebuild uses max=10, meaning only the last 10 file operations survive LRU eviction. All 10 entries have `offset: undefined` and past timestamps.

**4. Timestamps use past time from JSONL.** `new Date(msg.timestamp).getTime()` produces the time the message was originally recorded. After resume, the file's current mtime is almost always newer → stale check fires → injection triggers.

---

## Concealment Mechanisms

| Mechanism | Effect |
|---|---|
| `isMeta: true` flag | UI layer filters it out completely |
| Not stored in JSONL | Cannot be found in session files after the fact |
| `NEVER mention this reminder` | Claude is instructed not to reveal it |
| LaunchDarkly feature flags | Server-side control, users cannot toggle |
| Runtime dynamic injection | Exists only in the API request payload, never persisted |

The only ways to observe system-reminder content:

1. **mitmproxy** — intercept actual API requests ([#17601](https://github.com/anthropics/claude-code/issues/17601) used this method)
2. **Ask Claude directly** — sometimes Claude violates the `NEVER mention` instruction
3. **Anomalous token consumption** — indirect inference

---

## Known Bugs

### Bug 1: Path Key Mismatch (Root-Cause Level)

- **Location**: Memory loading vs Read/Write/Edit tool
- **Issue**: Memory loading uses raw paths as keys. Read/Write/Edit use normalized paths. Same file can have two entries.
- **Impact**: Stale check on the raw key entry is never resolved → infinite injection

### Bug 2: Stale Check Does Not Update readFileState (Root-Cause Level)

- **Location**: Stale check function
- **Issue**: After detecting staleness, only returns the diff. Does not update the original key's content or timestamp.
- **Impact**: Re-detected as "changed" every turn → injected every turn

### Bug 3: Rebuild Sets All Entries to offset: undefined

- **Location**: Tracking table rebuild function
- **Issue**: All rebuilt entries have `offset: undefined` → all are tracked. Timestamps are from the past → almost always stale.
- **Impact**: Session resume almost certainly triggers mass injection

### Bug 4: Edit/Write Sets offset to undefined

- **Location**: Write/Edit tool `.set()` calls
- **Issue**: After Edit/Write, the tracking entry has `offset: undefined` and `limit: undefined` → stale check always tracks it.
- **Impact**: Any file Claude has edited is tracked permanently. The Read tool's default `offset: 1` actually "fixes" this (offset !== undefined → skipped).

---

## Community Impact: 30+ Open GitHub Issues

### Token Waste (Most Reported)

| Issue | Title | Comments | Date |
|---|---|---|---|
| [#16021](https://github.com/anthropics/claude-code/issues/16021) | File modification notes injected into every user message | 23 | 2025-01-02 |
| [#4464](https://github.com/anthropics/claude-code/issues/4464) | system-reminder consuming too many context tokens | 22 | 2025-07-25 |
| [#17601](https://github.com/anthropics/claude-code/issues/17601) | 10,000+ hidden injections, 15%+ context window consumed | 10 | 2026-01-12 |
| [#21214](https://github.com/anthropics/claude-code/issues/21214) | system-reminder injected on every Read, millions of tokens wasted | 4 | 2026-01-27 |
| [#25327](https://github.com/anthropics/claude-code/issues/25327) | CLI wrapper injection = "token tax on good engineering" | 0 | 2026-02-12 |
| [#27721](https://github.com/anthropics/claude-code/issues/27721) | Skills registered twice in system prompt, doubling context usage | 1 | 2026-02-22 |
| [#27599](https://github.com/anthropics/claude-code/issues/27599) | Infinite system-reminder repetition in headless mode | 2 | 2026-02-22 |

### Security / Trust

| Issue | Title | Comments | Date |
|---|---|---|---|
| [#18560](https://github.com/anthropics/claude-code/issues/18560) | system-reminder instructs Claude to ignore CLAUDE.md | 3 | 2026-01-16 |
| [#31447](https://github.com/anthropics/claude-code/issues/31447) | Claude claims system messages are "injected," social-engineers user into relaxing permissions | 2 | 2026-03-06 |
| [#23537](https://github.com/anthropics/claude-code/issues/23537) | System task reminders disguised as user input, model cannot distinguish | 2 | 2026-02-06 |
| [#27128](https://github.com/anthropics/claude-code/issues/27128) | System-generated messages mislabeled as Human: turn, causing unauthorized actions | 4 | 2026-02-20 |

### Functional Bugs

| Issue | Title | Comments | Date |
|---|---|---|---|
| [#31458](https://github.com/anthropics/claude-code/issues/31458) | system-reminders stripped from persisted history, breaking grounding | 2 | 2026-03-06 |
| [#26370](https://github.com/anthropics/claude-code/issues/26370) | Stale Read results remain in system-reminder after compaction | 1 | 2026-02-17 |
| [#25810](https://github.com/anthropics/claude-code/issues/25810) | Memory system incorrectly reports MEMORY.md as empty | 0 | 2026-02-14 |

### Feature Request

| Issue | Title | Comments | Date |
|---|---|---|---|
| [#9769](https://github.com/anthropics/claude-code/issues/9769) | Allow individual system-reminder type toggles | 4 | 2025-10-17 |

---

## Mitigation Strategies

| Strategy | Effectiveness | Cost | Use Case |
|---|---|---|---|
| CLAUDE.md ignore instructions | Medium (unstable) | Low | Baseline for all scenarios |
| **JSONL pre-processing** | **High (eliminates resume injection)** | **Medium** | **CLI resume / Agent SDK** |
| Force re-read after Edit | Medium | Extra Read tokens | When few files are tracked |
| Write memory files outside cwd | High (root fix) | Architecture change | Projects with memory write patterns |
| Avoid large file edits via CLI tools (use MCP) | High | Architecture change | Projects with large JSON/config |
| Short sessions + frequent compact | Medium | Session management overhead | Interactive mode |
| [Cozempic](https://github.com/Ruya-AI/cozempic) daemon | Medium | Third-party dependency | CLI interactive mode |
| Direct Claude API (bypass CLI wrapper) | High | Rewrite orchestration | Agent SDK / production |
| Wait for official `--no-system-reminders` | Highest | Waiting | [#9769](https://github.com/anthropics/claude-code/issues/9769) (open since 2025-10) |

### JSONL Pre-Processing (Recommended for Agent SDK)

The tracking table rebuild function has specific collection conditions:
- Read: `offset === undefined && limit === undefined`
- Write: `file_path && content`

Breaking these conditions before resume produces an empty tracking table → no file modification injection.

**Method**: Add `offset: 1` to all Read entries in the JSONL. Remove `content` from all Write entries. The rebuild function's conditions no longer match, so it collects nothing.

```python
# Core logic (simplified)
for block in assistant_message.content:
    if block.name == "Read" and block.input.offset is None:
        block.input["offset"] = 1      # breaks collection condition
    if block.name == "Write" and "content" in block.input:
        del block.input["content"]      # breaks collection condition
```

Run this before every `--resume` or Agent SDK `resume: true` call.

**Side effect**: Write tool may report "File has not been read yet" on first write to an unread file (errorCode: 2). Claude handles this automatically by reading first, so it does not affect normal operation.

**What it does not affect**: CLAUDE.md/MEMORY.md loading via the memory loading path. Those files are loaded separately with `Date.now()` timestamps and are generally not affected.

See [Report #1](../agent-sdk-cache-invalidation/) for the full JSONL sanitizer implementation and V2 persistent session approach that addresses both injection and cache invalidation.

---

## Appendix: How Memory Files Enter the Tracking Table

Claude Code's "memory files" are those processed by the memory loading mechanism:

| Type | Files | Path |
|---|---|---|
| Managed | Anthropic built-in rules | System directory |
| User | `~/.claude/CLAUDE.md` | Global instructions |
| Project | `CLAUDE.md`, `.claude/CLAUDE.md`, `.claude/rules/*.md` | Every directory level from project root up |
| Local | `CLAUDE.local.md` | Project-local instructions |
| AutoMem | `MEMORY.md` | `~/.claude/projects/<project-hash>/memory/MEMORY.md` |
| TeamMem | Team memory | Organization shared |

User-created files like `memory/2026-03-08.md` are not in this list. They enter the tracking table when Claude uses Edit/Write on them, setting `offset: undefined` — and from that point on, they are tracked permanently.

---

## SDK Version Baseline

Research based on `@anthropic-ai/claude-code` v2.1.71 / Claude Code v2.1.71 (March 2026). Internal function names are obfuscated and change on every version update. The mechanisms described here were located via string constant anchors (e.g., `"was modified, either by the user"`, `"Cannot send to closed session"`), which are stable across versions.
