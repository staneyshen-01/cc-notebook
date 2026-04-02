# Claude Code 多 Agent 架構深度分析

> 基於原始碼約 **13,700 行、37+ 檔案**的深度探索，涵蓋 `src/utils/swarm/`、`src/tools/AgentTool/`、`src/coordinator/` 等核心模組。

---

## 一、整體架構

系統採用 **Leader-Worker 模型**，核心有三種執行後端和兩種編排模式：

```
                        ┌─────────────────────────────┐
                        │     使用者請求 (REPL/CLI)       │
                        └──────────┬──────────────────┘
                                   │
                        ┌──────────▼──────────────────┐
                        │     Leader Agent (team-lead)  │
                        │     QueryEngine → query()     │
                        └──┬────────┬────────┬────────┘
                           │        │        │
              ┌────────────▼┐  ┌────▼─────┐  ┌▼────────────┐
              │  InProcess   │  │  Tmux    │  │  iTerm2     │
              │  (同程序)     │  │ (分屏)   │  │  (分屏)     │
              └──────────────┘  └──────────┘  └─────────────┘
                     │               │              │
              AsyncLocalStorage   獨立程序         獨立程序
              隔離上下文          檔案郵箱通訊      檔案郵箱通訊
```

**兩種編排模式：**

- **Team Mode** — 透過 `TeamCreate` 建立團隊，手動管理 Worker
- **Coordinator Mode** — 自動化編排，Coordinator 只能用 Agent/TaskStop/SendMessage 三個工具

---

## 二、Agent Tool — 統一入口

`src/tools/AgentTool/AgentTool.tsx` 是所有子 Agent 的統一入口。

### 2.1 輸入引數

| 引數 | 型別 | 作用 |
|------|------|------|
| `prompt` | string | 任務描述 |
| `subagent_type` | string | Agent 型別 (Explore, Plan, 自定義等) |
| `model` | `'sonnet'\|'opus'\|'haiku'` | 模型覆蓋 |
| `run_in_background` | boolean | 後臺非同步執行 |
| `name` | string | 命名，使其可透過 SendMessage 定址 |
| `team_name` | string | 團隊名 |
| `mode` | string | 許可權模式 (plan, acceptEdits 等) |
| `isolation` | `'worktree'` | Git worktree 隔離 |

### 2.2 路由決策樹

```
call() 入口 (AgentTool.tsx:284)
  ├── team_name + name → spawnTeammate() → 生成持久化隊友
  ├── isolation === 'remote' → Cloud Code Runner
  ├── run_in_background === true → runAsyncAgentLifecycle() (fire-and-forget)
  └── 預設 → runAgent() 同步執行，阻塞父 Agent
```

### 2.3 關鍵常量

```typescript
// src/tools/AgentTool/constants.ts
AGENT_TOOL_NAME = 'Agent'
LEGACY_AGENT_TOOL_NAME = 'Task'  // 向後相容
ONE_SHOT_BUILTIN_AGENT_TYPES = {'Explore', 'Plan'}
```

---

## 三、Agent 定義系統

`src/tools/AgentTool/loadAgentsDir.ts` (755行) 負責從多來源載入 Agent 定義。

### 3.1 AgentDefinition 型別

```typescript
type AgentDefinition = {
  agentType: string           // 唯一標識
  whenToUse: string           // 何時使用的描述
  tools: string[]             // 可用工具列表，['*'] 表示全部
  disallowedTools: string[]   // 禁用工具
  model: string               // 模型選擇
  permissionMode: string      // 許可權模式
  maxTurns: number            // 最大對話輪次
  background: boolean         // 是否後臺執行
  isolation: 'worktree'       // 隔離方式
  getSystemPrompt()           // 系統提示詞生成函式
}
```

### 3.2 三種來源

- `BuiltInAgentDefinition` — 內建 Agent (source: 'built-in')
- `CustomAgentDefinition` — 使用者自定義 `.claude/agents/*.md` 檔案
- `PluginAgentDefinition` — 外掛提供的 Agent

### 3.3 內建 Agent 型別

