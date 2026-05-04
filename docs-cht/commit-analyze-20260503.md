# OpenClaw 提交分析報告 — 2026-05-03

> **分析範圍**：以英文 **`CHANGELOG.md` → `## 2026.5.3`** 為**產品行為主軸**（對齊 **`package.json`** **`2026.5.3`**、**`git tag v2026.5.3`** → **`06d46f7cf638a31c4852c068aeeaa76f5e949941`**）。  
> **工作區快照**：撰寫時 **`HEAD`** = **`3a0936b45cd7f70d30f1849b85f72e59c72b2fbf`**（**`git describe`**：`v2026.5.3-43-g3a0936b45c`）；若本地 **`HEAD`** 領先 **`v2026.5.3`**，多為 **合併／文件／後續提交**，**發行行為仍以 changelog 切段與標籤為準**。  
> **上一輪繁體對齊**：release **`v2026.5.2`**／**`commit-analyze-20260502.md`**。**本期**為 **`2026.5.3`** 切段；**5.2 深度主題**請仍交叉 **`commit-analyze-20260502.md`**。  
> **`### Fixes` 篇幅說明**：**2026.5.3** 區塊下方 **`Fixes`** 列項極長，且與 **`2026.5.2`** 部份條目在版面上一併滾動（整合／承載既有修正敘述）；**實質新增語意請以 Highlights／Changes 與下方「Fixes 精選分組」為索引**，細節仍以 **`CHANGELOG.md` 原文**為準。  
> **方法**：**不按條列舉上千筆提交**，改以 **Highlights → Changes → Fixes（主題化繁中摘要）**。

---

## 目錄

