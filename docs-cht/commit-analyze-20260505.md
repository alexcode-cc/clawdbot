# OpenClaw 提交分析報告 — 2026-05-05

> **分析範圍**：以上一輪繁體文件對齊點 **release `v2026.5.4`** 為起點，分析 **`v2026.5.4..v2026.5.5`** 的新增／更新提交，並以英文 **`CHANGELOG.md` → `## 2026.5.5`** 作為產品行為主軸。
> **版本對齊**：`package.json` 目前為 **`2026.5.5`**；`git tag v2026.5.5` 指向 **`b1abf9d8ae4410c6a6e08f7dfd2d617f4550281c`**。
> **前一輪文件**：`docs-cht/commit-analyze-20260504.md` 已對齊 **`v2026.5.4`**。
> **提交量**：`v2026.5.4..v2026.5.5` 共 **54** 筆提交，且 **54** 筆皆為非合併提交；檔案差異約 **325** 個檔案，約 **9,293** 行新增、**1,100** 行刪除。

## 目錄

1. [版本切點與發行脈絡](#1-版本切點與發行脈絡)
2. [2026.5.5 — 發行性質](#2-202655--發行性質)
3. [2026.5.5 — Fixes（依主題）](#3-202655--fixes依主題)
4. [受影響文件與程式碼面](#4-受影響文件與程式碼面)
5. [繁體技術文件同步策略](#5-繁體技術文件同步策略)
6. [本輪對齊統計](#6-本輪對齊統計)

---

## 1. 版本切點與發行脈絡

```text
v2026.5.4
  → 2026.5.5 beta 1 / beta 2
  → 2026.5.5 final
  → v2026.5.5
```

本期是一次 **高密度修復版**，英文 `CHANGELOG.md` 的 `2026.5.5` 區段只有 **Fixes**，沒有獨立 Highlights／Changes。提交歷史顯示此版本主要將 2026.5.4 後的 Gateway、通道、更新、插件、Control UI、TUI、Codex、Provider 與媒體生成修復收斂到穩定發行。

值得注意的是，提交日期橫跨 **2026-05-04 到 2026-05-06**；本繁體檔名依使用者要求採 **2026-05-05**，實際發行 tag 則以 **`v2026.5.5`** 為準。

## 2. 2026.5.5 — 發行性質

- **主軸不是新功能擴張，而是可靠性收斂**：多數條目修復既有通道、Gateway、更新、doctor、插件安裝與 session 路徑在真實部署下的邊界問題。
- **通道穩定性是最大面向**：Feishu topic session、LINE open DM policy、Discord guild text commands、Discord/Telegram/Codex draft progress、Matrix approvals、Slack reconnect logs、iOS pairing、WhatsApp responsiveness 都有修正。
- **Gateway 與狀態可觀測性更清楚**：shutdown、maintenance、restart handoff、health/status sampling、OpenAI-compatible streaming、HTTP media sidecar 路由等路徑持續去假陽性、去卡頓。
- **插件供應鏈修復延續 2026.5.4**：official npm／ClawHub 插件在 host update、disabled／exact-pinned 狀態、managed npm-root、peer link 與 corrupt record 狀態下的修復能力更完整。
- **Agent／session 邊界更乾淨**：runtime context 不再送進 context engine hooks；Control UI `/new` 只在明確建立 session 時觸發；TUI 避免復原 heartbeat session 與大型 stale transcript。

## 3. 2026.5.5 — Fixes（依主題）

### 3.1 通道與訊息交付

- **Feishu**：補齊缺失的 native topic starter thread id，讓 first turn 與 follow-up 留在同一 topic session。
- **LINE**：`dmPolicy: "open"` 必須搭配 wildcard `allowFrom`；錯誤配置會在驗證階段失敗，而不是 webhook ack 後才靜默阻擋。
- **Telegram／Codex**：message-tool-only progress draft 維持可見；Codex tool progress 以 native tool 粒度呈現，避免 item/tool draft line 重複。
- **Discord**：heartbeat ACK timeout 改以實際 heartbeat send 時間計算，避免初始 heartbeat 晚到造成假 reconnect loop；guild plain text control commands（如 `/steer`）會走正常授權與 mention gate。
- **Discord streaming**：progress draft 顯示 live reasoning text，不再只露出空泛的 `Reasoning` 狀態。
- **Matrix approvals**：approval delivery 遇暫時性 Matrix send failure 時短退避重試，降低 pending approval prompt 卡死。
- **Slack**：Socket Mode reconnect log 保留 SDK error context 與 Slack API structured fields，避免 startup failure 只剩 `unknown error`。
- **iOS pairing**：私有 LAN 與 `.local` gateway 可用 setup code／manual `ws://`；Tailscale／public route 仍要求 `wss://`，混合認證 reconnect 也優先明確 gateway password。

### 3.2 Control UI／Sessions／TUI

- **Control UI Sessions**：compaction count 改成精簡 `N Checkpoint(s)` disclosure，展開後用 checkpoint history cards 顯示 session-level detail；Sessions table 顯示 agent runtime 並支援 runtime label 篩選。
- **Control UI performance**：chat 與 channel tabs 在 history payload 或 channel probe 緩慢時仍可互動；partial channel status 會標示，慢 render timing 會寫進 event log。
- **Control UI `/new` hooks**：只有明確 Control UI session creation 才觸發 documented `/new` command 與 lifecycle hooks，避免 SDK parent-session create 被誤處理。
- **TUI session restore**：互動 launch 跳過 generic CLI respawn wrapper；terminal loss 時乾淨退出；不再把 heartbeat session 當 remembered chat session。
- **TUI session list**：session picker bound 到 recent rows，active session 走精準 lookup refresh，避免老舊 store 讓 TUI 先 hydrate 數週前 transcript。

### 3.3 Gateway／HTTP／Status／Doctor

- **Gateway shutdown**：close 期間取消 delayed post-ready maintenance，快速 restart 後不再啟動 orphaned maintenance／cron timers；shutdown warnings 與 HTTP close timeout 以結構化 `ShutdownResult` 回報。
- **Gateway status**：`openclaw gateway status --deep` 與 `openclaw doctor --deep` 會顯示 supervisor restart handoffs，讓 service-managed clean exits 不再看起來像不透明 stopped-service。
- **Gateway health sampling**：快速重複採樣時，不會因單次 CPU/utilization 就標記 event-loop degraded，必須先累積 sustained sampling window。
- **OpenAI-compatible HTTP**：streaming chat-completion headers 接受後立即送 assistant role SSE chunk，避免冷 agent setup 讓 `/v1/chat/completions` client 拿到 bodyless 200 直到 idle timeout。
- **HTTP media sidecar**：不相關 HTTP route 不載入 outgoing-image media handler，disabled OpenAI-compatible route 可快速回 404。
- **Doctor/session repair**：`doctor --fix` 會搬移 heartbeat-poisoned default main session store entries 到 recovery keys，並清除 stale TUI restore pointers。

### 3.4 插件供應鏈與更新

- **Official plugin update**：host update 期間，已安裝的官方 npm／ClawHub 插件（Codex、Discord、WhatsApp、diagnostics 等）即使 disabled 或先前 exact-pinned，也會同步；第三方 plugin pin 仍保留。
- **Managed npm-root repair**：插件更新前修復 stale managed npm-root `openclaw` peer packages，避免 beta channel 官方插件被舊 core package-lock 降級。
- **Peer link reassert**：shared-root npm install／update／uninstall 後重新確認 managed npm plugin 的 `openclaw` peer links，避免操作單一插件後其他 SDK-using 插件解析失敗。
- **Corrupt plugin records**：update 可容忍 corrupt managed plugin record，讓 core package update 仍可完成並提示 repair path。
- **Diagnostics**：source-only TypeScript package warning 更明確指出 missing compiled runtime output 是發布者包裝問題，並引導 update／reinstall 或 disable／uninstall。

### 3.5 Agent／Context／Hooks／Memory

- **Context engines**：隱藏的 OpenClaw runtime-context custom messages 不再送進 assemble、afterTurn、ingest hooks，避免 transcript reconstruction plugins 看到非對話訊息。
- **Session memory hooks**：重複 `/new` 或 `/reset` 在同一分鐘捕獲時，fallback memory filename 加上 collision suffix；reset memory capture 從 command reply path 移出，`llmSlug: true` 才啟用模型生成檔名 slug。
- **Generated media**：attachment-style message tool action 視為已完成 chat send，避免已上傳生成檔後又送 duplicate fallback media；async video／music completions 也避免 announce-agent pending 期間直接重複 fallback。
- **Memory wiki**：`corpus=all` 搜尋保留多 corpus representation，未用完的容量再回填，避免單一 corpus 因分數較高吃掉全部結果。

### 3.6 Providers／媒體生成／模型政策

- **xAI Grok**：native Grok Responses models 不再收到 OpenAI-style reasoning effort controls；bundled xAI thinking profile clamp 到 `off`，避免 live Docker／Gateway run 因 `Invalid reasoning effort` 失敗。
- **Fireworks Kimi**：Kimi models 標記為 thinking-off-only，K2.5／K2.6 送出 `thinking: disabled`，避免 manual model switch 後帶入 Fireworks 不接受的 `reasoning*` 參數。
- **Video generation**：工具邊界接受 provider-specific aspect-ratio／resolution hints；MiniMax 的 `720P` 正規化為支援的 `768P`；Gemini video request 不再送 Google `generateAudio`。
- **Auth profiles**：format-level rejection 不會讓 provider 進 cooldown，使 fallback profiles 仍有機會嘗試不支援的模型名稱之外的路徑。

### 3.7 安全與平台

- **Docker/Gateway**：bundled `docker-compose.yml` 移除 `NET_RAW`／`NET_ADMIN` capability 並啟用 `no-new-privileges`。
- **Exec approvals**：Windows 拒絕 rename-overwrite `exec-approvals.json` 時回落 guarded copy，同時保留 symlink、hard-link 與 owner-only permission 防護。
- **Doctor token shadowing**：`OPENCLAW_GATEWAY_TOKEN` 可能遮蔽不同 active `gateway.auth.token` source 時發出警告；同一 env token 不誤報。
- **CLI/update**：dev-channel preflight lint 改 opt-in 且受限；fetch failure 後乾淨停止，避免後續 update step 在壞狀態下繼續。

## 4. 受影響文件與程式碼面

本期差異橫跨 **325** 個檔案，主要集中在：

- **通道插件**：`extensions/discord`、`extensions/feishu`、`extensions/line`、`extensions/telegram`、`extensions/xai`。
- **Agent 與 runtime**：`src/agents/**`、`src/auto-reply/**`、`src/hooks/bundled/session-memory/**`。
- **Gateway 與狀態**：`src/gateway/**`、`src/status/**`、`src/commands/status*`、`src/commands/doctor*`。
- **CLI／update／sessions**：`src/cli/**`、`src/commands/sessions.ts`、`src/infra/update-runner.ts`。
- **插件安裝與更新**：`src/plugins/**`、`src/infra/npm-managed-root.ts`、`scripts/e2e/update-corrupt-plugin-docker.sh`。
- **Control UI／TUI**：`ui/src/ui/views/sessions.ts`、`src/tui/**`。
- **英文文件**：`docs/channels/feishu.md`、`docs/channels/line.md`、`docs/cli/status.md`、`docs/cli/update.md`、`docs/gateway/doctor.md`、`docs/plugins/codex-harness.md`、`docs/plugins/dependency-resolution.md`、`docs/tools/video-generation.md`、`docs/web/tui.md` 等。

## 5. 繁體技術文件同步策略

本次已將繁體技術文件的主版本標記由 **2026.5.4** 推進到 **2026.5.5**，並在以下主題加入對齊補充：

- **專案概覽**：新增 2026.5.5 release 摘要，標明這是修復導向版本。
- **Gateway**：補充 shutdown、maintenance、status sampling、restart handoff、OpenAI-compatible streaming 與 HTTP media sidecar 修復。
- **Agent**：補充 context engine 過濾、generated media、session memory hooks、TUI／Control UI session 建立與復原修復。
- **通訊頻道**：補充 Feishu、LINE、Telegram/Codex、Discord、Matrix、Slack、iOS pairing、WhatsApp responsiveness。
- **插件開發／供應鏈**：補充 official plugin sync、managed npm-root peer repair、corrupt plugin update、source-only runtime diagnostic。
- **應用程式**：補充 Control UI sessions、responsive performance、iOS private LAN pairing、TUI 啟動／session list。
- **配置系統**：補充 LINE DM policy、token shadowing、xAI／Fireworks thinking profile、Docker hardening、session memory hook naming。
- **CLI／Onboard**：補充 update、status、doctor、sessions cleanup、channels help、plugin repair、Git/peer-link 診斷。
- **大模型**：補充 xAI、Fireworks、OpenAI-compatible streaming、video generation hints、auth profile fallback。
- **長期記憶／TUI**：補充 memory wiki `corpus=all`、session-memory collision suffix、TUI heartbeat session restore 與 recent picker。

## 6. 本輪對齊統計

| 項目 | 結果 |
|------|------|
| **產品行為權威來源** | **`CHANGELOG.md` → `## 2026.5.5`** |
| **`package.json` version** | **`2026.5.5`** |
| **起點 tag** | **`v2026.5.4`** → **`325df3efef`** |
| **終點 tag** | **`v2026.5.5`** → **`b1abf9d8ae`** |
| **提交範圍** | **`v2026.5.4..v2026.5.5`** |
| **提交數** | **54** |
| **非合併提交數** | **54** |
| **檔案差異** | 約 **325** 個檔案 |
| **主要變更類型** | 通道修復、Gateway／status／doctor、插件更新、Control UI／TUI、provider policy、媒體生成、Docker／Windows 安全 |
| **本輪繁體文件主版本** | **`2026.5.5`** |

*若本地 checkout 或 tag 拓撲不同，請以實際 `git describe`、`git rev-parse "v2026.5.5^{}"` 與 `CHANGELOG.md` 頂段為準修正表頭。*
