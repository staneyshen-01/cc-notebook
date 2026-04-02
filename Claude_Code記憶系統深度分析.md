# Claude Code Memory 記憶系統深度分析

> 基於 Claude Code 原始碼（2026-03-31 快照，512K 行 TypeScript）逆向分析
> 核心檔案：`src/memdir/`（8個檔案）、`src/services/extractMemories/`（2個）、`src/services/SessionMemory/`（3個）、`src/services/teamMemorySync/`（5個）

---

## 1. 三層記憶架構總覽

Claude Code 的記憶系統不是一個簡單的檔案儲存，而是一個**三層遞進的知識管理體系**：

```
┌─────────────────────────────────────────────────────────────┐
│  第3層：團隊記憶（Team Memory）                               │
│  跨使用者、按倉庫   │ 服務端同步 │ REST API + 樂觀鎖           │
│  秘密掃描保護     │ 檔案監聽器 │ 30種憑證檢測規則             │
├─────────────────────────────────────────────────────────────┤
│  第2層：持久記憶（Persistent Memory / extractMemories）       │
│  跨會話、按專案   │ 本地檔案   │ 後臺 Fork Agent 自動提取     │
│  4種型別分類      │ MEMORY.md 索引 │ AI 相關性召回            │
├─────────────────────────────────────────────────────────────┤
│  第1層：會話記憶（Session Memory）                             │
│  僅當前會話      │ 單個檔案   │ 10個固定章節 │ 服務於壓縮     │
└─────────────────────────────────────────────────────────────┘
```

| 層級 | 範圍 | 永續性 | 觸發方式 | 儲存位置 |
|------|------|--------|----------|----------|
| 會話記憶 | 當前會話 | 臨時（會話結束即停用） | 後取樣鉤子（token+工具呼叫閾值） | 會話目錄下單檔案 |
| 持久記憶 | 跨會話、按專案 | 永久（直到手動刪除） | 每輪查詢結束時（Stop Hook） | `~/.claude/projects/<專案>/memory/` |
| 團隊記憶 | 跨使用者、按倉庫 | 遠端伺服器 + 本地映象 | 檔案監聽器（2秒去抖） | `memory/team/` + Anthropic API |

---

## 2. 持久記憶系統（核心）

### 2.1 記憶檔案格式

每條記憶是一個獨立的 Markdown 檔案，帶 YAML frontmatter：

```markdown
---
name: 使用者偏好-簡潔回覆
description: 使用者不希望在每次回覆末尾加總結
type: feedback
---

不要在回覆末尾總結剛做了什麼，使用者能看到 diff。

**Why:** 使用者明確要求過"stop summarizing what you just did"
**How to apply:** 所有回覆結束時，直接結束，不加回顧性總結。
```

### 2.2 四種記憶型別

```typescript
const MEMORY_TYPES = ['user', 'feedback', 'project', 'reference'] as const
```

| 型別 | 用途 | 儲存時機 | 作用域（團隊模式） |
|------|------|----------|:------------------:|
| **user** | 使用者角色、目標、知識背景 | 瞭解到使用者的身份資訊時 | 始終私有 |
| **feedback** | 使用者對工作方式的指導 | 使用者糾正或確認做法時 | 預設私有 |
| **project** | 專案進展、目標、事件 | 瞭解到工作計劃、截止日期時 | 偏向團隊 |
| **reference** | 外部系統指標 | 瞭解到 Linear/Grafana/Slack 等資源時 | 通常團隊 |

**每種型別要求的結構：**
- feedback 和 project 型別必須包含 `**Why:**` 和 `**How to apply:**` 行
- 相對日期必須轉換為絕對日期（如"週四"→"2026-03-05"）
- user 型別的示例："深度 Go 經驗，React 新手——用後端類比解釋前端概念"

### 2.3 明確不儲存的內容

原始碼中硬編碼了 6 類排除項（即使使用者明確要求也不儲存）：

