# Seven Patches That Push Cache Efficiency Past 90%
# 將快取效率推進 90% 以上的七個修補程式

[English Version](./report-en.md) | [繁體中文版](./report-zh.md)

---

## What This Report Covers

Persistent sessions improved cache efficiency from stuck at 25% to roughly 84%, but 16 percentage points remain. This report documents seven optimizations that close most of the gap: three patches to `cli.js`, two changes in wrapper code, one monitoring system, and one bonus optimization. Each targets a specific mechanism traced in Reports #3–5. The report includes the patching methodology using anchor strings to survive minification across SDK releases.

## Who Should Read This

- **Production system operators** — implement these seven optimizations to reduce your actual per-message costs
- **SDK wrapper developers** — learn which patches to apply in wrapper code versus the SDK binary
- **Cache optimization practitioners** — understand the anchor-string patching technique for durable modifications

## What You'll Learn

- The three `cli.js` patches and how to apply them using anchor strings
- Two critical wrapper-level changes for cost reduction
- How to build a monitoring system to detect cache failures
- The bonus optimization that combines two other improvements
- Why anchor-string patching is more durable than line-number or variable-name patching

---

## 這篇報告在講什麼

持久化會話將快取效率從卡住的 25% 改進到大約 84%，但仍有 16 個百分點的空間。此報告記錄了七項優化，可以縮小大部分差距：三個 `cli.js` 補丁、兩個包裝程式代碼變更、一個監視系統，以及一個額外的優化。每個都針對報告 #3–5 中追蹤的特定機制。報告包括使用錨字符串進行修補的方法，以在 SDK 發行版中倖存最小化。

## 適合誰讀

- **生產系統運營人員** — 實施這七項優化以降低您的實際每訊息成本
- **SDK 包裝程式開發者** — 了解哪些補丁在包裝程式代碼中應用，哪些在 SDK 二進制中應用
- **快取優化從業者** — 了解用於耐用修改的錨字符串修補技術

## 你會學到什麼

- 三個 `cli.js` 補丁以及如何使用錨字符串應用它們
- 兩個用於成本降低的關鍵包裝程式級變更
- 如何構建監視系統以檢測快取失敗
- 結合其他兩項改進的額外優化
- 為什麼錨字符串修補比行號或變數名稱修補更耐用