| Agent | 模型 | 工具 | 特點 |
|-------|------|------|------|
| `general-purpose` | 預設 | `['*']` | 全能通用 |
| `Explore` | haiku | 只讀工具 | 程式碼探索，禁止編輯/寫入/巢狀 Agent |
| `Plan` | inherit | 只讀工具 | 架構規劃，禁止編輯 |
| `verification` | inherit | 只讀 | 後臺對抗性測試，紅色標識 |
| `statusline-setup` | sonnet | Read, Edit | 狀態列配置專用 |
| `fork` (合成) | inherit | `['*']` | 繼承父上下文，許可權冒泡 |
| `claude-code-guide` | — | 有限工具 | Claude Code 使用指南 |

### 3.4 載入優先順序

```
built-in < plugin < userSettings < projectSettings < flagSettings < policySettings
```

後載入的同名 Agent 覆蓋先載入的。

---

## 四、三大執行後端

### 4.1 InProcess 後端 (同程序)

**檔案**: `src/utils/swarm/backends/InProcessBackend.ts` (339行)

最輕量的方式——所有 Worker 執行在**同一個 Node.js 程序**中：

```
Leader 程序
  ├── AsyncLocalStorage<TeammateContext> #1 → Worker A
  ├── AsyncLocalStorage<TeammateContext> #2 → Worker B
  └── AsyncLocalStorage<TeammateContext> #3 → Worker C
```

**核心流程：**

1. `spawn()` → `spawnInProcessTeammate()` 註冊到 AppState
2. `startInProcessTeammate()` fire-and-forget 啟動 Agent 迴圈
3. 透過 `runWithTeammateContext(context, fn)` 包裹執行，實現上下文隔離
4. 許可權請求直接使用 Leader 的 `ToolUseConfirm` 對話方塊（透過 `leaderPermissionBridge.ts`）

**上下文隔離** (`src/utils/teammateContext.ts`, 96行)：

```typescript
// AsyncLocalStorage 實現程序內隔離
const teammateContextStorage = new AsyncLocalStorage<TeammateContext>()

function runWithTeammateContext(context: TeammateContext, fn: () => T): T {
  return teammateContextStorage.run(context, fn)
}

function isInProcessTeammate(): boolean {
  return teammateContextStorage.getStore() !== undefined
}
```

### 4.2 Tmux 後端 (終端分屏)

**檔案**: `src/utils/swarm/backends/TmuxBackend.ts` (764行)

每個 Worker 是一個**獨立的 Claude CLI 程序**，執行在 tmux 分屏中。

**兩種模式：**

**Leader 在 tmux 內：**
```
┌─────────────────┬──────────────────┐
│                 │    Worker A      │
│    Leader       ├──────────────────┤
│    (30%)        │    Worker B      │
│                 ├──────────────────┤
│                 │    Worker C      │
└─────────────────┴──────────────────┘
         main-vertical 佈局
```

**Leader 在 tmux 外：** 建立 `claude-swarm-{pid}` 獨立 session，tiled 佈局。

**關鍵細節：**
- `acquirePaneCreationLock()` 序列化並行 spawn，防止競態
- 200ms shell 初始化延遲等待 rc 檔案載入
- 透過 `tmux send-keys` 向 pane 傳送命令
- 支援 `hidePane()` (break-pane) / `showPane()` (join-pane)

### 4.3 iTerm2 後端

**檔案**: `src/utils/swarm/backends/ITermBackend.ts` (370行)

透過 `it2` CLI (Python 自動化工具) 控制 iTerm2：

- 首個 Worker 垂直分割，後續水平分割
- 死會話恢復：split 失敗時驗證 session list 並重試
- `setPaneBorderColor`/`rebalancePanes` 為 no-op（避免頻繁啟動 Python 程序）
- `supportsHideShow = false`（無等效的 tmux break-pane）

### 4.4 後端選擇邏輯

`src/utils/swarm/backends/registry.ts` 的 `detectAndGetBackend()` (line 136)：

```
優先順序瀑布:
1. 在 tmux 內？ → TmuxBackend (原生)
2. 在 iTerm2 且有 it2 CLI？ → ITermBackend
3. 在 iTerm2 無 it2？ → 回退 tmux，提示安裝 it2
4. 有 tmux？ → TmuxBackend (外部 session)
5. 非互動模式 (-p)？ → InProcessBackend
6. teammateMode 配置為 'in-process'？ → InProcessBackend
7. 都沒有 → 報錯並給出平臺特定安裝指引
```

---

## 五、核心通訊機制 — 檔案郵箱

### 5.1 郵箱結構

`src/utils/teammateMailbox.ts` (1,183行)

每個 Agent 擁有一個 JSON 檔案郵箱：

