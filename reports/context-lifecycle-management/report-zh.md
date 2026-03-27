# Context 生命週期管理 — Claude Code 如何決定何時壓縮、保留什麼、代價是什麼

長對話會成長。每次工具呼叫、每段程式碼、每個來回都往 context window 裡加 token。終究，對話必須縮小，否則它會撞上硬牆。

Claude Code 有一套自動壓縮系統叫 autocompact。它沒有文件。它在你的 context 變太大時無聲啟動，把大部分對話替換成一份摘要，讓你繼續工作，像什麼都沒發生一樣。

但事情確實發生了。Prompt cache 剛剛被摧毀。下一條訊息的成本是正常的 125%。而在某些條件下，壓縮本身還會觸發另一次壓縮，再次摧毀 cache。

這份報告逆向工程了完整的生命週期：context 大小如何計算、五個控制決策的閾值、壓縮何時觸發、什麼能存活、以及兩個此前未被公開記錄的機制——它們如何在你不知情的情況下推高成本。

分析版本：`@anthropic-ai/claude-agent-sdk` v0.2.76（cli.js build 2026-03-14）

---

## Context 大小如何計算

Claude Code 不假設固定的 context window。它在 runtime 根據模型計算有效容量。

第一步決定原始 window 大小。模型名稱含 `[1m]` 或啟用了 `1M` beta flag 的得到 1,000,000 tokens。Sonnet 4.6 加上 high-latency-allowance beta 也是 1,000,000。其他一律 200,000。

接著扣除兩個預留量。一個是模型的最大輸出 token 數——Opus 4.6 是 64,000，Opus 4.5 和 Opus 4 是 32,000，Claude 3 Opus 是 4,096。另一個是固定的 20,000 token 緩衝區，不論模型都會從 window 中永久扣除。

公式：

```
有效容量 = 原始 window - 最大輸出 tokens - 20,000
```

對標準 200k 模型（輸出上限 4,096 token），有效容量大約是 176,000 tokens。但這不是壓縮觸發的位置——上面還疊著額外的邊距。

---

## 五個閾值，五種行為

Claude Code 每輪都拿當前 token 數對照五個閾值進行評估。這些全是硬編碼常數，沒有 UI、沒有設定檔、沒有 settings key。唯一的覆寫方式是環境變數。

**Autocompact 邊距**從有效容量再扣 13,000 tokens。這是自動壓縮的觸發點。在標準 200k 模型上，這代表壓縮在大約 163,000 tokens 時觸發——不是 200,000，不是 176,000，是 163,000。如果你按「接近上限時才壓縮」來規劃，你會差將近 40,000 tokens。

**Warning 閾值**在 autocompact 觸發點之外再加 20,000 token 邊距。Context 越過這條線時 UI 顯示黃色指示器。不採取任何動作。

**Error 閾值**再加 20,000 token 邊距。UI 顯示紅色指示器。除了視覺提示外仍不採取動作。

**硬封鎖限制**距離絕對上限 3,000 tokens。到這個點 Claude Code 完全拒絕接受新訊息。Session 凍結，直到 context 被縮減。

**Circuit breaker** 計算連續壓縮失敗的次數。連續 3 次後系統停止嘗試。這防止在壓縮無法將 context 降到閾值以下時形成無限迴圈。

五個閾值在每輪中以一次 pass 全部評估完。函式回傳一個狀態結構，系統其餘部分讀取它來決定下一步行動。

---

## 決策鏈：Autocompact 何時觸發

在檢查 context 是否太大之前，系統先通過一系列閘門。

首先是三個環境開關。`DISABLE_COMPACT` 設定了的話，所有壓縮都關掉——主動和被動都是。`DISABLE_AUTO_COMPACT` 設定了的話，只封鎖主動路徑；緊急路徑仍然有效。使用者設定中 `autoCompactEnabled` 設為 false，效果等同第二個開關。任何一個生效就代表系統完全跳過閾值檢查。