1. 程式碼模式、架構、檔案路徑、專案結構——可從程式碼推導
2. Git 歷史、最近變更——`git log`/`git blame` 是權威來源
3. 除錯方案或修復步驟——修復在程式碼中，commit message 有上下文
4. CLAUDE.md 中已有的內容——不重複
5. 臨時任務細節：進行中的工作、當前對話上下文
6. 如果使用者要求儲存 PR 列表或活動摘要，要追問"什麼是令人意外的或不明顯的？"——只儲存那部分

### 2.4 MEMORY.md 索引機制

```
MEMORY.md 是索引，不是記憶本身
```

**格式：** 純 Markdown，無 frontmatter。每條一行，約 150 字元以內：
```markdown
- [使用者偏好-簡潔回覆](feedback_concise.md) — 不在回覆末尾加總結
- [專案-合併凍結](project_freeze.md) — 3月5日起移動端釋出凍結
```

**限制：**
- 最大 200 行（`MAX_ENTRYPOINT_LINES`）
- 最大 25,000 位元組（`MAX_ENTRYPOINT_BYTES`）
- 超出時在末尾追加截斷警告
- 截斷按行邊界執行（不會切斷一行的中間）

**兩步儲存流程：**
1. 寫入記憶檔案（如 `feedback_concise.md`）
2. 在 MEMORY.md 中新增索引行

可選的 `skipIndex` 模式（透過 `tengu_moth_copse` 特性開關）可跳過第2步。

### 2.5 儲存目錄結構

```
~/.claude/
  projects/
    -Users-charlesqin-Desktop-myproject/   ← sanitizePath(專案根目錄)
      memory/                               ← AUTO_MEM_DIRNAME
        MEMORY.md                           ← 索引檔案
        user_role.md                        ← 私有記憶檔案
        feedback_testing.md
        project_deadline.md
        reference_linear.md
        team/                               ← 團隊記憶子目錄
          MEMORY.md                         ← 團隊索引
          project_api_migration.md
          reference_oncall_board.md
        logs/                               ← KAIROS 每日日誌
          2026/
            03/
              2026-03-31.md
```

### 2.6 路徑安全機制

`paths.ts` 包含多層安全驗證：

| 安全層 | 檢查內容 | 防禦目標 |
|--------|----------|----------|
| 絕對路徑檢查 | 拒絕相對路徑 | 路徑遍歷 |
| 根路徑檢查 | 拒絕長度 < 3 的路徑 | 寫入系統根目錄 |
| UNC 路徑檢查 | 拒絕 `\\server\share` 和 `//server/share` | NTLM 憑證洩露 |
| Null 位元組檢查 | 拒絕包含 `\0` 的路徑 | 路徑截斷攻擊 |
| Tilde 展開限制 | 拒絕 `~`、`~/`、`~/.`、`~/..` | 匹配整個 HOME |
| **專案設定排除** | `.claude/settings.json` 不能設 `autoMemoryDirectory` | **惡意倉庫重定向寫入到 `~/.ssh`** |
| NFC 規範化 | Unicode NFC 標準化 | macOS 路徑不一致 |
| Git 根規範化 | worktree 共享主倉庫的記憶 | 避免重複記憶目錄 |

**關鍵安全設計：** `projectSettings`（提交到倉庫的 `.claude/settings.json`）被有意排除在 `autoMemoryDirectory` 的信任來源之外。原始碼註釋明確說明原因：

```typescript
// SECURITY: projectSettings is intentionally excluded —
// a malicious repo could set autoMemoryDirectory: "~/.ssh"
// and gain write access to sensitive directories
```

---

## 3. 自動記憶提取（extractMemories）

### 3.1 觸發時機

```
使用者輸入 → 模型響應 → 工具執行 → 迴圈... → 模型最終響應（無工具呼叫）
                                                    ↓
                                              Stop Hooks 執行
                                                    ↓
                                        executeExtractMemories()  ← 即發即忘
```

**前置條件（全部滿足才執行）：**
1. `EXTRACT_MEMORIES` 編譯時特性開關開啟
2. `isExtractModeActive()` = true（`tengu_passport_quail` GrowthBook 開關）
3. `isAutoMemoryEnabled()` = true
4. 非子代理（`!toolUseContext.agentId`）
5. 非 bare 模式
6. 自上次提取以來有新的模型可見訊息

### 3.2 互斥機制