```
~/.claude/teams/{team_name}/inboxes/{agent_name}.json
```

```typescript
type TeammateMessage = {
  from: string       // 傳送者名稱
  text: string       // 訊息內容
  timestamp: string  // 時間戳
  read: boolean      // 是否已讀
  color?: string     // 傳送者 UI 顏色
  summary?: string   // 5-10 字摘要
}
```

### 5.2 核心操作

| 函式 | 作用 |
|------|------|
| `writeToMailbox()` | 原子寫入 + `proper-lockfile` 檔案鎖（10次重試，5-100ms 退避） |
| `readMailbox()` | 讀取所有訊息 |
| `readUnreadMessages()` | 過濾未讀訊息 |
| `markMessagesAsRead()` | 標記已讀 |

### 5.3 結構化訊息型別

郵箱不僅傳遞文字，還定義了一套協議訊息：

- `createShutdownRequestMessage` / `createShutdownApprovedMessage` — 關閉協議
- `createPermissionRequestMessage` / `createPermissionResponseMessage` — 許可權請求/響應
- `createSandboxPermissionRequestMessage` / `createSandboxPermissionResponseMessage` — 沙箱網路許可權
- `createIdleNotification` — Worker 空閒通知

### 5.4 輪詢機制

| Agent 型別 | 輪詢方式 |
|------------|----------|
| Pane 類 Worker | `useInboxPoller` React hook 定期輪詢 |
| InProcess Worker | `waitForNextPromptOrShutdown()` 直接讀郵箱 |
| Leader | `useInboxPoller` 處理許可權請求、空閒通知等 |

---

## 六、SendMessage Tool — Agent 間通訊

`src/tools/SendMessageTool/SendMessageTool.ts` (917行)

### 6.1 輸入引數

```typescript
{
  to: string      // 收件人名稱、"*" 廣播、"uds:<socket>"、"bridge:<session-id>"
  summary: string // 5-10 字摘要
  message: string | ShutdownRequest | ShutdownResponse | PlanApprovalResponse
}
```

### 6.2 路由邏輯

```
SendMessage.call() (line 741)
  ├── to: "bridge:<session-id>" → postInterClaudeMessage (跨機器 Remote Control)
  ├── to: "uds:<socket>"        → sendToUdsSocket (Unix Domain Socket 本地通訊)
  ├── to: "*"                   → 廣播到所有團隊成員郵箱
  ├── to: "<name>"              → 查 agentNameRegistry:
  │     ├── InProcess Agent running → queuePendingMessage (記憶體佇列)
  │     ├── InProcess Agent stopped → resumeAgentBackground (自動喚醒!)
  │     └── Pane Agent → writeToMailbox (檔案郵箱)
  └── 結構化訊息 → 分發到對應處理器
```

**關鍵設計：Agent 的純文字輸出對其他 Agent 不可見，必須透過 SendMessage 通訊。**

---

## 七、許可權委託系統

### 7.1 完整流程

`src/utils/swarm/permissionSync.ts` (928行)

```
Worker 遇到需要許可權的操作
    ↓
檢查 bash classifier 能否自動批准 (swarmWorkerHandler.ts:52)
    ↓ 不能
建立 PermissionRequest 物件 (line 71)
    ↓
註冊回撥 BEFORE 傳送請求 (避免競態, line 79)
    ↓
傳送 permission_request 到 Leader 郵箱 (line 123)
    ↓
顯示 "waiting for leader" 狀態 (line 126)
    ↓
Leader 輪詢檢測到請求
    ↓
在 Leader UI 展示許可權對話方塊（帶 worker badge）
    ↓
使用者批准/拒絕
    ↓
Leader 傳送 permission_response 到 Worker 郵箱
    ↓
Worker 輪詢獲取響應（每 500ms），繼續執行
```

### 7.2 InProcess 最佳化路徑

透過 `leaderPermissionBridge.ts` (54行)，InProcess Worker 直接使用 Leader 的 `ToolUseConfirm` 佇列：

```typescript
// REPL 註冊許可權 UI
registerLeaderPermissionQueue(setToolUseConfirmQueue, setToolPermissionContext)

// Worker 推送許可權請求到 Leader UI，帶 worker badge
pushToLeaderPermissionQueue(request, workerBadge)
```

跳過檔案郵箱的 I/O 開銷，提供與 Leader 自身工具相同的 UI 體驗。

