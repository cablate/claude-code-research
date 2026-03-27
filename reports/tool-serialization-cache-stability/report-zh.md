# 工具序列化與快取穩定性 — 為什麼你的 MCP 工具可能正在悄悄破壞 Prompt Cache

`tools` 陣列是每個 API 請求快取前綴的一部分。如果序列化後的工具在對話輪次之間有任何改變 — 哪怕一個 byte — Anthropic API 就會把整個前綴視為全新的，以 125% 的費率重新寫入快取。System prompt、messages、分歧點之後的所有內容：全部失效。

Claude Code 從不排序它的工具。穩定性完全取決於插入一致性。而延遲工具載入機制會在對話中途悄悄改變 tools 陣列，以開發者完全看不見的方式保證快取 miss。

這份報告追蹤了從工具列表建構到 API 序列化的完整 pipeline，找出每一個不穩定性可能進入的環節，並說明 SDK 使用者能做些什麼。

分析基於逆向工程 `cli.js` build `2026-03-14`，打包於 `@anthropic-ai/claude-agent-sdk` v0.2.76。

---

## 工具列表如何被建構

Claude Code 發出的每個 API 請求，都要經過四個階段來組裝 `tools` 陣列。理解這些階段，才能理解快取不穩定性從哪裡進入。

### 第一階段：內建工具登錄表

第一階段是一個硬編碼的字面陣列，包含所有內建工具 — Read、Write、Edit、Bash、Grep、Glob、WebFetch、TodoWrite 等。這個陣列在 bundle 編譯時就已決定。工具列表建構函數（source 中為 `ng`）每次呼叫都回傳同一個陣列、同一個順序。這是整個 pipeline 中唯一從構造上就完全確定性的階段。

### 第二階段：允許 / 拒絕過濾

第二階段取得內建陣列後，使用 `allowedTools` 和 `denyRules` 設定執行 JavaScript 的 `.filter()`（source 中為 `FX`）。因為 `.filter()` 保留插入順序，存活下來的條目相對排序與第一階段完全相同。

一個細節：如果你傳入 `allowedTools: ["Bash", "Read", "Write"]`，結果列表會按照這些工具在第一階段的順序排列（Bash 在 Read 前面，Read 在 Write 前面，依據編譯時位置），而不是你指定的順序。`allowedTools` 陣列是成員檢測，不是排序指令。

### 第三階段：合併內建與 MCP 工具

合併步驟（source 中為 `u66`）將內建工具和 MCP 工具串接：

```javascript
// 內建工具永遠在前，MCP 工具附加在後
[...builtins, ...mcpTools]
```

然後按名稱去重，保留第一次出現的。內建工具永遠排在 MCP 工具前面。如果 MCP 工具與內建工具同名，內建工具優先。

### 第四階段：序列化為傳輸格式

序列化器（source 中為 `Sh1`）將每個工具轉換成 API 需要的格式：

```javascript
{
  name: tool.name,
  description: await tool.prompt(),  // 非同步 — 這很重要
  input_schema: tool.jsonSchema ?? deriveFromZod(tool.inputSchema)
}
```

有兩個細節對快取穩定性很重要：

**Description 是非同步解析的。** 內建工具回傳靜態字串 — 永遠是同樣的內容。MCP 工具可能呼叫 server 取得動態 description。Server 回應中的任何非確定性都會直接傳播到序列化後的 payload。

**Schema 來源依工具類型不同。** MCP 工具使用 MCP server 在註冊時提供的 JSON schema。內建工具通過一個有記憶化的轉換器從 Zod 定義推導 schema — 在進程生命週期內穩定。

`cache_control` 標記不會在個別工具層級套用。工具不會被單獨快取；只有組裝好的 system prompt 才會有快取標記。

---

## 關鍵發現：整個路徑都沒有排序

我在整個 `cli.js` source 中搜尋工具相關程式碼附近的 `.sort()` 呼叫。找到的每一個 `.sort()` 都屬於不相關的功能 — worktree 路徑排序、insights 資料、help 選單顯示、compact metadata 輸出。

工具 pipeline 在任何階段都沒有 `.sort()`。工具順序從頭到尾嚴格依照插入順序。

這意味著快取前綴取決於一個沒有人明確控制的東西。對內建工具來說沒問題 — 編譯時陣列每次執行都一樣。對 MCP 工具來說，這表示順序取決於工具是如何、何時被註冊的，而這就是脆弱之處。

