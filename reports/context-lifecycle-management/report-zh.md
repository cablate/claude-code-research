# Context Lifecycle Management — Claude Code 如何決定何時壓縮、保留什麼、代價是什麼

一段對話跑到一半，Claude Code 無聲地把它壓縮了。帳單上出現一個費用尖峰。下一輪慢了。如果你在跑自動化 agent loop，可能兩輪後又重演一次。

這就是 autocompact。沒有文件說明它怎麼運作。這份報告對它進行逆向工程：什麼時候觸發、精確的觸發條件、什麼內容能存活下來、代價是多少——包括兩個據我們所知至今未被公開描述過的機制。

分析版本：`@anthropic-ai/claude-agent-sdk` v0.2.76（cli.js build 2026-03-14）

---

## 方法論

所有發現都來自對 `cli.js` 的直接原始碼分析——這是 12MB 的 minified 引擎，同時支撐 Claude Code CLI 和 Agent SDK。變數名稱已被混淆成無意義的短識別碼；函式透過字串常數錨點定位。

使用的主要錨點：

- `"DISABLE_AUTO_COMPACT"` → 閾值邏輯與觸發閘門
- `"session_memory"` → compaction 流程內部的遞迴防護
- `"compact"` → compaction 期間使用的 query source 標籤
- `"prompt_too_long"` → reactive compact 入口點
- `"input length and max_tokens exceed context limit"` → context overflow 還原解析器
- `"Today's date is"` → currentDate 注入位置

每個發現後面都附有對應的常數名稱或原始碼呼叫鏈。

---

## 發現

### 1. Context Window 大小是計算出來的，不是假設的

Claude Code 不假設固定的 context window。它在 runtime 根據模型逐一計算有效可用空間。

**`uM(model, betas)`** 決定原始 context window：

- 模型名稱包含 `[1m]` 或 `1M` beta feature flag → 1,000,000 tokens
- `sonnet-4-6` + HLA（high-latency-allowance）beta → 1,000,000 tokens
- 所有其他模型 → 200,000 tokens

**`oa(model)`** 決定每個模型的最大輸出 tokens：

| 模型 | 最大輸出 |
|---|---|
| claude-opus-4-6 | 64,000 |
| claude-opus-4-5 | 32,000 |
| claude-opus-4 | 32,000 |
| claude-3-opus | 4,096 |
| 其他 | 不等 |

**`OF(model)`** 計算有效可用 window：

```
effectiveWindow = contextWindow - maxOutputTokens - RmY(20,000)
```

`RmY` 這個 20,000 token 常數是一個永久預留量，從所有 window 大小計算中扣除。這意味著一個 200k window 的模型，在任何閾值生效之前，input 的實際上限約為 164,000 tokens。

---

### 2. 五個硬編碼閾值常數控制所有決策

```
RmY = 20,000   最大輸出 tokens 預留（從每個 window 中扣除）
Jp8 = 13,000   autocompact 安全邊距
hmY = 20,000   warning 閾值邊距
SmY = 20,000   error 閾值邊距
Mp8 = 3,000    硬封鎖限制邊距
aqq = 3        circuit breaker：最大連續 compaction 失敗次數
```

這些全部硬編碼在 minified 原始碼中。沒有設定檔、沒有 settings key、沒有 UI 控制介面。唯一的 override 方式是環境變數（見第 13 節）。

**`mz6(tokens, model)`** 在一次 pass 中評估所有閾值並回傳狀態結構：

- `isAboveAutoCompactThreshold` — 觸發 soft compaction
- `isAtWarningLevel` — 僅供 UI 指示器使用
- `isAtErrorLevel` — 僅供 UI 指示器使用
- `isAtBlockingLimit` — 硬停止，不接受新訊息

**`oc6(model)`** 推導 autocompact 觸發點：

```
autocompactThreshold = OF(model) - Jp8(13,000)
```