其次是遞迴防護。壓縮本身是用 LLM 呼叫實作的——它把整段對話送給 Claude 要求摘要。這代表壓縮本身也會增加 context。沒有保護的話，一個較大的摘要就可能觸發另一次壓縮，產生另一份摘要，又觸發下一次。防護機制檢查當前 query 的來源：如果已經是壓縮呼叫，或是 session memory 更新，就跳過閾值檢查。

只有所有閘門都通過之後，系統才拿當前 token 數和 autocompact 閾值比較。超過了，壓縮開始。

---

## 壓縮過程中發生什麼

閘門全部通過且超過閾值後，壓縮執行器啟動。以下是從頭到尾的完整過程。

它先記錄當前 token 數——在任何狀態變更前拍一張快照。然後執行已註冊的 pre-compact hooks。這是擴充點：如果任何 hook 以 exit code 2 退出，整個壓縮被取消。這是唯一的外部否決機制。

接著清空檔案追蹤表。這張表記錄 Claude 在 session 期間讀過或寫過哪些檔案，用來偵測過時內容。清空它會強制下一輪重新建立，防止陳舊的檔案 diff 在壓縮後的 context 中累積。

然後重建 memory 檔案和工具定義附件——確保新 context 有最新的專案狀態參考。

壓縮的核心是建構摘要 prompt。系統建立一個 10 段式請求，要求 LLM 提取並保留：

1. 環境設定 — OS、shell、工具版本
2. 實際完成的變更 — 具體建立、修改、刪除的檔案
3. 讀取過的檔案與程式碼段落 — 檢視了什麼、為什麼重要
4. 問題解決歷程 — 遇到的錯誤、嘗試的假設、找到的解法
5. 待辦工作與阻塞點 — 尚未完成的任務
6. 使用者決策與偏好 — session 期間做出的明確選擇
7. 專案背景與限制條件
8. 當前工作狀態（標記為最重要的段落）
9. 選擇性或低優先度項目 — 提到但未開始的
10. 關鍵產出物 — 需要逐字保留的程式碼塊、設定、檔案內容

這份 prompt 以 `maxTurns: 1` 發送為一次真實 API 呼叫。LLM 以兩個 XML 區段回應：`<analysis>` 區塊用於私有推理（丟棄），`<summary>` 區塊成為新的對話種子。

Summary 回傳後，系統從 API 回應的 usage 欄位測量實際的壓縮後 token 數——不是估算，是真實數字。它也計算一個字元長度估算值供 UI 顯示。最後執行已註冊的 post-compact hooks，流程完成。

舊的對話歷史消失了。取而代之的是：一份摘要，加上一段保留的最近訊息。

---

## 什麼能存活

壓縮不會取代整段歷史。最近的訊息會被原樣保留，接在摘要後面。

保留邏輯從最後一條訊息往回掃描，一邊走一邊累計 token 數。停止條件是：累積到 40,000 tokens，或者至少 10,000 tokens 且至少 5 條 text-block 訊息——看哪個先到。這個範圍內的所有內容原樣保留。

有一個完整性檢查：如果一個 tool call 被保留，對應的 tool result 也必須保留，反之亦然。不允許出現孤立的工具呼叫配對。這防止壓縮後的 context 出現有工具呼叫卻沒有輸出的情況，那會讓模型困惑。

最終結果是一段新對話：LLM 生成的摘要在最前面，然後是 10,000 到 40,000 tokens 的最近原始交談完整保留在最後面。保留參數是硬編碼的——沒有辦法透過設定來擴大或縮小這個保留窗口。

---

## 原創發現：每天無聲發生的 Cache 全面失效

我們發現一個 10 token 的日期字串每天都在無聲地讓數千 token 的 cached context 失效。原因如下。

每一輪，Claude Code 注入一個 `<system-reminder>` 區塊，包含當前日期：

```
Today's date is 2026-03-27
```

日期注入本身不奇怪——模型需要知道日期。問題在於這個注入在訊息組裝中的位置。

日期字串被串接到和 CLAUDE.md 內容、git status、以及其他很少變動的 context 相同的文字區塊中。這些全部合併成一個序列化單元後才送到 API。

