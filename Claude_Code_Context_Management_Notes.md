# Claude Code 上下文管理演算法學習筆記

> 來源：Claude Code 原始碼快照（2026-03-31）
> 核心檔案：`src/query.ts`, `src/services/compact/`, `src/utils/toolResultStorage.ts`

---

## 架構總覽：7 層遞進式防禦

每輪 query loop 按以下順序執行，從便宜到昂貴，每一層減輕壓力，可能阻止下一層觸發：

```
1. Tool Result Budget    ← 單輪預算（同步，零 API 呼叫）
2. Snip Compact          ← 歷史裁剪（零 API 呼叫）
3. Microcompact          ← 工具結果精細清理（零或極低 API 成本）
4. Context Collapse      ← 增量投影摘要（零 API 呼叫）
5. Auto-Compact          ← LLM 全量摘要（昂貴）
6. Blocking Limit        ← 硬停（所有自動措施關閉時）
7. Reactive Compact      ← 413 錯誤後緊急摘要（最後手段）
```

互斥門控防止昂貴操作競爭：Context Collapse 啟用時抑制 Auto-Compact，Reactive Compact 實驗模式抑制主動 Auto-Compact。

---

## 第 1 層：Tool Result Budget — 單輪聚合預算

**問題**：N 個並行工具可能在一輪中集體產生超大上下文。

**三分割槽演算法**：

| 分割槽 | 含義 | 處理 |
|------|------|------|
| mustReapply | 之前已替換過的結果 | 直接重用快取的替換字串（零 I/O，位元組級一致） |
| frozen | 之前見過但沒替換 | 永不替換（保護 prompt cache） |
| fresh | 新結果 | 參與預算分配 |

**閾值**：
- 總預算：200K 字元/訊息（遠端可配置）
- 單工具上限：50K 字元（預設）
- 超出時：保留前 ~2KB 預覽 + 持久化到磁碟，模型可用 Read 按需讀取

**核心洞察**：frozen 分割槽 — 寧可浪費空間也不破壞快取命中率。**Prompt cache 穩定性優先於空間效率**。

---

## 第 2 層：Snip Compact — 歷史裁剪

**特點**：零 API 呼叫，直接刪除舊訊息。

**協調機制**：snipTokensFreed 傳遞給下游層，因為 tokenCountWithEstimation 讀的是上一輪 API 返回的 input_tokens（反映 snip 前的數值），需要手動減去 snip 釋放的量。

**雙檢視**：REPL 保留被 snip 的訊息用於 UI 回滾，但投影層在傳送 API 前過濾掉它們。

---

## 第 3 層：Microcompact — 三種精細清理路徑

### 3a. 基於時間的清理

```
觸發條件：距上次助手訊息 > 60 分鐘（= 服務端 cache TTL）
邏輯：cache 已過期 → 全量字首會被重寫 → 趁機清理舊工具結果
行為：替換為 "[Old tool result content cleared]"
保留：最近 5 個結果
```

**洞察**：利用快取 TTL 過期的"免費視窗"搭車做清理。

### 3b. 快取編輯（Cache Editing）— 最精妙的設計

```
觸發條件：可壓縮工具結果數超過閾值
關鍵創新：不修改本地訊息！使用 API 的 cache_edits / cache_reference 機制
效果：服務端快取被精確編輯，本地保持不變
確認：用 API 返回的 cache_deleted_input_tokens（非客戶端估算）
範圍：僅主執行緒，僅特定工具（FileRead, Bash, Grep, Glob 等）
```

**洞察**：讀寫分離 — 本地訊息不變（保證重放一致性），服務端快取被精確編輯（節省空間）。

### 3c. API 原生上下文管理

```
策略：clear_tool_uses + clear_thinking
觸發：輸入超 180K tokens
目標：保留最後 40K tokens
思考塊：cache 冷（>1h）時僅保留最後一輪思考
```

---

## 第 4 層：Context Collapse — 增量投影式摘要（CQRS 思想）

**核心思想**：不破壞性替換訊息，而是維護一個 commit log，每輪查詢時 project 出壓縮檢視。

**閾值**：
- 90% 上下文視窗 → 開始提交 collapse
- 95% 上下文視窗 → 阻塞新 spawn
- ~93%（auto-compact 位置）→ 被抑制避免競爭

**關鍵 API**：
- `applyCollapsesIfNeeded()` — 投影壓縮檢視 + 可選提交新 collapse
- `recoverFromOverflow()` — 413 時排空所有暫存 collapse（第一道防線）
- `projectView()` — 每輪重放提交日誌

**設計亮點**：
- Collapse 摘要存在 commit store 中，而非 REPL 訊息陣列 → 跨輪持久化
- REPL 保留完整歷史（UI 回滾），API 呼叫看到投影檢視（節省 tokens）
- 會話恢復時從 commits + snapshot 重建

---

## 第 5 層：Auto-Compact — 帶熔斷的 LLM 摘要

**觸發**：`tokenCount > contextWindow - 13,000`

### 摘要 prompt 結構

```xml
<analysis>（內部草稿，生成後丟棄）</analysis>
<summary>
  1. Primary Request / Intent
  2. Key Technical Concepts
  3. Files / Code
  4. Errors / Fixes
  5. Problem Solving
  6. All User Messages（使用者說過的每句話都保留）
  7. Pending Tasks
  8. Current Work
  9. Optional Next Step
</summary>
```

### Prompt Cache 共享最佳化

摘要子 agent 複用主對話的快取字首（透過 runForkedAgent）。
沒有此最佳化：cache miss 率 98%，浪費全域性 ~38B tok/天的 cache_creation。