### 7.3 SwarmPermissionRequest 結構

```typescript
type SwarmPermissionRequest = {
  id: string
  workerId: string
  workerName: string
  toolName: string
  toolUseId: string
  description: string
  input: any
  permissionSuggestions: any
  status: 'pending' | 'approved' | 'rejected'
  resolvedBy?: string
  feedback?: string
  updatedInput?: any
  permissionUpdates?: any
}
```

### 7.4 許可權模式繼承

當 Leader 批准 Worker 的 plan 時：

```typescript
const modeToInherit = leaderMode === 'plan' ? 'default' : leaderMode
```

團隊檔案中每個成員有獨立的 `mode` 欄位，支援透過 `setMemberMode()` / `setMultipleMemberModes()` 動態調整。

---

## 八、Agent 執行引擎

### 8.1 runAgent() — 子 Agent 核心驅動

`src/tools/AgentTool/runAgent.ts` (973行)

`runAgent()` 是一個 `AsyncGenerator<Message, void>`：

```
runAgent() (line 248)
  1. 解析模型 → getAgentModel()
  2. 建立隔離上下文 → createSubagentContext()
     - fork 父訊息到獨立上下文
     - 隔離的檔案狀態快取
  3. 初始化 Agent 專屬 MCP servers → initializeAgentMcpServers()
  4. 解析工具集 → resolveAgentTools()
  5. 構建系統提示詞 (Agent 自身 + 環境資訊)
  6. 註冊 Perfetto 追蹤, hooks
  7. 呼叫 query() — 與主 REPL 完全相同的 agentic 迴圈
  8. yield 每條訊息回父 Agent
  9. 清理 MCP servers, shell tasks, agent tracking
```

**核心設計：子 Agent 複用完全相同的 `query()` 迴圈**，遞迴地擁有完整的工具執行能力。

### 8.2 非同步 Agent 生命週期

`agentToolUtils.ts` 的 `runAsyncAgentLifecycle()` (line 508)：

```
registerAsyncAgent() → 建立 task entry
    ↓
註冊 name 到 agentNameRegistry (用於 SendMessage 路由)
    ↓
void runWithAgentContext(...) → fire-and-forget
    ↓
迭代 runAgent() 的訊息流
    ↓
ProgressTracker 追蹤:
  - tool use 次數
  - token 消耗
  - 最近活動
    ↓
完成時 finalizeAgentTool():
  - 提取最終文字內容
  - 計算 totalDurationMs, totalTokens, totalToolUseCount
    ↓
completeAsyncAgent() → 標記 task 完成
    ↓
傳送 <task-notification> XML 通知父 Agent
```

**多個非同步 Agent 是真正並行的**——每個都是獨立的 async 操作。

### 8.3 工具過濾

`agentToolUtils.ts` 的 `filterToolsForAgent()` (line 70) 使用多層禁用列表：

| 禁用列表 | 適用範圍 | 禁用的工具 |
|----------|----------|-----------|
| `ALL_AGENT_DISALLOWED_TOOLS` | 所有 Agent | TaskOutput, ExitPlanMode, EnterPlanMode, AskUserQuestion, TaskStop |
| `CUSTOM_AGENT_DISALLOWED_TOOLS` | 非內建 Agent | 同上 + AgentTool |
| `ASYNC_AGENT_ALLOWED_TOOLS` | 後臺 Agent | 白名單制：Read, Grep, Glob, Edit, Write, WebSearch 等 |
| `COORDINATOR_MODE_ALLOWED_TOOLS` | Coordinator | 僅 Agent, TaskStop, SendMessage, SyntheticOutput |

---

## 九、Coordinator Mode — 自動化編排

`src/coordinator/coordinatorMode.ts` (369行)

### 9.1 啟用條件

```typescript
function isCoordinatorMode(): boolean {
  return feature('COORDINATOR_MODE') && process.env.CLAUDE_CODE_COORDINATOR_MODE === '1'
}
```

### 9.2 工作流程

```
Coordinator (只能用 Agent + TaskStop + SendMessage)
    │
    ├── Research Phase → 並行派出多個 Worker 調研
    │       ├── Worker A: 搜尋程式碼
    │       ├── Worker B: 讀取文件
    │       └── Worker C: 分析依賴
    │
    ├── Synthesis Phase → Coordinator 綜合分析研究結果
    │
    ├── Implementation Phase → Worker 執行程式碼實現
    │
    └── Verification Phase → Worker 驗證實現結果
```

