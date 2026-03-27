# Tool Serialization and Cache Stability — Why Your MCP Tools Might Be Silently Busting the Prompt Cache
# 工具序列化與快取穩定性 — 為什麼您的 MCP 工具可能無聲地破壞提示快取

[English Version](./report-en.md) | [繁體中文版](./report-zh.md)

---

## What This Report Covers

The `tools` array is part of every API request's cache prefix. A single-byte change in tool serialization triggers a full cache rewrite at 125% cost. Claude Code never sorts its tools, making stability dependent entirely on insertion order. The deferred tool loading mechanism silently changes the tools array mid-conversation, guaranteeing invisible cache misses. This report traces the complete pipeline from tool list construction to API serialization and identifies every instability point.

## Who Should Read This

- **MCP integration developers** — understand why your tools might be causing hidden cache costs
- **SDK wrapper builders** — learn what tool ordering and serialization patterns are safe
- **Production cache monitors** — understand what to watch for to catch tool-induced cache busts

## What You'll Learn

- How the tool list is built across four stages (built-in, static, deferred, plugin)
- Why tool ordering matters and what determines insertion order
- How the deferred loading mechanism silently changes tools mid-conversation
- The exact serialization format and where byte-level instability enters
- Practical patterns to ensure tool array stability across turns

---

## 這篇報告在講什麼

`tools` 陣列是每個 API 請求快取前綴的一部分。工具序列化中的單一位元組變更會觸發以 125% 成本進行的完整快取重寫。Claude Code 從不對工具進行排序，使穩定性完全依賴於插入順序。延遲的工具加載機制在對話中途無聲地改變工具陣列，保證了隱形快取未命中。此報告追蹤從工具列表構造到 API 序列化的完整管道，並識別每個不穩定點。

## 適合誰讀

- **MCP 整合開發者** — 了解為什麼您的工具可能導致隱藏的快取成本
- **SDK 包裝程式建構者** — 了解哪些工具順序和序列化模式是安全的
- **生產快取監視器** — 了解要監看什麼以捕捉工具引起的快取中斷

## 你會學到什麼

- 工具列表如何跨四個階段構建（內置、靜態、延遲、插件）
- 為什麼工具順序很重要以及什麼決定插入順序
- 延遲加載機制如何無聲地改變對話中途的工具
- 確切的序列化格式以及位元組級不穩定性進入的位置
- 確保工具陣列在轉折間穩定的實用模式
