# Prompt Cache 架構解析 — Claude Code 如何控制哪些內容被快取、快取多久

> **SDK 版本：** @anthropic-ai/claude-agent-sdk v0.2.76（cli.js build 2026-03-14）
> **日期：** 2026-03-27
> **狀態：** Draft

Claude Code 每次向 Anthropic API 發送 request 時，都會在背後決定哪些部分要標記為快取、要給多長的 TTL、哪些內容動態性太高不適合快取。這些決定不開放給使用者設定，全部在 `cli.js` 內部組裝完 request 之後才發出去。

這份報告透過追蹤 12MB minified 原始碼的呼叫鏈，完整還原 prompt cache 架構：從產生所有 `cache_control` 物件的單一 factory 函數，到每個 model 的快取啟停開關，再到伺服器端控制是否授予 1 小時延伸 TTL 的 feature flag。同時記錄系統提示如何被分割成靜態與動態區塊，message 層級的滑動視窗如何運作，以及為什麼底層的逐 byte 前綴比對機制，讓任何微小的順序差異都能摧毀一個原本正常運作的快取。

---

## 摘要

`cli.js` 透過幾個協調運作的機制管理 prompt caching：單一的 `cache_control` factory（`Ml()`）、model 專屬的啟停開關（`IGq()`）、伺服器端的 1h TTL 開關（`o3z()`）、將系統提示分成靜態與動態的邊界處理器（`_9z()`），以及只在最後一個 content block 標記的 message 層級斷點放置器（`z9z()`）。這幾個機制共同決定發給 API 的確切 bytes，以及哪些位置有資格被 cache read。

理解這個架構對任何以程式方式執行 SDK 的人都有實際意義：讓 CLI 互動模式如此高效的那些程式碼路徑，同時也解釋了為什麼某些 SDK 用法會持續產生 cache miss。本報告中的函數名稱和字串錨點在混淆版本間保持穩定，可以用來在未來版本中定位相同邏輯。

---

## 方法論

所有發現來自對 `cli.js` 的靜態分析，`cli.js` 提取自 npm 套件 `@anthropic-ai/claude-agent-sdk` v0.2.76 版。這個檔案是一個 12MB 的 minified JavaScript bundle，所有變數名稱都被混淆成 2-4 個字元的識別符。關鍵函數透過搜尋原始碼中嵌入的字串常量來定位——這些常量不會被混淆，在版本升級時也保持穩定。

使用的主要搜尋錨點：

| 字串常量 | 定位的函數 |
|---|---|
| `"DISABLE_PROMPT_CACHING"` | 快取啟停開關（`IGq()`） |
| `"tengu_prompt_cache_1h_config"` | 1h TTL feature flag 開關（`o3z()`） |
| `"__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__"` | 系統提示區域分割器（`_9z()` → `Jn8()`） |
| `"ephemeral"` | 快取控制 factory（`Ml()`） |
| `"skipCacheWrite"` | Message 層級斷點放置器（`z9z()`） |
| `"repl_main_thread"` | querySource 路由與快取編輯 beta 開關 |

本報告中的簡化程式碼片段保留了邏輯結構，但以描述性名稱取代了混淆名稱，不是原始碼的逐字複製。

---

## 發現

### 1. 快取控制 Factory：`Ml()`

`cli.js` 所有 API request 中每一個 `cache_control` 物件，都來自同一個 factory 函數，位置約在第 6370 行。這個函數永遠產生基礎物件 `{type: "ephemeral"}`，並根據條件額外加上兩個欄位：

```js
// 簡化版 — 實際原始碼使用混淆名稱
function Ml(scope, ttl) {
  const ctrl = { type: "ephemeral" };
  if (ttl === "1h") ctrl.ttl = "1h";
  if (scope) ctrl.scope = scope;
  return ctrl;
}
```

`type: "ephemeral"` 是硬編碼的。Claude Code 永遠不會產生 `{type: "persistent"}` 或任何其他變體。`ttl` 和 `scope` 欄位只在下面描述的個別開關函數滿足條件時才會出現。

這是整個 codebase 中所有快取斷點的唯一 factory。沒有任何程式碼路徑透過其他方式建立 `cache_control` 物件。

