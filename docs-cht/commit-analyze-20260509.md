# OpenClaw 提交分析報告 — 2026-05-09

> **分析範圍**：以上一輪繁體文件對齊點 **stable `v2026.5.7`（annotated tag 剝離後提交 `eeef4864494f859838fec1586bedbab1f8fa5702`）** 之後，至本機 **`HEAD`**（含 **`2026.5.9-beta.1`** 發行準備與 **`CHANGELOG.md` → `## 2026.5.9`**）所累積的變更。產品行為以英文 **`CHANGELOG.md` 頂段 `## 2026.5.9`**（含 **Breaking**／**Fixes**）為權威主軸；**不以逐筆羅列上千則提交**為目標，而以 **發行敘事 + 架構面歸因** 對照程式碼與文件。
> **版本對齊**：`package.json` 目前為 **`2026.5.9-beta.1`**；`git tag v2026.5.9-beta.1` 存在；`v2026.5.7^{commit}` 與前一輪文件記載之 **`eeef486449`** 一致（請以 `git rev-parse "v2026.5.7^{commit}"` 驗證）。
> **Git 規模說明**：自 **`eeef486449..HEAD`** 在典型 **`develop`** 合併拓撲下可達 **約 2,752** 筆非合併提交，**`git diff --shortstat`** 亦可能顯示 **六千餘檔／二十餘萬行** 量級——此為 **長分支累積 + 大型目錄合併** 的統計表象；**使用者可見行為**仍應以 **`CHANGELOG` 2026.5.9 條目** 與 **實際部署的 beta／stable tag** 收斂理解。

## 目錄