對於標準 200k 模型：`(200,000 - 4,096 - 20,000) - 13,000 = 162,904 tokens`。

---

### 3. 閘門前面還有閘門

在 autocompact 檢查 token 數量之前，三個獨立的開關必須全部通過。

**`Xh()`** 檢查：

1. `DISABLE_COMPACT` 環境變數未設定
2. `DISABLE_AUTO_COMPACT` 環境變數未設定
3. `settings.autoCompactEnabled` 不是 false

只要三者之一生效，當輪就完全跳過 autocompact。

**`CmY()`** 是在執行 compaction 前立即執行的前置防護：

- 如果 `querySource` 是 `"session_memory"` → 跳過（session memory 更新會產生遞迴）
- 如果 `querySource` 是 `"compact"` → 跳過（已在 compaction 中，會形成無限迴圈）

這是遞迴防護。Compaction 本身是以 `querySource: "compact"` 的 LLM 呼叫實作的。沒有這個檢查，一個較大的 compaction summary 就可能觸發另一次 compaction。

---

### 4. 十步 Compaction 流程

當所有閘門都通過且 token 數超過閾值，**`mf6()`** 開始執行。步驟依序如下：

1. 透過 `eW(messages)` 記錄 `preCompactTokenCount` — 在狀態變更前捕捉 token 計數
2. 執行 PreCompact hooks — 如果任何 hook 以 exit code 2 退出，compaction 整個取消
3. 清除 `readFileState` cache — 檔案追蹤表被清空（強制下一輪重新建立）
4. 重建 memory 檔案和 tool definition 附件
5. 使用 `C54(customInstructions)` 建立 10 段式 summary prompt
6. 以 `querySource: "compact"` 和 `maxTurns: 1` 呼叫 `Gqq()` — 實際的 LLM 摘要呼叫
7. 用 `sF6(summary)` 建構一個接續起始訊息作為新對話開頭
8. 透過 `Ck()` 測量 `postCompactTokenCount` — 從 API 回應的 usage 欄位讀取真實 token 數
9. 透過 `GF6()` 估算 `truePostCompactTokenCount` — 供 UI 顯示用的字元長度估算
10. 執行 PostCompact hooks

Compaction 呼叫本身是一次真實的 API 呼叫。它消耗 tokens。它產生的 summary 取代之前的所有內容。

---

### 5. Summary Prompt 要求什麼

Compaction 期間傳給 LLM 的 10 段式 prompt：

1. **環境/設定** — OS、shell、工具版本
2. **實際完成的變更** — 具體建立、修改、刪除的檔案
3. **檔案與程式碼段落** — 讀了什麼、為什麼重要
4. **問題解決** — 遇到的錯誤、嘗試過的假設、找到的解法
5. **待辦工作/下一步** — 未完成的任務、阻塞點
6. **使用者決策與偏好** — session 期間做出的明確選擇
7. **背景/環境** — 專案結構、限制條件
8. **當前工作**（標記為最重要）— 現在工作的精確狀態
9. **選擇性工作** — 提到但尚未開始的低優先度項目
10. **關鍵產出物** — 需要逐字保留的重要程式碼塊、設定、檔案內容

LLM 回應格式使用兩個 XML 標籤：

- `<analysis>` — 不包含在輸出中的私有推理（chain-of-thought）
- `<summary>` — 實際的壓縮 context，作為新對話種子使用

---

### 6. 什麼能存活：保留區段

Compaction 不會取代整個對話歷史。最近的訊息會被原樣保留並附加在 summary 之後。

**`dE1`** 定義保留參數：

```javascript
dE1 = { minTokens: 10000, minTextBlockMessages: 5, maxTokens: 40000 }
```

**`EmY(messages, lastIdx)`** 從最新訊息往回掃描：

- 往回累計 tokens
- 停止條件：`tokens >= 40,000` 或 (`tokens >= 10,000` 且 `textBlockMessages >= 5`)
- 符合條件範圍內的所有內容原樣保留