```typescript
// 如果主代理在對話中已經直接寫了記憶檔案，跳過自動提取
if (hasMemoryWritesSince(lastMemoryMessageUuid)) {
  // 推進遊標但不執行提取——避免覆蓋使用者顯式儲存的記憶
  lastMemoryMessageUuid = messages[messages.length - 1].uuid
  return
}
```

這防止了自動提取和使用者手動儲存之間的衝突。

### 3.3 Fork Agent 執行

```typescript
const result = await runForkedAgent({
  messages: conversationMessages,
  system: extractionPrompt,
  maxTurns: 5,           // 硬上限：5輪（讀1輪 + 寫1-4輪）
  canUseTool: createAutoMemCanUseTool(autoMemPath),
  // ... 共享主會話的 prompt cache
})
```

**工具許可權白名單：**

| 工具 | 許可權 |
|------|------|
| FileRead | 允許（無限制） |
| Grep | 允許（無限制） |
| Glob | 允許（無限制） |
| Bash | 僅只讀命令（ls, find, grep, cat, stat, wc, head, tail） |
| FileEdit | 僅限 `autoMemPath` 內 |
| FileWrite | 僅限 `autoMemPath` 內 |
| MCP / Agent / 寫入型 Bash | **全部禁止** |

### 3.4 提取節流

```typescript
let turnsSinceLastExtraction = 0
const turnsBeforeExtraction = getFeatureValue('tengu_bramble_lintel', 1)

// 每 N 輪查詢才執行一次提取（預設每輪都執行）
if (turnsSinceLastExtraction < turnsBeforeExtraction) {
  turnsSinceLastExtraction++
  return
}
```

### 3.5 提取後通知

成功提取後，將寫入的檔案路徑列表作為 `SystemMemorySavedMessage` 追加到主對話中，這樣主代理知道記憶已更新。

---

## 4. AI 驅動的記憶召回（findRelevantMemories）

### 4.1 召回流程

```
使用者傳送訊息
  ↓
scanMemoryFiles(memoryDir)
  ↓ 遞迴讀取所有 .md 檔案（排除 MEMORY.md）
  ↓ 解析前 30 行 frontmatter（description + type）
  ↓ 按 mtime 降序排序，上限 200 個檔案
  ↓
過濾掉 alreadySurfaced（之前輪次已展示的路徑）
  ↓
selectRelevantMemories(query, memories, signal, recentTools)
  ↓ 構建文字清單：每行 "- [type] filename (timestamp): description"
  ↓ 附加 "Recently used tools: ..." 部分
  ↓
sideQuery → Sonnet 模型
  ↓ 系統提示：SELECT_MEMORIES_SYSTEM_PROMPT
  ↓ 使用者訊息：Query + Available memories
  ↓ 結構化輸出：{ selected_memories: string[] }
  ↓ max_tokens: 256
  ↓
返回最多 5 條 RelevantMemory（路徑 + mtime）
```

### 4.2 選擇提示詞的關鍵規則

```
- 返回最多 5 個檔名
- 要有選擇性和辨別力；如果不確定，不要包含
- 如果提供了最近使用的工具列表：
  ✗ 不要選擇這些工具的使用參考/API 文件（已經在用了）
  ✓ 仍然選擇這些工具的警告/陷阱/已知問題
```

### 4.3 記憶新鮮度標註

```typescript
function memoryAge(mtimeMs: number): string {
  const days = memoryAgeDays(mtimeMs)
  if (days === 0) return 'today'
  if (days === 1) return 'yesterday'
  return `${days} days ago`
}

function memoryFreshnessText(mtimeMs: number): string {
  if (memoryAgeDays(mtimeMs) <= 1) return ''  // 新鮮記憶不加警告
  return `This memory is ${days} days old.
Memories are point-in-time observations, not live state —
claims about code behavior or file:line citations may be outdated.
Verify against current code before asserting as fact.`
}
```

**設計動機（原始碼註釋）：** 使用者報告過過期的記憶被模型當作事實斷言。"47 days ago" 比 ISO 時間戳更能觸發模型的過期推理。

### 4.4 召回前的驗證要求

