# Claude Code 預設 Effort Level 設定為 Max

## 問題

`/effort max` 命令只對當前會話生效，重啟 Claude Code 後會恢復為預設的 `medium`。

## 解決方案

在 `~/.zshrc` 中新增環境變數：

```bash
export CLAUDE_CODE_EFFORT_LEVEL=max
```

新開終端視窗後自動生效。

## 為什麼不能用 settings.json

Claude Code 的 `~/.claude/settings.json` 支援 `effortLevel` 欄位，但只接受 `low`、`medium`、`high` 三個值。`max` 是 Opus 4.6 專屬的深度推理模式，只能透過環境變數持久化。

## Effort Level 一覽

| 級別 | 說明 |
|------|------|
| `low` | 快速響應，適合簡單問答 |
| `medium` | 預設級別 |
| `high` | 更深入的推理 |
| `max` | 最深度推理，無 token 限制（僅 Opus 4.6） |

## 其他設定方式

- **啟動引數**（單次）：`claude --effort max`
- **會話內**（單次）：`/effort max`
- **環境變數**（永久）：`export CLAUDE_CODE_EFFORT_LEVEL=max` ← 推薦

---

*記錄於 2026-03-30*