---

## MCP 工具排序：通常穩定，偶爾不是

MCP 工具通過載入器（source 中為 `Fr6`）進入 pipeline，它會並行初始化所有已註冊的 MCP server。回應以非確定性的順序抵達 — 但 session 中儲存的工具列表是依啟動時的 client 註冊順序建構的，而非回應抵達順序。所以對於從磁碟讀取的設定檔，註冊順序在重啟之間是一致的。

脆弱性出現在兩種場景：

**程式化設定建構。** 如果你的程式碼是透過迭代一個 key 順序不保證的資料結構來建構 `mcpServers` 物件，註冊順序在不同執行之間就會變化。兩個名義上相同但 key 插入順序不同的設定，會產生不同的 MCP 工具陣列。

**MCP server 重新連線。** 如果 server 斷線後恢復，工具列表會重新註冊，可能在合併後的陣列中出現在不同位置。

兩種場景都會產生結構上不同的 `tools` 陣列。Anthropic API 將整個 `tools` 區塊視為快取前綴的一部分。不同的陣列 = 不同的前綴 = 完整的快取 miss。

---

## 延遲工具載入：隱藏的快取破壞者

這是整份報告中影響最大的發現。

Claude Code 預設會延遲所有 MCP 工具。延遲載入控制器（source 中為 `GX`）的規則很簡單：每個 `isMcp === true` 的工具都會被延遲。實際上是這樣運作的：

Session 開始時，MCP 工具根本不會被包含在 `tools` 陣列中。取而代之的是，只有它們的名稱被列在 system prompt 中的一個文字區塊裡：`<available-deferred-tools>mcp__server__toolname, ...</available-deferred-tools>`。模型能看到名稱、知道可以要求使用這些工具，但完整 schema 不在請求中。

當 Claude 決定它需要一個延遲工具時，會執行一個搜尋步驟來載入完整 schema。此時 `tools` 陣列獲得一個帶有完整 `{name, description, input_schema}` 物件的新條目。

**在對話中首次調用任何 MCP 工具，都是一次保證的快取 miss。** 呼叫前的 `tools` 陣列與呼叫後結構不同。快取前綴已經改變。這是繞不過去的。

### 內部機制：Discovery Scanner

我們追蹤到了確切的機制。每一輪對話中，主查詢函數（source 中的 `mGq`）都會從頭重建 `tools` 陣列。決定哪些延遲工具要被包含的邏輯，是由一個 discovery scanner（source 中的 `zF`）驅動的，它會掃描完整的訊息歷史來尋找之前已載入的工具：

```javascript
// 從 mGq 簡化 — 主查詢產生器
// 每一輪都會執行，不只是第一輪

let discoveredTools = scanMessageHistory(messages);  // zF(A)
let J;
if (dynamicToolLoading) {
  J = allTools.filter(tool => {
    if (!isDeferred(tool)) return true;      // 內建工具：永遠包含
    if (isToolSearch(tool)) return true;      // tool-search 本身：永遠包含
    return discoveredTools.has(tool.name);    // MCP 工具：只在被發現時包含
  });
}
```

這意味著 `tools` 陣列不是固定的 — 它會在對話過程中逐漸增長。每當 Claude 使用 tool-search 載入一個延遲的 MCP 工具，該工具的名稱就進入了訊息歷史。在下一輪中，scanner 找到它，filter 就把它包含進來。序列化後的 `tools` 陣列因此多了一個條目。

**成本隨對話長度複合增長。** 每次快取 miss 都會強制重建完整的快取，涵蓋 system prompt、所有工具定義、以及截至該時點的完整訊息歷史。在第 10 輪載入的工具造成的 miss，重寫的 token 量遠大於在第 2 輪載入的。

一個多輪 session 看起來是這樣：