### 9.3 Coordinator 系統提示詞要點

`getCoordinatorSystemPrompt()` (line 111) 的 369 行提示詞規定：

- Coordinator 透過 `Agent` 工具派出 Worker，透過 `SendMessage` 繼續對話，透過 `TaskStop` 終止
- Worker 完成時傳送 `<task-notification>` XML
- 明確的 spawn vs continue 決策指導
- 併發管理策略

---

## 十、Team Mode — 持久化協作

### 10.1 團隊檔案結構

`~/.claude/teams/{team-name}/config.json`：

```typescript
type TeamFile = {
  name: string
  description: string
  createdAt: string
  leadAgentId: string
  leadSessionId: string
  hiddenPaneIds: string[]
  teamAllowedPaths: string[]
  members: [{
    agentId: string          // 格式: name@teamName
    name: string
    agentType: string
    model: string
    color: string
    planModeRequired: boolean
    joinedAt: string
    tmuxPaneId?: string
    cwd: string
    worktreePath?: string
    sessionId: string
    subscriptions: string[]
    backendType: 'tmux' | 'iterm2' | 'in-process'
    isActive: boolean
    mode: string
  }]
}
```

### 10.2 團隊工作流

```
1. TeamCreate → 建立 config.json + tasks 目錄
2. TaskCreate → 在 ~/.claude/tasks/{team-name}/ 建立任務
3. Agent tool (team_name + name) → 派出 teammate
4. TaskUpdate → 分配任務 owner
5. Teammate 工作 → 完成後自動傳送 idle notification
6. SendMessage(shutdown_request) → 優雅關閉 teammate
```

### 10.3 Teammate 系統提示詞附加

`src/utils/swarm/teammatePromptAddendum.ts`：

> "You are running as an agent in a team. To communicate with anyone on your team, use the SendMessage tool. Just writing a response in text is not visible to others."

---

## 十一、Worktree 隔離

`src/utils/worktree.ts` 的 `createAgentWorktree()` (line 902)

### 11.1 建立流程

```
Agent spawn 時 isolation='worktree'
  → 生成 slug: agent-{agentId前8字元}
  → validateWorktreeSlug() — 拒絕路徑穿越，限制64字元
  → 路徑: .claude/worktrees/<slug>
  → 分支: worktree-<slug>
  → git worktree add -B worktree-<slug>
  → performPostCreationSetup():
      - 複製 settings.local.json
      - 配置 git hooks
      - 符號連結目錄
      - 複製 .worktreeinclude 檔案
```

### 11.2 清理流程

```
Agent 完成時:
  → hasWorktreeChanges()?
      ├── 有改動 → 保留 worktree，在 <task-notification> 中報告路徑和分支
      └── 無改動 → removeAgentWorktree() 自動清理
```

---

## 十二、Worker 完整生命週期

### 12.1 誕生

```
Leader 呼叫 Agent tool
  ├── InProcess: spawnInProcessTeammate()
  │     1. 生成 agentId = name@teamName
  │     2. 建立獨立 AbortController（不連結到 Leader）
  │     3. 建立 TeammateContext for AsyncLocalStorage
  │     4. 註冊 InProcessTeammateTaskState 到 AppState
  │     5. startInProcessTeammate() → runWithTeammateContext() → runAgent()
  │
  └── Pane: PaneBackendExecutor.spawn()
        1. assignTeammateColor() 分配顏色
        2. createTeammatePaneInSwarmView() 建立分屏
        3. 構建 CLI 命令:
           claude --agent-id X --agent-name Y --team-name Z
                  --agent-color C --parent-session-id P
        4. sendCommandToPane() 傳送命令
        5. writeToMailbox() 寫入初始 prompt
```

### 12.2 執行

```
InProcess Runner (inProcessRunner.ts, 1552行):
  runWithTeammateContext() 包裹:
    → runAgent() → query() 迴圈
    → 輪詢郵箱獲取新訊息（注入為 user-turn messages）
    → 許可權請求轉發到 Leader UI
    → ProgressTracker 追蹤工具使用和 token
    → 支援 auto-compact（上下文過大時自動壓縮）
```

### 12.3 通訊

```
Worker → Leader: writeToMailbox() 或 queuePendingMessage (InProcess)
Leader → Worker: writeToMailbox() 或 SendMessage
Worker → Worker: SendMessage tool → writeToMailbox()
廣播: SendMessage(to: "*") → 寫入所有成員郵箱
```

