# Prompt Cache 架構解析 — Claude Code 如何控制哪些內容被快取、快取多久

> **SDK 版本：** @anthropic-ai/claude-agent-sdk v0.2.76（cli.js build 2026-03-14）
> **日期：** 2026-03-27

Anthropic 的 API 對 cache read 收取基礎價格的 10%，對 cache write 收取 125%。在一個長對話中，光 messages block 就有 45,000 tokens，每輪都命中和每輪都 miss 之間的差距，在那部分 request 上大約是 12 倍的成本。這是任何 Claude Code 部署中，影響單則訊息成本最大的單一因素。

Claude Code 有一套精密的內部系統控制快取斷點放在哪裡、活多久、哪些使用者能拿到高級 TTL 層級。這些全部沒有文件記載，也全部不開放設定。這份報告透過追蹤 12MB minified `cli.js` 原始碼，完整還原整個機制。

---

## 前綴比對規則 — 以及為什麼它讓後面所有東西都重要

Anthropic API 的快取是嚴格的逐 byte 前綴比對。一個 request 在位置 N 命中快取，前提是從位置 0 到 N-1 的每個 byte 都與快取版本完全相同。沒有語義比對，沒有 hash 去重，沒有部分分數。

發給 API 的完整 payload 結構如下：

```
[系統提示 block] → [工具定義] → [messages 陣列]
```

位置 N 的任何一個 byte 差異都會讓從 N 往後的所有快取失效。這對任何在 SDK 上開發的人有三個直接影響：

- **注入順序很重要。** 兩個 system-reminder block 在不同輪次間交換位置，整段 messages 就全部 miss。
- **工具定義順序很重要。** 新增或重排工具，工具區段和其後所有內容的快取全部失效。
- **內容穩定性很重要。** 動態注入的 memory block 裡只要有一個時間戳改了，整個 messages 陣列就失效。

這條前綴規則就是 Claude Code 快取架構存在的理由。下面描述的每一個機制，都是為了在需要穩定的地方保持 byte 穩定，並在正確位置做標記讓 API 知道去哪裡找快取邊界。

---

## 快取斷點如何被放置

### 唯一的 factory

每一個 API request 中的每一個 `cache_control` 物件，都來自同一個 factory 函數。它永遠產生基礎物件 `{type: "ephemeral"}`，並根據條件加上兩個欄位：

```js
// 從 minified 原始碼簡化
function createCacheControl(scope, ttl) {
  const ctrl = { type: "ephemeral" };
  if (ttl === "1h") ctrl.ttl = "1h";
  if (scope) ctrl.scope = scope;
  return ctrl;
}
```

`type` 是硬編碼的。Claude Code 永遠不會產生 `{type: "persistent"}` 或其他變體。這是整個 codebase 中快取斷點的唯一 factory — 沒有任何程式碼路徑用其他方式建立 `cache_control` 物件。在 minified 原始碼中搜尋字串 `"ephemeral"` 作為值使用（不是在註解中），就會直接找到這個函數（目前 build 中是 `Ml`）。

### 系統提示區域：靜態與動態的分割

系統提示不是以單一 block 發送的。它在組裝程式碼中以一個嵌入的邊界標記 `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` 分割成多個 content block。

邊界之前組裝的一切是靜態且共享的 — 基礎系統指令、工具描述、session 間不變的組織層級情境。邊界之後的一切是動態的 — 目前工作目錄、載入的 CLAUDE.md 內容、啟用的 skill 設定。靜態 block 帶有 `"global"` scope 的 `cache_control`。動態 block 什麼都不加。

```
  Block A: 基礎指令            → 加 cache_control（global scope）
  Block B: 工具定義            → 加 cache_control（global scope）
  Block C: 組織情境            → 加 cache_control（global scope）
  ── __SYSTEM_PROMPT_DYNAMIC_BOUNDARY__ ──
  Block D: cwd、CLAUDE.md      → 不加 cache_control
  Block E: 啟用的 skills       → 不加 cache_control
```

這個分割就是為什麼系統提示部分在 V1 和 V2 SDK 用法中都能穩定命中快取。靜態區域有標記，動態區域沒有，而靜態區域很少改變。邊界標記字串是在任何版本的 minified 原始碼中定位這段邏輯的可靠錨點。

### Message 層級的滑動視窗

系統提示和工具之後，messages 陣列是 token 量最大的地方。斷點放置遵循三條規則：

**只有最後一則 message 會得到標記。** 組裝器遍歷 messages 陣列，只在最後一則 message 的最後一個 content block 上放 `cache_control`。所有更早的 messages 依靠穩定前綴比對 — 如果前綴和上一次 request 完全一致，那些位置不需要明確標記就能命中快取。

