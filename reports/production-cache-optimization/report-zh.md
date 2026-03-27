# 把快取效率推上 90% 的七個補丁

V2 持久 session 解決了最根本的問題——這在報告 #1 中有完整記錄。過去 SDK 每次呼叫都產生新的 process，從頭重建 runtime 注入，整個 prompt cache 直接 miss。改用長駐 process 之後，效率從卡死的 25% 爬到大約 84%。

但還有 16 個百分點的空間。剩下的浪費從哪來？快取 TTL 在閒置時過期。Compaction 事件把累積的快取整個清掉。MCP 重連後工具陣列的序列化順序改變。監控上的盲區讓這些問題靜默發生，直到帳單來了才發現。

本報告記錄七個最佳化——三個 `cli.js` 補丁、兩個 wrapper 層的修改、一個監控系統、以及一個把前兩者組合就自動獲得的效果。每一個都對應報告 #3 到 #5 追蹤出的具體機制。全部加起來，能補上大部分剩餘的差距。

---

## 為什麼我們用錨字串來打補丁

在進入個別最佳化之前，先理解補丁方法論。七個最佳化中有三個需要直接修改 `cli.js`。

`cli.js` 是一個 12MB 的 minified JavaScript 檔案。每次 SDK 發版都會重新混淆變數名稱——v0.2.76 裡叫 `o3z` 的函數，下個版本可能變成 `p4a`。用行號或變數名稱來定位補丁點，脆弱到完全不可用。

但字串常數不受 minification 影響。一個檢查 `QA()==="bedrock"` 的函數，在下一次 build 中仍然包含這個完全相同的字串，即使函數本身改了名字。錨字串的做法是：找到一個只出現在目標函數中的唯一字串常數，確認它在整個檔案中只出現一次，然後做定向的字串替換。一個 postinstall 腳本在每次 `npm install` 後自動執行，冪等地套用每個補丁。

這不是什麼優雅的工程。這是對一個你無法控制的 binary 進行有控制、可驗證的手術。替代方案——fork 整個 SDK——在各方面都更糟。

---

## 補丁一：強制啟用一小時快取 TTL

Anthropic API 支援兩個快取 TTL 等級：標準的五分鐘窗口，以及保留給第一方整合的一小時窗口。你拿到哪個等級取決於 server 端的 feature flag。在 `cli.js` 裡，一個閘門函數檢查當前的 query source 是否在允許名單上。SDK 的 query source 值可能不在那個名單中。

實際後果：一個 SDK session 只要閒置六分鐘——在 agent 工作流中，人類訊息之間這是很正常的間隔——就會失去全部快取。下一輪要付 125% 的成本從頭重建大約 45,000 個 token。

**補丁**在閘門函數的開頭插入一個短路判斷。如果 query source 是 `"sdk"` 或 `"repl_main_thread"`，直接回傳 true，跳過允許名單檢查。

```
錨字串：  function o3z(A){if(QA()==="bedrock"
替換為：  function o3z(A){if(A==="sdk"||A==="repl_main_thread")return!0;if(QA()==="bedrock"
```

Server 端仍然可能執行自己的 TTL 策略，所以這是 best-effort。但實際上，以前五分鐘閒置就丟失快取的 session，現在可以存活長達一小時。風險幾乎為零——這只改變快取持續時間，不影響正確性。

---

## 補丁二：縮小 Context Overflow Margin

當對話接近模型的 context 上限，`cli.js` 會觸發 overflow recovery——一個把訊息歷史壓縮成摘要的 compaction。Recovery 邏輯在判斷剩餘空間時，會減去 1,000 token 的安全邊際。這個邊際太保守了。SDK 本身已經另外維護了一個 3,000 token 的輸出下限作為獨立的安全緩衝。

把邊際從 1,000 縮小到 200，讓 overflow recovery 觸發前多出 800 個可用 token。每次避免掉的 compaction 事件都保住了現有的快取。單獨來看效果不大，但 compaction 是對快取效率最具破壞性的事件——每次 compaction 都會清除前綴，強制付出 125% 的完整重建。

```
錨字串：  contextLimit:W}=X,Z=1000,G=Math.max(0,W-P-1000)
替換為：  contextLimit:W}=X,Z=200,G=Math.max(0,W-P-200)
```

風險很低。3,000 token 的輸出下限是獨立的檢查，不受這個邊際縮小的影響。

---

## 補丁三：提高 Compaction 閾值

