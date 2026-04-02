# Claude Code 隱藏高階功能全景圖

> 基於 Claude Code 原始碼（2026-03-31 快照，512K 行 TypeScript）逆向分析
> 發現 55+ 個特性開關、20+ 個隱藏命令、以及多個未公開系統

---

## 目錄

1. [你現在就能用的隱藏功能](#1-你現在就能用的隱藏功能)
2. [55+ 個編譯時特性開關完整清單](#2-55-個編譯時特性開關完整清單)
3. [KAIROS — AI 助手守護程序](#3-kairos--ai-助手守護程序)
4. [CHICAGO_MCP — Computer Use 電腦控制](#4-chicago_mcp--computer-use-電腦控制)
5. [投機執行系統（Speculation）](#5-投機執行系統speculation)
6. [Undercover 隱身模式](#6-undercover-隱身模式)
7. [Dream Mode 記憶夢境整合](#7-dream-mode-記憶夢境整合)
8. [Voice Mode 語音輸入](#8-voice-mode-語音輸入)
9. [Proactive 自主代理模式](#9-proactive-自主代理模式)
10. [伴侶精靈系統（Buddy）](#10-伴侶精靈系統buddy)
11. [多代理團隊/Swarm 系統](#11-多代理團隊swarm-系統)
12. [Anthropic 內部專用命令](#12-anthropic-內部專用命令)
13. [隱藏鍵盤快捷鍵](#13-隱藏鍵盤快捷鍵)
14. [隱藏環境變數](#14-隱藏環境變數)
15. [其他彩蛋與冷知識](#15-其他彩蛋與冷知識)

---

## 1. 你現在就能用的隱藏功能

這些功能在當前公開版本中存在，但很少被提及：

| 功能 | 觸發方式 | 說明 |
|------|----------|------|
| 實體貼紙 | `/stickers` | 開啟 Claude Code 實體貼紙購買頁 |
| 堆轉儲 | `/heapdump` | 把 JS 堆轉儲到 `~/Desktop`（隱藏命令，不在 /help 中） |
| 裸模式 | `--bare` 或 `CLAUDE_CODE_SIMPLE=1` | 極簡模式：只保留 Bash、Read、Edit 三個工具 |
| 自定義載入詞 | `settings.json` → `spinnerVerbs` | 替換或追加 200+ 個花式載入詞（"Clauding"、"Flibbertigibbeting"…） |
| 輸出風格切換 | `/config` → output style | **Explanatory**（教學模式，附 Insight 解說）或 **Learning**（動手練習，含 TODO(human) 標記） |
| Vim 模式 | `/vim` | 完整狀態機：d/c/y 運算子、h/l/w/b/e/$ 移動、f/t 查詢、文字物件、dot-repeat |
| 訊息動作選擇器 | `shift+up` | 進入訊息導航：j/k 上下移動，c 複製訊息，p 固定訊息 |
| 年度回顧 | `/think-back` | "Your 2025 Claude Code Year in Review" 動畫回顧 |
| 轉儲系統提示 | `--dump-system-prompt` | 列印完整系統提示詞後退出（需特性開關） |
| 多代理團隊 | `--agent-teams` 或 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` | 多 Claude 例項在 tmux/iTerm 分屏中並行工作 |
| 後臺任務 | `ctrl+b` | 把當前執行中的任務轉入後臺 |
| 歷史搜尋 | `ctrl+r` | 反向搜尋命令歷史 |
| Transcript 模式 | `ctrl+o` | 檢視完整對話記錄，支援搜尋 |
| Todo 面板 | `ctrl+t` | 任務追蹤面板 |
| 速率限制選項 | `/rate-limit-options` | 隱藏命令，速率限制時顯示選項 |
| 使用洞察 | `/insights` | 分析你的 Claude Code 使用歷史（113KB 延遲載入模組） |
| 全域性檔案搜尋 | `ctrl+shift+f` | 跨檔案搜尋（需 QUICK_SEARCH 開關） |
| 快速開啟 | `ctrl+shift+p` | 快速開啟檔案 |

---

## 2. 55+ 個編譯時特性開關完整清單

所有開關透過 `feature('FLAG')` 在 Bun 編譯時評估。外部構建中為 `false` 的分支會被徹底刪除（死程式碼消除）。

### 核心功能開關

| 開關 | 功能 | 狀態 |
|------|------|:----:|
| `VOICE_MODE` | 語音輸入（push-to-talk） | 內部 |
| `KAIROS` | AI 助手守護程序模式 | 內部 |
| `KAIROS_DREAM` | 夢境記憶整合 | 內部 |
| `KAIROS_BRIEF` | Brief 摘要工具 | 內部 |
| `KAIROS_CHANNELS` | 頻道通知系統 | 內部 |
| `KAIROS_PUSH_NOTIFICATION` | 推送通知工具 | 內部 |
| `KAIROS_GITHUB_WEBHOOKS` | GitHub PR 訂閱 | 內部 |
| `CHICAGO_MCP` | Computer Use 電腦控制 | 付費使用者 |
| `PROACTIVE` | 自主代理模式（Sleep + 自動喚醒） | 內部 |
| `BRIDGE_MODE` | Remote Control 橋接 | 公開 |
| `DAEMON` | 後臺守護程序 | 內部 |

### 上下文管理開關

| 開關 | 功能 |
|------|------|
| `CONTEXT_COLLAPSE` | 智慧上下文摺疊（替代暴力壓縮） |
| `REACTIVE_COMPACT` | 響應式壓縮（API 報 prompt-too-long 時觸發） |
| `CACHED_MICROCOMPACT` | 快取感知微壓縮（不破壞 prompt cache） |
| `HISTORY_SNIP` | 歷史裁剪（SnipTool + /force-snip） |
| `TOKEN_BUDGET` | Token 預算追蹤和遞減收益檢測 |
| `EXTRACT_MEMORIES` | 自動記憶提取 |
| `TEAMMEM` | 團隊記憶同步 |

### 工具和代理開關

| 開關 | 功能 |
|------|------|
| `COORDINATOR_MODE` | 多代理協調者模式 |
| `AGENT_TRIGGERS` | 定時觸發器（Cron） |
| `AGENT_TRIGGERS_REMOTE` | 遠端觸發器 |
| `MONITOR_TOOL` | 監控工具和任務 |
| `WEB_BROWSER_TOOL` | 網頁瀏覽器自動化工具 |
| `TERMINAL_PANEL` | 終端面板（meta+j） |
| `WORKFLOW_SCRIPTS` | 工作流指令碼系統 |
| `BG_SESSIONS` | 後臺會話（claude ps/logs/attach/kill） |
| `OVERFLOW_TEST_TOOL` | 上下文溢位測試工具 |
| `BUDDY` | 伴侶精靈系統 |

### UI 和體驗開關

| 開關 | 功能 |
|------|------|
| `MESSAGE_ACTIONS` | 訊息動作選擇器（shift+up） |
| `QUICK_SEARCH` | 全域性搜尋和快速開啟 |
| `AUTO_THEME` | 自動主題檢測 |
| `STREAMLINED_OUTPUT` | 流式 JSON 模式最佳化輸出 |
| `CONNECTOR_TEXT` | 聯結器文字摘要 beta |

### 安全和除錯開關

| 開關 | 功能 |
|------|------|
| `BASH_CLASSIFIER` | Bash 命令 ML 安全分類器 |
| `TRANSCRIPT_CLASSIFIER` | 基於 transcript 的許可權模式分類 |
| `NATIVE_CLIENT_ATTESTATION` | 客戶端證明頭（cch=） |
| `HARD_FAIL` | 硬失敗模式（任何錯誤都崩潰） |
| `DUMP_SYSTEM_PROMPT` | --dump-system-prompt 隱藏 CLI 標誌 |
| `ABLATION_BASELINE` | A/B 測試消融基線 |
| `PROMPT_CACHE_BREAK_DETECTION` | Prompt 快取失效檢測 |

### 基礎設施開關

| 開關 | 功能 |
|------|------|
| `LODESTONE` | `cc://` 深度連結協議註冊 |
| `DIRECT_CONNECT` | 透過 cc:// 深度連結直連 |
| `SSH_REMOTE` | SSH 遠端會話 |
| `BYOC_ENVIRONMENT_RUNNER` | 自帶計算環境執行器 |
| `SELF_HOSTED_RUNNER` | 自託管執行器 |
| `CCR_REMOTE_SETUP` | /remote-setup 命令 |
| `FORK_SUBAGENT` | /fork 子代理分叉命令 |
| `MCP_SKILLS` | MCP 提供的技能 |
| `EXPERIMENTAL_SKILL_SEARCH` | 實驗性技能模糊搜尋 |
| `REVIEW_ARTIFACT` | 審查工件工具 |
| `BUILDING_CLAUDE_APPS` | /claude-api 技能 |
| `RUN_SKILL_GENERATOR` | 技能生成器 |
| `ULTRAPLAN` | 超級計劃模式 |
| `TORCH` | /torch 命令 |
| `TEMPLATES` | 任務分類器/模板系統 |
| `VERIFICATION_AGENT` | 驗證代理 |
| `COMMIT_ATTRIBUTION` | 提交歸屬（Co-Authored-By） |
| `AWAY_SUMMARY` | 離開摘要生成 |
| `FILE_PERSISTENCE` | 跨輪次檔案持久化追蹤 |
| `MEMORY_SHAPE_TELEMETRY` | 記憶檔案形狀遙測 |
| `COWORKER_TYPE_TELEMETRY` | Coworker 型別遙測 |
| `DOWNLOAD_USER_SETTINGS` | 設定下載同步 |
| `UPLOAD_USER_SETTINGS` | 設定上傳同步 |
| `BREAK_CACHE_COMMAND` | /break-cache 命令 |

---

## 3. KAIROS — AI 助手守護程序

**代號含義：** KAIROS（希臘語，"決定性時刻"）

### 架構

```
claude assistant                    ← 啟動入口
  ↓
永駐守護程序
  ├── 每日日誌: logs/YYYY/MM/YYYY-MM-DD.md（僅追加）
  ├── SleepTool: 休眠 → 定時自主喚醒
  ├── PushNotificationTool: 向使用者推送通知
  ├── SendUserFileTool: 傳送檔案給使用者
  ├── SubscribePRTool: 訂閱 GitHub PR webhook
  ├── BriefTool: 簡短狀態報告（ctrl+shift+b）
  ├── 頻道通知: 透過 MCP 接收外部訊息
  └── /dream: 夜間記憶蒸餾
```

### 關鍵特性

- **記憶正規化不同**：不寫獨立記憶檔案，而是追加到每日日誌
- **夜間夢境**：`/dream` 技能自動將日誌蒸餾為主題檔案
- **自主迴圈**：SleepTool + 自動喚醒 = 完全自主的工作迴圈
- **壓縮後行為**：提示詞包含 "你正在自主模式中執行，這不是首次喚醒——繼續工作迴圈"
- **涉及 6 個子特性開關**：KAIROS + KAIROS_DREAM + KAIROS_BRIEF + KAIROS_CHANNELS + KAIROS_PUSH_NOTIFICATION + KAIROS_GITHUB_WEBHOOKS

---

## 4. CHICAGO_MCP — Computer Use 電腦控制

**代號：** Chicago（子門控以芝加哥地標命名：`tengu_malort_pedway`）

### 功能

- 建立**程序內 MCP 伺服器**控制螢幕
- macOS 上使用**原生 Swift 執行器**
- 支援 `pixels` 和 `normalized` 兩種座標模式
- 自動列出已安裝應用供模型參考
- **ESC 熱鍵**緊急中斷電腦控制
- 可透過 `--computer-use-mcp` 獨立啟動 MCP 伺服器

### 子功能門控

| 子門控 | 功能 |
|--------|------|
| `pixelValidation` | 畫素座標驗證 |
| `clipboardPasteMultiline` | 多行剪貼簿貼上 |
| `mouseAnimation` | 滑鼠移動動畫 |
| `hideBeforeAction` | 操作前隱藏 UI |
| `autoTargetDisplay` | 自動目標顯示器選擇 |
| `clipboardGuard` | 剪貼簿保護 |

### 訪問控制

- 需要 **Max 或 Pro 訂閱**（外部使用者）
- Anthropic 員工可直接使用（繞過訂閱檢查）
- 有專用許可權 UI（`ComputerUseApproval.tsx`）

---

## 5. 投機執行系統（Speculation）

**僅 Anthropic 內部**（`USER_TYPE === 'ant'`）

### 工作原理

```
1. 系統生成"下一步建議"提示詞
   ↓
2. 使用者看到建議的同時，後臺 Fork Agent 已開始執行
   ↓
3. 檔案寫入到 overlay 目錄（copy-on-write 隔離）
   ~/.claude/tmp/speculation/<pid>/<id>/
   ↓
4. 使用者接受建議 → 投機結果注入對話 + overlay 檔案複製到真實檔案系統
   使用者拒絕/修改 → 丟棄投機結果
```

### 安全約束

| 工具型別 | 投機執行中的許可權 |
|----------|:----------------:|
| 只讀工具（Read, Glob, Grep） | 允許 |
| 寫入工具（Edit, Write） | 僅在許可權模式允許自動接受時 |
| Bash 命令 | 必須透過只讀驗證 |
| 未知工具 | 遇到即暫停（boundary） |

### 限制

- 最多 **20 輪**、**100 條訊息**
- 可以**流水線化**：當前投機完成後預生成下一步建議

### 使用者反饋（內部可見）

```
Speculated 5 tool uses — +12s saved (47s this session)
```

---

## 6. Undercover 隱身模式

**僅 Anthropic 員工**，自動啟用條件：在公開/開源倉庫工作。

### 啟用後的行為

| 行為 | 說明 |
|------|------|
| 剝除歸屬 | 所有 `Co-Authored-By` 行被移除 |
| 禁止提及內部代號 | Capybara、Tengu 等動物代號 |
| 禁止提及版本號 | 未釋出的模型版本 |
| 禁止提及內部倉庫 | Anthropic 內部倉庫名 |
| 禁止提及 Slack 頻道 | 內部 Slack 頻道名 |
| 禁止提及 "Claude Code" | commit 訊息必須看起來像人類寫的 |

### 控制

- 自動檢測：倉庫 remote 不在內部白名單 → 自動開啟
- 強制開啟：`CLAUDE_CODE_UNDERCOVER=1`
- 無強制關閉選項
- 首次啟用有說明對話方塊

---

## 7. Dream Mode 記憶夢境整合

### 自動觸發條件

同時滿足：
- 距上次整合 ≥ **24 小時**（`minHours`，可配置）
- 且觸及 ≥ **5 個會話**（`minSessions`，可配置）
- 無其他程序正在整合（鎖機制）

### 四階段執行

```
階段 1 — 定向
  讀取記憶目錄和 MEMORY.md 索引

階段 2 — 收集
  從每日日誌和會話 transcript 中提取新訊號

階段 3 — 整合
  將新資訊合併到已有主題檔案
  轉換相對日期為絕對日期
  刪除被新事實矛盾的舊事實

階段 4 — 修剪
  保持索引在 25KB 以內
  清理過時條目
```

### 工具限制

Dream 執行期間，Bash 被限制為只讀命令。

### 手動觸發

`/dream` 技能在前臺執行同樣的整合流程（需 KAIROS 或 KAIROS_DREAM 開關）。

---

## 8. Voice Mode 語音輸入

### 啟用方式

**按住空格鍵說話**（push-to-talk），透過 `/voice` 命令開啟。

### 技術棧

```
原生音訊捕獲
  macOS → CoreAudio (audio-capture-napi / cpal)
  Linux → ALSA
  回退 → SoX rec / arecord

錄音引數
  取樣率: 16kHz
  聲道: 單聲道
  靜音檢測: 2.0秒，3% 閾值（SoX 回退路徑）

STT 流式傳輸
  → claude.ai voice_stream 端點
  → 需要 Anthropic OAuth（API key/Bedrock/Vertex 不支援）
```

### UI 元素

- 音訊電平指示器
- 語音狀態顯示
- 語言選擇器（`LanguagePicker.tsx`）
- 自定義關鍵詞詞彙表（`voiceKeyterms.ts`）

### 控制

- GrowthBook 殺開關：`tengu_amber_quartz_disabled`
- 預設 fail-open（新安裝時啟用）

---

## 9. Proactive 自主代理模式

### 核心工具：SleepTool

```typescript
// 代理可以休眠指定時間後自主喚醒
SleepTool.call({ duration: '5m' })
// 5 分鐘後自動恢復執行
```

### 與 KAIROS 結合

```
claude assistant
  ↓
[工作] → [SleepTool 5m] → [自動喚醒] → [繼續工作] → [SleepTool 10m] → ...
  ↑                                                                      |
  └──────────────────────────── 無限迴圈 ────────────────────────────────┘
```

### 壓縮後的特殊行為

普通模式壓縮後：提示使用者是否有問題

自主模式壓縮後：
```
你正在自主/主動模式中執行。這不是首次喚醒——你在壓縮前已經在自主工作。
繼續你的工作迴圈：根據上面的摘要從離開的地方接續。不要問候使用者或詢問要做什麼。
```

---

## 10. 伴侶精靈系統（Buddy）

`feature('BUDDY')` — 一個完整的收藏型寵物系統。

### 物種（18種）

duck、goose、blob、cat、dragon、octopus、owl、penguin、turtle、snail、ghost、axolotl、capybara、cactus、robot、rabbit、mushroom、chonk

### 外觀組合

```
18 種物種 × 6 種眼睛 × 8 種帽子 = 864 種外觀組合
× 5 種稀有度 = 4,320 種可能的精靈
```

### 稀有度

| 等級 | 機率 | 顯示 | 屬性底值 |
|------|:----:|------|:--------:|
| Common | 60% | 1星 | 低 |
| Uncommon | 25% | 2星 | 略高 |
| Rare | 10% | 3星 | 中等 |
| Epic | 4% | 4星 | 高 |
| Legendary | 1% | 5星 | 最高 |

### 5種屬性

- **DEBUGGING** — 除錯力
- **PATIENCE** — 耐心
- **CHAOS** — 混沌
- **WISDOM** — 智慧
- **SNARK** — 吐槽力

每隻精靈有一個巔峰屬性和一個低谷屬性。

### 確定性生成

```typescript
// 種子 = hash(userId + 'friend-2026-401')
// PRNG = Mulberry32
// 無論何時重新生成，同一使用者始終得到同一只精靈
// 編輯配置檔案無法偽造稀有度——bones 從 hash 重新計算
```

### ASCII 動畫

```
每種物種 3 幀動畫，5行×12字元
500ms tick 的閒置動畫
包含眨眼幀和特殊效果（煙霧、天線發光、墨水泡泡等）
```

### 互動

```
/buddy pet → 觸發 2.5 秒心形粒子特效（❤ 字元向上飄浮）
模型可以讓精靈"說話"（透過 speech bubble 系統）
使用者對精靈說話時，模型會自動退讓
```

### 反作弊

```
物種名中某些與模型代號衝突的名稱用 String.fromCharCode 十六進位制編碼
配置檔案編輯不影響生成結果（bones 從 hash 重算）
```

### 啟動引導

首次未孵化時顯示彩虹色 `/buddy` 提示文字。

---

## 11. 多代理團隊/Swarm 系統

### 啟用方式

```bash
# 外部使用者
claude --agent-teams
# 或
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
```

### 三種後端

| 後端 | 實現 | 特點 |
|------|------|------|
| **TmuxBackend** | 隔離 tmux socket `claude-swarm-<PID>` | 彩色邊框，不影響使用者的 tmux |
| **ITermBackend** | iTerm2 的 `it2` CLI + Python API | 原生分屏 |
| **InProcessBackend** | 程序內執行 | 無終端 UI |

自動檢測選擇最佳可用後端。

### Tmux 隔離

```typescript
// Claude 建立自己的 tmux socket，防止命令影響使用者會話
const SWARM_SESSION_NAME = 'claude-swarm'
const HIDDEN_SESSION_NAME = 'claude-hidden'
// socket 名: claude-<PID>
```

### Coordinator 模式

```bash
export CLAUDE_CODE_COORDINATOR_MODE=1
```

Claude 變為協調者，透過 `AgentTool`、`SendMessageTool`、`TaskStopTool` 編排 worker 代理。Worker 擁有獨立工具訪問許可權。

### 團隊工具

| 工具 | 功能 |
|------|------|
| `TeamCreateTool` | 建立代理團隊 |
| `TeamDeleteTool` | 刪除代理團隊 |
| `SendMessageTool` | 代理間訊息傳遞 |
| `TaskStopTool` | 停止子任務 |
| `TungstenTool`（內部） | tmux 會話管理 |

### 共享便籤本

當 `tengu_scratch` 門控開啟時，worker 獲得共享 scratchpad 目錄，跨 worker 知識交換繞過許可權提示。

---

## 12. Anthropic 內部專用命令（20+個）

全部在外部構建中被替換為 `{ isEnabled: false, isHidden: true }` 存根。

### 除錯和診斷

| 命令 | 用途 |
|------|------|
| `/bridge-kick` | 注入 bridge 故障（close/poll/register/heartbeat 等 10+ 子命令） |
| `/mock-limits` | 模擬各種速率限制場景 |
| `/reset-limits` | 重置速率限制狀態 |
| `/ctx-viz` | 上下文視窗視覺化 |
| `/debug-tool-call` | 工具呼叫除錯 |
| `/ant-trace` | Anthropic 內部追蹤 |
| `/perf-issue` | 效能問題報告 |
| `/env` | 顯示環境變數 |
| `/stuck` | 診斷凍結/慢會話（掃描 CPU、程序狀態、記憶體洩漏） |
| `/force-snip` | 強制歷史裁剪 |
| `/break-cache` | 使 prompt cache 失效 |

### 工作流和自動化

| 命令 | 用途 |
|------|------|
| `/autofix-pr` | 自動修復 PR 問題 |
| `/bughunter` | 遠端審查的 bug 獵人模式 |
| `/teleport` | 遠端傳送到其他環境 |
| `/backfill-sessions` | 回填會話資料 |
| `/agents-platform` | 代理平臺管理 |
| `/ultraplan` | 超級計劃模式 |
| `/share` | 分享對話 |

### 開發輔助

| 命令 | 用途 |
|------|------|
| `/lorem-ipsum` | 精確 token 計數的填充文字生成（用 1-token 詞彙實現精確計數） |
| `/oauth-refresh` | OAuth token 手動重新整理 |
| `/good-claude` | （已存根化） |
| `/onboarding` | 引導流程 |
| `/init-verifiers` | 初始化驗證器 |

### 內部專用工具

| 工具 | 用途 |
|------|------|
| `ConfigTool` | 直接修改配置 |
| `TungstenTool` | tmux 會話管理 |
| `REPLTool` | 沙盒 REPL VM |
| `SuggestBackgroundPRTool` | 建議後臺 PR |
| `CtxInspectTool` | 上下文檢查 |
| `OverflowTestTool` | 溢位測試 |
| `SnipTool` | 歷史裁剪 |
| `MonitorTool` | 監控 |

### Bridge Kick 詳細子命令

```
/bridge-kick close <code>           — 觸發 WebSocket close
/bridge-kick poll 404 [type]        — 下次輪詢 404
/bridge-kick poll transient         — 下次輪詢 5xx
/bridge-kick register fail [N]      — 下 N 次註冊失敗
/bridge-kick register fatal         — 註冊 403（終端失敗）
/bridge-kick reconnect-session fail — 重連失敗
/bridge-kick heartbeat <status>     — 心跳致命錯誤
/bridge-kick reconnect              — 直接觸發重連
/bridge-kick status                 — 列印 bridge 狀態
```

支援**組合故障序列**，復現真實生產故障鏈。

---

## 13. 隱藏鍵盤快捷鍵

### 全域性快捷鍵

| 快捷鍵 | 動作 |
|--------|------|
| `ctrl+t` | 切換 Todo 面板 |
| `ctrl+o` | 切換 Transcript 檢視器 |
| `ctrl+r` | 歷史反向搜尋 |
| `ctrl+l` | 重繪螢幕 |
| `ctrl+b` | 後臺執行當前任務 |
| `shift+tab` | 迴圈切換許可權模式 |
| `meta+p` | 模型選擇器 |
| `meta+o` | Fast Mode 切換 |
| `meta+t` | Thinking 模式切換 |

### Chord 快捷鍵（兩步組合）

| 快捷鍵 | 動作 |
|--------|------|
| `ctrl+x ctrl+k` | 終止所有代理 |
| `ctrl+x ctrl+e` 或 `ctrl+g` | 開啟外部編輯器 |

### 編輯快捷鍵

| 快捷鍵 | 動作 |
|--------|------|
| `ctrl+_` / `ctrl+shift+-` | 撤銷文字輸入 |
| `ctrl+s` | 暫存當前輸入 |
| `ctrl+v` / `alt+v`(Win) | 貼上圖片 |

### 特性門控快捷鍵

| 快捷鍵 | 動作 | 需要 |
|--------|------|------|
| `shift+up` | 訊息動作選擇器 | `MESSAGE_ACTIONS` |
| `ctrl+shift+f` / `cmd+shift+f` | 全域性檔案搜尋 | `QUICK_SEARCH` |
| `ctrl+shift+p` / `cmd+shift+p` | 快速開啟 | `QUICK_SEARCH` |
| `meta+j` | 終端面板 | `TERMINAL_PANEL` |
| `ctrl+shift+b` | Brief 模式切換 | `KAIROS` |
| `ctrl+shift+o` | 隊友預覽切換 | 團隊模式 |
| `space`（按住） | Push-to-talk 語音 | `VOICE_MODE` |

### 滾動快捷鍵

| 快捷鍵 | 動作 |
|--------|------|
| `pageup` / `pagedown` | 翻頁滾動 |
| `wheelup` / `wheeldown` | 滑鼠滾輪滾動 |
| `ctrl+shift+c` / `cmd+c` | 複製選中內容（滾動模式中） |

### 保留快捷鍵（不可重繫結）

- `ctrl+c` — 中斷
- `ctrl+d` — 退出

---

## 14. 隱藏環境變數

### 核心配置

| 變數 | 效果 |
|------|------|
| `CLAUDE_CODE_SIMPLE=1` | 極簡模式（3個工具） |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` | 禁用自動記憶提取 |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` | 禁用所有非必要網路流量 |
| `CLAUDE_CODE_DISABLE_THINKING=1` | 禁用 thinking 模式 |
| `CLAUDE_CODE_DISABLE_FAST_MODE=1` | 禁用 fast mode |
| `CLAUDE_CODE_DISABLE_CLAUDE_MDS=1` | 禁用 CLAUDE.md 載入 |
| `CLAUDE_CODE_DISABLE_COMMAND_INJECTION_CHECK=1` | 禁用命令注入檢查 |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1` | 禁用後臺任務 |

### 效能調優

| 變數 | 效果 |
|------|------|
| `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY=N` | 最大併發工具數（預設10） |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW=N` | 覆蓋自動壓縮視窗 |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS=N` | 覆蓋最大輸出 token |
| `CLAUDE_CODE_MAX_RETRIES=N` | 覆蓋最大重試次數 |
| `CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE=N` | 覆蓋阻塞限制 |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=80` | 在 80% 上下文時觸發壓縮 |

### API 控制

| 變數 | 效果 |
|------|------|
| `ANTHROPIC_MODEL=model-id` | 覆蓋預設模型 |
| `ANTHROPIC_SMALL_FAST_MODEL=model-id` | 覆蓋後臺輔助任務模型 |
| `ANTHROPIC_BASE_URL=url` | 自定義 API 基礎 URL |
| `ANTHROPIC_CUSTOM_HEADERS='{"k":"v"}'` | 注入自定義 HTTP 頭 |
| `CLAUDE_CODE_EXTRA_BODY='{"k":"v"}'` | 注入額外 API 請求體 |
| `CLAUDE_CODE_EXTRA_METADATA='{"k":"v"}'` | 注入額外後設資料 |
| `CLAUDE_CODE_USE_BEDROCK=1` | 使用 AWS Bedrock |
| `CLAUDE_CODE_USE_VERTEX=1` | 使用 Google Vertex |
| `CLAUDE_CODE_USE_FOUNDRY=1` | 使用 Azure Foundry |
| `CLAUDE_CODE_UNATTENDED_RETRY=1` | 無人值守無限重試 |

### 除錯

| 變數 | 效果 |
|------|------|
| `CLAUDE_CODE_DEBUG=1` | 啟用除錯日誌 |
| `CLAUDE_CODE_FRAME_TIMING_LOG=/path` | 記錄幀渲染時間 |
| `CLAUDE_CODE_BASH_SANDBOX_SHOW_INDICATOR=1` | 顯示沙盒指示器 |
| `CLAUDE_CODE_SYNTAX_HIGHLIGHT=1` | 啟用語法高亮 |
| `OTEL_LOG_TOOL_DETAILS=1` | 遙測中記錄工具詳情 |
| `OTEL_LOG_USER_PROMPTS=1` | 遙測中記錄使用者提示詞（預設 REDACTED） |
| `DISABLE_COMPACT=1` | 完全禁用壓縮 |
| `DISABLE_AUTO_COMPACT=1` | 僅禁用自動壓縮 |

### 代理團隊

| 變數 | 效果 |
|------|------|
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` | 啟用多代理團隊 |
| `CLAUDE_CODE_COORDINATOR_MODE=1` | 協調者模式 |
| `CLAUDE_CODE_TEAMMATE_COMMAND=cmd` | 隊友啟動命令 |
| `CLAUDE_CODE_AGENT_COLOR=color` | 代理顏色 |
| `CLAUDE_CODE_PLAN_MODE_REQUIRED=1` | 強制 plan 模式 |

---

## 15. 其他彩蛋與冷知識

### 200+ 花式載入詞

`spinnerVerbs.ts` 中的精選：

```
Clauding, Boondoggling, Canoodling, Flibbertigibbeting,
Hullaballooing, Moonwalking, Prestidigitating, Razzmatazzing,
Shenaniganing, Tomfoolering, Whatchamacalliting, Beboppin'
```

完成時的過去式：
```
Baked for 5s, Cogitated for 12s, Noodled for 8s
```

### VCR 錄製/回放模式

`FORCE_VCR=1`（內部）：錄製和回放 API 互動，用於測試。

### Lorem Ipsum 精確 token 生成

`/lorem-ipsum 50000`（內部）：使用經過驗證的 1-token 單詞生成精確 token 數的填充文字。

### Fennec 代號

內部模型遷移中有 `migrateFennecToOpus.ts`：將 `fennec-latest` 對映到 `opus`，`fennec-fast-latest` 對映到 `opus[1m]` + fast mode。

### 消融基線模式

`CLAUDE_CODE_ABLATION_BASELINE`：降級功能用於 A/B 測試，驗證各功能的實際貢獻。

### 深度連結協議

`feature('LODESTONE')`：註冊 `cc://` URL 協議處理器，其他應用可直接開啟 Claude Code 會話。

### 物種名編碼

```typescript
// 伴侶精靈的某個物種名與內部模型代號衝突
// 用 String.fromCharCode 十六進位制編碼避免 excluded-strings 檢查
```

### 許可權模式迴圈

`shift+tab` 在以下模式間迴圈：
```
default → acceptEdits → plan → auto → default → ...
```

### Anthropic API Key 自檢迴避

```typescript
// 秘密掃描器中 Anthropic API key 字首在執行時拼接
// 避免匹配自身原始碼的 excluded-strings 檢查
const prefix = ['sk', 'ant', 'api'].join('-')
```

---

## 總結：冰山模型

```
                    ╭───────────────╮
         公開版本   │  CLI + 43工具  │  ← 你看到的
                    │  /命令 + UI   │
                    ╰───────┬───────╯
                            │
            ╭───────────────┼───────────────╮
  編譯時剝離 │  KAIROS守護程序 │ Computer Use  │  ← 原始碼中存在
            │  Voice Mode   │ Speculation   │     但外部構建被刪除
            │  Dream Mode   │ Undercover    │
            │  Buddy Pet    │ Swarm/Teams   │
            │  Workflows    │ BG Sessions   │
            │  Deep Links   │ Cron Triggers │
            ╰───────────────┴───────────────╯
                            │
            ╭───────────────┼───────────────╮
  內部專用   │ 20+ 除錯命令   │ bridge-kick   │  ← 僅 Anthropic 員工
            │ mock-limits   │ VCR 錄放      │
            │ lorem-ipsum   │ 消融基線      │
            │ stuck 診斷    │ ctx-viz       │
            ╰───────────────┴───────────────╯
```

**Claude Code 不只是一個 CLI 工具——它的原始碼中隱藏著一個完整的 AI 助手作業系統的雛形。** 守護程序、語音輸入、電腦控制、投機執行、夢境記憶整合、多代理協作、隱身模式——當前公開版本只暴露了冰山一角。

---

*來自：AI超元域 | B站頻道：https://space.bilibili.com/3493277319825652*

*基於 Claude Code 原始碼逆向分析，2026-03-31*
