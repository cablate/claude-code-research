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

一個多輪 session 看起來是這樣：

| 輪次 | 發生了什麼 | tools 陣列 | 快取 |
|-----|----------|-----------|-----|
| 1 | 沒使用 MCP 工具 | 只有延遲名稱 | Hit（如果前綴穩定）|
| 2 | 首次使用 `mcp__files__read` | 新增 1 個完整 schema | **Miss** |
| 3 | 再次使用 `mcp__files__read` | 與第 2 輪相同 | Hit |
| 4 | 首次使用 `mcp__search__query` | 再新增 1 個完整 schema | **Miss** |
| 5+ | 重複使用相同工具 | 穩定 | Hit |

每個首次被調用的 MCP 工具都會花費一次保證的快取 miss。Session 使用的不同 MCP 工具越多，在工具陣列穩定之前累積的 miss 就越多。

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
- 每個首次被調用的 MCP 工具都會觸發一次保證的快取 miss
- 在所有需要的工具都至少被使用過一次之前，tools 陣列不會穩定
- 如果平均 session 是 10 輪、你有 15 個 MCP 工具，session 可能在整個持續時間內都處於快取 miss 狀態

只使用內建工具的系統：

- 工具列表是編譯時穩定的
- 不會發生延遲載入
- 從第一輪起快取穩定性幾乎是有保證的

這兩種場景之間的差距，就是 MCP 工具整合中沒有人提到的隱形成本。

---

## 參考資料

- [逆向工程 Claude Agent SDK：每條訊息消耗 2-3% Credit 的根本原因與修復](../agent-sdk-cache-invalidation/README.md) — 涵蓋 prompt cache 的運作方式以及快取 miss 的代價
- SDK 版本：`@anthropic-ai/claude-agent-sdk` v0.2.76，`cli.js` build `2026-03-14`
- Anthropic API prompt caching：快取讀取費用為基礎 input 費用的 10%；快取寫入費用為 125%
- [Model Context Protocol 規範](https://modelcontextprotocol.io)
