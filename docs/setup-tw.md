# VibSDK 設定指南

VibSDK 本地首次設定指南 - 讓您的 AI 編碼平台在本地運行並準備好部署。

**在開始之前，請務必閱讀完整指南以了解重要注意事項，並準備好所有必要資訊。**

**重要提示：Cloudflare WARP 已知會導致本地開發中使用的匿名 Cloudflared tunnel 出現問題，可能導致預覽無法載入。如果您在本地開發預覽時遇到問題，請嘗試在使用 VibSDK 時停用 WARP（完整模式）。您可以使用 WARP 的 DNS only (1.1.1.1) 模式**

## 前置需求

在開始之前，請確保您具備：

### 必要項目
- **Node.js** (v18 或更新版本)
- **Cloudflare 帳號**，具有 API 存取權限
- **Cloudflare API Token**，具有適當權限

### 建議項目
- **Bun**
- 在 Cloudflare 中設定的**自訂網域**（用於正式環境部署）

### 正式環境功能所需
- **Workers Paid Plan**（用於遠端 Cloudflare 資源）
- **Workers for Platforms** 訂閱（用於應用程式部署功能）
- **Advanced Certificate Manager**（如果使用第一層子網域）

## 快速開始

使用我們的自動化設定腳本是啟動 VibSDK 最快的方式：

```bash
# 建議使用 Bun 而非 npm。如果尚未安裝 bun，請先安裝
curl -fsSL https://bun.sh/install | bash
# 然後安裝相依套件並執行設定
bun install
bun run setup
```

此互動式腳本將引導您完成整個設定流程，包括：

- **套件管理器設定**（自動安裝 Bun 以獲得更好的效能）
- **Cloudflare 憑證**收集（Account ID 和 API Token）
- **網域設定**（自訂網域或 localhost 用於開發）
- **遠端設定**（選用的正式環境部署設定）
- **AI Gateway 設定**（建議使用 Cloudflare AI Gateway）
- **API 金鑰收集**（OpenAI、Anthropic、Google AI Studio 等）
- **OAuth 設定**（Google、GitHub 登入 - 選用）
- **資源建立**（KV namespaces、D1 databases、R2 buckets、AI Gateway）
- **檔案生成**（`.dev.vars` 和選用的 `.prod.vars`）
- **設定更新**（`wrangler.jsonc` 和 `vite.config.ts`）
- **資料庫設定**（schema 生成和 migrations）
- **Template 部署**（範例應用程式 templates 至 R2）
- **就緒報告**（完整狀態和後續步驟）

## 設定期間需要的資訊

設定腳本會要求您提供以下資訊：

### Cloudflare 帳號資訊

1. **Account ID**：可在您的 Cloudflare dashboard 側邊欄找到
2. **API Token**：在 Cloudflare dashboard 的「My Profile」>「API Tokens」中，建立一個 token（建議使用「Edit Cloudflare Workers」template），需具備以下設定：
   - Your Account - Workers KV Storage:Edit, Workers Scripts:Edit, Account Settings:Read, Workers Tail:Read, Workers R2 Storage:Edit, Cloudflare Pages:Edit, Workers Builds Configuration:Edit, Workers Agents Configuration:Edit, Workers Observability:Edit, Containers:Edit, D1:Edit, AI Gateway:Read, AI Gateway:Edit, AI Gateway:Run, Cloudchamber:Edit, Browser Rendering:Edit
   - All zones - Workers Routes:Edit
   - All users - User Details:Read, Memberships:Read

   **如果使用 `Edit Cloudflare Workers` template，請確保手動新增上述缺少的權限。**

   **重要**：某些功能如 D1 databases 和 R2 可能需要付費的 Cloudflare 方案。

### 網域設定

**使用自訂網域：**
```bash
Enter your custom domain (or press Enter to skip): myapp.com
✅ Custom domain set: myapp.com
Use remote Cloudflare resources (KV, D1, R2, etc.)? (Y/n): 
Configure for production deployment? (Y/n): 
```

**不使用自訂網域：**
```bash
Enter your custom domain (or press Enter to skip): [press Enter]
⚠️  No custom domain provided.
   • Remote Cloudflare resources: Not available
   • Production deployment: Not available
   • Only local development will be configured

Continue with local-only setup? (Y/n): 
```

### AI Gateway 設定

**Cloudflare AI Gateway（建議）**
- **自動 token 設定**：選擇此選項時，`CLOUDFLARE_AI_GATEWAY_TOKEN` 會自動設定為您的 API token
- **無需手動設定**：腳本會處理所有 AI Gateway 驗證
- **更好的效能**：包含快取、速率限制和監控

