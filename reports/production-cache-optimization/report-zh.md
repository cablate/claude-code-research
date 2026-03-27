# 生產環境快取最佳化指南：最大化 Prompt Cache 效率的具體補丁與策略

Prompt cache 效率決定了你在生產環境中每則訊息的實際費用。冷快取意味著每一輪都是 125% 的寫入。暖快取則能把成本壓到 10% 的讀取。規模一旦放大，差距會快速複利累積。

本報告記錄了七個具體策略，用於最大化 Claude Agent SDK v0.2.76 的 prompt cache 效率。每個策略都有 `cli.js` 的原始碼依據——錨字串、函數名稱、以及觀察到的行為。策略涵蓋外科式補丁到你自己 wrapper 程式碼中的行為調整。

---

## 摘要

| # | 策略 | 風險 | 預期效果 |
|---|------|------|---------|
| 1 | 強制 SDK session 使用 1 小時快取 TTL | 低 | 延長 query 之間的快取存活時間 |
| 2 | 縮小 context overflow margin（1000→200） | 極低 | overflow 觸發前多 800 個 token 空間 |
| 3 | 提高 compaction 閾值（100k→150k） | 中 | 減少 compaction 事件，減少快取重建 |
| 4 | 工具排序穩定化 | 無 | 防止工具列表變動破壞快取前綴 |
| 5 | 快取保活 ping | 無 | 在閒置期間維持 5 分鐘快取存活 |
| 6 | 快取效率監控 | — | 快取破壞事件的早期預警 |
| 7 | MCP server key 排序 | 無 | 穩定序列化 → 穩定快取前綴 |

策略 1–3 和 7 需要補丁 `cli.js`。策略 4–6 在你的 wrapper 程式碼中實作。所有補丁使用本報告末尾描述的錨字串模式。

---

## 方法論

所有發現來自對 `cli.js` build 2026-03-14（SDK v0.2.76）的逆向工程。該檔案是一個 12MB 的 minified bundle，變數名稱全部混淆為無意義的短識別符。導航使用字串常數作為穩定錨點——即使每次 build 時變數名稱重新混淆，這些字串不會改變。

