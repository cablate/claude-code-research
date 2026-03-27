# Reverse-Engineering the Claude Agent SDK: Root Cause and Fix for the 2–3% Credit Burn Per Message
# 逆向工程 Claude Agent SDK：每條訊息 2–3% 額度燃燒的根本原因與修復

[English Version](./report-en.md) | [繁體中文版](./report-zh.md)

---

## What This Report Covers

This report investigates why multi-agent systems using the Claude Agent SDK consume 2–3% of the 5-hour credit quota per message, despite minimal message content. By reverse-engineering the SDK's 12MB minified source, the investigation reveals that the SDK spawns a new Node.js process on every call, which rebuilds the entire conversation history and completely misses the prompt cache. The report documents the root cause discovery process and the implemented solution.

## Who Should Read This

- **SDK users running multi-agent systems** — understand the true cost of each API call and why it's higher than expected
- **DevOps/Cost optimization engineers** — learn why persistent sessions dramatically reduce per-message costs
- **Anthropic SDK maintainers** — understand the performance implications of the current architecture

## What You'll Learn

- Why the Agent SDK spawns a new process on every call
- How the `cli.js` engine rebuilds conversation history on each invocation
- The impact of cache misses on overall credit consumption
- How persistent sessions reduce cost from 2–3% to below 1% per message
- The specific architectural changes needed for optimization

---

## 這篇報告在講什麼

這份報告調查為什麼使用 Claude Agent SDK 的多代理系統儘管訊息內容很少，卻消耗 5 小時額度配額的 2–3%。透過逆向工程 SDK 的 12MB 最小化程式碼，調查發現 SDK 在每次呼叫時都會生成新的 Node.js 程序，完整重建對話歷史記錄，完全錯過提示快取。報告記錄了根本原因的發現過程和實施的解決方案。

## 適合誰讀

- **運行多代理系統的 SDK 使用者** — 了解每次 API 呼叫的真實成本及其為何高於預期
- **DevOps/成本最佳化工程師** — 了解持久化會話如何將每訊息成本從 2–3% 降低到 1% 以下
- **Anthropic SDK 維護人員** — 了解當前架構的效能影響

## 你會學到什麼

- 為什麼 Agent SDK 在每次呼叫時都會生成新程序
- 如何在每次呼叫時重建 `cli.js` 引擎的對話歷史記錄
- 快取未命中對整體額度消耗的影響
- 持久化會話如何將成本從 2–3% 降低到每訊息 1% 以下
- 優化所需的特定架構變更
