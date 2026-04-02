# Claude Code 是否偷換模型？原始碼級逆向分析

> 基於 Claude Code 原始碼（2026-03-31 快照，512K 行 TypeScript）逆向分析
> 核心檔案：`src/utils/model/model.ts`、`src/services/api/withRetry.ts`、`src/query.ts`、`src/services/api/claude.ts`

---

## 結論先行

**不存在"偷換模型"。你的主對話模型完全由你控制，不會被偷偷降級。**

但 Claude Code 確實在後臺使用 Haiku（弱模型）執行輔助任務（配額檢查、摘要生成等），這些 Haiku 呼叫不會出現在你的對話中，也不會替代你與 Opus 的互動。

唯一可能改變主對話模型的場景是 **529 連續過載回退**（Opus → Sonnet），此時會顯示明確的系統訊息通知使用者。

---

## 1. 主對話模型選擇：完全由使用者控制

### 模型選擇優先順序（原始碼確認）

```typescript
// src/utils/model/model.ts:92
function getMainLoopModel(): ModelName {
  const model = getUserSpecifiedModelSetting()
  if (model !== undefined && model !== null) {
    return parseUserSpecifiedModel(model)
  }
  return getDefaultMainLoopModel()
}

// src/utils/model/model.ts:61
function getUserSpecifiedModelSetting(): ModelSetting | undefined {
  // 優先順序從高到低：
  // 1. /model 命令（會話內切換）— 最高優先順序
  const modelOverride = getMainLoopModelOverride()
  if (modelOverride !== undefined) return modelOverride

  // 2. --model 啟動引數 / ANTHROPIC_MODEL 環境變數 / settings.json
  return process.env.ANTHROPIC_MODEL || settings.model || undefined
}
```

**你設了 Opus，主對話就用 Opus。沒有任何程式碼在你不知情的情況下把它換掉。**

### 預設模型

```typescript
// src/utils/model/model.ts:105
function getDefaultOpusModel(): ModelName {
  // 支援環境變數覆蓋
  if (process.env.ANTHROPIC_DEFAULT_OPUS_MODEL) {
    return process.env.ANTHROPIC_DEFAULT_OPUS_MODEL
  }
  return getModelStrings().opus46  // 預設 Opus 4.6
}

function getDefaultSonnetModel(): ModelName {
  if (process.env.ANTHROPIC_DEFAULT_SONNET_MODEL) {
    return process.env.ANTHROPIC_DEFAULT_SONNET_MODEL
  }
  // 3P 提供商預設 Sonnet 4.5（可能尚未支援 4.6）
  if (getAPIProvider() !== 'firstParty') return getModelStrings().sonnet45
  return getModelStrings().sonnet46
}

function getDefaultHaikuModel(): ModelName {
  if (process.env.ANTHROPIC_DEFAULT_HAIKU_MODEL) {
    return process.env.ANTHROPIC_DEFAULT_HAIKU_MODEL
  }
  return getModelStrings().haiku45  // Haiku 4.5（所有平臺通用）
}
```

---

## 2. 唯一會改變主對話模型的場景：529 過載回退

### 觸發條件（極其嚴格）

```typescript
// src/services/api/withRetry.ts:326-350

// 必須同時滿足以下所有條件：
// 1. 收到 529 錯誤（伺服器過載）
// 2. 連續 529 次數 >= MAX_529_RETRIES
// 3. 配置了 fallbackModel
// 4. 以下二選一：
//    a. 設定了 FALLBACK_FOR_ALL_PRIMARY_MODELS 環境變數
//    b. 非 Claude.ai 訂閱使用者 且 使用非自定義 Opus 模型

if (is529Error(error) &&
    (process.env.FALLBACK_FOR_ALL_PRIMARY_MODELS ||
     (!isClaudeAISubscriber() && isNonCustomOpusModel(options.model)))) {
  consecutive529Errors++
  if (consecutive529Errors >= MAX_529_RETRIES) {
    if (options.fallbackModel) {
      // 觸發回退
      throw new FallbackTriggeredError(options.model, options.fallbackModel)
    }
  }
}
```

**關鍵限制：**