**Custom OpenAI URL（替代方案）**
- 適用於已有現成 OpenAI 相容端點的使用者
- 需要在 `worker/agents/inferutils/config.ts` 中手動設定模型

### AI Provider 選擇

設定腳本提供多個 AI provider，支援智慧多選：

**可用的 Providers：**
1. **OpenAI**（用於 GPT 模型）
2. **Anthropic**（用於 Claude 模型）
3. **Google AI Studio**（用於 Gemini 模型）- **預設且建議**
4. **Cerebras**（用於開源模型）
5. **OpenRouter**（用於各種模型）
6. **Custom provider**（用於任何其他 provider）

**Provider 選擇：**
- 使用逗號分隔的數字選擇多個 providers（例如：`1,2,3`）
- 每個選擇的 provider 都會提示輸入其 API 金鑰
- Custom providers 會自動生成 `PROVIDER_NAME_API_KEY` 變數
- Custom providers 會自動新增至 `worker-configuration.d.ts`

### 重要的模型設定注意事項

**Google AI Studio（建議）：**
- 預設模型設定使用 Gemini 模型
- 無需額外編輯 `worker/agents/inferutils/config.ts`
- 最佳相容性 - 這是官方部署在 https://build.cloudflare.dev 使用的模型
- 您可以從 https://aistudio.google.com/ 取得免費 API 金鑰

**其他 Providers：**
- **強烈警告**：您必須編輯 `worker/agents/inferutils/config.ts`
- 將預設模型設定從 Gemini 改為您選擇的 providers
- 模型格式：`<provider-name>/<model-name>`（例如：`openai/gpt-4`、`anthropic/claude-3.5-sonnet`）
- 檢查 fallback 模型設定

**不使用 AI Gateway：**
- **需要手動編輯 config.ts** 以設定所有模型
- 模型名稱必須遵循 `<provider-name>/<model-name>` 格式

### OAuth 設定

腳本也會要求 OAuth 憑證：

- **Google OAuth**：用於使用者驗證和登入（非 AI Studio 存取）
- **GitHub OAuth**：用於使用者驗證和登入
- **GitHub Export OAuth**：用於將生成的應用程式匯出至 GitHub repositories（與登入 OAuth 分開）

**如果您不提供 OAuth 憑證，預設在登入時，您只能使用基於電子郵件的註冊/登入。**

### Docker 需求

本地開發使用 sandbox instances 需要 Docker。在執行平台之前，請確保 Docker 已安裝並正在運行。

## 手動設定（替代方案）

如果您偏好手動設定：

### 1. 建立 `.dev.vars` 檔案

複製 `.dev.vars.example` 至 `.dev.vars` 並填入您的值：

```bash
cp .dev.vars.example .dev.vars
```

### 2. 設定必要變數

```bash
# 必要項目
CLOUDFLARE_API_TOKEN="your-api-token"
CLOUDFLARE_ACCOUNT_ID="your-account-id"

# 安全性
JWT_SECRET="generated-secret"
WEBHOOK_SECRET="generated-secret"

# 網域（選用）
CUSTOM_DOMAIN="your-domain.com"
```

### 3. 建立 Cloudflare 資源

在您的 Cloudflare 帳號中建立必要資源：
- `VibecoderStore` 的 KV Namespace
- 名為 `vibesdk-db` 的 D1 Database
- 名為 `vibesdk-templates` 的 R2 Bucket

### 4. 更新 `wrangler.jsonc`

使用步驟 3 的 ID 更新 `wrangler.jsonc` 中的資源 ID。

## 開始開發

設定完成後：

```bash
# 設定資料庫
bun run db:migrate:local

# 啟動開發伺服器
bun run dev
```

在 `http://localhost:5173` 訪問您的應用程式

**重要提示**：如果您在設定期間未指定任何 oauth 憑證，您需要首次註冊帳號。

## 疑難排解

### 常見問題

**D1 Database「Unauthorized」錯誤**：這通常表示：
- 您的 API token 缺少「D1:Edit」權限
- 您的帳號無法存取 D1（可能需要付費方案）
- 您已超過 D1 database 配額
- **解決方案**：更新您的 API token 權限或升級您的 Cloudflare 方案

**權限錯誤**：確保您的 API token 具有上述列出的所有必要權限。

**找不到網域**：確保您的網域：
- 已新增至 Cloudflare
- DNS 已正確設定
- API token 具有 zone 權限