Prompt caching 基於嚴格的前綴匹配。Cache key 是整個前綴直到每個 cached 區塊為止的位元組序列。當日期在午夜改變時，這個合併區塊的序列化內容就改變了。因為日期在字串中出現在靜態 CLAUDE.md 內容之前，它之後的每個位元組在前綴中都移位了。

後果：每天午夜，這個注入點之後的所有 cached context 全部失效。當天第一個 session 以 `cache_write` 費率——基礎成本的 125%——支付整段對話歷史的完整 token 數。數千 token 完全沒有變化的內容被重寫，只因為一個 10 字元的日期字串移動了前綴邊界。

這不是使用者程式碼的 bug。這是注入區塊組裝方式的結構性後果——日期被嵌入在和靜態內容相同的序列化單元中，而不是放在獨立區塊裡。據我們所知，這個機制此前未被公開描述過。

對大多數互動式使用者來說，這不可見——session 很少跨越午夜。但跑整夜任務的自動化 agent 每 24 小時都要付這筆費用，而且它隨對話長度成比例增長。

---

## 原創發現：壓縮連鎖反應

壓縮的目的是透過縮小 context 來節省成本。但在某些條件下，它觸發了一個讓成本倍增的連鎖反應。

以下是它的運作方式。

壓縮期間，系統正確地設定了一個 flag 來跳過寫入新的 cache 斷點——摘要是暫時性的，不應該建立新的 cache。壓縮完成後，session 狀態被重置。這使 session 期間累積的所有現有 prompt cache 斷點全部失效。

因此壓縮後的第一輪行為就像全新 session 的第一輪：每個 token 以 `cache_write` 費率（125%）傳送，從零建立一個新的 cache 斷點。

現在關鍵問題來了：如果壓縮後的摘要仍然太大呢？

壓縮後，系統用壓縮後 token 數對照閾值進行評估。如果仍然超過 autocompact 觸發點，它就標記下一輪會再次觸發壓縮。當使用者送出下一條訊息時，閾值檢查再次啟動。第二次壓縮執行。Session 狀態再次重置。又一次完整的 cache 重建隨之而來。

連鎖反應的每個環節造成三重成本：一次完整的 LLM 壓縮呼叫（把整段 context 送去摘要的 API 費用）、一次以 125% 寫入費率對所有 token 的完整 cache 重建、以及觸發下一個環節的可能性。

連鎖反應在兩種情況下停止：token 數終於降到閾值以下，或者 circuit breaker 在連續 3 次失敗後跳閘。

這個情境只在 context 真的非常龐大時才會發生——1M token 的 session 中，即使摘要形式也超過 autocompact 閾值。在互動式使用中，這很罕見。但在跑長時間多小時任務的自動化 agent loop 中，這是一個實際的失敗模式。一次 2-3 環的連鎖反應可能花費相當於 10-15 次正常對話輪次的成本。

---

## 緊急路徑：Reactive 壓縮

Autocompact 是主動的——它在 API 呼叫前根據 token 估算觸發。但估算可能不準。

如果系統低估了 context 大小，送出的請求太大，Anthropic API 回傳 `prompt_too_long` 錯誤。這時 reactive 壓縮啟動。

它執行和 autocompact 相同的壓縮流程，但有一個重要差異：你已經為那個失敗的 API 呼叫付過錢了。Tokens 已經送出、處理、以 `cache_write` 費率計費，然後才被拒絕。Reactive 壓縮再跑一次 API 呼叫。所以你付了兩次——一次是被拒的請求，一次是壓縮本身。

有一個安全限制：reactive 壓縮每輪最多嘗試一次。如果嘗試本身失敗，系統放棄並浮現錯誤。

---

## 自動輸出縮減

與壓縮分開，Claude Code 在 API 層級處理另一種溢出情境：當 input token 數加上請求的 output token 數超過 context 上限。

當這種情況發生時，系統解析錯誤訊息來提取實際數字——input 長度、請求的 output、context 上限。然後計算一個縮減後的 output 配額：

```
縮減後 output = context 上限 - input 長度 - 1,000
```

1,000 token 的邊距是硬編碼的。下限是 3,000 tokens——它不會把 output 配額降到這以下，以確保模型仍能產生有意義的回應。

