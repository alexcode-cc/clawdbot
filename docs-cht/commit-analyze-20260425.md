# OpenClaw 提交分析報告 — 2026-04-25

> **分析範圍**：從 `23890e657500a0fac8eebdbeff1c5570f29550cc`（doc: 更新繁體中文文件版本號至 2026.4.24）至工作區 `HEAD`（`3e93d48706` 時點；含 `chore(release): bump stable 2026.4.25` 等），歷史區間內共 **1,049 個非合併提交**。  
> **參考**：`docs-cht/commit-analyze-20260424.md`；本輪以 **官方 CHANGELOG**（`## 2026.4.25` 為主；檔案頂部另含精簡的 `## 2026.4.24` **Fixes** 維護條目）為主軸彙整；未逐條展開全部提交訊息。

---

## 目錄

1. [版本路徑與方法說明](#1-版本路徑與方法說明)
2. [CHANGELOG 頂部 `2026.4.24`（維護向 Fixes）](#2-changelog-頂部-2026424維護向-fixes)
3. [2026.4.25 — Highlights](#3-2026425--highlights)
4. [2026.4.25 — Changes（依主題）](#4-2026425--changes依主題)
5. [2026.4.25 — Fixes（精選主題）](#5-2026425--fixes精選主題)
6. [基礎建設與發布](#6-基礎建設與發布)
7. [摘要統計](#7-摘要統計)

---

## 1. 版本路徑與方法說明

```
2026.4.24  → 上一次繁體中文技術文件對齊終點（DeepSeek V4、Meet、瀏覽器、插件 mirror 等）
2026.4.25  → Stable：TTS／語音全面升級、冷持久化插件註冊表、OTEL／診斷大擴張、瀏覽器與 Control UI、安裝更新硬化、大量通道／Agent／插件修復
```

**為何以 CHANGELOG 為主？**  
逾千筆提交含 **多個 beta**、CI／Docker／release 管線、測試硬化與 backport；與先前報告相同，**產品 changelog + 主題分類**最能對齊英文文件與使用者可感知行為。

---

## 2. CHANGELOG 頂部 `2026.4.24`（維護向 Fixes）

> 下列為目前 `CHANGELOG.md` 在 **`## 2026.4.26`** 之前、緊接 **`## 2026.4.24`** 的 **Fixes** 區塊（無獨立 Highlights），屬 **延續性修復**，與 **2026.4.25** 大版並存於同一時間線：

- **Auto-reply**：在 replay 不安全之 provider／runtime 失敗後 **標記 dedupe 為不可用**，避免在有區塊輸出、工具副作用或 session 進度後 **重試重複發話**（#69303 等）。
- **Gateway／Bonjour (@homebridge/ciao)**：廣告器重啟後仍保留 **cancellation handler**，避免晚期 probe 取消在 Linux／高 churn mDNS 環境崩潰 Gateway。
- **Plugins**：Gateway 啟動時在允許前提下 **載入預設 `memory-core` slot**，使 active memory 不需明確 `plugins.slots.memory` 亦可呼叫 **`memory_search`／`memory_get`**（仍尊重 **`plugins.slots.memory: "none"`**）。
- **Plugins／CLI**：bundled 插件編譯後 JS **優先 native require** 再走 jiti，減慢機上 config／status／device／node 等冷路徑開銷（#62842）。
- **Plugins／compat**：doctor 遷移與 **runtime 相容**分庫治理；補齊 **dated compatibility** 紀錄（舊 extension-api、memory 註冊、別名等）。
- **Plugins／CLI**：受管插件刪除後 **刷新持久化 registry**；安裝／解除安裝寫入 **衝突感知**、denylist 清理、**先 commit 索引再刪檔**；**`plugins update`** 在追蹤錯誤時失敗；**unloadable extension** 拒絕安裝。
- **WebChat／Control UI**：聊天上傳支援 **非影片檔附件**，保留既有圖片路徑與 **MIME sniff**（#70947）。
- **Gateway／chat**：重試 **同 idempotency key** 的附件型 `chat.send` 維持在 **in-flight** 路徑（#70139）。
- **Plugins**：安裝與 discovery **共用 entrypoint 解析**、拒絕不匹配 **`runtimeExtensions`**、掃描時快取 **bundled runtime-dep manifest**。
- **WhatsApp／Web**：watchdog 改以 **Web transport 活動**為基準保留「安靜但健康」連線，並保留較長 **app silence cap**，避免 frame 活動永久遮掩卡住 session（#70678）。

---

## 3. 2026.4.25 — Highlights

- **語音／TTS**：`/tts latest`、依聊天 **auto-TTS**、**personas**、每 agent／每帳號覆寫；新增／強化 **Azure Speech**、**小米**、**Local CLI**、**Inworld**、**Volcengine**、**ElevenLabs v3** 等 bundled 語音提供者。
- **插件啟動／安裝**：遷移至 **冷持久化 registry／索引**，縮減廣泛 manifest 掃描；**update／repair／discovery／install metadata** 更確定。
- **OpenTelemetry**：涵蓋 **模型呼叫**、token、工具迴圈、harness、exec、出站傳遞、context 組裝、記憶體壓力等；**低基數**屬性與 **語意規範 opt-in**（`gen_ai_latest_experimental`）對齊。
- **瀏覽器自動化**：較安全的 **tab URL**、**iframe 感知**角色快照、**CDP readiness**、`browser doctor --deep`、**headless one-shot** `browser start --headless`、慢機 **timeout** 可調。
- **Control UI／Setup**：**PWA**、**Web Push**、**Crestodian** 首次修復、**TUI setup**、context mode、較短問候；安裝／更新涵蓋 **Win／mac／Linux／Docker**、**LaunchAgent token**、**版本一致**驗證等。

---

## 4. 2026.4.25 — Changes（依主題）

### TTS／通道／Agent

- **`/tts latest`**、**`/tts chat on|off|default`**（會話級 auto-TTS）；**Feishu／QQBot** 等 **`channels.<id>.accounts.<id>.tts`** 深度合併；**`agents.list[].tts`** 覆寫全域 **`messages.tts`**。
- **Personas**：提供者綁定合併、**`/tts persona`**、Gemini **`audio-profile-v1`**、OpenAI instruction 對應（#70748）。
- **Voice Wake**：macOS 片語可路由至指定 **agent／session**（#30354）。
- **Discord voice**：**`channels.discord.voice.model`** 覆寫語音頻道所用 LLM（#64368）。

### 語音提供者（bundled）

- **Azure Speech**（Speech 資源認證、SSML、Ogg/Opus voice-note、電話輸出）（#51776）。
- **小米 MiMo TTS**、**Local CLI TTS**、**Inworld**、**Volcengine／BytePlus Seed Speech**、**ElevenLabs `eleven_v3`** 等（各冇獨立 issue／PR 編號見英文 changelog）。

### 瀏覽器／CLI

- Agent 回覆 **安全 tab URL**；**CDP 原生角色快照**後備、**iframe** ref、**`browser doctor --deep`**。
- **`openclaw browser start --headless`** 一次性覆寫；**managed Chrome** discovery／CDP **readiness timeout** 可拉高（#66803）。
- **`infer image generate|edit`**：**`--background`** 泛型、`--openai-background` 別名；**fal** 支援 **`--output-format png|jpeg`**。
- **`infer image edit`**：**`--size`／`--aspect-ratio`／`--resolution`** 等（capability inspect 對齊）。

### Control UI／安裝體驗

- **PWA 安裝**、Gateway 聊天 **Web Push**（#44590）。
- **Crestodian**：首次修復、本機 planner、**全 TUI**、進度指示、**context mode**、較短啟動問候（#71720、#71760）。

### 插件／registry／CLI

- **`plugins registry`**（檢視／**`--refresh`**）；**`plugins list`** 預設讀 **冷快照**；Gateway 啟動規劃依 **版本化 cold registry index**；**postinstall** 修復舊格式。
- **Provider discovery／catalog hook／Provider Index** 多路徑改走 **installed index**，減少 **broad manifest scan**；**`models list --all --provider`** 對靜態目錄提供者加速。
- **`plugins.installs.json`** 為狀態管理之 **安裝索引**（取代 **`plugins/installed-index.json`** 與作者可寫 **`plugins.installs`**）。
- **`OPENCLAW_DISABLE_PERSISTED_PLUGIN_REGISTRY`** 標記為 **break-glass 棄用**；相容性登記 **dated removal**（最長約三月窗口）。
- **Hooks**：**before-agent-finalize**、cron **`jobId`** context、Codex **MCP hook relay**（#71765、#71758、#71707）。
- **Tokenjuice** runtime **0.6.3**。
- **Startup**：**stage runtime dependencies before load**；**materialize staged plugin chunks**；與 **ESM／packaged mirror** 修復一脈（#72058）。

### Google Meet

- **Calendar** 輔助之 **出席匯出**、manifest、**dry-run**、與會議紀錄工具對齊。

### 診斷／OTEL／Prometheus

- **GenAI** span／metric 與 **`OTEL_SEMCONV_STABILITY_OPT_IN`**；**signal 專用 OTLP endpoint**；**exporter 健康**診斷；**`openclaw.harness.run`** spans；**traceparent** 僅信任派發端；**`diagnostics-prometheus`** 插件與受保護 scrape 路由。
- **model_call_started／ended** metadata-only hooks；**context 組裝**、**tool loop**、**memory 壓力**、**出站傳遞**、**exec** 等 bounded telemetry；**`gen_ai.client.token.usage`**、**duration** histogram；**per-agent** token 標籤等（詳見英文 **Docs/OTEL** 條目）。

### Codex／ACPX／LiteLLM／fal

- Codex app-server **≥ 0.125.0**；native MCP **PreToolUse／PostToolUse／PermissionRequest** 經 hook relay。
- **`agents_list`**／提示偏好 **原生 app-server** 於 **Codex ACP**（除非明確 ACP/acpx）。
- **Factory Droid** 納入 live ACP bind Docker matrix。
- **LiteLLM** 註冊為 **image generation** 提供者。
- **fal Seedance 2.0** reference-to-video 等 **`video_generate`** 能力。

### Android

- **Talk Mode** 於 Voice 分頁露出；**voice capture modes**、mic **foreground service** 升級。

---

## 5. 2026.4.25 — Fixes（精選主題）

> **Fixes** 區塊條目極多，以下依 **使用者／維運高信號** 分組；完整列表以 `CHANGELOG.md` `## 2026.4.25` → **Fixes** 為準。

### 自動回覆、Agent、Fallback

- **dedupe poison**（與頂部 2026.4.24 Fixes 呼應）；**bundle-mcp allowlist** 語意修正；**live session model redirect** 捷徑；**OpenAI Responses web search** 與最小 thinking 相容。
- **Subagent**：無 thread 請求者在父 announce 無可見回覆時，**直接 fallback** 交付子代理結果。
- **Claude**：零 token **stop** 視為失敗並重試／fallback（#71880）。
- **Z.AI／vLLM／群組 NO_REPLY** 等邊角行為見 changelog 條目。

### 日誌、Transcript、Secret

- **持久化 transcript** 套用 **redaction**；自訂 regex 支援跳脫字元類別（#42982）。
- **主控台／檔案 log** 出口 **redact**（#67953）。

### Discord／Gateway／Bonjour

- **Health monitor** 重啟計入冷卻與上限；**Bonjour** 取消／斷言失敗抑制、**Docker** 預設關廣播、**probing** 迴圈防永遠失敗（#67578、#69011、#71879 等）。
- **exec approval**：較高模式已自動解決時 **靜默晚到點擊**（#66906）。

### 插件／通道冷路徑與解除安裝

- **channels／status／audit** 多處改 **唯讀 index metadata**，避免為顯示標籤載入完整 channel runtime。
- **`plugins uninstall --force`** 自 **記錄的 managed extensions root** 刪檔；**contextEngine slot** 隨記憶 slot 遷移。
- **WhatsApp**：無 **`channels.whatsapp`** 時不因持久 auth 觸發 **runtime-dep repair**（#71994）。
- **Signal**：改以 **Node HTTP client** 讀 RPC／SSE，避開 Node 24/25 **fetch** 迴歸（#51716、#53040）。

### 安裝／更新／服務

- **LaunchAgent**：偵測 **過期內嵌 gateway auth** 並刷新安裝狀態（#70752）。
- **更新後版本不一致** 視為失敗（#71835）；**磁碟空間**警告；**runtime dep** 驗證失敗阻擋 **假成功** repair（#71883）。
- **混版二進位** 禁止改寫較新 config 所對應之服務（#57079）。
- **Windows**：`schtasks` 逾時、`pnpm` 路徑引號、Lobster runtime 等（#69970、#45275、#69456）。
- **Linux**：apt **noninteractive**、**fnm PATH**、`openclaw node install` 重啟 **node unit** 等。

### Control UI／配對／裝置

- **Tailscale** 操作員可略過部分 **device pairing**（#71986）；**pairing store** 毀損不當空覆寫（#71873）。
- **Device token**：旋轉／撤銷 **caller scope** 封裝（#71990）。
- 歷史重載 **skeleton**、樂觀訊息、**heartbeat 列過濾後**筆數上限等 UX（#71844、#71878 等）。

### TTS／通道細節

- 串流區塊間 **TTS directive** 剥离；**Feishu／BlueBubbles** voice-note 路由；**WhatsApp** voice transcript 與 **mention** gating；**媒體理解**避免 **重複轉寫** 同一音訊等。

### Cron／Tasks／ACP

- **Cron ledger** 自耐久 log 復原（#71963）；**Task** 生命周期時間戳與 terminal 狀態（#71905、#71871）。
- **ACP**：`/acp`／`/status`／`/unfocus` 留在 Gateway 路徑（#66298）；完成 wake 對外部 harness 為 **純 prompt**；啟動 **不預設 probe** acpx。

---

## 6. 基礎建設與發布

- **Beta 線**：多個 **`2026.4.25 beta`** bump、release tarball／npm publish **驗證**、Docker **chunk／scheduler**、Telegram／QA workflow 調整等。
- **Plugins**：**preserve bundled runtime mirror chunks**、**stage startup deps before load**（與 changelog Fixes／Changes 對齊）。
- **Skills／memory**：Chokidar **具體根與 filter** 熱重載（#27404、#33585、#41606）。

---

## 7. 摘要統計

| 項目 | 數值 |
|------|------|
| 分析起點（上一輪 CHT 對齊提交） | `23890e657500a0fac8eebdbeff1c5570f29550cc` |
| 本輪非合併提交（約） | **1,049** |
| 對外穩定版標籤 | **2026.4.25**（接續 **2026.4.24**） |
| 主要參考 | `CHANGELOG.md` → `## 2026.4.25`（兼顧頂部 `## 2026.4.24` Fixes） |

*本報告與 `docs-cht/*.md` 版本標註對齊至 **2026.4.25**；細節以官方英文 changelog 與原始提交為準。*