原始碼中注入的系統提示明確要求模型在使用記憶前驗證：

```
- 如果記憶提到一個檔案路徑 → 檢查檔案是否存在
- 如果記憶提到一個函式或標誌 → grep 搜尋它
- 如果使用者即將根據你的建議行動 → 先驗證
- "記憶說 X 存在"不等於"X 現在存在"
- 活動日誌、架構快照是凍結在某個時間點的，優先使用 git log
```

---

## 5. 會話記憶（Session Memory）

### 5.1 與持久記憶的關鍵區別

| 維度 | 會話記憶 | 持久記憶 |
|------|----------|----------|
| 生命週期 | 當前會話 | 跨會話永久 |
| 檔案數量 | 1 個 | 多個 |
| 觸發頻率 | 每次後取樣（需滿足閾值） | 每輪查詢結束 |
| 主要用途 | **服務於上下文壓縮** | 跨會話知識保留 |
| 記憶提取者 | Fork Agent（僅 FileEdit） | Fork Agent（Read/Write/Edit/Grep） |
| 格式 | 10個固定章節 | 自由格式 + frontmatter |

### 5.2 觸發閾值

```typescript
// 預設配置（可透過 GrowthBook tengu_sm_config 遠端調整）
const defaults = {
  minimumMessageTokensToInit: 10_000,   // 上下文達到 10K tokens 才啟用
  minimumTokensBetweenUpdate: 5_000,    // 每增長 5K tokens 更新一次
  toolCallsBetweenUpdates: 3,           // 且至少 3 次工具呼叫
}

// 觸發條件：
// (token 閾值滿足 AND 工具呼叫閾值滿足)
// OR (token 閾值滿足 AND 最後一輪助手訊息無工具呼叫)
```

### 5.3 會話記憶模板（10 個固定章節）

```markdown
# Session Memory

## Session Title
_Brief description of what this session is about_

## Current State
_What is the current status of the work_

## Task specification
_Detailed description of the current task requirements_

## Files and Functions
_Key files and functions being worked on_

## Workflow
_Steps being followed or processes in use_

## Errors & Corrections
_Errors encountered and how they were resolved_

## Codebase and System Documentation
_Important codebase patterns, conventions, and system behavior_

## Learnings
_Insights gained during this session_

## Key results
_Important outputs, measurements, or achievements_

## Worklog
_Chronological log of actions taken_
```

### 5.4 大小限制

```typescript
const MAX_SECTION_LENGTH = 2_000      // 每個章節最大 2K tokens
const MAX_TOTAL_SESSION_MEMORY_TOKENS = 12_000  // 整個檔案最大 12K tokens
```

### 5.5 與壓縮的整合

Session Memory 是上下文壓縮的優先路徑（參見 `sessionMemoryCompact.ts`）：
- 自動壓縮觸發時，優先嚐試用 Session Memory 作為摘要
- 保留最近 10K-40K tokens 的訊息
- 比傳統壓縮快得多（不呼叫 LLM），且摘要質量更可預測

### 5.6 自定義

使用者可以自定義模板和提示詞：
- 模板：`~/.claude/session-memory/config/template.md`
- 提示詞：`~/.claude/session-memory/config/prompt.md`（支援 `{{variableName}}` 替換）

---

## 6. 團隊記憶同步

### 6.1 前置條件

同時滿足才啟用：
1. `TEAMMEM` 編譯時特性開關開啟
2. `isTeamMemoryEnabled()` = true（`tengu_herring_clock` GrowthBook 開關）
3. 第一方 OAuth + 同時擁有 `INFERENCE_SCOPE` 和 `PROFILE_SCOPE`
4. Git remote 是 github.com（非 GitHub 倉庫不同步）

### 6.2 API 端點

```
基礎 URL: {baseUrl}/api/claude_code/team_memory?repo={owner/repo}
```

| 方法 | 引數 | 用途 | 關鍵狀態碼 |
|------|------|------|:----------:|
| GET | `repo={slug}` | 拉取全部團隊記憶 | 200 / 304 / 404 |
| GET | `repo={slug}&view=hashes` | 僅拉取雜湊（輕量探測） | 200 / 404 |
| PUT | `repo={slug}` | 上傳記憶條目（upsert） | 200 / 412 / 413 |

