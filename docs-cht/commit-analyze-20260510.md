# OpenClaw 提交分析報告 — 2026-05-10

> **分析範圍**：以上一輪繁體文件對齊點 **stable `v2026.5.9`（annotated tag 剝離後提交 `eeef4864494f859838fec1586bedbab1f8fa5702`）** 之後，至本機目前 **`HEAD`**（含 **`v2026.5.10-beta.6`** 定版線與其後的繁中文件同步提交）所累積的變更。產品行為以英文 **`CHANGELOG.md`** 與對應程式碼為準；本文件聚焦 **2026.5.10** 這一輪的 **使用者可見行為、架構歸因、以及繁體文件同步重點**，不以逐筆羅列 3,000+ 筆提交為目標。
> **版本對齊**：`package.json` 目前為 **`2026.5.10-beta.6`**；`git tag v2026.5.10-beta.6` 存在，但 **`HEAD` 本身**已包含一筆後續的繁中文件同步提交 **`050e5656ec`**。若要對齊發行定版，請以 **`af86c5bf6a`**（`v2026.5.10-beta.6`）為 release 參照點。
> **Git 規模說明**：自 **`v2026.5.9-beta.1..HEAD`** 在典型 **`develop`** 合併拓撲下可達 **3,242** 筆提交，`git diff --shortstat` 亦可能顯示 **5,260** 檔、約 **+248,799 / -100,134** 行的量級。這反映的是 **長分支合併 + release prerelease + docs 同步** 的統計表象；真正應關注的是 **release 主題** 與 **目前行為**。

## 目錄