### PTL（Prompt-Too-Long）重試

```
最多重試 3 次：
  → 按 API 輪次分組（groupMessagesByApiRound）
  → 丟棄最老的組以覆蓋 token 缺口
  → 無法精確計算時 fallback 丟棄 20% 的組
圖片：所有圖片/文件塊替換為 [image]/[document] 標記後再傳送
```

### 熔斷器

```
MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
觸發後：本會話剩餘時間停止嘗試
背景資料：~1,279 個會話曾出現 50+ 次連續失敗（最高 3,272 次），
          每天浪費 ~250K API 呼叫
```

### Post-Compact 恢復清單

| 恢復項 | 預算 |
|--------|------|
| 最近讀取的檔案 | 最多 5 個，每個 5K tokens，總計 50K |
| Skill 附件 | 每個 5K，總計 25K |
| Plan 狀態 | 完整恢復 |
| Deferred tools | 完整恢復 |
| Agent 列表 | 完整恢復 |
| MCP 指令 | 完整恢復 |

### Session Memory Compact（替代路徑）

在 auto-compact 中**優先嚐試**，失敗回退到完整 LLM 摘要：
```
用預提取的 session memory 替代 LLM 摘要
保留範圍：從最後摘要訊息向後擴充套件
  滿足 minTokens(10K) 且 minMessages(5)
  不超過 maxTokens(40K)
不變數：tool_use / tool_result 對永不拆分
```

---

## 第 6-7 層：Reactive Compact + 錯誤恢復級聯

### 5 層錯誤恢復（從便宜到昂貴）

```
API 返回 prompt_too_long 或媒體尺寸被拒
  |
  v
Layer 1: Context Collapse drain（排空所有暫存 collapse，最便宜）
  |  失敗
  v
Layer 2: Reactive Compact（完整 LLM 摘要）
  |  失敗
  v
Layer 3: Max Output 升級（8K → 64K tokens）
  |  失敗
  v
Layer 4: Multi-turn Recovery（注入 nudge 訊息，最多 3 次）
  |  失敗
  v
Layer 5: Model Fallback（切換到備用模型）
```

### 錯誤扣留（Error Withholding）模式

```
流式傳輸期間，可恢復錯誤不 yield 給呼叫者
  → 推入 assistantMessages 供恢復檢查
  → 防止 SDK 消費者終止會話
  → 所有恢復手段都失敗後才暴露給使用者
```

防迴圈守衛：`hasAttemptedReactiveCompact` 防止同一輪無限重試。

---

## Token Budget Continuation — 輸出預算跟蹤（附加機制）

```
用途：長任務自動續跑（不是上下文管理，是輸出預算控制）
完成閾值：輸出 < 預算 90% → 繼續
遞減檢測：連續 3+ 次續跑 且 最近兩次增量 < 500 tokens → 停止
每次續跑注入 nudge 訊息告知進度百分比
```

---

## 值得借鑑的設計原則

### 1. Prompt Cache 穩定性高於空間效率

- frozen 分割槽：進入 cache 的內容永不修改
- 位元組級替換一致性：mustReapply 重用完全相同的替換字串
- Tool pool 排序穩定：`assembleToolPool()` 排序防止 MCP 變化破壞快取
- 快取編輯不改本地訊息：讀寫分離保證重放一致性

### 2. 分層防禦，從便宜到昂貴

7 層遞進，每層有可能阻止下一層觸發。互斥門控防止昂貴操作競爭（collapse 抑制 autocompact）。snipTokensFreed 顯式傳遞確保下游層看到真實 token 數。

### 3. 錯誤扣留 + 延遲恢復

流式傳輸中不立即暴露可恢復錯誤。給恢復機制留出空間後再決定是否暴露。這個模式可以推廣到任何有多層 fallback 的系統。

### 4. CQRS 式雙檢視

REPL 保留完整歷史（UI 回滾 + 會話恢復），API 看到投影檢視（節省 tokens）。Context Collapse 的 commit log 和 projectView 就是這個思想的實現。

### 5. 熔斷器模式

連續失敗 3 次後停止重試。用真實資料驅動：1,279 個會話的失控迴圈 → 250K API 呼叫/天的浪費。

### 6. 利用快取 TTL 做"免費清理"

cache 自然過期（60min）時搭車執行清理。既然字首要重建，就把髒活一起幹了。

### 7. Post-Compact 不從零開始

摘要後精心恢復關鍵上下文（最近檔案、plan、skills）。有明確的 token 預算分配，不是全量恢復也不是什麼都不恢復。

### 8. 不變數保護無處不在

- tool_use / tool_result 對永不拆分
- Compact 邊界訊息記錄 pre-compact 狀態供恢復重鏈
- Partial compact 的 "up_to" 變體清除舊邊界防止級聯剪枝 bug
- 子 agent compact 不重置主執行緒的模組級狀態

---

## Query Loop 狀態機一覽

```typescript
State = {
  messages,
  toolUseContext,
  autoCompactTracking,          // 熔斷計數
  maxOutputTokensRecoveryCount, // 輸出恢復計數
  hasAttemptedReactiveCompact,  // 防迴圈守衛
  maxOutputTokensOverride,      // 8K → 64K 升級
  pendingToolUseSummary,
  stopHookActive,
  turnCount,
  transition                    // 狀態轉移原因
}

transition 型別：
  collapse_drain_retry
  reactive_compact_retry
  max_output_tokens_escalate
  max_output_tokens_recovery
  stop_hook_blocking
  token_budget_continuation
```

每輪不是遞迴而是 `while(true)` + 顯式狀態轉移，避免長會話棧溢位。