### 2. 快取啟停開關：`IGq()`

在將任何快取控制加入 request 之前，`IGq()` 會確認對目前使用的 model 而言快取是否啟用。它依序讀取以下環境變數：

1. `DISABLE_PROMPT_CACHING` — 對所有 model 停用快取
2. `DISABLE_PROMPT_CACHING_HAIKU` — 只在 model 名稱包含 `"haiku"` 時停用
3. `DISABLE_PROMPT_CACHING_SONNET` — 只在 model 名稱包含 `"sonnet"` 時停用
4. `DISABLE_PROMPT_CACHING_OPUS` — 只在 model 名稱包含 `"opus"` 時停用

如果快取被停用，`Ml()` 就不會被呼叫，payload 中也不會出現任何 `cache_control` 欄位。這是在 request 層級完全抑制快取標記的唯一機制。

每個 model 的環境變數使用子字串比對來對應 model 名稱字串。名為 `claude-sonnet-4-5` 的 model 會被 `DISABLE_PROMPT_CACHING_SONNET` 匹配。

### 3. 1h TTL 開關：`o3z()`

標準 Anthropic API 的 prompt cache 5 分鐘後過期。1 小時延伸 TTL 是一個獨立的等級，`cli.js` 透過 `o3z()` 來把關。這個函數評估兩條通往 1h TTL 的路徑：

**路徑 A — Bedrock 搭配環境變數：**
```
platform === "bedrock"
&& process.env.CLAUDE_CODE_ENABLE_1H_CACHE === "1"
```

**路徑 B — 第一方 + 未超額 + feature flag：**
```
isFirstParty
&& !isInOverage
&& tengu_prompt_cache_1h_config.allowlist 包含目前的 userId
```

`tengu_prompt_cache_1h_config` allowlist 從 feature flag 讀取（不是從 SDK 套件讀取）。它支援萬用字元比對：allowlist 條目如果以 `*` 結尾，就能匹配任何具有該前綴的 userId。這表示 1h TTL 是由伺服器控制的，使用者只能透過 Bedrock 環境變數路徑來影響它。

`o3z()` 回傳 false 時，`Ml()` 產生 `{type: "ephemeral"}`（5 分鐘 TTL）。回傳 true 時，產生 `{type: "ephemeral", ttl: "1h"}`。

`scope: "global"` 欄位是另外根據被標記的 content block 的 `cacheScope` 屬性加上的（見下方系統提示區域一節）。

### 4. querySource：34 個觀測到的值及其對快取的影響

每個 request 都有一個內部 `querySource` 欄位，用來識別是系統的哪個部分發起了這次請求。這個欄位以至少一種有據可查的方式影響快取行為，並作為其他功能的開關。34 個觀測到的值分成六個類別：

| 類別 | querySource 值 |
|---|---|
| 主要互動迴圈 | `repl_main_thread` |
| 子代理呼叫 | `agent:custom`、`sdk`、`hook_agent`、`verification_agent` 及約 10 個變體 |
| 工具相關 | `bash_extract_prefix`、`web_fetch_apply` 及類似值 |
| 記憶體操作 | `extract_memories`、`session_memory` |
| 壓縮 | `compact` |
| 側邊呼叫和分類器 | 各種值 |

`repl_main_thread` 獲得特殊待遇：它是唯一能啟用 `cache editing beta` 標頭的 `querySource`，這個功能允許實驗性的快取內容手術（能夠就地修改快取內容而不用使整個快取失效）。這個功能還附加了第一方限制——由 `querySource: "sdk"` 發起的 SDK request 不會收到它。

`sdk` 值被指派給 V2 persistent session 的 request。`sdk` 來源的 request 是否能取得 1h TTL，完全取決於 `o3z()` 的伺服器端 allowlist。

### 5. 系統提示快取區域：`_9z()` → `Jn8()`

系統提示不是以單一區塊發送的。它被拆分成多個 content block，每個 block 帶有一個 `cacheScope` 屬性，值為三種之一：`null`、`"org"` 或 `"global"`。

