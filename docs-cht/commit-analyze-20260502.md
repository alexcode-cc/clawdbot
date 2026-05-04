# OpenClaw 提交分析報告 — 2026-05-02

> **分析範圍**：以英文 **`CHANGELOG.md` → `## 2026.5.2`** 為**唯一權威產品行為主軸**（對齊 **`package.json`** 版本 **`2026.5.2`** 與 **`git tag v2026.5.2`**）。  
> **工作區快照**：撰寫時 **`HEAD`** = **`46deaf83b7296087da12a3b03381c603175e9392`**（**`git describe`**：`v2026.5.2-1-g46deaf83b7`），較 **`v2026.5.2`**（**`8b2a6e57fef6c582ec6d27b85150616f9e3a7ba4`**）多 **一筆** `docs-cht` 同步類提交時屬正常。  
> **上一輪繁體對齊**：release **`v2026.4.29`**／文件標 **`2026.4.29`**（見 **`commit-analyze-20260429.md`**）；該輪曾以 **`Unreleased`** 與 **`commit-analyze-20260427.md`** 為行為主軸。**本期**官方已發 **`## 2026.5.2`** 切段，**請以本檔 + changelog 該段為準**，60427／60429 僅作歷史分叉／對齊說明。  
> **Git 附註**：部分環境若 **`v2026.4.29`** 標籤所指提交與目前 **`HEAD`** **非直系祖先**（分叉／補標／本地分支差異），**請勿以「標籤間 raw commit 數」強解讀差分**；**以 changelog 切段與 tag `v2026.5.2` 為準**即可。  
> **方法**：與 **`commit-analyze-20260427.md`** 相同——**不按條列举上千筆提交**，改以 **Highlights → Changes → Fixes** **主題化繁中摘要**，細節與 PR／issue 編號以 **`CHANGELOG.md`** 原文為準。

---

## 目錄