SDK 內建的 compaction 在 context 總量達到 100,000 token 時觸發。一旦觸發，訊息歷史被壓縮摘要取代，prompt cache 前綴被徹底摧毀。在工具使用頻繁的 session 中——這幾乎就是所有生產環境的 session——達到 10 萬是家常便飯。

每次 compaction 事件的成本：完整快取重建（約 45k token 的 125%）加上摘要生成本身的費用。在長期運行的 session 中，10 萬時的一次 compaction 接著 context 快速重新膨脹，可以觸發連鎖反應：compact、重建快取、再次膨脹回 10 萬、又 compact。

把閾值提高到 150,000 token，延長了跑道。每個 session 的 compaction 事件更少，完整快取重建也更少。

```
錨字串：  var nX7=1e5,
替換為：  var nX7=15e4,
```

這是三個補丁中風險最高的一個。Session 在 compaction 前會消耗更多 context，可能在 context window 較小的模型上觸發 `prompt_too_long` 錯誤。在生產環境中監控這些錯誤。如果出現了，10 萬到 15 萬之間的值——比如 12 萬或 13 萬——可能更適合你的工作量。

---

## Wrapper 修復：穩定工具排序

這個不需要打補丁。在你自己的 wrapper 程式碼中加一個 `.sort()` 呼叫，就能防止一個隱蔽且難以診斷的快取破壞。

`cli.js` 送到 API 的工具陣列從來沒有在內部排序過。順序取決於每個工具何時被註冊，而這又取決於 MCP server 回應初始化的順序。如果某個 server 斷線重連——以稍微不同的順序重新註冊工具——序列化後的工具列表就會改變。工具列表是 system prompt 的一部分，任何改變都會使 system prompt 的快取前綴失效。報告 #5 詳細追蹤了這個機制。

修復方法是在傳入 session 之前對工具陣列和 MCP server key 排序：

```javascript
const sortedTools = [...allowedTools].sort();
const sortedMcpServers = Object.fromEntries(
  Object.entries(mcpServers).sort(([a], [b]) => a.localeCompare(b))
);
```

排序對 API 而言是不可見的。可用工具集完全相同。唯一改變的是序列化後的位元組順序，而這正是快取前綴匹配所在乎的。零風險。

---

## Wrapper 修復：快取保活 Ping

API 快取有 TTL。當一個 session 進入閒置——任何方向都沒有訊息——計時器就在跑。TTL 一到，下一則訊息要付完整價格重建快取。

即使補丁一把 TTL 延長到一小時，真實世界的 agent 工作流還是有閒置間隔。使用者去吃午餐了。批次作業處理完佇列在等下一批。快取靜默地過期了。

每四分鐘一次的保活 ping 大約花費 10 個輸出 token——以 Sonnet 4 費率計算約 $0.000015。一次 60k token context 在 125% 下的完整快取 miss 大約花 $0.0028。算下來，只要能避免一次快取 miss，就能回收 187 次 ping 的成本。在有實際流量的 session 中，第一次避免的重建就回本了。

實作中會在 ping 之前檢查閒置時間——如果 session 在過去三分鐘內有被使用，就跳過 ping。如果 ping 失敗，計時器停止。不干擾 session 的實際工作。

```javascript
class CacheKeepalive {
  constructor(session, intervalMs = 4 * 60 * 1000) {
    this.session = session;
    this.lastUsed = Date.now();
    this.timer = setInterval(() => this.ping(), intervalMs);
  }

  touch() { this.lastUsed = Date.now(); }

  async ping() {
    if (Date.now() - this.lastUsed < 3 * 60 * 1000) return;
    try {
      for await (const _ of this.session.stream("Reply with only 'ok'")) {}
    } catch (err) {
      console.warn('[cache-keepalive] ping failed:', err.message);
      clearInterval(this.timer);
    }
  }

  stop() { clearInterval(this.timer); }
}
```

---

## 監控：追蹤快取效率

快取破壞不會產生錯誤。Session 繼續正常運作。唯一的信號是更高的帳單，而等你發現的時候，損失已經累積了好幾天甚至好幾週。

重要的指標是快取讀取佔總輸入 token 的比例：

```
效率 = cacheReadInputTokens / (inputTokens + cacheReadInputTokens + cacheCreationInputTokens)
```

單純的逐輪數字太嘈雜——任何 session 的第一輪一定是寫入快取，效率 0%。alpha 0.3 的指數移動平均（EMA）可以平滑這個波動，同時仍然能快速反應持續性的下降。