分割由嵌入在系統提示組裝中的邊界標記控制：`__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__`。在這個標記之前組裝的內容是靜態且共享的——包括基礎系統指令、工具描述，以及 session 間不會變動的組織層級情境。在這個標記之後組裝的內容是動態的——包括每個 session 專屬的情境，例如目前工作目錄、載入的 CLAUDE.md 內容，以及啟用的 skill 設定。

`Jn8()` 函數將 `cacheScope` 值對應到快取行為：

| cacheScope | 是否加 cache_control | scope 欄位 |
|---|---|---|
| `null` | 否 | — |
| `"org"` | 是，透過 `Ml()` | `"global"`（組織範圍） |
| `"global"` | 是，透過 `Ml()` | `"global"` |

動態邊界之前的 block 擁有非 null 的 scope，會加上 `cache_control`。邊界之後的 block 是動態的，`cacheScope: null`——不加快取標記。這個邊界確保靜態內容可以跨 session 快取，而動態內容每次都重新組裝。

這就是為什麼 request 中的系統提示部分，不論 V1 還是 V2 SDK 用法都能穩定命中快取：靜態區域有標記，動態區域沒有，而靜態區域很少改變。

```
[系統提示組裝]

  Block A: 基礎指令            cacheScope: "global"  → 加 cache_control
  Block B: 工具定義            cacheScope: "global"  → 加 cache_control
  Block C: 組織情境            cacheScope: "org"     → 加 cache_control
  ── __SYSTEM_PROMPT_DYNAMIC_BOUNDARY__ ──
  Block D: cwd、CLAUDE.md      cacheScope: null      → 不加 cache_control
  Block E: 啟用的 skills       cacheScope: null      → 不加 cache_control
```

![系統提示在動態邊界標記處分割成靜態與動態區域](./images/system-prompt-zones.webp)

### 6. Message 層級快取斷點：`z9z()` → `s3z()` / `t3z()`

系統提示之後，messages 陣列是 token 量最大的地方。`z9z()` 依照三條規則在陣列中放置快取斷點：

**規則 1 — 只有最後一則 message 會得到標記。** 函數遍歷組裝好的 messages 陣列，只在最後一則 message 的最後一個 content block 上放 `cache_control`。所有之前的 messages 依靠穩定前綴比對——如果前綴跟上一次 request 完全一致，那些位置不需要明確標記就能命中快取。

**規則 2 — `skipCacheWrite=true` 會移動斷點。** 當 request 以 `skipCacheWrite: true` 組裝時，斷點會移到倒數第二則 message，而不是最後一則。這用於預期回應是短暫的、不值得快取的情況——例如不會再被讀取的短暫驗證子呼叫。

**規則 3 — `thinking` block 永遠不會得到標記。** 型別為 `"thinking"` 或 `"redacted_thinking"` 的 content block，在 `s3z()` 和 `t3z()` 內部的明確型別檢查中被排除在快取標記放置之外。如果 message 中的最後一個 content block 是 thinking block，標記會移到同一則 message 中前一個非 thinking block 上。

```
messages: [
  { role: "user",      content: [...] },        ← 不加標記（穩定前綴）
  { role: "assistant", content: [...] },        ← 不加標記（穩定前綴）
  { role: "user",      content: [...] },        ← 不加標記（穩定前綴）
  { role: "assistant", content: [            ← 最後一則 message
      { type: "thinking", ... },              ← 跳過：不加標記
      { type: "text", ..., cache_control }    ← 最後一個非 thinking block：標記在這裡
  ]}
]
```

### 7. 前綴比對原則

Anthropic API 以逐 byte 前綴比對來做快取。一個 request 在位置 N 命中快取，必須且僅在從位置 0 到 N-1 的每個 byte 都與快取版本完全相同時才成立。沒有語義比對，沒有基於 hash 的去重，也沒有部分分數。

發給 API 的完整 request 結構如下：

```
[系統提示 block] [工具定義] [messages 陣列]
```

位置 N 的任何 byte 差異都會讓從 N 往後的所有快取失效。這表示：

- **注入順序很重要。** 如果兩個 system-reminder block 在不同輪次間交換了位置，整段 messages 就全部 miss。
- **工具定義順序很重要。** 新增或重排工具會讓工具部分的快取以及其後所有內容都失效。
- **內容穩定性很重要。** 動態注入的 memory block 中只要有一個 mtime 時間戳差異，就會讓整個 messages 陣列失效。

