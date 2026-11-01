# PubMed BigQuery QA 機器人設置指南

## 概述

這個項目現在支持通過 Google BigQuery API 查詢 PubMed 醫學文獻數據庫。用戶可以提出醫學研究問題，系統會自動從 PubMed 搜索相關文章，並使用 AI 生成基於這些文獻的答案。

## 功能

- **PubMed 搜索**：通過 Google BigQuery API 搜索 PubMed 數據庫
- **AI 問答**：使用 OpenAI GPT-OSS-120B 模型生成答案
- **文獻引用**：自動引用和展示相關的 PubMed 文章
- **臨床文獻檢索**：結合 BigQuery 公開資料、NCBI 即時搜尋與官方紀錄核對

## 前置要求

1. **Google Cloud 帳戶**
  - 選擇具有帳單設定的查詢專案並啟用 BigQuery API
  - OAuth 使用者具備 `BigQuery Job User` 角色
  - 建立 OAuth Web application 用戶端並取得 refresh token

2. **Cloudflare 帳戶**
   - 具有 Workers 計劃的有效帳戶
   - Wrangler CLI 已安裝

3. **Node.js**
   - 版本 18 或更高版本

## 詳細設定步驟

### 步驟 1：設定 Google Cloud

1. 在 [Google Cloud Console](https://console.cloud.google.com) 選擇或建立查詢專案，啟用 BigQuery API，並確認專案已設定帳單。
2. 在 **IAM** 將執行 OAuth 授權的使用者授予 `BigQuery Job User`。查詢公開資料集不需要 `BigQuery Data Editor`。
3. 在 **Google Auth Platform** 設定 OAuth consent screen。若 Audience 為 External 且仍在 Testing，將授權帳號加入 Test users。
4. 建立 OAuth 用戶端，應用程式類型選 **Web application**，Authorized redirect URI 加入 `https://developers.google.com/oauthplayground`。
5. 在 [OAuth Playground](https://developers.google.com/oauthplayground) 設定自己的 OAuth 用戶端，scope 使用 `https://www.googleapis.com/auth/bigquery`，完成授權並交換 tokens。
6. 安全保存 Client ID、Client secret、Refresh token 和 Google Cloud Project ID。不要將任何憑證貼到聊天、程式碼或 Git。

若組織政策允許服務帳戶金鑰，也可提供包含 `client_email` 和 `private_key` 的服務帳戶 JSON；不需要將該帳戶設為 BigQuery Data Editor。

### 步驟 2：設定 Cloudflare Secrets

在專案目錄執行下列指令，依 Wrangler 提示輸入值。`PUBMED_BIGQUERY_CREDENTIALS` 的內容為 OAuth JSON：

```json
{"client_id":"...","client_secret":"...","refresh_token":"..."}
```

```bash
npx wrangler secret put PUBMED_BIGQUERY_CREDENTIALS
npx wrangler secret put PUBMED_BIGQUERY_PROJECT_ID
```

Wrangler secret 指令會建立並部署 Worker 新版本。也可在 Cloudflare Dashboard 的 Worker **Settings → Variables and Secrets** 新增這兩項，類型都必須選 Secret，然後按 Deploy。

### 步驟 3：建置與部署

```bash
npm ci
npm run build
npm run deploy
```

## 使用方式

### 訪問應用

部署後，訪問您的 Worker URL：

1. **主頁與 PubMed 問答**：`https://your-worker-url.workers.dev/`
2. **PubMed QA 演示頁**：`https://your-worker-url.workers.dev/pubmed`

### 示例問題

在 PubMed 模式中，您可以提出以下類型的問題：

- "What are the latest studies on COVID-19 vaccines?"
- "Tell me about CRISPR gene therapy recent developments"
- "Summarize current research on cancer immunotherapy"
- "What are the recent findings on neurodegenerative diseases?"

## 架構

### 後端組件

1. **`src/pubmedBigQuery.ts`**
   - `PubMedBigQueryClient` 類：處理 BigQuery 查詢
   - 搜索和獲取文章詳情的方法
   - OAuth 2.0 認證邏輯

2. **`src/chatState.ts`**
   - 更新的 `ChatState` Durable Object
   - PubMed 集成支持
   - 上下文管理

3. **`src/index.ts`**
   - PubMed 演示頁面路由
   - 新的 `/pubmed` 端點
   - HTML UI 生成

### 前端組件

1. **`src/client.tsx`**
   - PubMed 臨床研究問答介面
   - 實時消息渲染

## API 端點

| 端點 | 方法 | 描述 |
|------|------|------|
| `/chat/init` | POST | 初始化聊天會話 |
| `/chat/{id}` | GET | 獲取聊天消息歷史 |
| `/chat/{id}` | POST | 發送消息（支持 `usePubMed` 參數） |
| `/chat/{id}` | DELETE | 清除聊天歷史 |
| `/pubmed` | GET | PubMed 演示頁面 |

## 環境變量

| 變量 | 類型 | 描述 |
|------|------|------|
| `PUBMED_BIGQUERY_CREDENTIALS` | Secret | OAuth JSON（`client_id`、`client_secret`、`refresh_token`），或服務帳戶 JSON |
| `PUBMED_BIGQUERY_PROJECT_ID` | Secret | Google Cloud 項目 ID |
| `PUBMED_QUERY_MODE` | Var | 啟用/禁用 PubMed 功能 (true/false) |

## 常見問題

### Q: 我收到 "BigQuery API error" 怎麼辦？

A: 確保：
1. 在 Google Cloud Console 中啟用了 BigQuery API
2. OAuth 使用者在查詢專案有 `BigQuery Job User` 權限
3. 項目 ID 正確

### Q: 搜索返回很少或沒有結果

A: 
1. 檢查 PubMed 數據集是否可訪問
2. 確保查詢語法正確
3. 嘗試使用更通用的搜索詞

### Q: 如何在開發中禁用 PubMed？

A: 在 `wrangler.jsonc` 的 `vars` 區塊將 `PUBMED_QUERY_MODE` 設為 `false`：
```json
{
   "vars": {
      "PUBMED_QUERY_MODE": "false"
   }
}
```

## 安全考慮

1. **密鑰管理**：永遠不要在代碼中硬編碼 Google 認證信息
2. **速率限制**：BigQuery 有配額限制，監控使用情況
3. **成本**：BigQuery 查詢可能會產生費用，設置 Google Cloud 預算警報

## 故障排除

### 部署失敗

```bash
# 清理並重新安裝
rm -rf node_modules package-lock.json
npm install

# 檢查 Wrangler 配置
npx wrangler deploy --dry-run
```

### 本地開發問題

```bash
# 啟用詳細日誌
WRANGLER_LOG=debug npm run dev

# 檢查類型
npm run cf-typegen
```

## 技術棧

- **後端**：Cloudflare Workers + Durable Objects
- **AI**：OpenAI GPT-OSS-120B on Workers AI
- **數據**：Google BigQuery + PubMed
- **前端**：React 18 + Emotion CSS-in-JS
- **構建**：esbuild + TypeScript
- **部署**：Wrangler

## 下一步

1. **自定義搜索**：修改 `pubmedBigQuery.ts` 中的 SQL 查詢
2. **增強上下文**：添加 MeSH 詞彙或其他元數據
3. **多語言支持**：擴展 UI 以支持更多語言
4. **緩存優化**：在 Durable Objects 中實現文章緩存

## 參考資源

- [PubMed BigQuery 文檔](https://www.ncbi.nlm.nih.gov/research/bionlm/APIs/BioC-PubMed/Server/)
- [Google BigQuery 文檔](https://cloud.google.com/bigquery/docs)
- [Cloudflare Workers 文檔](https://developers.cloudflare.com/workers/)
- [Cloudflare Durable Objects](https://developers.cloudflare.com/durable-objects/)

## 許可

本專案程式碼採 MIT License；第三方資料、商標、服務及模型仍依其各自授權與使用條款。