**資源建立失敗**：檢查您的帳號是否具有：
- 可用的 KV namespace 配額（免費方案為 10 個）
- D1 database 配額（可能需要付費方案）
- R2 bucket 配額（可能需要付費方案）
- 所請求功能的適當方案等級

**R2 Bucket「Unauthorized」錯誤**：這通常表示：
- 您的 API token 缺少「R2:Edit」權限
- 您的帳號無法存取 R2（可能需要付費方案）
- 您已超過 R2 bucket 配額
- **解決方案**：更新您的 API token 權限或升級您的 Cloudflare 方案

**AI 設定問題**：
- **「AI Gateway token already configured」但 token 不在 .dev.vars 中**：重新執行設定，這是已修復的 bug
- **使用 custom providers 時模型無法運作**：編輯 `worker/agents/inferutils/config.ts` 以變更預設模型設定
- **無法識別 custom provider**：檢查 provider 是否已新增至 `worker-configuration.d.ts`
- **AI Gateway 建立失敗**：確保您的 API token 具有 AI Gateway 權限

**本地開發與 Tunnel 問題**：
- **Cloudflared tunnel timeout**：等待 20-30 秒，然後重新整理。Tunnel 建立可能較慢
- **「Tunnel creation failed」**：這偶爾是正常的。應用程式仍可使用一般預覽 URL
- **Sandbox instances 停止運作**：如果它們成功重啟，這是正常行為。只有在持續發生時才需要擔心
- **無法存取預覽 URL**：檢查 tunnel 是否仍在建立中，或嘗試重新整理 instance
- **macOS 上的多個 port 暴露問題**：使用 tunnels（`USE_TUNNEL_FOR_PREVIEW=true`）- 這是預設設定

**Deploy to Cloudflare Button 問題（Chat 介面）**：
- **「本地部署按鈕無法運作」**：Chat 介面的部署按鈕需要自訂網域、初始部署和遠端 dispatch bindings
- **「找不到 Dispatch namespace」**：請先將您的 VibSDK 專案部署至 Cloudflare 至少一次
- **「部署失敗並出現驗證錯誤」**：確保您的自訂網域已正確設定並部署
- **注意**：這是指從 chat 介面部署生成的應用程式，而非 GitHub repository 部署

**企業網路問題**：
- **Docker containers 中的 SSL/TLS 憑證錯誤**：企業網路通常使用自訂 root CA 憑證
- **Cloudflared tunnel 失敗**：可能被企業 proxy 阻擋或需要憑證信任
- **套件安裝失敗**：npm/bun 安裝可能因憑證驗證而失敗

**企業網路解決方案**：
如果您在使用自訂 SSL 憑證的企業網路上，您需要修改 `SandboxDockerfile`：

1. **複製您的企業 root CA 憑證**至專案根目錄（不要提交至 git！）
2. **編輯 SandboxDockerfile** 以包含您的憑證：

```dockerfile
# 新增您公司的 Root CA 憑證以供企業網路存取
COPY your-root-ca.pem /usr/local/share/ca-certificates/your-root-ca.crt
RUN update-ca-certificates

# 為 cloudflared 和其他工具設定 SSL 環境變數
ENV SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt
ENV NODE_EXTRA_CA_CERTS=/usr/local/share/ca-certificates/your-root-ca.crt
ENV CURL_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt
```

**⚠️ 安全警告**：切勿將企業 CA 憑證提交至公開 repositories。使用 `.gitignore` 排除憑證檔案，僅用於本地開發。

### 取得協助

1. 檢查設定報告以了解特定問題和建議
2. 查閱 Cloudflare Workers 文件
3. 確保符合所有前置需求

## 正式環境部署

如果您在設定期間設定了遠端部署，您將擁有準備好用於正式環境的 `.prod.vars` 檔案。使用以下指令部署：

```bash
bun run deploy
```

這將會：
- 建置應用程式
- 更新 Cloudflare 資源
- 部署至 Cloudflare Workers
- 套用資料庫 migrations
- 設定自訂網域路由（如果有指定）

### 僅正式環境設定

如果您最初只設定了本地開發，您可以稍後設定正式環境：

1. **再次執行設定**並在遠端部署設定時選擇「yes」
2. 在提示時**提供正式環境網域**
3. 使用 `npm run deploy` **部署**

### 手動正式環境設定

或者，根據 `.dev.vars` 手動建立 `.prod.vars`，但需包含：
- `CUSTOM_DOMAIN` 中的正式環境網域
- 正式環境 API 金鑰和 secrets
- `ENVIRONMENT="prod"`