| 條件 | 說明 |
|------|------|
| `!isClaudeAISubscriber()` | **Claude.ai 訂閱使用者預設不觸發回退** |
| `isNonCustomOpusModel()` | 僅對標準 Opus 模型生效（自定義模型 ID 不觸發） |
| `MAX_529_RETRIES` | 需要連續多次 529 才觸發 |

### 回退執行過程（透明可見）

```typescript
// src/query.ts:894-946

if (innerError instanceof FallbackTriggeredError && fallbackModel) {
  // 1. 切換模型
  currentModel = fallbackModel  // Opus → Sonnet（不是 Haiku）

  // 2. 清理孤立的 assistant 訊息
  yield* yieldMissingToolResultBlocks(assistantMessages, 'Model fallback triggered')
  assistantMessages.length = 0

  // 3. 剝離 thinking 簽名塊（簽名繫結原模型，發給回退模型會導致 400）
  // 原始碼註釋：
  // "Strip before retry so the fallback model gets clean history."

  // 4. 更新工具上下文中的模型引用
  toolUseContext.options.mainLoopModel = fallbackModel

  // 5. 記錄分析事件
  logEvent('tengu_api_opus_fallback_triggered', {
    original_model: options.model,
    fallback_model: options.fallbackModel,
  })

  // 6. 顯示系統訊息通知使用者（在 UI 中可見）
  `Switched to ${renderModelName(fallbackModel)} due to high demand
   for ${renderModelName(originalModel)}`
}
```

### 回退目標

```typescript
// src/main.tsx:1336-1337
// 回退模型必須與主模型不同
if (fallbackModel && options.model && fallbackModel === options.model) {
  // 驗證失敗，不設定回退
}

// 回退目標是 Sonnet（不是 Haiku）
// 透過 getDefaultSonnetModel() 獲取
```

### 使用者如何知道發生了回退

回退時，UI 中會顯示一條系統訊息：

```
Switched to Claude Sonnet 4.6 due to high demand for Claude Opus 4.6
```

同時在 `AssistantTextMessage.tsx:178` 中提供操作建議：

```
To continue immediately, use /model to switch to Sonnet and continue coding.
```

---

## 3. 後臺輔助任務使用 Haiku — 不影響主對話

Claude Code 在後臺使用 Haiku（`getSmallFastModel()`）執行多種輕量輔助任務。這些呼叫**完全獨立於你的對話**，不會出現在訊息流中。

### 所有 Haiku 使用場景（逐一列出）

#### 3.1 配額檢查

```typescript
// src/services/claudeAiLimits.ts:199-218
async function makeTestQuery() {
  const model = getSmallFastModel()  // → Haiku
  const anthropic = await getAnthropicClient({ maxRetries: 0, model, source: 'quota_check' })
  // 傳送 max_tokens: 1 的最小請求，僅讀取響應頭的配額資訊
  return anthropic.beta.messages.create({
    model,
    max_tokens: 1,
    messages: [{ role: 'user', content: 'quota' }],
  }).asResponse()
}
```

**用途：** 檢測使用者是否已達配額限制。只讀取 HTTP 響應頭，不使用模型輸出。

#### 3.2 離開摘要（Away Summary）

```typescript
// src/services/awaySummary.ts:44-52
// 當使用者長時間離開後返回時，生成一個簡短摘要
{
  thinkingConfig: { type: 'disabled' },  // 關閉思考
  tools: [],                              // 不給工具
  model: getSmallFastModel(),             // → Haiku
}
```

**用途：** 使用者長時間不操作後返回時，快速生成"你離開期間發生了什麼"的簡要摘要。

#### 3.3 Token 計數

```typescript
// src/services/api/claude.ts:540-543
// WARNING: if you change this to use a non-Haiku model,
// this request will fail in 1P unless it uses getCLISyspromptPrefix.
const model = getSmallFastModel()  // → Haiku
// 呼叫 countTokens API 獲取精確 token 數
```

**用途：** 呼叫 Anthropic 的 `countTokens` 端點獲取精確 token 計數，用於自動壓縮閾值判斷。

#### 3.4 工具使用摘要

