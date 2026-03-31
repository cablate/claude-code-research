# Claude Code Research

[![Contributions Welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg)](./CONTRIBUTING.md)

Independent research on Claude Code internals.

## What's in this repo

| Directory | What | Source | Count |
|-----------|------|--------|-------|
| [`source-code-analysis/`](./source-code-analysis/) | Full architecture reverse-engineering | Leaked TypeScript source (v2.1.88) | **75 reports** |
| [`reports/`](./reports/) | Problem-oriented investigations (cache, cost, injection) | Minified npm bundle (`cli.js`) | **8 reports** |

> **Looking for the source code analysis?** Go to [`source-code-analysis/`](./source-code-analysis/).
> **Looking for specific bug/cost investigations?** Go to [`reports/`](./reports/).

---

## Source Code Analysis (`source-code-analysis/`)

On 2026-03-31, the full source code of Claude Code was exposed via a sourcemap file in the npm registry ([discovered by Chaofan Shou](https://x.com/shoucccc), archived by [Kuberwastaken](https://github.com/Kuberwastaken/claude-code) and [sanbuphy](https://github.com/sanbuphy/claude-code-source-code)).

We performed a 10-domain, 75-report analysis — the most thorough publicly available breakdown of how a production AI coding agent works.

**[Browse all reports →](./source-code-analysis/)**

Interactive viewer — open `source-code-analysis/index.html` in your browser (all 75 reports are embedded, no server needed).

### Key Findings

| Domain | Reports | Highlights |
|--------|---------|------------|
| [Harness Engineering](./source-code-analysis/phase-09-harness-engineering/) ⭐ | 7 | Agent Loop reverse-engineering, 12 transferable harness design principles |
| [System Prompt](./source-code-analysis/phase-01-system-prompt/) ⭐ | 6 | Complete 13-section prompt with dynamic assembly logic, 17 prompt engineering patterns |
| [Cost & Quota](./source-code-analysis/phase-10-cost-quota/) ⭐ | 8 | Cost envelope, prompt cache break detection (12 causes), Haiku→Opus 37.5x cost gap |
| [Tool Definitions](./source-code-analysis/phase-02-tool-definitions/) | 8 | All 36 tool prompts, read/write concurrency separation, ant vs public prompt variants |
| [Agent Architecture](./source-code-analysis/phase-03-agent-architecture/) | 7 | 6 built-in agents, Coordinator mode, Swarm multi-agent, 50-message cap (from 36.8GB incident) |
| [Security](./source-code-analysis/phase-06-security-permissions/) | 8 | 7-layer defense-in-depth, 23 Bash validators, Parser Differential threat model |
| [Skills System](./source-code-analysis/phase-04-skills-system/) | 5 | 16 bundled skills, 12 design patterns + 5 anti-patterns |
| [Memory & Context](./source-code-analysis/phase-05-memory-context/) | 9 | 6 memory subsystems, AutoDream consolidation, Team Memory dual-layer security |
| [API & Models](./source-code-analysis/phase-07-api-model-architecture/) | 7 | 17 beta headers, 4 providers, model selection 5-layer priority |
| [Hidden Features](./source-code-analysis/phase-08-special-features/) | 10 | 82 feature flags, KAIROS proactive mode, Buddy AI pet, UltraPlan, anti-distillation |

---

## Behavioral Reports (`reports/`)

Problem-oriented investigations based on reverse engineering the minified `cli.js` from npm. Each includes English and Chinese versions.

| # | Topic | TL;DR |
|---|-------|-------|
| 1 | [Agent SDK Cache Invalidation](./reports/agent-sdk-cache-invalidation/) | SDK `query()` costs 3–10x more than CLI — process-per-call kills prompt cache |
| 2 | [System-Reminder Injection](./reports/system-reminder-injection/) | 15+ hidden injection types, 4 root-cause bugs |
| 3 | [Prompt Cache Architecture](./reports/prompt-cache-architecture/) | Static/dynamic zone split, sliding window, byte-prefix matching |
| 4 | [Tool Serialization & Cache Stability](./reports/tool-serialization-cache-stability/) | Zero `.sort()` on tools, deferred loading busts cache mid-conversation |
| 5 | [Context Lifecycle Management](./reports/context-lifecycle-management/) | 5 threshold constants, 10-step compaction flow, chain reactions |
| 6 | [Production Cache Optimization](./reports/production-cache-optimization/) | 3 concrete cli.js patches + monitoring strategies |
| 7 | [Cache Invalidation Verification](./reports/cache-invalidation-verification/) | `defer_loading` excludes deferred tools from cache prefix entirely |
| 8 | [Auto Mode Classifier Cost](./reports/auto-mode-classifier-cost/) | Hidden Opus-level call before every side-effecting tool use |

---

## Resources

- [CLI Reverse Engineering Guide](./research/cli-reverse-engineering-guide.md) — search patterns & function locators for minified cli.js
- [Analysis Plan](./source-code-analysis/ANALYSIS-PLAN.md) — methodology for the 10-phase source code analysis

## Source Code Archives

The analysis is based on the full TypeScript source of Claude Code v2.1.88. The source code itself is not included in this repo.

- [chatgptprojects/claude-code](https://github.com/chatgptprojects/claude-code) — full source code
- [Kuberwastaken/claude-code](https://github.com/Kuberwastaken/claude-code) — source code breakdown & analysis
- [sanbuphy/claude-code-source-code](https://github.com/sanbuphy/claude-code-source-code) — source code archive

## Version Baseline

| Scope | Version |
|-------|---------|
| Source Code Analysis | v2.1.88 (sourcemap leak, 2026-03-31) |
| Behavioral Reports #1–2 | v2.1.71 |
| Behavioral Reports #3–6 | Agent SDK v0.2.76 (build 2026-03-14) |
| Behavioral Reports #7–8 | v2.1.85 / v2.1.88 |

## How to Cite

```
CabLate, "Claude Code Research," GitHub, 2026.
https://github.com/cablate/claude-code-research
```

## Disclaimer

This is independent research, **not affiliated with or endorsed by Anthropic**. Behavioral reports analyze publicly distributed npm packages. Source code analysis is based on code exposed through npm registry sourcemaps.

