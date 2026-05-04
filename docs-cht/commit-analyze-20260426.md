# OpenClaw 提交分析報告 — 2026-04-26

> **分析範圍**：從 `71de5b07391ce02a7bf57f5ebe0721a507f9a7b3`（doc: 更新繁體中文文件版本號至 2026.4.25）至工作區 `HEAD`（`9369ef188cf0780e2e1dc0725d4ce063c2763017`；含 `chore(release): prepare 2026.4.26 stable`、`docs: consolidate 2026.4.26 changelog` 等），歷史區間內共 **829 個非合併提交**。  
> **參考**：`docs-cht/commit-analyze-20260425.md`；本輪以 **官方 CHANGELOG**（`## 2026.4.26` 於檔案靠前區段之 **Changes／Fixes**；較舊的 **`## 2026.4.25`／`## 2026.4.24`** 完整條目仍保留於檔案後段供對照）為主軸彙整；未逐條展開全部提交訊息。

---

## 目錄

1. [版本路徑與方法說明](#1-版本路徑與方法說明)
2. [2026.4.26 — Changes（依主題）](#2-2026426--changes依主題)
3. [2026.4.26 — Fixes（精選主題）](#3-2026426--fixes精選主題)
4. [插件 metadata／啟動效能（橫切）](#4-插件-metadata啟動效能橫切)
5. [基礎建設與發布](#5-基礎建設與發布)
6. [摘要統計](#6-摘要統計)

---

## 1. 版本路徑與方法說明

```
2026.4.25  → 上一次繁體中文技術文件對齊終點（TTS、冷 registry、OTEL、瀏覽器／Control UI、大量修復）
2026.4.26  → Stable：QQBot 群組與引擎重構、元宝／Yuanbao 頻道、Talk 瀏覽器 realtime、manifest 驅動模型／定價、openclaw migrate、Matrix E2EE setup、Compaction 預檢、插件安裝／Windows／記憶體大量硬化
```

**為何以 CHANGELOG 為主？**  
八百餘筆提交含 **插件 metadata 快照循環破除**、**provider discovery 重用**、release baseline／CI／Docker／Windows updater；與先前報告相同，**產品 changelog + 主題分類**最能對齊英文文件與可感知行為。

---

## 2. 2026.4.26 — Changes（依主題）

### 通道

- **QQBot**：完整 **群組聊天**（歷史、`@` 提及 gate、啟用模式、每群組設定、**FIFO 佇列**與 deliver debounce）；C2C **`stream_messages`** 串流與 **`StreamingController`**；大檔 **chunked `sendMedia`**；引擎拆為 **pipeline stages**、outbound 子模組、內建 slash 模組、**`createEngineAdapters()`** 顯式 DI（#70624）。  
- **Yuanbao（騰訊元宝）**：官方目錄註冊 **`openclaw-plugin-yuanbao`**、合約測試與社群文件；新增 **`docs/channels/yuanbao.md`**（WebSocket bot 私訊／群組）（#72756）。

### Talk／即時語音與 Gateway

- **Control UI／Talk**：通用 **瀏覽器 realtime transport** 合約；**Google Live** 瀏覽器會話（受限 **ephemeral token**）；Gateway **relay** 支援僅後端存在的 realtime voice 插件。

### 模型列表與提供者

- **`models list`**（provider 過濾）：經 **explicit source plan** 組裝（使用者設定、installed manifest、Provider Index 預覽、scoped runtime fallback），維持 **權威順序**且不另堆 catalog cache。  
- **Cerebras**：bundled 插件（onboarding、靜態目錄、文件、manifest endpoint metadata）。

### 記憶／嵌入

- **OpenAI-compatible memory**：可選 **`memorySearch.inputType`／`queryInputType`／`documentInputType`**，支援 **非對稱嵌入端點**、direct query embedding、provider batch indexing（承接 #63313、#60727）。  
- **Ollama／memory**：對 **`nomic-embed-text`、`qwen3-embedding`、`mxbai-embed-large`** 等注入 **檢索 query 前綴**（文件批次不變）（#45013）。

### 插件／manifest／設定突變

- **模型 id 正規化**、endpoint host metadata、OpenAI-compatible request-family 提示、**catalog 別名／抑制**、OpenAI stale Spark 抑制、**可重用啟動 metadata 快照**：移入 **plugin manifest**，核心不再扛 bundled-provider 路由表或重複重建 manifest。  
- **插件設定**：直接 **load／write helpers 棄用**，改為 **傳入 runtime snapshot** + **具交易語意之 mutation helper**（restart 後續政策、scanner 護欄、runtime 警告、**revision 快取失效**）。  
- **安裝**：**`OPENCLAW_PLUGIN_STAGE_DIR`** 可含 **分層 runtime-dep 根**（唯讀預裝優先，再補裝至可寫最終根）（#72396）。

### Control UI

- **原始設定 pending diff**：解析 **JSON5**、敏感值 **redact** 至手動揭露、避免假 **raw-edit** callback（#39831）。  
- **Quick Settings** 網格：桌機／平板／手機 **對齊卡片**、減少橫向浪費。

### Matrix

- **`openclaw matrix encryption setup`**：一鍵啟用加密、bootstrap recovery、列印驗證狀態。

### Agent／Compaction

- **Opt-in**：**`agents.defaults.compaction.maxActiveTranscriptBytes`** **預檢觸發**本機 compaction；成功時需 **transcript rotation** 將後續回合寫入較小 **successor** 檔，而非純 byte-split 歷史。

### CLI 遷移

- **`openclaw migrate`**：**plan／dry-run／JSON**、遷移前備份、onboarding 偵測、僅 archive 報告；**Claude Code／Desktop** 與 **Hermes** 匯入器（設定、memory／插件提示、提供者、MCP、skills、commands、支援的憑證）。

---

## 3. 2026.4.26 — Fixes（精選主題）

> **Fixes** 條目極長；以下為 **高信號分組**。完整列表以 `CHANGELOG.md` → **`## 2026.4.26` → Fixes** 為準。

### Agent／工具／子代理

- **LSP**：Gateway shutdown／runtime dispose 時 **終止 bundled stdio LSP 程序樹**（含巢狀 `tsserver`）（#72357）。  
- **`subagents.allowAgents`**：對 **顯式同 agent `sessions_spawn(agentId=...)`** 強制遵守，不再自動允許「自己呼叫自己」（#72827）。  
- **`sessions_spawn(runtime="acp")`**：在 **`acp.dispatch.enabled=false`** 下仍可做 **顯式** bootstrap turn（#63591）。  
- **Tool loop**：偵測歷史 **限定目前 run**；**忽略 exec volatile metadata** 比對結果；**空參數 object schema** 正規化為 `{}`（Pi 驗證前）。  
- **推理／replay**：`<think>` **未閉合但有後續正文**的回收；**orphan closing reasoning tag** 隱私邊界；Anthropic／Gemini **尾端 assistant prefill** 剔除；Bedrock **空白 user turn** 修復。

### Gateway／Control UI／WebChat

- **啟動**：重用 **config snapshot 內 plugin manifests** 做 auto-enable／validation／bootstrap **減少重複 pass**。  
- **前景 Gateway**：略過 CLI **自我 respawn**，避免低記憶體 Linux／Node 24 **hang 在 log 前**（#72720）。  
- **device.token.rotate**：**不再回傳**已旋轉的 bearer 至 shared／admin 回應（保留同裝置 handoff）（#66773）。  
- **Talk／Google Live**：維持 **WebSocket**、驗證端點、**relay 會話數上限**、移除不跟隨設定提供者的過時按鈕。  
- **WebChat**：handshake **綁定作用中 socket**、拒絕 close 後註冊；**Reload Config** 丟棄過期本地狀態（#72624、#40352）。  
- **hello-ok**：含 **連線客戶端** 與 **presence 版本**，不需再等後續事件才看見自己上線。

### 插件／安裝／discovery

- **Symlink**：全域／工作區插件根 **跟隨 symlink 目錄**（斷鏈仍忽略）（#36754）。  
- **Security scan**：略過 **test 檔／目錄**、仍強制掃描 **宣告的 runtime entrypoints**（#66840）。  
- **Profile**：**`--profile`** 安裝目的地對齊 **active profile state dir**（#69960）。  
- **npm update**：先裝入 **驗證過的 temp prefix** 再 swap，避免 **新旧混版**（#72396 思路延伸）。  
- **Duplicate 警告**：npm 插件 **刻意覆蓋**同名 bundled 時抑制重複警告。  
- **Windows**：**`c:` drive-letter** 路徑於 **Jiti／ESM／lazy import** 前正規化（Feishu、插件 service、subagent-registry）（多則 #72783、#72636、#72573）。

### 瀏覽器／Meet／Voice

- **Google Meet**：本機 Chrome join **走 OpenClaw browser 控制**、媒體權限、**BlackHole 2ch** 預設音訊、使用 **設定檔**（避免 Permission needed／裸 Chrome）。  
- **CLI**：**bundled browser 插件**在存在 **`browser`** 設定時可 **自動啟動**（含嚴格 allowlist）；**Chrome 啟動失敗 circuit breaker**。  
- **Voice／SecretRef**：Twilio／TTS 等 **apiKey SecretRef** 於 runtime snapshot 解析（#68690）。

### 網路／Proxy

- **`ALL_PROXY`／`all_proxy`** 進入 Undici 全域 dispatcher 與 provider proxy helper；**SSRF trusted-proxy auto-upgrade** 仍僅跟 **`HTTP(S)_PROXY`**（#43821）。  
- **WhatsApp QR**：登入 WebSocket **遵守 `HTTPS_PROXY`／`HTTP_PROXY`**（#72547）。

### 記憶／QMD／dreaming／tasks

- **QMD**：**`--mask`** 收窄根記憶索引；多集合 **單次搜尋**；lexical **`searchMode: "search"`** 跳過向量探測；狀態輸出解析 **Vectors =／:** 等變體；**watcher dirty** 反映在 status。  
- **memory CLI**：one-shot **`memory index`／`status`／`search`** 走 **transient manager**，避免長駐 watcher **EMFILE**（#59101）。  
- **`memory_search`／`memory_get`**：每次執行 **重解 active runtime config**（#61098）。  
- **Dreaming**：**`dreaming.model`**、 narrative cleanup／fallback 噪音修復。  
- **SQLite WAL**：tasks／TaskFlow／proxy／builtin memory **定時 checkpoint**，抑制 **`*.sqlite-wal`** 膨脹（#72774）。

### Cron／delivery／Discord／其他

- **Cron**：**`maxConcurrentRuns`** 套用 isolated lane、legacy `jobs.json`、schedule 編輯後 **invalid pending slots**、delivery 結果分類等（多 issue）。  
- **Discord**：thread 繼承父 channel **`/model` override**（僅 model fallback）；Markdown 圖片 badge 行為調整（#72642）。  
- **Mattermost**：DM **不強制 reply root**；chat kind 由 **trusted lookup** 決定。  
- **Feishu**：**互動卡片**原生送出、**broadcast `@all`** 不當作 bot mention（#37706）。  
- **Bonjour**：預設廣告用 **系統 hostname**（DNS-safe），減少 **`openclaw.local`** 衝突（#72355）。  
- **Docker slim**：映像 **安裝 CA bundle**，TLS 不因 base 切換而壞（#72787）。

---

## 4. 插件 metadata／啟動效能（橫切）

- **Gateway `PluginLookUpTable`**：單次 manifest registry pass 跨 **啟動 plugin id、載入、deferred channel reload、定價、唯讀通道預設、capability／provider／media、合約、web fallback、owner map、cold discovery cache**；啟動 trace 新增 **timing／count**（installed-index、manifest、startup-plan、owner-map）。  
- **Fingerprint／stale inventory**：重用啟動 metadata、拒絕 **過期 plugin metadata inventory**、**plugin metadata 不進 config snapshot**（多則 refactor／fix）。

---

## 5. 基礎建設與發布

- **Release**：`2026.4.26 beta 1` → **stable**；**config／plugin-sdk API baseline**、**bundled channel metadata** 刷新；**npm provenance** 等 release evidence 文件化。  
- **Coven**：workspace **coven extension 移除**；**ACP opt-in Coven runtime bridge** 等後續可從 changelog／提交查訊。  
- **UI**：`tweakcn` theme import 修飾；config **explicit reload** 丟棄過期狀態（#72624）。

---

## 6. 摘要統計

| 項目 | 數值 |
|------|------|
| 分析起點（上一輪 CHT 對齊提交） | `71de5b07391ce02a7bf57f5ebe0721a507f9a7b3` |
| 本輪非合併提交（約） | **829** |
| 對外穩定版標籤 | **2026.4.26**（接續 **2026.4.25**） |
| 主要參考 | `CHANGELOG.md` → **`## 2026.4.26`**（Changes／Fixes） |

*本報告與 `docs-cht/*.md` 版本標註對齊至 **2026.4.26**；細節以官方英文 changelog 與原始提交為準。*
