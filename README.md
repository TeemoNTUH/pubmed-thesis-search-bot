# 🔬 PubMed BigQuery 醫學問答機器人

**🔗 Cloudflare Workers Demo：<https://pubmed-thesis-search-bot.teemo-f15.workers.dev>**

醫院實習期間開發的醫學文獻問答系統。透過 Google BigQuery 上的 PubMed 公開資料集查詢數百萬篇醫學研究論文，並由 LLM 依實際文獻內容生成有 PMID 引用依據的回答，協助快速掌握特定主題的研究現況。

> 本專案是供來院實習生了解 RAG 智慧問答系統概念的展示專案，僅供學習與展示，不作為醫療診斷或治療建議。

程式碼 MIT 授權範圍及 PubMed 第三方資料說明請見 [DATA_NOTICES.md](./DATA_NOTICES.md)。

## 功能

- **PubMed 文獻檢索**：透過 Google BigQuery 查詢大量醫學研究論文，例如：「請提供癌症免疫療法的最新文獻摘要」
- **AI 生成回答**：根據實際文獻內容生成專業回答，而非憑空生成
- **引用管理**：自動附上相關論文的 PMID 引用
- **對話式介面**：可針對查到的文獻持續追問

## 系統運作方式

1. 使用者提問後，Worker 將問題轉換為 BigQuery 查詢，搜尋 PubMed 公開資料集中的相關論文
2. 查到的論文摘要與中繼資料（PMID、標題等）作為上下文，交給 Workers AI 上的 LLM（`gpt-oss-120b`）
3. LLM 依實際文獻內容生成回答，並附上引用的 PMID，避免無依據的回答
4. Durable Objects 保存對話狀態，支援多輪追問

## 技術架構 (Cloudflare + Google Cloud)

| 元件 | 用途 |
|---|---|
| Cloudflare Workers | Serverless 後端 / API，同時提供線上 Demo |
| Durable Objects | 對話狀態管理 |
| Workers AI | 模型推論（`@cf/openai/gpt-oss-120b`） |
| Google BigQuery | PubMed 公開資料集候選文獻查詢 |
| NCBI E-utilities | 即時 PubMed 搜尋與官方紀錄核對 |

前端：React 18 + TypeScript + Emotion（CSS-in-JS）+ esbuild 打包

核心程式：
```
src/
├── index.ts            # Worker 主入口與路由
├── chatState.ts         # Durable Object，對話狀態管理
├── client.tsx            # React 前端
└── pubmedBigQuery.ts     # PubMed BigQuery 查詢客戶端
```

## 本機開發 / 部署

```bash
npm install
npm run dev      # 本機開發
npm run build    # 打包前端
```

### 設定 Google BigQuery

1. 在 [Google Cloud Console](https://console.cloud.google.com) 選擇查詢專案、啟用 BigQuery API，並確認帳單與 `BigQuery Job User` 權限
2. 建立 OAuth Web application 用戶端，使用 Google OAuth Playground 取得 refresh token
3. 將用戶端資訊和 refresh token 組成 JSON，設定 Wrangler secrets（**不要**把憑證寫進程式碼或 commit 進 repo）：

```bash
wrangler secret put PUBMED_BIGQUERY_CREDENTIALS
```

在 Wrangler 提示中輸入以下格式的 JSON（填入自己的值）：

```json
{"client_id":"...","client_secret":"...","refresh_token":"..."}
```

再設定 Google Cloud Project ID：

```bash
wrangler secret put PUBMED_BIGQUERY_PROJECT_ID
```

4. 部署：`npm run deploy`

更完整的設定步驟見 [PUBMED_SETUP.md](./PUBMED_SETUP.md)，快速上手範例見 [PUBMED_QUICK_START.md](./PUBMED_QUICK_START.md)。

## 已知限制與未來規劃

- BigQuery 查詢成本會隨使用量增加，正式上線前需評估用量與費用
- 目前查詢範圍為 PubMed 公開資料集摘要，未涵蓋全文內容
- 規劃：加入依期刊影響係數或發表年份排序的檢索選項

## 專案背景

本專案以經原作者授權的 [Cloudflare Workers + AutoRAG 範例](https://github.com/elizabethsiegle/chat-w-taylor-on-newheights-and-travis-gq-autorag-openaioss) 為起點，在醫院實習期間加入 PubMed BigQuery 文獻檢索功能，改造為醫學文獻問答用途。