**`Op8()`** 確保完整性：如果一個 `tool_use` 塊被保留，對應的 `tool_result` 塊也必須保留（反之亦然）。不允許出現孤立的 tool call 配對。

最終結果是新的對話歷史：LLM 生成的 summary 在前，然後是 10k–40k tokens 的最新原始訊息完整保留。

---

### 7. 原創發現：currentDate 的 Cache 殺傷問題

每一輪，**`eE1()`** 注入一個 `<system-reminder>` 塊，包含：

```
Today's date is YYYY-MM-DD
```

這本身不奇怪。問題在於它被注入到*哪裡*。

`currentDate` 字串被串接到**同一個文字塊**中，和以下內容放在一起：

- `claudeMd`（CLAUDE.md 內容——靜態，很少變動）
- `gitStatus`（每次 commit 後會變）
- 其他靜態 context

Prompt caching 是嚴格前綴匹配。Cache key 是整個前綴直到每個 cached 塊為止的位元組序列。當 `currentDate` 改變——每天午夜發生一次——這個合併文字塊的序列化結果就會改變。因為日期在字串中位於靜態內容之前，所有在這個位置之後的 prefix 中的每個位元組現在都在不同的偏移量上。

結果：每天午夜，這個注入點之後的所有 cached context 全部失效。當天第一個 session 以 `cache_write` 費率（125% 基礎成本）支付整個對話歷史的完整 token 數。

大約 10 個 token 的改變，造成數千個 token 的未變更靜態內容 cache miss。

這不是使用者程式碼的 bug。這是 `eE1()` 組裝這個塊的方式所造成的結構性特性。據我們所知，這是第一次公開記錄這個機制。

---

### 8. 原創發現：Compact 連鎖反應

Compaction 以一種特定的方式與 prompt cache 互動，可能導致它在短時間內連續觸發兩次。

**Compaction 期間：**

`sqq()`（主要 compaction 執行器）在 compaction 呼叫上設定 `skipCacheWrite: true`。這是正確的——summary 是暫時性的，不應該建立新的 cache 斷點。

**Compaction 之後：**

`gl()` 重置 session 狀態。這使 session 期間累積的所有現有 prompt cache 斷點全部失效。下一輪從零 cached context 開始。

因此 compaction 後的第一輪行為就像新 session 的第一輪：

- 所有 tokens 以 `cache_write` 費率傳送（125%）
- 寫入一個新的 cache 斷點

**如果 summary 仍然很大：**

`mz6()` 評估新的 `postCompactTokenCount`。如果它仍然超過 autocompact 閾值，`willRetriggerNextTurn = true`。在下一次使用者輪次，閾值檢查再次觸發。第二次 compaction 執行。`gl()` 再次重置狀態。另一次完整的 cache rebuild 隨之而來。

這個連鎖反應的每個環節：

1. 觸發一次完整的 LLM compaction 呼叫（API 費用）
2. 從零重建 prompt cache（所有 tokens 的 125% 寫入費用）
3. 可能觸發下一個環節

連鎖反應在 token 計數終於降到閾值以下，或者 `aqq = 3` 連續失敗的 circuit breaker 跳閘時停止。

這只在 context 確實非常大的情況下才可能發生——1M token session 中，即使壓縮後的形式也超過 200k 預設閾值。但在跑長時間任務的自動化 agent loop 中，這不是理論上的邊緣情況。

---

### 9. Reactive Compact：緊急路徑

Autocompact 是主動的——它在 API 呼叫前、當 token 估算超過閾值時觸發。

Reactive compact 在 API 呼叫*失敗後*觸發。

如果 Anthropic API 回傳 `prompt_too_long` 錯誤，**`Bi6.tryReactiveCompact()`** 被呼叫：

- 每輪最多嘗試一次
- 如果已經嘗試過且嘗試失敗 → 回傳 `{reason: "prompt_too_long"}` 並放棄
- 執行相同的 `mf6()` 流程，但由 API 拒絕觸發而非閾值估算