警告閾值設在 50%。Session 的前幾輪過後，健康的快取應該讀取多於寫入。如果滾動 EMA 持續低於 50%，代表每一輪都在破壞快取：工具排序不穩定、MCP server 重連、compaction 連鎖反應、或是 session process 本身重啟了。

```javascript
class CacheEfficiencyMonitor {
  constructor(alpha = 0.3, warnThreshold = 50) {
    this.alpha = alpha;
    this.threshold = warnThreshold;
    this.ema = null;
  }

  record(usage) {
    const { input_tokens, cache_read_input_tokens, cache_creation_input_tokens } = usage;
    const total = input_tokens + cache_read_input_tokens + cache_creation_input_tokens;
    if (total === 0) return;

    const efficiency = (cache_read_input_tokens / total) * 100;
    this.ema = this.ema === null
      ? efficiency
      : this.alpha * efficiency + (1 - this.alpha) * this.ema;

    if (this.ema < this.threshold) {
      console.warn(
        `[cache-monitor] 效率 ${this.ema.toFixed(1)}% 低於 ${this.threshold}% 閾值`,
        { efficiency, input_tokens, cache_read_input_tokens, cache_creation_input_tokens }
      );
    }
  }
}
```

監控觸發時，診斷路徑很直觀：檢查工具排序是否改變了（補丁四）、MCP server 是否重連了、compaction 是否剛執行了（補丁三）、或是 session process 是否整個重啟了。

---

## 我們調查了但選擇不補丁的項目

不是每個機會都值得出手。三個機制在逆向工程階段看起來很有潛力，但最終被擱置了。

**currentDate 注入。** 每一輪，一個日期字串被注入到 messages 陣列的第一個位置。在持久 session 中它一天只變一次，意味著每天恰好破壞一次快取前綴——煩人但可以接受。問題在於這個注入深埋在組裝管道裡，和 CLAUDE.md、記憶檔案的注入交織在一起。外科式移除有可能靜默損壞整個 context 組裝。V2 持久 session 已經把傷害限制在每天一次而非每輪一次。

**延遲工具載入。** `cli.js` 有一個工具的懶載入路徑，能減少初始 prompt 大小。停用它會讓所有工具一開始就載入，產生一個更大但更穩定的初始 prompt。這個取捨——更大的初始快取寫入 vs. 更少的增量變動——沒有明確的優劣，而且實作和 MCP 初始化序列緊密耦合。上行空間不足以證明耦合風險的合理性。

**Subagent 結果剪枝。** Subagent 完成後，結果被附加到訊息歷史，並以摘要形式在 compaction 後存活。剪枝這些條目會減少 context 大小，但那些摘要承載了影響後續輪次任務連貫性的資訊。這是一場我們選擇不賭的品質賭注。

---

## SDK 升級之後

當新版本的 `@anthropic-ai/claude-agent-sdk` 安裝時，postinstall 腳本自動執行。如果某個錨字串移動或消失了，腳本會報錯退出，build 會失敗——這是刻意設計的。一個靜默的補丁失敗去損壞 `cli.js`，遠比一個大聲的 build 失敗更糟。

部署前手動驗證：

```bash
grep -c 'function o3z(A){if(QA()==="bedrock"' node_modules/@anthropic-ai/claude-agent-sdk/cli.js
grep -c 'contextLimit:W}=X,Z=1000' node_modules/@anthropic-ai/claude-agent-sdk/cli.js
grep -c 'var nX7=1e5,' node_modules/@anthropic-ai/claude-agent-sdk/cli.js
```

每個都應該回傳 `1`。回傳 `0` 代表錨字串移動了——用各補丁段落中描述的周邊字串常數去重新定位它。函數的用途在版本之間不會改變；只有 minified 後的名字會變。

Wrapper 層的修復（工具排序、保活 ping、監控）不受 SDK 升級影響。它們作用在你自己的程式碼上，不碰 SDK 的內部。

---

## 參考資料

- [報告 #1：Agent SDK 快取失效](../agent-sdk-cache-invalidation/)——建立 84% 基準線的 V2 持久 session 修復
- [報告 #3：Prompt Cache 架構](../prompt-cache-architecture/)——快取前綴匹配在內部如何運作
- [報告 #5：工具序列化與快取穩定性](../tool-serialization-cache-stability/)——為什麼工具排序對快取很重要
- `cli.js` build 2026-03-14，`@anthropic-ai/claude-agent-sdk` v0.2.76
- [Anthropic API：Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
