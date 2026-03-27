# 工具序列化與快取穩定性 — 為什麼你的 MCP 工具可能正在悄悄破壞 Prompt Cache

你在 Claude Code 加入一個 MCP server。快取效率突然下降。再加第二個，情況更糟。SDK 文件和 Claude Code 的 changelog 都沒有提到工具排序與 prompt caching 之間的關係。沒有任何警告。

這份報告記錄了 Claude Code 內部工具 pipeline 的運作方式、MCP 工具載入為什麼在設計上是非確定性的，以及這些行為中哪些會在對話中途悄悄讓 prompt cache 失效。

分析基於逆向工程 `cli.js` build `2026-03-14`，打包於 `@anthropic-ai/claude-agent-sdk` v0.2.76。

---

## 方法論

`cli.js` 是一個 12MB 的壓縮 JavaScript 檔案。每次 build 都會重新混淆變數名稱。方法是透過不隨 build 改變的字串常數來定錨。

本次調查使用的錨點：

- `"isMcp"` — 定位 MCP 工具過濾與延遲載入邏輯
- `"available-deferred-tools"` — 定位延遲工具在 system prompt 中的區塊
- `"input_schema"` — 定位 `Sh1()`，工具序列化函數
- `"tengu_defer_all"` — 定位控制全工具延遲的 feature flag
- `"disallowedTools"` — 定位 `_c()`，工具過濾函數
- `"K0"`（去重 key）— 定位 `u66()`，合併步驟

從每個錨點出發，追蹤周圍函數及其調用圖。透過跟隨資料流從初始工具列表建構到最終 API payload，重建了這個 4 階段 pipeline。

---

## 4 階段工具 Pipeline

Claude Code 發出的每個 API 請求，都要經過四個獨立階段來建構 `tools` 陣列。

### 第一階段：`ng()` — 內建工具登錄表

`ng()` 回傳一個硬編碼的字面陣列，包含所有內建工具（Read、Write、Edit、Bash、Grep、Glob、WebFetch、TodoWrite 等）。這個陣列中條目的順序在 bundle 編譯時就已確定，不同執行之間不會改變。

這是整個 pipeline 中唯一從構造上就完全確定性的階段。

### 第二階段：`FX()` — allowedTools / denyRules 過濾

`FX()` 取得內建陣列後，使用 `allowedTools` 和 `denyRules` 設定執行 `.filter()`。因為 `.filter()` 保留插入順序，存活下來的條目相對順序與第一階段相同。

如果你傳入 `allowedTools: ["Bash", "Read", "Write"]`，結果列表會按照這些工具在第一階段的順序排列，而不是你指定的順序。`allowedTools` 陣列被當作成員檢測，不是排序指令。

### 第三階段：`u66()` — 內建 + MCP 合併

`u66()` 通過串接內建工具和 MCP 工具產生最終工具列表：

```javascript
[...builtins, ...mcpTools]
```

然後用 key 函數 `K0` 去重（基於 Map 的去重，保留第一次出現的）。結果是：內建工具永遠排在 MCP 工具前面，如果 MCP 工具與內建工具同名，內建工具優先。

### 第四階段：`mGq()` → `Sh1()` — 序列化

`Sh1()` 將每個工具轉換成 API wire format：

```javascript
{
  name: tool.name,
  description: await A.prompt(),   // 非同步
  input_schema: tool.inputJSONSchema ?? fU(tool.inputSchema)
}
```

這裡有兩個細節很重要：

**`description` 是非同步的。** 內建工具同步回傳靜態字串。MCP 工具可能呼叫 server 取得動態 description。server 回應中的任何非確定性都會直接傳播到序列化後的 payload。

**schema 來源依工具類型不同。** MCP 工具使用 `inputJSONSchema`（MCP server 提供的 schema）。內建工具通過 `fU()` 從 Zod 定義推導其 schema，`fU()` 用 WeakMap 做記憶化——所以內建工具的 schema 在進程內是穩定的。

在主要查詢路徑中，`cache_control` 不會應用於個別工具層級。工具不會被單獨快取；只有組合好的 system prompt 才會有快取標記。

---

## 關鍵發現：整個工具路徑中沒有任何 `.sort()`

在整個 `cli.js` 原始碼中搜尋工具相關代碼附近的 `.sort()` 呼叫。找到的每一個 `.sort()` 呼叫都屬於以下類別之一：

- Worktree 路徑排序
- Insights 資料排序
- Help 選單顯示
- Compact metadata 輸出

在 `ng()`、`FX()`、`u66()`、`mGq()` 或 `Sh1()` 中沒有任何 `.sort()`。

**工具順序在所有四個階段都嚴格依插入順序。** 這有一個具體後果：如果 MCP 工具的插入順序在兩次執行之間不同，序列化後的 `tools` 陣列就不同，prompt cache 就會 miss。

---

## MCP 工具排序：理論上穩定，實際上脆弱

