# OpenClaw 提交分析報告 — 2026-05-04

> **分析範圍**：以上一輪繁體文件對齊點 **release `v2026.5.3`** 為起點，分析 **`v2026.5.3..v2026.5.4`** 的新增／更新提交，並以英文 **`CHANGELOG.md` → `## 2026.5.4`** 作為產品行為主軸。
> **版本對齊**：`package.json` 目前為 **`2026.5.4`**；`git tag v2026.5.4` 指向 **`325df3efefe9c0887d9357732e68fc8556e78d79`**。
> **上一輪繁體對齊**：release **`v2026.5.3`**／**`docs-cht/commit-analyze-20260503.md`**。
> **提交量**：`v2026.5.3..v2026.5.4` 約 **527** 筆提交，其中 **522** 筆為非合併提交；檔案差異約 **1437** 個檔案。
> **方法**：不逐筆列舉所有提交，改以 **Highlights → Changes → Fixes → 文件同步重點** 分組，保留可追溯的 commit／issue／PR 編號。

---

## 目錄

1. [版本路徑與判讀方式](#1-版本路徑與判讀方式)
2. [2026.5.4 — Highlights](#2-202654--highlights)
3. [2026.5.4 — Changes（依主題）](#3-202654--changes依主題)
4. [2026.5.4 — Fixes（精選分組）](#4-202654--fixes精選分組)
5. [繁體文件同步重點](#5-繁體文件同步重點)
6. [摘要統計](#6-摘要統計)

---

## 1. 版本路徑與判讀方式

```text
v2026.5.3
    → 2026.5.4 beta / release preparation
    → v2026.5.4
```

本期不是單一大型新功能版本，而是一次 **語音會議、插件外部化、Gateway 啟動效能、通道串流、Windows 安全／相容性、Codex harness** 的密集穩定化版本。英文 changelog 的 `2026.5.4` 區段可視為權威摘要；提交歷史則顯示許多修復先在 beta／測試／CI 迭代中落地，再收斂到發行切段。

---

## 2. 2026.5.4 — Highlights

- **Google Meet／Voice Call 即時語音**：Twilio dial-in 會議現在可透過 realtime Gemini voice bridge 發聲，包含節流音訊串流、背壓緩衝、插話時清空佇列，以及 realtime speech 期間不回落 TwiML。這讓會議內的 OpenClaw 語音代理反應更快、更不像批次播放器（#77064）。
- **Gateway／插件啟動效能**：模型目錄、manifest contract、channel plugin barrel、TypeBox memory schema、QR pairing helpers、sessions list enrichment 等熱路徑被拆離冷啟；可重用 workspace-compatible plugin metadata snapshot，減少重複 cold scan。
- **插件外部化與更新修復**：官方 externalized 插件在 npm／ClawHub／beta channel／managed npm-prefix／peer link／runtime alias 之間的遷移更穩定；doctor 與 update 現在能修復缺失的 `openclaw` peer link、stale bundled load path、missing official plugin payload。
- **串流進度與多通道交付**：Slack rich progress draft、跨通道 `toolProgressDetail`、compact tool lines、Telegram 長文 final preview reuse、Discord dropped final reply failure、Slack mention gating thread participation 等大量修復，讓「看得見的進度」與「真正送達」更一致。
- **Codex harness 與 OpenAI/Codex**：Codex audio transcription metadata、usage-limit reset details、Codex OAuth routing、bound thread recovery、app-server protocol mirror、direct Responses SSE 預設等修復，主要目標是避免 Codex turn 靜默失敗或走錯認證。
- **Windows／安全硬化**：Windows loopback 只綁 `127.0.0.1`、Docker bind 支援 drive-letter、POSIX tmp path 跳過、`SystemRoot`／`WINDIR`／`LOCALAPPDATA`／`.cmd` wrapper／`reg.exe` 等 host-env 解析固定至可信 Windows 根，降低 workspace `.env` 汙染風險。

---

## 3. 2026.5.4 — Changes（依主題）

### Gateway／啟動／診斷

- **Windows loopback**：預設 Gateway listener 在 Windows 只綁 `127.0.0.1`，避免 libuv dual-stack `::1` 行為卡住 localhost HTTP（#69701）。
- **Startup 熱路徑瘦身**：model catalog test helpers、run-session lookup、QR pairing、TypeBox memory tool schema、channel barrel import、`jiti` fallback 等都更晚載入或只在需要時載入。
- **診斷可觀測性**：`pnpm gateway:watch` 新增 startup phase spans、active work labels、stale terminal bridge markers、sync-I/O tracing；benchmark 模式會把 trace 導向 artifact/log，避免 terminal 被 stack trace 淹沒。
- **安全重啟協調**：新增 safe restart coordinator，更新後 health probe 會使用已安裝 config 的 local gateway auth，避免 token/device-auth VPS 被誤判成不健康或 port conflict。
- **sessions.list 邊界**：繼續收斂大型 stores 的 row construction；CLI `sessions list` 輸出也有界，避免長期 Slack／多代理 store 造成無界輸出。

### Agent／工具／Prompt Context

- **post-compaction loop guard**：`pi-embedded-runner` 在 auto-compaction retry 後觀察相同 `(tool, args, result)` 三元組；預設視窗 3 次，命中後以 `compaction_loop_persisted` 中止，可由 `tools.loopDetection.*` 調整。
- **工具 allowlist**：embedded runtime 會尊重窄 tool allowlist 建構工具族與 bundled MCP/LSP runtimes，避免 cron／subagent run 因缺工具或多餘 bootstrap 失敗（#77519、#77532）。
- **runtime prompt context 隔離**：每回合 runtime context 不再混入一般 chat system prompt，仍可透過 hidden current-turn context 傳遞，恢復 prompt-cache reuse（#77431）。
- **hook bootstrap**：`agent:bootstrap` hook 注入的 `BOOTSTRAP.md` 內容會被視為 pending bootstrap，避免必要 setup 指令被漏進系統提示。
- **tool progress detail**：`agents.defaults.toolProgressDetail: "raw"` 可讓 Slack、Discord、Telegram、Matrix、Teams 等 progress draft 顯示 raw command/detail；預設則採用較精簡摘要。

### 插件／ClawHub／SDK

- **官方 externalized 插件提示**：當 `plugins.entries` 或 `plugins.allow` 參照未安裝的官方外部插件，migration 會提供 catalog-backed install hint，而不是要求移除有效設定。
- **beta channel install**：onboarding／doctor-managed install 在 beta 更新通道會用 `@beta` 浮動 spec 請求 npm／ClawHub，同時持久紀錄仍保留 catalog default。
- **managed npm 修復**：更新流程會修補缺失的 plugin-local `openclaw` peer link，並在 package-manager upgrade 後恢復 managed-npm external plugins。
- **runtime entry 與 dist 修剪**：externalized plugin entry chunks、package-excluded plugin dist、blank runtime entries、source-only shadows 等路徑持續硬化，避免容器或 release artifact 帶入半套 runtime。
- **Plugin SDK session slots**：新增 plugin-owned `SessionEntry` slot projection 與 trusted-policy scoped session extension reads（#75609）。
- **`registerIfAbsent`**：plugin runtime keyed store 支援原子宣告，避免 dedupe claim 覆寫已存在 live value。

### 通道／串流／回覆

- **Slack rich progress**：`streaming.progress.render: "rich"` 可用 Block Kit 呈現結構化 progress lines；超出 Block Kit 限制時保留最新行。
- **Slack mention gating**：成功的 threaded visible send 會記錄 thread participation，bot 曾參與的 thread 可依文件規則通過 mention gating（#77648）。
- **Discord final delivery**：final reply delivery 失敗會讓 turn 失敗，不再把 dropped final reply 當作已送達；Discord startup 也偏好 IPv4，降低 IPv4-only 網路卡住風險。
- **Telegram**：長文 final 會重用 active preview 作為第一 chunk，減少「多一個氣泡又消失」的閃爍；入站媒體 placeholder 改由 MIME metadata 判斷，避免非圖片被標為 `<media:image>`。
- **WhatsApp**：onboarding allowlist 會正規化為 WhatsApp digit-only phone ids；`@newsletter`／Channel target 與群組 visible reply mode 行為延續修復。
- **Matrix／Mattermost／Teams／Feishu**：progress tool status 與 quiet preview／draft preview 行為更接近共用 channel-streaming formatter。

### Google Meet／Voice Call／Realtime

- **agent-mode 預設**：Google Meet 的 Chrome talk-back 預設為 `mode: "agent"`／`realtime.strategy: "agent"`，即 STT → OpenClaw agent → TTS；direct realtime bidirectional 行為保留為 `bidi`。
- **Twilio realtime**：音訊 pacing、queue bound、backpressure close、barge-in clear、post-DTMF speech delay、silent join、mic readiness wait 等路徑被修正。
- **provider split**：realtime provider 設定拆成 agent-mode transcription provider 與 bidi-mode voice provider；legacy Gemini Live bidi config 可由 doctor 遷移。
- **echo suppression**：抑制 assistant 自己的 playback／transcript echo，避免會議把代理輸出誤當新使用者 turn。
- **OpenAI／realtime failure**：provider readiness 前 socket close 會標為 closed-before-ready，不再誤報 timeout。

### 模型／提供者／媒體

- **`models auth list`**：新增 `openclaw models auth list [--provider <id>] [--json]`，可檢視每 agent auth profiles 而不輸出 secrets。
- **Codex audio transcription**：Codex runtime／manifest metadata 宣告 audio transcription，並將 active Codex chat model 路由到 OpenAI transcription default。
- **OpenRouter**：新增 opt-in response caching headers，並擴大 app-attribution categories。
- **DeepSeek／OpenRouter**：DeepSeek V4 reasoning effort policy surface 修正，避免無效 reasoning effort。
- **OpenAI direct Responses**：direct OpenAI Responses 預設 SSE，避免 WebSocket path 在部分環境卡住。
- **媒體 Windows durability**：Windows attachment temp files 改以 read/write 開啟再 fsync；`EPERM` 也可視為 best-effort，降低 Windows WebChat／channel upload 失敗率。

### Windows／安全／代理

- **Windows host env 防護**：阻擋 workspace `.env` 覆寫 `SystemRoot`／`WINDIR`／`LOCALAPPDATA` 等會影響 Windows helper resolution 的值。
- **`cmd.exe`／`reg.exe` 固定解析**：`.cmd`／`.bat` wrapper、registry probe、ACL helpers 都走可信 Windows install root，不信任 workspace 注入路徑。
- **Browser current-tab SSRF**：tab-scoped debug/export/read/screenshot/snapshot/storage 等 route 在讀取既有 tab 前先執行 URL navigation policy。
- **Managed proxy**：debug proxy 在 managed proxy 啟用時預設禁止 direct upstream／CONNECT，APNs HTTP/2 direct delivery 也會走 active managed proxy。
- **Sandbox Windows**：Docker bind source 接受 Windows drive-absolute path，同時維持 blocked-path／allowed-root 比對大小寫不敏感。

---

## 4. 2026.5.4 — Fixes（精選分組）

### 安裝／更新／doctor

- package update plugin sync 失敗時，失敗插件會被停用／跳過，不讓單一壞插件把 core package update 判為失敗。
- `doctor --fix` 保留 active `auth.profiles` metadata，修復舊 `<provider>:default` metadata 時不破壞仍被模型 fallback 或 `model@profile` 使用的資料。
- `doctor --fix` 會修復 `plugins.allow` only 的官方插件、configured plugin repair、channel owner plugin repairs、stale session route state、group config drift migrations。
- npm script shell、post-core package child、npm-prefix staging、beta npm fallback、stable correction version ordering 等 update path 均有補強。

### Codex／OpenAI

- Codex usage-limit reset details 會出現在聊天回覆；API 或 subscription 限制不再造成 Telegram 等通道靜默。
- OpenAI Codex app-server turn 會遵守 `auth.order.openai-codex`，bound `/codex bind` session 保留 Codex-native OAuth profile。
- stale bound app-server thread 會重建一次，保留 auth profile 與 turn overrides 後重試。
- Codex app-server protocol mirror 與 generated TypeScript formatting 對齊，避免 drift check 失敗。

### 通道交付

- Discord dropped final reply 會失敗；非文字 payload scrub 不再吃掉可見 Discord labeled replies。
- Slack thread participation、rich progress line trimming、toolProgress quiet preview、Socket retry error wording 持續硬化。
- Telegram topic `requireMention` precedence、topic dispatch runtime、media placeholder、button-only interactive replies、long final preview reuse 持續修復。
- WhatsApp login QR 經 runtime、login outcome output、allowlist number canonicalization、group visible reply mode 皆有修復。
- Google Chat auth response headers 正規化，修復 google-auth-library 讀取 `cache-control` 的相容性。

### 記憶／Active Memory

- `corpus=all` 記憶搜尋會限制每 corpus 結果數，避免 memory-hit starvation。
- Active Memory 在沒有 memory tools 註冊時優雅跳過 sub-agent；session-store channel 含 `:`（例如 QQ c2c id）時不再誤進 plugin `dirName` 驗證。
- memory-core plugin runtime deps 補入 `json5`，讓 packaged `memory_search` sandbox 可解析 generated OpenClaw runtime chunks。
- timeout partial recovery、memory-only recall latency、Windows path／lock 測試都有補強。

### UI／WebChat／Control UI

- Control UI header 顯示 active agent；Cron New Job sidebar 可摺疊；chat session picker 以 agent-first filter 改善多 agent 導航。
- Chat controls/composer 跨手機／平板／桌面寬度重新打磨；滾動時隱藏 control row；連續重複文字可合併成帶 count 的 bubble。
- Dream Diary markdown、text-block tool results、expanded tool output scroll、Talk session retry／stop／startup error dismiss 均有修復。
- Dashboard manual token fallback、long animation frame／long task event log、responsiveness diagnostics 加入除錯路徑。

### Release／CI／QA

- Windows Blacksmith Testbox、Windows package upgrade、cross-OS release checks、plugin publish workflow、ClawHub publish rate limit／runtime preflight 持續硬化。
- Mantis Slack desktop smoke、desktop screenshots、runtime env forwarding、Testbox lease id 支援納入 QA 工具鏈。
- release validation 拆分 soak validation，避免單一路徑過長；Package Acceptance／plugin SDK API baseline／runtime postbuild checks 更嚴格。

---

## 5. 繁體文件同步重點

本次已將繁體技術文件的主版本標記由 **2026.5.3** 推進到 **2026.5.4**，並在以下主題加入對齊補充：

- **專案概覽**：新增 2026.5.4 release 摘要，涵蓋 Meet voice、Gateway startup、plugin update、streaming、Codex、Windows security。
- **Gateway**：新增 2026.5.4 啟動效能、Windows loopback、diagnostics、safe restart、sessions list 等段落。
- **Agent**：新增 post-compaction loop guard、runtime context 隔離、tool allowlist、bootstrap hook、subagent／messaging final delivery 修復。
- **通訊頻道**：新增 Slack rich progress、Discord final delivery、Telegram long final reuse、WhatsApp allowlist、Google Meet agent-mode 等段落。
- **插件開發**：新增 official externalized plugin hints、beta install、peer link repair、Plugin SDK session slots、runtime entry cleanup。
- **配置／模型／記憶／TUI**：新增 `tools.loopDetection.postCompactionGuard.windowSize`、`models auth list`、Codex audio transcription、Active Memory colon channel guard、TUI stale response copy／long-token sanitizer 等補充。

---

## 6. 摘要統計

| 項目 | 撰寫時量測／約定 |
|------|------------------|
| **產品行為權威來源** | **`CHANGELOG.md` → `## 2026.5.4`** |
| **`package.json` version** | **`2026.5.4`** |
| **起點 tag** | **`v2026.5.3`** → **`06d46f7cf6`** |
| **終點 tag** | **`v2026.5.4`** → **`325df3efef`** |
| **提交範圍** | **`v2026.5.3..v2026.5.4`** |
| **總提交數** | **527** |
| **非合併提交數** | **522** |
| **差異檔案數** | 約 **1437** |
| **上一輪使用者對齊點** | **`v2026.5.3`**／**`commit-analyze-20260503.md`** |
| **本輪繁體文件主版本** | **`2026.5.4`** |

*若本地 checkout 或 tag 拓扑不同，請以實際 `git describe`、`git rev-parse "v2026.5.4^{}"` 與 `CHANGELOG.md` 頂段為準修正表頭。*