這就是為什麼逐一消除注入源（如報告 #1 所記錄）效果有限：即使大多數注入被移除，任何輪次間仍有變動的動態內容都會斷開前綴，整個 45k tokens 的 messages block 就要以 125% 費率重新寫入。

唯一可靠的解法是讓組裝器在各輪次間具有確定性——相同內容、相同順序、相同 bytes。V2 persistent session 能做到這一點，因為每次都由同一個 process 組裝 request。V1 每輪 spawn 的方式從頭組裝，無法保證 byte 層級的穩定性。

### 8. 快取費用結構

Anthropic API 以三種費率計費：

| 操作 | 費率 |
|---|---|
| Cache read（命中） | 基礎輸入價格的 10% |
| Cache write（未命中，新建項目） | 基礎輸入價格的 125% |
| 一般輸入（無快取標記） | 基礎輸入價格的 100% |

Cache miss 比一般 request 多付 25%。Cache hit 比一般便宜 90%。在 messages block（約 45k tokens）每輪都 miss 的對話中，那部分輸入的實際費用大約是基礎費率的 1.25 倍。同樣的 block 命中時，降到 0.10 倍。

效率公式：

```
cache_efficiency = cacheReadTokens / (inputTokens + cacheReadTokens + cacheCreationTokens)
```

任何 production SDK 部署都應該監控這個指標。持續低於 50% 表示部署設定有問題，或是存在類似報告 #1 所記錄的 V1 SDK 結構性快取斷裂。

---

## 證據

### `Ml()` factory — 錨點：`"ephemeral"`

在 `cli.js` 中搜尋 `"ephemeral"`，會找到一個將這個字串作為值使用的位置（不是在注釋或條件中）。那個位置就是 `Ml()` 函數體。這個函數從 codebase 的三個地方被呼叫：`Jn8()`（系統提示區域處理器）、`s3z()`（message 最後 block 標記器）、`t3z()`（`skipCacheWrite` 的倒數第二 block 標記器）。

### `IGq()` 開關 — 錨點：`"DISABLE_PROMPT_CACHING"`

字串 `"DISABLE_PROMPT_CACHING"` 作為 `process.env` key 存取出現四次，彼此相鄰，對應四個環境變數檢查（全局、haiku、sonnet、opus）。這四次讀取都在同一個開關函數內。

### `o3z()` 1h 開關 — 錨點：`"tengu_prompt_cache_1h_config"`

feature flag 名稱 `"tengu_prompt_cache_1h_config"` 只出現一次。周圍的程式碼讀取這個 flag 的 allowlist 欄位，並執行一個迴圈，檢查 userId 字串是否滿足比對條件。萬用字元邏輯使用 `endsWith("*")` 來偵測前綴模式。

### `_9z()` / `Jn8()` 邊界 — 錨點：`"__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__"`

邊界字串在系統提示組裝函數中作為字面量出現一次。程式碼在這個標記處分割組裝好的 block，並根據每個 block 落在哪一側來指派 `cacheScope` 值。

### `z9z()` message 斷點 — 錨點：`"skipCacheWrite"`

字串 `"skipCacheWrite"` 作為 request options 物件上的屬性存取出現。包含這段程式碼的函數反向遍歷 messages 陣列，並透過 `Ml()` 在找到的 block 上放 `cache_control`。thinking 型別的排除檢查在 `Ml()` 呼叫之前，以對 `"thinking"` 和 `"redacted_thinking"` 的字串比較形式清晰可見。

### `repl_main_thread` 快取編輯 beta — 錨點：`"repl_main_thread"`

字串 `"repl_main_thread"` 在一個條件式 block 中與 `querySource` 欄位進行比對。true 分支設定了對應 `cache editing beta` 功能的 header。同一個 block 中的第二個條件在套用 header 之前檢查第一方身份。

所有原始碼參考位置：`node_modules/@anthropic-ai/claude-agent-sdk/dist/cli.js`，快取相關函數大約在第 6340–6500 行。

---

## 影響

### 對 SDK 使用者（V1 `query()`）