```typescript
// src/services/api/claude.ts:3273-3280
// 在主模型流式輸出期間，並行用 Haiku 生成工具呼叫的簡短描述
{
  model: getSmallFastModel(),             // → Haiku
  enablePromptCaching: false,
}
```

**用途：** 當模型連續呼叫多個工具時，生成簡短的工具使用描述（如"讀取了 config.ts 並修改了第 42 行"）。

原始碼註釋明確說明了並行關係：
```typescript
// src/query.ts:1054
// Yield tool use summary from previous turn —
// haiku (~1s) resolved during model streaming (5-30s)
```
Haiku 在後臺 1 秒內完成摘要，同時 Opus 還在處理你的主對話。

#### 3.5 記憶相關性評分

```typescript
// src/memdir/findRelevantMemories.ts
// 用 Sonnet（非 Haiku）評估記憶檔案的相關性
// 從標題中選擇最多 5 條與當前查詢相關的記憶
```

#### 3.6 Web 搜尋預處理

```typescript
// src/tools/WebSearchTool/WebSearchTool.ts:280
model: useHaiku ? getSmallFastModel() : context.options.mainLoopModel,
toolChoice: useHaiku ? { type: 'tool', name: 'web_search' } : undefined,
```

**用途：** 當條件滿足時（`useHaiku` 標誌），用 Haiku 預處理搜尋查詢。

#### 3.7 其他輔助呼叫

| 呼叫者 | 模型 | 用途 |
|--------|------|------|
| `tokenEstimation.ts` | Haiku | Vertex 全域性區域 token 計數回退 |
| `analyzeContext.ts` | Haiku | 上下文分析的 countTokens 回退 |
| `claudeAiLimits.ts` | Haiku | 配額狀態檢查 |
| Bedrock `client.ts` | Haiku | 小模型可指定不同 AWS 區域 |

### 輔助任務 vs 主對話的隔離

```
你的對話：
  使用者 → [Opus 4.6] → 助手響應 → [Opus 4.6] → 助手響應 → ...

後臺並行：
  [Haiku] → 配額檢查（1 token）
  [Haiku] → 工具使用摘要（~1s）
  [Haiku] → token 計數

兩條線完全隔離，Haiku 的輸出不進入你的對話訊息流。
```

---

## 4. Fast Mode：同模型加速，不換模型

這是一個常見誤解，原始碼中有明確宣告：

```typescript
// src/constants/prompts.ts:702
`Fast mode for Claude Code uses the same ${FRONTIER_MODEL_NAME} model
 with faster output. It does NOT switch to a different model.
 It can be toggled with /fast.`
```

Fast Mode 透過 API 的 speed 引數請求更快的輸出，模型不變。如果 API 拒絕 fast mode（如組織未啟用），會自動降回標準速度：

```typescript
// src/services/api/withRetry.ts:310-313
if (wasFastModeActive && isFastModeNotEnabledError(error)) {
  handleFastModeRejectedByAPI()
  retryContext.fastMode = false
  continue  // 重試，同一模型，標準速度
}
```

---

## 5. Plan Mode 臨時模型切換 — 透明可見

```typescript
// src/commands/model/model.tsx:214
// Do not update fast mode in settings since this is an automatic downgrade

// src/commands/model/model.tsx:256
onDone(`Current model: ${chalk.bold(renderModelLabel(mainLoopModelForSession))}
  (session override from plan mode)\nBase model: ${displayModel}`)
```

Plan Mode 可能臨時使用 Sonnet 執行實現步驟，但：
- UI 顯示 "session override from plan mode"
- 顯示 Base model 讓你知道原始模型
- 不修改 settings 中的模型設定

---

## 6. Effort 降級 — 不換模型，換引數

```typescript
// src/utils/effort.ts:162
// API rejects 'max' on non-Opus-4.6 models — downgrade to 'high'.
```

如果設定了 `effort: max` 但模型不支援，會降級 effort 引數（max → high），不換模型。

---

## 7. 全域性搜尋：不存在的場景

透過對 512K 行原始碼的全域性搜尋，以下場景**被確認不存在**：

