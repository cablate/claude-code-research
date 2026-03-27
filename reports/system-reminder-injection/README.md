# Reverse-Engineering Claude Code's System-Reminder Injection Mechanism
# 逆向工程 Claude Code 的系統提醒注入機制

[English Version](./report-en.md) | [繁體中文版](./report-zh.md)

---

## What This Report Covers

Claude Code dynamically injects `<system-reminder>` content blocks into every API request that are invisible in the UI, not stored in conversation history, and actively hidden from the model. This report documents all 15+ injection categories, their triggers, and estimated token costs by reverse-engineering the `cli.js` source and analyzing session-level JSONL logs. It is a companion to Report #1, focusing on the injection mechanism itself rather than cost impact.

## Who Should Read This

- **Claude Code power users** — understand what hidden instructions are in your requests and why
- **Cost analysts** — learn what comprises the "hidden token tax" on every conversation turn
- **Security researchers** — understand how system prompts are managed and what risks that implies

## What You'll Learn

- The complete catalogue of 15+ injection types and their triggers
- How system-reminder blocks are assembled and injected invisibly
- Token cost breakdown per injection type
- Which operations (like Read calls) carry the highest hidden overhead
- Why some injections fire every turn while others are conditional

---

## 這篇報告在講什麼

Claude Code 動態地將 `<system-reminder>` 內容區塊注入每個 API 請求，這些區塊在 UI 中不可見、未儲存在對話歷史記錄中，並且被主動隱藏在模型中。此報告透過逆向工程 `cli.js` 原始碼並分析會話級 JSONL 日誌，記錄所有 15 以上的注入類別、觸發器和預估代幣成本。它是報告 #1 的配套報告，專注於注入機制本身而不是成本影響。

## 適合誰讀

- **Claude Code 進階使用者** — 了解您的請求中有哪些隱藏指令以及原因
- **成本分析師** — 了解每個對話轉折上「隱藏代幣稅」包含什麼
- **安全研究人員** — 了解系統提示如何管理及其隱含的風險

## 你會學到什麼

- 15 個以上注入類型及其觸發器的完整目錄
- 系統提醒區塊如何組裝和隱形注入
- 每個注入類型的代幣成本明細
- 哪些操作（如 Read 呼叫）具有最高隱藏開銷
- 為什麼某些注入在每個轉折上触发，而其他的是條件性的