1. [版本切點與本輪性質](#1-版本切點與本輪性質)
2. [2026.5.10 — 主要主題](#2-2026510--主要主題)
3. [繁中技術文件同步重點](#3-繁中技術文件同步重點)
4. [本輪對齊統計](#4-本輪對齊統計)

---

## 1. 版本切點與本輪性質

```text
v2026.5.9-beta.1
  → 2026.5.10-beta.4 / beta.5 / beta.6 release 線
  → 繁中技術文件同步提交
```

本輪的性質比較像是 **5.9 大型能力發行後的穩定化 / 收尾 / 文件追齊**，而不是再開一個新的能力爆炸期。提交主題大致可分成以下幾類：

- **Telegram 會話上下文修復**：把 reply chain cache、live chat context、session transcript mirror 與 command menu cache key 重新整理成一條更一致的資料流。
- **Discord realtime voice 穩定化**：修正 voice proxy、native event stream 與 realtime session 的輸入輸出邊界，減少 voice turn 被中途打斷或誤判失敗。
- **新通道插件 ClickClack**：新增一個完整的 first-class channel extension，從 account、target、inbound/outbound 到 gateway bridge 都有自己的實作與測試。
- **Agent / subagent 契約演進**：加入 subagent delegation preference mode，讓主 agent 的提示語氣可以更明確地偏向分派慢工；同時把 runtime 身分注入 system prompt。
- **技能與安裝信任邊界**：允許受信任的 skill symlink 目標，讓刻意設計的 sibling-repo skills 仍可載入，但不放寬到任意上層目錄。
- **Gateway / session / memory 修補**：restart continuation authority、retry budget、atomic reindex cleanup、delayed queue recovery 等都在這一輪收斂。
- **Codex / app-server / harness**：dynamic tool timeout、native tool observability、OAuth harness auth 與 stream setup timeout 的行為補強，讓長回合和嵌入式工具更穩。

---

## 2. 2026.5.10 — 主要主題

### 2.1 Telegram：reply chain、live context、session transcript

這一輪 Telegram 修補不是單點 bug，而是把 **「目前回合到底該看哪個上下文」** 的規則重新收斂：

- **local chat context windows**：針對單一 chat / thread 建立更清楚的 context window，避免舊訊息與目前互動窗口互相污染。
- **live chat context 優先**：當 live context 與其他回溯資料同時存在時，優先使用正在發生的那條訊息鏈。
- **outbound reply mirror**：把 outbound replies 回寫到 session transcript，讓之後的重播、replay 與 debug 能看到實際送出的內容。
- **prompt context cleanup**：清掉不該進 prompt 的 window metadata，避免 reply target 或 cache note 混進模型輸入。
- **command menu cache keys**：修正 menu cache key 後，native command menu 的解析更不容易因路由殘留而失真。

這些變動的共同目標，是讓 Telegram 的 **「看見什麼 → 送出什麼 → 記錄什麼」** 三者一致。

### 2.2 Discord realtime voice：proxy 與 stream 穩定化

Discord 這輪重點不是擴功能，而是把 realtime voice 交付鏈路修穩：

- **voice proxy 穩定化**：realtime voice proxy 的中繼和回傳狀態更可靠，不再那麼容易在 session 切換或 retry 時失去同步。
- **native event streams**：preserve 原生 event stream，避免長回合把可觀測事件切碎成不完整片段。
- **voice / talk 交界**：voice turn、talk runtime 和 provider event 的邊界更明確，減少 barge-in 與 playback reset 的 race。

實務上，這類修正會直接影響 Discord 上的語音助理是否能「講完、講對、講得可追蹤」。

### 2.3 ClickClack：新 first-class channel extension

`clickclack` 是本輪最明確的新擴充頻道：

- **bot-token channel**：以 ClickClack bot token 連線，支援獨立 service bot 與 user-owned bot。
- **多 account / 多 workspace**：每個 account 各自持有 token、workspace、default target 與 agent id。
- **target 語意**：`channel:<name-or-id>`、`dm:<user_id>`、`thread:<message_id>` 的路由模式明確化。
- **plugin surface**：這個頻道不是臨時 glue code，而是完整的 extension package、gateway bridge、HTTP client、inbound/outbound、以及測試矩陣。

對文件的意義是：OpenClaw 的頻道生態已經進一步從「內建通道」走向「可獨立發布的 first-class channel plugin」。

### 2.4 Agent / subagent：delegation mode 與 runtime identity

Agent 線的核心變更集中在兩件事：

- **subagent delegation mode**：`agents.defaults.subagents.delegationMode: "prefer"` 只改提示層級，不是權限層級的強制。它會鼓勵主 agent 把較慢、較長的工作交出去，但不會改寫工具政策。
- **system prompt 注入 runtime identity**：目前 provider / model identity 會進入 system prompt，讓模型自我敘述、工具選擇和真正 runtime selection 對得起來。

搭配既有的 current-turn context unify，這輪把 **「主 agent 在講誰、子 agent 在做誰的活、當前回合看哪份上下文」** 的邊界整理得更清楚。

### 2.5 Skills / trust boundary：symlink targets

`skills.load.allowSymlinkTargets` 的引入很實際：

- 允許被明確信任的實體目錄作為 symlink 目標。
- 保留 sibling repo 或工作區外掛 skills 的既有布局。
- 仍維持 containment boundary，不把任意上層根目錄放進來。

這是典型的 **「可用性提升，但不放寬安全模型」** 的修法。

### 2.6 Gateway / session / memory

Gateway 與基礎設施層這輪主要在清 race 與清殘留：

- **restart continuation authority**：重啟流程中的 continuation 權限不再在 shutdown / retry race 中失真。
- **restart retry budget**：重啟續接有明確 retry budget，不會無限搶救。
- **in-process restart reread**：重啟後重新讀配置，避免用舊 snapshot 繼續跑。
- **atomic reindex cleanup**：memory reindex 的 cleanup 路徑更健壯，避免壞狀態留在索引流程。
- **delivery queue recovery**：session delivery queue 的 recover 與 storage 邏輯更一致。

這些都屬於典型的「使用者未必看得見，但出問題就很痛」的穩定化修補。

### 2.7 Codex / app-server / harness

Codex 線這輪較像是 **執行層的邊界修復**：

- **dynamic tool timeouts**：原本可能被過短或錯置的 timeout 卡死的 dynamic tool 現在會被正確尊重。
- **native tool observability**：native tool completions 有更完整的觀測資料，讓 watchdog 不會把長工具誤判成 stalled embedded run。
- **OAuth harness auth**：隔離式 app-server harness 的登入與 refresh key 更穩。
- **stream setup timeout**：stream setup 還沒真正起來時就先超時的情境，會以更明確的方式失敗。

這代表 Codex / app-server 不只是「能跑」，而是開始要求 **觀測、超時、完成語意** 都要一致。

### 2.8 Release / build / CI

本輪還包含不少 release 和 build 基礎維護：

- beta release 的 verify / pretag / parse error flow 被修得更穩。
- canvas bundle hash、generated protocol models、release config baseline 持續刷新。
- docs-cht 同步提交本身也屬於這一段的最終產物。

---

## 3. 繁中技術文件同步重點

本輪繁中技術文件不應只改版本號，還要補下面幾個面向：

- **專案概覽 / README**：版本線改為 `2026.5.10-beta.6`，並補上 ClickClack、新 subagent delegation mode、Telegram / Discord / Gateway 穩定化摘要。
- **Gateway**：加入 restart continuation authority、retry budget、in-process config reread 與 session delivery queue recovery 的敘述。
- **Agent / Subagents**：補 `agents.defaults.subagents.delegationMode`、runtime identity 注入 system prompt、current-turn context unify。
- **通訊頻道**：新增 ClickClack 頻道，並補 Telegram reply-chain cache / live context / transcript mirror、Discord realtime voice proxy 穩定化。
- **配置系統**：補 `skills.load.allowSymlinkTargets`，讓受信任 symlink skills 的行為有明確文件。
- **Onboard / Skills**：同步介紹 ClickClack 與受信任 skills symlink 目錄的安裝與設定敘事。
- **Memory**：補 atomic reindex cleanup 的實際影響，避免繁中記憶文件仍只寫到較舊的 reindex 修補。

---

## 4. 本輪對齊統計

| 項目 | 結果 |
|------|------|
| **起始對齊點** | `v2026.5.9-beta.1`（`eeef4864494f859838fec1586bedbab1f8fa5702`） |
| **release 參照點** | `v2026.5.10-beta.6`（`af86c5bf6a`） |
| **本機 `HEAD`** | `234afa3a47fb40495b153175eaa589bc4474bd7b` |
| **`v2026.5.9-beta.1..HEAD` 提交數** | `3,242` |
| **`git diff --shortstat`** | `5,260` files；`+248,799 / -100,134` lines |
| **主要變更類型** | Telegram reply context、Discord realtime voice、ClickClack channel extension、subagent delegation mode、skills symlink trust、Gateway restart/session hardening、Codex harness timeout/observability |
| **本輪繁體文件主版本** | `2026.5.10-beta.6` |

*若要對照 2026.5.10 之前的穩定化內容，仍可回讀 `commit-analyze-20260509.md`；但從目前版本線起算，應以本檔與 `CHANGELOG.md` 及實際程式碼為準。*