| 輪次 | 發生了什麼 | tools 陣列 | 快取 | 重建規模 |
|-----|----------|-----------|-----|---------|
| 1 | 沒使用 MCP 工具 | 只有內建工具 | Hit | — |
| 2 | Claude 用 tool-search 載入 `mcp__files__read` | 與第 1 輪相同（tool-search 結果是訊息，不是工具條目） | Hit | — |
| 3 | Claude 呼叫 `mcp__files__read` | Discovery scanner 在歷史中找到它 → 陣列 +1 完整 schema | **Miss** | ~20k tokens |
| 4 | 再次使用 `mcp__files__read` | 與第 3 輪相同 | Hit | — |
| 5 | Claude 用 tool-search 載入 `mcp__search__query` | 與第 3 輪相同 | Hit | — |
| 6 | Claude 呼叫 `mcp__search__query` | Scanner 找到它 → 陣列再 +1 schema | **Miss** | ~40k tokens |
| 7 | Claude 用 tool-search 載入 `mcp__browser__click` | 與第 6 輪相同 | Hit | — |
| 8 | Claude 呼叫 `mcp__browser__click` | Scanner 找到它 → 陣列再 +1 schema | **Miss** | ~60k tokens |
| 9+ | 重複使用相同工具 | 穩定 | Hit | — |

每個首次被調用的 MCP 工具都會花費一次保證的快取 miss。Miss 不是在工具透過 tool-search 載入時發生 — 而是在**下一輪**，當 discovery scanner 重建工具陣列並包含新發現的工具時發生。

**成本會複合增長。** 第 8 輪的快取 miss 重建 ~60k tokens（所有先前的訊息），而第 3 輪的 miss 只重建 ~20k。在八輪對話中使用三個 MCP 工具，累計成本大約是單次 close-and-resume 的三倍。

還有一個 feature flag（source 中為 `tengu_defer_all`）會將延遲擴展到所有工具，包括內建工具。啟用時，初始工具陣列幾乎是空的，每次工具使用都會觸發延遲載入。這似乎用於特定的內部場景。

---

## 工具 Description：內建安全，MCP 危險

序列化器透過呼叫一個非同步方法來解析每個工具的 description。對內建工具來說，這會回傳一個靜態字串 — 硬編碼在 bundle 中，每次都一樣。內建 description 對快取是安全的。

對 MCP 工具來說，description 來自 MCP server。如果 server 動態生成 description — 包含時間戳、記錄數量、環境變數、任何在不同呼叫之間會變化的東西 — 序列化後的 description 在不同輪次之間就會改變。即使任何工具 description 中有一個字元的差異，也會讓整個 `tools` 區塊在快取前綴中失效。

這特別隱蔽，因為工具本身可能運作完全相同。搜尋結果一樣、行為一樣，但 description 說「在 1,847 個文件中搜尋」而不是「在 1,846 個文件中搜尋」，整個快取就沒了。

---

## disallowedTools 和 allowedTools：保留順序的過濾器

禁用工具過濾器（source 中為 `_c`）將 `disallowedTools` 設定物化為 Set，然後用 `.filter()` 移除匹配的條目。順序被保留。過濾在 agent 或 subagent 層級套用，來源於 agent 定義的 frontmatter。

從 SDK 側，`disallowedTools` 以 `--disallowedTools ToolA,ToolB` CLI 參數傳遞。這個階段也沒有任何排序。

`allowedTools` 和 `disallowedTools` 本身都不會引入排序不穩定性 — 它們保留過濾前的既有順序。風險只在於這些過濾器的輸入本身就不穩定的情況。

---

## SDK 使用者能做什麼

### 在傳遞給 SDK 之前排序 `allowedTools`

```javascript
// 不穩定 — 取決於 getEnabledTools() 怎麼建構列表
const options = { allowedTools: getEnabledTools() };

// 穩定 — 字母順序，永遠一致
const options = { allowedTools: getEnabledTools().sort() };
```

一行程式碼。消除一個排序不穩定的來源。

### 排序 `mcpServers` 的 key

```javascript
// 確保無論設定怎麼建構，key 順序都一致
const mcpServers = Object.fromEntries(
  Object.entries(buildMcpConfig()).sort(([a], [b]) => a.localeCompare(b))
);
```

### 讓 MCP 工具 description 完全靜態

如果你控制 MCP server，就把 description 硬編碼。不要有時間戳、資料庫查詢或任何執行時狀態。

```python
# 動態 description — 快取殺手
@mcp.tool()
def search(query: str) -> str:
    """在 {len(self.documents)} 個文件中搜尋。"""
    ...

# 靜態 description — 快取安全
@mcp.tool()
def search(query: str) -> str:
    """在文件索引中搜尋。"""
    ...
```

### 在 session 開始時預熱 MCP 工具

