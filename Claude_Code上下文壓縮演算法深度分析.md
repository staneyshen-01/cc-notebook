# Claude Code 上下文壓縮演算法深度分析

> 基於 Claude Code 原始碼（2026-03-31 快照，512K 行 TypeScript）逆向分析
> 核心檔案：`src/services/compact/` 目錄下 11 個檔案

---

## 目錄

1. [架構總覽](#1-架構總覽)
2. [第1層：微壓縮（Microcompact）](#2-第1層微壓縮microcompact)
3. [第2層：自動壓縮（Auto-Compact）](#3-第2層自動壓縮auto-compact)
4. [第3層：傳統壓縮（Full Compact）](#4-第3層傳統壓縮full-compact)
5. [第4層：Session Memory 壓縮](#5-第4層session-memory-壓縮)
6. [訊息分組演算法](#6-訊息分組演算法)
7. [Token 估算演算法](#7-token-估算演算法)
8. [壓縮提示詞工程](#8-壓縮提示詞工程)
9. [5層錯誤恢復中的壓縮角色](#9-5層錯誤恢復中的壓縮角色)
10. [各層對比與設計哲學](#10-各層對比與設計哲學)

---

## 1. 架構總覽

Claude Code 的上下文壓縮不是單一演算法，而是一個 **4層遞進的壓縮體系**，每層解決不同層面的問題：

```
使用者訊息 → [第1層：微壓縮] → [第2層：自動壓縮] → API 呼叫
              ↓                    ↓
         細粒度清理舊工具輸出    上下文即將超限時觸發
         (不丟語義，<1ms)       (呼叫 LLM 或 Session Memory)
                                   ↓
                    ┌──────────────┴──────────────┐
              [第4層：SM壓縮]              [第3層：傳統壓縮]
              (用已有摘要，<10ms)          (Fork Agent 生成摘要，5-30s)
```

**核心原則：** 儘可能用廉價的規則操作延遲昂貴的 LLM 呼叫，只在不得已時丟棄資訊。

### 涉及的原始檔

| 檔案 | 行數 | 職責 |
|------|------|------|
| `microCompact.ts` | ~400 | 微壓縮：規則清理舊工具結果 |
| `apiMicrocompact.ts` | — | API 層快取編輯整合 |
| `timeBasedMCConfig.ts` | — | 時間觸發微壓縮配置 |
| `autoCompact.ts` | ~350 | 自動壓縮：閾值判斷 + 斷路器 |
| `compact.ts` | ~600+ | 傳統壓縮：Fork Agent 摘要 |
| `prompt.ts` | ~375 | 壓縮提示詞模板 |
| `sessionMemoryCompact.ts` | ~630 | Session Memory 壓縮路徑 |
| `grouping.ts` | ~63 | 訊息按 API 輪次分組 |
| `postCompactCleanup.ts` | — | 壓縮後清理 |
| `compactWarningHook.ts` | — | 壓縮警告鉤子 |
| `compactWarningState.ts` | — | 壓縮警告狀態 |

---

## 2. 第1層：微壓縮（Microcompact）

**原始檔：** `microCompact.ts`

### 核心思想

不呼叫 LLM，純規則操作——清理舊的、大塊的工具輸出結果，保留語義資訊。這是每輪查詢前都會執行的最輕量操作。

### 可壓縮工具白名單

```typescript
const COMPACTABLE_TOOLS = new Set([
  'Read',      // 檔案讀取結果可能很大
  'Bash',      // Shell 輸出可能很長
  'Grep',      // 搜尋結果
  'Glob',      // 檔案列表
  'WebSearch', // 網頁搜尋結果
  'WebFetch',  // 網頁抓取結果
  'Edit',      // 檔案編輯的 diff
  'Write',     // 檔案寫入確認
])
```

不在白名單中的工具（如 Agent、Skill、MCP 等）的結果不會被微壓縮。

### 兩個子路徑

#### 子路徑 A：時間觸發微壓縮

```
觸發條件：距上次助手訊息的時間間隔超過閾值（API 快取已過期）

執行邏輯：
1. 收集所有可壓縮工具的 tool_use ID
2. 保留最近 N 個工具結果
3. 將更早的 tool_result 內容替換為：
   "[Old tool result content cleared]"
4. 不修改 tool_use 塊（保持 API 配對完整性）

特點：
- 快取已過期，所以無需保護快取
- 直接修改本地訊息內容
- 減少重傳時的 token 消耗
```

#### 子路徑 B：快取編輯微壓縮（Cached MC）

這是更精巧的路徑——在快取仍然有效時工作：

```
觸發條件：
- 特性開關 CACHED_MICROCOMPACT 開啟
- 模型支援快取編輯 API
- 當前是主執行緒查詢（非 fork agent）

執行邏輯：
1. collectCompactableToolIds(): 收集所有可壓縮的 tool_use ID
2. registerToolResult(): 註冊每個工具結果（按使用者訊息分組）
3. registerToolMessage(): 記錄工具訊息組
4. getToolResultsToDelete(): 根據 count/keep 閾值決定刪除哪些
5. createCacheEditsBlock(): 生成 cache_edits API 塊

關鍵區別：
- 不修改本地訊息內容！
- 透過 API 的 cache_edits 欄位告訴服務端刪除特定工具結果的快取
- 保持 prompt cache 命中率
- 狀態透過 pendingCacheEdits / pinnedCacheEdits 管理
```

**快取編輯的狀態管理：**

```typescript
// 全域性狀態
let cachedMCState: CachedMCState | null = null
let pendingCacheEdits: CacheEditsBlock | null = null

// consumePendingCacheEdits():
//   返回待插入的快取編輯塊，然後清空 pending 狀態
//   呼叫者在 API 請求後必須呼叫 pinCacheEdits() 固定它們

// getPinnedCacheEdits():
//   返回之前已固定的快取編輯，需要在後續請求中重新傳送

// markToolsSentToAPIState():
//   標記工具已傳送給 API（成功響應後呼叫）
```

### Token 估算輔助

```typescript
function calculateToolResultTokens(block: ToolResultBlockParam): number {
  if (typeof block.content === 'string') {
    return roughTokenCountEstimation(block.content)  // 字元數 / 4
  }
  // 陣列：逐項計算
  return block.content.reduce((sum, item) => {
    if (item.type === 'text') return sum + roughTokenCountEstimation(item.text)
    if (item.type === 'image' || item.type === 'document') return sum + 2000
    return sum
  }, 0)
}

function estimateMessageTokens(messages: Message[]): number {
  let totalTokens = 0
  for (const message of messages) {
    for (const block of message.message.content) {
      switch (block.type) {
        case 'text': totalTokens += roughTokenCountEstimation(block.text); break
        case 'tool_result': totalTokens += calculateToolResultTokens(block); break
        case 'image':
        case 'document': totalTokens += 2000; break  // 固定估算
        case 'thinking': totalTokens += roughTokenCountEstimation(block.thinking); break
        case 'redacted_thinking': totalTokens += roughTokenCountEstimation(block.data); break
        case 'tool_use': totalTokens += roughTokenCountEstimation(block.name + JSON.stringify(block.input ?? {})); break
        default: totalTokens += roughTokenCountEstimation(JSON.stringify(block)); break
      }
    }
  }
  return Math.ceil(totalTokens * (4 / 3))  // × 4/3 保守填充
}
```

---

## 3. 第2層：自動壓縮（Auto-Compact）

**原始檔：** `autoCompact.ts`

### 閾值計算

```typescript
// 有效上下文視窗 = 模型上下文視窗 - 摘要輸出預留
const MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000  // p99.99 的摘要輸出是 17,387 tokens

function getEffectiveContextWindowSize(model: string): number {
  const reservedTokensForSummary = Math.min(
    getMaxOutputTokensForModel(model),
    MAX_OUTPUT_TOKENS_FOR_SUMMARY,
  )
  let contextWindow = getContextWindowForModel(model)
  // 支援環境變數覆蓋（用於測試）
  if (process.env.CLAUDE_CODE_AUTO_COMPACT_WINDOW) {
    contextWindow = Math.min(contextWindow, parsed)
  }
  return contextWindow - reservedTokensForSummary
}

// 自動壓縮閾值 = 有效上下文視窗 - 13K 緩衝
const AUTOCOMPACT_BUFFER_TOKENS = 13_000

function getAutoCompactThreshold(model: string): number {
  return getEffectiveContextWindowSize(model) - AUTOCOMPACT_BUFFER_TOKENS
}

// 舉例（Opus 200K 上下文）：
// 有效視窗 = 200,000 - 20,000 = 180,000
// 自動壓縮閾值 = 180,000 - 13,000 = 167,000 tokens
```

### 其他閾值

```typescript
const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000   // 警告閾值
const ERROR_THRESHOLD_BUFFER_TOKENS = 20_000     // 錯誤閾值
const MANUAL_COMPACT_BUFFER_TOKENS = 3_000       // 手動 /compact 的阻塞限制
```

### 斷路器機制

```typescript
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3

// BQ 資料分析發現：1,279 個會話連續失敗 50+ 次（最多 3,272 次），
// 每天浪費約 250K 次 API 呼叫。所以加了斷路器。

if (tracking.consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
  return { wasCompacted: false }  // 直接跳過，不再嘗試
}
```

### shouldAutoCompact 決策樹

```
shouldAutoCompact(messages, model, querySource):

1. querySource 是 'session_memory' 或 'compact'？
   → 返回 false（防止遞迴死鎖——壓縮代理自己不能觸發壓縮）

2. querySource 是 'marble_origami'（上下文摺疊代理）？
   → 返回 false（防止破壞主執行緒的已提交日誌）

3. isAutoCompactEnabled() 返回 false？
   → 返回 false
   （檢查 DISABLE_COMPACT、DISABLE_AUTO_COMPACT 環境變數和使用者配置）

4. 響應式壓縮模式開啟？（tengu_cobalt_raccoon gate）
   → 返回 false（讓 API 的 prompt-too-long 錯誤觸發響應式壓縮）

5. 上下文摺疊模式開啟？
   → 返回 false（上下文摺疊是 90%/95% 流程，自動壓縮在 93% 會干擾它）

6. tokenCountWithEstimation(messages) - snipTokensFreed >= threshold？
   → 返回 true
```

### autoCompactIfNeeded 執行流程

```
autoCompactIfNeeded(messages, context, ...):

1. DISABLE_COMPACT 環境變數？→ 跳過

2. 斷路器檢查：consecutiveFailures >= 3？→ 跳過

3. shouldAutoCompact() 返回 true？
   ↓ 是
4. 優先嚐試 Session Memory 壓縮
   ↓ 成功 → 返回結果
   ↓ 失敗

5. 回退到傳統壓縮（compactConversation）
   ↓ 成功 → 重置 consecutiveFailures = 0，返回結果
   ↓ 失敗

6. consecutiveFailures++
   ↓ 達到 3 次
7. 日誌：斷路器觸發，本次會話不再嘗試自動壓縮
```

---

## 4. 第3層：傳統壓縮（Full Compact）

**原始檔：** `compact.ts` + `prompt.ts`

### 核心機制：Fork Agent

傳統壓縮使用一個 **Fork Agent**——建立當前會話的一個分支，讓它生成摘要。關鍵優勢是**共享主會話的 prompt cache**。

```
主會話訊息: [user1, assistant1, user2, assistant2, ...]
                ↓ 全部傳入
Fork Agent（共享 prompt cache）
                ↓ 單輪迴復
生成結構化摘要（<analysis> + <summary>）
                ↓
後處理 → 替換原訊息
```

### 預處理管線

```
原始訊息
  ↓
stripImagesFromMessages()    ← 圖片 → "[image]"，文件 → "[document]"
  ↓                            （防止壓縮請求自身超過上下文限制）
stripReinjectedAttachments() ← 刪除技能發現/列表附件
  ↓                            （壓縮後會自動重新注入）
normalizeMessagesForAPI()    ← 規範化訊息格式
  ↓
傳送給 Fork Agent
```

### 摘要輸出格式

Fork Agent 被要求生成兩個 XML 塊：

```xml
<analysis>
[思考草稿——用於提高摘要質量的中間推理過程]
[這部分最終會被刪除，不會進入壓縮後的上下文]
</analysis>

<summary>
1. Primary Request and Intent:
   [詳細描述使用者的所有請求和意圖]

2. Key Technical Concepts:
   - [概念1]
   - [概念2]

3. Files and Code Sections:
   - [檔名1]
     - [為什麼這個檔案重要]
     - [程式碼片段]
   - [檔名2]
     - [程式碼片段]

4. Errors and fixes:
   - [錯誤描述]:
     - [修復方式]
     - [使用者反饋]

5. Problem Solving:
   [問題解決過程]

6. All user messages:
   - [逐條列出所有非工具結果的使用者訊息]

7. Pending Tasks:
   - [待辦事項1]
   - [待辦事項2]

8. Current Work:
   [精確描述當前工作內容，包含檔名和程式碼片段]

9. Optional Next Step:
   [下一步計劃，包含最近對話的直接引用]
</summary>
```

### 防止工具呼叫的強力前導詞

```typescript
const NO_TOOLS_PREAMBLE = `CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.

- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- You already have all the context you need in the conversation above.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
- Your entire response must be plain text: an <analysis> block followed by a <summary> block.
`
```

為什麼需要這麼強力？註釋解釋了：

```
// Sonnet 4.6+ 自適應思考模型有時會忽略較弱的尾部指令並嘗試呼叫工具。
// 在 maxTurns: 1 的情況下，被拒絕的工具呼叫意味著沒有文字輸出
// → 回退到流式備用路徑（4.6 上 2.79% vs 4.5 上 0.01%）。
// 把這個放在最前面並明確說明拒絕後果，可以防止浪費輪次。
```

### 後處理（formatCompactSummary）

```typescript
function formatCompactSummary(summary: string): string {
  // 1. 刪除 <analysis> 塊（草稿，已無價值）
  summary = summary.replace(/<analysis>[\s\S]*?<\/analysis>/, '')

  // 2. 提取 <summary> 內容，替換為可讀標題
  const match = summary.match(/<summary>([\s\S]*?)<\/summary>/)
  if (match) {
    summary = summary.replace(/<summary>[\s\S]*?<\/summary>/,
      `Summary:\n${match[1].trim()}`)
  }

  // 3. 清理多餘空行
  summary = summary.replace(/\n\n+/g, '\n\n')
  return summary.trim()
}
```

### 壓縮後訊息序列

```
[CompactBoundaryMessage]     ← 標記壓縮邊界（含 token 統計、trigger 型別）
[SummaryUserMessage]         ← 格式化後的摘要
[messagesToKeep]             ← 保留的最近訊息（如果有）
[Attachments]                ← 重新注入的附件
  - 最近讀取的檔案（前 5 個，每個 ≤ 5K tokens，總預算 50K）
  - Plan 檔案（如果有活躍計劃）
  - MCP 指令增量
  - 技能發現增量
  - 代理列表增量
[HookResults]                ← PreCompact/PostCompact 鉤子結果
```

### Prompt-Too-Long 重試機制

當壓縮請求本身超過上下文限制時：

```typescript
const MAX_PTL_RETRIES = 3

function truncateHeadForPTLRetry(messages, ptlResponse): Message[] | null {
  // 1. 移除上一次重試的標記訊息（防止重試停滯）
  // 2. 按 API 輪次分組
  const groups = groupMessagesByApiRound(messages)
  if (groups.length < 2) return null  // 無法再裁剪

  // 3. 決定刪除多少組
  const tokenGap = getPromptTooLongTokenGap(ptlResponse)
  if (tokenGap !== undefined) {
    // 能解析出 token 缺口 → 精確刪除
    let acc = 0, dropCount = 0
    for (const g of groups) {
      acc += roughTokenCountEstimationForMessages(g)
      dropCount++
      if (acc >= tokenGap) break
    }
  } else {
    // 不能解析 → 刪除 20% 最舊的組
    dropCount = Math.max(1, Math.floor(groups.length * 0.2))
  }

  // 4. 保證至少保留 1 組
  dropCount = Math.min(dropCount, groups.length - 1)

  // 5. 如果裁剪後第一條是 assistant 訊息，補一個 user marker
  //    （API 要求第一條訊息必須是 user）
  const sliced = groups.slice(dropCount).flat()
  if (sliced[0]?.type === 'assistant') {
    return [createUserMessage({ content: PTL_RETRY_MARKER, isMeta: true }), ...sliced]
  }
  return sliced
}
```

### 部分壓縮（Partial Compact）

支援兩個方向：

| 方向 | 提示詞 | 用途 |
|------|--------|------|
| `from`（預設） | `PARTIAL_COMPACT_PROMPT` | 保留舊訊息，僅摘要"最近的訊息" |
| `up_to` | `PARTIAL_COMPACT_UP_TO_PROMPT` | 摘要舊訊息，保留新訊息。摘要會放在開頭，後續訊息跟在後面 |

`up_to` 模式的摘要包含一個特殊的第 9 章節 "Context for Continuing Work"，專門為後續訊息提供上下文。

---

## 5. 第4層：Session Memory 壓縮

**原始檔：** `sessionMemoryCompact.ts`

### 核心思想

不呼叫 LLM 生成新摘要，而是直接使用已經透過後臺記憶提取（`extractMemories`）積累的 Session Memory 作為"摘要"。

### 優勢

- **速度快**：不需要 API 呼叫，<10ms
- **質量可預測**：Session Memory 是在每輪查詢後漸進更新的，不是一次性壓縮
- **保留近期訊息**：不像傳統壓縮那樣替換所有訊息

### 配置引數

```typescript
const DEFAULT_SM_COMPACT_CONFIG = {
  minTokens: 10_000,           // 至少保留 10K tokens 的最近訊息
  minTextBlockMessages: 5,      // 至少保留 5 條含文字的訊息
  maxTokens: 40_000,           // 最多保留 40K tokens（硬上限）
}
// 這些值透過 GrowthBook 遠端配置，可動態調整
```

### calculateMessagesToKeepIndex 演算法

這是 Session Memory 壓縮的核心演算法——決定保留哪些最近的訊息：

```
輸入：messages[], lastSummarizedIndex（Session Memory 已覆蓋到哪條訊息）

演算法：
1. startIndex = lastSummarizedIndex + 1
   （即：從 Session Memory 尚未覆蓋的訊息開始）

2. 計算當前 [startIndex, end] 範圍的 token 總量和含文字訊息數

3. 如果已經超過 maxTokens (40K) → 直接返回（不再擴充套件）

4. 如果同時滿足 ≥ minTokens (10K) AND ≥ minTextBlockMessages (5)
   → 直接返回（已足夠）

5. 否則，從 startIndex 往前逐條擴充套件：
   - 每加入一條訊息，更新 token 和訊息計數
   - 停止條件：
     a. 達到 maxTokens (40K)
     b. 同時滿足 minTokens 和 minTextBlockMessages
     c. 到達上一個 CompactBoundary（不跨越舊的壓縮邊界）

6. adjustIndexToPreserveAPIInvariants(messages, startIndex)
   → 確保不切斷 tool_use/tool_result 配對
   → 確保不分離共享 message.id 的 thinking 塊
```

### adjustIndexToPreserveAPIInvariants 演算法

這個演算法解決一個棘手的問題——流式傳輸時，一個 API 響應會產生多條訊息（thinking、tool_use 等），它們共享同一個 `message.id`。如果在中間切斷，`normalizeMessagesForAPI` 合併時會丟失 thinking 塊。

```
輸入：messages[], startIndex

步驟 1：修復 tool_use/tool_result 配對
  1a. 收集 [startIndex, end] 範圍內所有 tool_result 的 tool_use_id
  1b. 收集範圍內已有的 tool_use_id
  1c. 找出缺失的 tool_use_id（在範圍外）
  1d. 向前搜尋，把包含缺失 tool_use 的 assistant 訊息納入範圍

步驟 2：修復 thinking 塊分離
  2a. 收集範圍內所有 assistant 訊息的 message.id
  2b. 向前搜尋，把共享同一 message.id 的 assistant 訊息納入範圍

返回：調整後的 startIndex
```

**原始碼註釋中的真實 bug 場景：**

```
Session 儲存（壓縮前）：
  Index N:   assistant, message.id: X, content: [thinking]
  Index N+1: assistant, message.id: X, content: [tool_use: ORPHAN_ID]
  Index N+2: assistant, message.id: X, content: [tool_use: VALID_ID]
  Index N+3: user, content: [tool_result: ORPHAN_ID, tool_result: VALID_ID]

如果 startIndex = N+2：
  舊程式碼：只檢查 N+2 的 tool_results，找不到，返回 N+2
  normalizeMessagesForAPI 合併後：
    msg[1]: assistant with [tool_use: VALID_ID]  ← ORPHAN tool_use 被排除！
    msg[2]: user with [tool_result: ORPHAN_ID, tool_result: VALID_ID]
  API 報錯：孤立的 tool_result 引用了不存在的 tool_use
```

### trySessionMemoryCompaction 完整流程

```
trySessionMemoryCompaction(messages, agentId, autoCompactThreshold):

1. shouldUseSessionMemoryCompaction()？
   → 檢查 tengu_session_memory AND tengu_sm_compact 特性開關
   → 支援 ENABLE_CLAUDE_CODE_SM_COMPACT / DISABLE_CLAUDE_CODE_SM_COMPACT 環境變數

2. 初始化遠端配置（僅首次）

3. 等待正在進行的 Session Memory 提取完成（帶超時）

4. 獲取 lastSummarizedMessageId 和 sessionMemory 內容

5. Session Memory 檔案不存在？→ 返回 null
6. Session Memory 是空模板？→ 返回 null

7. 確定 lastSummarizedIndex：
   a. 正常情況：在 messages 中查詢 lastSummarizedMessageId 的索引
   b. 恢復的會話：設為 messages.length - 1（從末尾開始）

8. calculateMessagesToKeepIndex() → startIndex
9. 過濾掉 messagesToKeep 中的舊 CompactBoundary

10. 執行 SessionStart 鉤子（恢復 CLAUDE.md 等上下文）

11. 建立壓縮結果：
    - 截斷過大的 Session Memory 章節
    - 生成摘要使用者訊息
    - 附加 Plan 檔案（如果有）

12. 檢查壓縮後 token 是否仍超過閾值
    → 是：返回 null（回退到傳統壓縮）
    → 否：返回 CompactionResult
```

---

## 6. 訊息分組演算法

**原始檔：** `grouping.ts`

```typescript
function groupMessagesByApiRound(messages: Message[]): Message[][] {
  const groups: Message[][] = []
  let current: Message[] = []
  let lastAssistantId: string | undefined

  for (const msg of messages) {
    // 當出現新的 assistant message.id 時，開始新的一組
    if (
      msg.type === 'assistant' &&
      msg.message.id !== lastAssistantId &&
      current.length > 0
    ) {
      groups.push(current)
      current = [msg]
    } else {
      current.push(msg)
    }
    if (msg.type === 'assistant') {
      lastAssistantId = msg.message.id
    }
  }

  if (current.length > 0) {
    groups.push(current)
  }
  return groups
}
```

**設計細節：**

- 同一 API 請求的流式塊共享同一個 `message.id`
- `StreamingToolExecutor` 在流式輸出期間交錯插入 `tool_result`，但它們屬於同一輪次
- 只要 `message.id` 不變，所有訊息都在同一組內
- 不跟蹤未解決的 `tool_use` ID——讓分組邊界自然形成，由 `ensureToolResultPairing` 在 API 層修復殘留的配對問題

---

## 7. Token 估算演算法

貫穿所有壓縮路徑的核心輔助：

### 粗略估算

```typescript
function roughTokenCountEstimation(text: string): number {
  return text.length / 4  // 經驗法則：4 字元 ≈ 1 token
}
```

### 訊息級估算

```typescript
function estimateMessageTokens(messages: Message[]): number {
  let totalTokens = 0
  for (const message of messages) {
    if (message.type !== 'user' && message.type !== 'assistant') continue
    for (const block of message.message.content) {
      switch (block.type) {
        case 'text':
          totalTokens += roughTokenCountEstimation(block.text)
          break
        case 'tool_result':
          totalTokens += calculateToolResultTokens(block)
          break
        case 'image':
        case 'document':
          totalTokens += 2000  // 固定值，不論實際大小
          break
        case 'thinking':
          totalTokens += roughTokenCountEstimation(block.thinking)
          // 注意：不計算簽名（signature 是後設資料，不被模型 tokenize）
          break
        case 'redacted_thinking':
          totalTokens += roughTokenCountEstimation(block.data)
          break
        case 'tool_use':
          totalTokens += roughTokenCountEstimation(
            block.name + JSON.stringify(block.input ?? {})
          )
          // 不計算 JSON wrapper 和 id 欄位
          break
        default:
          totalTokens += roughTokenCountEstimation(JSON.stringify(block))
          break
      }
    }
  }
  return Math.ceil(totalTokens * (4 / 3))  // 保守填充 33%
}
```

### 精確計算

```typescript
// 當有 API 響應的 usage 資料時，使用精確值
function tokenCountFromLastAPIResponse(messages: Message[]): number | undefined {
  // 從最後一條 assistant 訊息的 usage 欄位讀取
  // usage.input_tokens 包含了 API 看到的實際 token 數
}

// 混合策略
function tokenCountWithEstimation(messages: Message[]): number {
  // 優先使用 API 返回的精確值
  // 不可用時回退到估算
}
```

---

## 8. 壓縮提示詞工程

**原始檔：** `prompt.ts`

### 三種提示詞模板

| 模板 | 變數名 | 用途 |
|------|--------|------|
| 完整壓縮 | `BASE_COMPACT_PROMPT` | 摘要整個對話 |
| 部分壓縮（from） | `PARTIAL_COMPACT_PROMPT` | 只摘要最近的訊息 |
| 部分壓縮（up_to） | `PARTIAL_COMPACT_UP_TO_PROMPT` | 摘要舊訊息，作為後續訊息的前導 |

### 提示詞結構

```
[NO_TOOLS_PREAMBLE]        ← 強力禁止工具呼叫
[DETAILED_ANALYSIS_INSTRUCTION]  ← 要求 <analysis> 思考草稿
[MAIN_PROMPT]              ← 9 章節結構化摘要要求 + 示例
[Custom Instructions]      ← 使用者自定義指令（如果有）
[NO_TOOLS_TRAILER]         ← 再次強調不要呼叫工具
```

### <analysis> 塊的作用

```
提示詞要求模型先在 <analysis> 中"打草稿"：
1. 按時間順序分析每條訊息
2. 識別使用者意圖、技術決策、程式碼模式
3. 特別關注使用者反饋（"使用者讓你做不同的事情"）
4. 雙重檢查技術準確性和完整性

這類似於 Chain-of-Thought，但最終會被 formatCompactSummary() 刪除，
只保留 <summary> 部分進入壓縮後的上下文。
```

### 壓縮後的摘要訊息

```typescript
function getCompactUserSummaryMessage(summary, suppressFollowUp, transcriptPath, recentPreserved): string {
  let msg = `This session is being continued from a previous conversation that ran out of context.
The summary below covers the earlier portion of the conversation.

${formatCompactSummary(summary)}`

  // 如果有 transcript 路徑，提供回溯指引
  if (transcriptPath) {
    msg += `\n\nIf you need specific details from before compaction
(like exact code snippets, error messages, or content you generated),
read the full transcript at: ${transcriptPath}`
  }

  // 如果保留了最近訊息
  if (recentMessagesPreserved) {
    msg += `\n\nRecent messages are preserved verbatim.`
  }

  // 如果是自動壓縮（suppressFollowUp = true）
  if (suppressFollowUpQuestions) {
    msg += `\nContinue the conversation from where it left off
without asking the user any further questions.
Resume directly — do not acknowledge the summary,
do not recap what was happening,
do not preface with "I'll continue" or similar.
Pick up the last task as if the break never happened.`
  }

  // 如果是 Proactive/KAIROS 模式
  if (proactiveModule?.isProactiveActive()) {
    msg += `\n\nYou are running in autonomous/proactive mode.
This is NOT a first wake-up — you were already working autonomously before compaction.
Continue your work loop: pick up where you left off based on the summary above.
Do not greet the user or ask what to work on.`
  }

  return msg
}
```

---

## 9. 5層錯誤恢復中的壓縮角色

在 `query.ts` 的 5 層錯誤恢復機制中，壓縮系統承擔了前 2 層：

```
API 呼叫失敗
  ↓
第1層：上下文摺疊排水（Context Collapse Drain）
  ↓ 失敗
第2層：響應式壓縮（Reactive Compact）  ← 使用 compact.ts
  ↓ 失敗
第3層：最大輸出升級（8K → 64K tokens）
  ↓ 失敗
第4層：多輪恢復（注入"請繼續"訊息，最多 3 次）
  ↓ 失敗
第5層：模型回退（切換到備用模型，剝離 thinking 簽名塊）
  ↓ 失敗
暴露錯誤給使用者
```

**響應式壓縮** vs **主動壓縮**：

| 屬性 | 主動壓縮（Auto-Compact） | 響應式壓縮（Reactive Compact） |
|------|--------------------------|-------------------------------|
| 觸發 | token 數超過閾值 | API 返回 prompt-too-long 錯誤 |
| 時機 | API 呼叫前 | API 呼叫失敗後 |
| 路徑 | `autoCompact.ts` → `compact.ts` | `query.ts` 錯誤恢復層 |
| 回退 | SM → 傳統壓縮 → 放棄 | 傳統壓縮 → 裁剪最舊訊息 |

---

## 10. 各層對比與設計哲學

### 橫向對比

| 屬性 | 微壓縮 | Session Memory | 傳統壓縮 | PTL Recovery |
|------|--------|---------------|----------|-------------|
| **呼叫 LLM** | 否 | 否 | 是（Fork Agent） | 否 |
| **資訊損失** | 最小（僅刪工具輸出） | 中等（靠已有摘要） | 中等（9章摘要） | 高（丟棄最舊訊息） |
| **延遲** | <1ms | <10ms | 5-30s | ~0 |
| **觸發** | 每輪自動 | 自動壓縮優先路徑 | 自動/手動 | 壓縮自身超限 |
| **prompt cache** | 保留（快取編輯） | 破壞 | 共享（Fork） | 破壞 |
| **最大輸出** | — | — | 20K tokens | — |
| **斷路器** | 無 | 無 | 有（3次） | 有（3次） |
| **token 預算** | — | 40K 最大保留 | 50K 檔案重注入 | 按組裁剪 |

### 設計哲學總結

1. **漸進式降級**：從無損操作（微壓縮）到有損操作（傳統壓縮）到丟棄操作（PTL Recovery），每一層都比上一層代價更高

2. **快取優先**：微壓縮的快取編輯路徑專門設計為不破壞 prompt cache，傳統壓縮透過 Fork Agent 共享 cache

3. **安全性保證**：
   - 永遠不切斷 tool_use/tool_result 配對
   - 永遠不分離共享 message.id 的訊息塊
   - 斷路器防止失敗時無限重試
   - 遞迴保護防止壓縮代理觸發自身壓縮

4. **可觀測性**：每個壓縮操作都記錄 `logEvent`（如 `tengu_compact`, `tengu_cached_microcompact`），包含壓縮前後的 token 數、觸發原因、重試次數等

5. **可配置性**：幾乎所有閾值都支援環境變數覆蓋（用於測試）或 GrowthBook 遠端配置（用於動態調整）

---

*來自：AI超元域 | B站頻道：https://space.bilibili.com/3493277319825652*

*基於 Claude Code 原始碼逆向分析，2026-03-31*
