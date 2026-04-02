# macOS 上清除 Claude Code 追蹤資料指南

> 基於 Claude Code 原始碼（2026-03-31 洩露快照，512K 行 TypeScript）的逆向分析。
> 所有檔案路徑和欄位名均從原始碼中確認。

---

## 背景：Claude Code 追蹤了什麼？

Claude Code **不收集硬體指紋**（無 MAC 地址、CPU 型號、記憶體大小、GPU 資訊），但透過以下機制實現使用者追蹤：

| 追蹤標識 | 永續性 | 儲存位置 | 說明 |
|----------|--------|----------|------|
| `userID` | 永久（直到手動刪除） | `~/.claude.json` | 隨機生成的 64 位十六進位制字串，跨會話追蹤主鍵 |
| `anonymousId` | 永久 | `~/.claude.json` | 格式 `claudecode.v1.<uuid>`，備用追蹤 ID |
| `accountUuid` | 永久（繫結賬號） | `~/.claude.json` → `oauthAccount` | OAuth 登入後直接關聯身份 |
| `emailAddress` | 永久 | `~/.claude.json` → `oauthAccount` | 登入郵箱 |
| `rh` (倉庫雜湊) | 按倉庫 | 每次 API 請求 Header | git remote URL 的 SHA256 前 16 位 |
| Statsig Stable ID | 永久 | `~/.claude/statsig/` | 特性開關係統的裝置標識 |

**資料流向：**
- **Anthropic 1P** → `/api/event_logging/batch`（完整環境資料 + Auth）
- **Datadog** → `https://http-intake.logs.us5.datadoghq.com`（白名單事件，已脫敏）
- **OTLP**（可選）→ 使用者自配的端點（預設關閉）

---

## 第一級：重置裝置標識（最重要）

這是跨會話追蹤的主鍵。清除後你對 Anthropic 來說就是一臺"全新裝置"。

```bash
# 檢視當前的追蹤 ID
grep -E '"userID"|"anonymousId"|"firstStartTime"|"claudeCodeFirstTokenDate"' ~/.claude.json
```

```bash
# 刪除追蹤標識（保留其他配置不變）
python3 -c "
import json, os
p = os.path.expanduser('~/.claude.json')
with open(p, 'r') as f: d = json.load(f)
removed = []
for k in ['userID', 'anonymousId', 'firstStartTime', 'claudeCodeFirstTokenDate']:
    if k in d:
        removed.append(k)
        del d[k]
with open(p, 'w') as f: json.dump(d, f, indent=2)
print(f'已刪除: {removed}')
print('下次啟動 Claude Code 會自動生成新的 userID')
"
```

**效果：** 下次啟動時會生成全新的 `userID`，之前的使用記錄無法與你關聯。

---

## 第二級：清除遙測和分析資料

```bash
# 未成功上報的分析事件（包含完整的環境資訊、會話資料）
rm -rf ~/.claude/telemetry/

# Statsig/GrowthBook 特性開關快取（包含 stable_id 裝置標識）
rm -rf ~/.claude/statsig/

# 統計快取
rm -f ~/.claude/stats-cache.json
```

**效果：** 清除本地快取的遙測資料和特性開關係統的裝置標識。

---

## 第三級：清除會話和歷史記錄

```bash
# 完整命令歷史（你輸入過的所有提示詞）
rm -f ~/.claude/history.jsonl

# 會話快照
rm -rf ~/.claude/sessions/

# 大段貼上內容的雜湊快取
rm -rf ~/.claude/paste-cache/

# Shell 環境快照
rm -rf ~/.claude/shell-snapshots/

# 會話環境變數
rm -rf ~/.claude/session-env/

# 檔案編輯歷史（Claude 做過的每次檔案修改）
rm -rf ~/.claude/file-history/

# 除錯日誌
rm -rf ~/.claude/debug/
```

**效果：** 清除所有本地會話痕跡。不影響 Claude Code 正常使用。

---

## 第四級：清除 OAuth 賬號關聯

```bash
# 檢視當前關聯的賬號資訊
python3 -c "
import json, os
with open(os.path.expanduser('~/.claude.json')) as f: d = json.load(f)
oa = d.get('oauthAccount', {})
print(f\"賬號 UUID: {oa.get('accountUuid', '無')}\")
print(f\"郵箱: {oa.get('emailAddress', '無')}\")
"
```

```bash
# 從 macOS Keychain 刪除 OAuth Token
security delete-generic-password -s "claude-code" 2>/dev/null
security delete-generic-password -s "claude-code-credentials" 2>/dev/null

# 清除配置檔案中的賬號快取
python3 -c "
import json, os
p = os.path.expanduser('~/.claude.json')
with open(p, 'r') as f: d = json.load(f)
removed = []
for k in ['oauthAccount', 's1mAccessCache', 'groveConfigCache',
          'passesEligibilityCache', 'clientDataCache',
          'cachedExtraUsageDisabledReason', 'githubRepoPaths']:
    if k in d:
        removed.append(k)
        del d[k]
with open(p, 'w') as f: json.dump(d, f, indent=2)
print(f'已刪除: {removed}')
"
```

**效果：** 斷開與 Claude.ai 賬號的本地關聯。下次使用需要重新登入。