MCP 工具通過 `Fr6()` 進入 pipeline，它使用並行 server 初始化（`ZL1`）從所有已登錄的 MCP server 載入工具。載入是並行的——server 回應的到達順序是非確定性的。

session 中儲存的工具列表是在啟動時從 client 登錄順序建構的。設定檔的順序決定登錄順序。對於從磁碟讀取的穩定設定檔，登錄順序在重啟之間是一致的。對於程式化構建的設定（例如通過 SDK options 注入），插入順序取決於呼叫方。

實際上，MCP 工具順序通常是穩定的。脆弱性出現在兩種場景：

1. **程式化設定構建**，其中工具順序沒有被明確控制（例如，對一個 key 順序不確定的物件進行迭代）。
2. **MCP server 重新連接事件**，server 斷線後恢復。重新連接時，工具列表重新登錄，可能在合併後的陣列中出現在不同位置。

這兩種場景都會產生結構上不同的 `tools` 陣列。Anthropic API 將整個 `tools` 區塊視為快取前綴的一部分。不同的陣列 = 不同的前綴 = 完整的快取 miss。

---

## 延遲工具載入：對話中途的快取破壞者

這是最有影響力的發現。

`GX()` 實現了延遲工具載入。規則很簡單：每個 `isMcp === true` 的工具預設都是延遲的。

「延遲」在實際中的含義：

- 工具的完整 schema **不會** 在初始 API 請求中發送
- 相反，只有工具名稱被列在 system prompt 中的 `<available-deferred-tools>` 區塊裡
- system prompt 看起來像：`<available-deferred-tools>mcp__server__toolname, ...</available-deferred-tools>`

當 Claude 判斷它需要一個延遲工具時，會執行一個工具搜尋步驟來載入完整 schema。此時 `tools` 陣列獲得一個帶有完整 `{name, description, input_schema}` 物件的新條目。

**這意味著在對話中首次調用任何 MCP 工具，對那次對話輪次來說是一個確定性的快取 miss。** 呼叫前的 `tools` 陣列與呼叫後的結構不同。快取前綴已經改變。

在多輪 session 中：

| 對話輪次 | 使用的 MCP 工具 | tools 陣列狀態 | 快取結果 |
|---------|---------------|---------------|---------|
| 1 | 無 | 僅延遲條目 | hit（如果穩定）|
| 2 | `mcp__files__read`（首次使用）| 新增 1 個完整 schema 條目 | **miss** |
| 3 | `mcp__files__read`（已快取）| 與第 2 輪相同 | hit |
| 4 | `mcp__search__query`（首次使用）| 再新增 1 個完整 schema 條目 | **miss** |
| 5+ | 相同工具 | 穩定 | hit |

session 中每個首次被調用的 MCP 工具都會花費一次快取 miss。你使用的不同 MCP 工具越多，session 在工具陣列穩定之前積累的確定性 miss 就越多。

Feature flag `tengu_defer_all` 將延遲擴展到所有工具，包括內建工具。啟用時，初始工具陣列幾乎是空的，每次工具使用都會觸發延遲載入。這個 flag 似乎用於特定的內部場景。

---

## `disallowedTools` 過濾：`_c()` 保留順序

`_c()` 將 `disallowedTools` 設定物化為 Set，然後應用 `.filter()` 移除匹配的條目。順序被保留。過濾在 agent 或 subagent 層級應用，來源於 agent 定義的 frontmatter。

從 SDK 側，`disallowedTools` 作為 `--disallowedTools ToolA,ToolB` CLI 參數傳遞。這個階段也沒有應用任何排序。

---

## 完整快取影響分析

按可能性和嚴重性排序：

### 高影響：延遲工具載入（每個 session 首次 MCP 使用）

在 MCP 工具首次載入的那輪對話，確定性地快取 miss。影響所有使用 MCP 工具的 session。miss 是結構性的——延遲載入前後的 `tools` 陣列字面上就是不同的。

### 高影響：動態 MCP 工具 description

如果 MCP server 的工具 description 是動態生成的（例如，包含時間戳、資料庫記錄計數或任何運行時狀態），序列化後的 description 在對話輪次之間會改變。即使任何工具 description 中有一個字元的差異，也會讓整個 `tools` 區塊失效。

`Sh1()` 中的 `description` 欄位來自 `await A.prompt()`。對於 MCP 工具，這從 server 解析。如果 server 回傳任何變化的內容，快取就會中斷。

### 中等影響：MCP server 設定排序

如果 `mcpServers` 以 JavaScript 物件傳遞（在 SDK 使用中很常見），key 迭代順序取決於插入順序，這在所有環境和版本中不保證是一致的。兩個名義上相同但 key 插入順序不同的設定，會產生不同的 MCP 工具陣列。

### 中等影響：`extraToolSchemas` 變化