1. [版本切點與本輪性質](#1-版本切點與本輪性質)
2. [2026.5.9 — 重大功能與架構主題](#2-202659--重大功能與架構主題)
3. [Breaking：iMessage／BlueBubbles 遷移](#3-breakingimessagebluebubbles-遷移)
4. [Fixes 與穩定性（精選分群）](#4-fixes-與穩定性精選分群)
5. [受影響程式碼與文件面](#5-受影響程式碼與文件面)
6. [繁體技術文件同步策略（本輪）](#6-繁體技術文件同步策略本輪)
7. [本輪對齊統計](#7-本輪對齊統計)

---

## 1. 版本切點與本輪性質

```text
v2026.5.7（eeef486449…）
  → CHANGELOG ## 2026.5.9（大量 Changes + Breaking + Fixes）
  → package.json 2026.5.9-beta.1
  → tag v2026.5.9-beta.1
```

本輪 **`2026.5.9`** 在 changelog 上呈現為 **超大集合版本**：同時涵蓋 **Discord 語音 realtime 全棧**、**Talk 會話控制器與 RPC**、**Plugin SDK／channel-message／fs-safe**、**Matrix 外部化**、**iMessage 原生能力與 catchup**、**Control UI／Gateway 效能與安全**、**模型目錄與供應商正規化（Gemini 3.x、Mistral Medium 3.5、Qwen thinkingFormat）**、**依賴與 Node 底線上調** 等。相較 **`2026.5.6`／`2026.5.7`** 以穩定化修復為主的節奏，`2026.5.9` 更接近 **能力釋出 + 契約演進 + 長尾修復** 同捆發行。

---

## 2. 2026.5.9 — 重大功能與架構主題

### 2.1 Talk／語音／Discord realtime

- **Talk 統一控制面**：realtime relay、轉寫、managed-room、Voice Call、Google Meet、VoiceClaw 與原生客戶端收斂到 **共用 Talk session controller**，並新增 **`talk.session.*` Gateway RPC**；診斷／日誌路徑對 **OTLP／Prometheus** 匯出 **有界** lifecycle／audio 指標，且刻意 **不** 外洩轉寫全文、音訊 payload、room／turn／session id。
- **Discord 語音**：預設 **`voice` 模式為 `agent-proxy`**（realtime 作為已路由 OpenClaw agent session 的麥克風／喇叭延伸）；保留 **`stt-tts`** 作明確後備；新增 **STT/TTS、talk buffer、與 `openclaw_agent_consult` 的 bidi realtime** 等 `/vc` 模式；**OpenAI realtime** 預設模型 **`gpt-realtime-2`**、GA WebSocket session 形狀與後端／WebRTC／Google Live／Gateway relay 測試補強；**barge-in** 與 **`voice.realtime.minBargeInAudioEndMs`** 可調，降低回音房誤切模型音訊；ElevenLabs TTS **串流直入 Discord 播放** 與延遲最佳化 query；verbose 日誌含 **單行 STT 預覽**。
- **Voice consult**：可選 **OpenClaw agent voice context capsules** 與 **consult cadence**，讓 Gemini／OpenAI realtime **聽起來像已設定的 agent**，且不必每個一般 turn 都 consult 全量 agent。

### 2.2 Plugin SDK／channel 訊息生命週期／fs-safe

- **Unified model catalog**：文字／圖像／影片／音樂供應商的 **統一 catalog 註冊**（`providerCatalogEntry`、共用 media list help、live cache、per-model video capability overlays）。
- **`plugin-sdk/channel-message`**：`defineChannelMessageAdapter`、inbound reply context、durable 能力推導、**`createChannelMessageAdapterFromOutbound`**、**`actions.prepareSendPayload`** 等，讓核心掌握 **佇列／hook／retry／receipt**，插件專注 shaping。
- **Deprecated 橋**：直接 **`deliverOutboundPayloads`** 與 legacy reply-dispatch 標為 deprecated；**`check:deprecated-api-usage`** 強制分界；Discord／Slack／Mattermost／Matrix **live-preview finalization** 與 Telegram／Teams **final attach receipts** 遷移到 channel-message 語意。
- **`@openclaw/fs-safe`／`plugin-sdk/security-runtime`**：原子替換、sibling-temp、跨裝置 move fallback；瀏覽器／媒體／通道／QA 輸出 **先 staged fs-safe 再發布**；公開 temp API 更名為 **`tempWorkspace`／`withTempWorkspace`** 等。

### 2.3 Matrix 外部化與通道目錄

- **Matrix** 自核心同捆 **移回官方外部 ClawHub／npm 插件**，降低 **core 安裝對 Matrix SDK runtime** 的硬依賴；文件與 onboarding 需改以 **「外部通道 + 精確安裝／doctor 修復指令」** 敘述。
- **Matrix 簡報 metadata**：`com.openclaw.presentation` 附加於 semantic presentation，**OpenClaw-aware 用戶端** 可渲染 rich controls，一般客戶端仍走純文字後備。

### 2.4 iMessage／BlueBubbles 與通道能力

- **Breaking**：**移除 bundled BlueBubbles 表面**；既有 **`channels.bluebubbles`** 必須遷移到 **`channels.imessage`**（**`imsg`** 本機 Mac 或 SSH wrapper）；非 macOS 預設 **`imsg`** 設定會提示 **遠端 Mac wrapper** 指引。
- **iMessage 強化**：**`imsg rpc`** 暴露 reactions／edits／unsend／replies／rich send／attachments／group management（受 **`imsg status --json`** capability 約束）；**可選離線 catchup**（`channels.imessage.catchup.*`、狀態目錄游標、與 dedupe／allowlist 一致的重播語意）；**每群組 `systemPrompt`**（含 `groups["*"]` wildcard）與 WhatsApp **byte-identical** 解析語意對齊。
- **其他通道**：QQBot WebSocket **走環境 proxy**；Telegram **gramMY throttler 共用**、**reasoningDefault** 與 **streaming／poll／DM topic guard** 多項修復；Slack **implicit channel + thread** 與 root turn **單一 session**；Feishu／Telegram **reasoning 預設與預覽** 行為對齊 #73182 面向。

### 2.5 Gateway／sessions／tasks／效能

- **Task ledger RPC**：`tasks.list`／`tasks.get`／`tasks.cancel` **文件化 + 穩定化**，含 Swift optional typing；**stale CLI run-context** 與 **channel hot-reload deferral 有界 timeout** 對齊（呼應 2026.5.7 已敘述之 reload blocker 主題的延伸）。
- **Session store**：索引寫入 **原子化**、writer lock 內 **跳過 durable fsync** 以降低慢檔案系統上 cron／通道 turn 捱餓；**session list** 對已限定 model ref **fast-path**；**transcript path rotate** 與 rollover 搭配；**WebChat `/new` + `session.dmScope: main`** 行為與 Telegram／Discord 對齊。
- **Gateway restart**：`gateway.restart.request` 暴露 **`skipDeferral`**，CLI **`openclaw gateway restart --safe --skip-deferral`**；**macOS LaunchAgent** 多項生命週期修復（SIGUSR1、throttle、Homebrew PATH、`bootout` 預設、`--disable` 持久化等）。
- **效能**：多處 **避免全陣列排序**（provider auto-select、thinking level、Codex usage 訊息、ClawHub search、dreaming doctor entries、compaction contributor top-N 等）；**重用 plugin metadata snapshot** 於 dashboard／通道回合。

### 2.6 Control UI／安全／可觀測性

- **Exec policy badge**：改讀 **`tools.exec.security`**（修正誤讀 **`agents.defaults.exec.security`** 的非 schema 路徑）。
- **Usage 譜系**：transcript-backed **historical lineage rollups**（輪替邏輯 session 仍可追溯用量）；**Windows WebView2** bridge（draft → composer、ready handshake）。
- **Chat 安全與體驗**：即時流／轉寫 **去除不可信 sender metadata**；**連續重複純文字** 折疊為單氣泡計數；**繼承 thinking 預設** 與明確 override **分開標示**。
- **Logging／redaction**：引號內 HTTP client secret 欄位與 auth／cookie headers **脫敏**（關聯既有資安議題編號）。

### 2.7 模型／供應商／CLI

- **GitHub Copilot**：執行時自 **`${baseUrl}/models`** 刷新 catalog（權益與 context window），失敗時回落靜態 manifest（含 **`gpt-5.5`**）。
- **Google Gemini**：多點 **canonicalize `google/gemini-3.1-pro-preview`**，淘汰／巢狀 proxy catalog 中的 **`gemini-3-pro-preview`** 寫入與解析；**Gemini 3 Flash** catalog 缺列時走 template path **保留圖像能力**（修復 text-only 後備）。
- **Bedrock `serviceTier`**：`default`／`flex`／`priority`／`reserved` 可經 **`agents.defaults.params.serviceTier`** 或 per-model 設定。
- **Qwen `compat.thinkingFormat`**：支援 **`qwen`** 與 **`qwen-chat-template`**，並映射 `/think` 到 **`enable_thinking`** 或 **`chat_template_kwargs.enable_thinking`**。
- **Mistral**：catalog 新增 **`mistral-medium-3-5`**（含 reasoning）；文件補 **Medium 3.5** 設定、local infer、`reasoning_effort="high"` + `temperature:0` 的 HTTP 400 注意事項。
- **Chat 指令**：**`/think default`**、**`/fast default`** 清除 session override 並回到設定／供應商預設（#79385）；**infer** 路徑補 **thinking override** 與 **simple completions** 路由修正。
- **Codex app-server／PI**：多項 harness 版本釘選、tool search 預設 defer、**PI native WebSocket-capable transport** 承載 explicit Codex Responses、completion watchdog／idle timeout 診斷、native hook relay **長 turn 存活** 等（詳見 changelog 子條目）。

### 2.8 安裝／執行環境／Docker／Windows

- **Node 底線**：支援的 Node 22 地板提升至 **`22.16+`**（`node:sqlite` statement metadata 依賴），仍建議 Node 24。
- **Docker**：runtime 映像以 **`tini`** 為 PID 1，**回收孤兒行程** 並轉發訊號。
- **Windows**：**`openclaw update`** 子行程 **`stdio: "pipe"`** 避免繼承主控台 handle 導致 PowerShell／CMD **卡住**；登入 shell 環境探測 **隱藏子視窗**；插件 skill 目錄在 Windows 以 **junction** 發布避免 symlink EPERM；**WebView2** 見上。

### 2.9 其他 notable 能力

- **Agents**：system prompt **注入當前 provider／model identity**（含 prompt override／CLI hook 替換後仍反映 runtime）。
- **`oc-path` 插件**：`openclaw path` 提供 **`oc://`** 精準讀取 workspace markdown／JSONC／JSONL。
- **ACPX**：`agents.<name>` 支援可選 **`args` 陣列**，避免路徑或旗標值含空白時被錯誤拆段。
- **Active Memory**：`plugins.entries.active-memory.config.toolsAllow` 可列 **具名 recall 工具**（非 memory-core 預設三件套時仍可控）。
- **Cron CLI**：`cron list --agent <id>` 正規化 agent id，並在篩選語意下包含 **未存 agent id 的 job**（對齊 default agent）。
- **Managed proxy**：**`proxy.loopbackMode`** 控制 Gateway loopback 控制平面是否繞過／強制走 proxy／阻擋。
- **`before_agent_run`** hook：可在送模型前阻擋 user prompt（保留 redacted transcript 條目）；並澄清 raw conversation hooks 需 **`hooks.allowConversationAccess=true`**。
- **Plugin install**：**`npm-pack:<path.tgz>`** 與 guarded **環境變數覆寫**（onboarding／repair 測試路由）；managed npm root **省略冗餘 `--prefix .`** 修復 Windows WhatsApp 安裝 crash；**legacy peer** 行為避免 prune 回 **stale registry openclaw**。

---

## 3. Breaking：iMessage／BlueBubbles 遷移

- **使用者影響**：不再視 BlueBubbles 為 **內建／預設 iMessage 路徑**；設定與文件必須改寫為 **`channels.imessage` + `imsg`**（或遠端 Mac 包裝）。
- **維運影響**：doctor、onboard wizard、通道表、依賴掃描與 **core 預設依賴** 相關敘述皆需移除「BlueBubbles 為一等公民」假設；**Matrix** 改以 **外部插件安裝路徑** 描述。

---

## 4. Fixes 與穩定性（精選分群）

以下為 **`## 2026.5.9` → `### Fixes`** 中具 **跨模組代表意義** 的聚類（完整列表仍以英文 changelog 為準）：

- **Agents／CLI runner**：resume JSONL、supervisor 輸出有界、embedded CLI capture 策略、subagent spawn **`model:"default"`** 語意、failover **prefill 格式錯誤不重試**、**`stream_read_error`** 暫態化、auth cooldown **persist 順序**、compaction **空摘要保留 tail**、context-engine **loop-hook checkpoint 與 finalizer 共享** 等。
- **安全／合規**：圖像生成 **SSRF policy** 貫穿多供應商；browser SSRF deny **不關閉使用者自有 Chrome 分頁**（僅 OpenClaw 導覽）；exec approval **`argPattern`** 在 Linux／macOS 與 Windows **一致 enforce**；memory wiki **`wiki_search`／`wiki_get` session visibility**（GHSA 級修復敘述於 changelog）。
- **Backup**：暫存 manifest **不落在備份來源路徑內**（修復雙 manifest verify 失敗）；live backup **跳過易變動** session transcript／cron log／delivery queue 等。
- **Status**：Codex harness **`openai/*` usage** 走正確 quota provider，**`/status`** 與 **`openclaw status --usage`** 恢復 Codex 視窗顯示。
- **Telegram**：**DM `conversationId`+`parentConversationId` 同 id** 時 **不誤判為 topic**（#79700）；native commands **早於** workspace／agent bootstrap 處理，避免 `/status` 等 **卡在完整 turn 初始化** 後。
- **Google catalog**：**normalize model ids**（與 `fix: normalize google catalog model ids` 提交呼應）。
- **Launchd**：**`ProcessType=Interactive`** 降低 macOS App Nap 對 Gateway 的影響。

---

## 5. 受影響程式碼與文件面

本輪 **`CHANGELOG` 條目密度極高**，主要檔案族分布（**代表路徑**，非穷举）：

- **Discord／Talk／voice**：`extensions/discord/**`、`src/talk/**`、Gateway relay／realtime 相關模組。
- **Plugin SDK／通道抽象**：`src/plugin-sdk/**`、`packages/fs-safe/**`、各通道插件之 **channel-message** 遷移。
- **Matrix**：外部化後的 **ClawHub／npm 安裝與 catalog**、核心 **依賴圖** 精簡。
- **iMessage**：`extensions/imessage/**`、`imsg` 整合文件、`channels.imessage.catchup` schema 生成檔。
- **Gateway／sessions／tasks**：`src/gateway/**`、session store、task ledger、restart RPC。
- **Control UI**：`ui/**` 下 chat、usage、exec approval、WebView2 bridge。
- **CLI**：infer、cron、gateway restart、path 插件、migrate 互動流程。
- **英文文件**：`docs/channels/*`、`docs/cli/*`、`docs/plugins/*`、`docs/concepts/*`、`docs/install/*` 等隨 2026.5.9 大面積更新——**繁體 `docs-cht` 本輪以「版本標記 + 架構敘事 + 2026.5.9 精選段落」對齊**，細節仍應交叉 **https://docs.openclaw.ai**。

---

## 6. 繁體技術文件同步策略（本輪）

- **專案概覽**：版本 **`2026.5.9-beta.1`**；**Matrix 外部插件化**；**BlueBubbles 移除／iMessage 遷移**；新增 **`## 2026.5.9` 發行摘要** 與 **`commit-analyze-20260509.md`** 入口。
- **通訊頻道**：更新 **Matrix／BlueBubbles／iMessage** 列與說明；補 **Telegram DM guard、Discord voice 預設、Slack thread session** 等與 changelog 對齊的一句話導讀。
- **Gateway**：補 **Talk RPC、session store 原子寫入、restart skipDeferral、Tailscale preserveFunnel** 等運維向敘述（精簡）。
- **Agent／插件**：補 **prepared runtime foundation**、**channel-message／fs-safe**、**deprecation check** 的 **「契約演進」** 說明，避免讀者仍以 legacy reply pipeline 心智模型閱讀。
- **CLI／Onboard**：**`/think default`／`/fast default`**、**`cron list --agent`**、**`gateway restart --safe --skip-deferral`**、**`openclaw path`（oc-path）**；Onboard 表移除 **bluebubbles** 為建議路徑，改指向 **imessage**。
- **大模型／配置**：**Gemini 3.1 canonical id**、**Qwen thinkingFormat**、**Mistral Medium 3.5**、**Bedrock serviceTier**、**Copilot 動態 catalog**。
- **應用程式／TUI／記憶**：僅 **版本與精選條目** 滾動更新；細部 UI 差異以英文 docs 為準。

---

## 7. 本輪對齊統計

| 項目 | 結果 |
|------|------|
| **產品行為權威來源** | **`CHANGELOG.md` → `## 2026.5.9`**（含 **Breaking**／**Fixes**） |
| **`package.json` version** | **`2026.5.9-beta.1`**（請以工作區實際值為準） |
| **起始對齊點（上一輪 stable）** | **`v2026.5.7`** → commit **`eeef4864494f859838fec1586bedbab1f8fa5702`**（與 `commit-analyze-20260507.md` 一致） |
| **Git 可見非合併提交（`eeef486449..HEAD`）** | **約 2,752**（develop 拓撲；**不代表**單一 release 手動 cherry-pick 數） |
| **`git diff --shortstat`（同上範圍）** | 約 **6,067** 檔；**+257,060／−221,597** 行（**大分支合併** 量級；實際閱讀以模組／changelog 為主） |
| **主要變更類型** | Talk／Discord realtime、Plugin SDK channel-message／fs-safe、Matrix 外部化、iMessage 遷移與 catchup、Gateway／session／task、Control UI／安全／效能、模型目錄與供應商行為 |
| **本輪繁體文件主版本** | **`2026.5.9-beta.1`**（對齊 `package.json`） |

*若本地 tag 輕量／附註標籤與遠端不一致，請以 `git rev-parse "v2026.5.7^{commit}"`、`git rev-parse "v2026.5.9-beta.1^{commit}"`（或前 beta tag）與 `CHANGELOG.md` 頂段交叉驗證後修正本表。*