**認證頭：** `Authorization: Bearer {oauthToken}`

### 6.3 同步協議

#### 拉取（Pull）

```
1. GET 請求，攜帶 ETag（條件請求）
2. 304 = 未變化，跳過
3. 200 = 有更新：
   a. 對每個條目驗證路徑（防遍歷攻擊）
   b. 跳過 >250KB 的檔案
   c. 跳過本地內容已匹配的檔案
   d. 並行寫入本地檔案系統
   e. 重新整理 serverChecksums
```

#### 推送（Push — Delta 上傳 + 樂觀鎖）

```
1. readLocalTeamMemory(): 遍歷 team/ 目錄
   → 每個檔案掃描秘密（30種規則）
   → 跳過 >250KB 的檔案
   → 計算 SHA-256 雜湊

2. 計算 delta：僅本地雜湊與 serverChecksums 不同的 key

3. 分批上傳：貪心裝箱，每批 ≤ 200KB (MAX_PUT_BODY_BYTES)

4. 每批攜帶 If-Match 頭（樂觀鎖）

5. 衝突處理：
   412 Conflict → 探測 ?view=hashes → 重新整理 serverChecksums → 重算 delta → 重試（最多 2 次）
   413 Too Many → 學習 serverMaxEntries 上限，截斷後重試

6. 衝突策略：本地優先（local-wins）
   → 當前使用者正在編輯的內容覆蓋服務端
```

### 6.4 檔案監聽器

```typescript
// watcher.ts
fs.watch(teamMemDir, { recursive: true })
// macOS: FSEvents（O(1) 檔案描述符）
// Linux: inotify（O(子目錄數)）

// 2 秒去抖：最後一次變更後 2 秒才觸發推送
```

**推送抑制：** 永久失敗（no_oauth、no_repo、4xx 非 409/429）後，抑制後續推送，直到發生檔案刪除（恢復動作）或會話重啟。

### 6.5 團隊 vs 私有的作用域劃分

原始碼注入的系統提示中，每種記憶型別都帶有 XML 作用域標籤：

```xml
<type>
  <name>user</name>
  <scope>always private</scope>
  ...
</type>

<type>
  <name>project</name>
  <scope>strongly bias toward team</scope>
  ...
</type>
```

額外安全要求：
```
MUST avoid saving sensitive data within shared team memories
```

---

## 7. 秘密掃描保護

### 7.1 雙層防護

| 層 | 時機 | 檔案 | 行為 |
|----|------|------|------|
| 寫入時攔截 | FileWrite/FileEdit 呼叫 | `teamMemSecretGuard.ts` | 阻止寫入，返回錯誤 |
| 上傳前掃描 | pushTeamMemory 讀取本地檔案 | `secretScanner.ts` | 跳過該檔案，不上傳 |

### 7.2 檢測的 30 種秘密模式

**雲服務商：**
- AWS Access Token（A3T/AKIA/ASIA/ABIA/ACCA 字首）
- GCP API Key（AIza 字首）
- Azure AD Client Secret
- DigitalOcean PAT / Access Token

**AI API：**
- Anthropic API Key（`sk-ant-api03-` 字首，執行時拼接避免自身匹配）
- Anthropic Admin API Key（`sk-ant-admin01-`）
- OpenAI API Key（`sk-proj`/`svcacct`/`admin` + `T3BlbkFJ` 標記）
- HuggingFace Access Token（`hf_`）

**版本控制：**
- GitHub PAT / Fine-grained PAT / App Token / OAuth / Refresh Token
- GitLab PAT / Deploy Token

**通訊平臺：**
- Slack Bot/User/App Token（`xoxb-`/`xoxp-`/`xapp-`）
- Twilio API Key
- SendGrid API Token

**開發工具：**
- npm Access Token
- PyPI Upload Token
- Databricks API Token
- HashiCorp Terraform API Token
- Pulumi API Token
- Postman API Token

**可觀測性：**
- Grafana API Key / Cloud API Token / Service Account Token
- Sentry User/Org Token