---

## 第五級：完全重置（核彈選項）

```bash
# 1. 備份你的自定義配置
mkdir -p ~/Desktop/claude-backup
cp ~/.claude/CLAUDE.md ~/Desktop/claude-backup/ 2>/dev/null
cp ~/.claude/settings.json ~/Desktop/claude-backup/ 2>/dev/null
cp -r ~/.claude/skills ~/Desktop/claude-backup/ 2>/dev/null
cp -r ~/.claude/hooks ~/Desktop/claude-backup/ 2>/dev/null
echo "已備份到 ~/Desktop/claude-backup/"

# 2. 刪除所有 Claude Code 資料
rm -rf ~/.claude/
rm -f ~/.claude.json

# 3. 清除 Keychain 中的 Token
security delete-generic-password -s "claude-code" 2>/dev/null
security delete-generic-password -s "claude-code-credentials" 2>/dev/null

echo "已完全重置。下次啟動 Claude Code 會重新初始化。"
```

**效果：** 等同於全新安裝。所有配置、歷史、記憶、技能、外掛設定全部清除。

---

## 防止未來追蹤

### 方法 1：禁用非必要網路流量

在 `~/.claude/settings.json` 中新增：

```json
{
  "env": {
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1"
  }
}
```

**效果：** 禁用分析上報、GrowthBook 特性開關拉取、配額預檢等非必要請求。

### 方法 2：使用第三方雲後端

原始碼確認：使用 Bedrock 或 Vertex 時，`isAnalyticsDisabled()` 返回 `true`，完全跳過所有分析程式碼。

```bash
# 使用 AWS Bedrock（需要 AWS 憑證）
export CLAUDE_CODE_USE_BEDROCK=1

# 或使用 Google Vertex AI（需要 GCP 憑證）
export CLAUDE_CODE_USE_VERTEX=1
```

### 方法 3：定期自動清理

建立一個定時清理指令碼 `~/claude-privacy-clean.sh`：

```bash
#!/bin/bash
# 清除 Claude Code 追蹤資料（保留配置）

rm -rf ~/.claude/telemetry/
rm -rf ~/.claude/statsig/
rm -f ~/.claude/stats-cache.json
rm -f ~/.claude/history.jsonl
rm -rf ~/.claude/sessions/
rm -rf ~/.claude/paste-cache/
rm -rf ~/.claude/shell-snapshots/
rm -rf ~/.claude/session-env/
rm -rf ~/.claude/debug/

# 重置 device ID
python3 -c "
import json, os
p = os.path.expanduser('~/.claude.json')
if os.path.exists(p):
    with open(p,'r') as f: d = json.load(f)
    for k in ['userID','anonymousId','firstStartTime','claudeCodeFirstTokenDate']:
        d.pop(k, None)
    with open(p,'w') as f: json.dump(d,f,indent=2)
"

echo "[$(date)] Claude Code 追蹤資料已清除"
```

```bash
# 新增執行許可權
chmod +x ~/claude-privacy-clean.sh

# 可選：新增到 crontab 每天自動執行
# crontab -e
# 0 3 * * * ~/claude-privacy-clean.sh >> ~/claude-clean.log 2>&1
```

---

## 各級清理的影響對比

| 操作 | 對 Anthropic 的效果 | 對你的影響 |
|------|---------------------|------------|
| **重置 Device ID** | 無法關聯歷史使用資料 | 無感知，自動生成新 ID |
| **清除遙測** | 本地快取的分析事件不會補發 | 無感知 |
| **清除歷史** | — | 丟失命令歷史和 ctrl+r 搜尋 |
| **清除 OAuth** | 斷開賬號關聯 | 需要重新登入 |
| **完全重置** | 等同全新使用者 | 丟失所有自定義配置 |
| **禁用非必要流量** | 不再接收分析資料 | 特性開關可能不更新 |
| **使用 Bedrock/Vertex** | 分析程式碼完全不執行 | 需要雲廠商憑證 |

---

## 附：資料檔案速查表

| 檔案/目錄 | 包含的追蹤資料 |
|-----------|----------------|
| `~/.claude.json` | userID, anonymousId, oauthAccount, 首次使用時間, GitHub 倉庫對映 |
| `~/.claude/telemetry/` | 未上報的 1P 分析事件（JSON，含完整 EnvContext） |
| `~/.claude/statsig/` | Statsig stable_id, 特性開關快取 |
| `~/.claude/stats-cache.json` | 統計資料快取 |
| `~/.claude/history.jsonl` | 完整命令歷史（你輸入的每一條提示詞） |
| `~/.claude/sessions/` | 會話後設資料 |
| `~/.claude/paste-cache/` | 貼上內容的雜湊地址快取 |
| `~/.claude/file-history/` | Claude 做過的檔案修改記錄 |
| `~/.claude/debug/` | 除錯日誌 |
| `~/.claude/shell-snapshots/` | Shell 環境快照 |
| macOS Keychain → `claude-code` | OAuth access/refresh token |

---

*來自：AI超元域 | B站頻道：https://space.bilibili.com/3493277319825652*

*基於 Claude Code 原始碼逆向分析，2026-03-31*
