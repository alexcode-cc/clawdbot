# OpenClaw 提交分析報告 — 2026-03-23

> **分析範圍**：從 `f6e5b6758`（2026-03-13 前次分析基準）至 `0ed979a69d`（2026.3.23 release），共 **2,694 個非合併提交**，涉及約 **3,500+ 個檔案**、新增與刪除大量行數。

---

## 目錄

1. [重大變更 (Breaking / Major Changes)](#1-重大變更)
2. [新功能 (New Features)](#2-新功能)
3. [安全強化 (Security)](#3-安全強化)
4. [頻道改進 (Channels)](#4-頻道改進)
5. [Agent 與 Session](#5-agent-與-session)
6. [Context Engine 與 Compaction](#6-context-engine-與-compaction)
7. [Gateway 與 CLI](#7-gateway-與-cli)
8. [模型與提供者 (Models / Providers)](#8-模型與提供者)
9. [Plugin SDK 與擴充系統](#9-plugin-sdk-與擴充系統)
10. [Web Search 生態系重構](#10-web-search-生態系重構)
11. [媒體能力遷移 (Image / TTS / Media)](#11-媒體能力遷移)
12. [Outbound 與 Delivery Queue](#12-outbound-與-delivery-queue)
13. [平台相容性 (Platform)](#13-平台相容性)
14. [效能優化與重構](#14-效能優化與重構)
15. [測試與 CI](#15-測試與-ci)
16. [文件更新](#16-文件更新)
17. [摘要統計](#17-摘要統計)

---

## 1. 重大變更

> 升級前必讀，可能影響現有部署或外掛開發

### ClawHub 優先插件安裝（Breaking）

```
feat!: prefer clawhub plugin installs before npm
```

插件安裝流程的預設順序變更：**ClawHub（marketplace）安裝優先於 npm**。當使用者執行 `openclaw plugin install <id>` 時，系統會先嘗試從 ClawHub registry 取得插件 archive，僅在 ClawHub 找不到時才 fallback 至 npm。

**影響範圍**：
- 所有透過 CLI 安裝插件的使用者；先前直接指定 npm 套件名稱的腳本可能需要調整
- `openclaw.install.npmSpec` 仍可明確指定 npm 套件
- 文件和 docs 已更新為 ClawHub-first 預設描述

**相關提交**：`8d9686bd0f`、`91b2800241`（native clawhub install flows）

---

### 移除 moltbot 狀態目錄遷移 fallback（Breaking）

```
refactor!: remove moltbot state-dir migration fallback
```

移除從舊 `moltbot` 狀態目錄自動遷移至 `openclaw` 的 fallback 邏輯。先前使用 moltbot 名稱的安裝必須已完成遷移，否則狀態目錄不再被自動偵測。

**影響範圍**：極少數仍使用舊 moltbot 路徑的使用者

---

### 移除 CLAWDBOT 環境變數相容性（Breaking）

```
refactor!: drop legacy CLAWDBOT env compatibility
```

移除對舊 `CLAWDBOT_*` 環境變數前綴的相容性支援。所有環境變數現在必須使用 `OPENCLAW_*` 前綴。

**影響範圍**：使用舊 `CLAWDBOT_*` 環境變數的部署和腳本

---

## 2. 新功能

### 2.1 Plugin 能力系統全面重構

本次更新最重大的架構變更是 **Plugin Capability System** 的全面重構，將多項核心能力從 `src/` 遷移至插件架構：

#### Media Understanding 插件化

```
feat(plugins): add media understanding provider registration
feat(plugins): move media understanding into vendor plugins
```

媒體理解（圖片、音訊、影片分析）從核心模組遷移至插件能力系統。各 vendor（如 Google、OpenAI）透過 `mediaUnderstanding` capability 註冊其媒體分析能力。

#### Image Generation 插件化

```
feat(plugins): add image generation capability
feat(image-generation): add image_generate tool
feat(google): add image generation provider
```

圖片生成成為正式的 Plugin Capability：
- 新增 `image_generate` 工具
- Google 和 fal.ai 作為首批 image generation providers
- Provider 透過 `imageGeneration` capability 在 plugin manifest 中註冊

**fal Provider**（#49454）：新增 fal.ai 圖片生成提供者。

#### Speech / TTS 插件化

```
feat(plugins): add speech provider registration
feat(plugins): expand speech runtime ownership
feat(tts): add microsoft voice listing
feat(tts): enrich speech voice metadata
feat(tts): add in-memory speech synthesis
```

語音合成（TTS）從核心遷移至插件：
- Deepgram 和 Groq speech providers 移入插件
- Microsoft 語音列表支援
- 記憶體內語音合成支援
- `speech` capability 標準化

#### Web Search 插件化

```
feat(plugins): add web search runtime capability
feat(plugins): derive bundled web search providers from plugins
```

Web Search 成為標準 Plugin Capability，詳見 [第 10 節](#10-web-search-生態系重構)。

---

### 2.2 新 Web Search 插件

#### DuckDuckGo 搜尋插件

```
feat(web-search): add DuckDuckGo bundled plugin (#52629)
```

新增 DuckDuckGo 作為 bundled web search plugin，標記為 **experimental**。免費使用，不需 API key。

#### Exa 搜尋插件

```
feat(web-search): add bundled Exa plugin (#52617)
```

新增 Exa（語意搜尋引擎）作為 bundled web search plugin，支援 search 和 extract 功能。

#### Tavily 搜尋插件

```
feat: add Tavily as a bundled web search plugin with search and extract tools (#49200)
```

新增 Tavily 作為 bundled web search plugin，提供 search 和 extract 雙工具。

---

### 2.3 Anthropic Vertex Provider

```
feat: add anthropic-vertex provider for Claude via GCP Vertex AI (#43356)
```

新增 `anthropic-vertex` 提供者，可透過 Google Cloud Vertex AI 存取 Claude 模型。這使得使用 GCP 基礎設施的企業可以透過 Vertex AI 的計費和治理框架使用 Claude。

**提供者 ID**：`anthropic-vertex`

---

### 2.4 xAI Provider 整合

```
feat: finish xai provider integration
feat(xai): support fast mode
```

完成 xAI（Grok）提供者的完整整合，包含 fast mode 支援。

---

### 2.5 GitHub Copilot 動態模型解析

```
feat(github-copilot): resolve any model ID dynamically (#51325)
```

GitHub Copilot 提供者現在可以動態解析任何 model ID，不再受限於預定義的模型目錄。

---

### 2.6 ClawHub 插件市場

```
feat: add native clawhub install flows
feat!: prefer clawhub plugin installs before npm
feat: add slash plugin installs
```

新增 ClawHub 插件市場的原生安裝流程：
- **ClawHub 優先**：插件安裝預設從 ClawHub 取得
- **Slash 安裝**：支援 `/install <plugin>` 快捷安裝
- **Auth 支援**：ClawHub auth token 解析和 skill browsing

---

### 2.7 Context Engine 重大改進

```
feat: add context engine transcript maintenance (#51191)
feat(context-engine): pass incoming prompt to assemble (#50848)
feat: pass modelId to context engine assemble() (#47437)
feat(compaction): truncate session JSONL after compaction (#41021)
feat: notify user when context compaction starts and completes (#38805)
```

Context Engine 獲得多項重大改進：
- **Transcript 維護**：新增自動 transcript 維護機制
- **Prompt 傳遞**：`assemble()` 現在接收 incoming prompt，改善上下文組裝品質
- **Model ID 傳遞**：`assemble()` 接收 modelId，支援 per-model 上下文策略
- **JSONL 截斷**：compaction 後截斷 session JSONL，防止無限制增長
- **使用者通知**：compaction 開始和完成時通知使用者

---

### 2.8 Nostr 頻道

```
feat: add nostr setup and unify channel setup discovery
```

新增 Nostr 協議頻道支援，包含設定精靈和統一的頻道設定發現機制。

---

### 2.9 OpenShell 沙箱

```
feat: add openshell sandbox backend
feat: add remote openshell sandbox mode
feat: move ssh sandboxing into core
```

新增 OpenShell 遠端沙箱後端，支援透過 SSH 的遠端程式碼執行沙箱。SSH sandboxing 從外部遷入核心。

---

### 2.10 Synology Chat 設定精靈

```
feat: add synology chat setup wizard
```

為 Synology Chat 頻道新增引導式設定精靈。

---

### 2.11 ModelStudio DashScope 標準端點

```
feat(modelstudio): add standard (pay-as-you-go) DashScope endpoints for Qwen (#43878)
```

Model Studio（阿里百煉）新增標準（按量付費）DashScope 端點，區別於先前的 Coding Plan 端點。

---

### 2.12 Telegram 改善

```
feat(telegram): auto-rename DM topics on first message (#51502)
feat(telegram): support custom apiRoot for alternative API endpoints (#48842)
feat(telegram): add configurable silent error replies (#19776)
```

- **DM 主題自動重新命名**：首次訊息時自動重新命名 DM 主題
- **自訂 API 端點**：支援替代的 Telegram API 端點（如自架 Bot API Server）
- **可配置靜默錯誤回覆**：錯誤回覆可配置為靜默模式

---

### 2.13 其他新功能

| 功能 | 說明 | PR |
|------|------|-----|
| Chutes 擴充 | 新增 bundled Chutes extension | #49136 |
| Firecrawl Onboarding | 新增 firecrawl onboarding search plugin | — |
| Multi-session 刪除 | Control UI 支援多 session 選取和刪除 | #51924 |
| Usage 總覽改善 | 改善 usage overview 樣式和本地化 | #51951 |
| Memory pluggable prompt | 記憶插件可插入自訂 system prompt section | #40126 |
| Claude bundle commands | 原生註冊 claude bundle commands | — |
| Webchat 圖片持久化 | Gateway 將 webchat inbound 圖片持久化至磁碟 | #51324 |
| Gateway talk speak RPC | 新增 talk speak RPC | — |
| Skills compact fallback | Compaction 前透過 compact fallback 保留所有 skills | #47553 |
| Xiaomi MiMo V2 | 新增 MiMo V2 Pro 和 MiMo V2 Omni 模型 | #49214 |
| Mistral 模型目錄 | 新增 curated catalog models | — |
| MiniMax fast mode | 新增 fast mode 支援和 pi defaults 同步 | — |
| Android SMS search | 支援 Android node sms.search | #48299 |
| Android benchmark | 新增 benchmark script | — |
| Android Play builds | 隱藏受限能力在 Play builds 中 | — |
| Mattermost DM 重試 | 新增 DM channel creation 重試邏輯和逾時處理 | #42398 |
| Compaction delegate | 暴露 context-engine compaction delegate helper | #49061 |
| Image gen defaults | Agent 自動推斷 image generation defaults | — |
| Hook pack installs | 統一 hook pack 安裝至 plugins | — |
| Devcontainer SSHD | 新增帶 SSHD 的 devcontainer for Codespaces | — |

---

## 3. 安全強化

### 3.1 Gateway Auth 與 Canvas 安全

```
fix(gateway): require auth for canvas routes
fix(gateway): require admin for agent session reset
Fix Control UI operator.read scope handling (#53110)
```

- Canvas 路由現在要求身份驗證
- Agent session reset 需要 admin 權限
- 修復 Control UI operator.read scope 處理

---

### 3.2 Exec Approval 持續強化

```
Infra: tighten shell-wrapper positional-argv allowlist matching (#53133)
Infra: support shell carrier allow-always approvals
fix(security): unify dispatch wrapper approval hardening
fix(exec): escape invisible approval filler chars
Exec: harden host env override handling across gateway and node (#51207)
fix(security): harden exec approval boundaries
fix(exec): harden jq safe-bin policy
```

Exec approval 系統持續收緊：
- **Shell wrapper argv 匹配**：收緊 positional-argv allowlist 匹配規則
- **Shell carrier allow-always**：支援 shell carrier 的 allow-always approvals
- **Dispatch wrapper 統一強化**：統一 dispatch wrapper approval 邏輯
- **不可見字元轉義**：轉義 approval 格式中的不可見字元
- **Host env override**：強化 gateway 和 node 的 host env override 處理
- **jq safe-bin policy**：強化 jq 的安全二進位政策
- **Wrapper trust semantics**：重構 wrapper 解析模組，引入 trust semantics

---

### 3.3 媒體路徑安全

```
fix(media): block remote-host file URLs in loaders
fix(media): harden secondary local path seams
```

- 封鎖媒體 loader 中的遠端主機 file:// URL
- 強化次要本地路徑邊界

---

### 3.4 Plugin 遠端來源限制

```
fix: restrict remote marketplace plugin sources
```

限制遠端 marketplace 插件來源，防止未授權的插件來源。

---

### 3.5 Gateway 安全強化

```
fix(gateway): fail closed on unresolved discovery endpoints
fix(gateway): harden supervised lock and browser attach readiness
fix(gateway): gate internal command persistence mutations
fix(gateway): preserve async hook ingress provenance
fix(gateway): guard openrouter auto pricing recursion (#53055)
```

- Discovery endpoints 未解析時 fail closed
- 強化 supervised lock 和 browser attach 就緒檢查
- 限制內部指令的持久化變異
- 保留非同步 hook ingress 來源資訊
- 防止 OpenRouter auto pricing 遞迴

---

### 3.6 診斷資料安全

```
fix(diagnostics): redact credentials from cache-trace diagnostic output
```

從 cache-trace 診斷輸出中遮蔽認證資料。

---

### 3.7 Android 安全

```
fix(android): gate canvas bridge to trusted pages (#52722)
```

Android canvas bridge 限制為受信任的頁面。

---

### 3.8 Nostr 安全

```
fix(nostr): enforce inbound dm policy before decrypt
```

在解密前強制執行 Nostr inbound DM 政策。

---

### 3.9 Synology Chat 安全

```
fix: gate synology chat reply name matching
fix(synology-chat): fail closed shared webhook paths
```

Synology Chat 收緊 reply name matching 和 shared webhook path 處理。

---

### 3.10 SecOps 所有權

```
Security: add secops ownership for sensitive paths (#46440)
```

新增 CODEOWNERS 規則，為敏感路徑指定 secops 團隊所有權。

---

### 3.11 Auth 安全

```
fix(auth): prevent stale auth store reverts (#53211)
fix(openai-codex): bootstrap proxy on oauth refresh (openclaw#53078)
```

- 防止過期的 auth store 回退
- OpenAI Codex OAuth refresh 時正確 bootstrap proxy

---

## 4. 頻道改進

### 4.1 Telegram

| 改進項目 | 說明 | PR |
|---------|------|-----|
| DM 主題自動重新命名 | 首次訊息時重命名 DM 主題 | #51502 |
| 自訂 apiRoot | 支援替代 Telegram API 端點 | #48842 |
| 可配置靜默錯誤回覆 | 錯誤回覆可配置為靜默 | #19776 |
| 回覆上下文文字 | 保留 Telegram reply context text | #50500 |
| Topic announce 交付 | 恢復 Telegram topic announce delivery | #51688 |
| 訊息工具按鈕選填 | buttons schema 改為 optional | #52589 |
| asDocument 別名 | 記錄 Telegram asDocument 別名 | #52461 |
| Debounce 排序 | 修復 fire-and-forget debounce 排序 | — |
| Inbound debounce | 保留 inbound debounce 排序 | — |
| 媒體 loader 注入 | 透過 bot deps 注入 media loader | — |

---

### 4.2 Discord

| 改進項目 | 說明 | PR |
|---------|------|-----|
| 原生指令 auth 失敗 | 在 auth 失敗時回覆 | #53072 |
| 訊息工具發送 | 解除 Discord message tool 發送阻塞 | #52991 |
| 圖片交付 | 修復 generated image 交付至 Discord | #52489 |

---

### 4.3 Slack

| 改進項目 | 說明 | PR |
|---------|------|-----|
| 交付鏡像重複防止 | 防止 delivery-mirror 重複交付 | #45489 |
| Chunk 限制提升 | 提升 Slack chunk 限制 | #45489 |
| 訊息工具發送 | 解除 Slack message tool 發送阻塞 | #52991 |

---

### 4.4 Matrix

| 改進項目 | 說明 | PR |
|---------|------|-----|
| Runtime API exports | 避免重複 runtime api exports | — |
| Send aliases | 保留 send aliases 和 voice intent | — |
| Voice bubble | 新增 matrix 至 VOICE_BUBBLE_CHANNELS | #37080 |
| Room bindings | 避免觸碰 dropped room bindings | — |

---

### 4.5 WhatsApp

| 改進項目 | 說明 | PR |
|---------|------|-----|
| Outbound cycle | 移除 outbound runtime cycle | — |
| Extension checks | 穩定 extension checks | — |

---

### 4.6 Mattermost

| 改進項目 | 說明 | PR |
|---------|------|-----|
| DM 重試邏輯 | DM channel creation 重試和逾時處理 | #42398 |
| Typebox dep | 宣告 typebox runtime 依賴 | — |
| Lockfile 同步 | 同步 mattermost plugin lockfile | — |
| 按鈕選填 | 保持 message-tool buttons optional | #52589 |

---

### 4.7 Synology Chat

| 改進項目 | 說明 | PR |
|---------|------|-----|
| 設定精靈 | 新增 setup wizard | — |
| 多帳號 webhook | 釐清 multi-account webhook paths | — |
| 危險名稱匹配 | 集中化和審計 dangerous name matching | — |
| Reply name gating | 收緊 reply name matching | — |
| Fail closed webhooks | Shared webhook paths fail closed | — |
| Config schema 去重 | 去重 config schema | — |

---

### 4.8 Feishu / Lark

| 改進項目 | 說明 | PR |
|---------|------|-----|
| Config 範例 | 將 botName 替換為 name | #52753 |
| 媒體解除阻塞 | 解除 Feishu media 發送阻塞 | #52991 |

---

### 4.9 其他頻道

| 頻道 | 改進項目 | PR |
|------|---------|-----|
| Nostr | 新增頻道、setup wizard、inbound DM policy | — |
| Voice-Call | Plivo v2 replay keys 穩定、webhook pre-auth 強化、spoken-output 合約 | #51500 |
| LINE | 導出 line runtime subpath | — |

---

## 5. Agent 與 Session

### 5.1 Anthropic Thinking Block 排序

```
fix(agents): preserve anthropic thinking block order (#52961)
```

修復 Anthropic thinking blocks 的排序，確保推理過程按正確順序呈現。

---

### 5.2 Skill Secrets

```
fix(agents): prefer runtime snapshot for skill secrets
```

Skill secrets 現在優先使用 runtime snapshot，而非靜態配置。

---

### 5.3 ACP 改善

| 項目 | 說明 | PR |
|------|------|-----|
| Hidden thought replay | 保留 session load 時的 hidden thought replay | — |
| Hidden thought chunks | 保留 gateway chat 的 hidden thought chunks | — |
| ACPX 版本對齊 | 對齊 pinned runtime version | #52730 |
| Inline delivery | 恢復 run-mode spawns 的 inline delivery | #52426 |
| sessions_spawn 文件 | 釐清 ACP vs subagent policies | — |

---

### 5.4 Subagent 改善

```
fix(subagents): recheck timed-out announce waits (#53127)
```

Subagent announce 等待逾時後重新檢查，避免 false timeout。

---

### 5.5 Usage 改善

```
Usage: include reset and deleted session archives (#43215)
```

Usage 統計現在包含已重設和已刪除的 session archives。

---

### 5.6 Web Search Provider 選擇

```
Agents: fix runtime web_search provider selection (#53020)
```

修復 agent runtime 的 web_search provider 選擇邏輯。

---

## 6. Context Engine 與 Compaction

### 6.1 Transcript 維護

```
feat: add context engine transcript maintenance (#51191)
```

新增自動 transcript 維護機制，保持 session transcript 的健康狀態。

---

### 6.2 Compaction 改善

| 項目 | 說明 | PR |
|------|------|-----|
| JSONL 截斷 | Compaction 後截斷 session JSONL 防止無限增長 | #41021 |
| 使用者通知 | Compaction 開始和完成時通知使用者 | #38805 |
| Content-aware guard | 使 compaction guard 具備內容感知，防止 heartbeat sessions 的誤取消 | #42119 |
| Memory flush dedup | 使用 content hash 替代 compactionCount 進行記憶 flush 去重 | #30115 |
| Safeguard summary | 修復 compaction safeguard summary budget | #27727 |
| Transcript 指標 | 保持 session transcript 指標在 compaction 後保持最新 | #50688 |

---

### 6.3 assemble() 增強

```
feat(context-engine): pass incoming prompt to assemble (#50848)
feat: pass modelId to context engine assemble() (#47437)
feat: expose context-engine compaction delegate helper (#49061)
```

`assemble()` 方法現在接收更多上下文資訊（incoming prompt、modelId），使上下文組裝更加精確。

---

## 7. Gateway 與 CLI

### 7.1 Gateway 改善

| 項目 | 說明 | PR |
|------|------|-----|
| SIGTERM 關閉 | 強化 gateway SIGTERM 關閉行為 | #51242 |
| Probe auth | 完成 gateway probe auth landing | #52513 |
| Status 韌性 | Status helpers 在 netif 失敗時保持韌性 | — |
| Plugin context | 延遲解析 fallback plugin context | — |
| Discovery target | 集中化 discovery target 處理 | — |
| .env 變數 | 安裝時包含 .env 變數在 gateway service 環境中 | — |
| Install token | 提取 gateway install token helpers | — |

---

### 7.2 CLI 改善

| 項目 | 說明 | PR |
|------|------|-----|
| POSIX git dir | 保留 POSIX 預設 git dir | — |
| Control UI assets | 修復 packaged control ui asset lookup | #53081 |
| Deferred plugin logs | 在 status --json 時將 plugin logs 導向 stderr | — |
| Debounce 排序 | 保留 debounce 和 followup 排序 | #52998 |
| Debounce keys | 限制追蹤的 debounce key 數量 | — |
| Followup callbacks | 清除 idle followup callbacks 和 refresh drain callbacks | — |
| No-debounce 併發 | 保留 no-debounce inbound 併發 | — |

---

### 7.3 Doctor 改善

```
Doctor: prune stale plugin allowlist and entry refs (#53187)
Doctor: add bundle plugin capability summary to workspace status
doctor: clarify orphan transcript archive prompt
```

- 清理過期的 plugin allowlist 和 entry refs
- 在 workspace status 中新增 bundle plugin capability 摘要
- 釐清孤立 transcript archive 提示

---

### 7.4 Windows Gateway

```
fix: restart windows gateway after npm update
```

npm 更新後正確重啟 Windows gateway。

---

## 8. 模型與提供者

### 8.1 新增 / 更新的提供者

| 提供者 | 類型 | 說明 | PR |
|--------|------|------|-----|
| Anthropic Vertex | Provider | Claude via GCP Vertex AI | #43356 |
| xAI | Provider | Grok 完整整合 + fast mode | — |
| GitHub Copilot | Enhancement | 動態 model ID 解析 | #51325 |
| ModelStudio | Enhancement | 標準 DashScope 端點 | #43878 |
| Xiaomi MiMo V2 | Models | MiMo V2 Pro/Omni，切換至 OpenAI completions API | #49214 |
| Mistral | Models | Curated catalog models | — |
| MiniMax | Enhancement | Fast mode 支援 + pi defaults 同步 | — |
| DeepSeek | Refactor | 重構 bundled plugin | #48762 |

---

### 8.2 Fast Mode 擴展

```
feat(xai): support fast mode
feat(minimax): support fast mode and sync pi defaults
```

Fast mode 支援擴展至 xAI 和 MiniMax 提供者。

---

### 8.3 Provider 修復

| 提供者 | 修復 | PR |
|--------|------|-----|
| Mistral | 修復 max-token defaults 和 doctor migration | #53054 |
| MiniMax / OpenAI Codex | 確保 env proxy dispatcher 在 OAuth flows 之前 | #52228 |
| OpenRouter | 防止 auto pricing 遞迴 | #53055 |
| Moonshot / xAI | 集中化 moonshot compat 和 xai fast remaps | — |
| 通用 | 泛化 api_error 偵測以觸發 fallback model | #49611 |
| Provider runtime | 恢復 provider runtime lazy boundary | — |

---

### 8.4 Pi Provider Catalogs

```
feat(models): sync pi provider catalogs
feat(minimax): add missing pi catalog models
```

同步 Pi（嵌入式模型）provider catalogs，新增缺失的模型。

---

## 9. Plugin SDK 與擴充系統

### 9.1 Plugin Capability 系統

本次更新的核心主題是將更多能力遷移至 Plugin Capability 系統：

| 能力 | 狀態 | 說明 |
|------|------|------|
| Web Search | 已遷移 | Provider 和 runtime 完全由插件控制 |
| Image Generation | 已遷移 | Provider 註冊和工具由插件提供 |
| Speech / TTS | 已遷移 | Provider 移入插件，legacy core builders 移除 |
| Media Understanding | 已遷移 | Vendor providers 移入插件 |
| Media routing | 增強 | 圖片工具透過 media providers 路由 |

---

### 9.2 Plugin SDK 修復

| 項目 | 說明 | PR |
|------|------|-----|
| Hashed diagnostic events | 正規化和解析 hashed diagnostic event exports/chunks | — |
| Root alias fallback | 落回 src root alias 檔案 | — |
| Line runtime subpath | 導出 line runtime subpath | — |
| Stale surfaces | 修復 stale plugin sdk surfaces | — |
| Diagnostic subscriptions | Fast-path root diagnostic subscriptions | — |
| API baseline | 刷新 plugin-sdk api baseline | — |

---

### 9.3 Plugin 安裝與管理

| 項目 | 說明 | PR |
|------|------|-----|
| ClawHub 優先安裝 | 預設從 ClawHub 安裝（breaking） | — |
| Slash 安裝 | 支援 `/install <plugin>` 快捷安裝 | — |
| Clawhub uninstall | 接受 clawhub uninstall specs | — |
| Bundled plugins in pack | 在 npm pack artifacts 中包含 bundled plugins | — |
| Peer resolution | 跳過 bundled plugin deps 的 peer resolution | — |
| Keyed queue imports | 透過 core 路由 keyed queue imports | #52608 |
| Live hook registry | 在 gateway runs 期間保留 live hook registry | — |
| Stale allow entries | 忽略過期的 plugin allow entries | — |
| Built-in allowlists | 保持 built-in channels 不在 plugin allowlists 中 | #52964 |
| Auto-enable idempotent | 保持 built-in auto-enable 冪等 | — |

---

### 9.4 Outbound 插件化

```
Outbound: move target resolution heuristics behind plugins
Outbound: route sessions through channel plugins
Outbound: move target display fallbacks behind plugins
Outbound: preserve shared interactive payloads
Outbound: deliver shared interactive payloads
```

Outbound delivery 系統大量遷移至插件架構，target resolution 和 display fallbacks 現在由 channel plugins 控制。

---

## 10. Web Search 生態系重構

本次更新對 Web Search 系統進行了全面重構：

### 10.1 架構遷移

```
refactor web search provider execution out of core
refactor(web-search): share provider clients and config helpers
refactor(web-search): share scoped provider config plumbing
fix(web-search): split runtime provider resolution
```

Web Search provider 執行從核心移出，改由插件系統管理。Provider clients、config helpers 和 scoped config plumbing 共用化。

### 10.2 新增搜尋引擎

| 搜尋引擎 | 類型 | 需要 API Key | PR |
|---------|------|-------------|-----|
| DuckDuckGo | Bundled plugin | 否（免費） | #52629 |
| Exa | Bundled plugin | 是 | #52617 |
| Tavily | Bundled plugin | 是 | #49200 |
| Brave | Bundled plugin（預設啟用） | 是 | #52072 |

### 10.3 文件重組

```
docs(tools): restructure web search into nested group with provider sub-pages
docs(tools): add DuckDuckGo Search provider page
docs(tools): add Exa Search page
docs(nav): move Web Tools above Web Search group
```

Web Search 文件重新組織為巢狀結構，每個 provider 有獨立的子頁面。

---

## 11. 媒體能力遷移 (Image / TTS / Media)

### 11.1 Image Generation

```
feat(image-generation): add image_generate tool
feat(google): add image generation provider
Image generation: add fal provider (#49454)
Image generation: native provider migration and explicit capabilities (#49551)
refactor(image-generation): move provider builders into plugins
```

圖片生成成為完整的 plugin capability：
- `image_generate` 工具作為核心工具
- Google 和 fal.ai 作為 provider plugins
- Provider builders 遷入插件
- fal 加入 image-generation contract registry

### 11.2 TTS / Speech

```
feat(tts): add in-memory speech synthesis
feat(tts): enrich speech voice metadata
feat(tts): add microsoft voice listing
refactor(tts): move speech providers into plugins
refactor(tts): remove legacy core speech builders
refactor(media): move deepgram and groq providers into plugins
```

TTS 系統完整遷移至插件：
- Deepgram 和 Groq providers 移入插件
- 新增 Microsoft 語音列表
- 語音 metadata 豐富化
- 記憶體內語音合成
- Legacy core speech builders 移除

### 11.3 Media Understanding

```
feat(plugins): add media understanding provider registration
feat(plugins): move media understanding into vendor plugins
feat(plugins): tighten media runtime integration
```

媒體理解能力完整遷入插件系統。

---

## 12. Outbound 與 Delivery Queue

### 12.1 Delivery Queue 修復

```
fix(delivery-queue): increment retryCount on deadline-deferred entries
fix(delivery-queue): break immediately on deadline instead of failing all remaining entries
fix(delivery-queue): increment retryCount on deferred entries when time budget exceeded
```

Delivery queue 多項重要修復：
- Deadline deferred entries 的 retryCount 正確遞增
- 到達 deadline 時立即中斷而非 fail all remaining
- 時間預算耗盡時正確遞增 retryCount

### 12.2 Outbound 架構遷移

```
refactor(outbound): split delivery queue storage and recovery
Outbound: route sessions through channel plugins
Outbound: move target resolution heuristics behind plugins
Outbound: skip broadcast channel scan when channel is explicit
Outbound: remove channel-specific message action fallbacks
```

Outbound 系統大幅重構，delivery queue storage 和 recovery 分離，session routing 和 target resolution 遷至 channel plugins。

---

## 13. 平台相容性

### 13.1 macOS

| 項目 | 說明 | PR |
|------|------|-----|
| macOS publishing 自動化 | 自動化 macOS 發布流程 | #52853 |
| macOS publish privatize | 私有化 macOS publish flow | #53166 |
| Preflight artifacts | 上傳 macOS preflight artifacts | #53105 |
| CI fan out | 從 preflight scope fan out macOS | #52467 |

---

### 13.2 Android

| 項目 | 說明 | PR |
|------|------|-----|
| Canvas bridge 安全 | 限制 canvas bridge 至受信任頁面 | #52722 |
| Bitmap memory leaks | 修復 PhotosHandler、CanvasController、CameraCaptureManager 記憶體洩漏 | #41888、#41889、#41890 |
| Camera bitmap leak | 修復 camera bitmap leak | #41902 |
| Sensor race condition | 修復 MotionHandler sensor callback 競態 | #43781 |
| JS string escaping | 修復 A2UI action status 的 JS string escaping | #43784 |
| Permission dialogs | 強化 permission dialog 跨 activity 拆卸 | — |
| Contacts search | 修復 SQL LIKE wildcard escaping | #41891 |
| Current-location | 修復 cancellation-safe callback | #52318 |
| Status bar | 更新 OpenClawTheme 狀態列外觀 | #51098 |
| Gateway setup URLs | 使用 scheme 預設 port | #43540 |
| Gradle tooling | 更新 Gradle tooling | — |
| SMS search | 支援 node sms.search | #48299 |
| Play builds | 隱藏受限能力 | — |
| Benchmark script | 新增 benchmark script | — |

---

### 13.3 iOS

```
iOS: improve QR pairing flow (#51359)
```

改善 iOS QR 配對流程。

---

### 13.4 Windows

| 項目 | 說明 |
|------|------|
| Gateway restart | npm 更新後重啟 gateway |
| Windows media path | 媒體路徑防護 |
| Parallels smoke | 強化 Windows Parallels smoke installs |
| CI formatter spawn | 使用 Windows-safe formatter spawn |

---

## 14. 效能優化與重構

### 14.1 Reply 模組 Lazy-Load 優化

本次更新最顯著的效能改善是 **reply 模組的全面 lazy-load 優化**，共 **12+ 項** lazy-load 提交：

| 項目 | 說明 |
|------|------|
| Session store writes | Lazy-load session store 寫入 |
| Media path normalization | Lazy-load 媒體路徑正規化 |
| Context token lookup | Lazy-load context token 查詢 |
| Usage cost resolution | Lazy-load 使用成本解析 |
| Runner execution and memory | Lazy-load runner 執行和記憶 |
| Embedded queue steering | Lazy-load 嵌入式佇列轉向 |
| Runner auth profile seam | 分離 runner auth profile 邊界 |
| Payload dedupe helpers | 分離 payload 去重 helpers |
| Followup context lookup | Lazy-load followup 上下文查詢 |

**效果**：大幅降低 reply 模組的初始載入成本，改善 gateway 啟動和首次回覆時間。

---

### 14.2 Plugin Runtime Lazy-Load

```
perf: lazy-load plugin runtime heavy surfaces
perf: reduce plugin and memory startup overhead
perf: reduce memory startup overhead
perf: add lightweight memory status manager
perf: lazy-load memory runtime surfaces
```

Plugin runtime 和 memory 系統的重要 surfaces 改為 lazy-load，降低啟動開銷。

---

### 14.3 Vitest 效能優化

```
perf: trim vitest hot imports and refresh manifests
perf: enable vitest fs module cache by default
perf: default channel vitest lanes to threads
perf: trim vitest thread pins to hotspot tail
perf: add vitest test perf workflows
```

Vitest 測試框架效能改善：
- 修剪 hot imports
- 啟用 fs module cache
- Channel lanes 預設使用 threads
- 修剪 thread pins 至 hotspot tail
- 新增測試效能 workflows

---

### 14.4 大規模重構

本次更新包含約 **441 個重構提交**，主要方向：

- **Plugin capability 遷移**：Web search、image generation、TTS、media understanding 從 core 遷至插件
- **Outbound 架構**：delivery queue 拆分、session routing 插件化
- **Exec 模組**：wrapper resolution 拆分、trust semantics 引入
- **Synology Chat**：config schema 去重、name matching 集中化
- **Bootstrap profile**：集中化 bootstrap profile 處理
- **Gateway helpers**：install token、discovery target 提取
- **Session manager**：cache 轉為 factory 模式

---

### 14.5 Sandbox 優化

```
perf(core): narrow sandbox status imports for error helpers (#51897)
```

縮窄 sandbox status imports，降低 error helper 的載入開銷。

---

## 15. 測試與 CI

### 15.1 CI 改善

| 項目 | 說明 | PR |
|------|------|-----|
| CI shard | Bun test lane 分片 | — |
| Preflight workflows | 強化 preflight workflows | #53087 |
| Windows / Bun 穩定 | 穩定 Windows 和 Bun unit lanes | — |
| Formatter 序列化 | 在 CI 中序列化 formatter checks | #52428 |
| macOS CI fan out | 從 preflight scope fan out macOS | #52467 |
| npm release preflight | 修復 pnpm 下的 npm release preflight | #52985 |
| npm release preview | 移除 npm release preview workflow | — |
| Fast setup | 收合 fast setup jobs 至 preflight | — |

---

### 15.2 測試穩定性

本次共 **516 個測試/CI 提交**（含 432 個 test/ci 前綴），主要類型：

- **Thread-safe 隔離**（~100 個）：大量 `inject thread-safe` 提交，為 gateway、ACP、model、agent tools 注入 thread-safe seams
- **Module isolation**（~80 個）：`isolate`、`stabilize` 開頭的提交，確保 module isolation
- **No-isolate hardening**（~50 個）：強化無隔離模式下的 mock 重設和模組重設
- **Coverage 擴展**（~40 個）：擴展 web search、xai、firecrawl、tavily 等新功能的覆蓋

---

### 15.3 Release 改善

| 項目 | 說明 | PR |
|------|------|-----|
| Control UI tarball | 驗證 control-ui assets 包含在 npm tarball 中 | — |
| npm pack size | 提升 npm pack size budget | — |
| Channel surfaces | 保留 shipped channel surfaces 在 npm tar 中 | #52913 |
| macOS publishing | 自動化 macOS publishing | #52853 |
| macOS asset upload | 記錄手動 macOS asset upload | #53178 |

---

## 16. 文件更新

### 16.1 重大文件變更

| 文件主題 | 說明 |
|---------|------|
| Extensions → Plugins | 將 Extensions 重命名為 Plugins，重寫 building guide 為 capability-agnostic |
| Web Search 重組 | 重組為 nested group + provider sub-pages |
| CLI command tree | 修復 CLI command tree、SDK import path 和 tool group listing |
| Nav ordering | 修復 nav ordering、missing pages 和 stale model references |
| Config baseline | 刷新 generated config baseline |
| Plugin SDK baseline | 刷新 plugin-sdk api baseline |
| Sandbox docs | 整合 sandbox docs，新增 OpenShell 頁面 |
| Agents steering | 更新 steering semantics |
| Hooks | 修復 hook load order、command event payload 和 session-memory 範例 |
| Web Tools | 重組 Web Tools IA 和重寫 web.md |
| Capability docs | 去重複和交叉引用 plugin capability docs |

### 16.2 Provider / Channel 文件

| 文件主題 | 說明 |
|---------|------|
| DuckDuckGo Search | 新增 provider page |
| Exa Search | 新增 provider page |
| MiniMax M2.7 | 同步 references 和 examples |
| Feishu config | 將 botName 替換為 name |
| Synology Chat | 釐清 multi-account webhook paths |
| Telegram asDocument | 記錄 alias |
| WhatsApp media | 修復 media caps 文件 |
| Signal reaction | 修復 reaction mode 文件 |
| Memory loading | 修復 memory loading 文件 |

### 16.3 部署文件

| 文件主題 | 說明 |
|---------|------|
| Northflank / Railway | 移除 SETUP_PASSWORD 和 /setup wizard |
| Render | 修復 port env var，移除不存在的 setup wizard |
| Plugin install | 更新為 ClawHub-first default |
| Pi package versions | 更新至 0.61.1 |

---

## 17. 摘要統計

| 指標 | 數值 |
|------|------|
| 總提交數 | **2,694** |
| 影響檔案數 | **3,500+** |
| 版本跨度 | `2026.3.13` → `2026.3.23` |
| 新功能 | **60+ 項** |
| 安全修復 | **25+ 項** |
| Bug 修復 | **651** |
| 頻道修復 | **50+ 項** |
| 重構提交 | **~441** |
| 測試/CI 提交 | **~516** |
| 效能優化 | **~81** |
| 文件提交 | **~245** |

### 按模組分布

| 模組 | 變更強度 |
|------|---------|
| Plugin Capability 遷移 | ■■■■■ 極高 |
| 測試 / 覆蓋率 | ■■■■■ 極高 |
| Reply lazy-load 優化 | ■■■■ 高 |
| Web Search 重構 | ■■■■ 高 |
| Outbound 插件化 | ■■■■ 高 |
| Exec Approval 強化 | ■■■ 中高 |
| Context Engine | ■■■ 中高 |
| Gateway / CLI | ■■■ 中高 |
| Android 修復 | ■■■ 中 |
| Telegram | ■■■ 中 |
| Models / Providers | ■■■ 中 |
| macOS Release | ■■■ 中 |
| Synology Chat | ■■ 中 |
| ACP | ■■ 中 |
| TTS / Speech | ■■ 中 |
| Image Generation | ■■ 中 |
| ClawHub | ■■ 中 |
| Discord / Slack | ■■ 低中 |
| Matrix | ■■ 低 |
| Mattermost | ■ 低 |
| Delivery Queue | ■ 低 |
| Memory | ■ 低 |

### 與前次分析（2026-03-13）比較

| 指標 | 2026-03-13 | 2026-03-23 | 變化 |
|------|-----------|-----------|------|
| 提交數 | 1,101 | 2,694 | +145% |
| 新功能 | 33+ | 60+ | +82% |
| 安全修復 | 20+ | 25+ | +25% |
| Bug 修復 | 80+ | 651 | +714% |
| 重構 | ~434 | ~441 | +2% |
| 測試/CI | ~403 | ~516 | +28% |
| 效能優化 | — | ~81 | 新增 |

本次更新的突出特點：

1. **Plugin Capability 系統全面建立**：Web Search、Image Generation、TTS/Speech、Media Understanding 四大能力完整遷入插件系統，這是 OpenClaw 架構的重大里程碑
2. **Reply 模組極致 lazy-load**：12+ 項 lazy-load 優化大幅降低啟動和回覆延遲
3. **Outbound 架構插件化**：delivery、target resolution、session routing 遷至 channel plugins
4. **新搜尋引擎生態**：DuckDuckGo、Exa、Tavily 三個新 bundled web search plugins
5. **ClawHub 市場優先**：插件安裝預設從 ClawHub 取得，標誌著生態系統成熟
6. **Context Engine 進化**：transcript 維護、assemble() 增強、JSONL 截斷

---

*本文件由 AI 自動分析生成，分析基準點：`f6e5b6758` → `0ed979a69d`（2026-03-23）*
