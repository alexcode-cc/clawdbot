# OpenClaw 提交分析報告 — 2026-03-13

> **分析範圍**：從 `91b38f7a7`（2026-03-10 前次分析基準）至 `f6e5b6758`（2026.3.13 release），共 **1,101 個非合併提交**，涉及 **2,185 個檔案**、新增 132,471 行、刪除 52,321 行。

---

## 目錄

1. [重大變更 (Breaking / Major Changes)](#1-重大變更)
2. [新功能 (New Features)](#2-新功能)
3. [安全強化 (Security)](#3-安全強化)
4. [google-gemini-cli-auth 插件深度分析](#4-google-gemini-cli-auth-插件深度分析)
5. [頻道改進 (Channels)](#5-頻道改進)
6. [Agent 與 Session](#6-agent-與-session)
7. [ACP (Agent Communication Protocol)](#7-acp)
8. [Gateway 與 CLI](#8-gateway-與-cli)
9. [Cron 與排程](#9-cron-與排程)
10. [瀏覽器工具 (Browser)](#10-瀏覽器工具)
11. [記憶與壓縮 (Memory / Compaction)](#11-記憶與壓縮)
12. [模型與提供者 (Models / Providers)](#12-模型與提供者)
13. [Plugin SDK 與擴充系統](#13-plugin-sdk-與擴充系統)
14. [平台相容性 (Platform)](#14-平台相容性)
15. [效能優化與重構](#15-效能優化與重構)
16. [測試與 CI](#16-測試與-ci)
17. [文件更新](#17-文件更新)
18. [摘要統計](#18-摘要統計)

---

## 1. 重大變更

> 升級前必讀，可能影響現有部署或外掛開發

### 工作區插件自動載入停用（安全修復）

```
BREAKING: Plugins: require explicit trust for workspace-discovered plugins (#44174)
```

**GHSA-99qw-6mr3-36qr** — 停用隱含的工作區插件自動載入，克隆的儲存庫不再能在未經明確信任決定的情況下執行工作區插件程式碼。這是一個重要的安全修復，防止惡意儲存庫透過 workspace plugin 自動執行程式碼。

**影響範圍**：使用工作區插件的使用者需要透過 `plugins.allow` 明確信任插件 ID。

---

### Provider Plugin 架構模組化

```
MAJOR: feat: modularize provider plugin architecture
```

Ollama、vLLM 和 SGLang 已遷移至 provider-plugin 架構，provider 擁有自己的 onboarding、discovery、model-picker setup 和 post-selection hooks。這使得核心 provider wiring 更加模組化。

**影響範圍**：自訂 provider 整合可能需要更新以適應新的 plugin 架構。

---

### Cron / Doctor 收緊

```
BREAKING: cron/doctor tightening — stricter validation and error handling
```

Cron 和 doctor 模組收緊驗證和錯誤處理邏輯，可能導致先前寬鬆接受的配置被拒絕。

---

## 2. 新功能

### 2.1 Dashboard v2（全面翻新）

Control UI dashboard 經過三個 slice 的完整翻新：

1. **Slice 1/3**（#41497）：聊天基礎架構模組（chat infrastructure modules）
2. **Slice 2/3**（#41500）：工具、主題、i18n 更新
3. **Slice 3/3**（#41503）：dashboard-v2 views 重構

新 dashboard 包含模組化的 overview、chat、config、agent 和 session 視圖，加上 command palette、行動版底部 tabs、豐富的聊天工具（slash commands、搜尋、匯出、釘選訊息等）。

**相關 PR**：#41497、#41500、#41503

---

### 2.2 Fast Mode（快速模式）

新增 session-level 快速模式切換：

- **OpenAI Fast Mode**（#d5bffcdea）：可透過 `/fast`、TUI、Control UI 和 ACP 切換，支援 per-model config defaults 和 OpenAI/Codex request shaping
- **Anthropic Fast Mode**（#35aafd7ca）：將共用 `/fast` toggle 和 `params.fastMode` 映射至 Anthropic API-key `service_tier` 請求，支援即時驗證

---

### 2.3 Chrome DevTools MCP Existing-Session 模式

```
feat(browser): add chrome MCP existing-session support
```

新增官方 Chrome DevTools MCP attach mode，支援已登入的 Chrome 瀏覽器 session，搭配 `chrome://inspect/#remote-debugging` 啟用文件和 Chrome 原生設定指南的直接反向連結。

同時新增 `profile="user"`（已登入的主機瀏覽器）和 `profile="chrome-relay"`（extension relay）內建設定。

---

### 2.4 瀏覽器批次操作（Browser Batched Actions）

```
feat(browser): add batched actions, selector targeting, and delayed clicks
```

瀏覽器自動化新增批次 actions、selector targeting 和 delayed clicks 支援，搭配 normalized batch dispatch。

---

### 2.5 Sessions Yield 工具

```
feat: add sessions_yield tool for cooperative turn-ending (#36537)
```

新增 `sessions_yield` 工具，讓 orchestrator 可以立即結束當前 turn、跳過排隊的 tool work，並攜帶隱藏的 follow-up payload 進入下一個 session turn。

**相關 PR**：#36537

---

### 2.6 Kubernetes 安裝指南

```
docs: add Kubernetes install guide, setup script, and manifests (#34492)
```

新增入門級 K8s 安裝路徑，包含 raw manifests、Kind 設定和部署文件。

**相關 PR**：#34492

---

### 2.7 iOS 改善

| 項目 | 說明 | PR |
|------|------|-----|
| 歡迎導覽頁 | 首次啟動新增 welcome pager | #45054 |
| Home Canvas 工具列 | 重新設計首頁工具列 | — |
| 本地 Beta 發布流程 | 新增 iOS beta release flow | #42991 |
| APNs Relay Gateway | 新增 iOS APNs relay gateway | #43369 |

---

### 2.8 Android 改善

| 項目 | 說明 | PR |
|------|------|-----|
| 聊天設定重新設計 | 分組的裝置和媒體設定 | #44894 |
| 語音 Tab 更新 | 新增 speaker label 和 status pill | — |
| 連線 Tab 重新設計 | 統一狀態卡片 | — |
| Onboarding 重新設計 | 全新的 onboarding flow UI | — |
| Google Code Scanner | 替換 legacy ZXing 掃描器 | #45021 |

---

### 2.9 Docker 時區覆寫

```
feat: add OPENCLAW_TZ env var for Docker timezone override (#34119)
```

新增 `OPENCLAW_TZ` 環境變數，讓 `docker-setup.sh` 可以將 gateway 和 CLI 容器固定至指定的 IANA 時區。

---

### 2.10 Slack 互動式回覆

```
feat: Slack Block Kit messages and interactive replies (#44592, #44607)
```

Slack 頻道新增 `channelData.slack.blocks` 支援 Block Kit 訊息，以及 opt-in 的互動式按鈕和選擇器回覆（需設定 `channels.slack.capabilities.interactiveReplies`）。

---

### 2.11 Mattermost replyToMode

```
feat(mattermost): add replyToMode support (off | first | all) (#29587)
```

Mattermost 新增 `replyToMode` 配置選項，可控制回覆的串接行為。

---

### 2.12 阿里巴巴 Model Studio（百煉 Coding Plan）

```
feat: integrate Alibaba Bailian Coding Plan into onboarding wizard
refactor: rename bailian to modelstudio and fix review issues
fix: wire modelstudio env discovery (#40634)
```

阿里巴巴雲 Model Studio（原內部代號 Bailian）作為新的 LLM 提供者整合至 OpenClaw，提供透過 DashScope API 存取通義千問等模型的能力。

**提供者 ID**：`modelstudio`（從 `bailian` 重新命名為官方英文名稱）

**API 端點**：
- 中國區：`https://coding.dashscope.aliyuncs.com/v1`
- 國際區：`https://coding-intl.dashscope.aliyuncs.com/v1`

**支援的模型目錄**：

| 模型 | Context Window | 最大 Tokens | 輸入類型 |
|------|---------------|-------------|---------|
| `qwen3.5-plus`（預設） | 1M | 65k | 文字 + 圖片 |
| `qwen3-max-2026-01-23` | 262k | — | 文字 |
| `qwen3-coder-next` | 262k | — | 文字 |
| `qwen3-coder-plus` | 1M | — | 文字 |
| `MiniMax-M2.5` | 1M | — | 文字 |
| `glm-5` / `glm-4.7` | 202k | — | 文字 |
| `kimi-k2.5` | 262k | — | 文字 + 圖片 |

**Onboarding 整合**：在引導精靈中新增 `--auth-choice modelstudio-api-key`（國際區）和 `--auth-choice modelstudio-api-key-cn`（中國區）選項，搭配自動 API Key 發現。

**修復**：
- 國際端點變體現在正確覆寫 baseUrl，不再靜默保留過期的中國區 URL
- `PROVIDER_ENV_VARS` 新增 modelstudio 條目，修復 `secret-input-mode=ref` 拋出例外的問題

---

### 2.13 OpenRouter Hunter/Healer Stealth 模型

```
OpenRouter: surface free Hunter and Healer stealth models for the next week (#43642)
```

OpenRouter 發布的臨時 alpha stealth 模型，在限定期間（約一週）內免費開放使用。這些是進階推理模型，僅透過 OpenRouter API 提供。

| 模型 | ID | 推理能力 | 輸入類型 | Context Window | Max Tokens |
|------|-----|---------|---------|---------------|------------|
| Hunter Alpha | `openrouter/hunter-alpha` | 推理啟用 | 文字 | 1M | 64k |
| Healer Alpha | `openrouter/healer-alpha` | 推理啟用 | 文字 + 圖片 | 256k | 64k |

> **注意**：這些是早期存取的 alpha 模型，可用性受限於 OpenRouter 的推廣期間。

---

### 2.14 Opencode Go 統一路由提供者

```
Providers: add Opencode Go support (#42313)
```

Opencode Go 是一個統一路由提供者，將請求轉發至多個相容的基礎模型（Kimi、GLM、MiniMax），透過單一 API 介面存取。補充現有的 Opencode Zen 提供者。

**提供者 ID**：`opencode-go`
**預設模型**：`opencode-go/kimi-k2.5`

**模型別名**：

| 模型 ID | 顯示名稱 |
|---------|---------|
| `opencode-go/kimi-k2.5` | Kimi |
| `opencode-go/glm-5` | GLM |
| `opencode-go/minimax-m2.5` | MiniMax |

**Onboarding 整合**：`--auth-choice opencode-go` 搭配 `--opencode-go-api-key`。

---

### 2.15 Gemini Embedding 2 Preview

```
feat(memory): add gemini-embedding-2-preview support (#42501)
```

OpenClaw 記憶搜尋系統新增 Google Gemini Embedding 2 Preview 模型支援，用於語意搜尋和上下文管理的向量嵌入。

- **模型 ID**：`gemini-embedding-2-preview`
- **提供者**：Google（Gemini API）
- **用途**：文字嵌入，用於記憶語意搜尋和相似度比對
- **搭配修復**：Gemini embeddings 正規化（#43409），確保嵌入向量品質一致

---

### 2.16 多模態記憶索引

```
Memory: add multimodal image and audio indexing (#43460)
Memory: revalidate multimodal files before indexing
```

記憶系統新增多模態索引能力，支援圖片和音訊檔案的索引與搜尋。索引前會重新驗證多模態檔案的有效性。

---

### 2.17 其他新功能

| 功能 | 說明 | PR |
|------|------|-----|
| node-connect skill | 新增 node-connect 技能 | — |
| ACP resumeSessionId | Session resume 支援 | #41847 |
| Gateway runtime version | 狀態中暴露 runtime 版本 | — |
| Discord autoArchiveDuration | 新增 config 選項 | #35065 |
| Context Engine sessionKey | plumb sessionKey 至所有方法 | #44157 |
| macOS 聊天模型選擇器 | 新增 model selector + thinking 持久化 | #42314 |
| macOS 遠端 gateway auth | onboarding prompt for remote tokens | #43100 |
| Zalouser Markdown 解析 | Zalo 文字樣式解析 | #43324 |
| 壓縮狀態 reaction | 壓縮期間顯示狀態 reaction | #35474 |
| LLM-task thinking override | 新增 thinking 覆寫功能 | — |
| Windows update 套件覆寫 | 新增 update package spec override | — |

---

## 3. 安全強化

> 本次更新包含大量安全修復，多項涉及 GHSA（GitHub Security Advisory）

### 3.1 工作區插件隱含信任停用

**GHSA-99qw-6mr3-36qr** — 停用隱含的工作區插件自動載入。克隆的儲存庫不再能自動執行 workspace plugin 程式碼，需要明確的信任決定。

**影響等級**：高
**相關 PR**：#44174

---

### 3.2 裝置配對 Bootstrap Token 單次使用

```
fix(auth): make device bootstrap tokens single-use to prevent scope escalation
fix: switch pairing setup codes to bootstrap tokens
```

`/pair` 和 `openclaw qr` setup codes 切換為短期 bootstrap tokens，且為單次使用。待處理的裝置配對請求無法再被靜默重播並在批准前提升至 admin 權限。

---

### 3.3 iMessage 遠端附件路徑注入

```
fix(imessage): sanitize SCP remote path to prevent shell metacharacter injection
```

在生成 SCP 前拒絕不安全的遠端附件路徑，防止發送者控制的檔名將 shell metacharacters 注入遠端媒體暫存。

---

### 3.4 Telegram Webhook 驗證

```
fix(telegram): validate webhook secret before reading request body
```

在讀取或解析 request body 之前先驗證 Telegram webhook secret，未驗證的請求立即被拒絕，不再先消耗最多 1 MB 的資料。

---

### 3.5 外部內容標記消毒

```
fix: harden external content marker sanitization
fix(security): strip Mongolian selectors in exec obfuscation detector
```

在邊界消毒期間剝離零寬度和 soft-hyphen 標記分割字元，防止偽造的 `EXTERNAL_UNTRUSTED_CONTENT` 標記繞過標記正規化。同時在 exec 混淆偵測器中剝離蒙古語選擇器。

---

### 3.6 Exec Approvals 全面強化

本次更新包含 **8+ 項** exec approval 安全修復：

| 修復項目 | 說明 |
|---------|------|
| pnpm runtime 解包 | 解包更多 `pnpm` runtime 形式（`pnpm --reporter ... exec`、`pnpm node`） |
| Perl `-M`/`-I` 封鎖 | preload/load-path module resolution 預設 fail closed |
| PowerShell `-File`/`-f` | 辨識檔案型 PowerShell 啟動 |
| `env` dispatch 解包 | macOS 上解包 `env FOO=bar /path/to/bin` |
| Shell line continuation | 反斜線換行作為 shell line continuation 處理 |
| Skill trust 綁定 | 綁定至可執行名稱和解析路徑 |
| 解譯器 approvals | 未綁定的解譯器 approvals fail closed |
| system.run argv 綁定 | 綁定至精確的 argv 文字 |

---

### 3.7 Gateway / Auth 安全

| 修復項目 | 說明 | PR |
|---------|------|-----|
| SecretRef 遍歷拒絕 | 拒絕跨 schema/runtime/gateway 的 exec SecretRef 遍歷 ID | #42370 |
| 本地 auth SecretRef fail-closed | 未解析的本地 gateway auth refs fail-closed | #42672 |
| 配置寫入目標帳號 | 強制目標帳號 configWrites | — |
| 裝置 token scope | 裝置 tokens 限制至已批准的 scopes | #43686 |
| 子代理控制邊界 | 限制 leaf subagent 控制範圍 | — |
| WebSocket origin 驗證 | 強制瀏覽器 origin 檢查（不受 proxy headers 影響） | — |
| 分段寫入釘選 | pin staged writes 和 fs mutations | — |
| GIT_EXEC_PATH 封鎖 | host env sanitizer 封鎖 | #43685 |

---

### 3.8 Feishu Webhook 安全

| 修復項目 | 說明 | PR |
|---------|------|-----|
| 加密金鑰要求 | 要求 Feishu webhook encrypt key | #44087 |
| Reaction chat type 保護 | 保護 Feishu reaction chat type | #44088 |

---

### 3.9 其他安全修復

| 修復項目 | 說明 | PR |
|---------|------|-----|
| Hook agent routing | 審計不受限制的 hook agent routing | — |
| Nodes owner-only 工具 | 強化 owner-only tool gating | — |
| Invisible chars 轉義 | 轉義不可見的 exec approval 格式字元 | #43687 |
| Docker .env 排除 | 從 Docker build context 排除 .env | #44956 |
| SecretRef 強化 | 強化 custom/provider secret 持久化和重用 | #42554 |
| pnpm prod audit | 清除 pnpm prod 審計漏洞 | — |
| Session status 可見性 | 強制沙箱化的 session_status 可見性 | #43754 |
| Config write target | 統一 config write target policy | — |
| 洩漏的模型控制 tokens | 從使用者可見文字中剝離洩漏的模型控制 tokens | #42173 |
| Anthropic 啟動 crash | 避免 Anthropic 啟動 crash | #45520 |

---

## 4. google-gemini-cli-auth 插件深度分析

> 使用者詢問：此插件在 2026.3.13 是否被移除，是否涉及資安問題

### 4.1 現狀：插件仍然存在

**結論：google-gemini-cli-auth 插件在 2026.3.13 中並未被移除。**

插件仍然位於 `extensions/google-gemini-cli-auth/`，版本已同步至 `2026.3.13`。在 `91b38f7a7` 到 `f6e5b6758` 的提交範圍中，此插件僅有以下變更：

- **版本同步**：`2026.3.9` → `2026.3.13`（跟隨主專案版本）
- **測試重構**：提取共用的 `expectFakeCliCredentials()` 和 `runRemoteLoginExpectingProjectId()` helper 函式，減少測試重複

### 4.2 插件功能

此插件是 Google Gemini CLI 的 OAuth 提供者插件：

- 實作 PKCE OAuth 流程，搭配 localhost callback
- 從本地安裝的 Gemini CLI 提取 OAuth client credentials
- 處理本地瀏覽器型和遠端/WSL2 手動 OAuth 流程
- 發現並佈建 Google Cloud 專案以供 Gemini CLI 存取
- 儲存和管理 OAuth tokens（access token、refresh token、expiration time）

### 4.3 安全相關分析

#### 認證提取機制

`oauth.ts` 中的 `extractGeminiCliCredentials()` 函式：

1. 在系統 PATH 中定位已安裝的 `gemini` CLI
2. 搜尋 Gemini CLI 套件中的 `oauth2.js` 檔案（多個路徑候選）
3. 使用正規表達式提取 OAuth client ID（`*.apps.googleusercontent.com`）和 client secret（`GOCSPX-*`）
4. 快取提取結果避免重複操作

#### 提取的認證類型

```typescript
type GeminiCliOAuthCredentials = {
  access: string;        // OAuth access token
  refresh: string;       // OAuth refresh token
  expires: number;       // 過期時間戳
  email?: string;        // Google OAuth 使用者 email
  projectId: string;     // Google Cloud 專案 ID
};
```

#### 安全風險評估

| 面向 | 評估 |
|------|------|
| 認證提取 | **設計意圖內**：插件目的即為重用 Gemini CLI 的 OAuth credentials |
| 範圍 | **僅限本地**：僅讀取 Gemini CLI 安裝目錄的本地檔案系統 |
| Token 管理 | **安全**：OAuth tokens 正確管理（refresh tokens 持久化、access tokens 過期） |
| 使用者警告 | **已標示**：README 明確警告帳號限制/停權風險 |
| 信任模型 | **in-process trusted code**：符合 OpenClaw 的插件信任模型（`SECURITY.md`） |

#### 帳號安全警告

插件 README 包含明確的帳號安全警告：

> - 此插件為非官方整合，未獲 Google 背書
> - 部分使用者回報使用第三方 Gemini CLI 和 Antigravity OAuth 客戶端後帳號受限或停權
> - 建議謹慎使用、審閱適用的 Google 條款，避免使用關鍵任務帳號

#### CHANGELOG 歷史紀錄

在先前版本（2026.3.8 之前）中，曾新增以下相關安全措施：

- **帳號風險警告**（#16683）：Gemini CLI OAuth 開始前新增明確的帳號風險警告和確認閘道
- **OAuth 專案發現**（#16684）：對齊 OAuth 專案發現 metadata 和端點 fallback 處理
- **npm shim 安裝佈局**（#27585）：解析 npm 全域 shim 安裝佈局以發現 Gemini CLI credentials

### 4.4 NPM 發布狀態

google-gemini-cli-auth **不在** npm 發布清單中。它是 disk-tree only（與 OpenClaw 套件綁定，但不單獨發布至 npm）。

### 4.5 結論

**google-gemini-cli-auth 在 2026.3.13 中未被移除**。插件仍然活躍維護，版本同步至最新。雖然其認證提取機制涉及安全敏感操作（從 Gemini CLI 提取 OAuth credentials），但這是插件的設計目的，且已透過帳號風險警告和 OpenClaw 信任模型進行適當管理。**未發現 GHSA 安全通報或任何移除/棄用計畫**。

---

## 5. 頻道改進

### 5.1 Telegram

| 改進項目 | 說明 | PR |
|---------|------|-----|
| 媒體下載 IPv4 fallback | 重試 SSRF-guarded 檔案下載，IPv6 故障主機支援 | #45327 |
| Webhook secret 驗證 | 讀取 body 前先驗證 secret | — |
| 媒體 URL 遮蔽 | 失敗下載時遮蔽 bot token | — |
| 回覆 progress typing | 擴展回覆 progress typing 範圍 | — |
| 回覆分塊串接 | 共用 chunk threading | — |
| Monitor startup helpers | 共用測試 helpers | — |
| Account helpers | 共用帳號 helpers | — |
| Draft stream helpers | 共用 draft stream helpers | — |
| Sticky fetch helpers | 共用 sticky fetch helpers | — |
| Polling 清理計時器 | 清除 polling cleanup timers | — |
| Polling restart hang | 避免 stall detection 後 polling restart 掛起 | — |
| 重複回覆預防 | 慢 LLM provider 時防止重複訊息 | #41932 |
| 長訊息分塊 | 分塊長 HTML 出站訊息 | #42240 |
| Retain 清除 | 暫存 final fallback 前清除 stale retain | #41763 |
| 首次預覽 fallback | 模糊的首次預覽發送 fallback | — |

---

### 5.2 Discord

| 改進項目 | 說明 | PR |
|---------|------|-----|
| Deploy rate limit 啟動 | 部署 rate limit 不阻塞啟動 | — |
| Guild allowlist 強化 | hydrated guild 物件缺失時使用 raw guild_id | — |
| autoArchiveDuration | 新增自動封存時長配置 | #35065 |
| Gateway startup 錯誤 | 視 metadata 擷取失敗為暫時性錯誤 | #44397 |
| Rate limit mock 對齊 | 測試與 carbon 對齊 | — |
| Reaction ingress | 強制 users/roles allowlist 在 reaction ingress | — |

---

### 5.3 Feishu / Lark

| 改進項目 | 說明 | PR |
|---------|------|-----|
| 事件級去重 | 新增早期事件級去重防止重複回覆 | #43762 |
| 非 ASCII 檔名 | 保留非 ASCII 檔名在檔案上傳中 | #33912、#34262 |
| Webhook 加密金鑰 | 要求 webhook encrypt key | #44087 |
| Reaction chat type | 保護 reaction chat type | #44088 |
| Startup mock 模組 | 共用測試 mock 模組 | — |

---

### 5.4 Slack

| 改進項目 | 說明 | PR |
|---------|------|-----|
| Block Kit 訊息 | 支援 `channelData.slack.blocks` | #44592 |
| 互動式回覆 | opt-in 按鈕和選擇器回覆 | #44607 |
| Probe 簡化 | auth.test() bot/team metadata 穩定 | #44775 |
| 文字截斷 | 共用 slack text truncation | — |

---

### 5.5 其他頻道

| 頻道 | 改進項目 | PR |
|------|---------|-----|
| Mattermost | replyToMode 支援（off/first/all） | #29587 |
| Mattermost | DM 媒體上傳修復（unprefixed user IDs） | #29925 |
| Mattermost | Markdown 格式保留和原生表格 | #18655 |
| Mattermost | 重複訊息修復（block streaming + threading） | #41362 |
| iMessage | SCP 遠端路徑注入防護 | — |
| iMessage | 控制指令旗標修復 | — |
| iMessage | 自聊天重複去重 | #38440 |
| BlueBubbles | 自聊天重複去重 | #38442 |
| MS Teams | 使用 General channel conversation ID 作為 team key | #41838 |
| Signal | 配置 schema 新增 groups 支援 | #27199 |
| WhatsApp | 直接出站發送前綴空白修剪 | #43539 |
| Zalouser | Markdown 至 Zalo 文字樣式解析 | #43324 |
| Zalouser | 可變群組 allowlist 審計 | — |
| LINE | Webhook gating helpers 共用 | — |

---

## 6. Agent 與 Session

### 6.1 Compaction 改善

| 項目 | 說明 | PR |
|------|------|-----|
| Token 計數比較 | 使用全 session pre-compaction totals 比較 | #28347 |
| 摘要語言延續 | 保持 safeguard 壓縮摘要語言連續性 | #10456 |
| 狀態 reaction | 壓縮期間顯示狀態 reaction | #35474 |
| 巢狀 cron 車道 | 隔離的 cron 觸發嵌入工作路由至巢狀車道 | — |
| Fast mode | 隔離 cron 運行啟用 fast mode | — |
| Image-only pruning | 修剪包含圖片的 tool results | #41789 |

---

### 6.2 Provider 相容性

#### Kimi / Moonshot 相容性修復

Moonshot API（Kimi）在 thinking 模式啟用時對 tool choice 有嚴格的相容性要求。本次更新修復了多項 payload 格式問題：

| 項目 | 說明 | PR |
|------|------|-----|
| Kimi Coding tools | 恢復以原生 Anthropic format 發送 `tool_use` blocks（不再降級為 XML/plain-text pseudo invocations） | #38669 |
| Kimi Coding User-Agent | 預設發送 `User-Agent: OpenClaw` header（訂閱模型需要特定識別，保留使用者覆寫能力） | #30099 |
| Kimi Coding baseUrl | 遵從使用者配置的明確 `baseUrl`，允許 CN 和 global 端點切換 | #36353 |
| Moonshot CN API | 遵從 `platform.moonshot.cn` 的明確 baseUrl，確保中國區 API Key 正確認證 | #33637 |
| Ollama Kimi Cloud | 將 Moonshot payload 相容性 wrapper 應用至 Ollama 雲端的 Kimi 模型（如 `kimi-k2.5:cloud`）。偵測邏輯：provider 為 `ollama` + model 以 `kimi-k` 開頭 + model 包含 `:cloud` | #41519 |

**Moonshot Thinking + Tool Choice 規則**：
- Thinking 啟用時，`tool_choice: "required"` 自動轉換為 `"auto"`
- 指定工具的 tool choice 會停用 thinking（轉為 `thinking.type: "disabled"`）
- 支援的 tool choice（thinking 啟用時）：`"auto"`、`"none"`

#### OpenAI Codex Spark 模型限制

`gpt-5.3-codex-spark` 是 OpenAI 的專用 Spark 模型變體，**僅限** OpenAI Codex OAuth provider 使用，不可透過直接 API 存取：

- **正確路徑**：`openai-codex/gpt-5.3-codex-spark`
- **被抑制的路徑**：`openai/gpt-5.3-codex-spark` 和 `azure-openai-responses/gpt-5.3-codex-spark` 會回傳清楚的錯誤訊息引導使用者
- **原因**：上游 OpenAI 在直接 API 路徑上拒絕此模型

#### Azure OpenAI 內容過濾器修復

Azure OpenAI 部署的 `/new` 和 `/reset` 指令觸發 HTTP 400 false positive：

- **問題**：`"Execute your Session Startup sequence now"` 被 Azure 內容過濾器標記為潛在有害
- **修復**：改為 `"Run your Session Startup sequence"`，語意相同但避免觸發過濾器
- **影響**：所有 Azure OpenAI 使用者的 `/new` 和 `/reset` 指令恢復正常（PR #43403）

#### Billing / Failover 錯誤分類

| 項目 | 說明 | PR |
|------|------|-----|
| Venice billing | 辨識 Venice HTTP 402 billing errors，觸發模型 fallback 而非硬失敗 | #43205 |
| Poe billing | 辨識 Poe HTTP 402 `"used up your points"` 訊息，觸發 fallback。偵測正則：`^\s*402\s+.*used up your points\b` | #42278 |
| OpenRouter billing | HTTP 422 分類為 format error，credits exhausted 分類為 billing | #43823 |
| Gemini MALFORMED_RESPONSE | 分類為可重試的 timeout（而非硬失敗） | #42292 |
| Gemini PDF URL | 防止 `/v1beta` 路徑重複 | #34369 |
| 重複 cooldown probes | 避免同一 provider 的重複 cooldown probes | #41711 |
| Billing 錯誤優先 | 在 context overflow 啟發式之前檢查 billing errors，防止錯誤分類 | #40409 |
| 有效回應保護 | 防止 false billing error 取代有效回應文字 | #40616 |
| Ollama 推理可見性 | 停止將 native `thinking`/`reasoning` 欄位洩漏至最終回覆文字 | #45330 |

**Billing 錯誤分類機制**：`classify402Message()` 區分硬性 billing 錯誤（credit/quota 耗盡 → 停止）和暫時性 402 錯誤（rate limit/spend limit 重設 → 可重試 fallback）

---

### 6.3 工具和執行

| 項目 | 說明 | PR |
|------|------|-----|
| Gated core tools | 區分 gated core tools 和 plugin-only unknown entries | — |
| Web tools profile | 恢復 web tools 至 coding profile | #43436 |
| Exec summaries | 保持 exec summaries inline | — |
| Tool call arg 修復 | 修復 Kimi 和 Anthropic-compatible 的 malformed tool args | #43824、#42835 |
| Nodes owner-only | 新增 nodes 至 owner-only tool policy fallbacks | — |

---

### 6.4 Session 管理

| 項目 | 說明 | PR |
|------|------|-----|
| Transcript 檔案 | chat.inject 時建立缺失的 transcript 檔案 | #36645 |
| 記憶 bootstrap | 載入單一根記憶檔案，偏好 MEMORY.md | #26054 |
| Config 驗證 | 接受 per-agent params overrides | #41171 |
| Web fetch 驗證 | 恢復 readability/firecrawl 設定驗證 | #42583 |
| Discovery 驗證 | 接受 wideArea.domain | #35615 |
| Stale model reuse | 修復 session reset 時的 stale runtime model | #41173 |
| Gateway session reset | 保留 lastAccountId/lastThreadId | #44773 |
| 模型控制 tokens | 從使用者可見文字剝離洩漏的 tokens | #42173 |

---

## 7. ACP

| 項目 | 說明 | PR |
|------|------|-----|
| resumeSessionId | sessions_spawn 支援 session resume | #41847 |
| ACPX bump | 綁定的 acpx 升至 0.1.16 | #41975 |
| 重啟 session 恢復 | rehydrate 重啟的 main ACP sessions | #43285 |
| Child ACP env | 剝離 child ACP processes 的 provider auth env | #42250 |
| runId 範圍化 | scope cancellation 和 event routing by runId | #41331 |
| implicit streamToParent | mode=run without thread 時隱含 streamToParent | #42404 |
| Event builder 提取 | 提取 acpx event builders | — |
| Context engine compact | guard compact() throw + fire hooks | #41361 |

---

## 8. Gateway 與 CLI

### 8.1 Gateway 改善

| 項目 | 說明 | PR |
|------|------|-----|
| 用戶端請求限制 | 限時並清理懸掛的 RPC calls | #45689 |
| 殘留 socket 清理 | 強制停止殘留的 gateway client sockets | — |
| Control UI auth | 恢復 operator-only device-auth bypass | #45512 |
| Status --require-rpc | 新增 RPC 探測強制失敗選項 | — |
| Scope-limited probe | 視為 degraded reachability | #45622 |
| Session reset routing | 保留路由資訊跨 reset | #44773 |
| Main-session routing | TUI/UI mode 保持在內部 surface | #43918 |
| 會話分離 | 分離 conversation reset 和 admin reset | — |
| Token fallback | 強化 token fallback/reconnect 行為 | #42507 |
| Config 驗證議題 | dashboard 顯示 config validation issues | #42664 |
| Before_tool_call hooks | HTTP tools 運行 before_tool_call | — |

---

### 8.2 Windows Gateway

| 項目 | 說明 | PR |
|------|------|-----|
| schtasks 限時 | 限制 schtasks 呼叫並 fallback | — |
| Startup-folder 停止 | 解析正確的 port 進行 fallback stop | — |
| Status env 重用 | 重用安裝的 service command env | — |
| Device identity 靜默 | 停止 loopback 上的 stale device signature 噪音 | — |
| windowsHide | 隱藏 detached spawn console windows | #44693 |
| ASCII-safe logs | 保持 onboarding logs ASCII-safe | — |

---

### 8.3 CLI 改善

| 項目 | 說明 | PR |
|------|------|-----|
| Plugin 碰撞 | channel 和 binding 碰撞 fail fast | #45628 |
| Auth profile logging | 記錄 auth profile resolution 失敗 | #41271 |
| Env proxy bootstrap | 修復 model traffic 的 env proxy bootstrap | #43248 |
| Hooks context | 新增 trigger/channelId 至 hook contexts | #42362 |
| 安裝診斷 | 改善 onboarding install 診斷 | — |
| Plugin 發現 env | 隔離 plugin discovery env | — |
| 權限強化 | 強化 state dir permissions | — |
| Plugin 修復 | 安裝後修復 bundled plugin dirs | — |
| Plugin 更新 | 修復 plugin update dependency failures | — |
| macOS PortGuard | 防止殺死 remote mode 的 Docker Desktop | #13798 |

---

## 9. Cron 與排程

| 項目 | 說明 | PR |
|------|------|-----|
| 巢狀車道死鎖 | 修復 isolated cron nested lane deadlocks | #45459 |
| 手動 run type | 恢復 manual run type narrowing | — |
| 重複交付防止 | isolated direct cron sends 不進入重發佇列 | #40646 |
| Fast mode | 隔離 cron runs 啟用 fast mode | — |

---

## 10. 瀏覽器工具

| 項目 | 說明 | PR |
|------|------|-----|
| Chrome MCP | 新增 Chrome DevTools MCP existing-session 支援 | — |
| User profile 優先 | 偏好 user profile over chrome relay | — |
| Session 選擇 | 新增 browser session selection | — |
| Driver 驗證 | 強化 existing-session driver 驗證和 session lifecycle | #45682 |
| Batch actions | 新增批次 actions、selector targeting、delayed clicks | #45457 |
| Batch failure scoping | scope nested batch failures（+ revert） | — |
| 429 rate limit | 顯示 actionable hints | #40491 |
| Proxy attachment | 恢復 proxy attachment media size cap | #43684 |
| Existing-session control | 穩定 existing-session 控制 | — |
| Console formatting | 共用 console result formatting | — |
| Loopback auth | 共用 loopback auth error assertions | — |

---

## 11. 記憶與壓縮

| 項目 | 說明 | PR |
|------|------|-----|
| Gemini Embedding 2 | 新增 gemini-embedding-2-preview 支援 | #42501 |
| Gemini embeddings 正規化 | 正規化 Gemini embeddings | #43409 |
| 多模態索引 | 新增多模態圖片和音訊索引 | #43460 |
| 多模態重新驗證 | 索引前重新驗證多模態檔案 | — |
| Bootstrap 載入 | 偏好 MEMORY.md，fallback memory.md | #26054 |
| LanceDB temp config | 臨時配置修復 | — |
| 搜尋 config helpers | 共用測試 helpers | — |
| Tool builders | 共用 memory tool builders | — |

---

## 12. 模型與提供者

### 12.1 Anthropic Fast Mode

透過 Anthropic API 的 `service_tier` 參數實現優先處理推理：

- **Fast Mode ON**：`service_tier: "auto"`（優先推理佇列）
- **Fast Mode OFF**：`service_tier: "standard_only"`（標準佇列）
- **適用範圍**：僅限直接 API Key 模型（非 OAuth tokens），僅限公開 Anthropic API（`api.anthropic.com`），不適用於自訂 baseUrl
- **配置方式**：per-model 設定 `agents.defaults.models["anthropic/<model>"].params.fastMode`
- **切換方式**：`/fast` 指令、TUI、Control UI、ACP

---

### 12.2 OpenAI Fast Mode

為 OpenAI 模型（GPT-5.4、Codex）實現 session-level 快速模式：

- **參數映射**：
  - `reasoning.effort: "low"`（降低推理深度）
  - `text.verbosity: "low"`（降低輸出冗長度）
  - `service_tier: "priority"`（優先處理）
- **適用範圍**：OpenAI Responses API 模型（`openai-responses`、`openai-codex-responses`），僅限公開 OpenAI API
- **輸入正規化**：支援 `"on"`/`"off"`/`"true"`/`"false"`/`"yes"`/`"no"`/`"1"`/`"0"`/`"fast"`/`"normal"`
- **切換方式**：`/fast` 指令、TUI、Control UI、ACP

---

### 12.3 新模型/提供者總覽

| 模型/提供者 | 類型 | 關鍵特性 | PR |
|------------|------|---------|-----|
| ModelStudio（百煉） | Provider | Qwen 3.5+ 模型，CN/Global 端點，DashScope API | — |
| OpenRouter Hunter Alpha | Model | 推理啟用，1M context，限時免費 | #43642 |
| OpenRouter Healer Alpha | Model | 推理+圖片，256k context，限時免費 | #43642 |
| Opencode Go | Provider | 統一路由至 Kimi/GLM/MiniMax | #42313 |
| Gemini Embedding 2 Preview | Embedding | 記憶搜尋向量嵌入 | #42501 |

---

### 12.4 Provider Plugin 架構

```
feat: modularize provider plugin architecture
```

Ollama、vLLM、SGLang 遷移至 provider-plugin 架構：

- **Provider 自有 onboarding**：每個 provider 定義自己的引導流程
- **Discovery hooks**：provider 自行發現可用模型
- **Model-picker setup**：provider 控制模型選擇器的行為
- **Post-selection hooks**：模型選擇後的回呼（如設定特定 headers）
- **非互動式 setup**：支援 `--non-interactive` 模式
- **效果**：核心 provider wiring 更加模組化，新增 provider 不需修改核心程式碼

---

### 12.5 Provider 修復總覽

| 提供者 | 修復 | 詳情 | PR |
|--------|------|------|-----|
| Gemini/google-vertex | model-id 正規化 | `gemini-3.1-flash-lite` 正確解析為 `-preview` 變體 | #42435 |
| Kimi Coding | 原生 tool format | 恢復 Anthropic `tool_use` blocks（不再降級為 XML） | #38669 |
| Kimi Coding | User-Agent | 預設 `"OpenClaw"` header（訂閱認證需要） | #30099 |
| Kimi Coding | baseUrl 遵從 | 使用者配置的端點不被預設覆寫 | #36353 |
| Moonshot CN | baseUrl 認證 | `platform.moonshot.cn` API Keys 正確認證 | #33637 |
| Ollama/Kimi | Moonshot 相容性 | Thinking + tool_choice 衝突自動化解 | #41519 |
| Ollama | 推理可見性 | 不再將內部 thinking/reasoning 洩漏至回覆 | #45330 |
| OpenAI Codex Spark | 路徑限制 | 僅限 `openai-codex/` 路徑，直接 API 被抑制 | — |
| Azure OpenAI | 內容過濾器 | `"Execute...now"` → `"Run..."` 避免 false positive | #43403 |
| Venice | Billing 偵測 | HTTP 402 觸發 fallback | #43205 |
| Poe | Billing 偵測 | `"used up your points"` 觸發 fallback | #42278 |
| OpenRouter | 錯誤分類 | HTTP 422→format，credits→billing | #43823 |
| Gemini | 錯誤重試 | MALFORMED_RESPONSE 視為可重試 | #42292 |
| ModelStudio | env 發現 | 修復 `secret-input-mode=ref` 拋出例外 | #40634 |
| Custom provider | compat opt-in | 使用者明確的 `compat` 設定被遵從 | #44432 |
| Anthropic | 啟動穩定 | 避免啟動 crash | #45520 |
| OpenAI completions | transport | stale completions transport 正規化 | — |

---

## 13. Plugin SDK 與擴充系統

### 13.1 Plugin-SDK 記憶體修復

```
perf(build): deduplicate plugin-sdk chunks to fix ~2x memory regression (#45426)
```

修復 plugin-sdk subpath entries 在 build 時的重複 shared chunks 問題，解決近期的 plugin-sdk 記憶體膨脹問題。

---

### 13.2 Plugin 碰撞偵測

```
Plugins: fail fast on channel and binding collisions (#45628)
```

插件載入時快速偵測 channel 和 binding 碰撞，立即失敗而非靜默覆寫。

---

### 13.3 工作區插件安全

```
Plugins: require explicit trust for workspace-discovered plugins (#44174)
```

（詳見安全章節 3.1）

---

### 13.4 Plugin 架構文件

```
docs: explain plugin architecture
docs(plugins): clarify workspace shadowing
```

新增插件架構說明文件和工作區 shadowing 釐清。

---

## 14. 平台相容性

### 14.1 macOS

| 項目 | 說明 | PR |
|------|------|-----|
| Node.js 版本 | 對齊最低版本至 22.16.0 | #45640 |
| PortGuard Docker | 防止殺死 remote mode 的 Docker Desktop | #13798 |
| Exec approvals | 遵從 gateway prompter 中的 per-agent 設定 | #13707 |
| Voice wake crash | 修復 foreign transcript ranges 造成的 crash | — |
| 聊天模型選擇器 | 新增 model selector + thinking 持久化 | #42314 |
| 遠端 auth prompt | onboarding 提示 remote gateway auth tokens | #43100 |
| Launchd restart | 使用 kickstart -k 替代 bootout | — |
| Daemon onboarding | 避免 self-restart + 給新安裝更長啟動時間 | — |
| Shell continuation | 強化 backslash-newline 解析 | — |
| Env wrapper | 強化 env dispatch wrapper resolution | — |
| Skill trust | 綁定至可執行名稱和解析路徑 | — |
| Reminders 權限 | 新增 NSRemindersUsageDescription | — |

---

### 14.2 iOS

| 項目 | 說明 | PR |
|------|------|-----|
| 歡迎導覽頁 | 首次啟動 welcome pager | #45054 |
| Home Canvas | 全新首頁 canvas 和工具列 | — |
| Beta 發布流程 | 本地 beta release flow | #42991 |
| APNs Relay | iOS APNs relay gateway | #43369 |
| 配對指示 | 泛化配對指示 | — |
| 版本 xcconfig | ios-write-version-xcconfig 整合 | — |

---

### 14.3 Android

| 項目 | 說明 | PR |
|------|------|-----|
| 聊天設定重設計 | 分組的裝置和媒體設定 | #44894 |
| Onboarding 重設計 | 全新 onboarding flow UI | — |
| 連線 Tab | 統一狀態卡片 | — |
| 語音 Tab | Speaker label 和 status pill | — |
| Google Code Scanner | 替換 ZXing 掃描器 | #45021 |
| TLS 設定碼 | 預設 port 443 | — |
| Release bundle | 縮小 release bundle | — |
| Debug symbols | 上傳 native debug symbols | — |
| R8 strip | 移除未使用的 dnsjava resolver service | — |
| Auto-bump script | 新增自動版號 signed aab 腳本 | — |
| HttpURLConnection | 修復 HttpURLConnection leak | — |

---

### 14.4 Windows

| 項目 | 說明 | PR |
|------|------|-----|
| Gateway 限時 | schtasks 呼叫限時 + Startup-folder fallback | — |
| Gateway stop | 解析 fallback listeners 的正確 port | — |
| Gateway status | 重用 service command env | — |
| Gateway auth | 停止 loopback stale device signature 噪音 | — |
| windowsHide | 隱藏 detached spawn console windows | #44693 |
| ASCII-safe logs | onboarding logs ASCII-safe | — |
| Update restart | 修復 updater refresh cwd for service reinstall | #45452 |

---

### 14.5 Linux

| 項目 | 說明 | PR |
|------|------|-----|
| Parallels smoke harness | 新增 Linux smoke test harness | — |

---

### 14.6 Docker / Podman

| 項目 | 說明 | PR |
|------|------|-----|
| .env 排除 | 從 build context 排除 .env | #44956 |
| Timezone override | OPENCLAW_TZ env var | #34119 |
| Docker builds | 強化 builds + 解除 config docs 封鎖 | — |

---

## 15. 效能優化與重構

### 15.1 大規模重構

本次更新包含約 **434 個重構提交**，主要方向：

- **共用 helpers 提取**：exec host approval、extension channel setup、onboarding diagnostics、daemon lifecycle、gateway connection auth 等大量提取為共用模組
- **測試 helpers 共用**：403 個測試提交中大量提取共用 fixtures 和 helpers（memory search config、oauth profile、workspace skill、subagent announce timeout、provider discovery auth、model selection config、context lookup、sandbox fs bridge 等）
- **頻道重構**：telegram reply chunk threading、whatsapp outbound adapter、zalo status helpers、slack text truncation、feishu startup mocks 等共用化
- **Plugin-SDK 修復**：修復 subpath entries 重複 chunks 造成的 ~2x 記憶體回歸

---

### 15.2 效能改善

| 項目 | 說明 | PR |
|------|------|-----|
| Plugin-SDK chunks | 去重複化 shared chunks 修復記憶體膨脹 | #45426 |
| Allowlist matchers | 編譯 allowlist matchers | — |
| Terminal grapheme width | 精確量測 grapheme display width | — |
| Table wrapping | 改善 CLI 表格換行和寬度處理 | — |

---

## 16. 測試與 CI

### 16.1 CI 改善

| 項目 | 說明 |
|------|------|
| npm release workflow | 新增 npm release workflow 和 CalVer 檢查 |
| npm token fallback | 新增 npm token fallback |
| PR critical path | 修剪 PR critical path |
| Scoped workflow lanes | 加速 scoped workflow lanes |
| GitHub Actions 現代化 | 現代化 workflow versions |
| Node 24 runtime | opt-in 至 Node 24 action runtime |
| detect-secrets | 暫時移除 detect-secrets 檢查 |
| zizmor findings | 抑制 expected pull_request_target findings |
| Parallels harness | 新增 Windows 和 Linux smoke harness |

---

### 16.2 測試穩定性

本次共 **403 個測試提交**，主要類型：

- **tighten coverage**（~150 個）：大量 `tighten` 開頭的提交擴展邊界案例覆蓋（safe bin policy、duration formatter、transport ready、path guard、device identity、exec approval 等）
- **share helpers**（~200 個）：提取共用測試 helpers 和 fixtures 減少重複
- **expand coverage**（~50 個）：擴展 browser existing-session、gemini auth、apns 等覆蓋

---

## 17. 文件更新

| 文件主題 | 說明 |
|---------|------|
| Kubernetes 安裝 | K8s manifests、Kind 設定、部署指南 |
| Plugin 架構 | 插件架構說明和工作區 shadowing |
| Slack 互動回覆 | 互動式 Block Kit 回覆文件 |
| Ollama onboarding | onboarding 流程和指引更新 |
| Contributing | 要求 codex review |
| Android 狀態 | 標示 app 尚未公開發布 |
| Telegram allowlists | 釐清 group/sender allowlists |
| ACP session resume | 記錄 resumeSessionId |
| Voice-call allowlists | 釐清 caller ID 限制 |
| Parallels 測試 | macOS retest 和 snapshot 指引 |
| American English | 編碼慣例：美式英語拼寫 |
| Brave Search 成本 | 修復渲染 |
| Session key 修復 | `:dm:` → `:direct` |
| Changelog 管理 | 持續依影響排序和格式化 |

---

## 18. 摘要統計

| 指標 | 數值 |
|------|------|
| 總提交數 | **1,101** |
| 影響檔案數 | **2,185** |
| 新增行數 | 132,471 |
| 刪除行數 | 52,321 |
| 淨增行數 | +80,150 |
| 版本跨度 | `2026.3.9` → `2026.3.13` |
| 新功能 | **33+ 項** |
| 安全修復 | **20+ 項**（含 GHSA） |
| Bug 修復 | **80+ 項** |
| 頻道修復 | **40+ 項** |
| 重構提交 | **~434** |
| 測試提交 | **~403** |

### 按模組分布

| 模組 | 變更強度 |
|------|---------|
| 重構 / 共用 helpers | ■■■■■ 極高 |
| 測試 / 覆蓋率 | ■■■■■ 極高 |
| 安全強化 | ■■■■ 高 |
| Dashboard v2 | ■■■■ 高 |
| Exec Approvals | ■■■■ 高 |
| Gateway / CLI | ■■■■ 高 |
| Browser (Chrome MCP) | ■■■ 中高 |
| Telegram | ■■■ 中 |
| Models / Providers | ■■■ 中 |
| Agent / Compaction | ■■■ 中 |
| Android | ■■■ 中 |
| iOS | ■■■ 中 |
| macOS | ■■■ 中 |
| ACP | ■■ 中 |
| Feishu / Lark | ■■ 中 |
| Discord | ■■ 中 |
| Windows | ■■ 中 |
| Slack | ■■ 低 |
| Mattermost | ■■ 低 |
| Cron | ■ 低 |
| Memory | ■ 低 |
| Docker / Podman | ■ 低 |
| Plugin SDK | ■ 低 |

### 與前次分析（2026-03-10）比較

| 指標 | 2026-03-10 | 2026-03-13 | 變化 |
|------|-----------|-----------|------|
| 提交數 | 1,120 | 1,101 | 相當 |
| 檔案數 | 2,778 | 2,185 | -21% |
| 新增行 | 151,485 | 132,471 | -13% |
| 刪除行 | 30,799 | 52,321 | +70% |
| 新功能 | 30+ | 33+ | +10% |
| 安全修復 | 18+ | 20+ | +11% |
| 重構 | ~200+ | ~434 | +117% |

本次更新的突出特點是**安全修復的深度和廣度**（GHSA advisory、exec approval 全面強化、設備配對 token 單次使用等）以及**大規模重構和測試去重**（434+403 個重構/測試提交），顯示專案正在經歷一個系統性的程式碼品質提升階段。

---

*本文件由 AI 自動分析生成，分析基準點：`91b38f7a7` → `f6e5b6758`（2026-03-13）*
