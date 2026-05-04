# OpenClaw 提交分析報告 — 2026-04-24

> **分析範圍**：從 `ea64ef8d02fee82c8b24982f123751118654e986`（doc: 更新繁體中文文件版本號至 2026.4.23）至工作區 `HEAD`（`59fed91cf7` 時點之 `chore(release): prepare 2026.4.24` 等），歷史區間內共 **899 個非合併提交**。  
> **參考**：`docs-cht/commit-analyze-20260423.md`；本輪以 **官方 CHANGELOG**（`## 2026.4.24`）為主軸彙整使用者可感知變更；未逐條展開全部提交訊息（含大量測試、CI、beta 標籤與插件基礎建設修補）。

---

## 目錄

1. [版本路徑與方法說明](#1-版本路徑與方法說明)
2. [2026.4.24 — Highlights（產品亮點）](#2-2026424--highlights產品亮點)
3. [2026.4.24 — Fixes（修復與硬化，依主題）](#3-2026424--fixes修復與硬化依主題)
4. [基礎建設與發布節奏](#4-基礎建設與發布節奏)
5. [摘要統計](#5-摘要統計)

---

## 1. 版本路徑與方法說明

```
2026.4.23  → 上一次繁體中文技術文件對齊終點（Pi 0.70、影像/OAuth、安全硬化等）
2026.4.24  → Stable：Google Meet 插件、DeepSeek V4、即時語音與 full agent、瀏覽器與插件載入架構、大量通道／MCP／Gateway 修復
```

**為何提交數多、文件卻以 CHANGELOG 為主？**  
899 個提交橫跨 **多個 beta 標籤**、Google Meet 整包落地、插件 runtime mirror 與 ESM 別名修復、Vitest／gateway 測試隔離等；與先前大型報告相同，**以產品 changelog + 主題分類**最能對齊「程式碼與英文專案文件」。

---

## 2. 2026.4.24 — Highlights（產品亮點）

### Google Meet（bundled participant 插件）

- **個人 Google OAuth**、**Chrome／Twilio 即時會話**、**paired-node Chrome** 支援。  
- **Artifacts／出席紀錄匯出**、**「最新會議」類指令**、OAuth 相關 **doctor** 檢查。  
- 針對 **已開啟的分頁** 提供恢復／整理工具，降低「人已在會議中」時的摩擦。

### DeepSeek

- 目錄新增 **V4 Flash**、**V4 Pro**；**V4 Flash** 作為 **onboarding 預設**。  
- 修正 **thinking／replay** 在後續 **tool-call 回合**的行為，避免多輪工具流程斷裂。

### 即時語音與「完整 Agent」

- **Talk**、**Voice Call**、**Google Meet** 路徑上，可採用 consult **完整 OpenClaw agent** 的即時語音迴圈（含更深層、可下工具的回應），而非僅輕量應答。

### 瀏覽器自動化（Browser / Playwright）

- **座標點擊**、**較長的預設 action 逾時預算**、**依 browser profile 的 headless 覆寫**。  
- **分頁重用與恢復**更穩定；**aria snapshot** 在 `format=aria` 下將 `axN` 參照綁到實際 DOM（Playwright 可用時），利於後續動作接續。

### 插件與模型基礎建設

- 啟動負擔減輕：**靜態模型目錄**、**manifest 支援的模型列**、**提供者依賴惰性載入**。  
- **封裝安裝（packaged installs）** 下，bundled 插件 **runtime mirror** 回退複製共享 chunk 時，**保留套件根 runtime 依賴與匯出子路徑**（含 ESM 安全別名），修復部分 Windows npm 更新後無法載入複製之 `dist` 模組的問題。

---

## 3. 2026.4.24 — Fixes（修復與硬化，依主題）

### 封裝安裝與插件 runtime

- 同上：runtime mirror **別名萬用字元／子路徑匯出**、**以 canonical 來源載入**、**保留 cli metadata skip** 等，使 **Bun／npm 全域與同機重複更新**較不易撞 `ENOTEMPTY` 或解析失敗。

### Heartbeat 與排程

- **過大的 `every` 延遲**經共用安全 timer **夾在 Node `setTimeout` 上限內**，避免變成 **1 ms 崩潰迴圈**（#71414、#71478）。  
- **Heartbeat 提示**僅在 heartbeat 執行路徑注入，避免污染一般 agent run（後續精簡提交）。

### Telegram

- 移除啟動時 **persisted offset 的 `getUpdates` preflight**，避免輪詢重啟 **自我衝突**。  
- **同一 bot token** 禁止重入長輪詢；**外部重複 poller** 的 `getUpdates` 衝突診斷更清楚（#56230）。  
- **Webhook**：在通過驗證後 **先 ack 更新** 再跑 bot middleware，降低慢回合觸發 Telegram 重試，同時維持每聊天處理槽（#71392）。

### 瀏覽器

- **Guarded navigation** 期間 Playwright **route 競態**（已處理後拆除）改為略過良性錯誤（#68708）。  
- **Linux**：在要求使用者設定 `browser.executablePath` 前，先嘗試 `/opt/google`、`/opt/brave.com`、`/usr/lib/chromium`、`/usr/lib/chromium-browser` 等常見路徑（#48563）。

### Session、瀏覽器分頁與 fork

- **閒置／每日／`/new`／`/reset` 換 session 並封存舊 transcript** 時，**關閉受追蹤的 browser tabs**，避免舊 session 洩漏分頁。  
- **Thread fork**：當快取 token 計數過期或缺失時，**回退以 transcript 估算父訊息量**，過大時改 **全新 fork** 而非整包複製父 transcript。

### OpenAI / Codex

- **Codex Responses**：系統提示改走頂層 **`instructions`**，並保留既有 native Codex payload 控制。  
- **`gpt-image-2`**：將舊式 `openai-codex.baseUrl`（如 `https://chatgpt.com/backend-api`）**正規化**到 Codex Responses backend，與聊天傳輸一致（#71460）。

### MCP 與 CLI 單次執行

- **`openclaw agent`**、**`openclaw infer model run`** 等 **gateway／本機單次**路徑結束時 **回收 bundled MCP runtime**，避免腳本反覆執行累積 stdio 子程序（#71457）。  
- 更廣：**閒置 bundled MCP** 退場、**工具 allowlist 無法觸及 bundle-MCP 時跳過啟動**、**`mcp.sessionIdleTtlMs` 閒置驅逐**、**`mcp.*` 熱重載** 釋放 session runtime、**Gateway shutdown** 釋放 bundled MCP（#71106、#71110、#70389、#70808、#60656）。

### Control UI

- **`/usage`** 使用**較新 context snapshot** 計算用量百分比；**cache-write tokens** 納入快取命中分母（#47885）。  
- **Control UI bundle** 維持 **browser-safe** 匯出（與 browser profile facade 對齊）。

### 其他通道與執行環境

- **GitHub Copilot**：保留加密 **Responses reasoning item IDs** 於 replay，通過加密 reasoning 驗證（#71448）。  
- **Agents 回覆**：串流 assistant chunk **僅空白**時仍**回收最終答案文字**（#71454、#71467）。  
- **Feishu TTS**：語意語音等 **MP3** 先 **轉 Ogg/Opus** 再發原生語音氣泡；一般 MP3 附件仍當檔案（#61249、#37868）。  
- **Feishu**：主題群 **session key** 穩定化；串流卡片失敗 **backoff**。  
- **Gateway restart continuation**：**持久交接** restart continuation 至 session-delivery queue、**崩潰重啟後恢復佇列**、無 channel route 時 **session-only wake** 後援（#70780）。

### Agent 工具與記憶

- **Tool-result 修剪**：強化 **字元估算與修剪迴圈**，對畸形 `{ type: "text" }`（void／undefined 結果）**序列化非字串 text** 以正確計入大小（#34979、#51267）。  
- **Memory**：**llama runtime** 維持可選載入路徑（#71425）；**hybrid search** 可暴露 **各元件分數**（除錯／調校）。

### 診斷與可觀測性

- **Diagnostics**：**exec 子程序** telemetry 發射（#71451）。  
- **OpenTelemetry 預載 SDK 模式**支援（#71450）。  
- **Logging**：log transport 內部維持私有，避免誤用（#71322）。

### 開發者體驗與環境

- **`openclaw dev reset`** 僅影響 **active profile**。  
- **Daemon／service PATH**：macOS／Linux 產生的 gateway 服務 PATH 納入 **Nix Home Manager** profile bin（`NIX_PROFILES` 右到左、`~/.nix-profile/bin` 後援）（#44402、#59935）。

---

## 4. 基礎建設與發布節奏

- **發布**：`2026.4.24 beta 1` → **beta 6** → **stable**（`chore(release): prepare 2026.4.24`）。  
- **測試**：gateway live model sessions 隔離、Google Meet 測試加速、Vitest **native worker pool** 上限等。  
- **品質／文件**：changelog 整理、外部貢獻 PR 補登、MCP lifecycle 文件交叉引用、release 流程要求自提交改寫 changelog 等（維運向，使用者面仍以上述 Highlights／Fixes 為準）。

---

## 5. 摘要統計

| 項目 | 數值 |
|------|------|
| 分析起點（上一輪 CHT 對齊提交） | `ea64ef8d02` |
| 本輪非合併提交（約） | **899** |
| 對外穩定版標籤 | **2026.4.24**（接續 **2026.4.23**） |
| 主要參考 | `CHANGELOG.md` → `## 2026.4.24` |

*本報告與 `docs-cht/*.md` 版本標註對齊至 **2026.4.24**；細節以官方英文 changelog 與原始提交為準。*
