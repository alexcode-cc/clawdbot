# OpenClaw 提交分析報告 — 2026-04-23

> **分析範圍**：從 `c15c72645d7b1a8ce3ea1390f4c4b90ce87f3fae`（doc: 更新繁體中文文件版本號至 2026.4.21）至整合樹 `7028b6aa1d`（Merge branch `2026-04-23` into develop），工作區歷史區間內共 **1,277 個非合併提交**。  
> **參考**：`docs-cht/commit-analyze-20260421.md`（2026.4.21 小 patch）；本輪以 **官方 CHANGELOG**（`2026.4.22`、`2026.4.23`）為主軸彙整使用者可感知變更；未逐條展開全部提交訊息。

---

## 目錄

1. [版本路徑與方法說明](#1-版本路徑與方法說明)
2. [2026.4.22 — 重大功能與提供者](#2-2026422--重大功能與提供者)
3. [2026.4.22 — Agent、Gateway、CLI 與設定](#3-2026422--agentgatewaycli-與設定)
4. [2026.4.22 — 頻道與修復精選](#4-2026422--頻道與修復精選)
5. [2026.4.23 — 變更總覽](#5-2026423--變更總覽)
6. [2026.4.23 — 安全與合規強化](#6-2026423--安全與合規強化)
7. [2026.4.23 — 影像、媒體理解與 Codex](#7-2026423--影像媒體理解與-codex)
8. [相依性與維運](#8-相依性與維運)
9. [摘要統計](#9-摘要統計)

---

## 1. 版本路徑與方法說明

```
2026.4.21  → 上一次繁體中文技術文件對齊終點（小 patch）
2026.4.22  → Stable：大版本（多提供者、TUI 內嵌、診斷輸出、大量修復）
2026.4.23  → Stable：Pi 0.70、gpt-5.5 目錄、影像/OAuth 路由、安全面再強化
```

**為何提交數多、文件卻以 CHANGELOG 為主？**  
1,277 個提交含大量 **測試／CI／backport／chore**，與使用者文件對齊成本高；與先前 `commit-analyze-20260420.md` 等大型報告相同，**以產品 changelog + 主題分類**最能反映「程式碼與英文專案文件」對應關係。

---

## 2. 2026.4.22 — 重大功能與提供者

### xAI（Grok）：影像、TTS、STT、Voice Call

- 影像：`grok-imagine-image` / `grok-imagine-image-pro`、參考圖編輯  
- TTS：六種 live 聲音；MP3／WAV／PCM／G.711  
- STT：`grok-stt`、Voice Call **即時轉寫**  
  
### STT 共用與 Voice Call

- Deepgram、ElevenLabs、Mistral 等 **Voice Call 串流轉寫**  
- ElevenLabs **Scribe v2** 批次轉寫（入站媒體）

### TUI：無 Gateway 內嵌模式

- 本機終端聊天可在 **無 Gateway** 下執行，仍保留插件核准等閘門  

### Onboarding

- 首次設定 **自動安裝** 缺失的 provider／channel 插件  

### OpenAI Responses + Web Search

- 啟用 web search 且未釘選受管搜尋提供者時，對 **直接 OpenAI Responses 模型** 自動使用 OpenAI 原生 `web_search`  

### 模型指令

- 新增 **`/models add <provider> <modelId>`**：從聊天註冊模型，無需重啟 Gateway  

### WhatsApp

- **`replyToMode`**：可設定原生回覆引用  
- **每群組／每私訊 `systemPrompt`** → 入站 `GroupSystemPrompt`；支援 `"*"`；帳戶級 `channels.whatsapp.accounts.<id>.{groups,direct}` **整表取代**根設定（與 `requireMention` 模式一致）

### Sessions

- **`sessions_list`**：信箱式篩選（label、agent、search）、衍生標題與最後訊息預覽  

### Control UI

- 瀏覽器端 **操作者個人身份**（名稱 + 本機安全頭像）  
- Quick Settings、後援 chip、窄螢幕聊天版面調整  

### Gateway 診斷

- 預設啟用 **無 payload 穩定性記錄**  
- **支援用診斷匯出**（日誌、狀態、健康、配置、穩定性快照，已脫敏）  

### Trajectory 匯出

- 預設本機軌跡捕獲；**`/export-trajectory`** 打包（脫敏 transcript、事件、提示、metadata、artifact）  

### Tencent Cloud 提供者

- TokenHub 引導、`hy3-preview` 目錄、分層 Hy3 定價 metadata  

### WeCom

- 官方 WeCom 頻道在 setup 中露出，顯示名稱與描述更新  

### Amazon Bedrock Mantle

- **Claude Opus 4.7** 經 Mantle Anthropic Messages 路徑；**bearer 串流**；不把 AWS bearer 當 Anthropic API key  

### GPT-5 Prompt Overlay

- 共享 provider runtime；`agents.defaults.promptOverlays.gpt5.personality` 全域友好風格；OpenAI 插件設定仍為後備  

### OpenAI Codex Onboarding

- **移除**從 onboarding／discovery **匯入 Codex CLI `~/.codex`** 至 agent auth store；改瀏覽器登入或裝置配對  

### CLI / Claude Code

- **`claude-cli` 預設暖 stdio session**；Gateway 重啟或閒置退出後從儲存 session 恢復  

### Pi 與 OpenCode Go

- Bundled Pi **0.68.1**；OpenCode Go 目錄由 pi 提供（含 `opencode-go/kimi-k2.6`、Qwen、GLM、MiMo、MiniMax 等）  

### Tokenjuice

- 可選插件：在 Pi embedded run 中 **壓縮吵雜的 exec／bash 工具輸出**  

### ACPX

- **`openClawToolsMcpBridge`**：為選定內建工具注入 OpenClaw MCP server（先從 `cron` 開始）  

### Doctor 插件路徑

- **懶載入** doctor 插件路徑；優先已安裝插件 `dist/*`，**doctor --non-interactive 量測執行時間約 -74%**  

### CLI 除錯計時

- 可選 **暫時性 CLI 計時探針**（stderr／JSONL），文件提醒落地前移除  

### 文件 i18n

- 文件站新增 **泰文**  

### OpenAI-Compatible 本地後端

- vLLM、SGLang、llama.cpp、LM Studio、LocalAI、Jan、TabbyAPI、text-generation-webui 等標為 **streaming-usage 相容**  
- **llama.cpp 風格** `timings.prompt_n` / `predicted_n` 回收用量並消毒累加  

### 插件啟動：Jiti

- 支援的 runtime 上 bundled 插件 dist **原生 Jiti 載入**，量測 **bundled 插件載入 -82%～90%**  

### Plugin SDK（STT／Pi）

- 即時轉寫 WebSocket、批次 multipart 表單 **跨 bundled STT 共用**  
- **Bundled-plugin embedded extension factory**：非同步 runtime hook（如 `tool_result`）  

### Codex Harness Hooks

- 與 Pi 對齊：`before_prompt_build`、`before_compaction`／`after_compaction`、`after_tool_call`、`before_message_write`、`llm_input`／`llm_output`／`agent_end` 等  

### QA / Status

- Telegram live QA **每場景回覆 RTT**  
- **`/status` 新增 `Runner:`**（embedded Pi、CLI 後端、ACP harness 等）  

---

## 3. 2026.4.22 — Agent、Gateway、CLI 與設定

**精選修復（CHANGELOG Fixes 節錄，非完整列表）：**

- **CLI／Gateway**：單次 RPC 等命令在程序退出前等待 WebSocket 拆卸，減少 `status`／`version` 掛起  
- **Thinking 預設**：無明確設定時，具推理能力模型由舊 `off`／`low` 隱式行為提升至 **安全 `medium` 等價**；`/status` 與 runtime 一致  
- **定價**：OpenRouter／LiteLLM 定價非同步拉取；catalog 逾時 **30s**  
- **Failover**：undici 傳輸失敗、pi-ai codex `Request failed` 等歸類 **timeout**，進入 fallback 鏈  
- **Anthropic Vertex**：輕量 discovery 後仍解析 ADC discovery 條目（與 2026.4.23 條目重疊敘述處以最新版為準）  
- **Plugins install**：安裝新插件時 **附加** 至既有 `plugins.allow`  
- **Azure OpenAI 影像**：偵測 Azure 端點、`api-key`、部署 URL、`AZURE_OPENAI_API_VERSION`  
- **Control UI ctx%**：message footer 納入 **cache read／write tokens**  
- **`models auth login`**：合併 provider 預設模型新增，避免覆寫 `agents.defaults.models`  
- **媒體理解 STT**：偏好已設定或金鑰型 API STT，避免本機 Whisper CLI 蓋過 Groq／OpenAI  
- **ACPX `probeAgent`**：可設定探針代理，避免預設 codex 未安裝時整後端標為不可用  
- **WhatsApp outbound**：記憶體內 **進行中寄送 claim**，避免重連 drain **重複 cron 寄送 7–12x**  
- **Matrix**：DM allowlist **不**再授權 room 控制命令  
- **SDK Retry-After**：長睡眠 cap，利於 OpenClaw failover  
- **Linux Gateway**：子程序 **oom_score_adj** shim（`OPENCLAW_CHILD_OOM_SCORE_ADJ=0` 可關）  
- **Moonshot／Kimi**：`functions.<name>:<index>` tool id **可選關閉 sanitize**（`sanitizeToolCallIds`）  
- **Config includes**：`plugins install`／`update` 對 **單檔 top-level include** 寫回，不扁平化 modular `$include`  
- **Config reload**：以 **來源撰寫 config** 規劃 reload，避免衍生路徑假陽性重啟  
- **Gateway clobber 恢復**：嚴重損毀簽名時恢復 last-known-good；修復 **非 JSON 前綴** 誤寫入  
- **MCP 生命週期**：stdio 樹拆卸、bundled MCP 在 session delete／reset 釋放；cron 退場路徑統一  
- **Cron／doctor**：修復持久化 job id 畸形列  
- **Slack**：Connect 串流退回一般回覆、HTTP Request URL 與 thread 綁定 ACP、檔案下載 token 解析  
- **Telegram**：webhook callback **5s**、polling **409 後重建 transport**  
- **Codex harness**：tool 描述不觸發 thread 重置、model catalog 與 websocket token 輪換、`serviceTier` 過濾  
- **大量其他**：Discord thread、Feishu、CLI Claude session 鍵、Pi 內建工具傳遞、Bedrock Mantle token 刷新等  

---

## 4. 2026.4.22 — 頻道與修復精選

- **WhatsApp**：群組＋私訊 systemPrompt、replyToMode、重複寄送修復、多帳戶政策集中（延續主線）  
- **Slack**：串流、檔案、HTTP webhook、執行緒 ACP 綁定  
- **Telegram**：webhook 逾時、polling 409  
- **Discord**：thread metadata 強化、反應與 `requireMention`、prefix channel 正規化邊界  
- **Matrix**：DM allowlist 與 room 命令分離  
- **Voice／Realtime**：OpenAI Realtime STT 強化；多提供者 STT（見 §2）  

---

## 5. 2026.4.23 — 變更總覽

### 影像與 OAuth／OpenRouter

- **Codex OAuth 下** `openai/gpt-image-2` **可不依賴 `OPENAI_API_KEY`**  
- **OpenRouter**：`image_generate` + `OPENROUTER_API_KEY` 支援影像與參考圖編輯  
- **`image_generate` 工具**：品質／輸出格式提示；OpenAI 專用 background、moderation、compression、user hints  
- **OpenAI 參考圖編輯**：**受控 multipart 上傳**（修復多參考 gpt-image-2）  
- **路由**：活躍 `openai-codex` profile 時 **`openai/gpt-image-2` 走 Codex OAuth**，不先探測 API key  
- **私網 SSRF opt-in**：OpenAI-compatible 影像端點、Gemini 影像生成 **可於信任代理／LAN 啟用**  

### Subagents

- **`sessions_spawn` 可選 forked context**：子執行可繼承請求者 transcript（預設仍隔離）  

### 生成工具逾時

- 影像、影片、音樂、TTS 工具支援 **每呼叫 `timeoutMs`**  

### Memory：本機嵌入

- **`memorySearch.local.contextSize`**（預設 **4096**）可調本機 embedding 上下文  

### Pi 與目錄

- Bundled Pi **0.70.0**；OpenAI／OpenAI Codex **gpt-5.5** 目錄來自 pi；**gpt-5.5-pro** 僅本機 forward-compat  

### Codex Harness 可觀測性

- Embedded harness **選擇決策**結構化 debug log（`/status` 保持簡潔）  

### 其他變更（節錄）

- **Codex `request_user_input`** 回到**來源聊天**；排隊後續答案；較新 app-server 核准修正  
- **Context-engine 組裝失敗 log 脫敏**  
- **WhatsApp onboarding**：首次 setup **不經 Baileys runtime dep 路徑**載入 setup entry（QuickStart 可先顯示設定）  
- **Block streaming**：部分區塊送達中止後，若已送區塊已覆蓋最終文，**抑制重複最終組裝文**  
- **Windows codex.cmd**：經 **PATHEXT** 解析 npm shim  
- **Slack 群組**：MPIM 視為群組語境；**非 DM 表面抑制冗長 tool／plan 進度**（避免「Working…」洩漏到房間）  
- **Replay**：OpenAI／Codex transcript **不再合成缺失 tool results**（Anthropic／Gemini／Bedrock transport-owned 仍保留合成修復）  
- **Telegram**：最終回覆路徑將遠端 markdown 圖片語法解析為 **出站媒體**  
- **WebChat embedded**：計費／認證／限速等 **不可重試** 錯誤對使用者可見  
- **WebChat 圖片**：純文字主模型時 **圖片卸載為 media ref** 供影像工具使用  
- **Google Meet 插件**：離開時掛斷 Twilio 委派通話、Chrome realtime 音訊橋清理、扁平 tool schema  
- **媒體理解**：尊重 `agents.defaults.imageModel`、`tools.media.image.models`、MiniMax VL 等 **顯式影像模型**  
- **Codex 媒體理解**：`codex/*` 影像經 **有界 app-server 影像 turn**；`openai-codex/*` 仍走 OAuth  
- **openai-codex/gpt-5.5**：目錄遺漏時 **合成 OAuth 模型列**  
- **undici**：embedded run **不再降低程序級 stream 逾時**（慢速 Gemini 影像等）  
- **Control UI**：助理生成圖 **持久化為受管媒體**；paired device token 可取圖；**Stop 在重連後仍取消進行中 run**  
- **Memory／QMD**：stale managed collection 修復時 **重建 collection**  
- **Voice Call**：OpenAI session 設定完成前不問候／不轉發緩衝音訊；**非 allowlist Twilio 主叫**在 stream 前拒絕  
- **ACPX／Codex**：不再物化 `auth.json` bridge；使用正常 `CODEX_HOME`／`~/.codex`  
- **Exec 事件**：非同步完成透過 **持久 session 傳遞 context** 回到來源頻道  
- **Webchat session 變更防護**：`sessions.compact`／`compaction.restore` 與既有 patch／delete 一致拒絕 `WEBCHAT_UI`  
- **QA 通道**：非 HTTP(S) 入站附件 URL **拒絕**  
- **外部插件 peer `openclaw`**：安裝後 **link 宿主套件** 解析 SDK  
- **Teams**：共享 Bot Framework audience token 須 **驗證 appid／azp** 對應設定 App  
- **Bun 全域安裝**：Jiti 載入相對 **目標插件模組** 解析，避免影像提供者 discovery 掛起  
- **Claude CLI `bypassPermissions`**：由 OpenClaw **YOLO exec policy** 衍生；保留手動 `--permission-mode`；畸形參數剝除  
- **Pairing**：cleartext 行動配對限 **私網 IP 或 loopback**；**.local** 與無點主機名不再視為安全 cleartext  
- **Dreaming cron**：與 heartbeat **解耦**，以 **輕量隔離 agent turn** 執行；doctor 遷移舊型 main-session dreaming job  
- **CLI `--agent` + `--session-id`**：查詢 **限於該 agent store**（#70985）  

---

## 6. 2026.4.23 — 安全與合規強化

| 主題 | 說明 |
|------|------|
| **Gateway `config.apply`／`config.patch`** | 對 agent 可改路徑改為 **白名單**（提示、模型、mention gating、Telegram topic `requireMention` 等），取代易漏的 denylist |
| **Webhook SecretRef** | **每請求重新解析**，`secrets.reload` 立即失效舊密鑰 |
| **Setup API** | 不再 fallback 啟動目錄，避免 workspace `extensions/.../setup-api.*` 被誤執行 |
| **Exec approval** | 需 **明確啟用** chat exec-approval；不因 approver／owner allowlist 自動開啟 |
| **Discord slash** | 原生 slash **channel policy** 不可繞過 owner／member 限制（無更嚴規則時仍 fallback） |
| **Android** | 手動／掃描路由 **僅 loopback 明文**；私網／link-local `ws://` 預設關閉（除非 TLS） |
| **Android ASK_OPENCLAW** | 外部 intent **只預填草稿**，不自動送出 |
| **Secrets（Windows）** | 檔案密鑰 **去 BOM**；ACL 不可用時 **fail-closed**（除非提供者明確 `allowInsecurePath`） |
| **影像工具警告** | 忽略之 override 值 **轉義**，防 `MEDIA:` 注入 |
| **QQBot `/bot-approve`** | 需 **framework auth** |
| **ACPX MCP 橋** | 不列出／不呼叫 **owner-only** 工具（如 `cron`） |
| **WhatsApp 結構化物件** | vCard／聯絡人／位置等 **不進 inline body**，以 **柵欄未信任 metadata JSON** 呈現 |
| **群組系統提示** | 頻道來源群名與參與者標籤 **不進 inline**，改柵欄 JSON |
| **Memory doctor** | 根 durable 記憶 **正規為 `MEMORY.md`**；不再把小寫 `memory.md` 當運行 fallback；`doctor --fix` 可合併 split-brain 並備份 |

---

## 7. 2026.4.23 — 影像、媒體理解與 Codex

- OpenRouter 多模態：**使用者文字在前、影像 part 在後**，修復空白 vision 回覆  
- 停止廣告已移除的 **`gpt-5.3-codex-spark`**；stale 列附 **GPT-5.5** 復原提示  
- Codex harness：**依 session 釘選** harness；`/status` 顯示 `codex` 等非 PI harness；換設定不 hot-switch 既有 transcript（`/new`／`/reset` 前維持 PI legacy）  

---

## 8. 相依性與維運

- **uuid** transitive override **14.0.0**（安全建議）  
- **發布線**：WhatsApp **setup 前 stage runtime deps**、channels **延遲 setup deps 至 login**、Gemini trust flag、models registry fallback、Docker／Parallels／release CI 多所硬化（細節見提交）

---

## 9. 摘要統計

| 項目 | 數值／說明 |
|------|------------|
| 非合併提交（自 2026.4.21 文件基準至整合點） | **1,277** |
| 對外穩定版標籤 | **2026.4.22**、**2026.4.23** |
| 文件對齊策略 | CHANGELOG 主軸 + 安全／提供者／頻道分塊 |
| 與 2026.4.21 關係 | 2026.4.21 為小 patch；本輪為 **跨兩個次版本** 的大量產品與安全更新 |

---

*本報告供 `docs-cht/` 技術文件版本對齊與維運查閱；細部 API／配置鍵請以英文官方文件與 repo 內 schema 為準。*
