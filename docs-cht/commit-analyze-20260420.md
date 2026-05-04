# OpenClaw 提交分析報告 — 2026-04-20

> **分析範圍**：從 `153c183b25960f2ac07b01108d41714b0f771968`（doc: 更新繁體中文文件版本號至 2026.4.15）至最新提交（`115f05d595`，chore: prepare 2026.4.20 release），共 **1,503 個非合併提交**，涵蓋 2026-04-15 至 2026-04-20。

---

## 目錄

1. [版本路徑概覽](#1-版本路徑概覽)
2. [重大新功能（2026.4.18–2026.4.20）](#2-重大新功能)
3. [安全修復](#3-安全修復)
4. [OpenAI Codex / OAuth 大修](#4-openai-codex--oauth-大修)
5. [Cron 交付系統重構](#5-cron-交付系統重構)
6. [Telegram 改善](#6-telegram-改善)
7. [Agent / Context Engine 改善](#7-agent--context-engine-改善)
8. [記憶體 / Memory 改善](#8-記憶體--memory-改善)
9. [Gateway 改善](#9-gateway-改善)
10. [頻道修復](#10-頻道修復)
11. [Control UI 改善](#11-control-ui-改善)
12. [效能優化與重構](#12-效能優化與重構)
13. [Plugin SDK 與架構](#13-plugin-sdk-與架構)
14. [CI / 建置改善](#14-ci--建置改善)
15. [文件更新](#15-文件更新)
16. [摘要統計](#16-摘要統計)

---

## 1. 版本路徑概覽

```
2026.4.15   → 穩定版（上次更新終點）
2026.4.18   → 穩定版（含 3 項新功能 + 大量修復）
2026.4.19-beta.1 → Beta（Agent / Browser / Codex 修復）
2026.4.19-beta.2 → Beta（OpenAI streaming、nested lanes 修復）
2026.4.20   → 穩定版（含 14 項新功能 + 大量修復）
```

版本號跳過了 2026.4.16 / 2026.4.17（為內部版本或未發布版）。

---

## 2. 重大新功能

### 2026.4.18 新功能

#### Anthropic Claude Opus 4.7 xhigh Reasoning

```
Anthropic/models: add Claude Opus 4.7 xhigh reasoning effort support
```

為 Claude Opus 4.7 新增 `xhigh` reasoning effort 支援，並與 adaptive thinking 分開管理。

#### Control UI Settings 全面改版

```
Control UI/settings: overhaul settings and slash-command experience (#67819)
```

Settings 和 slash-command 體驗全面翻新：
- 更快速的 presets
- Quick-create flows
- 重新設計的 command discovery

#### macOS Screen Snapshot

```
macOS/gateway: add screen.snapshot support for macOS app nodes (#67954)
```

macOS app nodes 新增 `screen.snapshot` 支援，包含 runtime plumbing 和預設 macOS allowlisting。

### 2026.4.20 新功能

#### Onboarding Wizard UI 改善

```
wizard: support searchable select, restore hint in search haystack
onboard: clearer security disclaimer, loading spinners, api key placeholder
```

- 安全聲明改版：黃色警告橫幅 + 清單格式
- 模型 catalog 載入期間顯示 loading spinner
- provider API key 輸入框新增「API key」placeholder
- wizard 支援可搜尋選取

#### 分層模型定價支援

```
feat: add tiered model pricing support (#67605)
```

新增 **tiered model pricing** 支援：
- 從快取 catalog 和配置模型讀取分層定價
- 包含 Moonshot Kimi K2.6/K2.5 cost estimates
- Token 用量報告更準確

#### Session Maintenance 預設強制執行

```
fix(sessions): enforce maintenance by default and prune on load to prevent gateway OOM
```

**重要修復**：Session maintenance 現在預設強制執行內建條目上限和年齡修剪，並在載入時修剪過大的 stores，防止累積的 cron/executor session backlog 在寫入路徑執行前造成 gateway OOM。

#### Cron Jobs 狀態分離

```
Cron: split runtime execution state into jobs-state.json (#63105)
```

Cron runtime 執行狀態分離至 `jobs-state.json`：
- `jobs.json` 保持穩定，適合 git 追蹤的工作定義
- `jobs-state.json` 儲存 runtime 狀態

#### Compaction 通知

```
Agents/compaction: send opt-in start and completion notices during context compaction (#67830)
```

Context compaction 期間可選發送開始和完成通知。

#### Moonshot Kimi K2.6 為新預設

```
Moonshot/Kimi: default bundled Moonshot setup to kimi-k2.6 (#69477)
Moonshot/Kimi: allow thinking.keep = "all" on moonshot/kimi-k2.6 (#68816)
providers: default Moonshot to Kimi 2.6
```

- Moonshot 預設從 Kimi K2.5 升級至 **Kimi K2.6**
- `kimi-k2.6` 支援 `thinking.keep = "all"`
- 其他 Moonshot 模型或 pinned `tool_choice` 停用 thinking 時自動去除

#### BlueBubbles 群組 System Prompt

```
BlueBubbles/groups: forward per-group systemPrompt config into inbound context GroupSystemPrompt (#69198)
```

每個群組的 `systemPrompt` 配置現在注入至入站 context `GroupSystemPrompt`，支援 `"*"` 萬用字元 fallback。

#### Detached Task Registration Contract

```
Plugins/tasks: add a detached runtime registration contract (#68915)
```

插件 executors 可透過新的 detached runtime registration contract 管理 detached task lifecycle 和 cancellation。

#### Terminal Logging 優化

```
Terminal/logging: optimize sanitizeForLog() (#67205)
```

將控制字元剝離的迴圈替換為單一 regex pass，效能顯著提升。

#### QA Suite 預設失敗行為

```
QA/CI: make openclaw qa suite fail by default when scenarios fail (#69122)
```

`openclaw qa suite` 和 `openclaw qa telegram` 現在在場景失敗時預設返回非零退出碼；使用 `--allow-failures` 進行 artifact-only 執行。

#### Mattermost 串流預覽

```
Mattermost: stream thinking, tool activity, and partial reply text into single draft preview post (#47838)
```

Mattermost 支援將 thinking、tool activity 和 partial reply text 串流進單一 draft preview post，最終確定後就位。

#### 強化系統提示

```
Agents/prompts: strengthen default system prompt and OpenAI GPT-5 overlay
```

強化預設系統提示和 OpenAI GPT-5 overlay：
- 更清晰的 completion bias
- Live-state checks
- Weak-result recovery
- Verification-before-final guidance

---

## 3. 安全修復

### Gateway Websocket Broadcasts 權限管控（重要）

```
Gateway/websocket broadcasts: require operator.read for chat/agent/tool-result event frames (#69373)
```

**重要安全修復**：Chat、agent 和 tool-result event frames 現在需要 `operator.read`（或更高）權限：
- Pairing-scoped 和 node-role sessions 不再被動接收 session chat 內容
- Plugin `plugin.*` broadcasts 限定 operator.write/admin
- Status/transport events（heartbeat、presence、tick 等）不受限制
- Per-client sequence numbers 保持 per-connection 單調性

### Agent Gateway Tool Config 突變守護擴展

```
Agents/gateway tool: extend agent-facing gateway tool config mutation guard (#69377)
```

擴展 agent 面向的 gateway tool config mutation guard：
- 防止 `config.patch` 和 `config.apply` 覆寫 operator 信任路徑（sandbox、plugin trust、gateway auth/TLS、hook routing、SSRF policy、MCP servers、workspace filesystem hardening）
- 防止透過 `agents.list[]` 的 per-agent sandbox、tools 或 embedded-Pi overrides 繞過守護

### Gateway Device Pairing 限制

```
Gateway/device pairing: restrict non-admin paired-device sessions (#69375)
```

非 admin paired-device sessions 現在限制只能操作自己的 pairing list、approve 和 reject actions，不能枚舉其他 devices 或處理其他 device 的 pairing requests。

### Security/dotenv OPENCLAW_* 阻止

```
Security/dotenv: block all OPENCLAW_* keys from untrusted workspace .env files (#473)
```

阻止所有 `OPENCLAW_*` keys 從不受信任的工作區 `.env` files 載入，防止新的 runtime-control 變數被靜默繼承。

### QQBot SSRF Guard

```
fix(qqbot): add SSRF guard to direct-upload URL paths (#69595)
```

QQBot 的 `uploadC2CMedia` 和 `uploadGroupMedia` 新增 SSRF guard。

### Gateway SessionKey Gate

```
fix(gateway): enforce allowRequestSessionKey gate on template-rendered mapping sessionKeys (#69381)
```

在 template-rendered mapping sessionKeys 上強制執行 `allowRequestSessionKey` gate。

### MINIMAX 工作區環境注入阻止

```
fix(security): block MINIMAX_API_HOST workspace env injection (#67300)
```

阻止 MINIMAX_API_HOST 工作區環境注入，移除 env 驅動的 URL routing。

### Gateway Auth 佔位符拒絕

```
Gateway/auth: reject gateway auth credentials matching published example placeholders (#68404)
```

啟動和 secret reload 時拒絕與已發布示例佔位符匹配的 gateway auth credentials，防止 copy-paste 部署使用已知 secrets。

### Gateway Assistant Media 權限

```
Gateway/assistant media: require operator.read scope for assistant-media requests (#68175)
```

Identity-bearing HTTP auth 路徑上的 assistant-media 文件和元資料請求需要 `operator.read` scope。

### macOS Remote SSH StrictHostKeyChecking

```
macOS/remote SSH: require already-trusted host key (#68199)
```

macOS remote command 路徑從 `StrictHostKeyChecking=accept-new` 改為 `StrictHostKeyChecking=yes`，第一次連接不再靜默接受未知 host key。

### Exec Approvals 控制字元轉義

```
Exec approvals/display: escape raw control characters in approval-prompt command sanitizers (#68198)
```

Approval 提示命令消毒器中轉義原始控制字元（包括 newline 和 carriage return）。

### Browser CDP URL 遮蔽

```
Config/redact: add browser.cdpUrl and browser.profiles.*.cdpUrl to sensitive URL config paths (#67679)
```

`browser.cdpUrl` 和 `browser.profiles.*.cdpUrl` 新增至敏感 URL 配置路徑清單，避免嵌入 credentials 洩漏。

---

## 4. OpenAI Codex / OAuth 大修

本版本對 Codex OAuth 進行了大量修復，主要目標是讓外部 CLI OAuth 保持 runtime-only，不污染 OpenClaw 持久化狀態：

```
OpenAI Codex/OAuth: keep OpenClaw as canonical owner for imported CLI OAuth sessions
OpenAI Codex/OAuth: keep external CLI OAuth imports runtime-only
OpenAI Codex/OAuth: drop legacy CLI-manager routing
OpenAI Codex/OAuth: only bootstrap from external CLI OAuth when local profile is missing/unusable
OpenAI Codex/OAuth: rename external CLI bootstrap helper
OpenAI Codex/OAuth: treat OpenAI TLS prerequisites probe as advisory
OpenAI Codex/OAuth: keep Codex-specific auth bridging inside owning plugins
```

關鍵改變：
- OpenClaw 保持為 imported Codex CLI OAuth sessions 的規範擁有者
- 外部 CLI OAuth imports 保持 runtime-only，不寫入 `auth-profiles.json`
- 只有在本地 OpenClaw profile 缺失或不可用時才從外部 CLI OAuth bootstrap
- OpenAI TLS prerequisites 探測改為 advisory 而非 hard blocker
- Legacy CLI-manager routing 移除

### Codex /backend-api/codex 端點修復

```
OpenAI Codex: route ChatGPT/Codex OAuth Responses requests through /backend-api/codex endpoint (#69336)
```

修復 `openai-codex/gpt-5.4` 在 `/backend-api/responses` 別名被移除後的 404 問題。

### Codex App-Server 改善

```
fix(codex): default app-server approvals to on-request (#68721)
fix(codex/app-server): release session lane when projector throws (#69072)
```

- Codex harness sessions 預設使用 `on-request` approval，不再 over-permissive
- 修復 `turn/completed` 通知拋出時的 session lane 洩漏

### Models OAuth Health 對齊

```
Models status/OAuth health: align OAuth health reporting with effective credential view
```

OAuth health 報告與 runtime 使用的同等有效憑證視圖對齊，過期可刷新 sessions 不再預設顯示為健康。

---

## 5. Cron 交付系統重構

本版本對 Cron 交付邏輯進行了大量修復和改進：

### Cron 交付目標預覽

```
feat(cron): preview resolved delivery targets
fix(cron): resolve delivery preview server-side
```

`cron list/show` 現在顯示交付目標預覽（server-side 解析）。

### Cron 交付 Policy 修復

```
fix(cron): gate delivery prompt on message tool availability (#69587)
fix(cron): align dry-run delivery previews with target policy
fix(cron): track implicit message sends
fix(cron): make delivery previews dry-run safe
fix: cron chat delivery policy
fix(cron): require verified message delivery target
```

大量 cron 交付 policy 修復，確保：
- 交付提示根據 message tool 可用性進行 gate
- dry-run 預覽與目標 policy 對齊
- 追蹤 implicit message sends

### Cron 隔離 Agent 交付修復

```
fix(cron): keep message tool for chat delivery
fix(cron): preserve explicit delivery.mode: "none" message targets
fix(cron): keep delivery.mode: "none" from inheriting stale implicit recipient
fix(cron): ignore disabled channels in announce delivery ambiguity check
fix(cron): preserve heartbeat.target="last" through deferred wake queuing
```

- `delivery.mode: "none"` 不再繼承 stale implicit recipient
- heartbeat.target="last" 在 deferred wake queuing 中保持
- 停用頻道在 announce delivery 歧義檢查中被忽略

### Cron Telegram 去重複修復

```
fix(cron/telegram): key isolated direct-delivery dedupe to each cron execution
```

去重複 key 改為每次 cron execution，修復 recurring Telegram announce runs 靜默跳過後續 sends 的問題。

### Cron State 分離

```
Cron: split runtime execution state into jobs-state.json (#63105)
```

`jobs-state.json` 分離 runtime 狀態，`jobs.json` 保持穩定。

---

## 6. Telegram 改善

### Telegram Polling Transport 有界連接池（重要修復）

```
Telegram/polling transport: give undici dispatcher pool bounded keep-alive defaults (#68718)
```

**重要修復**：修復長期運行的 gateway 程序累積大量 `api.telegram.org` 已建立連接的問題：
- Transports 現在暴露 `close()`
- `TelegramPollingTransportState` 在 dirty-rebuild 時銷毀過期 transport
- `TelegramPollingSession` 在 polling 退出時 dispose transport
- 每個構建的 `Agent`、`ProxyAgent` 和 `EnvHttpProxyAgent` 都有嚴格的 per-origin pool cap

### Telegram Polling 修復

```
fix(telegram): harden polling transport liveness (#69476)
fix(telegram): bound offset confirmation timeout (#50368)
fix(telegram): tune polling stall threshold
```

- Polling liveness 強化：成功 `getUpdates` 發布帳戶健康 liveness
- Offset confirmation `getUpdates` probe 加入 client-side timeout
- Polling stall 閾值從 90s 提升至 120s，並可透過 `channels.telegram.pollingStallThresholdMs` 配置

### Telegram Streaming 修復

```
Telegram/streaming: keep transient preview on same message during auto-compaction (#66939)
Telegram/streaming: clear compaction replay guard after visible non-final boundaries
Telegram/streaming: fence stale preview and finalization work after aborts
```

- auto-compaction 時保持 transient preview 在同一訊息
- 修復 abort 確認後重播舊回覆的問題
- 清除 compaction replay guard 讓 post-tool assistant 回覆輪換至新 preview

### Telegram Callback 修復

```
Telegram/callbacks: treat permanent callback edit errors as completed updates (#68588)
```

永久 callback edit 錯誤視為已完成，修復 stale command pagination buttons 阻塞更新水位線的問題。

### Telegram ACP Bindings 清理

```
Telegram/ACP bindings: drop persisted DM bindings pointing at missing/failed ACP sessions (#67822)
```

重啟時清理指向缺失或失敗 ACP sessions 的持久化 DM bindings。

### Telegram Status Reactions

```
fix(telegram): honor removeAckAfterReply for status reactions (#68067)
```

status reactions 啟用時遵守 `messages.removeAckAfterReply` 設定。

---

## 7. Agent / Context Engine 改善

### Agent 嵌套 Lane 隔離

```
Agents/nested lanes: scope nested agent work per target session (#67785)
```

嵌套 agent 工作按目標 session 隔離，長時間的嵌套 run 不再跨 gateway 阻塞無關 sessions。

### Agent Bootstrap 改善

```
Agents/bootstrap: resolve bootstrap from workspace truth
Agents/bootstrap: keep embedded bootstrap instructions on hidden user-context prelude
Agents/bootstrap: suppress normal /new and /reset greetings while BOOTSTRAP.md pending
```

- Bootstrap 從 workspace 真實狀態解析，不從過期的 session transcript markers
- Bootstrap 指令保持在隱藏的 user-context prelude
- `BOOTSTRAP.md` 待處理時抑制正常的 `/new` 和 `/reset` 問候語

### Session Auto-failover Override 清除

```
Sessions/reset: clear auto-sourced model/provider/auth-profile overrides on /new and /reset
Agents/model selection: clear transient auto-failover session overrides before each turn
```

- `/new` 和 `/reset` 時清除 auto-sourced model、provider 和 auth-profile overrides
- 每次 turn 前清除 transient auto-failover session overrides，確保恢復的 primary models 立即重試

### Session Cost Snapshot 修復

```
fix(sessions): snapshot estimatedCostUsd (#69403)
```

修復 `estimatedCostUsd` 在多次 persist 路徑中被累積（最高 x 幾十倍）的問題，改為 snapshot 語意。

### Pi Runner 靜默 Error Turn 重試

```
Agents/Pi runner: retry silent stopReason=error turns with no output (#68310)
```

在沒有 side effects 時重試靜默 `stopReason=error` turns，讓非 frontier providers 短暫返回空 error turns 時有機會重試。

### Context Engine 第三方插件修復

```
Context engine/plugins: stop rejecting third-party engines with different info.id (#66678)
```

停止拒絕 `info.id` 與 registered slot id 不同的第三方 context engines（如 `lossless-claw`）。

### Exec/YOLO 修復

```
Exec/YOLO: stop rejecting gateway-host exec in security=full plus ask=off mode
```

修復 Python/Node script preflight hardening path 阻止 `security=full` + `ask=off` 模式的問題，`node <<'NODE' ... NODE` 等 heredoc 形式再次可用。

### Agent Compaction Settings 重載修復

```
Agents/compaction: always reload embedded Pi resources through explicit loader
fix(agents): reapply compaction settings after resource loader reload (#65602)
```

修復沒有 extension factories 的 runs 在 session 開始前靜默丟失 compaction settings 的問題。

### 子代理通道繫結修復

```
Agents/channels: route cross-agent subagent spawns through target agent's bound channel account (#67508)
```

跨代理子代理 spawns 現在通過目標代理的 bound channel account 路由，child sessions 不再繼承 caller 的 account。

---

## 8. 記憶體 / Memory 改善

### Active Memory 優雅降級

```
Active Memory: degrade gracefully when memory recall fails during prompt building (#69485)
```

記憶回憶在 prompt 建置期間失敗時優雅降級，記錄 warning 並讓回覆繼續而非失敗整個 turn。

### Active Memory Timeout 提升

```
Active Memory: raise blocking recall timeout ceiling to 120 seconds (#68480)
```

阻塞回憶 timeout 上限從預設提升至 120 秒。

### Dreaming Narrative 洩漏修復

```
Memory-core/dreaming: normalize sweep timestamps and reuse hashed narrative session keys (#67023)
```

修復 Dreaming narrative sub-sessions 洩漏的問題，正規化 sweep timestamps 並重用 hashed narrative session keys。

### Memory sqlite-vec 警告改善

```
Memory/sqlite-vec: emit degraded sqlite-vec warning once per episode (#67898)
```

每次降級 episode 只發出一次 `sqlite-vec` 降級警告，不再為每次文件寫入重複發出。

### Memory-core 向量維度保留

```
Memory-core: preserve stored vector dimensions during read-only recovery
```

read-only 恢復期間保留儲存的 vector dimensions，防止 memory indexes 在修復 read-only 狀態時丟失 vector metadata。

---

## 9. Gateway 改善

### Gateway HTTP Bind 延遲

```
Gateway/startup: delay HTTP bind until websocket handlers are attached (#43392)
```

延遲 HTTP bind 直到 websocket handlers 附加完成，防止啟動後立即的 websocket health/connect 探測觸發啟動競態。

### Gateway Stale PID 祖先保護

```
Gateway/restart: keep stale-gateway cleanup from terminating current process's parent (#68517)
```

修復 stale-gateway cleanup 終止當前程序父程序或祖先的問題，防止 WeChat 等 plugin sidecars 殺死 active gateway 觸發無限 supervisor 重啟迴圈。

### Gateway Sessions 過期 Agent 拒絕

```
Gateway/sessions: reject stale agent-scoped sessions after agent removed from config (#65986)
```

agent 從配置中移除後拒絕過期的 agent-scoped sessions，保留 legacy default-agent main-session aliases。

### Gateway TUI Startup Retry

```
Gateway/TUI: retry session history while local gateway finishes startup (#69164)
```

`openclaw tui` 重連時重試 session history，修復 `chat.history unavailable during gateway startup` 錯誤。

### Gateway Doctor 配對改善

```
Doctor/gateway: surface pending device pairing requests and stale token repair steps (#69210)
```

`openclaw doctor --fix` 現在顯示待處理的設備配對請求、scope-upgrade approval drift 和 stale device-token 修復步驟。

### Gateway Pairing 改善

```
Gateway/pairing: treat loopback shared-secret clients as local for pairing (#69431)
Gateway/pairing: return reason-specific PAIRING_REQUIRED details (#69227)
```

- Loopback shared-secret node-host、TUI 和 gateway clients 視為 local 進行配對決定
- `PAIRING_REQUIRED` 錯誤返回原因特定的詳細資訊和修復建議

### Gateway Usage Cache 有界

```
fix(gateway): bound costUsageCache with MAX + FIFO eviction (#68842)
```

Cost usage cache 使用 FIFO 驅逐設置上限，防止 date/range lookups 無限增長。

---

## 10. 頻道修復

### Browser / CDP 改善

```
Browser/CDP: allow selected remote CDP profile host for health checks without widening SSRF policy (#68207)
Browser/CDP: add phase-specific CDP readiness diagnostics (#68715)
Browser/Chrome MCP: surface DevToolsActivePort attach failures as browser-connectivity errors
Browser/user-profile: let existing-session profile="user" auto-route to connected browser node
```

- WSL-to-Windows Chrome endpoints 不再在 strict defaults 下顯示 offline
- Phase-specific CDP readiness diagnostics
- `DevToolsActivePort` attach failures 顯示為 browser-connectivity errors
- 現有 session `profile="user"` 自動路由至已連接的 browser node

### Slack 改善

```
fix(slack): fix SecretRef token resolution for file/exec secret sources (#68954)
Slack/streaming: resolve native streaming recipient teams from inbound user
Slack/threads: log failed thread starter and history fetches at verbose level
```

- 修復透過 `file` 或 `exec` secret sources 配置的帳戶 outbound replies 失敗的問題
- Native streaming recipient team 從 inbound user 解析

### WhatsApp 改善

```
WhatsApp/multi-account: centralize named-account inbound policy
WhatsApp/gateway: harden WhatsApp auth persistence and backup recovery
```

- Named-account inbound policy 集中管理
- WhatsApp auth persistence 和 backup recovery 強化

### Matrix 改善

```
Matrix: fix sessions_spawn --thread subagent session spawning (#67643)
Matrix/allowlists: hot-reload dm.allowFrom and groupAllowFrom entries on inbound messages (#68546)
Matrix/commands: recognize slash commands prefixed with bot's Matrix mention (#68570)
Matrix: honor dangerouslyAllowPrivateNetwork for private-network homeservers (#68332)
```

- `sessions_spawn --thread` 子代理 session spawning 修復
- `dm.allowFrom` 和 `groupAllowFrom` 條目熱重載
- 識別以 bot Matrix mention 為前綴的 slash commands（如 `@bot:server /new`）

### BlueBubbles 改善

```
BlueBubbles: raise outbound send timeout default to 30s (#69193)
BlueBubbles: consolidate outbound HTTP through typed BlueBubblesClient (#68234)
BlueBubbles: always set method explicitly on outbound text sends (#69070)
BlueBubbles: prefer iMessage over SMS when both chats exist (#61781)
BlueBubbles/reactions: fall back to love when agent reacts with emoji outside tapback set (#64693)
```

- 出站 send timeout 預設從 10s 提升至 30s（可配置 `channels.bluebubbles.sendTimeoutMs`）
- 通過 `BlueBubblesClient` 集中管理出站 HTTP（修復 localhost 圖片附件 SSRF 阻塞）
- 出站文字發送明確設定 `method`（修復 macOS 26 AppleScript -1700 錯誤）
- 同一 handle 同時有 iMessage 和 SMS 時優先使用 iMessage
- agent 用不在 tapback 集的 emoji 反應時 fallback 至 `love`

### Feishu 改善

```
Feishu/card actions: resolve card-action chat type from Feishu chat API (#68201)
```

從 Feishu chat API 解析 card-action chat type，DM 來源的 card actions 不再繞過 `dmPolicy`。

### Discord 改善

```
Discord/think: only show adaptive in /think autocomplete for applicable models
Discord/slash commands: tolerate partial Discord channel metadata (#68953)
```

- `/think` autocomplete 只對支援 adaptive thinking 的 model 顯示 `adaptive`
- Slash commands 和 model picker flows 容忍部分 Discord channel 元資料

### Mattermost

```
Mattermost: stream thinking, tool activity, and partial reply text into single draft preview (#47838)
```

串流 thinking、tool activity 和 partial reply text 至單一 draft preview post。

---

## 11. Control UI 改善

### Settings 全面改版

```
Control UI/settings: overhaul settings and slash-command experience (#67819)
Control UI/settings: reset scroll position when switching settings pages (#68150)
```

- Settings 和 slash-command 體驗全面改版：更快速 presets、quick-create flows
- 切換 settings 頁面時重置 scroll 位置

### Device Pairing UI 改善

```
Control UI/device pairing: explain scope and role approval upgrades during reconnects (#69221)
Gateway/Control UI: surface pending scope, role, and device-metadata pairing approvals (#69226)
```

- 重連期間解釋 scope 和 role approval upgrades
- Control UI 顯示待處理的 scope、role 和 device-metadata 配對審批

### Webchat 改善

```
fix(gateway): restore webchat pure-image turn handling (#69358)
fix(gateway): handle webchat image-only turns (#69474)
macOS/webchat: enable Undo and Redo in composer text input (#34962)
```

- Webchat pure-image turn handling 修復
- macOS webchat composer 啟用 Undo 和 Redo（使用 `NSTextView` undo manager）

### Setup TUI 修復

```
Fix setup TUI hatch terminal handoff (#69524)
```

Setup hatch TUI 在新 process 重啟，保留已配置的 gateway target 和 auth source，不在命令列 args 上暴露 gateway secrets。

---

## 12. 效能優化與重構

### Plugin Loader Alias 重用

```
Plugins/tests: reuse plugin loader alias and Jiti config resolution across repeated loads (#69316)
```

跨重複的同 context 載入重用 plugin loader alias 和 Jiti config resolution，減少 import-heavy 測試開銷。

### 大量測試 Fixture 共用

1503 個提交中有大量測試重構，將測試 fixtures 改為合成（synthetic）版本，去除對 bundled plugins 的耦合：
```
test: use synthetic auto reply fixtures
test: use synthetic agent infra fixtures  
test: use synthetic cli provider fixtures
test: use synthetic outbound dispatch fixtures
...（30+ 個類似提交）
```

### 熱路徑優化

```
perf: avoid sort-for-single selection
perf: trim hot path allocations
perf: avoid bundled channel cold-loads in hot paths
```

### 測試結構重構

```
refactor: share oauth callback flow
refactor: reuse shared local file access
refactor: share thread binding lifecycle
refactor: share allow-from store file reads
refactor: share ssrf policy merging
refactor: share fast mode normalization
refactor: dedupe install scan skill spec
...（50+ 個類似提交）
```

---

## 13. Plugin SDK 與架構

### Plugins 優先級管理

```
Plugins: keep only highest-precedence manifest when distinct plugins share an id (#41626)
```

不同發現的插件共享相同 id 時，只保留最高優先級的 manifest，防止低優先級的全域或工作區重複 plugin 在 bundled 旁邊載入。

### Plugin SDK Secret Input 保留

```
Plugin SDK: preserve secret-input-runtime function exports in published builds
```

在發布版本中保留 `secret-input-runtime` function exports，讓 provider plugins 可以讀取 SecretRef-backed setup inputs。

### Plugin 記憶 Capability 保留

```
Plugins/memory: preserve active memory capability when read-only snapshot plugin loads run (#69219)
```

read-only snapshot plugin 載入時保留 active memory capability，防止 status 和 provider discovery 路徑清除 memory public artifacts。

### Plugin Webhooks 強化

```
Plugins/webhooks: enforce synchronous plugin registration with full rollback (#67941)
```

強制同步 plugin 註冊並在失敗時完整回滾，快取 SecretRef-backed webhook auth per route。

### Plugin Discovery 改善

```
Plugins/discovery: reuse bundled and global plugin discovery results across workspace cache misses (#67940)
```

跨 workspace cache misses 重用 bundled 和 global plugin discovery results，防止 Windows multi-workspace startup 重複做共享同步掃描。

### MCP stdio 環境安全過濾

```
fix(mcp): block dangerous stdio env overrides
fix: narrow MCP stdio env safety filter (#69540)
```

阻止 MCP stdio servers 的 interpreter-startup env keys（如 `NODE_OPTIONS`），同時保留普通 credential 和 proxy env vars。

---

## 14. CI / 建置改善

### 捆綁 Channel 依賴 Docker Smoke

```
test: add bundled channel dependency Docker smoke
```

新增 bundled channel 依賴的 Docker smoke 測試。

### CI Shard 拆分

```
ci: isolate gateway watch regression harness
test: split contract vitest shards
test: split chat view coverage
```

多個 CI shard 拆分，提升並行效率。

### Bundled Plugin Runtime

```
fix(plugins): install bundled runtime dependencies into plugin's own runtime directory
fix(plugins): ignore pnpm npm_execpath when repairing bundled plugin runtime deps
fix: isolate bundled plugin runtime deps
fix: use npm for bundled runtime dep repair
```

Bundled plugin runtime 依賴管理大量修復，確保 packaged installs 可靠工作。

---

## 15. 文件更新

```
docs(openai): clarify GPT-5 prompt defaults
docs(cron): clarify delivery modes
docs(telegram): clarify polling stall tuning
docs: document Kimi cost live smoke
docs: add release tweet style guide
docs: support release branch workflow
docs(codex): clarify approval override example
docs: add setup tui hatch changelog
docs: credit onboarding polish (#69553)
docs: note Codex approval default fix (#68721)
docs(changelog): clarify beta changelog policy
```

---

## 16. 摘要統計

### 版本里程碑

| 版本 | 日期 | 性質 |
|------|------|------|
| 2026.4.15 | 2026-04-15 | Stable（起始點） |
| 2026.4.18 | 2026-04-18 | Stable（含 Opus 4.7 xhigh、Settings 改版、Screen Snapshot） |
| 2026.4.19-beta.1 | 2026-04-19 | Beta（Agent 通道、Browser CDP、Codex 修復） |
| 2026.4.19-beta.2 | 2026-04-19 | Beta（OpenAI streaming、nested lanes、token totals） |
| 2026.4.20 | 2026-04-20 | Stable（含 14 項新功能） |

### 關鍵技術趨勢

1. **Session Maintenance 預設強制執行**：防止 gateway OOM，最重要的基礎設施修復
2. **Moonshot Kimi K2.6 為新預設**：並支援 `thinking.keep = "all"`
3. **分層模型定價**：token 用量報告更準確
4. **Cron 交付系統大修**：大量 policy/routing/dedupe 修復
5. **OpenAI Codex OAuth 系列修復**：外部 CLI OAuth 保持 runtime-only
6. **Telegram Polling Transport 有界連接池**：修復長期運行 gateway 連接洩漏
7. **Gateway Websocket Broadcasts 權限管控**：重要安全改善
8. **Agent Config Mutation Guard 擴展**：更廣泛的 operator 信任路徑保護
9. **Device Pairing 非 Admin 限制**：細粒度 pairing 權限
10. **Context Engine 第三方插件修復**：停止拒絕 `info.id` 不同的第三方 engines
11. **BlueBubbles 大量改善**：timeout、SSRF、iMessage 優先、tapback fallback
12. **Matrix `sessions_spawn --thread` 修復**：Matrix subagent thread spawning
13. **Active Memory 優雅降級**：recall 失敗不再讓整個 reply 失敗
14. **macOS Screen Snapshot**：新增本地螢幕截圖支援

### 主要社群貢獻

| 貢獻者 | 主要貢獻 |
|--------|---------|
| @omarshahine | BlueBubbles 大量改善 |
| @obviyus | Cron 交付系統大修 |
| @stainlu | Agent nested lanes、token totals |
| @vincentkoc | Codex OAuth 系列修復 |
| @mcaxtr | WhatsApp multi-account 改善 |
| @rubencu | Telegram streaming 多項修復 |
| @eleqtrizit | Security: device pairing、gateway tool guard、media scope |
| @gumadeiras | Matrix sessions_spawn、plugin discovery |
| @openperf | Gateway stale-pid、Codex EPIPE crash |
| @sk7n4k3d | Session auto-override clear、shell fallback |
| @BunsDev | Control UI settings overhaul |
| @shakkernerd | Setup TUI hatch、Gateway TUI retry |
| @Magicray1217 | Active Memory graceful degradation |
| @bobrenze-bot | Session maintenance defaults |
| @sliverp | Tiered model pricing |