這個區別在費用上很重要：reactive compact 意味著在 compaction 開始前，你已經為那個失敗的 API 呼叫支付了完整的 `cache_write` 費率。

---

### 10. Context Overflow 還原：自動降低 max_tokens

與 compaction 分開，Claude Code 在 API 層級處理 `input + max_tokens > context limit` 的情況。

**`$54()`** 解析錯誤訊息：

```
input length and max_tokens exceed context limit: X + Y > Z
```

它提取數字，計算降低後的 `max_tokens`：

```
reducedMaxTokens = contextLimit - inputLength - 1000
```

1000 token 的邊距是硬編碼的。下限是 **`fN8 = 3,000 tokens`**——它不會將 `max_tokens` 降低到 3,000 以下。

請求以降低後的值自動重試。沒有使用者介入，沒有可見的錯誤。

---

### 11. Microcompact：一個 Stub

**`pg()`** 在主迴圈中、autocompact 執行前被呼叫：

```
microcompact → autocompact
```

`pg()` 的當前實作回傳原始訊息陣列不變。它是一個占位符。Claude Code 的未來版本可能實作部分 context 修剪（針對特定訊息類型或 tool output），在退回到基於摘要的完整 autocompact 之前執行。

---

### 12. Token 估算：三種方法，一個真實

**文字估算** — `j5(text, charsPerToken=4)`：

```javascript
Math.round(text.length / 4)
```

**JSON 估算** — 同一函式，`charsPerToken=2`：

JSON 比散文更密集，所以除數減半。

**真實 token 計數** — 來自 API 回應的 `usage` 欄位：

```javascript
fF6(usage): input + cache_creation + cache_read + output
```

`mz6()` 和閾值檢查在 API 呼叫前使用估算計數。`postCompactTokenCount` 和 circuit breaker 決策在呼叫後使用 API 回應的真實計數。兩者之間存在一致的差距——估算計數用來決定是否要 compact，真實計數用來決定是否成功。

---

### 13. 環境變數控制（完整列表）

| 變數 | 效果 |
|---|---|
| `DISABLE_COMPACT` | 停用所有 compaction（autocompact 和 reactive） |
| `DISABLE_AUTO_COMPACT` | 僅停用主動式 autocompact；reactive compact 仍然執行 |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | 以 context window 的百分比設定閾值：`floor(contextWindow * N / 100)` |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | Override 閾值計算中使用的 context window 大小 |
| `CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE` | Override 硬封鎖限制邊距（`Mp8`） |
| `CLAUDE_CODE_DISABLE_1M_CONTEXT` | 即使對 1M 合格的模型也強制使用 200k window |
| `ENABLE_CLAUDE_CODE_SM_COMPACT` | 啟用基於 session-memory 的替代 compact 路徑（預設停用） |

---

## 影響

### 費用模型

一次標準 autocompact 事件的費用結構如下：

| 事件 | 費用 |
|---|---|
| Compaction LLM 呼叫 | 所有 input tokens 以 `cache_write` 費率的完整 API 呼叫 |
| Compaction 後第一輪 | 所有 tokens 完整 `cache_write`（cache 尚不存在） |
| 後續每輪 | Cache 正常重建——第二輪後 `cache_read` |
| 連鎖反應（N 個環節） | N × （compaction 呼叫 + cache rebuild） |

對於一個 160k token session，單次 compaction 加上一輪 post-compaction 可能相當於 5–8 次正常輪次的費用。

### 對 Agent Loop 的操作影響

跑長時間任務的自動化 agent loop 受影響最深：

1. **閾值意外** — 13,000 token 的 `Jp8` 邊距意味著 compaction 在 window 真正滿之前 13k token 就觸發。預設「在 200k 時 compact」是錯的；實際上在約 163k 時 compact。

