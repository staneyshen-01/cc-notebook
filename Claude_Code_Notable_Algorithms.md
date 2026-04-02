# Claude Code 值得借鑑的演算法與設計模式

> 來源：Claude Code 原始碼快照（2026-03-31）
> 上下文管理部分見另一份筆記 `Claude_Code_Context_Management_Notes.md`

---

## 目錄

1. [許可權系統：多層安全級聯](#1-許可權系統多層安全級聯)
2. [流式工具執行：模型推理與工具執行重疊](#2-流式工具執行模型推理與工具執行重疊)
3. [併發控制與重試策略](#3-併發控制與重試策略)
4. [終端渲染引擎：雙緩衝 + Diff 寫入](#4-終端渲染引擎雙緩衝--diff-寫入)
5. [純 TypeScript 原生模組移植](#5-純-typescript-原生模組移植)
6. [FileEditTool：12 步驗證鏈](#6-filedittool12-步驗證鏈)
7. [記憶系統：提取與召回](#7-記憶系統提取與召回)
8. [多 Agent 協調：Swarm 架構](#8-多-agent-協調swarm-架構)
9. [啟動最佳化：並行預取](#9-啟動最佳化並行預取)
10. [其他精巧設計](#10-其他精巧設計)

---

## 1. 許可權系統：多層安全級聯

### 1.1 Bash 命令許可權解析（8 步級聯）

檔案：`src/tools/BashTool/bashPermissions.ts`

```
Step 0: Tree-sitter AST 解析
  → simple（乾淨命令）/ too-complex（無法靜態分析）/ parse-unavailable（降級正則）
  + Shadow 模式：Tree-sitter 觀察性執行，記錄與 legacy 路徑的分歧

Step 1: 沙盒自動放行
  → sandboxing 啟用 + autoAllowBashIfSandboxed → 允許
  → 複合命令拆分，逐子命令檢查 deny 規則

Step 2: 精確匹配許可權檢查
  → 優先順序：deny > ask > allow > passthrough

Step 3: LLM 分類器 deny/ask 規則
  → Haiku 模型並行分類 deny 和 ask 描述列表
  → 僅高置信度結果觸發

Step 4: 命令運算子拆分
  → 對 |, &&, ||, ; 遞迴呼叫許可權檢查
  → 即使管道段被允許，原始命令仍重新驗證

Step 5: Legacy 誤解析門控
  → 僅當 Tree-sitter 不可用時執行

Step 6: 逐子命令檢查
  → splitCommand → 過濾 cd ${cwd} → checkCommandAndSuggestRules

Step 7: 8 步子級聯
  → 精確匹配 → 字首 deny → ask 規則 → 路徑約束
  → 精確 allow → 字首/萬用字元 allow → sed 約束
  → 模式許可權 → 只讀檢查 → passthrough
```

### 1.2 規則匹配中的安全技巧

**複合命令防護**：字首和萬用字元規則拒絕匹配複合命令。防止 `cd /path && python3 evil.py` 被 `cd:*` 規則放行。

**環境變數剝離的不對稱設計**：
- Deny 規則：使用激進的 `stripAllLeadingEnvVars`（固定點迴圈剝離所有環境變數），防止 `FOO=bar denied_cmd` 繞過
- Allow 規則：使用保守的 `stripSafeWrappers`，只接受白名單中的 ~60 個安全環境變數

**安全環境變數白名單**（SAFE_ENV_VARS）：
- 包含：`NODE_ENV`, `RUST_LOG`, `CGO_ENABLED` 等
- 排除：`PATH`, `LD_PRELOAD`, `LD_LIBRARY_PATH`, `DYLD_*`, `PYTHONPATH`, `NODE_OPTIONS`, `BASH_ENV`

### 1.3 Shell 安全分析（23 個驗證器）

檔案：`src/tools/BashTool/bashSecurity.ts`

按安全優先順序排序的驗證鏈：

| 驗證器 | 檢測內容 |
|--------|---------|
| `validateJqCommand` | `system()`, `-f`, `--from-file`, `-L` |
| `validateObfuscatedFlags` | 引號內隱藏的標誌（如 `"-rf"`） |
| `validateShellMetacharacters` | 引數中的 `;`, `\|`, `&` |
| `validateDangerousVariables` | `BASH_ENV`, `PROMPT_COMMAND`, `PS1`, `BASH_FUNC_*` |
| `validateCommentQuoteDesync` | 引號內的 `#`（註釋跟蹤混淆） |
| `validateCarriageReturn` | CR 字元（shell-quote/bash 分詞差異） |
| `validateIFSInjection` | `IFS=` 賦值 |
| `validateDangerousPatterns` | 反引號, `$()`, `${}`, zsh 擴充套件 (`=cmd`, `~[`, `(e:`) |
| `validateUnicodeWhitespace` | 非 ASCII 空白字元 |
| `validateBraceExpansion` | `{...,...}` 模式 |
| `validateZshDangerousCommands` | 20+ 個 zsh 內建命令（`zmodload`, `syswrite`, `ztcp` 等） |

**關鍵排序技巧**：非誤解析驗證器的 `ask` 結果被延遲返回。迴圈繼續執行誤解析驗證器；只有沒有誤解析驗證器觸發時，延遲的非誤解析結果才被返回。防止非誤解析的 `ask` 掩蓋應該設定 `isBashSecurityCheckForMisparsing` 的誤解析 `ask`。

### 1.4 只讀命令分類（雙層判定）

檔案：`src/tools/BashTool/readOnlyValidation.ts`

**Tier 1 — Flag 級別白名單**：每個命令的每個 flag 都有型別標註（`none`, `number`, `string`, 特定字面量）。特例：
- `xargs` 的 `-i`/`-e` 被移除（GNU `getopt_long` 可選引數語義漏洞）
- `tree` 的 `-R` 被移除（它會寫檔案）
- `fd`/`fdfind` 的 `-x`/`--exec` 被排除

**Tier 2 — 正則匹配**：`cat`, `head`, `tail`, `wc`, `jq`, `echo`, `pwd` 等簡單命令。

**複合命令安全**：`cd && git` 組合被阻止（沙盒逃逸 — 透過惡意 git hooks）。檢測寫入 `.git/hooks/`、`objects/`、`refs/` 後執行 git 的命令鏈。

### 1.5 Auto-Mode (YOLO) 分類器

檔案：`src/utils/permissions/yoloClassifier.ts`

**不是傳統 ML — 是 LLM 即分類器**。

**兩階段 XML 分類**：
```
Stage 1（快速判定）：
  max_tokens=64, stop_sequences=['</block>']
  → "no"（允許）：立即返回
  → "yes" 或無法解析：升級到 Stage 2

Stage 2（深度推理）：
  max_tokens=4096, 啟用 chain-of-thought
  → 用 <thinking> 標籤推理後再決定
  → 解析時先剝離 <thinking> 塊再匹配 <block> 標籤
```

**200ms 競賽模式**（interactiveHandler.ts）：
```
5 個參賽者同時啟動：
  1. 使用者許可權對話方塊
  2. Hooks 非同步執行
  3. Bash 分類器非同步執行
  4. Bridge 許可權響應（claude.ai）
  5. Channel 許可權中繼（Telegram 等）

createResolveOnce 原子 claim() — 第一個到達的贏，其他 no-op

200ms 寬限期：
  前 200ms 內忽略使用者互動（防止意外按鍵取消分類器）
  200ms 後任何使用者互動都會殺死分類器的自動批准機會
```

**投機性分類器檢查**：在許可權對話方塊出現之前就啟動分類器（`startSpeculativeClassifierCheck`），與 deny/ask 分類器、hooks、對話方塊設定並行執行。

---

## 2. 流式工具執行：模型推理與工具執行重疊

檔案：`src/services/tools/StreamingToolExecutor.ts`

### 核心思想

工具在模型**還在生成 token 時就開始執行**，而非等完整響應結束。

### 併發控制演算法

```typescript
canExecuteTool(isConcurrencySafe: boolean): boolean {
  const executing = this.tools.filter(t => t.status === 'executing')
  return (
    executing.length === 0 ||
    (isConcurrencySafe && executing.every(t => t.isConcurrencySafe))
  )
}
```

- 沒有正在執行的工具 → 可以執行
- 自身併發安全 且 所有正在執行的也併發安全 → 可以執行
- 否則 → 等待

### 工具狀態機

```
queued → executing → completed → yielded
```

佇列有序處理。非併發工具遇到無法執行時直接 break（保持嚴格順序）；併發工具可跳過。

### 錯誤級聯

**只有 Bash 錯誤才取消兄弟工具**。原因：Bash 命令有隱式依賴鏈（mkdir 失敗 → 後續命令無意義）。Read/WebFetch 等獨立工具不取消。

`siblingAbortController` 是 parent 的子控制器 — 兄弟子程序立即死亡，但父 query loop 不中斷。

### 進度流式傳輸

使用 Promise 訊號機制（`progressAvailableResolve`）。`getRemainingResults()` async generator 用 `Promise.race` 在工具完成和新進度到達之間等待，確保進度訊息實時傳遞而非緩衝。

### 與 query.ts 的整合

```
流式迴圈中：
  API content_block_stop 事件
    → 立即 yield AssistantMessage
    → query.ts 餵給 StreamingToolExecutor.addTool()
  
  事件間隙：
    → getCompletedResults() 收割已完成的工具
  
  流式結束後：
    → getRemainingResults() 排空所有待處理工具
```

---

## 3. 併發控制與重試策略

### 3.1 併發 Generator 合併（all 函式）

檔案：`src/utils/generators.ts`

```
維護 waiting 佇列 + active Promise 集合
1. 初始填充 concurrencyCap 個 generator
2. Promise.race 所有活躍 generator
3. 某個 yield → 立即 .next() 推進它
4. 某個完成 → waiting 中取一個填補
5. 公平交錯 + 有界並行（預設上限 10）
```

### 3.2 工具併發安全分類

| 分類 | 工具 | 依據 |
|------|------|------|
| 始終安全 | FileRead, LSP, TaskCreate, TaskGet, Brief | 返回 `true` |
| 輸入依賴 | BashTool | 委託給 `isReadOnly(input)`，只讀命令併發，寫命令獨佔 |
| 預設不安全 | FileEdit, FileWrite 等 | 使用預設 `false`（fail-closed） |

### 3.3 指數退避 + 抖動重試

檔案：`src/services/api/withRetry.ts`

```
baseDelay = min(500ms * 2^(attempt-1), 32s)
jitter = random(0, 25% * baseDelay)       // 加性抖動，非乘性
finalDelay = baseDelay + jitter
// server retry-after header 存在時覆蓋
```

**529（過載）處理 — 級聯放大防護**：
```
非前臺查詢（摘要、標題、分類器）→ 收到 529 立即放棄（不重試）
前臺查詢 → 重試最多 3 次
3 次 529 後 → FallbackTriggeredError → 切換備用模型
```

**Fast Mode 降級**：
```
429/529 + fast mode 啟用：
  retry-after < 20s → 繼續 fast mode 重試（保持 prompt cache）
  retry-after ≥ 20s → 冷卻（最少 10min）切換標準速度
```

**無人值守模式**（`CLAUDE_CODE_UNATTENDED_RETRY`）：
```
429/529 無限重試，最大退避 5 分鐘
長休眠拆為 30s 心跳間隔（防止主機標記會話空閒）
6 小時硬上限重置
```

### 3.4 流式看門狗

檔案：`src/services/api/claude.ts`

```
空閒超時預設 90s（可配置）
50% 時間 → 警告
100% 時間 → 硬中止掛起的流
每個 chunk 重置計時器
```

---

## 4. 終端渲染引擎：雙緩衝 + Diff 寫入

### 4.1 雙緩衝幀交換

檔案：`src/ink/ink.tsx`, `src/ink/output.ts`, `src/ink/screen.ts`

```
每次渲染後：
  backFrame = frontFrame
  frontFrame = newFrame
```

**Screen 緩衝區**：packed `Int32Array`，每個單元格 2 個 word（8 bytes）。
- Word 0: char pool ID
- Word 1: style ID + hyperlink ID + cell width（位打包）
- `resetScreen()` 複用同一 buffer（只增不縮），用 `BigInt64Array` 檢視批次清零

**charCache**（16384 上限）：跨幀快取 grapheme 聚類結果，大多數行不變時直接命中。

### 4.2 Diff 演算法

檔案：`src/ink/screen.ts` (diffEach), `src/ink/log-update.ts`

```
1. 計算兩幀 damage rectangle 的並集
2. 僅在 damage 區域內逐 word 比較 Int32Array
3. findNextDiff() 是純函式，設計為 JIT 內聯
4. VirtualScreen 跟蹤游標位置，只在目標不一致時發移動指令
```

**關鍵最佳化**：
- **DECSTBM 硬體滾動**：ScrollBox 的 scrollTop 變化時用終端硬體滾動（`CSI top;bot r + CSI n S/T`），而非重寫整個區域。先對 prev.screen 執行 `shiftRows()` 模擬硬體位移，後續 diff 自然只找到新滾入的行。
- **StylePool.transition()**：按 (fromId, toId) 對快取 ANSI 樣式轉換字串 — 預熱後零分配
- **fg-only 空格跳過**：只有前景色的空格單元格視為不可見，跳過寫入

### 4.3 Blit 最佳化

檔案：`src/ink/render-node-to-output.ts`

```
節點乾淨（not dirty）且佈局位置不變
  → 直接從 prevScreen 複製單元格（blit）
  → blitRegion() 使用 TypedArray.set() 批次記憶體複製
  → 每行一次呼叫，連續全寬區域只需一次
  → 跳過整個子樹的重新渲染
```

### 4.4 渲染器 Peephole 最佳化

檔案：`src/ink/optimizer.ts`

單趟掃描 Diff 陣列：
- 合併連續 `cursorMove`（加 dx/dy）
- 摺疊連續 `cursorTo`（只保留最後一個）
- 拼接相鄰 `styleStr`
- 取消 cursor hide/show 對
- 去重相同 URI 的 hyperlink patch
- 移除 count=0 的 clear patch

---

## 5. 純 TypeScript 原生模組移植

### 5.1 Yoga 佈局引擎（C++ → TypeScript）

檔案：`src/native-ts/yoga-layout/index.ts`（~2400 行）

完整的 Flexbox 佈局實現，消除了 native binary 依賴。

**多層快取策略**：
| 快取 | 機制 | 效果 |
|------|------|------|
| Dirty-flag | 乾淨子樹 + 匹配輸入 → 跳過 | 最基本的剪枝 |
| 雙槽快取 | 分別快取 layout 和 measure 結果 | 同一節點兩種呼叫模式 |
| 4 槽環形快取 | packed Float64Array | 500 訊息 scrollbox: 76k→4k layoutNode 呼叫 |
| flex-basis 快取 | generation-stamped | 短路遞迴 computeFlexBasis |
| 快速路徑標誌 | `_hasAutoMargin` 等 | 全零情況單分支跳過 |

**`resolveEdges4Into()`**：一次遍歷解析全部 4 條物理邊到預分配元組，提升共享 fallback 查詢。

### 5.2 模糊搜尋（Rust nucleo/fzf → TypeScript）

檔案：`src/native-ts/file-index/index.ts`

**逐步過濾架構**：
```
Step 1: 字元點陣圖過濾（O(1) 拒絕）
  → 每個路徑一個 26-bit charBits 掩碼
  → (charBits & needleBitmap) !== needleBitmap → 跳過

Step 2: 融合 indexOf 掃描
  → String.indexOf()（V8/JSC 中 SIMD 加速）
  → 同時找到匹配位置 + 累積 gap/consecutive 分數
  → 無需第二次評分遍歷

Step 3: Gap-bound 拒絕
  → 計算分數上限（所有邊界獎勵）減去已知 gap 懲罰
  → 無法超過當前 top-k 閾值 → 跳過昂貴的邊界評分

Step 4: 邊界/駝峰評分
  → 路徑分隔符 (/\-_.) 匹配獎勵
  → 駝峰轉換匹配獎勵
  → 首字元匹配獎勵
  → 常數近似 nucleo/fzf-v2 權重

Top-k 維護：升序陣列 + 二分插入（避免全量 O(n log n) 排序）
```

**其他特性**：
- 非同步構建：每 ~4ms yield 事件迴圈，`readyCount` 支援構建中的部分索引搜尋
- 智慧大小寫：全小寫查詢 = 大小寫不敏感；有大寫 = 敏感
- 測試檔案懲罰：路徑包含 "test" → 1.05x 分數懲罰

### 5.3 語法高亮 + Word-Level Diff（Rust syntect/bat → TypeScript）

檔案：`src/native-ts/color-diff/index.ts`

- highlight.js 延遲載入（避免 ~200ms 的 190+ 語法註冊啟動成本）
- `diff` npm 包的 `diffArrays` 做詞級 diff
- RGB → ANSI-256 顏色近似：移植 `ansi_colours` Rust crate 的立方體 vs 灰階感知最近索引演算法
- Monokai Extended / GitHub-light 作用域到顏色對映
- Storage 關鍵字重分割（highlight.js 把 `const`/`function`/`class` 歸為 "keyword"；埠重分割以匹配 syntect 的 cyan storage 顏色）

---

## 6. FileEditTool：12 步驗證鏈

檔案：`src/tools/FileEditTool/FileEditTool.ts`

```
 1. 金鑰檢測 → 阻止向 team memory 檔案寫入金鑰
 2. 空操作檢測 → old_string === new_string 直接拒絕
 3. Deny 規則檢查 → 檔案路徑匹配 deny 許可權規則
 4. UNC 路徑安全 → 跳過 \\server\share（防止 NTLM 憑證洩露）
 5. 檔案大小守衛 → > 1 GiB 拒絕
 6. 編碼檢測 → UTF-16LE BOM (0xFF 0xFE) / UTF-8, \r\n → \n
 7. 檔案存在檢查 → 不存在時建議相似檔案（findSimilarFile）
 8. 空 old_string → 僅在檔案為空時允許（建立檔案場景）
 9. Notebook 重定向 → .ipynb 必須使用 NotebookEditTool
10. 陳舊性檢測 → mtime 比較，失敗時回退到內容比較（避免雲同步/防毒軟體時間戳干擾的誤報）
11. 引號規範化 → 彎引號→直引號搜尋；寫入時用啟發式恢復彎引號樣式
12. 歧義匹配 → 多處匹配 + 非 replace_all → 拒絕並要求更多上下文
```

### 引號規範化演算法（utils.ts）

```
搜尋階段：
  1. 精確匹配 old_string → 找到則使用
  2. normalizeQuotes(old_string) → 彎引號轉直引號
  3. 在 normalizeQuotes(fileContent) 中搜尋
  4. 返回 fileContent 中的原始子串（保留彎引號）

寫入階段（preserveQuoteStyle）：
  檢測到規範化被應用時：
  → 將 new_string 中的直引號轉回彎引號
  → 啟發式：空白/行首/開括號後 = 開引號；字母間 = 撇號
```

### 反序列化對映（desanitizeMatchString）

模型不會看到某些 XML 標籤（傳送給 API 前被清理）。當模型在編輯中輸出清理後的形式時，反向對映：
- `<fnr>` → `<function_results>`
- `\n\nH:` → `\n\nHuman:`

### call() 寫入路徑中的雙重陳舊性檢查

```
validateInput 時檢查一次陳舊性
  → 透過
    → call() 中重新同步讀取檔案，再次檢查
    → 防止 validate 和 call 之間的 TOCTOU 競態
```

---

## 7. 記憶系統：提取與召回

### 7.1 記憶提取（extractMemories）

檔案：`src/services/extractMemories/`

**架構**：每個完整 query loop 結束時，fork 一個子 agent（共享父級 prompt cache）執行提取。

```
提取流程：
1. 門控：僅主 agent，非子 agent，非遠端模式
2. 重疊守衛：已有提取在執行 → 暫存當前上下文（最新覆蓋舊的）
3. 輪次節流：合格輪次未達閾值 → 跳過
4. 互斥：主 agent 已手動寫入記憶 → 跳過並推進遊標
5. 注入記憶清單：掃描目錄 + 讀 frontmatter → 預格式化
6. Fork agent 執行：最多 5 輪，受限工具訪問
7. 遊標推進：僅在成功後；失敗時留在原位以重新考慮
8. 尾隨執行：完成後檢查暫存的待處理上下文
```

**工具限制**：Read/Grep/Glob 無限制；只讀 Bash；Edit/Write 僅限記憶目錄內。

### 7.2 記憶召回（findRelevantMemories）

檔案：`src/memdir/findRelevantMemories.ts`

**不是啟發式評分 — 是 LLM 評分**：

```
Phase 1: 掃描
  → 讀取記憶目錄所有 .md 檔案（排除 MEMORY.md）
  → 每個檔案讀前 30 行提取 frontmatter（name, description, type）
  → 按 mtime 降序排列，上限 200 個檔案
  → 單遍設計：讀取後排序（而非 stat-排序-讀取），syscall 減半

Phase 2: LLM 選擇
  → 傳送查詢 + 格式化清單 + 最近使用工具列表給 Sonnet
  → 結構化 JSON 輸出
  → "只包含你確定有幫助的記憶。不確定就不包含。最多 5 個。"
  → 最近使用工具列表防止為已活躍使用的工具推薦 API 文件
  → 但關於這些工具的警告/陷阱仍然會被選中

Phase 3: 新鮮度處理
  → 超過 1 天的記憶注入 <system-reminder> 告警
  → "此記憶已 N 天。關於程式碼行為的宣告可能過時。"
```

**already-surfaced 過濾**：過濾掉之前輪次已展示的路徑，5 個名額全部花在新候選上。

---

## 8. 多 Agent 協調：Swarm 架構

### 8.1 協調者模式（Coordinator）

檔案：`src/coordinator/coordinatorMode.ts`

```
任務工作流階段：
  Research（並行 workers）→ Synthesis（協調者）→ Implementation（workers）→ Verification（workers）

併發規則：
  只讀任務 → 自由並行
  寫密集任務 → 按檔案區域序列
  驗證可與不同檔案的實現重疊

Worker prompt 必須自包含：
  Worker 看不到協調者對話 → 每個 prompt 需要完整上下文（檔案路徑、行號等）

Continue vs Spawn 決策：
  高上下文重疊 → continue（複用已載入上下文）
  低上下文重疊 → spawn 新 worker
```

### 8.2 兩種後端策略

| 後端 | 隔離方式 | 通訊 | 特點 |
|------|---------|------|------|
| In-Process | `AsyncLocalStorage` 上下文隔離 | 基於檔案的 mailbox | 共享 API client + MCP 連線；獨立 AbortController（leader 中斷不殺 worker） |
| Pane-Based (tmux/iTerm2) | 獨立 OS 程序 | 基於檔案的 mailbox | CLI flag 傳播（`--agent-id`, `--agent-name`, `--team-name`, `--agent-color`）；leader 的模型/許可權/環境變數全部傳播 |

### 8.3 許可權橋接

In-process teammate 透過 `leaderPermissionBridge` 路由許可權提示到 leader 的 UI（複用 BashPermissionRequest、FileEditToolDiff 等對話方塊）。Bridge 不可用時回退到 mailbox 許可權同步。

### 8.4 Fork Subagent

檔案：`src/tools/AgentTool/forkSubagent.ts`

```
子程序繼承父級完整對話上下文 + system prompt

遞迴 fork 防護：
  isInForkChild() 檢查對話歷史中的 <fork_boilerplate> 標籤

Cache 共享設計：
  保留完整父級 assistant 訊息（所有 tool_use 塊）
  構建 tool_result 塊（佔位文字："Fork started -- processing in background"）
  只有最後的 text 塊不同 → 最大化 prompt cache 命中

子程序 10 條嚴格規則：
  不生成子 agent、不評論、只用工具、提交變更、
  結構化輸出（Scope/Result/Key files/Files changed/Issues）、最多 500 字
```

---

## 9. 啟動最佳化：並行預取

檔案：`src/main.tsx` 前 20 行

```typescript
// 這些副作用必須在所有其他 import 之前執行：
profileCheckpoint('main_tsx_entry')     // 標記入口（在 ~135ms import 之前）
startMdmRawRead()                       // 並行：MDM 子程序讀取（plutil/reg query）
startKeychainPrefetch()                 // 並行：macOS 鑰匙串雙讀取
                                        // 無此最佳化：~65ms 同步阻塞（每次 macOS 啟動）
```

**啟動分析器**（`startupProfiler.ts`）：
- 取樣日誌：內部 100%，外部 0.5%。記錄 import_time, init_time, settings_time, total_time
- 詳細分析：`CLAUDE_CODE_PROFILE_STARTUP=1`，帶 `process.memoryUsage()` 快照的完整時間線
- 非取樣使用者零開銷（`profileCheckpoint` 立即返回）

---

## 10. 其他精巧設計

### 10.1 35 行狀態管理

檔案：`src/state/store.ts`

```typescript
createStore<T>(initialState, onChange?) => {
  getState()          // 返回當前狀態
  setState(updater)   // updater: (prev) => next
                      // Object.is() 相等檢查（引用相同則跳過）
                      // 觸發 onChange + 通知 Set<Listener>
  subscribe(listener) // 返回 unsubscribe
}
```

無中介軟體，無選擇器，無 devtools。配合 `useSyncExternalStore` 實現最小化重渲染。

### 10.2 ToolSearchTool — 延遲工具發現

檔案：`src/tools/ToolSearchTool/ToolSearchTool.ts`

**兩種查詢模式**：

直接選擇（`select:ToolA,ToolB`）：精確查詢，返回 `tool_reference` 塊。

關鍵詞搜尋評分：
| 匹配型別 | 分數 |
|---------|------|
| 名稱部分精確匹配 | +10（MCP: +12） |
| 名稱部分子串匹配 | +5（MCP: +6） |
| 全名回退 | +3 |
| searchHint 詞邊界匹配 | +4 |
| 描述詞邊界匹配 | +2 |

`+` 字首標記必需詞（全部必須匹配才入圍）。

工具名解析：MCP 工具去 `mcp__` 字首後按 `__` 和 `_` 分割；普通工具按駝峰轉換和 `_` 分割。

### 10.3 Token 估算（無 API 呼叫）

檔案：`src/utils/tokens.ts`

```
粗略估算：content.length / bytesPerToken

檔案型別感知：
  JSON/JSONL/JSONC: 2 bytes/token（密集單字元 token）
  其他: 4 bytes/token

區塊級估算：
  text/thinking: length / 4
  image/document: 固定 2000 tokens
  tool_use: (name + JSON.stringify(input)).length / 4

上下文視窗估算（tokenCountWithEstimation）：
  1. 從後向前找到最後一條有 API usage 資料的訊息
  2. 處理並行工具呼叫：跨越共享 message.id 的兄弟記錄
  3. 返回 usage.input_tokens + 粗略估算(後續訊息)
  4. 無 usage 資料時全量粗略估算
```

### 10.4 錯誤扣留模式（Error Withholding）

```
流式傳輸中可恢復錯誤不暴露給呼叫者：
  → prompt_too_long, media_size, max_output_tokens
  → 推入 assistantMessages 供恢復檢查
  → 所有恢復失敗後才 yield 給使用者
  → 防止 SDK 消費者在中間錯誤時終止會話
```

### 10.5 Prompt Cache 穩定性設計集錦

| 技術 | 位置 | 效果 |
|------|------|------|
| 工具池排序 | `assembleToolPool()` | 防止 MCP 變更破壞快取字首 |
| 系統 prompt 分界標記 | `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` | 靜態部分 `scope: global` 跨組織快取 |
| Agent 列表附件注入 | AgentTool prompt.ts | 從工具描述中移出動態列表（減少 10.2% cache_creation） |
| Fork 訊息構造 | forkSubagent.ts | 所有子程序共享相同 tool_result 佔位符，僅最後文字塊不同 |
| Tool result 替換一致性 | toolResultStorage.ts | mustReapply 重用完全相同的替換字串（位元組級一致） |
| frozen 分割槽 | toolResultStorage.ts | 進入 cache 的內容永不修改 |

### 10.6 文字選擇演算法

檔案：`src/ink/selection.ts`

- anchor + focus 點用螢幕緩衝區座標（col, row）
- 雙擊/三擊選詞/行模式：`anchorSpan` 啟用拖拽時按詞/行擴充套件
- 滾動捕獲：`scrolledOffAbove`/`scrolledOffBelow` 累加器捕獲拖拽滾動時離開視口的行
- 詞邊界檢測匹配 iTerm2 預設行為（路徑字元 `/-+\~_.` 視為詞字元）
- `getSelectedText()` 合併離屏和在屏行，尊重軟換行標記重建邏輯行

### 10.7 滑鼠 Hit Testing

檔案：`src/ink/hit-test.ts`

遞迴深度優先遍歷 DOM 樹。**子節點逆序遍歷**（後繪製的在上層），確保正確 z-order。
- `dispatchClick()`：從最深命中節點沿 parentNode 冒泡
- `dispatchHover()`：類 DOM mouseEnter/mouseLeave（非冒泡），diff hovered-node 集合

### 10.8 GrepTool 分頁

```
預設限制：250 條（未指定時）
head_limit=0：無限制（謹慎使用）
offset 引數：跳過前 N 條
分頁在 ripgrep 返回後、路徑相對化之前應用（節省 CPU）
appliedLimit 僅在實際截斷時報告（讓模型知道有更多結果）
```

### 10.9 Partial Compact 方向性

```
from（預設）：
  摘要 pivot 之後的訊息，保留之前的
  → prompt cache 保留（保留的訊息在前）

up_to：
  摘要 pivot 之前的訊息，保留之後的
  → prompt cache 失效（摘要在保留訊息之前）
  → 剝離舊 compact 邊界和摘要（防止陳舊邊界混淆掃描器）
```

### 10.10 API 訊息輪次分組

檔案：`src/services/compact/grouping.ts`

```
groupMessagesByApiRound：
  按 API 輪次邊界分組（不同 message.id 標記新輪次）
  比之前的 human-turn 分組更細粒度
  → 支援單 human turn 的 SDK/CCR/eval 會話中的精確 compact
  流式 chunk 共享 id → 同一響應內的交錯 tool_result 保持正確分組
```