### 12.4 終止

三條終止路徑：

| 方式 | 觸發 | 流程 |
|------|------|------|
| 優雅關閉 | `shutdown_request` via mailbox | Worker 處理請求 → 退出 → idle notification |
| 強制終止 (InProcess) | `killInProcessTeammate()` | abort AbortController → 更新 task 狀態為 'killed' → 移出團隊檔案 |
| 強制終止 (Pane) | `backend.killPane(paneId)` | 殺死 tmux/iTerm2 pane |

### 12.5 會話清理

`cleanupSessionTeams()` 在 Leader 退出時執行：
- 殺死所有孤立 pane
- 移除團隊和任務目錄
- 銷燬 git worktree

---

## 十三、Feature Gates 彙總

| Feature Flag | 控制內容 | 啟用方式 |
|-------------|---------|---------|
| `COORDINATOR_MODE` | Coordinator 自動編排 | 編譯時 + `CLAUDE_CODE_COORDINATOR_MODE=1` |
| `FORK_SUBAGENT` | Fork 模式子 Agent | 編譯時 feature gate |
| `BUILTIN_EXPLORE_PLAN_AGENTS` | Explore/Plan 內建 Agent | 編譯時 + A/B test `tengu_amber_stoat` |
| `VERIFICATION_AGENT` | 驗證 Agent | 編譯時 + gate `tengu_hive_evidence` |
| Agent Teams (Swarm) | 團隊協作模式 | `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` + GrowthBook gate |

---

## 十四、通訊協議總覽

| 通道 | 機制 | 方向 | 用途 |
|------|------|------|------|
| 檔案郵箱 `inboxes/{name}.json` | JSON + 檔案鎖 | 任意 Agent → 任意 Agent | 所有 teammate 訊息、許可權、關閉協議 |
| 記憶體佇列 `pendingUserMessages` | AppState | 父 → InProcess 子 | 執行中 Agent 的即時訊息 |
| Leader Permission Bridge | 模組級函式註冊 | InProcess Worker → Leader UI | 許可權彈窗 |
| UDS Socket | Unix Domain Socket | Session → Session (本地) | 跨 session 訊息 |
| Bridge (Remote Control) | HTTP via Anthropic | Session → Session (跨機器) | 遠端控制訊息 |
| Task 系統 | 檔案 `~/.claude/tasks/` | 所有團隊成員 | 工作項追蹤、狀態更新 |
| `<task-notification>` XML | 注入對話 | 後臺 Agent → 父 Agent | 完成/失敗通知 |

---

## 十五、架構規模統計

| 層次 | 核心元件 | 程式碼規模 |
|------|---------|---------|
| 入口 | AgentTool | ~800行 |
| 執行引擎 | runAgent + query() 迴圈 | ~2,700行 |
| Agent 定義 | loadAgentsDir + builtInAgents | ~850行 |
| 後端管理 | registry + 3個後端實現 | ~1,800行 |
| 通訊 | teammateMailbox + SendMessageTool | ~2,100行 |
| 許可權委託 | permissionSync + leaderPermissionBridge | ~1,000行 |
| 團隊管理 | teamHelpers + spawnMultiAgent | ~1,800行 |
| 上下文隔離 | agentContext + teammateContext | ~300行 |
| Coordinator | coordinatorMode + workerAgent | ~400行 |
| InProcess Runner | inProcessRunner | ~1,550行 |
| **總計** | **37+ 檔案** | **~13,700行** |

---

## 十六、核心設計理念

1. **query() 迴圈複用** — 子 Agent 使用與主 REPL 完全相同的 agentic 迴圈，天然具備完整工具能力
2. **AsyncLocalStorage 隔離** — InProcess 模式下零 IPC 開銷，透過 Node.js 原生機制隔離上下文
3. **檔案郵箱統一通訊** — 跨程序、跨後端的統一訊息協議，用檔案鎖保證併發安全
4. **許可權始終由人類把控** — 無論多少層 Agent 巢狀，危險操作最終都路由到 Leader 的 UI 由使用者決定
5. **優雅降級** — 從 iTerm2 → tmux → InProcess，自動選擇最佳可用後端
6. **Fire-and-forget 非同步** — 後臺 Agent 透過 `<task-notification>` XML 非侵入式通知父 Agent