**支付：**
- Stripe Access Token（`sk_test`/`live`/`prod` 或 `rk_`）
- Shopify Access Token / Shared Secret

**密碼學：**
- PEM 格式私鑰（`BEGIN/END PRIVATE KEY` 塊）

### 7.3 設計原則

```typescript
// 匹配到的秘密值永遠不被記錄或返回——只返回規則 ID 和人類可讀標籤
// Anthropic API key 字首在執行時拼接，避免匹配自身的 excluded-strings 檢查：
const prefix = ['sk', 'ant', 'api'].join('-')
```

---

## 8. KAIROS 模式的記憶（每日日誌）

當 KAIROS（AI 助手模式）啟用時，記憶正規化完全改變：

```
普通模式：寫獨立檔案 + 更新 MEMORY.md 索引
KAIROS 模式：追加到 logs/YYYY/MM/YYYY-MM-DD.md 日誌檔案
```

- 只追加，不編輯已有內容
- 每日一個檔案，按日期路徑組織
- 由單獨的 `/dream` 技能在夜間蒸餾日誌為主題檔案和 MEMORY.md
- 提示詞中使用 `YYYY-MM-DD` 佔位符而非實際日期（因為提示詞被快取，日期變化不會觸發失效）

---

## 9. 記憶系統注入的完整系統提示詞

### 9.1 個人模式（auto memory）

```
# auto memory

You have a persistent, file-based memory system at `<memoryDir>`.
This directory already exists — write to it directly with the Write tool.

You should build up this memory system over time so that future conversations
can have a complete picture of who the user is, how they'd like to collaborate
with you, what behaviors to avoid or repeat, and the context behind the work.

[如果使用者要求記住 → 立即儲存]
[如果使用者要求忘記 → 找到並刪除]

## Types of memory
[4種型別，每種含 description/when_to_save/how_to_use/examples]

## What NOT to save in memory
[6類排除項]

## How to save memories
Step 1: 寫入檔案（帶 frontmatter）
Step 2: 在 MEMORY.md 中新增索引行

## When to access memories
- 看起來相關時
- 使用者明確要求時（必須訪問）
- 使用者說"忽略記憶"時，假裝 MEMORY.md 為空

## Before recommending from memory
- 提到檔案路徑 → 檢查檔案是否存在
- 提到函式/標誌 → grep 搜尋
- 使用者要行動 → 先驗證

## Memory and other forms of persistence
- 實施方案 → 用 Plan，不用記憶
- 當前對話步驟 → 用 Tasks，不用記憶

## Searching past context
[grep 記憶檔案，或搜尋會話 transcript]

## MEMORY.md
[實際的 MEMORY.md 內容，或"當前為空"]
```

### 9.2 團隊模式額外內容

在個人模式基礎上增加：

```
## Memory scope
- 私有記憶：<privateDir> — 你的個人偏好、反饋
- 團隊記憶：<teamDir> — 專案上下文、外部引用

[型別定義中增加 <scope> XML 標籤]

額外安全規則：不得在團隊記憶中儲存敏感資料
```

---

## 10. 團隊記憶的路徑安全（深度防禦）

`teamMemPaths.ts` 是整個記憶系統中安全措施最密集的檔案：

### 10.1 自定義錯誤類

```typescript
class PathTraversalError extends Error {
  name = 'PathTraversalError'
}
```

### 10.2 sanitizePathKey — 服務端提供的相對路徑清理

```typescript
function sanitizePathKey(key: string): string {
  // 1. 拒絕 null 位元組
  // 2. URL 編碼遍歷檢測：解碼 %2e%2e%2f 等
  // 3. Unicode 規範化攻擊：全形字元 \uFF0E\uFF0E\uFF0F → ../
  // 4. 拒絕反斜槓（Windows 路徑遍歷）
  // 5. 拒絕絕對路徑
}
```

### 10.3 realpathDeepestExisting — 符號連結解析

```typescript
async function realpathDeepestExisting(path: string): Promise<string> {
  // 逐級向上 realpath() 直到成功
  // 檢測懸掛符號連結（link 存在但 target 不存在）
  //   → 安全威脅：writeFile 會跟隨連結在 team 目錄外建立檔案
  // 檢測符號連結迴圈（ELOOP）→ PathTraversalError
  // 不可恢復的錯誤（EACCES, EIO）→ fail-closed
}
```

