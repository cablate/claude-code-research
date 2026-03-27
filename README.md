# Claude Code Research

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC_BY_4.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)
[![Contributions Welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg)](./CONTRIBUTING.md)

Independent research on Claude Code internals, Claude Agent SDK, and related tooling. All findings are based on reverse engineering the publicly distributed npm packages.

Each report includes both English and Chinese versions.

## Why This Exists

Over **30 open GitHub issues** document unexplained token waste, security concerns, and architectural problems in Claude Code — many with dozens of comments and no official response. This repository provides the technical root-cause analysis that the community has been asking for, along with working solutions.

## Reports

| # | Topic | Description |
|---|-------|-------------|
| 1 | [Agent SDK Cache Invalidation](./reports/agent-sdk-cache-invalidation/) | Why Agent SDK V1 `query()` costs 3–10x more than CLI per message — prompt cache invalidation caused by process-per-call architecture. Fix: V2 persistent session + SDK patch. |
| 2 | [System-Reminder Injection](./reports/system-reminder-injection/) | Complete reverse-engineering of the `<system-reminder>` injection mechanism — 15+ hidden injection types, the readFileState tracking system, 4 root-cause bugs, and why the Agent SDK triggers injection on every single turn. |
| 3 | [Prompt Cache Architecture](./reports/prompt-cache-architecture/) | How Claude Code controls what gets cached and for how long — the single `cache_control` factory (`Ml()`), per-model disable gates, server-side 1h TTL gating via feature flag allowlist, system prompt static/dynamic zone split, message-level sliding window, and why byte-for-byte prefix matching makes injection order critical. |
| 4 | [Tool Serialization & Cache Stability](./reports/tool-serialization-cache-stability/) | The 4-stage tool pipeline, why there is zero `.sort()` on tool arrays, how deferred tool loading silently busts the cache mid-conversation, and MCP tool description dynamism as a hidden instability source. |
| 5 | [Context Lifecycle Management](./reports/context-lifecycle-management/) | How Claude Code decides when to compress context — 5 hardcoded threshold constants, the 10-step compaction flow, preserved message segments, the `currentDate` daily cache-kill problem, and compact chain reactions that compound cache rebuild costs. |
| 6 | [Production Cache Optimization](./reports/production-cache-optimization/) | Concrete, tested patches and strategies for maximizing prompt cache efficiency — 3 cli.js patches (context margin, 1h TTL force, compaction threshold), cache keepalive, tool ordering stabilization, efficiency monitoring, and the postinstall patch delivery pattern. |

## Related Issues

### Token Waste
- [anthropics/claude-code#16021](https://github.com/anthropics/claude-code/issues/16021) — File modification reminders injected every message (23 comments)
- [anthropics/claude-code#4464](https://github.com/anthropics/claude-code/issues/4464) — system-reminder consuming too many context tokens (22 comments)
- [anthropics/claude-code#17601](https://github.com/anthropics/claude-code/issues/17601) — 10,000+ hidden injections consuming 15%+ context
- [anthropics/claude-code#27599](https://github.com/anthropics/claude-code/issues/27599) — Infinite system-reminder repetition in headless mode

### Security / Trust
- [anthropics/claude-code#18560](https://github.com/anthropics/claude-code/issues/18560) — system-reminder instructs Claude to ignore CLAUDE.md
- [anthropics/claude-code#31447](https://github.com/anthropics/claude-code/issues/31447) — Claude social-engineers users into relaxing permissions
- [anthropics/claude-code#23537](https://github.com/anthropics/claude-code/issues/23537) — System reminders disguised as user input
- [anthropics/claude-code#27128](https://github.com/anthropics/claude-code/issues/27128) — System messages mislabeled as Human: turn

### SDK / Architecture
- [anthropics/claude-agent-sdk-typescript#188](https://github.com/anthropics/claude-agent-sdk-typescript/issues/188) — SDK defaults to 1h cache TTL (2x write cost)
- [anthropics/claude-code#9769](https://github.com/anthropics/claude-code/issues/9769) — Request for system-reminder toggle (open since 2025-10)
- [anthropics/claude-code#30730](https://github.com/anthropics/claude-code/issues/30730) — Sub-agent injection overrides custom agent definitions

## SDK Version Baseline

Reports #1–2 are based on `@anthropic-ai/claude-code` v2.1.71. Reports #3–6 are based on `@anthropic-ai/claude-agent-sdk` v0.2.76 (cli.js build 2026-03-14). Findings may change with future SDK updates.

## How to Cite

If you reference this research in blog posts, articles, or other projects:

```
CabLate, "Claude Code Research," GitHub, 2026.
https://github.com/cablate/claude-code-research
```

## Contributing

We welcome new research findings, corrections, and updates. See [CONTRIBUTING.md](./CONTRIBUTING.md) for submission guidelines, quality standards, and the report structure template.

## Disclaimer

This is independent research. It is **not affiliated with, endorsed by, or sponsored by Anthropic**. All analysis is performed on publicly distributed npm packages. No proprietary source code is reproduced.

## License

[CC-BY-4.0](./LICENSE) — You are free to share and adapt this material with appropriate attribution.
