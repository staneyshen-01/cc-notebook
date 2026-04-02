# Claude Code 自動回滾機制解析與版本切換指南

> 日期：2026-03-31

## 背景

Claude Code 從 v2.1.88 自動回滾到 v2.1.87，本文記錄了完整的排查過程和解決方案。

## 回滾原因

**Anthropic 在 npm 做了服務端回滾**，不是本地觸發的。

### npm Registry 狀態

| dist-tag | 版本 |
|----------|------|
| `stable` | 2.1.81 |
| `latest` | 2.1.87 |
| `next` | 2.1.89 |

- v2.1.88 **已被完全從 npm 刪除**（registry 中不存在）
- `latest` tag 從 2.1.88 退回到 2.1.87

### 時間線

| 時間 | 事件 |
|------|------|
| Mar 28 19:25 | 自動更新下載 v2.1.87 |
| Mar 30 18:31 | 自動更新下載 v2.1.88 |
| Mar 30 ~ Mar 31 | Anthropic 從 npm 撤掉 v2.1.88，將 `latest` tag 退回 v2.1.87 |
| Mar 31 05:59 | 自動更新器檢查 npm registry，發現 `latest` 是 2.1.87，將 symlink 切回 |

## 自動更新機制

### 核心架構

```
~/.local/bin/claude  →  symlink  →  ~/.local/share/claude/versions/{version}
```

- 包名：`@anthropic-ai/claude-code`
- 版本二進位制儲存：`~/.local/share/claude/versions/`
- 入口 symlink：`~/.local/bin/claude`

### AutoUpdater 工作流程

1. 每次 Claude Code 啟動時，AutoUpdater 檢查 npm registry 的 `latest` dist-tag
2. 如果本地版本與 `latest` 不匹配，下載目標版本二進位制到 `versions/` 目錄
3. 更新 symlink 指向新版本
4. 邏輯是**跟隨 `latest` tag**，不是單調遞增 — 所以 Anthropic 退 tag 就等於回滾

### 關鍵發現

- 二進位制是 Bun 編譯的 Mach-O arm64 可執行檔案
- 內部包含 `auto_updater_disabled`、`AutoUpdater`、`autoUpdaterStatus` 等標識
- 啟動遙測會上報 `auto_updater_disabled` 狀態
- 併發更新有互斥鎖保護（"Another instance is currently performing an update"）

## 禁用自動更新的正確方式

透過逆向二進位制中的 `h1H()` / `isAutoUpdaterDisabled` 函式，確認自動更新器的檢查邏輯：

```javascript
// 反編譯後的禁用檢查邏輯（簡化）
function getAutoUpdaterDisabledReason() {
  if (process.env.DISABLE_AUTOUPDATER) return { type: "env" };
  if (config.autoUpdates === false)     return { type: "config" };
  return null; // 未禁用，自動更新正常執行
}
```

### 踩坑記錄

`autoUpdaterDisabled: true` 是**錯誤的 key**，寫了不生效，自動更新器仍會在啟動時搶先將 symlink 切回 `latest` 指向的版本。

### 方法 A：環境變數（推薦，最可靠）

```bash
# 加到 ~/.zshrc，每次 shell 啟動自動生效
echo 'export DISABLE_AUTOUPDATER=1' >> ~/.zshrc
source ~/.zshrc
```

### 方法 B：settings.json

編輯 `~/.claude/settings.json`，注意 key 是 `autoUpdates`：

```json
"autoUpdates": false
```

> 注意：對於 native 安裝方式，如果同時設定了 `autoUpdatesProtectedForNative: true`，則 `autoUpdates: false` 會被覆蓋，此時只能用環境變數方式。

## 解決方案：切回 v2.1.88

### 步驟 1：禁用自動更新

```bash
# 確保環境變數已生效
export DISABLE_AUTOUPDATER=1
```

### 步驟 2：切換 symlink

```bash
ln -sf ~/.local/share/claude/versions/2.1.88 ~/.local/bin/claude
```

### 步驟 3：驗證

```bash
claude --version
# 輸出：2.1.88 (Claude Code)
```

### 步驟 4：使用

```bash
claude --dangerously-skip-permissions
```

## 恢復自動更新

```bash
# 1. 從 ~/.zshrc 刪掉 export DISABLE_AUTOUPDATER=1
# 2. 如果用了方法 B，從 settings.json 刪掉 "autoUpdates": false
# 3. 重啟終端，自動更新器會在下次啟動時恢復工作
```

## 其他版本切換方式

```bash
# 切到 next channel (v2.1.89)
claude update --channel next

# 切到 stable channel (v2.1.81)
claude update --channel stable

# 檢視本地已有的版本
ls ~/.local/share/claude/versions/

# 手動切換到任意本地版本
ln -sf ~/.local/share/claude/versions/<版本號> ~/.local/bin/claude
```

## 注意事項

- v2.1.88 被 Anthropic 從 npm 刪除，可能存在已知問題
- 禁用自動更新後不會收到安全修復，需定期手動檢查
- 本地殘留的 v2.1.88 二進位制不會被自動清理