**`skipCacheWrite` 會移動斷點。** 當 request 以 `skipCacheWrite: true` 組裝時，斷點移到倒數第二則 message。這用於短暫的子呼叫（短驗證查詢、分類器），回應不值得快取 — 燒掉一個永遠不會被讀取的 cache write 位只是浪費錢。

**Thinking block 永遠被跳過。** 型別為 `"thinking"` 或 `"redacted_thinking"` 的 content block 透過明確的型別檢查被排除在標記放置之外。如果 message 中最後一個 content block 是 thinking block，標記會落在同一則 message 中前一個非 thinking block 上。Thinking 輸出本身永遠不會是快取項目的錨點。

```
messages: [
  { role: "user",      content: [...] },          ← 不加標記（穩定前綴）
  { role: "assistant", content: [...] },          ← 不加標記（穩定前綴）
  { role: "user",      content: [...] },          ← 不加標記（穩定前綴）
  { role: "assistant", content: [
      { type: "thinking", ... },                  ← 跳過
      { type: "text", ..., cache_control: ... }   ← 標記放在這裡
  ]}
]
```

---

## TTL 如何決定 — 1 小時的關卡

標準 Anthropic API 的 prompt cache 5 分鐘後過期。另有一個 1 小時延伸 TTL 層級，`cli.js` 有一個關卡函數決定誰能拿到。決定分兩條路徑：

**路徑 A — Bedrock 搭配環境變數：**
```
platform === "bedrock"
AND process.env.CLAUDE_CODE_ENABLE_1H_CACHE === "1"
```

**路徑 B — 第一方使用者，未超額，在 allowlist 上：**
```
isFirstParty
AND NOT isInOverage
AND tengu_prompt_cache_1h_config.allowlist 包含 userId
```

`tengu_prompt_cache_1h_config` allowlist 是一個伺服器端的 feature flag，不是 SDK 套件的一部分。它支援萬用字元比對 — 以 `*` 結尾的條目會匹配任何具有該前綴的 userId。這表示 1 小時 TTL 是由伺服器控制的，使用者唯一能動的開關就是 Bedrock 環境變數。

關卡回傳 false 時，factory 產生 `{type: "ephemeral"}` — 5 分鐘 TTL。回傳 true 時，產生 `{type: "ephemeral", ttl: "1h"}`。

實際差別在於：一個 V2 persistent session 閒置了 6 分鐘，5 分鐘 TTL 的快取項目已經過期，下一輪就要付全額重建費用。有 1 小時 TTL 的使用者可以承受這段閒置期不受影響。對於每輪都 spawn 新 process 的 V1 用法，這個差別毫無意義 — 快取因為結構性原因斷裂，遠在任何 TTL 過期之前。

關卡函數可以透過搜尋字串 `"tengu_prompt_cache_1h_config"` 在 minified 原始碼中定位（目前 build 中是 `o3z`）。

---

## 每個 Model 的停用開關

在將任何 cache control 加入 request 之前，有另一個關卡檢查目前使用的 model 是否啟用快取。它讀取四個環境變數：

| 環境變數 | 效果 |
|---|---|
| `DISABLE_PROMPT_CACHING` | 對所有 model 停用快取 |
| `DISABLE_PROMPT_CACHING_HAIKU` | model 名稱包含 `"haiku"` 時停用 |
| `DISABLE_PROMPT_CACHING_SONNET` | model 名稱包含 `"sonnet"` 時停用 |
| `DISABLE_PROMPT_CACHING_OPUS` | model 名稱包含 `"opus"` 時停用 |

如果任何匹配的變數被設定，cache control factory 就不會被呼叫，payload 中不會出現任何 `cache_control` 欄位。匹配使用子字串查找 — `claude-sonnet-4-5` 會被 `DISABLE_PROMPT_CACHING_SONNET` 捕捉到。

這是在 request 層級完全抑制快取標記的唯一機制。關卡函數可以透過搜尋 `"DISABLE_PROMPT_CACHING"` 在原始碼中找到（目前 build 中是 `IGq`）。

---

## 34 個 querySource 值 — 以及誰獲得特殊待遇

每個 request 攜帶一個內部 `querySource` 欄位，識別是系統的哪個部分發起了這次請求。已觀測到 34 個不同的值，分成六個類別：