某些功能只在啟用時才向陣列注入額外的工具 schema。Web search 是一個例子：它的 schema 只在那輪對話中 web search 功能啟用時才出現在 tools 陣列中。在對話中途切換功能會改變 tools 陣列結構。

### 低影響：`isEnabled()` 狀態變化

每個工具都有一個 `isEnabled()` 斷言。如果 MCP server 連接中斷，MCP 工具可能變為停用。重新連接時改變了哪些工具是 `isEnabled()` 的狀態，會改變過濾後的工具列表。在穩定設置中很少見，但值得注意。

### 無影響：內建工具 description

內建工具從 `A.prompt()` 回傳靜態字串。這些字串在實際中不是非同步的，也不會在執行之間變化。從快取角度來看，內建工具 description 是安全的。

### 無影響：內建工具 schema

內建工具 schema 通過 `fU()` 從 Zod 定義推導，在進程生命週期內通過 WeakMap 記憶化。它們在對話輪次之間不會改變，是安全的。

---

## 緩解策略

### 1. 在傳遞給 SDK 之前對 `allowedTools` 排序

```javascript
// 之前（不穩定——取決於物件插入順序）
const options = {
  allowedTools: getEnabledTools()
};

// 之後（穩定——字母順序）
const options = {
  allowedTools: getEnabledTools().sort()
};
```

一行代碼。消除一個排序不穩定的來源。

### 2. 對 `mcpServers` 的 key 排序以獲得穩定的序列化

```javascript
// 之前
const mcpServers = buildMcpConfig();

// 之後
const mcpServers = Object.fromEntries(
  Object.entries(buildMcpConfig()).sort(([a], [b]) => a.localeCompare(b))
);
```

確保程式化構建的 MCP 設定，無論插入順序如何，都產生一致的 key 順序。

### 3. 讓 MCP 工具 description 完全靜態

如果你控制 MCP server，確保其工具 description 是硬編碼的常數。`description` 欄位中不要有時間戳、資料庫查詢或任何環境相關的內容。

```python
# 不好：動態 description
@mcp.tool()
def search(query: str) -> str:
    """在 {len(self.documents)} 個文件中搜尋。"""
    ...

# 好：靜態 description
@mcp.tool()
def search(query: str) -> str:
    """在文件索引中搜尋。"""
    ...
```

### 4. 在 session 開始時預熱 MCP 工具

如果你知道一個 session 會使用哪些 MCP 工具，可以提前觸發它們，強制延遲載入在快取敏感的輪次之前發生。在 session 初始化時對每個工具做一次無操作調用，把確定性的 miss 放在前面而不是對話中途。

```javascript
// Session 初始化
await session.query("列出可用的 mcp 工具", { maxTurns: 1 });
// 現在所有 MCP 工具都已載入；後續輪次可以快取
```

### 5. 考慮在快取關鍵 session 中停用延遲載入

如果你的 session 使用已知的固定 MCP 工具集，停用延遲載入意味著所有 schema 都預先序列化。不會有來自新工具載入的對話中途快取破壞。這是一個取捨：初始請求更大，但從第一輪就有穩定的 `tools` 陣列。

這需要打補丁或 flag——目前沒有直接停用延遲載入的公開 API 選項。

### 6. 監控跨對話輪次的 tools 陣列

加入日誌記錄，捕獲 session 前幾輪的序列化 tools 陣列。如果你看到第 2、3、4 輪中出現新條目，那些就是導致快取 miss 的延遲載入。在決定緩解措施之前，先量化這個模式。

---

## 對真實系統的影響

對於一個使用 3 個 MCP server（每個 4–6 個工具）的系統：

- **基準**：session 開始時 15–18 個延遲工具
- **前 15–18 輪**：每個首次被調用的 MCP 工具至少有一次確定性 miss
- **所有工具載入後**：`tools` 陣列穩定，快取可以累積

如果你的平均 session 是 10 輪，你有 15 個 MCP 工具，每個 session 可能在整個持續時間內都處於快取 miss 狀態。

對於一個沒有 MCP 工具（只有內建工具）的系統：

- 第一階段輸出是編譯時穩定的
- 第二階段輸出如果 `allowedTools` 排序或省略就是穩定的
- 沒有延遲載入
- 從第一輪起快取穩定性幾乎是有保證的

---

## 參考資料

- [逆向工程 Claude Agent SDK：每條訊息消耗 2–3% Credit 的根本原因與修復](../agent-sdk-cache-invalidation/README.md) — 前置閱讀；涵蓋 prompt cache 的運作方式以及快取 miss 的代價
- SDK 版本：`@anthropic-ai/claude-agent-sdk` v0.2.76，`cli.js` build `2026-03-14`
- Anthropic API prompt caching：快取讀取費用為基礎 input 費用的 10%；快取寫入費用為 125%
- [Model Context Protocol 規範](https://modelcontextprotocol.io) — `Sh1()` 中引用的 MCP 工具 schema 格式
