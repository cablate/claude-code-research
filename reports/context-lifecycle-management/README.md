# Context Lifecycle Management — How Claude Code Decides When to Compress, What to Keep, and What It Costs
# 上下文生命週期管理 — Claude Code 如何決定何時壓縮、保留什麼及其成本

[English Version](./report-en.md) | [繁體中文版](./report-zh.md)

---

## What This Report Covers

Long conversations force Claude Code to run autocompact — an undocumented automatic compression system that replaces most of the conversation with a summary when context grows too large. The compression destroys the prompt cache, making the next message cost 125% normal. Under certain conditions, compression triggers another compression, wasting even more cost. This report reverse-engineers the complete lifecycle: context size calculation, the five thresholds governing decisions, when compression fires, what survives, and two previously undocumented cost-inflation mechanisms.

## Who Should Read This

- **Long-conversation users** — understand when and why compression happens to your context
- **Cost forecasters** — learn the hidden costs of context management and compression cascades
- **SDK maintainers** — understand the five thresholds and their interaction patterns

## What You'll Learn

- How context size is calculated per model (1M vs 200k window detection)
- The five thresholds that trigger autocompact, reserve windows, and summary sizing
- What content survives compression and what gets discarded
- How compression cascades multiply costs
- Two undocumented mechanisms that inflate context size beyond raw token counts

---

## 這篇報告在講什麼

長對話會強制 Claude Code 運行 autocompact — 一個未文件化的自動壓縮系統，當上下文增長過大時，它會用摘要替換大部分對話。壓縮會摧毀提示快取，使下一條訊息的成本為正常成本的 125%。在某些情況下，壓縮會觸發另一次壓縮，浪費更多成本。此報告逆向工程完整的生命週期：上下文大小計算、管理決策的五個閾值、何時壓縮觸發、什麼被保留，以及兩個之前未文件化的成本膨脹機制。

## 適合誰讀

- **長對話使用者** — 了解何時以及為什麼壓縮發生在您的上下文中
- **成本預測員** — 了解上下文管理和壓縮級聯的隱藏成本
- **SDK 維護人員** — 了解五個閾值及其相互作用模式

## 你會學到什麼

- 如何按模型計算上下文大小（1M 對 200k 窗口檢測）
- 觸發 autocompact、保留窗口和摘要大小調整的五個閾值
- 壓縮中倖存的內容和被丟棄的內容
- 壓縮級聯如何乘以成本
- 兩個未文件化的機制，使上下文大小超過原始代幣計數
