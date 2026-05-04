# OpenClaw 提交分析報告 — 2026-04-27

> **分析範圍**：從 `112b58dc764bf60fb97f7003ba4077153c177388`（doc: 更新繁體中文文件版本號至 2026.4.26）對照至 **`origin/main`** 工作區快照（`da1e1435ad3109231fa5d52121d8b295e62bd7d4`）。  
> **分支關係**：`112b58dc76`（多見於 `develop` 繁體對齊提交）與目前 **`main`** **非直接祖先關係**；共同祖先為 `f7797ca62b841be551b9ddaf76e32620ca14600f`。自該分叉點起，**`main` 側**另含 **約 4000 個非合併提交**（`112b58dc76..origin/main` 之「可达自 main、不可达自 112b58」集合）；**`develop` 側**相對 merge-base 另有 **約 232** 個非合併提交未納入 `main`。本報告以 **主線 `main` 程式＋英文 `CHANGELOG.md` → `## Unreleased`** 為產品行為主軸（官方尚未發佈獨立 `## 2026.4.27` 標題時，繁體技術文件版本 **2026.4.27** 對齊 **2026-04-27** 之快照與 Unreleased 彙整）。  
> **參考**：`docs-cht/commit-analyze-20260426.md`；未逐條展開全部提交訊息。

---

## 目錄