1. [版本路徑與方法說明](#1-版本路徑與方法說明)
2. [2026.5.3 — Highlights](#2-202653--highlights)
3. [2026.5.3 — Changes（依主題）](#3-202653--changes依主題)
4. [2026.5.3 — Fixes（精選分組）](#4-202653--fixes精選分組)
5. [摘要統計](#5-摘要統計)

---

## 1. 版本路徑與方法說明

```
2026.5.2（上一發行／上一輪 docs-cht 主對齊點）
    → 官方 CHANGELOG 切段 ## 2026.5.3（本期主軸）
2026-05-03（使用者對齊日期）≈ CLI／套件 2026.5.3 與英文文件快照
```

---

## 2. 2026.5.3 — Highlights

- **檔案傳輸插件（bundled）**：paired node 二進位 **`file_fetch`／`dir_list`／`dir_fetch`／`file_write`**；每節點路徑 **預設拒絕**、運營核准、**預設拒絕 symlink**（可選 **`followSymlinks`**）、**單趟 16 MB** 上限（#74742）。  
- **插件安裝／解除／更新／onboarding**：強化 **官方插件**路徑；**ClawHub 後備**、**npm 依賴狀態回報**、**beta 通道更新**語意與 **externalized** 插件「一等套件」體驗對齊。  
- **Gateway／Control UI 效能**：啟動與熱路徑 **按需 lazy-load**——插件／runtime **discovery**、**cron**、**schema**、**shutdown**、**sessions**、**模型 metadata** 等僅在需要時載入。  
- **通道／回覆**：Discord **狀態 reaction** 與 **degraded transport** 回報改善；WhatsApp **Channel／Newsletter** 目標；Telegram、飛書、Matrix、Microsoft Teams、Slack **交付／復原**收緊。  
- **安裝／更新**：修復 **損壞的 macOS LaunchAgent** 升級路徑；**runtime 載入前拒絕 source-only 套件**；**過時 Gateway／插件狀態**於 **update／doctor** 可修。  
- **Agent／runtime 可靠性**：串流提供者回覆、**延遲的 A2A session 回覆**、prompt／tool 交付、**memory recall**、**web search provider discovery**、thinking／模型 metadata 等在常見邊角案例下 **更一致保留**。

---

## 3. 2026.5.3 — Changes（依主題）

### 串流／指令／Doctor

- **統一 progress draft**：**`streaming.mode: "progress"`**、自動單字狀態標籤；**Discord／Telegram／Matrix／Slack／Microsoft Teams** 共用進度設定面。  
- **`/steer <message>`**：session **閒置**時對 **進行中 run** **佇列外轉向**（#76934）。  
- **`/side`**：**`/btw`** 的 **文字與原生 slash** 別名。  
- **`doctor --fix`**：即使其他驗證仍失敗（例如缺插件），仍 **提交安全 legacy migration**，確保 **`agents.defaults.llm`** 等已知鍵可被清理（#76798、#76800）。

### Gateway／設定／效能

- **無效設定**：啟動／熱重載 **fail closed**，**不**自動還原；**`doctor --fix`** 負責 last-known-good。  
- **延後載入**：早期 runtime discovery、**shutdown-hook**、**cron**、**通道設定 schema metadata**、**restart sentinel**、**maintenance timer** 等多類工作在 **ready 之後**；削減 **plugin auto-enable** 重複工作；附 **startup CPU／profile** 控制（見 changelog）。

### 工具／沙盒／Discord

- **PDF／media factories**：若 effective **`tools.deny`** 已阻擋，則 **跳過**對應 factory 熱路徑設定（#76773）。  
- **Sandbox registry**：容器／瀏覽器登錄改 **每 runtime shard 檔**，降鎖競爭（#74831）。  
- **Discord status reaction**：**顯式** reaction tool 可 **`trackToolCalls: true`** 追蹤後續工具進度；共用 **emoji 對照表**；**degraded transport／event-loop starvation** 進 **status**（#76327）。

### 插件／Onboarding／CLI

- **Manual setup**：可裝 **選用官方插件**（ClawHub＋npm fallback）、**外部 Codex 插件**可選。  
- **`plugins list --json`**：**套件依賴安裝狀態**；信任 **official externalized npm migrations**；清理 **stale bundled load paths**；**beta OpenClaw 通道**下插件更新 **先試 `@beta`**。  
- **ClawHub 429**：附 **重置視窗**與 **未登入可提高額度** 提示。

### QA／維運

- **Mantis Discord smoke**：**`pnpm openclaw qa mantis discord-smoke`** 與 **手動 GitHub workflow**（guild／channel／smoke／reaction／artifacts）。  
- **Slack live transport QA**：canary／mention-gating（私密 bot-to-bot harness）。

---

## 4. 2026.5.3 — Fixes（精選分組）

> **完整列表**請以 **`CHANGELOG.md` → `## 2026.5.3` → `### Fixes`** 為準；下列為 **閱讀索引**，涵蓋 **5.3 切段中新強調或反覆出現的修復族**。

### Web／proxy／fetch

- **`web_fetch`**：**late-bind** 設定與 provider fallback，對齊 **`web_search`** 語意（避免長生命周期工具用到過時快照）。  
- **`tools.web.fetch.useTrustedEnvProxy`**：可選，proxy-only DNS 環境（預設仍維持嚴格 hostname 政策）。  
- **Managed HTTPS proxy**：保留 **目標 TLS SNI／hostname 驗證**（CONNECT 不生動態將憑證校驗對到本機 proxy）（#74809）。

### 插件／發佈／scanner／ClawHub

- **已安裝 global 套件**：來源 TS runtime **檢查降級為警告**，允許 **jiti** 後備；**新安裝** source-only **仍拒絕**。  
- **ClawHub resolver**：接受 **`kind`／`sha256`** 與 **`artifactKind`／`artifactSha256`** 兩種欄位命名。  
- **Release**：無 **`dist/*.js`** 的 TS-only **code-plugin** 拒絕；publish **後驗證 tarball runtime entries**。  
- **npm install**：自 **managed npm-root manifest** 執行，避免 **兄弟 `@openclaw/*` 被 prune**（#76571）。  
- **Integrity**：安裝 **釘選解析後版本**，拒絕 **lock integrity drift**。  
- **Matrix／Mattermost**：**維持同捆於 core**，在未 cut-over 前 **不**對外宣傳為獨立 npm 安裝（與 changelog 末尾策略一致）。

### Google Meet／Realtime／語音橋

- **Realtime**：socket 在 **`session.updated`** 前就關閉 → **closed-before-ready**（勿標成連線逾時）；Meet join **等待 bridge 真正就緒** 才返回／播音。  
- **Chrome**：對 **實際 Meet 分頁**授權媒體；**本地 realtime 音訊橋**在加入後才啟動；**BlackHole** 靜音／無輸出類問題硬化。

### 串流設定 metadata／通道

- **`streaming.progress.label`／`labels`／`maxLines`／`toolProgress`**：進 **bundled 通道設定 metadata**（設定／文件／Control UI 可見）。  
- **Mattermost**：接受 **`channels.mattermost.streaming`**；**`streaming: off`** 關 preview；progress **toolProgress=false** 時緊湊工具列隱藏。  
- **Microsoft Teams**：progress draft **tool lines** 與 **`toolProgress=false`** 語意對齊。  
- **Discord**：progress draft **boundary callback** 維持綁定；SecretRef bot token **runtime snapshot**（#76987）。

### Telegram／Feishu／Slack／WhatsApp

- **`channels.telegram.mediaGroupFlushMs`**（全域／帳户）調 album 緩衝。  
- **`channels.feishu.blockStreaming`**：**頂層與每帳戶**皆可設定並生效；**同 chat 序列佇列** **逾時釋放**（預設 5 分）避免後續訊息永久 **queued**。  
- **Slack**：Socket Mode **pong-timeout** 重連 **收斂日誌**；**persist sent-message markers** 跨重啟（#75585）。  
- **WhatsApp**：**`onlyBuiltDependencies`** 允許 **`@whiskeysockets/libsignal-node`**（#76539）；群組 **@提及 metadata**（LID／token 邊界）；提及 token **詞邊界**需求。

### Memory／Active Memory／向量

- **LanceDB**：bundled memory 套件宣告 **`apache-arrow`** peer（#76910）。  
- **`memory status`**：**cheap path** vs **`--deep`／`--index`** 分工（#76769）；sqlite-vec 就緒與 **embedding provider** 診斷拆分。  
- **索引**：旋轉／刪除之 **archive transcript** 仍可搜尋並映射回 live stem（#56131 脈絡）。

### Gateway／sessions／Cron／更新

- **`sessions.list`**：**輕量列**——界限 title／preview hydrate、快取 manifest model-id normalize；chat **`sessions.changed`** **避免全表 reload**（#76676）。  
- **`/v1/responses`**：**單回合多個 client tool call** 全數透出（#52288）。  
- **Cron**：lazy startup **競態**修正；**manual `cron.run` ID** 保留於歷史；**`jobs-state.json`** 修復後 **persist**。  
- **更新**：Control UI **全域套件更新**後先 **`doctor --non-interactive --fix`**；**self-update** 不可在 **進行中之 gateway 行程樹**內發起（#75691）；**continuationMessage** 自 **`update.run`** 帶入重啟 sentinel（#71178 脈絡）。

### Control UI／CLI／Codex

- **Skills modal**：**`showModal()`** 延後至 dialog **connected**（修正全瀏覽器失敗）。  
- **Talk（Realtime WebRTC）**：瀏覽器 offer **剔除**僅伺服器端標頭（**`originator`／`version`／`User-Agent`**），通過 OpenAI CORS（#76435）。  
- **`openclaw logs --follow`**：瞬斷 **自動重連**（#74782、#75059、#75372）。  
- **Codex WhatsApp**：message-tool 交付時保留 **`message` dynamic tool**（#76660）。  
- **`/think`**：作用中 session 模型與預設不同時仍見 **provider-specific** 階梯（#76482）。

### Doctor／devices／配置

- **devices approve**： pairing 拒絕後可 **`operator.admin`** **重試裝置核准**（#76956）。  
- **`config unset array[index]`**：只移除 **指定索引**（#76290）。  
- **`messages.visibleReplies`**：**boolean → 文件化 enum** coerce（#75390）。  
- **`.clobbered.*` forensic**：每路徑 **上限**＋寫入序列化（#76454）。

---

## 5. 摘要統計

| 項目 | 撰寫時量測／約定 |
|------|------------------|
| **產品行為權威來源** | **`CHANGELOG.md` → `## 2026.5.3`** |
| **`package.json` version** | **`2026.5.3`** |
| **`git tag`** | **`v2026.5.3`** → **`06d46f7cf6`** |
| **`HEAD`（撰寫時）** | **`3a0936b45c`**（可能 **領先標籤**） |
| **`v2026.5.2..HEAD` 非合併提交數（參考）** | **約 561**（含標籤後客製合併／分支工作時會膨脹） |
| **上一輪使用者對齊點（自述）** | **`v2026.5.2`**／**`commit-analyze-20260502.md`** |
| **`docs-cht` 版本標註（commit-analyze 除外）** | **`2026.5.3`** |

*若你的環境 **`HEAD`／標籤拓扑**不同，請以實際 **`git describe`**／**`CHANGELOG` 頂段標題**為準並自行修正本檔案表頭。*