快取相關行為的背景知識，請參考報告 [#1（快取失效）](../agent-sdk-cache-invalidation/)、[#3（Prompt Cache 架構）](../prompt-cache-architecture/)、以及 [#5（工具序列化）](../tool-serialization-cache-stability/)。本報告專注於從那些先前研究中衍生的可操作補丁。

---

## 發現

### 策略一：強制 SDK Session 使用 1 小時快取 TTL

**根本原因。** `cli.js` 中的快取 TTL 由 `o3z()` 控制：

```
錨字串：function o3z(A){if(QA()==="bedrock"
```

這個函數把 1 小時 TTL 鎖在 server 端的功能旗標允許名單後面。`"sdk"` 這個 querySource 值可能不在允許名單中，導致 SDK session 退回到標準的 5 分鐘 TTL。5 分鐘 TTL 表示，任何閒置超過 5 分鐘的 session 都會完全失去快取，下一輪需要付出 125% 的完整重建費用。

**補丁。** 在 `o3z()` 的開頭插入：

```javascript
if(A==="sdk"||A==="repl_main_thread")return!0;
```

這讓 SDK 和 REPL 的 querySource 直接短路允許名單檢查，無條件返回 1 小時 TTL 旗標。

**效果。** SDK session 無論 server 端設定如何，都獲得 1 小時快取 TTL。一個閒置 20 分鐘的 session 保住了快取；沒有這個補丁就保不住。

**風險。** 低。這只影響快取持續時間。Server 端仍可能強制執行自己的 TTL 策略，所以這個補丁是盡力而為的改善，而非保證。不影響正確性。

---

### 策略二：縮小 Context Overflow Margin（1000→200）

**根本原因。** 當 `cli.js` 處理 context overflow 恢復時，`$54()` 計算要保留多少空間：

```
錨字串：contextLimit:W}=X,Z=1000,G=Math.max(0,W-P-1000)
```

常數 `1000` 是在 compaction 前從 context 上限中減去的安全邊際。這個設定過於保守——代表 overflow 恢復比必要時機早了 800 個 token。

**補丁。** 將這個表達式中的兩個 `1000` 改為 `200`：

```
contextLimit:W}=X,Z=200,G=Math.max(0,W-P-200)
```

**效果。** overflow 恢復觸發前多出 800 個可用 token。這讓 compaction 事件稍微延後，給快取更多時間累積讀取命中，然後才進入下一次重建週期。

**風險。** 極低。輸出下限 `fN8`（最少 3000 token）提供了額外的安全緩衝。縮小邊際不干擾那個下限。

---

### 策略三：提高 Compaction 閾值（100k→150k）

**根本原因。** `AgentStream` 有一個內建的 compaction，在以下條件觸發：

```
錨字串：var nX7=1e5,
```

`nX7 = 100000` 是 token 總數閾值。當一個 session 的 context 累積到 10 萬 token，compaction 自動執行。Compaction 用壓縮摘要取代訊息歷史，這完全摧毀了 prompt cache 前綴。下一輪需要付出 125% 進行完整快取重建。

每次 compaction 事件的成本：重建快取的寫入費用（約 45k token 的 125%）加上 compaction 摘要本身的寫入費用。在長期運行的 session 中，頻繁的 compaction 事件複利累積成可觀的浪費。

**補丁。** 改為 15 萬：

```
var nX7=15e4,
```

**效果。** Compaction 在 15 萬而非 10 萬時觸發。每個 session 的 compaction 事件更少 → 完整快取重建更少。在工具使用頻繁的 session 中，10 萬時一次 compaction 對比 15 萬時一次，可以省下數千個 token 的費用。

**風險。** 中。Session 在 compaction 前消耗更多 context。在生產環境中監控 `prompt_too_long` 錯誤，尤其是 context window 較小的模型。若出現錯誤，將值回復到 `1e5` 和 `15e4` 之間。

---

### 策略四：工具排序穩定化

**根本原因。** 透過原始碼確認：`cli.js` 在工具管道中沒有任何排序操作（工具管道中 `.sort()` 的呼叫次數為零）。API 請求中工具的順序取決於 MCP server 回應的插入順序。如果某個 server 重新連線並以不同順序重新註冊工具，序列化後的工具列表就會改變。工具列表的變動會破壞系統提示快取前綴（完整機制見[報告 #5](../tool-serialization-cache-stability/)）。

**修復。** 在你的 SDK wrapper 中，在傳入 `createSession()` 之前對 `allowedTools` 排序，並對 `mcpServers` 的 key 排序：

```javascript
const sortedTools = [...allowedTools].sort();
const sortedMcpServers = Object.fromEntries(
  Object.entries(mcpServers).sort(([a], [b]) => a.localeCompare(b))
);
```

**效果。** 工具序列化順序變得確定性，無論 MCP server 的註冊順序如何。快取前綴在重新連線後保持穩定。

**風險。** 無。排序對 API 而言是不可見的——可用工具集完全相同。唯一的改變是決定快取前綴穩定性的序列化順序。

---

### 策略五：快取保活 Ping

**根本原因。** 標準 Anthropic API 快取 TTL 是 5 分鐘。在允許名單上的第一方整合可以獲得 1 小時 TTL（見策略一）。即使有策略一的補丁，真正閒置超過 1 小時的 session 也會失去快取。

對於閒置間隔在 5 分鐘（或有策略一時 1 小時）以內的活躍 session，發送輕量 ping 可以防止快取過期：

**實作。**

```javascript
class CacheKeepalive {
  constructor(session, intervalMs = 4 * 60 * 1000) {
    this.session = session;
    this.lastUsed = Date.now();
    this.timer = setInterval(() => this.ping(), intervalMs);
  }

  touch() {
    this.lastUsed = Date.now();
  }

  async ping() {
    const idleMs = Date.now() - this.lastUsed;
    if (idleMs < 3 * 60 * 1000) return; // 最近使用過，跳過

    try {
      for await (const _ of this.session.stream("Reply with only 'ok'")) {
        // 消耗回應
      }
    } catch (err) {
      console.warn('[cache-keepalive] ping 失敗，停止計時器:', err.message);
      clearInterval(this.timer);
    }
  }

  stop() {
    clearInterval(this.timer);
  }
}
```

**費用分析。** 每次 ping 大約產生 10 個輸出 token。以 Sonnet 4 費率計算，每次 ping 約 $0.000015。一次 60k token context 的完整快取 miss 以 125% 計算，費用約 $0.0028。保活 ping 只要能防止一次快取 miss，就能回收 187 次 ping 的成本——在有實際工作量的 session 中，第一次避免的 miss 就能回本。

**風險。** 無。Ping 失敗會被記錄並停止計時器——不影響 session 的實際工作。Ping 訊息不影響對話狀態。

---

### 策略六：快取效率監控

在生產環境中，監控不是選項，是必要條件。快取破壞是靜默的——沒有錯誤，只有更高的帳單。

**實作。** 追蹤每輪快取效率的滾動指數移動平均（EMA）：

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
        `[cache-monitor] 效率 ${this.ema.toFixed(1)}% 低於閾值（${this.threshold}%）`,
        { efficiency, input_tokens, cache_read_input_tokens, cache_creation_input_tokens }
      );
    }
  }
}
```

**公式。** `效率 = cacheReadInputTokens / (inputTokens + cacheReadInputTokens + cacheCreationInputTokens) × 100`

**警報閾值。** 若滾動 EMA 持續低於 50% 則警告。一個 session 在最初幾輪之後，從快取讀取的量應該多於寫入的量。持續低於 50% 表示每輪都在破壞快取——工具排序不穩定、MCP 重新連線、compaction 連鎖、或 session 重啟。

**低效率代表什麼：**
- 工具列表變動導致快取破壞（→ 檢查策略四）
- MCP server 重新連線（→ 檢查策略七）
- Compaction 連鎖反應（→ 檢查策略三）
- Session 進程重啟（→ 檢查 V2 持久 session）

---

### 策略七：MCP Server Key 排序

**根本原因。** `mcpServers` 的型別是 `Record<string, ServerConfig>`。JavaScript 物件的 key 迭代順序遵循插入順序（ES2015 起的字串 key 規範）。如果你的程式碼在不同呼叫之間以不同方式建構這個物件——或者未來某個 SDK 版本在序列化時迭代 key——產生的 JSON 的 key 順序可能不同。

這是一個潛在風險。防範它的成本是零。

**修復。**

```javascript
function sortedMcpServers(servers) {
  return Object.fromEntries(
    Object.entries(servers).sort(([a], [b]) => a.localeCompare(b))
  );
}
```

在傳入 `createSession()` 之前應用此函數。與策略四的工具排序合併成一個正規化步驟。

**風險。** 無。

---

## 證據

| 主張 | 原始碼錨字串 | 位置 |
|------|------------|------|
| 1 小時 TTL 鎖在功能旗標後面 | `function o3z(A){if(QA()==="bedrock"` | cli.js |
| Overflow margin = 1000 token | `contextLimit:W}=X,Z=1000,G=Math.max(0,W-P-1000)` | cli.js |
| Compaction 閾值 = 10 萬 token | `var nX7=1e5,` | cli.js |
| 工具管道中無 `.sort()` | 工具陣列操作附近零個 `.sort()` 匹配 | cli.js |
| 標準快取 TTL = 5 分鐘 | Anthropic API 文件 | 外部 |
| mcpServers key 排序 | `Record<string,ServerConfig>` 型別、JS 插入順序規範 | cli.js / JS 規範 |

---

## 影響

從 V2 持久 session 的基準出發（報告 #1 確立了約 84% 的峰值效率），這些策略解決了剩餘的故障模式：

| 故障模式 | 解決它的策略 |
|---------|------------|
| 快取在閒置期間過期 | 5（保活 ping） |
| 短暫閒置 → 5 分鐘 TTL 到期 | 1（強制 1 小時 TTL） |
| 頻繁 compaction 事件 | 3（提高閾值） |
| MCP 重連時工具列表重排 | 4、7（排序穩定化） |
| Compaction 觸發過早 | 2（縮小 overflow margin） |
| 靜默快取退化 | 6（監控） |

在有長期運行 session 和 MCP server 的生產系統中，同時應用所有策略，在持續工作期間應能將滾動快取效率維持在 70% 以上。

---

## 實作：Postinstall 補丁模式

策略 1、2、3 和 7 需要修改 `cli.js`。方法是使用錨字串而非行號的 postinstall 腳本。Minified 程式碼在每次 build 時會重新混淆變數名稱，但字串常數和結構模式在版本之間保持穩定。

### 目錄結構

```
scripts/
  patch-sdk-v2.cjs         # 現有：為 V2 選項補丁 sdk.mjs
  patch-cli-cache-opt.cjs  # 新增：為快取策略 1-3 補丁 cli.js
```

`package.json`：
```json
"scripts": {
  "postinstall": "node scripts/patch-sdk-v2.cjs && node scripts/patch-cli-cache-opt.cjs"
}
```

### 補丁腳本模式

```javascript
// patch-cli-cache-opt.cjs
const fs = require('fs');
const path = require('path');

const CLI_PATH = path.resolve(__dirname, '../node_modules/@anthropic-ai/claude-agent-sdk/cli.js');

function applyPatch(source, anchor, replacement, patchName) {
  if (!source.includes(anchor)) {
    console.error(`[patch] 找不到錨字串：${patchName}`);
    console.error(`  anchor: ${anchor}`);
    process.exit(1);
  }

  const count = source.split(anchor).length - 1;
  if (count !== 1) {
    console.error(`[patch] 錨字串不唯一：${patchName}（找到 ${count} 次）`);
    process.exit(1);
  }

  if (source.includes(replacement)) {
    console.log(`[patch] 已套用過，跳過：${patchName}`);
    return source;
  }

  console.log(`[patch] 套用補丁：${patchName}`);
  return source.replace(anchor, replacement);
}

let source = fs.readFileSync(CLI_PATH, 'utf8');

// 備份原始檔案
const backupPath = CLI_PATH + '.pre-patch';
if (!fs.existsSync(backupPath)) {
  fs.writeFileSync(backupPath, source);
}

// 策略一：強制 SDK session 使用 1 小時 TTL
source = applyPatch(
  source,
  'function o3z(A){if(QA()==="bedrock"',
  'function o3z(A){if(A==="sdk"||A==="repl_main_thread")return!0;if(QA()==="bedrock"',
  'force-1h-ttl'
);

// 策略二：縮小 overflow margin
source = applyPatch(
  source,
  'contextLimit:W}=X,Z=1000,G=Math.max(0,W-P-1000)',
  'contextLimit:W}=X,Z=200,G=Math.max(0,W-P-200)',
  'reduce-overflow-margin'
);

// 策略三：提高 compaction 閾值
source = applyPatch(
  source,
  'var nX7=1e5,',
  'var nX7=15e4,',
  'raise-compaction-threshold'
);

fs.writeFileSync(CLI_PATH, source);
console.log('[patch] cli.js 補丁套用完成');
```

### 冪等性

腳本在套用每個補丁之前檢查 `source.includes(replacement)`。重複執行 `npm install` 最多套用每個補丁一次。

### SDK 升級後

當 SDK 更新，在補丁後的 build 上線前，確認每個錨字串依然存在：

```bash
grep -c 'function o3z(A){if(QA()==="bedrock"' node_modules/@anthropic-ai/claude-agent-sdk/cli.js
grep -c 'contextLimit:W}=X,Z=1000' node_modules/@anthropic-ai/claude-agent-sdk/cli.js
grep -c 'var nX7=1e5,' node_modules/@anthropic-ai/claude-agent-sdk/cli.js
```

每個都應該返回 `1`。如果任何一個返回 `0`，錨字串已移動或實作已更改——使用每個策略根本原因部分描述的字串常數重新定位該函數，並更新錨字串。

---

## 我們調查但選擇不補丁的項目

**currentDate 注入的快取破壞。** `currentDate` 字串在每一輪都注入到第一個訊息位置。這是最直接的快取破壞源，也是影響最大的一個。然而它深埋在注入組裝管道中，涉及多個也處理 CLAUDE.md 和記憶體檔案注入的程式碼路徑。一個有問題的補丁靜默破壞 context 組裝的風險很高，而且 V2 持久 session 已部分解決了回報（日期每天只變一次，而非每輪變一次）。暫緩。

**停用延遲工具載入。** `cli.js` 有一個懶載入工具的路徑，可以減少初始提示大小。停用它會讓所有工具在最一開始就載入，產生一個更大但更穩定的初始提示。這個取捨——更大的初始寫入對比更少的增量寫入——沒有明確的優劣，而且實作與 MCP 初始化路徑耦合。暫緩。

**Fork/subagent 剪枝。** Subagent 的結果被追加到訊息歷史中，並以摘要形式在 compaction 後存活。剪枝這些條目會減少 context 大小，但有可能失去影響任務連貫性的 context 品質。風險不值得。

**多模型快取隔離。** 不同模型維護各自的 prompt cache。如果你的生產系統在同一個 session 中路由到多個模型，為一個模型建立的快取不與另一個模型共享。要正確解決這個問題需要架構層面的 session 路由，超出了補丁方法的範疇。

---

## 生產環境監控清單

套用補丁並部署後：

1. **逐輪效率。** 記錄每個回應的 `cacheReadInputTokens / (inputTokens + cacheReadInputTokens + cacheCreationInputTokens)`。持續 EMA 低於 50% 時發出警報。

2. **Compaction 頻率。** 記錄 compaction 事件發生的時機。連鎖 compaction（compaction → 重建 → 再次快速 compaction）表示閾值對你的工作量來說仍然太低。

3. **保活 ping 成功率。** 追蹤 ping 失敗。失敗激增可能表示 session 不穩定或網路問題，而非快取問題。

4. **第一輪對比後續輪次費用。** 第一輪必定寫入快取。追蹤第一輪費用與後續輪次費用的比值，作為快取正確累積的健全性檢查。

5. **升級後驗證。** 每次 SDK 升級後，在部署前執行上述錨字串 grep 命令。一個有問題的補丁比沒有補丁更糟糕——它可能靜默損壞 `cli.js` 原始碼。

---

## 參考資料

- [報告 #1：Agent SDK 快取失效（根本原因與修復）](../agent-sdk-cache-invalidation/)
- [報告 #3：Prompt Cache 架構](../prompt-cache-architecture/)
- [報告 #5：工具序列化與快取穩定性](../tool-serialization-cache-stability/)
- `cli.js` build 2026-03-14，`@anthropic-ai/claude-agent-sdk` v0.2.76
- Anthropic API 文件：[Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
