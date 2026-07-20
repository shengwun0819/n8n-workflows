# Sentry Weekly Report — n8n Workflow

自動化 Sentry 週報：每週定時自動撈取指定專案過去 7 天的 unresolved errors，透過 AI 分析分類後發送至 Slack，並將原始資料存入 Google Sheets 供歷史趨勢對比。

---

## 功能

- 每週一 09:00 台北時間自動執行
- 撈取過去 7 天 Sentry unresolved errors（Top 40，依頻率排序）
- AI 自動分類（Network / BLE / Firmware / 交易 / App Hang 等）
- 對比上週資料，標示 🆕 新增 / 🔄 持續 / ✅ 改善
- Slack 摘要自動發送至指定頻道
- Google Sheets 歷史紀錄（每週建立新分頁）

---

## Workflow 架構

```
Schedule Trigger
  → Create sheet（建立當週 Google Sheets 分頁）
  → HTTP Request（Sentry API，撈 100 issues）
  → Aggregate（合併為 1 item）
  → Code in JavaScript（取 Top 40，精簡欄位）
  ├→ Append row in sheet（寫入 Google Sheets）
  └→ Aggregate1（重新合併）
       → AI Agent（分析 + Google Sheets Tool 讀取上週資料）
            → Send a message（Slack）
```

---

## Prerequisites

匯入前需在 n8n 建立以下 Credentials：

| Credential | 類型 | 用途 |
|------------|------|------|
| Sentry API Token | HTTP Header Auth | 撈取 Sentry issues |
| AI Model API Key | Anthropic / Google Gemini 等 | AI 分析（依所選模型設定） |
| Google Sheets OAuth2 | Google Sheets OAuth2 API | 儲存歷史紀錄 |
| Slack Bot Token | Slack API | 發送週報 |

### Sentry API Token
1. Sentry → Settings → Account → API → Auth Tokens
2. 建立 token，勾選 `project:read`、`event:read`

### AI Model API Key
依選用的模型取得對應 API Key：
- **Anthropic（Claude）**：[console.anthropic.com](https://console.anthropic.com) → API Keys → Create Key
- **Google Gemini**：[aistudio.google.com](https://aistudio.google.com) → Get API Key

### Google Sheets OAuth2
1. [Google Cloud Console](https://console.cloud.google.com) → 建立專案
2. 啟用 Google Sheets API 和 Google Drive API
3. 建立 OAuth 2.0 憑證（Web Application）
4. Redirect URI 填入：`https://<your-n8n-domain>/rest/oauth2-credential/callback`

### Slack Bot Token
1. [api.slack.com/apps](https://api.slack.com/apps) → Create App
2. OAuth & Permissions → Bot Token Scopes：`chat:write`、`chat:write.public`
3. Install to Workspace → 複製 `xoxb-` token

---

## 匯入步驟

1. n8n → Workflows → **Import from file**
2. 選擇 `workflow.json`
3. 更新所有節點的 Credentials
4. 調整以下設定（依實際環境）：

### HTTP Request 節點
- URL 中的 `your-org-slug` → 替換為你的 Sentry organization slug
- URL 中的 `your-project-slug` → 替換為你的 Sentry project slug

### Slack 節點
- Channel → 替換為你的 Slack 頻道名稱
- Options → **Use Markdown?** → 開啟（確保 mrkdwn 格式正確 render）
- Message Text 中的 `your-org.sentry.io/issues/views/YOUR_VIEW_ID/` → 替換為你的 Sentry saved search URL
- Message Text 中的 `YOUR_PROJECT_ID` → 替換為你的 Sentry project ID
- Message Text 中的 `YOUR_GOOGLE_SHEET_ID` → 替換為你的 Google Sheets document ID

### Google Sheets 節點（Create sheet & Append row）
- Document → 替換為你的 Google Sheets 檔案

5. **Publish** 啟用 workflow

---

## 設定說明

### 調整分析期間（7D / 14D）

修改 HTTP Request 節點的 `start` 參數：
```
// 7 天
{{ $now.minus({days: 7}).toISO() }}

// 14 天
{{ $now.minus({days: 14}).toISO() }}
```

### 調整分析數量（Top 40）

修改 Code 節點：
```javascript
const slimmed = issues.slice(0, 40).map(...)
// 改為 slice(0, 50) 等，視 API rate limit 調整
```

> ⚠️ 若使用 Anthropic 個人帳號，Haiku 模型限制 10,000 TPM，超過會報錯，建議上限 40 issues。
> Google Gemini 免費額度較寬鬆，可視需求調整數量。

### 排程時間

Schedule Trigger 預設為 **Sunday 21:00 NY（= 台北時間週一 09:00）**。
n8n server 時區為 America/New_York，以此換算可在台北時間週一早上觸發。
如需調整，修改 Schedule Trigger 節點的 Day 與 Hour 設定。

---

## Sentry 查詢條件

```
is:unresolved level:error !message:"GQL" !message:"App Loaded" !message:"successfully" !message:"Success" !message:"Network Error"
sort: freq（依頻率排序）
limit: 100
```

- `is:unresolved`：只撈未解決的 issues
- `level:error`：只撈 error 等級，排除 info / warning（如 updateFirmware is called 等日誌）
- `!message:"GQL"`：排除 GraphQL 相關 errors
- `!message:"App Loaded"`：排除 App 啟動成功日誌（非 error）
- `!message:"successfully"` / `!message:"Success"`：排除交易成功等 info 事件
- `!message:"Network Error"`：排除泛型網路錯誤（AxiosError: Network Error、Error: Network Error 等），這類錯誤通常為用戶端網路不穩，無法由開發端直接修復

---

## AI 分析模型

- **預設模型**：Claude Haiku 4.5（`claude-haiku-4-5-20251001`），每次執行約 $0.01–$0.02
- **替代選項**：Google Gemini（`models/gemini-3.5-flash` 等），免費額度充足適合個人使用
- **替換方式**：AI Agent 節點 → Chat Model → 換接其他模型節點

---

## Google Sheets 結構

每週自動建立新分頁，命名格式：`yyyy-MM-dd`（台北時間）

| 欄位 | 說明 |
|------|------|
| run_date | 執行日期 |
| issue_id | Sentry Issue ID（如 YOUR-PROJECT-WDA） |
| title | 錯誤標題 |
| events | 事件數 |
| users | 影響用戶數 |
| first_seen | 首次出現時間 |
| last_seen | 最後出現時間 |