### 10.4 validateTeamMemWritePath — 雙重驗證

```
寫入團隊記憶檔案前的完整驗證鏈：

第1遍：字串級
  1. Null 位元組檢查
  2. path.resolve() 解析
  3. startsWith(teamDir) 驗證

第2遍：符號連結級
  4. realpathDeepestExisting() 解析實際路徑
  5. isRealPathWithinTeamDir() 驗證實際路徑在團隊目錄內

透過 → 返回解析後的路徑
失敗 → 丟擲 PathTraversalError
```

### 10.5 字首攻擊防護

```typescript
// 團隊目錄路徑以分隔符結尾：/foo/team/
// 這樣 /foo/team-evil/ 不會透過 startsWith 檢查
```

---

## 11. 記憶的生命週期全景

```
會話開始
  │
  ├─ loadMemoryPrompt()
  │   → 讀取 MEMORY.md → 注入系統提示
  │
  ├─ initSessionMemory()
  │   → 註冊後取樣鉤子
  │
  ├─ startTeamMemoryWatcher()
  │   → 拉取服務端 → 啟動檔案監聽
  │
  ▼
每輪對話
  │
  ├─ findRelevantMemories(query)
  │   → 掃描檔案 → Sonnet 選擇 → 返回最多 5 條
  │   → 新鮮度標註（>1天 → 加過期警告）
  │
  ├─ [模型可能直接讀/寫記憶檔案]
  │   → 寫入團隊記憶 → checkTeamMemSecrets() 攔截秘密
  │   → 檔案監聽觸發 → 2秒後推送到服務端
  │
  ├─ Session Memory 後取樣鉤子
  │   → 檢查閾值 → Fork Agent 更新單檔案
  │
  ├─ 模型最終響應（無工具呼叫）
  │   → Stop Hooks → executeExtractMemories()
  │   → 互斥檢查 → Fork Agent 提取 → 更新持久記憶檔案
  │
  ▼
自動壓縮觸發時
  │
  ├─ trySessionMemoryCompaction()（優先路徑）
  │   → 用 Session Memory 內容作為摘要
  │   → 保留最近 10K-40K tokens 的訊息
  │
  ├─ compactConversation()（回退路徑）
  │   → Fork Agent 生成 9 章節結構化摘要
  │
  ▼
會話結束
  │
  ├─ drainPendingExtraction()
  │   → 等待進行中的記憶提取完成
  │
  └─ 團隊記憶推送最後一批變更
```

---

## 12. 關鍵設計決策總結

| 決策 | 原因 |
|------|------|
| 記憶檔案是獨立的 .md 檔案，不是資料庫 | 使用者可以直接編輯、Git 追蹤、跨工具使用 |
| MEMORY.md 是索引不是記憶 | 防止單檔案膨脹，支援截斷不丟核心內容 |
| 自動提取在 Stop Hook 中即發即忘 | 不阻塞主對話迴圈 |
| 互斥檢查（主代理 vs 自動提取） | 防止覆蓋使用者顯式儲存的內容 |
| 召回用 Sonnet 不用 Haiku | 需要理解語義相關性，Haiku 不夠準確 |
| 新鮮度用"X days ago"不用 ISO 時間 | 模型對相對時間的推理比絕對時間好 |
| 專案設定排除 autoMemoryDirectory | 防止惡意倉庫劫持寫入路徑 |
| 團隊記憶雙重路徑驗證 | 符號連結攻擊需要在檔案系統層面檢查 |
| 秘密模式在執行時拼接 | 避免自身的 excluded-strings 檢查 |
| 本地優先的衝突策略 | 使用者正在編輯 = 最新意圖 |
| 斷路器（MAX_CONFLICT_RETRIES = 2） | 防止推送衝突的無限重試 |

---

*來自：AI超元域 | B站頻道：https://space.bilibili.com/3493277319825652*

*基於 Claude Code 原始碼逆向分析，2026-03-31*