| 類別 | 範例值 |
|---|---|
| 主要互動迴圈 | `repl_main_thread` |
| 子代理呼叫 | `agent:custom`、`sdk`、`hook_agent`、`verification_agent`、約 10 個以上 |
| 工具相關 | `bash_extract_prefix`、`web_fetch_apply` 及類似值 |
| 記憶體操作 | `extract_memories`、`session_memory` |
| 壓縮 | `compact` |
| 側邊呼叫和分類器 | 各種值 |

主要互動迴圈的來源（`repl_main_thread`）獲得特殊待遇：它是唯一能啟用 `cache editing beta` HTTP header 的值，這個功能允許實驗性的快取內容手術 — 就地修改快取內容而不使整個快取失效。這個功能還額外限定為第一方才有。從 SDK 路徑發起的 request（`querySource: "sdk"`）不會收到它。

`sdk` 值被指派給 V2 persistent session 的 request。這些 request 是否能取得 1 小時 TTL，完全取決於 TTL 關卡中的伺服器端 allowlist，與 querySource 本身無關。

---

## 對 SDK 使用者的意義

### V1 `query()` — 每次 resume 輪次都有結構性 cache miss

V1 每次呼叫都建立新 process。系統提示的靜態區域能可靠地快取，因為它很少改變。Messages 陣列則不行 — 每次 spawn 都從頭重組動態注入，無法保證 byte 層級的一致性。結果是 message block（約 45k tokens）在每次帶 `resume` 的 V1 呼叫中，都以 125% 費率重新寫入。

1 小時 TTL 在這裡毫無意義。快取因為結構性原因斷裂，不是因為過期。

### V2 persistent session — 逐步累積的快取命中

V2 保持 process 存活。組裝器在各輪次都在同一個 process 中執行，讓 byte 層級的輸出具有確定性。Messages 跨輪次在快取中持續累積。1 小時 TTL 在這裡變得有意義：5 分鐘 TTL 的閒置 session 超過 5 分鐘就會失去快取，而 1 小時使用者可以承受更長的空檔。

### 工具定義排序

工具定義在 request 前綴中位於系統提示和 messages 陣列之間。任何工具定義的變動 — 新增、移除或重排 — 都會讓整個 messages 部分的快取失效。動態載入 MCP server 或工具組的部署，可能產生看起來與 message 內容無關的快取 miss。

解法很直接：在 session 開始時載入完整工具組，各輪次間保持穩定。不要在 session 中間新增或移除工具。

### 監控

在每次 SDK 呼叫中加入 token 計數。記錄每個回應 usage 欄位中的 `cacheReadTokens`、`cacheCreationTokens` 和 `inputTokens`。計算：

```
efficiency = cacheRead / (input + cacheRead + cacheCreation)
```

低於 50%：結構性快取斷裂 — 有東西在輪次間不該變卻變了。高於 70%：健康。輪次間的效率下滑（不只是第 1 輪）通常指向動態注入不穩定。

---

## 方法論

所有發現來自對 `cli.js` 的靜態分析，提取自 npm 套件 `@anthropic-ai/claude-agent-sdk` v0.2.76 版。檔案是一個 12MB 的 minified JavaScript bundle，所有變數名稱被混淆成 2-4 個字元的識別符。關鍵函數透過搜尋字串常量來定位 — 這些常量不會被混淆，在版本升級時保持穩定。

使用的主要錨點及其定位目標：

| 字串常量 | 定位目標 |
|---|---|
| `"ephemeral"` | 快取控制 factory |
| `"DISABLE_PROMPT_CACHING"` | 每 model 啟停開關 |
| `"tengu_prompt_cache_1h_config"` | 1 小時 TTL feature flag 關卡 |
| `"__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__"` | 系統提示區域分割器 |
| `"skipCacheWrite"` | Message 層級斷點放置器 |
| `"repl_main_thread"` | querySource 路由與快取編輯 beta 關卡 |

本報告中的程式碼片段保留邏輯結構但使用描述性名稱，不是 minified 原始碼的逐字複製。所有原始碼參考位置：`node_modules/@anthropic-ai/claude-agent-sdk/dist/cli.js`，快取相關函數大約在第 6340-6500 行。

---

## 參考資料

- 報告 #1：[逆向 Claude Agent SDK：找出每則訊息 2-3% 額度消耗的根因與解法](../agent-sdk-cache-invalidation/report-zh.md)
- Anthropic prompt caching 文件：https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
- GitHub Issue #9769 — 請求讓 system-reminder 可個別開關：https://github.com/anthropics/claude-code/issues/9769
- GitHub Issue #16021 — file-modification 注入造成的 token 浪費：https://github.com/anthropics/claude-code/issues/16021