## 後續步驟

設定完成後：

1. 使用 `npm run dev` **開始開發**
2. **訪問** `http://localhost:5173` 以存取 VibSDK
3. **嘗試生成**您的第一個 AI 驅動應用程式
4. 準備好時使用 `npm run deploy` **部署至正式環境**

## 設定後的檔案結構

設定腳本會建立和修改這些檔案：

```
vibesdk/
├── .dev.vars              # 本地開發環境變數
├── .prod.vars             # 正式環境環境變數（如果已設定）
├── wrangler.jsonc         # 已更新資源 ID 和網域
├── vite.config.ts         # 已更新遠端/本地 bindings
├── migrations/            # 資料庫 migration 檔案
└── templates/             # Template repository（已下載）
```

## 總結

VibSDK 設定腳本提供全面、智慧的設定體驗：

### **主要功能：**
- **簡化的網域設定** - 一次性網域設定，清楚說明功能影響
- **智慧 AI provider 選擇** - 多 provider 支援與自動設定
- **AI Gateway 自動化** - 自動 token 設定和配置
- **Custom provider 支援** - 動態 API 金鑰生成和 worker 設定更新
- **正式環境就緒** - 本地開發和正式環境部署設定
- **使用者友善的預設值** - Y/n 提示與清楚的預設指示

### **設定內容：**
- Cloudflare 資源（KV、D1、R2、AI Gateway、dispatch namespaces）
- 環境變數（.dev.vars 和 .prod.vars）
- Worker 設定（wrangler.jsonc、worker-configuration.d.ts）
- 資料庫設定和 migrations
- Template 部署
- ARM64 相容性

設定腳本處理從基本 Cloudflare 資源建立到進階 AI provider 設定的所有事項，無論您的 Cloudflare 方案或 AI provider 偏好如何，都能輕鬆開始。

如果在設定期間遇到任何問題，請查看上述疑難排解部分，或參考腳本在結束時提供的完整狀態報告。

## 重要注意事項與已知問題

### **使用 Cloudflared Tunnels 的本地開發**

**預設行為**：本地開發預設使用 cloudflared tunnels（`USE_TUNNEL_FOR_PREVIEW=true`）

**為什麼使用 Tunnels？**
- **MacBook 相容性**：Cloudflare sandbox SDK Docker images 在 macOS 上使用多個暴露 ports 時有問題
- **簡化網路設定**：避免複雜的 localhost proxy 設定
- **快速開發**：提供即時外部存取以供測試

**Tunnel 限制**：
- **啟動時間**：Tunnel 建立可能需要 10-20 秒
- **Timeouts**：Tunnel 建立可能偶爾 timeout（這是正常的）
- **外部相依性**：需要網際網路連線和 cloudflare.com 存取

### **「Deploy to Cloudflare」Button 限制（Chat 介面）**

Chat 介面中的「Deploy to Cloudflare」按鈕（用於生成的應用程式）對本地開發有特定需求：

> **注意**：這是指 VibSDK 平台 chat 介面中的部署按鈕，而非 GitHub repository 部署按鈕。

**需求**：
1. **自訂網域**必須在設定期間正確設定
2. **初始部署** - 專案必須至少部署一次至您的 Cloudflare 帳號
3. **遠端 dispatch bindings** - `wrangler.jsonc` 必須啟用遠端 dispatch namespace
4. **Dispatch worker** - 您的帳號中必須有正在運行的 dispatch worker

**為什麼有這些需求？**
- 部署功能使用 Cloudflare 的 dispatch namespace 系統
- Dispatch 需要您帳號中正在運行的 worker 來處理部署請求
- vibesdk 尚未支援僅本地開發模式

**目前狀態**：讓「Deploy to Cloudflare」在完全僅本地模式下運作尚未實作。

### **Sandbox Instance 行為**

**正常行為**：
- **Instance 重啟**：Sandbox 部署可能偶爾停止並重啟
- **暫時性失敗**：預期會有短期部署失敗
- **自我修復**：系統會自動重試和恢復

**何時需要擔心**：
- 持續失敗超過 5 分鐘以上
- 完全無法建立 instances
- 持續的網路問題

**什麼是正常的**：
- 快速解決的個別 instance 失敗
- 偶爾的 tunnel 連線問題
- 重啟期間的短暫無法使用 如果問題持續存在，請在 GitHub 上開啟 issue，並附上狀態報告和您認為有幫助的任何其他資訊。
