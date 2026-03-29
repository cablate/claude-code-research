# Claude Code CLI 逆向工程指南

## cli.js 的兩個來源

Claude Code 的核心邏輯打包在一個 ~12MB 的 minified JavaScript 檔案 `cli.js` 裡。這個檔案出現在兩個 npm 套件中：

| 套件 | 用途 | cli.js 位置 |
|------|------|------------|
| `@anthropic-ai/claude-code` | 獨立 CLI 工具（使用者直接 `npm install -g` 的） | `node_modules/@anthropic-ai/claude-code/cli.js` |
| `@anthropic-ai/claude-agent-sdk` | Agent SDK（程式碼中用 `createSession()` 呼叫的） | `node_modules/@anthropic-ai/claude-agent-sdk/cli.js` |

**兩者是同一個檔案的不同版本。** SDK 打包的 cli.js 可能比最新的 CLI 套件落後幾個版本。例如 SDK v0.2.76 打包的 cli.js build date 是 2026-03-14，而 CLI v2.1.85 的 build date 是 2026-03-26。

逆向研究時可以用任一來源。要做 postinstall patch 時，目標是你專案實際用的那個（通常是 SDK 的）。

## 安裝研究用環境

```bash
# 方式一：安裝獨立 CLI（取得最新版）
mkdir /tmp/cc-research && cd /tmp/cc-research
npm init -y
npm install @anthropic-ai/claude-code
# 目標：node_modules/@anthropic-ai/claude-code/cli.js

# 方式二：安裝 Agent SDK（跟你的專案同版本）
npm install @anthropic-ai/claude-agent-sdk
# 目標：node_modules/@anthropic-ai/claude-agent-sdk/cli.js

# 方式三：直接看專案裡的（如果你要 patch 的話）
# 目標：your-project/node_modules/@anthropic-ai/claude-agent-sdk/cli.js
```

確認版本：
```bash
grep -o "VERSION:\"[^\"]*\"" cli.js | head -1
grep -o "BUILD_TIME:\"[^\"]*\"" cli.js | head -1
```

## 搜尋方法

函式名稱被壓縮（每個版本不同），但以下內容跨版本穩定：
- **字串常數**：telemetry event 名稱、error messages、環境變數名稱
- **API 欄位名**：`cache_control`、`defer_loading`、`tool_reference`
- **Feature flag 名稱**：`tengu_` 開頭的 flag
- **內嵌文件**：如 "Render order is: tools → system → messages"

用這些作為定位錨點，再順藤摸瓜找到周圍的函式邏輯。

```bash
# 用 grep -b 取得 byte offset（比 -n 行號有用，因為整個檔案幾乎是一行）
grep -bo "cache_control" cli.js | head -20

# 用 grep -P（Perl regex）搜尋附近的上下文
grep -oP ".{0,100}cache_control.{0,100}" cli.js | head -10

# 讀取特定 offset 附近的內容（用 dd 或 tail -c +offset | head -c length）
dd if=cli.js bs=1 skip=5295400 count=200 2>/dev/null
```

## 常用搜尋 pattern

```bash
# === 快取機制 ===
grep -bo "cache_control" cli.js | head -20
grep -bo "tengu_sysprompt" cli.js
grep -bo "__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__" cli.js
grep -bo "skipGlobalCacheForSystemPrompt" cli.js
grep -bo "cacheScope" cli.js | head -10
grep -bo "ephemeral" cli.js | head -10

# === 工具與延遲載入 ===
grep -bo "defer_loading" cli.js | head -20
grep -bo "tool_reference" cli.js | head -10
grep -bo "isMcp" cli.js | head -10
grep -bo "ToolSearch" cli.js | head -10

# === Beta headers ===
grep -bo "prompt-caching-scope" cli.js
grep -bo "context-management" cli.js
grep -bo "advanced-tool-use" cli.js
grep -bo "fast-mode" cli.js

# === 內嵌文件（非常有價值）===
grep -bo "Render order is" cli.js
grep -bo "Don't change tools" cli.js

# === Compact ===
grep -bo "preCompactTokenCount" cli.js
grep -oP ".{0,50}querySource.{0,10}compact.{0,50}" cli.js

# === Streaming ===
grep -bo "CLAUDE_ENABLE_STREAM_WATCHDOG" cli.js
grep -bo "CLAUDE_STREAM_IDLE_TIMEOUT_MS" cli.js
grep -bo "content_block_delta" cli.js

# === 記憶體 ===
grep -bo "memoryUsage" cli.js | head -5
grep -bo "heapUsed" cli.js | head -5

# === 日期注入 ===
grep -bo "currentDate" cli.js
grep -bo "date_change" cli.js
grep -oP ".{0,30}Today.{0,30}date.{0,30}" cli.js

# === Auto-mode classifier ===
grep -bo "auto_mode" cli.js | head -10
grep -oP ".{0,30}classifier.{0,30}" cli.js | head -5

# === 環境變數（可控的旋鈕）===
grep -oP "process\.env\.[A-Z_]+" cli.js | sort -u
```

## 關鍵函式定位表

minified 名稱每版不同，但定位方式跨版本穩定。

| 功能 | 定位方式 |
|------|---------|
| cache_control 工廠 | `grep "type:\"ephemeral\""` 附近的函式定義 |
| 系統提示快取策略 | `grep "skipGlobalCacheForSystemPrompt"` |
| 工具 schema 建構器 | `grep "defer_loading"` 附近有 `input_schema` 的函式 |
| 訊息快取斷點 | `grep "skipCacheWrite"` |
| 延遲工具判斷 | `grep "isMcp.*true"` 附近有 `shouldDefer` |
| 發現掃描器 | `grep "tool_reference"` 附近有 `new Set` |
| Tool search 模式 | `grep "tst-auto"` |
| 日期產生器 | `grep "getFullYear.*getMonth.*getDate"` |
| userContext 建構器 | `grep "currentDate.*Today"` |
| system-reminder 注入 | `grep "system-reminder.*context"` |
| date_change 偵測 | `grep "date_change.*newDate"` |
| compact 執行器 | `grep "preCompactTokenCount"` |
| auto-mode classifier | `grep "querySource.*auto_mode"` |
| 動態邊界標記 | `grep "SYSTEM_PROMPT_DYNAMIC_BOUNDARY"` |

## 慣例與 pattern

| Pattern | 含義 |
|---------|------|
| `z1(fn)` | memoize/once 包裝器，fn 只執行一次 |
| `y(()=>{...})` | esbuild module lazy init（bundler 產生的，非 app-level） |
| `g8("tengu_xxx", false)` | Feature flag 讀取（GrowthBook），第二參數是 default |
| `d("tengu_xxx", {...})` | Telemetry event 發送 |
| `o6(process.env.XXX)` | 環境變數 truthy 判斷 |
| `G7()` | Provider 判斷（"firstParty" / "bedrock" / "vertex"） |
| `rM()` | 是否指向 api.anthropic.com |
| `S4()` | 是否為 internal/staff 使用者 |

## 注意事項

- cli.js 經常是一行或幾行，`grep -n` 的行號意義不大，用 `grep -b` 的 byte offset
- 不同版本的 minified 名稱**一定會不同**，不要硬記名稱，靠字串常數定位
- 做 patch 時要先確認 find pattern 在目標 cli.js 中**唯一**（`grep -c` 確認 count=1）
- SDK 的 sdk.mjs 也是 minified 的，同樣方法適用