2. **午夜失效** — 任何跨越午夜執行的 session，在下一輪都會因為 `currentDate` 注入機制而支付完整的 cache rebuild 費用。跑整夜任務的 agent 應將此計入費用預測。

3. **連鎖反應風險** — 在工作集非常大的 1M context session 中，即使是壓縮後的 summary 也可能超過 200k 預設閾值。這觸發連鎖反應。監控 `willRetriggerNextTurn`（如果有揭露的話）或觀察連續兩次 compaction 事件是唯一的早期預警。

4. **Reactive compact 雙重費用** — 如果主動式 autocompact 被停用但 context 增長很大，第一個 `prompt_too_long` 錯誤觸發 reactive compact。你為失敗的呼叫付了一次，然後又為 compaction 付了一次。

---

## 緩解措施

### 不要盲目停用 Compaction

`DISABLE_COMPACT` 會停用 autocompact 和 reactive compact 兩者。如果 context 真的超過 window，session 會卡在 API 錯誤上。只有在你自己從外部管理 context 大小時，才使用 `DISABLE_AUTO_COMPACT`。

### 使用 `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` 提早 Compact

如果你想避免連鎖反應，在 window 較低百分比時 compact。範例：`CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=70` 在 context window 的 70% 時觸發 compaction，留出空間來吸收比預期更大的 summary。

### 為午夜 Cache 失效做預算

如果在跑整夜 agent，每個 session 每天預算一次完整的 cache rebuild。費用與午夜時的總對話 token 計數成正比。執行時間長到跨越多個午夜的 session 每次都要付這個費用。

### 監控 `postCompactTokenCount` vs 閾值

Compaction 後，將 post-compaction token 計數與 autocompact 閾值比較。如果仍然高於閾值，下一次使用者輪次就保證會發生連鎖反應。此時唯一的解法是降低 `max_tokens` 輸出限制、拆分任務，或手動再次 compact。

### 保留區段不可設定

`dE1` 中的 10k–40k 保留區段參數是硬編碼的。沒有辦法透過設定來擴大或縮小保留 window。必須在 compaction 後原樣存活的工作，應該盡量放到接近對話末尾的 tool result 或結構化輸出中，讓它落在保留 window 範圍內。

---

## 參考資料

### 原始碼位置（v0.2.76 cli.js）

| Symbol | 錨點字串 |
|---|---|
| `uM(model, betas)` | `"CLAUDE_CODE_DISABLE_1M_CONTEXT"` |
| `OF(model)` | context window 減去 `RmY` 和 `oa(model)` 呼叫 |
| `mz6(tokens, model)` | `"isAboveAutoCompactThreshold"` |
| `oc6(model)` | `"CLAUDE_AUTOCOMPACT_PCT_OVERRIDE"` |
| `Xh()` | `"DISABLE_AUTO_COMPACT"` |
| `CmY()` | `"session_memory"` 鄰近 `"compact"` 防護 |
| `mf6()` | `"preCompactTokenCount"` |
| `sqq()` | `"skipCacheWrite"` 鄰近 compact 邏輯 |
| `eE1()` | `"Today's date is"` |
| `dE1` | `"minTextBlockMessages"` |
| `EmY()` | 保留掃描中的 `"minTokens"` |
| `$54()` | `"input length and max_tokens exceed context limit"` |
| `Bi6.tryReactiveCompact()` | `"prompt_too_long"` |
| `pg()` | 出現在主迴圈中 autocompact 之前，回傳訊息不變 |

### 本倉庫相關報告

- [逆向工程 Claude Agent SDK：每條訊息 2–3% 額度消耗的根本原因與修復](../agent-sdk-cache-invalidation/README.md) — session 層級的 cache 失效機制
- [Prompt Cache 架構](../prompt-cache-architecture/) — cache 斷點如何放置與維護
- [system-reminder 注入](../system-reminder-injection/) — 完整注入分類與 token 費用
