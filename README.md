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

Research is based on `@anthropic-ai/claude-code` v2.1.71 / Claude Code v2.1.71 (March 2026). Findings may change with future SDK updates.

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