請求以縮減後的值自動重試。不需要使用者介入，不顯示錯誤。模型只是產出比原本更短的回應。

---

## 你可以調整什麼

Claude Code 開放七個影響 context 生命週期行為的環境變數：

| 變數 | 作用 |
|---|---|
| `DISABLE_COMPACT` | 關掉所有壓縮。如果 context 真的超過 window，session 會卡在 API 錯誤。只在你從外部管理 context 時使用。 |
| `DISABLE_AUTO_COMPACT` | 只關掉主動壓縮。緊急路徑仍有效，但觸發時你付雙倍。 |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | 以 context window 的百分比設定壓縮觸發點。`70` 代表在 70% 容量時壓縮——留出空間來避免連鎖反應。 |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | 覆寫閾值計算中使用的 context window 大小。 |
| `CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE` | 覆寫硬封鎖邊距（預設 3,000 tokens）。 |
| `CLAUDE_CODE_DISABLE_1M_CONTEXT` | 即使模型符合 1M 資格也強制使用 200k window。適合成本控制。 |
| `ENABLE_CLAUDE_CODE_SM_COMPACT` | 啟用基於 session memory 的替代壓縮路徑。預設停用。 |

對成本控制最有用的是 `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`。設成 60-70% 代表壓縮在 context 較小時就觸發，產生較小的摘要，使摘要不太可能仍超過閾值。這是防止連鎖反應最簡單的方法。

---

## 完整費用全貌

單次壓縮事件不只是一次 API 呼叫。以下是你的帳單上實際發生的事：

| 事件 | 費用影響 |
|---|---|
| 壓縮呼叫本身 | 完整 API 呼叫——整段對話送去摘要 |
| 壓縮後第一輪 | 所有 tokens 以 125% `cache_write` 費率（cache 已被摧毀） |
| 第二輪起 | Cache 正常重建，回到 `cache_read` 費率 |
| 連鎖反應（N 個環節） | 以上的 N 倍，疊加計算 |
| 午夜 cache 失效 | 一次完整 `cache_write` 重建，與總對話大小成比例 |

對 160,000 token 的 session，一次壓縮加壓縮後的 cache 重建大約花費 5-8 次正常對話輪次的成本。2-3 環的連鎖反應使之加倍或三倍。

實務建議：如果你在跑 agent loop，監控壓縮後 token 數。如果它仍高於觸發閾值，連鎖反應已經在進行中。降低你的 `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`、把任務拆成更小的 session，或者接受費用尖峰讓 circuit breaker 在 3 次迭代後停止它。

---

## 原始碼位置

所有發現來自 `@anthropic-ai/claude-agent-sdk` v0.2.76 的 `cli.js`。函式透過字串常數錨點定位——這些在版本間的 minification 中不會改變。

| 錨點字串 | 定位什麼 |
|---|---|
| `"CLAUDE_CODE_DISABLE_1M_CONTEXT"` | 原始 context window 大小判定 |
| `"isAboveAutoCompactThreshold"` | 閾值評估函式 |
| `"CLAUDE_AUTOCOMPACT_PCT_OVERRIDE"` | Autocompact 觸發點推導 |
| `"DISABLE_AUTO_COMPACT"` | 環境開關閘門 |
| `"session_memory"` / `"compact"` | 遞迴防護 |
| `"preCompactTokenCount"` | 主壓縮執行器 |
| `"skipCacheWrite"` | 壓縮期間的 cache flag |
| `"Today's date is"` | 日期注入位置 |
| `"minTextBlockMessages"` | 保留參數 |
| `"input length and max_tokens exceed context limit"` | 溢出還原解析器 |
| `"prompt_too_long"` | Reactive 壓縮入口點 |

---

## 相關報告

- [逆向工程 Claude Agent SDK：每條訊息 2–3% 額度消耗的根本原因與修復](../agent-sdk-cache-invalidation/README.md) — 每次呼叫都 spawn 新 process 如何摧毀 prompt cache
- [Prompt Cache 架構](../prompt-cache-architecture/) — cache 斷點如何放置與維護
- [system-reminder 注入](../system-reminder-injection/) — 完整注入分類與 token 費用
