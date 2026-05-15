# OpenClaw 提交分析報告 — 2026-05-12

> **分析範圍**：以上一輪繁體文件對齊點 **`v2026.5.10-beta.6`** 之後，至目前 **`HEAD`**（含 **`v2026.5.12`** 定版線與其後的繁中文件同步提交）所累積的變更。產品行為以英文 **`CHANGELOG.md`** 與對應程式碼為準；本文件聚焦 **2026.5.12** 這一輪的 **使用者可見行為、契約變動、以及繁中技術文件同步重點**，不以逐筆羅列千餘筆提交為目標。
> **版本對齊**：`package.json` 目前為 **`2026.5.12`**；`git tag v2026.5.12` 已存在。若要對齊發行定版，請以 **`v2026.5.12`** 為 release 參照點。
> **Git 規模說明**：自 **`v2026.5.10-beta.6..HEAD`** 在典型合併拓撲下可達 **1,523** 筆非合併提交，`git diff --shortstat` 亦可能顯示 **2,288** 檔、約 **+100,202 / -12,922** 行的量級。這反映的是 **長分支收斂 + release prerelease + 文件同步** 的統計表象；真正應關注的是 **release 主題** 與 **目前行為**。

## 目錄

1. [版本切點與本輪性質](#1-版本切點與本輪性質)
2. [2026.5.12 — 主要主題](#2-2026512--主要主題)
3. [繁中技術文件同步重點](#3-繁中技術文件同步重點)
4. [本輪對齊統計](#4-本輪對齊統計)

---

## 1. 版本切點與本輪性質

```text
v2026.5.10-beta.6
  → v2026.5.12 beta / stable release 線
  → 繁中技術文件同步提交
```

本輪的性質比起上一版更偏向 **契約整理 + 依賴收斂 + 具體 UX 修補**。它不是單一大功能點爆發，而是把多個容易被誤讀的邊界補齊：

- **供應器與插件外部化**：Bedrock、Bedrock Mantle、Slack、OpenShell sandbox、Anthropic Vertex 進一步從核心依賴圖中抽離，讓未安裝對應插件時不會無端拉進 AWS / provider runtime cone。
- **Control UI / WebChat 自動捲動模式**：把「跟隨串流」「近底自動維持」「手動新訊息按鈕」拆成可持久化選項，減少長串回合時 UI 行為不穩。
- **ACP 退路策略**：`acp.fallbacks` 讓 ACP turn 在主後端暫時不可用時，能在未輸出前換到備援 runtime backend。
- **Telegram ingress / transcript 穩定化**：把 Bot API 輪詢搬到隔離 worker，補上 spool 與 HTML 保真，並修正群組媒體下載前的 mention gate。
- **Agent / subagent 模型優先序**：`agents.defaults.subagents.model` 在 `sessions_spawn` 的套用順序比 target agent primary model 更前，讓具名 runtime 真的能跟著預設子回合走。
- **Gateway / session / transcript 一致性**：live 更新保留單調序列，SSE 歷史在 stale 輸入下重新整理，避免增量 state 被壞資料污染。
- **安全與配置解析**：`message` 工具權限、provider SecretRef、Windows home roots、provider env marker inference 都在這輪收束，目標是讓 doctor 與 runtime 對同一件事說同一套話。

---

## 2. 2026.5.12 — 主要主題

### 2.1 供應器與插件外部化：從「核心內建」轉向「按需安裝」

這輪 changelog 最具結構意義的變更之一，是把幾個原本還會被核心安裝帶入的 runtime cone 再切薄：

- **Amazon Bedrock**：Bedrock 與 Bedrock Mantle provider packages 外部化，沒有安裝對應提供者時，核心不再默默拉入 AWS SDK 依賴。
- **Slack / OpenShell sandbox / Anthropic Vertex**：這三者也跟著外部化，將 runtime 依賴縮回各自插件。
- **Provider discovery**：provider discovery 開始讀取 `setup.providers[].envVars` 所宣告的 credentials，同時保留舊的 `providerAuthEnvVars` fallback。

這代表文件敘事要從「這些能力存在於核心」改寫成「這些能力是可選插件，核心只保證 contract」。

### 2.2 Control UI / WebChat：自動捲動不再只有單一行為

這版把串流 UI 的滾動策略明確產品化：

- **persisted auto-scroll mode**：使用者可在近底自動維持、完全跟隨串流、或關閉自動跟隨之間切換。
- **manual new-messages affordance**：在關閉自動跟隨時，UI 以新訊息按鈕維持可見性，而不是強迫畫面跳動。
- **長回合可預期性**：這不是單純的視覺調整，而是讓 WebChat / Control UI 在大輸出和卡頓情境下仍能維持可讀性。

文件上要把它理解成一個 **持久化行為設定**，而不是單次 UI 快捷鍵。

### 2.3 ACP：先試備援，再輸出

`acp.fallbacks` 的語意很明確：

- 先選主 ACP runtime backend。
- 如果主後端不可用，且 turn 尚未輸出，則改走配置好的備援後端。
- 不把這件事做成靜默重試黑箱，而是保留 turn 的可觀測性。

這讓 ACP 比較像一個 **有退路的執行面**，而不是單點失敗即中止的直連協議。

### 2.4 Telegram：隔離 ingress worker、spool 與 HTML 保真

Telegram 的修補這輪有三個核心點：

- **隔離 worker**：Bot API polling 不再跟主 event loop 綁死，主執行緒 stall 時還能維持 ingress。
- **durable local spool**：輪詢結果先落本地 spool，減少短暫卡頓造成的訊息遺失。
- **HTML rendering 保真**：lazy cron announce delivery 會保留 rendered HTML，避免 Markdown link 退化成 literal anchor tag。

另外，群組媒體在 `requireMention` 啟用時會先做 mention gate，再決定是否下載，這能避免「本來就不該回覆」的訊息先消耗下載成本。

### 2.5 Agent / subagent：模型優先序與 runtime 對齊

這輪最值得注意的 agent 變化不是新工具，而是 **既有設定的套用順序**：

- `agents.defaults.subagents.model` 會先於 target agent primary models 套用。
- 這樣像 `claude-cli` 這類 runtime 才能在預設子回合裡保持正確掛載。
- 配合 `sessions_spawn`，子回合的 model scope 不會被後來的 primary model 決策覆蓋掉。

換句話說，文件裡對 subagent 的描述要更強調 **預設子回合** 與 **target agent 主模型** 是兩個不同層次，不可混寫成單一 model fallback。

### 2.6 Gateway / session history：單調序列與 stale refresh

Gateway 這輪沒有新增大面積路由，但修了幾個會讓歷史污染的邊界：

- **monotonic transcript sequence**：live update 要帶著單調遞增的 transcript message sequence。
- **stale SSE refresh**：當 SSE 歷史輸入過舊時，會先刷新再 append，而不是把壞的 incremental state 接進來。
- **session history consistency**：這使 session mirror、live update、replay 與歷史查詢更一致。

這類修復的價值在於：它們通常不是第一時間看得見，但一旦發生會讓整段回合的診斷失真。

### 2.7 安全、SecretRef 與錯誤語意

這輪還有幾個值得同步進繁中技術文件的收斂點：

- **message tool runtime grant**：doctor / Codex 不再把 runtime 已授權的 `message` 工具誤報成不可用。
- **provider apiKey 解析**：只透過結構化 SecretRef 解析 provider `apiKey`，不再靠寬鬆 env marker 猜測。
- **Windows sandbox roots**：`USERPROFILE` 被納入受阻擋 home roots，避免 `.ssh`、`.openclaw`、`.codex` 等在 Windows profile 下漏出。
- **Slack media redirect**：格式錯誤的 private-file `Location` header 會被當成不可跟隨的 redirect，而不是把媒體下載打死。
- **CLI help path**：bare plugin / parent command help 保持輕量路徑，不先做昂貴的 registry discovery。

---

## 3. 繁中技術文件同步重點

本輪繁中技術文件除了更新版本號，也要把以下契約寫進對應章節：

- **README / 專案概覽**：版本線改為 `2026.5.12`，並補上 Bedrock / Slack / OpenShell / Anthropic Vertex 外部化、Control UI 自動捲動、ACP 備援與 Telegram worker/spool 摘要。
- **Gateway**：補上 `acp.fallbacks`、transcript sequence、stale SSE refresh、session history 一致性。
- **Agent / Subagents**：補 `agents.defaults.subagents.model` 的套用順序，並把 runtime / target model 的優先序寫清楚。
- **通訊頻道**：補 Telegram ingress worker、HTML 保真與 mention gate；同步 Slack private-file redirect 與 Weixin catalog package bump 的外部可見性。
- **擴充套件開發**：補 provider 外部化與 `setup.providers[].envVars` discovery，說清楚 plugin 依賴 cone 已經收薄。
- **配置系統**：補 `acp.fallbacks`、SecretRef 驗證與 provider env marker 解析收斂。
- **CLI / Onboard**：補 provider-specific auth flags 的透傳，以及 plugin help 的輕量路徑行為。
- **模型 / 安裝 / TUI / 記憶**：把版本號換到 `2026.5.12`，並把與 release 行為直接相關的修補補齊。

---

## 4. 本輪對齊統計

| 項目 | 結果 |
|------|------|
| **起始對齊點** | `v2026.5.10-beta.6` |
| **release 參照點** | `v2026.5.12` |
| **本機 `HEAD`** | `6a3cfe884f` |
| **`v2026.5.10-beta.6..HEAD` 提交數** | `1,523` |
| **`git diff --shortstat`** | `2,288` files；`+100,202 / -12,922` lines |
| **主要變更類型** | Bedrock / Slack / OpenShell / Anthropic Vertex 外部化、Control UI/WebChat auto-scroll、ACP fallbacks、Telegram ingress worker/spool、subagent model precedence、session history sequence hardening |
| **本輪繁體文件主版本** | `2026.5.12` |

*若要對照 2026.5.10 之前的穩定化內容，仍可回讀 `commit-analyze-20260510.md`；但從目前版本線起算，應以本檔與 `CHANGELOG.md` 及實際程式碼為準。*
