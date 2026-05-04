# OpenClaw 提交分析報告 — 2026-03-31

> **分析範圍**：從 `2f5fc7baad`（2026.3.28 stable）至 `c153b06817`（2026.3.31 stable），共 **2,125 個非合併提交**，涉及 **2,843 個檔案**、新增 141,654 行、刪除 53,871 行。

---

## 目錄

1. [重大變更 (Breaking Changes)](#1-重大變更)
2. [新功能 (New Features)](#2-新功能)
3. [安全強化 (Security)](#3-安全強化)
4. [背景任務系統 (Background Tasks)](#4-背景任務系統)
5. [頻道改進 (Channels)](#5-頻道改進)
6. [MCP 協議擴充](#6-mcp-協議擴充)
7. [記憶系統改善](#7-記憶系統改善)
8. [Gateway 與認證](#8-gateway-與認證)
9. [模型與提供者](#9-模型與提供者)
10. [Plugin SDK 與 Runtime 重構](#10-plugin-sdk-與-runtime-重構)
11. [Exec Approvals 強化](#11-exec-approvals-強化)
12. [xAI 深度整合](#12-xai-深度整合)
13. [平台應用 (Apps)](#13-平台應用)
14. [CLI 與 TUI](#14-cli-與-tui)
15. [測試與 CI](#15-測試與-ci)
16. [文件與發布](#16-文件與發布)
17. [摘要統計](#17-摘要統計)

---

## 1. 重大變更

### Nodes/exec 路徑統一

```
refactor(nodes): remove nodes.run execution path
refactor(nodes): split media and invoke handlers
```

移除 CLI 與 Agent `nodes` 工具中重複的 `nodes.run` shell 包裝器。所有 node shell 執行現一律通過 `exec host=node`，node 特定能力（媒體、位置、通知）保留在 `nodes invoke` 與專用 action 上。

### Plugin SDK 棄用舊相容路徑

```
refactor(plugin-sdk): remove direct extension source leaks
refactor(plugin-sdk): drop legacy provider compat subpaths
refactor(plugin-sdk): remove channel-specific sdk shims
refactor(plugin-sdk): remove bundled provider setup shims
```

棄用舊版 provider compat subpaths 以及舊版 bundled provider setup 和 channel-runtime 相容層，發出遷移警告。正式前進路徑為 `openclaw/plugin-sdk/*` 進入點加上本地 `api.ts` / `runtime-api.ts` barrels。

### Skills/Plugins 安裝安全閘門

```
feat(security): fail closed on dangerous skill installs
feat(plugins): add dangerous unsafe install override
```

內建的 `critical` 危險碼掃描結果與安裝時掃描失敗現在預設 **fail closed**。先前可成功安裝的外掛可能需要明確的 `--dangerously-force-unsafe-install` 才能繼續。

### Gateway 認證強化

```
Gateway: reject mixed trusted-proxy token config (#58371)
fix(gateway/auth): local trusted-proxy fallback to require token auth (#54536)
```

`trusted-proxy` 現在拒絕混合的 shared-token 配置，本地直連回退需要配置的 token 而非隱式認證同一主機呼叫者。

### Gateway Node 命令與事件範圍縮限

```
fix(gateway): require node pairing before enabling node commands (#57777)
Gateway: harden node event trust boundaries (#57691)
```

Node 命令在配對核准前保持停用。Node 觸發的執行保持在縮減的信任表面上。

### 核心 Provider Model 定義移除

```
refactor: remove core provider model definitions compat
```

核心中的 provider model 定義相容層完全移除，所有模型定義現在完全由 provider plugins 擁有。

---

## 2. 新功能

### 2.1 背景任務控制平面（ClawFlow）

```
feat(tasks): move task ledger to sqlite and add audit CLI (#57361)
feat(tasks): add status health and maintenance command (#57423)
feat(tasks): harden maintenance repair paths (#57439)
feat(status): surface task run pressure (#57350)
feat(status): show session task counts in slash status
ClawFlow: add linear flow control surface (#58227)
ClawFlow: add runtime substrate (#58336)
```

將任務轉為真正的共享背景執行控制平面，統一 ACP、subagent、cron 和背景 CLI 執行於 SQLite 支持的帳本下。新增 `openclaw flows list|show|cancel` 命令、維護/修復路徑、狀態健康監控。

### 2.2 MCP 遠端伺服器與工具橋接

```
feat(mcp): add SSE transport support for remote MCP servers
feat(mcp): add HTTP transport support and tool namespacing
feat(mcp): add plugin tools MCP server for ACP sessions
feat: add openclaw channel mcp bridge
```

支援遠端 HTTP/SSE MCP 伺服器連接、工具命名空間化（`serverName__toolName`）、ACP session 的 plugin tools MCP 伺服器，以及 channel MCP bridge。

### 2.3 新頻道：QQ Bot

```
Feature/add qq channel (#52986)
docs: update qq bot channel docs
fix(qqbot): align speech schema and setup validation (#58253)
fix(qqbot): mark npm-publishable package public
```

新增 QQ Bot 為 bundled channel plugin，支援多帳號設定、SecretRef 憑證、斜線指令、提醒和媒體收發。

### 2.4 Microsoft Foundry Provider

```
feat: Add Microsoft Foundry provider with Entra ID authentication (#51973)
fix: wire microsoft foundry into contract registry
```

新增 Microsoft Foundry 提供者，支援 Entra ID 認證。

### 2.5 LLM 閒置超時

```
feat: add LLM idle timeout for streaming responses
fix: unify idle timeout with runner abort path
```

新增可配置的閒置串流超時，讓停滯的 model streams 乾淨地中止而非掛到更廣的執行超時觸發。

### 2.6 Slack 原生 Exec Approvals

```
feat(slack): add native exec approvals (#58155)
feat(slack): status reaction lifecycle for tool/thinking progress indicators (#56430)
```

Slack 頻道新增原生 approval 路由和授權，exec approval 提示可留在 Slack 中。同時新增狀態 reaction 生命週期指示器。

### 2.7 Pi/Codex 原生 Web Search

```
Codex: add native web search for embedded Pi runs
Codex: use model-level API for native search relevance
CLI: suppress Codex native search summary when web search is off
```

為嵌入式 Pi 執行新增原生 Codex web search 支援。

### 2.8 Matrix 重大增強

```
feat(matrix): add draft streaming (edit-in-place partial replies) (#56387)
feat(matrix): thread-isolated sessions and per-chat-type threadReplies (#57995)
feat(matrix): add group chat history context for agent triggers (#57022)
feat(matrix): add explicit channels.matrix.proxy config (#56930)
```

- **Draft streaming**：部分回覆以就地編輯方式更新
- **Thread-isolated sessions**：per-DM threadReplies 覆寫
- **Group history context**：可配置 `channels.matrix.historyLimit`
- **Proxy config**：明確 HTTP(S) 代理配置

### 2.9 LINE 外發媒體

```
feat(line): add outbound media support for image, video, and audio
fix(line): add ACP binding parity (#56700)
```

LINE 頻道新增圖片、影片和音訊的外發傳送能力，以及 ACP 綁定功能。

### 2.10 ACP 對話綁定

```
feat(acp): add conversation binds for message channels
feat(hooks): add async requireApproval to before_tool_call (#55339)
feat(plugins): expose runId in agent hook context (#54265)
```

ACP 新增 message channel 的對話綁定、工具呼叫前非同步 approval 要求、以及 agent hook 中暴露 runId。

### 2.11 Anthropic Claude CLI 遷移

```
feat: add anthropic claude cli migration
docs: explain anthropic claude cli migration
```

新增 Claude CLI 對話歷史匯入管線。

### 2.12 其他新功能

| 功能 | 說明 |
|------|------|
| OpenAI text verbosity | 在 Responses HTTP 和 WebSocket 傳輸中轉發 `text.verbosity` 設定 |
| TTS 結構化診斷 | 新增提供者診斷和 fallback 分析 |
| Memory QMD extra collections | per-agent 跨 agent session 搜尋 |
| Tavily extra headers | 支援 API 請求額外 headers |
| MS Teams member info | Graph API 成員資訊查詢 |
| Plugin CLI registration | 基於描述符的 lazy root command 註冊 |
| CLI JSON schema | 為 CLI 工具新增 JSON schema |
| CLI inference backends 插件化 | 推論後端移入插件系統 |
| ACPX 自動安裝 | npm postinstall 自動安裝 ACPX 依賴 |

---

## 3. 安全強化

本次版本包含 **50+ 個安全相關提交**，涵蓋多個層面：

### 3.1 Exec 環境隔離

```
fix(exec): block proxy-style env overrides (#58202)
fix(exec): block risky host env overrides (#58209)
fix(host-env): block Python package index redirection env vars (#58011)
Infra: block compiler env overrides (#57832)
Infra: block auth env vars from workspace dotenv (#57767)
fix(infra): block BROWSER, GIT_EDITOR, GIT_SEQUENCE_EDITOR from inherited host env (#57559)
fix(config): block workspace bundled-root dotenv overrides (#58170)
```

全面封鎖 proxy、TLS、Docker endpoint、Python 套件索引、編譯器 include 路徑等環境變數覆蓋。

### 3.2 Sandbox 與 SSH

```
Sandbox: sanitize SSH subprocess env (#57848)
fix(agents): reject escaping symlinks in ssh sandbox uploads (#58220)
sandbox: block sensitive external bind sources (#56024)
fix(sandbox): relabel managed workspace mounts for SELinux (#58025)
```

SSH 子程序環境清理、符號連結逃逸防護、SELinux mount 標籤修正。

### 3.3 Gateway 認證與授權

```
fix(gateway): enforce trusted-proxy HTTP origin checks (#58229)
fix(gateway): revoke active sessions on token rotation (#57646)
fix(gateway): preserve shared-auth rate limits during mixed handshakes (#57647)
Gateway: require verified scope for chat provenance (#55700)
gateway: enforce embeddings HTTP write scope (#57721)
gateway: cap concurrent pre-auth websocket upgrades (#55294)
```

### 3.4 頻道安全

```
fix(telegram): gate audio preflight transcription on sender authorization (#57566)
fix(discord): gate voice ingress by allowlists (#58245)
fix(matrix): filter fetched room context by sender allowlist (#58376)
fix(nostr): verify inbound dm signatures before pairing replies (#58236)
fix(media): drop auth headers on cross-origin redirects (#58224)
fix(media): reject oversized image inputs before decode (#58226)
```

### 3.5 Exec Approvals 強化

```
Exec approvals: detect command carriers in strict inline eval (#57842)
Exec approvals: reject shell init-file script matches (#58369)
Security: block exec approval shell carrier targets (#57871)
fix(exec): unwrap arch and xcrun dispatch wrappers (#58203)
fix(exec): harden shell-side approval guardrails (#57839)
```

### 3.6 其他安全修正

```
fix(scripts/pr): shell-escape env file values to prevent command injection via branch names
fix(config): harden SecretRef round-trip handling in Control UI and RPC writes (#58044)
fix(voice-call): pin plivo callback origins (#58238)
fix(voice-call): reject oversized pre-start media frames (#58241)
fix(pairing): scope pending request caps per account (#58239)
Media: secure image temp dirs (#58270)
```

---

## 4. 背景任務系統

這是 2026.3.31 最大的架構變更之一，將任務從 ACP-only 簿記轉為共享的背景執行控制平面：

### 架構演進

```
refactor(tasks): add owner-key task access boundaries (#58516)
refactor(tasks): remove flow registry layer
refactor(tasks): extract delivery policy
refactor(tasks): add executor facade
refactor(tasks): split delivery state from task runs
refactor(tasks): replace generic task mutation api
refactor(tasks): route subagents through executor
refactor(tasks): route acp through executor
refactor(cron): split main and detached dispatch
```

### 資料存儲

```
feat(tasks): move task ledger to sqlite and add audit CLI (#57361)
fix(tasks): make task-store writes atomic (#58521)
fix(tasks): scope shared run updates by session
fix(tasks): restore owner-key task scope
```

- SQLite 支持的任務帳本取代先前的檔案式存儲
- 原子寫入確保資料一致性
- Session-scoped 更新和 owner-key 存取邊界

### Flow 控制

```
ClawFlow: add linear flow control surface (#58227)
ClawFlow: add runtime substrate (#58336)
Tasks: add minimal flow registry scaffold (#57865)
Tasks: route one-task emergence through parent flows (#57874)
Tasks: add blocked flow retry state (#58204)
```

新增 `openclaw flows list|show|cancel` CLI 命令，支援線性任務流程控制。

---

## 5. 頻道改進

### 5.1 Matrix

**40+ 個 Matrix 相關提交**，主要改進：

- **Draft streaming** 就地編輯回覆
- **Thread-isolated sessions**
- **Group chat history** 上下文
- **Proxy config** 支援
- **E2EE 縮圖加密** (#54711)
- **DM 分類修正** 三層 `is_direct` 邏輯 (#57124)
- **Crypto 載入修正** 使用 `createRequire` (#54566)
- 顯示名稱 mentions 修正 (#55393)
- 引用訊息上下文解析 (#55056)
- 檔案名稱保存修正 (#55692)

### 5.2 Telegram

- Forum-topic 路由修正 (#56060)
- Polling watchdog 修正防止掉回覆 (#56343)
- RFC2544 媒體下載允許 (#57624)
- 長訊息在單字邊界分割 (#56595)
- `replyToMessageId` 驗證 (#56587)
- 空文字回覆跳過 (#56620)
- 模型選擇顯示名稱而非 ID (#56165)
- 配置遷移 `groupMentionsOnly` (#55336)

### 5.3 Discord

- Group DM 元件互動路由修正 (#57763)
- 原生命令 DM allowlist 強化 (#57735)
- Voice ingress allowlist 閘門 (#58245)
- 元件互動政策閘門 (#56014)
- 過期 socket 重連修正 (#54697, #55991)

### 5.4 Slack

- 原生 exec approvals (#58155)
- 狀態 reaction 生命週期 (#56430)
- 重複 draft reply 修正
- 互動式 block delivery 修正 (#57473)
- Table block mode 修正 (#57591)

### 5.5 WhatsApp

- Emoji reaction 支援
- 群組按連接 hydrate (#58007)
- 自聊 DM 去重修正 (#54570)
- 引用包裝訊息解包
- 引用 mentionedJids 排除 (#52711)

### 5.6 MS Teams

- Graph API 成員資訊 (#57528)
- Thread history via Graph (#51643)
- 搜尋訊息 action (#54832)
- Progressive delivery 與 blockStreaming 配置 (#56134)
- 串流重置修正 (#56071)

### 5.7 LINE

- 圖片/影片/音訊外發媒體
- ACP binding 同等性
- Markdown 底線保留 (#47465)
- 時序安全簽章驗證 (#55663)

### 5.8 其他頻道

| 頻道 | 改進 |
|------|------|
| Feishu | 群組 thread 上下文過濾、webhook 簽章驗證 (#55083, #58237) |
| BlueBubbles | 群組參與者名稱充實 (#54984)、webhook auth 節流 (#55133) |
| Nostr | DM 簽章驗證 (#58236) |
| Mattermost | 過期 websocket 偵測 (#53604)、reply 配置傳遞 (#48347) |
| Synology Chat | webhook in-flight guard (#57722)、token guess 節流 (#55141) |
| Nextcloud Talk | webhook auth 失敗節流 (#56007) |
| QQ Bot | 全新頻道（見上方） |
| Voice Call | Plivo callback origin 固定 (#58238)、oversized frame 拒絕 (#58241) |

---

## 6. MCP 協議擴充

```
feat(mcp): add SSE transport support for remote MCP servers
feat(mcp): add HTTP transport support and tool namespacing
feat(mcp): add plugin tools MCP server for ACP sessions
feat: add openclaw channel mcp bridge
fix: normalize MCP tool schemas missing properties field for OpenAI Responses API
fix: harden bundle MCP session runtime cache (#55090)
```

### 主要變更

- **SSE transport**：支援遠端 MCP 伺服器 URL 配置
- **HTTP (streamable-http) transport**：HTTP 傳輸與工具命名空間化
- **Plugin tools bridge**：ACP session 的 MCP 工具橋接
- **Channel MCP bridge**：頻道事件的 MCP 橋接
- **Schema 正規化**：修正缺少 `properties` 的 MCP tool schema
- **Session cache 強化**：bundle MCP runtime 按 session 快取

---

## 7. 記憶系統改善

### QMD 增強

```
feat(memory): add per-agent QMD extra collections for cross-agent session search (#58211)
fix(memory): preserve qmd query semantics and collection recovery (#58183)
fix(memory): use explicit qmd snippet line metadata (#58181)
fix(memory): stagger qmd embed maintenance across agents (#58180)
fix(memory): surface qmd degraded vector status
fix(memory): rebind qmd collections on pattern drift (#57438)
fix(memory): add QMD sync parity hooks (#57354)
fix(memory): keep qmd embeddings active in search mode (#54509)
fix(memory): share embedding providers across plugin runtime splits (#55945)
fix(memory): resolve slugified qmd search paths (#50313)
fix(memory): export archived qmd session transcripts (#57446)
```

- **跨 Agent 搜尋**：per-agent `memorySearch.qmd.extraCollections`
- **Collection 復原**：集合漂移重新綁定
- **維護排程**：跨 agent 嵌入維護交錯執行
- **降級狀態**：向量搜尋降級狀態顯示

### CJK 支援

```
fix(memory): account for CJK characters in QMD memory chunking
fix(memory): add CJK/Kana/Hangul support to MMR tokenize()
Memory: add configurable FTS5 tokenizer for CJK text support
fix: guard fine-split against breaking UTF-16 surrogate pairs
```

- CJK 字元計算修正
- MMR 多樣性偵測 CJK/Kana/Hangul 支援
- FTS5 tokenizer 配置化
- UTF-16 surrogate pair 保護

### CLI 整合

```
fix(memory): add cli qmd session context (#57493)
fix(memory): add qmd mcporter search tool override (#57363)
fix(memory): point qmd config dir at nested path (#39078)
fix: wire memorySearch.extraPaths to QMD indexing (#57315)
```

---

## 8. Gateway 與認證

### 認證強化

- `trusted-proxy` 拒絕混合 shared-token 配置
- Token rotation 後立即撤銷 active sessions
- Mixed handshake 下保持 shared-auth rate limiting
- HTTP tool-invoke 授權收緊
- Node pairing 核准前禁用 node commands
- 並行 pre-auth websocket upgrade 上限
- Bearer-declared HTTP operator scope 忽略

### 會話管理

- Session-scoped task 更新
- Background task 生命週期追蹤
- 會話 thread ID 保留在各事件中
- 會話 reset 時保留 ownership metadata
- Session key 寫入時正規化

### OpenAI 相容性

- `/v1/responses` 支援 flat function tool 定義
- Responses API 格式的 tool schema
- OpenAI version attribution header
- Codex 原生搜尋支援

---

## 9. 模型與提供者

### Provider 架構重構

```
refactor: move provider runtime into extensions (64bf80d, 4c27c90)
refactor: route provider catalogs through public api barrels
refactor: move provider model helpers into plugins
refactor: move provider default model refs into extension apis
refactor: move bundled provider catalogs through plugins
refactor: scope provider runtime to enabled provider plugins
```

Provider runtime 完全移入 extension 插件，透過 Plugin SDK public API 暴露。

### 具體修正

| 提供者 | 修正 |
|--------|------|
| Azure | 停用推理負載省略 (#58208) |
| OpenAI | 停用推理負載省略、text verbosity 轉發 |
| Ollama | `think=false` 支援 (#53200)、串流事件發射 (#53891)、WSL2 網路診斷 (#55435) |
| Google/Gemini | 3.1 模型解析 (#56567)、空 `required` 陣列剔除 (#52106) |
| Mistral | 跨 proxy transport 相容性 |
| xAI | Responses 整合（見下方） |
| Amazon Bedrock | AWS SDK 版本鎖定 |
| Moonshot/Kimi | Code 恢復 (#54619)、stream wrapper 解耦 |
| Microsoft Foundry | 全新 Entra ID 認證 |
| Copilot | Auth refresh overflow 修正 (#55360) |

### 模型容錯備份

```
fix: make overload failover configurable
bef4fa55f5 fix(model-fallback): add HTTP 410 to failover reason classification (#55201)
984f98be95 Fix: treat HTTP 500 as a transient failover error (#55332)
84401223c7 fix: per-model cooldown scope, stepped backoff, and user-facing rate-limit message (#49834)
```

- 過載重試可配置（`auth.cooldowns`）
- HTTP 410 和 500 加入容錯分類
- Per-model cooldown 範圍和階梯式退避

---

## 10. Plugin SDK 與 Runtime 重構

這是本次版本最大規模的重構，**320+ 個 refactor 提交**：

### Plugin SDK Facades 生成

```
refactor: generate plugin sdk facades (5d3d54ee)
refactor: generate bundled channel seams (a10763e1)
refactor: route plugin sdk through extension barrels
refactor: route plugin-sdk model and whatsapp facades through public barrels
```

Plugin SDK facades 現在自動生成，所有 extension 透過 public barrel 路由。

### Channel Runtime 抽象

```
refactor: route discord channel through outbound runtime
refactor: route slack interactions through channel runtime
refactor: route telegram bot deps through channel runtime
refactor: move discord system events onto channel runtime
refactor: move slack system events onto channel runtime
refactor: move heartbeat helpers onto channel runtime
refactor: move transport readiness onto channel runtime
```

所有核心頻道遷移到統一的 channel runtime 抽象。

### Provider 插件化

```
refactor: move provider runtime into extensions
refactor: finish moving provider runtime into extensions
refactor: move remaining provider policy into plugins
refactor: move provider seams behind plugin sdk surfaces
refactor: generalize provider transport hooks
```

Provider runtime 完全遷移到 extension packages。

### Test 邊界清理

```
refactor: move extension-owned tests to extensions (60+ commits)
test: dedupe plugin * suites (40+ commits)
test: share * fixtures (30+ commits)
```

大量測試重組，確保 extension-owned 覆蓋在 extension 套件中，消除重複。

---

## 11. Exec Approvals 強化

```
Exec approvals: detect command carriers in strict inline eval (#57842)
Exec approvals: reject shell init-file script matches (#58369)
Exec approvals: prevent interpreter allow-always persistence (#57772)
Exec approvals: reject wrapper carrier allow-always targets (#55947)
Security: block exec approval shell carrier targets (#57871)
fix(exec): unwrap arch and xcrun dispatch wrappers (#58203)
fix(exec): harden shell-side approval guardrails (#57839)
refactor(exec): unify channel approvals and restore routing/auth (#57838)
fix(exec): keep awk and sed out of safeBins fast path (#58175)
fix(exec): dedupe Discord approval delivery (#58002)
fix(exec): deliver approval followups directly to chat
```

### 關鍵改進

- **指令載體偵測**：偵測 strict inline eval 中的指令載體
- **Shell init 攔截**：拒絕 shell 初始化檔案腳本匹配
- **Interpreter 持久化防護**：防止 interpreter allow-always 持久化
- **Wrapper 解包**：解包 `arch`、`xcrun`、`caffeinate`、`sandbox-exec` 等
- **safeBins 收緊**：`awk` 和 `sed` 移出低風險快速路徑
- **統一頻道 approvals**：跨頻道統一 approval 路由和授權

---

## 12. xAI 深度整合

```
xAI: switch bundled provider defaults to Responses
Tools: add x_search via xAI Responses
Tools: add xAI-backed code_execution
xAI: route x_search through public api seam
feat(xai): add plugin-owned x_search onboarding
refactor(xai): move x_search into plugin
refactor(xai): move code_execution into plugin
refactor(xai): move bundled xai runtime into plugin
```

### 主要變更

- **Responses API**：xAI bundled provider 預設切換到 Responses
- **x_search**：透過 xAI Responses API 的 web search，移入 plugin 所有權
- **code_execution**：xAI 支持的程式碼執行工具
- **認證統一**：x_search 和 provider 共用 fallback 認證
- **Plugin 所有權**：x_search 和 code_execution 完全移入 xAI plugin

---

## 13. 平台應用

### Android

```
fix: finalize android sms search (#50146)
fix: finalize android notification forwarding controls (#40175)
fix(android): auto-send voice turns on silence
fix(android): require node connection before onboarding finish
fix(android): preserve bootstrap auth for manual reconnect
fix(android): drop bootstrap auth after manual endpoint changes
```

- SMS 搜尋功能完成
- 通知轉發控制（套件過濾、安靜時段、速率限制）
- 語音靜音自動發送
- 節點連接前 onboarding 完成檢查
- Bootstrap auth 手動重連保持

### iOS

```
fix(ios): mark activitykit import as preconcurrency (#57180)
```

ActivityKit import 標記 `@preconcurrency` 以支援 Xcode 26.4 / Swift 6。

### macOS

```
macOS: use MagicDNS for wide-area gateway discovery (#57833)
fix(macos): strip "OpenClaw " prefix before parsing gateway version
build: update macOS package dependencies
build: update Peekaboo for macOS SDK compatibility
```

- MagicDNS 廣域 gateway 發現
- Gateway 版本解析修正
- 套件依賴更新

---

## 14. CLI 與 TUI

### CLI 改進

```
feat(cli): add json schema to cli tool (#54523)
CLI: confirm discovered remote gateways before saving config (#55895)
CLI: reset remote URL after trust decline (#57828)
feat: add anthropic claude cli migration
fix: backfill claude cli chat history
fix(cli): defer zsh compdef registration until compinit is available (#56555)
fix(tui): validate activation slash commands
fix(tui): prune chat log system messages atomically
fix(pi): flush message-boundary block replies on message end
```

- CLI 工具 JSON schema
- 遠端 gateway 發現確認
- Anthropic Claude CLI 遷移
- Zsh 補全延遲註冊
- TUI 斜線指令驗證
- Pi TUI 回覆刷新修正

---

## 15. 測試與 CI

**326 個 test 相關提交** + **32 個 Tests 提交**：

### 測試基礎建設

```
test: introduce planner-backed test runner, stabilize local builds (#54650)
test: improve test runner help text (#55227)
perf: speed up test parallelism (663ba5a)
perf: speed up shared extension test batches (8b42ad0)
perf: speed up channel test runs (339cc33)
perf: speed up core runtime suites (6b6ddcd)
perf: speed up cli and command suites (3f67285)
perf: enable local channel planner parallelism on node 25
```

- Planner-backed 測試執行器
- 測試平行化加速
- Extension/channel/core runtime 套件加速
- Node 25 平行化支援

### 測試穩定化

```
test: stabilize windows * suites (20+ commits)
test: stabilize discord * mocks (10+ commits)
test: stabilize matrix * checks (10+ commits)
fix(ci): rebalance * timing (10+ commits)
```

大量 Windows CI、Discord、Matrix 測試穩定化工作。

### 測試去重

```
test: dedupe plugin * suites (40+ commits)
test: share * fixtures/helpers (30+ commits)
test: move extension-owned coverage into plugins (10+ commits)
```

---

## 16. 文件與發布

### 文件改進

```
docs: deep audit of memory section
docs: rewrite sessions/memory section
docs: add Background Tasks page
docs: add Honcho memory engine page
docs: add automation hub + Related sections
docs: fix Gateway & Ops audit findings
docs: fix English link audits (#57039)
docs: add Related sections to 30+ pages
docs: add 11 missing config sections to configuration-reference
docs: add complete plugin hook reference
```

- 記憶系統段落深度審計與重寫
- 背景任務頁面
- Honcho 記憶引擎頁面
- 自動化 hub 頁面
- 30+ 頁面新增 Related 段落
- 配置參考新增 11 個缺失段落
- Plugin hook 完整參考

### 發布流程

```
build: prepare 2026.3.31 stable release
build: prepare 2026.3.31-beta.1 release
build: exclude source maps from release tarball
build: refresh plugin sdk api baseline (多次)
build: refresh bundled plugin metadata (多次)
```

- 從 beta.1 到 stable 的完整發布流程
- 版本號推進：2026.3.26 → 2026.3.27 → 2026.3.28 → 2026.3.29 → 2026.3.30 → 2026.3.31

---

## 17. 摘要統計

### 提交分類

| 類別 | 數量 | 百分比 |
|------|------|--------|
| fix | 876 | 41.2% |
| test/Tests | 358 | 16.8% |
| refactor | 320 | 15.1% |
| docs/Docs | 141 | 6.6% |
| feat | 33 | 1.6% |
| build | 34 | 1.6% |
| style | 31 | 1.5% |
| chore | 27 | 1.3% |
| perf | 13 | 0.6% |
| 其他（含模組名前綴） | 292 | 13.7% |

### 檔案變更

- **變更檔案數**：2,843
- **新增行數**：141,654
- **刪除行數**：53,871
- **淨增行數**：87,783

### 社群貢獻

- **社群致謝提交**：65 個
- **社群貢獻者**：49 人

### 主要貢獻者

| 貢獻者 | 提交數 |
|--------|--------|
| Peter Steinberger | 844 |
| Vincent Koc | 275 |
| Tak Hoffman | 255 |
| Gustavo Madeira Santana | 122 |
| Ayaan Zaidi | 121 |
| Jacob Tomlinson | 98 |
| Shakker | 47 |
| huntharo | 27 |
| Vignesh Natarajan | 23 |
| Robin Waslander | 19 |

### 關鍵技術趨勢

1. **Plugin SDK 大規模重構**：provider runtime、channel runtime 全面遷移到 extension packages，SDK facades 自動生成
2. **安全縱深防禦**：exec 環境隔離、sandbox 強化、gateway 認證收緊、頻道授權閘門
3. **背景任務架構**：從 ACP-only 簿記演進為 SQLite-backed 共享控制平面（ClawFlow）
4. **MCP 生態擴展**：SSE/HTTP transport、tool namespacing、channel bridge、plugin tools bridge
5. **CJK 支援完善**：記憶體分塊、MMR tokenization、FTS5 tokenizer、surrogate pair 保護
6. **測試平行化與穩定化**：planner-backed 執行器、大量套件去重、Windows CI 穩定化

### 版本里程碑

| 版本 | 日期 | 性質 |
|------|------|------|
| 2026.3.28 | 2026-03-28 | Stable（起始點） |
| 2026.3.28-beta.1 | — | Beta |
| 2026.3.29 | — | Internal |
| 2026.3.30 | — | Internal |
| 2026.3.31-beta.1 | — | Beta |
| 2026.3.31 | 2026-03-31 | Stable |
