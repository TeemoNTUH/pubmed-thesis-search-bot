# PubMed BigQuery 問答機器人快速開始

## 前置需求

- Node.js 18 或更新版本
- Cloudflare 帳戶與 Wrangler CLI 登入
- 已啟用 BigQuery API 並設定帳單的 Google Cloud 專案
- OAuth refresh token；OAuth 使用者在查詢專案須有 `BigQuery Job User` 權限

## 設定 Google OAuth

1. 在 Google Cloud 建立 OAuth 用戶端，應用程式類型選 **Web application**。
2. Authorized redirect URI 加入 `https://developers.google.com/oauthplayground`。
3. 若 OAuth consent screen 設為 External 且仍在 Testing，將授權帳號加入 Test users。
4. 在 OAuth Playground 啟用自有用戶端，scope 使用 `https://www.googleapis.com/auth/bigquery`，完成授權並交換 tokens。
5. 安全保存 Client ID、Client secret、Refresh token 和 Project ID。不要將憑證貼到聊天、程式碼或 Git。

## 設定 Cloudflare Secrets

在專案目錄執行下列指令，依 Wrangler 提示輸入值：

```powershell
npx wrangler secret put PUBMED_BIGQUERY_CREDENTIALS
```

輸入以下格式的 JSON，並替換成你自己的憑證：

```json
{"client_id":"...","client_secret":"...","refresh_token":"..."}
```

再設定 Google Cloud Project ID：

```powershell
npx wrangler secret put PUBMED_BIGQUERY_PROJECT_ID
```

`wrangler secret put` 會建立並部署 Worker 新版本。`wrangler.jsonc` 已啟用 `PUBMED_QUERY_MODE`，一般不需再修改。

## 建置與部署

```powershell
npm ci
npm run build
npm run deploy
```

部署後開啟 README 中的 Cloudflare Workers Demo，使用非敏感的醫學研究問題測試。聊天可能呼叫 BigQuery、NCBI PubMed 與 Workers AI，請留意用量與費用；不要輸入可識別病患的資料。

## 常見檢查

- BigQuery API 已啟用，Project ID 正確，且查詢專案已設定帳單。
- OAuth 使用者具有 `BigQuery Job User` 權限。
- Cloudflare Worker 的 Secrets 清單包含 `PUBMED_BIGQUERY_CREDENTIALS` 與 `PUBMED_BIGQUERY_PROJECT_ID`，兩者皆為 Secret 類型。
- Refresh token 需由 OAuth Playground 的自有用戶端取得；不要使用短效 Access token 取代。