1. [版本路徑與方法說明](#1-版本路徑與方法說明)
2. [Unreleased — Highlights](#2-unreleased--highlights)
3. [Unreleased — Changes（依主題）](#3-unreleased--changes依主題)
4. [Unreleased — Fixes（精選分組）](#4-unreleased--fixes精選分組)
5. [基礎建設／QA／發布](#5-基礎建設qa發布)
6. [摘要統計](#6-摘要統計)

---

## 1. 版本路徑與方法說明

```
2026.4.26  → 上一次繁體中文技術文件對齊終點（QQBot／Yuanbao／migrate／Matrix E2EE、manifest 驅動、大量 Fixes）
2026.4.27（本輪 docs-cht 標註）→ 對齊 main 快照：CHANGELOG「Unreleased」：file-transfer 插件、Meet／Voice realtime 語音對齊、統一 progress 串流、Gateway 效能與 fail-closed 設定、插件安裝／ClawHub／MCP 政策、大量 Control UI／通道／Cron／安全硬化
```

**為何以 Unreleased 為主？**  
分叉後主線提交量極大，且條目已收斂於 **Highlights／Changes／Fixes**；與先前報告相同，**英文 changelog + 主題分類**最能對齊可感知行為與營運修復。

---

## 2. Unreleased — Highlights

- **檔案傳輸插件（bundled）**：`file_fetch`、`dir_list`、`dir_fetch`、`file_write` 等 **對 paired node 的二進位檔案工具**；每節點路徑 **預設拒絕**、**運營核准**、`followSymlinks` 可選、**單趟 16 MB** 上限（#74742）。  
- **Google Meet／Voice Call**：Twilio 撥入與 **realtime Gemini 語音橋** 對齊——**節流音訊**、**背壓緩衝**、**插話清空佇列**、realtime 語音期間 **不走 TwiML 後援**（#77064）。

---

## 3. Unreleased — Changes（依主題）

### Control UI

- **儀表標頭**：顯示 **作用中 agent 名稱**（不塞入 session key）。  
- **Cron**：**新增工作**側欄 **可摺疊**，釋出列表空間。  
- **效能診斷**：瀏覽器 **long animation frame／long task** 寫入 debug 事件日誌（若環境支援）。  
- **Talk**：失敗橫幅可關閉、清 stale 錯誤、下一點 Talk **可從失敗狀態重試**；**Canvas／Talk** 與 TLS、CORS（Realtime 標頭）等修復見 Fixes。

### 串流／通道

- **統一 `streaming.mode: "progress"`** 草稿：自動單字狀態標籤；**Discord、Telegram、Matrix、Slack、Microsoft Teams** 共用進度設定。  
- **Slack**：`streaming.progress.render: "rich"`（Block Kit）；**保留最新 rich 行**於長草稿裁切時。  
- **工具列顯示**：預設 **精簡** `/verbose` 與 progress 之工具摘要；**`agents.defaults.toolProgressDetail: "raw"`** 與 per-agent 覆寫可還原 raw command／detail。  
- **Discord 狀態**：`channels status`／`status --deep` 可見 **degraded transport**、**event-loop starvation** 等訊號（#76327）。  
- **WhatsApp**：支援 **`@newsletter`** 頻道／電子報 **外送目標**（#13417）。  

### Agent／指令

- **`/steer <message>`**：在 **session 閒置**時對 **目前 run** 做 **佇列外轉向**，不另開新回合（#76934）。  
- **`/side`**：作為 **`/btw`** 的 **文字與原生 slash 別名**。  
- **Subagent**：direct completion fallback 繞過 announce 時仍 **保留所有分組子結果**。  

### Gateway／設定／效能

- **無效設定**：啟動與熱重載 **不再自動還原**無效 config；**fail closed**，由 **`openclaw doctor --fix`** 負責 last-known-good。  
- **冷啟動**：**延後**重負載模組（cron、部分 discovery、shutdown hook、maintenance timer 至 ready 後）、**縮減** plugin auto-enable 重複工作、**避免** native 路徑不必要 **jiti**、**fast-path** 信任之 bundled metadata。  
- **`pnpm gateway:watch --benchmark-no-force`**、startup benchmark **`--cpu-prof-dir`** 等剖析支援。  

### 插件／ClawHub／安裝

- **Manual setup** 可裝 **選用官方插件**（ClawHub＋npm fallback）、**Codex 外掛**可選。  
- **外部化／信任鏈**：official **npm migration**、ClawHub↔npm **回補與回迁**、清 **stale bundled load paths**；**beta 通道**插件更新 **先試 `@beta`**。  
- **`openclaw plugins list --json`** 含 **依賴安裝狀態**（不必 runtime 載入插件）。  
- **ClawHub 429**：附 **RateLimit-Reset／Retry-After** 提示與 **登入可提高額度** 說明。  
- **Runtime state**：**`registerIfAbsent`** keyed-store（原子宣告、避免覆寫既有 live 值）。

### 其他與開發者體驗

- **`doctor --fix`**：即使 **其他驗證失敗**（例如缺插件），仍 **寫入安全 legacy migration**（如 `agents.defaults.llm` 等）（#76798、#76800）。  
- **IRC 文件**：釐清 **raw TCP／TLS**，與 **轉發代理** 管理邊界。  
- **OpenRouter（可選）**：**response caching** 標頭與 **app 歸因**類別擴充。  
- **Exec approvals**：**tree-sitter** shell **command explainer** 基礎（#75004）。  
- **Sandbox registry**：容器／瀏覽器登錄改 **每 runtime shard 檔**，降鎖競爭（#74831）。  
- **QA／Mantis**：Discord smoke、**Slack desktop（Crabbox VNC）**、Blacksmith **`tbx_...` lease**、Slack live transport 等 runner 與 workflow。

---

## 4. Unreleased — Fixes（精選分組）

> 下列為 **高信號**分組；完整列表以 **主線 `CHANGELOG.md` → `## Unreleased` → Fixes** 為準。

### Google Meet／Voice／Realtime

- **Agent 預設 talk-back**：**`mode: "agent"`**（STT → OpenClaw agent → TTS）；**`mode: "bidi"`**／**`realtime.strategy`** 保留 **雙向 realtime**；**`realtime.introMessage: ""`** 可靜音 intro。  
- **音訊／橋接**：Meet 麥克風 unmute 等待、**BlackHole 2ch**、**chrome.audioBufferBytes**、**voiceCall.postDtmfSpeechDelayMs** schema、realtime **`session.updated`** 就緒前不視為已連線、**echo／VAD** 與播放管線錯誤處理；**Twilio realtime 佇列有界**、超載關閉串流。  
- **Consult context**：Meet 顧問 **fork 目前 agent transcript**。  
- **TTS logging**：電話／Meet 後端 **voice／model／sample rate** 日誌對齊實際合成（與 Changes 呼應）。

### 串流／Discord／Slack／Matrix／Mattermost／Telegram／Feishu／MSTeams

- **`streaming.preview.toolProgress`／`streaming.progress.toolProgress`** 與 **quiet preview** 行為在多通道 **對齊文件**；**`commandText: "status"`** 隱藏指令文字選項。  
- **Discord**：probe **非阻塞啟動**（#77103）、SecretRef bot token **runtime snapshot**（#76987）。  
- **Matrix**：approval **reaction 目標先綁定**；進度草稿與 QA 情境硬化。  
- **Telegram**：forum topic **數字目標**、reply-dispatch **stable dist 別名**、**按鈕式互動**送 fallback 文案、album **`mediaGroupFlushMs`**、多項 **streaming／status** 修復。  
- **Feishu**：**`blockStreaming`**、佇列 **逾時釋放**、`audioAsVoice` 回落文案等。

### Gateway／sessions／models／chat

- **`sessions.list` 預設有界**＋**截斷 metadata**（#77062）；thinking 選項 **memo**、列表列 **減負載**（#76931）。  
- **usage／cost**：**transcript aggregate cache**、可 **stale** 標示（#76650）。  
- **`models.list` 唯讀路徑**避免不必要的 provider discovery（#76382 系列延續）。  
- **聊天／WebChat**：**重複送出**折疊到同一 run（#75737）、compaction **邊界說明**、**重播／壓縮**與 WebChat **重複 assistant turn**（#76424、#77033）。

### 插件／工具／MCP／安全

- **工具 denylist**：在 **factory 建立前**即跳過 **PDF／media**（#76997）；**MCP bridge** 套用 **`tools.profile`／`alsoAllow`／deny`**；**plugin tools** runtime **尊重 deny** 與 **manifest optional**。  
- **Plugin tools**：**optional 同捆** factory  siblings **隱藏策略**修正。  
- **Presentation**：**理由文字**自 **rich presentation** 標題／區塊 **剝除**，防 **message tool** 外洩（#77139 類思想防護鏈）。  
- **Doctor／lock**：**stale plugin lock**、**managed shadow**、**feishu vs lark owner**（#76623）等修復（與 HEAD `da1e1435` doctor 修補一致）。  
- **WhatsApp**：**libsignal** `onlyBuiltDependencies`、登入 QR **走 runtime**、**群組提及** metadata（#76539、#39879 等）。

### CLI／Codex／裝置／安裝

- **Codex**：**`auth.order.openai-codex`**、**IPv4／IPv6 fallback**（#76857）、**Talk WebRTC CORS** 標頭剔除（#76435）。  
- **Device pair**：**operator.admin** 重試（#76956）。  
- **LaunchAgent／systemd**：**`.env` 管理鍵**保留、**operator secrets** 跨 re-stage（#75374、#76860）；**kickstart** 時序避免誤殺新進程（#76261 等）。  
- **Logs `--follow`**：斷線 **重連**與 JSON **notice**（#74782、#75059）。

### 其他（單列高信號）

- **web_search／web_fetch**：**late-bind runtime**、**manifest 範圍載入**、**auto-detect 可見性**（#77073 等）。  
- **DeepSeek V4**：**`xhigh`／`max`** 於 policy／TUI **`/think`**（#77139、#76482）。  
- **Media**：**HEIC** 缺 Sharp 時 **fail-closed**；**Telegram** 無 optimizer 仍可送 **尺寸內原圖**（#77117）。  
- **Canvas**：Gateway **TLS scheme** 保留於 canvas URL **日誌**與連結。  
- **iOS pairing**：非 loopback **`ws://`** **拒絕**於 QR／setup code 發放前。

---

## 5. 基礎建設／QA／發布

- **Windows Blacksmith testbox**、**release validation**（含 **2026.5.3 updater** native module swap **後續驗證**）、多條 **QA live**／**Matrix**／**Slack** 硬化。  
- **Maintainer**：預設經 GitHub **verified commit API** 推 PR head（需明確 override 才允許未簽協定 push）。  
- **文件**：**Google Meet ElevenLabs voice**、**Pi transcript ownership**、**Crabbox maintainer** 等補述。

---

## 6. 摘要統計

| 項目 | 數值 |
|------|------|
| 上一輪繁體對齊提交 | `112b58dc764bf60fb97f7003ba4077153c177388` |
| 本輪參考主線 `HEAD` | `da1e1435ad3109231fa5d52121d8b295e62bd7d4`（`origin/main`） |
| `main` 相對上一輪提交之非合併提交（約） | **4000**（分叉後主線增量；見文首分支說明） |
| 共同祖先（`develop`／`main`） | `f7797ca62b841be551b9ddaf76e32620ca14600f` |
| 官方變更主軸 | `CHANGELOG.md` → **`## Unreleased`** |

*本報告與 `docs-cht/*.md`（`commit-analyze-*` 除外之版本標註）對齊至 **2026.4.27**；若你僅追蹤 `develop`，請留意與 `main` 分叉，並以實際 checkout 之 `CHANGELOG` 為準。*