| 假設的偷換場景 | 搜尋結果 | 證據 |
|----------------|----------|------|
| 根據訂閱層級偷換弱模型 | **不存在** | `getMainLoopModel()` 不檢查訂閱型別 |
| 根據額度用量偷換弱模型 | **不存在** | 額度用完返回 429 錯誤，不換模型 |
| 隨機/機率性降級到弱模型 | **不存在** | 無 `Math.random()` 與模型選擇相關的程式碼 |
| A/B 測試使用弱模型回答 | **不存在** | GrowthBook 控制功能開關，不控制模型 |
| 靜默替換後不通知使用者 | **不存在** | 529 回退有系統訊息，其他場景不換主模型 |
| 把 Opus 對話路由到 Haiku | **不存在** | Haiku 僅用於輔助任務 |
| 根據問題複雜度選擇弱模型 | **不存在** | 主迴圈始終使用使用者指定的模型 |
| 根據時間段降級（如高峰期） | **不存在** | 無時間相關的模型選擇邏輯 |
| 首次使用者用弱模型 | **不存在** | 預設模型是 Opus 4.6 |

---

## 8. 模型使用全景圖

```
┌─────────────────────────────────────────────────────────┐
│                   你的主對話                              │
│                                                         │
│  模型：你指定的（預設 Opus 4.6）                          │
│  控制：/model 命令 > --model 引數 > 環境變數 > 設定       │
│  唯一例外：529 連續過載 → Sonnet 回退（有通知）            │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                   後臺輔助任務（Haiku）                    │
│                                                         │
│  配額檢查      → 1 token 請求，只讀響應頭                 │
│  Token 計數    → countTokens API 呼叫                    │
│  工具摘要      → 並行於主對話，~1s 完成                    │
│  離開摘要      → 使用者回來時顯示簡要資訊                    │
│  Web 搜尋      → 搜尋查詢預處理（條件觸發）                │
│                                                         │
│  這些不進入你的對話訊息流                                  │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                   Fork Agent 任務                        │
│                                                         │
│  上下文壓縮    → 使用你的主模型（共享 prompt cache）       │
│  記憶提取      → 使用你的主模型                           │
│  許可權分類器    → Haiku（auto 模式下的命令安全分類）        │
│                                                         │
│  壓縮結果進入上下文，但不替代你的對話                      │
└─────────────────────────────────────────────────────────┘
```

---

## 9. 如何驗證你正在使用的模型

### 方法 1：檢視 API 響應

每條助手訊息的內部結構中包含模型資訊：

```typescript
// AssistantMessage.message.model 欄位包含實際使用的模型 ID
// 例如："claude-opus-4-6-20260301"
```

### 方法 2：使用 /cost 命令

`/cost` 命令顯示按模型分類的 token 使用量，你可以看到每個模型消耗了多少 token。

### 方法 3：檢查 status line

狀態列顯示當前模型名稱。如果發生回退，會更新為回退後的模型。

### 方法 4：環境變數除錯

```bash
# 檢視所有 API 呼叫的模型資訊
export CLAUDE_CODE_DEBUG=1

# 強制禁用回退
unset FALLBACK_FOR_ALL_PRIMARY_MODELS
```

---

## 10. 總結

| 問題 | 答案 | 依據 |
|------|------|------|
| Opus 會被偷偷換成 Haiku 嗎？ | **不會** | Haiku 僅用於輔助任務，不進入主對話 |
| Opus 會被偷偷換成 Sonnet 嗎？ | **僅在 529 連續過載時，且會通知** | `withRetry.ts:347` + `query.ts:946` |
| Claude.ai 訂閱使用者會被降級嗎？ | **預設不會** | `!isClaudeAISubscriber()` 條件排除 |
| 後臺有 Haiku 呼叫嗎？ | **有，但不影響你的對話** | 6 種輔助任務，全部隔離 |
| Fast Mode 換模型嗎？ | **不換** | 原始碼明確宣告 "does NOT switch to a different model" |
| 有沒有根據問題難度選模型？ | **沒有** | 主迴圈始終使用使用者指定模型 |

**一句話：你付費用 Opus，對話就用 Opus。Haiku 只在幕後幹雜活。**

---

*來自：AI超元域 | B站頻道：https://space.bilibili.com/3493277319825652*

*基於 Claude Code 原始碼逆向分析，2026-03-31*
