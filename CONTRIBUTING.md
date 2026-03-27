# Contributing to Claude Code Research

Thank you for your interest in contributing to this research repository. This project documents reverse-engineering findings about Claude Code internals and the Claude Agent SDK. Contributions that advance collective understanding of these systems are welcome.

---

## Table of Contents

- [What We Accept](#what-we-accept)
- [Research Quality Standards](#research-quality-standards)
- [Report Structure Template](#report-structure-template)
- [How to Submit New Research](#how-to-submit-new-research)
- [How to Report Inaccuracies](#how-to-report-inaccuracies)
- [PR Process](#pr-process)
- [Writing Guidelines](#writing-guidelines)

---

## What We Accept

- **New research reports** — Original findings from reverse-engineering Claude Code or Claude Agent SDK
- **Corrections** — Inaccuracies in existing reports backed by evidence
- **Updates** — Re-analysis of a finding after an SDK version upgrade
- **Supplements** — Additional evidence, reproductions, or diagrams for existing reports

We do not accept:
- Speculative content without evidence
- Findings based on Claude's responses rather than source code or observable behavior
- Content that reproduces Anthropic's proprietary source code verbatim

---

## Research Quality Standards

Every submission must meet all three criteria:

### 1. Evidence

Claims must be backed by at least one of:
- Direct reference to decompiled/minified source in the npm package (file path + line range)
- A reproducible test showing observable behavior (token counts, injection content, timing)
- Network traces or API logs confirming the behavior

Statements like "Claude seems to..." or "probably because..." without supporting evidence will be rejected.

### 2. Reproducibility

Include enough information for another researcher to independently verify:
- Exact SDK/package version (e.g., `@anthropic-ai/claude-code@2.1.71`)
- Steps to reproduce the observation
- Expected output and actual output
- Environment notes if platform-specific (OS, Node.js version)

### 3. SDK Version Baseline

Always state which SDK version your findings apply to. The current research baseline is:

```
@anthropic-ai/claude-code v2.1.71 (March 2026)
```

If your findings differ from published reports, note whether the difference is due to a version change.

---

## Report Structure Template

Each new research report should be placed in `reports/<slug>/` and follow this structure:

```
reports/your-report-slug/
  README.md       # English version (required)
  README.zh-TW.md # Chinese Traditional version (encouraged)
  diagrams/       # Optional: architecture diagrams, flow charts
  evidence/       # Optional: annotated screenshots, log excerpts
```

The `README.md` must follow this template:

```markdown
# [Title]

> **SDK Version:** @anthropic-ai/claude-code vX.Y.Z
> **Date:** YYYY-MM-DD
> **Status:** Draft | Reviewed | Verified

## Summary

One paragraph describing the finding and why it matters.

## Methodology

How you investigated this. Tools used (e.g., `npm pack`, `node --inspect`, token counting),
decompilation approach, test harness design.

## Findings

Structured presentation of what you found. Use subheadings. Include code blocks where relevant.

## Evidence

Direct references to source locations, log excerpts, or test output that supports each finding.
Format source references as: `node_modules/@anthropic-ai/claude-code/dist/...` + line range.

## Impact

Token cost, security implications, behavioral effects, or API misuse potential.

## Mitigation / Workaround

Known workarounds, patches, or SDK-level fixes if applicable.

## References

Links to related GitHub issues, SDK changelogs, or prior research.
```

---

## How to Submit New Research

1. **Fork** the repository
2. Create a branch: `research/<your-topic>`
3. Create your report directory under `reports/`
4. Follow the [Report Structure Template](#report-structure-template)
5. Update the report index table in `README.md`
6. Open a Pull Request with title: `[Report] Your topic title`

Before opening the PR, confirm:
- [ ] SDK version is stated
- [ ] Evidence section has at least one verifiable reference
- [ ] Reproduction steps are included
- [ ] No Anthropic proprietary source code is reproduced verbatim

---

## How to Report Inaccuracies

If you find an error in an existing report:

1. **Open a GitHub Issue** with title: `[Correction] Report #N — brief description`
2. In the body, include:
   - Which specific claim is incorrect
   - What the correct finding is
   - Your evidence (SDK version, source reference, or test output)

If the correction is straightforward, you may also submit a PR directly that edits the relevant report and adds a note in the `## Changelog` section at the bottom.

---

## PR Process

1. Open PR against `main`
2. Fill in the PR template (title, summary, evidence checklist)
3. A maintainer will review within 7 days
4. Feedback will focus on: evidence quality, reproducibility, and accuracy — not writing style
5. Once approved, the report is merged and the README index is updated

For significant research, expect a discussion period before merge.

---

## Writing Guidelines

- **Language:** English is required. Chinese Traditional (zh-TW) is strongly encouraged as a parallel version. Both are equally welcome — do not feel obligated to write both if it reduces quality.
- **Tone:** Neutral and technical. This is documentation, not advocacy.
- **Code blocks:** Use fenced code blocks with language tags. For minified JS excerpts, use `js`.
- **Diagrams:** Welcome and encouraged. Use Mermaid (renders in GitHub), PNG exports, or SVG. Place under `reports/<slug>/diagrams/`.
- **Links:** Link to specific GitHub Issues when discussing known problems. Use permalink format for source file references when possible.
- **Length:** No minimum or maximum. Say what needs to be said. Prefer depth over breadth in a single report.
- **Speculation:** Clearly mark speculative conclusions with `> **Note:** This is inferred from...` — do not present them as facts.

---

## Questions

Open a GitHub Issue with the `question` label, or start a Discussion.