1. [版本路徑與方法說明](#1-版本路徑與方法說明)
2. [2026.5.2 — Highlights](#2-202652--highlights)
3. [2026.5.2 — Changes（依主題）](#3-202652--changes依主題)
4. [2026.5.2 — Fixes（精選分組）](#4-202652--fixes精選分組)
5. [摘要統計](#5-摘要統計)

---

## 1. 版本路徑與方法說明

```
2026.4.29（上一輪 docs-cht／使用者自述對齊點）
    → 官方發行 CHANGELOG 切段 ## 2026.5.2（本期行為主軸）
2026-05-02（使用者對齊日期）≈ npm／CLI 版本 2026.5.2 與文件快照
```

**為何以官方 changelog 切段為主？**  
**2026.5.2** 的 **Changes／Fixes** 列項極長（數百條），且大量為互相牽動的 **效能、通道交會、插件供應鏈、Doctor／更新／Gateway 冷熱路徑**；逐 commit 人工對照成本高且易與分支拓扑混淆。**主題摘要 + changelog 原文**最能對齊**可感知行為與維運修復**。

---

## 2. 2026.5.2 — Highlights

- **外部插件生命周期（npm／ClawHub 割接）**：安裝、更新、**doctor 修復**、**依賴回報**、**artifact／digest 元數據**一條龍——涵蓋 **npm-first**、**過時已設定路徑**、**缺少套件 payload**、**beta 通道插件 fallback** 等營運情境（延續多個 PR／維護者貢獻）。  
- **Gateway／Agent 熱路徑精簡**：啟動、**session 列表**、**task 維護**、**prompt 準備**、**插件載入**、**工具描述子規劃**、**檔案系統護欄**、**大型 runtime 設定**等多點降載與避免阻塞。  
- **Control UI／WebChat 韌性**：Sessions、Cron、**長連線 WebSocket**、**群組訊息寬度**、**斜線指令回饋**、**iOS PWA 邊界**、**選取對比度**、**Talk 診斷**等 UI／連線體驗硬化。  
- **通道交付**：WhatsApp **Channel／Newsletter** 目標、Telegram **topic／指令／網路**、Discord **交付／啟動邊角**、Slack **thread**、Signal **群組／媒體**、**可見回覆路由**等多項修正與對齊。  
- **提供者／媒體**：OpenAI-compat **TTS／Realtime**、OpenRouter／DeepSeek **replay**、Anthropic-compat **串流**、LM Studio **reasoning 元數據**、Brave／SearXNG／Firecrawl **網搜**、**媒體路徑**、**音樂**、**語音通話路由**等。

---

## 3. 2026.5.2 — Changes（依主題）

### Gateway／啟動／服務

- 啟動 **secrets preflight** 可跳過 **插件背書 auth-profile overlay**，縮短 readiness；reload／OAuth 恢復仍保留 overlay 能力。  
- **`openclaw gateway restart`**：**`--force`**、**`--wait <duration>`**；重啟延遲前記錄 **作用中 task run id**；逾時重啟明確視為 **強制重啟**。  
- **`$include`**：允許從運營核准的 **`OPENCLAW_INCLUDE_ROOTS`** 讀檔，**預設**仍限制在設定目錄範圍。  
- **定價目錄**：可延後到 sidecar／通道 ready 後再拉；關閉時 **中止**進行中定價 fetch、避免 shutdown 後寫快取。

### 插件／ClawHub／CLI

- **診斷／onboarding／doctor／通道 setup**：安裝紀錄貫穿 **ClawPack** 元數據；**`clawhub:`** 顯式規格走 ClawHub，**裸 npm 包名**維持 npm 為 launch 預設。  
- **`openclaw plugins list --json`**：含 **套件依賴安裝狀態**（腳本可偵測缺依賴而不必 runtime 載入插件）。  
- **beta 更新通道**：預設線路對 npm／ClawHub 插件更新 **先試 `@beta`**，無 beta 再回落 **latest**。  
- **Runtime preload 收斂**：寬預載改為依 **有效插件 id**（設定、啟動規劃、通道、slots、auto-enable）驅動，**不**再匯入所有可發現插件。  
- **來源 checkout**：bundled 插件自 **`extensions/*` pnpm workspace** 樹載入，**套件安裝**仍用建置後 runtime 樹。  
- **`git:` 插件安裝**：ref checkout、commit 元數據、scanner／staging、**`plugins update`** 支援 git 來源紀錄。  
- **工具描述子**：平台級 **descriptor planner**（可見性、可用性檢查、executor 參照）；快取 **`api.registerTool`** 捕獲的描述子，**prompt 規劃**可跳過重複載入插件（執行仍載入 live tool）。  
- **重型栈外部化**：ACPX → **`@openclaw/acpx`**、診斷 OTEL → **`@openclaw/diagnostics-otel`**；並為大量通道／提供者／通用插件準備 **`2026.5.1-beta.x`** npm／ClawHub 發佈與 **dist 移出 core npm 包**。  
- **Crestodian**：ClawHub **搜尋** + **列出／搜尋／安裝／解除安裝**，含核准與稽核。

### Agents／runtime／工具合約

- **請求時刻**：重用 **啟動載入的插件 registry**（providers、tools、channel actions、web／capability／memory／migration helpers）；**memo** provider extra-params、**memo** transcript replay-policy（維持 transport hook／custom-env 語意）。  
- **`agents.defaults.skipOptionalBootstrapFiles`**：bootstrap 時可跳過指定**選用** workspace 檔，不必關閉整個 workspace setup。  
- **`heartbeat_respond`**：**結構化** heartbeat 工具輸出（取代僅仰賴字串 **`HEARTBEAT_OK`** 解析）。  
- **Codex**：預設動態工具 **native-first**（檔案／patch／exec／process 交給 Codex harness）；可見回覆未設定時預設改走 **`message` tool**。  
- **`tools.invoke` RPC**：SDK 面向、共用 HTTP 政策、型別化核准／拒絕結果與 helper（#74705 脈絡）。

### 執行緒／綁定／存取控制

- **`threadBindings.spawnSessions`**：**取代**分裂的 subagent／ACP thread-spawn 開關；**thread-bound spawn 預設開**；**`doctor --fix`** 遷移 legacy 鍵（#75943）。  
- **Discord**：可重用 **message-channel access groups**；**channel-audience DM 授權**可引用 **`accessGroup:<name>`**（#75813）。

### 通道亮點（Changes 節錄）

- **WhatsApp**：明確 **`@newsletter`** Channel／Newsletter **外送目標**與 session metadata（#13417 等）。  
- **BlueBubbles**：可選 **`replyContextApiFallback`**（API 拉回被回覆原文；預設關）；**SSRF** 三模式政策、併發合併 fetch、附件失敗升格為 runtime error、`sanitizeForLog` 強化 query／Authorization 脱敏（#71820，含 CWE-532 面向）。  
- **Slack**：`app_home_opened` **預設 App Home tab**；manifest 收錄該事件；**持久化 bot 參與 thread**，Gateway 重啟後可延續 threaded auto-reply（#11655）。  
- **Discord**：Gateway 重啟後 **buttons／selects／forms** 可沿用至過期（多步互動較不易斷）。

### Google Meet／Voice Call

- API 建房可設 **`accessType`**／**`entryPointAccess`**；**`googlemeet end-active-conference`**；**`googlemeet test-listen`** 與 **`test_listen`** action（transcribe 模式等待字幕／轉寫實際移動）（#74824；#72478）。  
- **Live caption health**（Chrome transcribe）：observer 狀態、計數、最後字幕、近期行進 doctor／status。  
- Twilio Meet：**預連線 DTMF**、realtime stream、問候交接等 **階段日誌**強化。

### 提供者／搜尋／模型目錄

- **xAI**：目錄新增 **Grok 4.3** 並為預設 chat 模型。  
- **OpenAI-compat TTS**：**`extraBody`／`extra_body`** 穿透（#39900）。  
- **Docs**：ChatGPT／Codex 訂閱路線應 **`openai/gpt-*`** + **`agentRuntime.id: "codex"`**；**`openai-codex/*`** 仍為 PI OAuth 路線。

### Control UI／用量／macOS app

- **Usage Mosaic**：UTC **15 分鐘 token buckets**；沿用於小時過濾（#74337）。  
- **macOS**：近期 session context 收入 **Context 子選單**，用量／成本維持根層（選單緊湊）。

### 文件／維運

- **`BodyForAgent`** vs **`Body`**（入站模型文字 vs legacy envelope）；Signal 涵蓋強化（#66198）。  
- **`openclaw proxy validate`**：部署前驗證 proxy 設定／連通性／允許目的地（#73438）。  
- **Crabbox**：執行前印出選定 binary **版本／支援 providers**；拒絕過舊缺 **`blacksmith-testbox`** 的二進位。

### 相依／上游套件（節錄）

- TypeBox、AWS SDK、Teams SDK、Marked、Pi、OpenAI、Codex、Zod、Matrix 等 **workspace／bundled／插件 pin 刷新**（見 changelog 原文版本號）。

---

## 4. 2026.5.2 — Fixes（精選分組）

> 下列為 **高信號分組**；**完整列表必須以 `CHANGELOG.md` → `## 2026.5.2` → `### Fixes` 為準**。

### OpenAI／Codex／GPT-5 與 UI beta

- 預設 GPT-5 API-key session：**SSE Responses** 為預設 transport（除非明確選 WebSocket），修復 **WebChat／Control UI beta「已連線但無模型事件」**類問題。  
- **Codex app-server**：managed binary 解析（bundled dist、`@openai/codex` bin）；容忍連線二次 close；npm 插件安裝路徑與 **`/codex`** 指令所有權等硬化。  
- **Status**：**`openai/gpt-*` + native Codex runtime** 顯示 **`openai-codex` OAuth profile**（#76197）。

### Gateway／sessions／transcripts／tasks

- **`sessions.list`**：大型 session store **輕量 checkpoint／索引重用**，維持輪詢可負擔。  
- **Session-store 寫入**：集中 **in-process writer**、借用已驗證 mutable cache，減少 **`sessions.json`** 鎖競爭與重讀／clone（#68554）。  
- **Transcript 鎖**：單一 **`session.writeLock.acquireTimeoutMs`** 政策；預設等待 **抬高至 60s**（#75894）。  
- **Restart recovery**：topic suffix transcript **鎖檔清理**與 canonical fallback 對齊（#76052）。  
- **Terminal lifecycle**：completed／timeout 後不因 stale snapshot **卡在 running**。  
- **Task registry**：維護時避免 **全量 snapshot／clone**（#73517、#75708 等）。

### Control UI／WebChat／Cron／Talk

- **`gateway.controlUi.chatMessageMaxWidth`**（取代 patch CSS）；Cron **畸形列驗證**；Sessions **預設查詢有界**（#67935、#76050 等）。  
- **長連線**：dashboard WebSocket **協議 ping**；Stop 在重連後可用（恢復 abort 狀態）；iOS PWA **safe-area**；選取 **高對比**；斜線指令 **本地錯誤回饋**。  
- Talk：**Realtime WebRTC offer CSP**、瀏覽器 session **明確 VAD／轉寫輸入**、錯誤／生命周期事件上浮（#73427）。

### 插件／更新／Doctor／externalization

- **`auth` 命令根**不再誤判為 bundled plugin id；**beta** 外部插件維持對齊 **stable runtime alias**；偵測 **install record 目錄消失**並於 **`update`** 先修復；校驗 **web-search provider** 與 **靜態抑制 model／provider** 對有效插件集合。  
- **Doctor 2026.5.2**：一次性 **configured-plugin install repair**（`meta.lastTouchedVersion`）、過時 manifest **`channelConfigs`** 修補、啟用中下載插件經設定來源安裝、保留第三方 **`node_modules`**、標記 config touched。

### Discord／Telegram／Slack／Signal／WhatsApp

- Discord：**delivery／threads／PluralKit／components／voice／retry／429／CDN upload** 等多項交付與互動硬化（見 changelog 長列表）。  
- Telegram：**forum topic／commands／DNS／IPv4 sticky fallback／長文分段 HTML chunk** 等。  
- Slack：**DM／thread routing／directory CLI／typing／block kit／錯誤範圍**等。  
- Signal：**reply routing／BodyForAgent** 硬化（與 Changes 呼應）。  
- WhatsApp：**logout 經 live Gateway**、長連線 **`end(error)`**、quoted image **入站媒體**等。

### 提供者／推理／網搜／媒體

- OpenRouter／LM Studio／Anthropic：**DeepSeek V4 reasoning replay**、**Gemma reasoning**、**Anthropic stream delta 順序**（#76018、#76007）。  
- Google Gemini：**Flash-Lite minimal reasoning floor**、**3.1 Pro reasoning chunk idle**（#76071、#76080）。  
- 網搜：**Brave diagnostics／proxy baseUrl**、**SearXNG category retry**、**Firecrawl／Kimi／Gemini／DDG** 等多提供者 SSRF／freshness／abort 語意對齊（見 changelog）。

### 安全／infra／Windows／Docker

- 阻擋 workspace **`CLOUDSDK_PYTHON`**、`CLOUDSDK` 類覆寫（#75940）；**Homebrew** 環境污染 brew 解析（#74463）；Windows **`.env` PATH** 與 **`taskkill.exe`** 來源驗證等。  
- Docker：**Bun digest pin**（#74356）；gateway image **python3** 恢復（#75041）。

### Heartbeat／Cron／Subagents／自動回覆

- Heartbeat：**exec 事件循環／cooldown** 中央化（#64016；避免過密 wake）；**active-hours timezone** 計時對齊（#75487）；Discord **不把 exec tail 當 System untrusted 通道訊息**（#66366）。  
- Cron：**implicit cron／heartbeat fallback** 不收 DM pairing-store（#62339）；**malformed job** 不比較拖垮 reload（#75886）；**Wake-now retry busy**（#75964）。  
- Subagents：**sessions_spawn expectsCompletionMessage**、`sessions_send` thread target 拒絕、duplicate reply 等（見 changelog）。

---

## 5. 摘要統計

| 項目 | 撰寫時量測／約定 |
|------|------------------|
| **產品行為權威來源** | **`CHANGELOG.md` → `## 2026.5.2`** |
| **`package.json` version** | **`2026.5.2`** |
| **`git tag`** | **`v2026.5.2`** → **`8b2a6e57fe`** |
| **`HEAD`（撰寫時）** | **`46deaf83b7`**（可能為 **`v2026.5.2` + 1 docs commit**） |
| **上一輪使用者對齊點（自述）** | **`v2026.4.29`**／文件 **`2026.4.29`** |
| **本期新建完整主題分析** | **本檔**（**60427／60429** 為 **Unreleased／對齊說明**歷史） |
| **`docs-cht` 版本標註（commit-analyze 除外）** | **`2026.5.2`** |

*若你的環境 `HEAD`／標籤拓扑不同，請以實際 **`git describe`**／**`CHANGELOG` 頂段標題**為準並自行修正本檔案表頭 SHA／統計表。*