V1 每次呼叫都建立新 process。系統提示的靜態區域能可靠地快取，因為它很少改變。Messages 陣列則不行——每次 spawn 都重組動態注入，無法保證各次呼叫間的 byte 層級一致性。實際結果是，每次 V1 `query()` 帶 `resume` 呼叫時，message block 的 cache write 都會發生，費率 125%。

1h TTL 開關（`o3z()`）由伺服器控制。`userId` 不在 `tengu_prompt_cache_1h_config` allowlist 中的 SDK 使用者只能得到 5 分鐘 TTL。對於每輪都 spawn 新 process 的 V1 用法，這個差別無關緊要——5 分鐘視窗從來不是瓶頸所在。

### 對 V2 Persistent Session 使用者

V2 保持 process 存活。組裝器在各輪次都在同一個 process 中執行，讓 byte 層級的輸出具有確定性。Messages 跨輪次在快取中持續累積。1h TTL 在這裡變得有意義：如果 session 閒置超過 5 分鐘，5 分鐘 TTL 的快取項目就會過期，下一輪需要重建。具有 1h TTL（透過第一方 allowlist 或 Bedrock 環境變數）的使用者，能在更長的閒置期後仍不需要付重建費用。

### 對 thinking block 的排除

`thinking` 和 `redacted_thinking` block 被明確排除在快取標記放置之外，帶來一個具體影響：如果 extended thinking 的回應是 assistant 輪次中的最後一個 content block，就不會有快取標記放在它上面。標記改為落在前一個文字 block 上。這表示 thinking 輸出本身永遠不會是快取項目的錨點——只有文字內容才是。

### 對工具定義排序

工具定義在 request 結構中位於系統提示 block 和 messages 陣列之間。任何工具定義的變動（新增工具、移除工具、重排工具）都會讓整個 messages 部分的快取失效。在根據情境動態載入 MCP server 或工具組的部署中，這可能產生看起來與 message 內容無關的快取 miss。

---

## 緩解方案 / Workaround

### SDK 部署中的穩定快取

根本要求是 messages block 在各輪次間具有 byte 層級的一致性。唯一能可靠實現這一點的方法是 V2 persistent session（同一個 process，同一個組裝器狀態）。為 V2 API 補齊完整選項支援的 patch 方式已在報告 #1 中記錄。

無法使用 V2 的部署，次佳方法是盡量減少動態注入源。報告 #1 的 JSONL sanitizer 技術可消除 file-modification diff 注入。設定 `CLAUDE_CODE_REMOTE=1` 可抑制 git status 注入。兩者都無法完全解決問題，但都能減少快取浪費的幅度。

### 監控快取效率

在每次 SDK 呼叫中加入 token 計數。記錄每個回應 usage 欄位中的 `cacheReadTokens`、`cacheCreationTokens` 和 `inputTokens`。計算：

```
efficiency = cacheRead / (input + cacheRead + cacheCreation)
```

低於 50%：結構性快取斷裂。高於 70%：健康狀態。輪次間的效率下滑（不只是第 1 輪）通常表示動態注入不穩定。

### 1h TTL 存取

沒有公開機制可以申請加入 `tengu_prompt_cache_1h_config` allowlist。Bedrock 路徑（`CLAUDE_CODE_ENABLE_1H_CACHE=1`）是使用者唯一可存取的開關。Bedrock 部署可透過該環境變數無條件啟用它。

### 工具定義穩定性

如果工具是動態載入的，在 session 開始時載入完整工具組，並在各輪次間保持穩定。避免在 session 中間新增或移除工具。工具定義位於 request 前綴中 messages block 之前——那裡的任何變動都會讓後面所有內容失效。

---

## 參考資料

- 報告 #1：[逆向 Claude Agent SDK：找出每則訊息 2-3% 額度消耗的根因與解法](../agent-sdk-cache-invalidation/report-zh.md)
- Anthropic prompt caching 文件：https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
- GitHub Issue #9769 — 請求讓 system-reminder 可個別開關：https://github.com/anthropics/claude-code/issues/9769
- GitHub Issue #16021 — file-modification 注入造成的 token 浪費：https://github.com/anthropics/claude-code/issues/16021
