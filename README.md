# n8n Workflows

個人 n8n 自動化 workflow 收藏庫，涵蓋 QA 監控、報告自動化、通知整合等場景。
每個 workflow 皆可獨立匯入使用，並附有完整設定說明。

---

## Workflows

| 資料夾 | 說明 | 觸發方式 | 整合服務 | 狀態 |
|--------|------|----------|----------|------|
| [sentry-weekly-report](./sentry-weekly-report/) | Sentry 每週 Error 分析，AI 分類後發送 Slack 週報 | 排程（週一 09:00） | Sentry / Claude / Google Sheets / Slack | ✅ 上線中 |

---

## 通用 Prerequisites

以下為多數 workflow 共用的服務，建議提前準備：

- **n8n**：Self-hosted 或 n8n Cloud
- **Anthropic API Key**：[console.anthropic.com](https://console.anthropic.com)
- **Slack Bot Token**：[api.slack.com/apps](https://api.slack.com/apps)
- **Google Sheets OAuth2**：[Google Cloud Console](https://console.cloud.google.com)

---

## 匯入方式

1. 下載目標 workflow 資料夾內的 `workflow.json`
2. n8n → Workflows → **Import from file**
3. 依各 workflow 的 `README.md` 設定 Credentials 與參數

---

## 未來計畫

| Workflow | 說明 | 狀態 |
|----------|------|------|
| sentry-to-clickup | Sentry 新 issue 自動建立 ClickUp bug ticket | 規劃中 |
| app-store-monitor | App Store / Google Play 評論監控，AI 分類後發 Slack | 規劃中 |
| release-checklist | 發版前從 ClickUp 自動產生回歸測試清單 | 規劃中 |
| ci-failure-report | CI/CD 失敗率週報 | 規劃中 |
