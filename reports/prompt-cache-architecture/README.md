# Prompt Cache Architecture — How Claude Code Controls What Gets Cached and for How Long
# 提示快取架構 — Claude Code 如何控制緩存內容及保留時長

[English Version](./report-en.md) | [繁體中文版](./report-zh.md)

---

## What This Report Covers

Anthropic's prompt cache charges 10% of base cost for reads and 125% for writes. In conversations with 45,000-token message blocks, the difference between hitting and missing cache is roughly 12x in cost. This report reverse-engineers Claude Code's undocumented cache control system, tracing how it decides where cache breakpoints go, how long they live, and how different users receive different TTL tiers. None of this is documented or configurable.

## Who Should Read This

- **Production deployment engineers** — understand the cost leverage points in your Claude Code infrastructure
- **Large-context application builders** — learn how to architect prompts to maximize cache efficiency
- **API cost forecasters** — understand what drives the actual cache behavior in your deployments

## What You'll Learn

- The Anthropic cache's strict byte-for-byte prefix matching requirement
- How Claude Code structures requests to optimize cache breakpoints
- The five internal thresholds that govern cache decisions
- What "premium TTL tier" means and who gets it
- How to estimate cache hit rates for your specific usage patterns

---

## 這篇報告在講什麼

Anthropic 的提示快取對讀取收費為基礎成本的 10%，寫入為 125%。在具有 45,000 代幣訊息區塊的對話中，命中和未命中快取之間的成本差異約為 12 倍。此報告逆向工程 Claude Code 未文件化的快取控制系統，追蹤它如何決定快取斷點的位置、保留時長，以及不同使用者接收不同 TTL 層級的方式。這些都沒有被記錄或可配置。

## 適合誰讀

- **生產環境部署工程師** — 了解您的 Claude Code 基礎設施中的成本槓桿點
- **大型上下文應用程式構建者** — 了解如何架構提示以最大化快取效率
- **API 成本預測員** — 了解在您的部署中驅動實際快取行為的因素

## 你會學到什麼

- Anthropic 快取的嚴格位元組逐位元組前綴匹配要求
- Claude Code 如何構造請求以優化快取斷點
- 管理快取決策的五個內部閾值
- 「高級 TTL 層」的含義及誰會獲得它
- 如何針對您的特定使用模式估計快取命中率