如果你知道 session 會用到哪些 MCP 工具，提前觸發它們。在 session 初始化時用一個拋棄式 prompt 強制延遲載入，在快取敏感的輪次之前完成：

```javascript
// 強制所有 MCP 工具預先載入
await session.query("list the available mcp tools", { maxTurns: 1 });
// 後續輪次受益於穩定的 tools 陣列
```

這樣做把保證的快取 miss 集中到 session 開頭，而不是散落在正式工作的輪次中。

### 考慮完全停用延遲載入

如果你的 session 使用的是已知、固定的 MCP 工具集，停用延遲載入意味著所有 schema 都預先序列化。初始請求較大，但從第一輪起 `tools` 陣列就穩定。對話中途不會有意外。

目前沒有公開的 API 選項可以做到這件事 — 需要打補丁或使用 feature flag。

### 監控跨輪次的 tools 陣列

對 session 的前幾輪記錄序列化後的 tools 陣列。如果第 2、3、4 輪出現新條目，那些就是延遲載入造成的快取 miss。在決定緩解策略之前，先量化這個模式。

---

## 真實世界的成本

使用 3 個 MCP server（每個 4-6 個工具）的系統：

- Session 開始時有 15-18 個延遲工具
- 每個首次被調用的 MCP 工具都會在**下一輪**觸發一次保證的快取 miss
- 在所有需要的工具都至少被使用過一次之前，tools 陣列不會穩定
- 每次 miss 的成本隨對話長度增長 — 後期的 miss 比早期的更昂貴
- 以 Opus 4 的費率（cache write $37.50/M），在一段逐漸增長的對話中連續 5 次 miss 的累計成本，可能超過 5 次獨立的 close-and-resume

只使用內建工具的系統：

- 工具列表是編譯時穩定的
- 不會發生延遲載入
- 從第一輪起快取穩定性幾乎是有保證的

這兩種場景之間的差距，就是 MCP 工具整合中沒有人提到的隱形成本。

---

## Skill 呢？一個常見的誤解

一個自然的問題：如果 MCP 工具載入會破壞快取，那載入一個 Skill（斜線指令）是否會造成同樣的問題？

不會。兩者的機制完全不同。

當一個 Skill 被調用時，Claude Code 做兩件事：Skill tool 將 skill 內容作為普通的 `tool_result` 回傳到當前輪次的位置，同時建立一個 `invoked_skills` attachment 記錄已載入的 skill（用於 session resume 時恢復）。這兩者都是**追加到 messages 陣列的尾部** — Skill 內容在對話的當前位置進入，不會被回溯插入到 prefix 的較前面位置。

驅動延遲工具載入機制的 discovery scanner（source 中的 `zF`）只搜尋 `tool_reference` blocks — 這是一種只有 ToolSearch 工具才會產生的特殊 block 類型。Skill 調用不會產生 `tool_reference` blocks。`tools` 陣列完全不受 Skill 載入影響。

| | MCP 工具（透過 ToolSearch） | Skill（透過 Skill tool） |
|---|---|---|
| 改變了什麼 | `tools` 陣列（prefix 前段） | Messages 陣列（追加到當前輪次） |
| 對先前輪次的快取影響 | **全部失效** — tools prefix 偏移，之後的所有內容都 miss | **無** — 新內容在尾部，先前的 prefix 不變 |
| 機制 | `tool_reference` block → discovery scanner → tools 陣列重建 | `tool_result` + `invoked_skills` attachment → messages 追加 |
| 首次使用的成本 | 完整快取重建（整個對話的 125%） | 接近零（只有新增內容是 cache write） |

如果你在選擇將功能實作為 MCP 工具還是 Skill，而快取效率是考量因素，這個區別值得注意。Skill 對快取是中性的；MCP 工具在每個對話中每個首次使用的工具都會帶來一次性的快取重建代價。

---

## 參考資料

- [逆向工程 Claude Agent SDK：每條訊息消耗 2-3% Credit 的根本原因與修復](../agent-sdk-cache-invalidation/README.md) — 涵蓋 prompt cache 的運作方式以及快取 miss 的代價
- SDK 版本：`@anthropic-ai/claude-agent-sdk` v0.2.76，`cli.js` build `2026-03-14`
- Anthropic API prompt caching：快取讀取費用為基礎 input 費用的 10%；快取寫入費用為 125%
- [Model Context Protocol 規範](https://modelcontextprotocol.io)
