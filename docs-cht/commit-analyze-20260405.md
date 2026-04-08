# OpenClaw 提交分析報告 — 2026-04-05

> **分析範圍**：從 `ff413e312a`（doc: 更新繁體中文技術文件至 v2026.3.31）至 `1eaa6ece04`（2026.4.5 release merge），共 **3,098 個非合併提交**，涉及 **6,437 個檔案**、新增 403,870 行、刪除 282,290 行。

---

## 目錄

1. [重大變更 (Breaking Changes)](#1-重大變更)
2. [新功能 (New Features)](#2-新功能)
3. [安全強化 (Security)](#3-安全強化)
4. [Provider 與模型系統](#4-provider-與模型系統)
5. [記憶系統與 Dreaming](#5-記憶系統與-dreaming)
6. [頻道改進 (Channels)](#6-頻道改進)
7. [TaskFlow 任務系統](#7-taskflow-任務系統)
8. [Plugin SDK 與架構重構](#8-plugin-sdk-與架構重構)
9. [Exec Approvals 與安全閘門](#9-exec-approvals-與安全閘門)
10. [Prompt Cache 穩定性](#10-prompt-cache-穩定性)
11. [平台應用 (Apps)](#11-平台應用)
12. [CLI 與 TUI](#12-cli-與-tui)
13. [Gateway 改進](#13-gateway-改進)
14. [測試與 CI](#14-測試與-ci)
15. [文件與發布](#15-文件與發布)
16. [摘要統計](#16-摘要統計)

---

## 1. 重大變更

### Legacy 配置別名移除

```
fix(config): remove legacy config aliases from public schema
fix(config): surface legacy channel streaming aliases (#60358)
fix(config): migrate legacy sandbox perSession alias (#60346)
fix(config): migrate legacy talk config via doctor (#60333)
fix(config): migrate legacy group allow aliases (#60597)
```

從公開 schema/types/help/baseline 中移除舊版配置別名。舊版配置透過 `openclaw doctor --fix` 進行遷移修復，不再保留在公開 API 表面。影響 `hooks.internal.handlers`、channel `streamMode`、`sandbox.perSession`、talk 配置等。

### ClawFlow 更名為 TaskFlow

```
refactor(tasks): rename flow registry modules to task-flow
docs: rename ClawFlow to TaskFlow and update references
```

先前版本的 ClawFlow 背景任務控制平面正式更名為 **TaskFlow**，所有 CLI 命令、文件、API 隨之更新。

### Plugin SDK Facades 生成器移除

```
refactor: remove plugin sdk facade generator
refactor: remove generated plugin sdk facades
```

先前自動生成的 Plugin SDK facades 被移除，改為手動維護的 public barrel exports，減少建置複雜度。

### 核心 CLI Backends 移除

```
refactor(cli): remove custom cli backends
refactor(cli): delete removed backend files
```

移除自定義 CLI backends 支援，統一透過 plugin 系統處理。

### Plugin 行為移入 Extensions

```
refactor(plugins): move channel behavior into plugins
refactor(plugins): move command ui policy into extensions
refactor(plugins): move outbound dep aliases into extensions
refactor: move bundled replay policy ownership into plugins (#60452)
refactor: move provider discovery config into plugins
refactor: move legacy config migrations under doctor
```

大量核心行為遷移至 extension packages，核心保持 extension-agnostic。Legacy 配置遷移統一收斂至 `openclaw doctor`。

### Exec 預設模式變更

```
feat(exec): default host exec to yolo
```

Host exec 預設模式變更為 `yolo`（自動執行），使用者可透過配置覆蓋。

---

## 2. 新功能

### 2.1 Dreaming 記憶系統

```
feat(memory-core): introduce sleep phases
feat(memory-core): add dreaming promotion with weighted recall thresholds (#60569)
feat(memory-core): add REM preview and safe promotion replay (#61540)
feat(memory-core): add dreaming aging controls
feat(memory-core): add dreaming verbose logging
refactor(memory-core): rename sleep surface back to dreaming
Memory: move dreaming trail to dreams.md (#61537)
Dreaming: simplify sweep flow and add diary surface
Dreaming: move setup controls to header and tighten status plumbing
Dreaming: update multiphase stats and UI polish
Dreaming UI: explain modes on hover in header controls
```

全新的 **Dreaming** 記憶促進系統：
- **Sleep Phases**：多階段記憶處理（類似人類睡眠中的記憶鞏固）
- **Weighted Recall**：加權回憶閾值的記憶提升
- **REM Preview**：安全的預覽和重播機制
- **Aging Controls**：記憶老化控制
- **DREAMS Trail**：記憶追蹤日誌移至 `dreams.md`
- **Dreaming UI**：Control UI 中的 Dreaming 控制面板

### 2.2 音樂生成

```
feat: add music generation tooling
feat: add comfy workflow media support
fix: route comfy music through shared tool
docs: document music generation async flow
docs: improve music generation docs
```

新增音樂生成工具支援，透過 ComfyUI workflow 與共享媒體工具路由。

### 2.3 影片生成

```
feat(video): add runway provider
feat(video): add xai and alibaba providers
feat(agents): track video generation tasks
feat(agents): detach video generation completion
docs: document runway support
```

新增影片生成功能：
- **Runway** 影片生成提供者
- **xAI** 和 **Alibaba** 影片生成提供者
- 非同步任務追蹤和完成分離
- 影片時長正規化

### 2.4 QA Lab 擴充

```
feat: add qa channel foundation
feat: add qa lab extension
feat(qa): recreate qa lab docker stack
feat(qa): add repo-backed qa suite runner
feat(qa): add live suite runner and harness
feat(qa): improve qa lab debugger ui
```

新增 QA Lab 測試擴充：
- QA 頻道基礎和 Docker stack
- 基於 repo 的測試套件執行器
- Live suite runner 和 harness
- Debugger UI 改進

### 2.5 新 Provider 插件

| Provider | 說明 |
|----------|------|
| **Vydra** | 新增媒體生成提供者 |
| **Fireworks** | 新增 Fireworks 推論提供者 |
| **StepFun** | 新增 StepFun bundled provider plugin (#60032) |
| **Bedrock Mantle** | OpenAI-compatible Bedrock 提供者 (#61296) |
| **Bedrock IAM** | IAM credential auth (#61563) |
| **Bedrock Guardrails** | Guardrails 支援 (#58588) |
| **Bedrock Inference Profiles** | 推論 profile 發現和 region injection (#61299) |

### 2.6 新 Web Search 提供者

```
feat(web-search): add SearXNG as bundled web search provider plugin (#57317)
feat(tools): add MiniMax as bundled web search provider
feat(ollama): add bundled web search provider (#59318)
```

新增 SearXNG、MiniMax 和 Ollama 作為 bundled web search 提供者。

### 2.7 原生聊天 Approvals

```
feat(approvals): auto-enable native chat approvals
Matrix: add native exec approvals (#58635)
```

原生聊天 approval 自動啟用，Matrix 頻道新增原生 exec approval 支援（含 reaction 快捷鍵）。

### 2.8 Android Assistant 整合

```
feat: add Android assistant role entrypoint
feat: route Android assistant launches into chat
feat: add Google Assistant App Actions entrypoint
feat: auto-send Android assistant prompts
```

Android 新增 Google Assistant App Actions 入口，可直接將語音指令路由到 OpenClaw 聊天。

### 2.9 控制 UI 多語系

```
feat(ui): add control ui locale sync pipeline
feat(ui): add tr id and pl control ui locales
feat(i18n): add Ukrainian docs and control UI locale
```

Control UI 新增自動化多語系同步管線，支援土耳其語、印尼語、波蘭語、烏克蘭語等新語系。

### 2.10 其他新功能

| 功能 | 說明 |
|------|------|
| Prompt cache diagnostics | 新增 prompt cache break 診斷 (#60707) |
| Configurable context visibility | 可配置的 context visibility (#35e1605) |
| Chat-native task board | 聊天原生任務面板 (#58828) |
| Gateway chat history max chars | 可配置的聊天歷史最大字元數 (#58900) |
| Heartbeat task batching | 透過 HEARTBEAT.md 的任務批次支援 |
| Inherited agent skill allowlists | Agent skill 繼承允許清單 (#59992) |
| Cron --tools flag | 每個 cron job 的工具允許清單 (#58504) |
| agents.defaults.params | 全域預設 provider params (#58548) |
| agents.defaults.compaction.notifyUser | 壓縮通知使用者配置 (#54251) |
| before_agent_reply hook | Reply claiming pattern hook (#20067) |
| Reply dispatch hook | Plugin reply dispatch hook |
| Plugin config TUI | Plugin 配置 TUI 整合到 onboard/configure (#60590) |
| Rich config schema descriptions | JSON Schema rich description (#60067) |
| Wildcard peer bindings | `peer.id="*"` 多 agent 路由 (#58609) |
| ClawHub skill search | Control UI 中的 ClawHub 搜尋 (#60134) |
| Structured plan updates | 實驗性結構化計畫更新 |
| Provider system prompt contributions | Provider 擁有的 system prompt 貢獻 |
| GPT personality overlay | 可選的 GPT 個性覆蓋 |
| Slack scoped prompts | Slack scoped prompts 和 mrkdwn hints (#59100) |
| Diffs configurable viewer URL | Diffs 可配置的 viewer base URL (#59341) |
| TinyFish browser plugin | TinyFish bundled browser automation (#58645) |
| Feishu comment event | Feishu 留言事件支援 (#58497) |
| WhatsApp reaction levels | WhatsApp reaction 指導等級 (#58622) |
| Async media delivery flag | 非同步媒體直接投遞旗標 |
| Gemini managed prompt caching | Google managed prompt caching |
| Model request transport overrides | 模型請求傳輸覆蓋 (#60200, #60327) |
| LLM transport adapter seam | LLM 傳輸轉接器 (#60264) |
| OpenAI default prompt overlay | OpenAI 預設 prompt overlay |
| Claude CLI JSONL streaming | Claude CLI JSONL 輸出串流 |
| Media request transport overrides | 媒體請求傳輸覆蓋 (#59848) |
| Qwen provider and video generation | Qwen 提供者和影片生成 |
| MiniMax TTS provider | MiniMax native TTS (#55921) |
| SOUL personality guide | 新增 SOUL 個性指南文件 |
| Lobster managed TaskFlow | Lobster 管理的 TaskFlow 模式 (#61555) |
| Lobster in-process workflows | Lobster 程序內工作流 (#61523) |

---

## 3. 安全強化

本次版本包含大量安全相關提交：

### 3.1 Exec 安全

```
fix(security): close fail-open bypass in exec script preflight (#59398)
fix(exec): hide windows console windows
fix(exec): clarify auto routing semantics (#58897)
Security: block exec host overrides under auto target
fix(windows): reject unresolved cmd wrappers (#58436)
fix: preserve strict inline-eval approval boundaries (#59780)
fix: evaluate shell wrapper inline commands against allowlist (#57377)
fix(dotenv): block helper interpreter workspace overrides (#58473)
Block remaining host env override pivots (#59233)
fix(exec): keep strict inline-eval interpreter approvals reusable
```

- **Fail-open 修復**：關閉 exec script preflight 中的 fail-open 繞過
- **Windows 安全**：隱藏 console windows、拒絕未解析的 cmd wrappers
- **Auto target 保護**：封鎖 exec host override 在 auto target 下的使用
- **Inline-eval 邊界**：保持 strict inline-eval approval 邊界
- **Shell wrapper 檢查**：shell wrapper inline 命令對照 allowlist 評估
- **Dotenv 隔離**：封鎖 helper interpreter workspace 環境變數覆蓋

### 3.2 Browser 安全

```
fix(browser): block SSRF redirect bypass via real-time route interception (#58771)
fix(browser): validate profile cdp urls
fix(browser): validate initial cdp endpoints
fix(browser): validate cdp websocket pivots
fix: harden direct CDP websocket validation (#60469)
```

- **SSRF 防護**：透過即時路由攔截封鎖 SSRF redirect 繞過
- **CDP 驗證**：全面驗證 CDP URLs、endpoints 和 websocket pivots

### 3.3 Gateway 認證

```
fix(gateway): enforce session kill HTTP scopes (#59128)
Gateway: disconnect shared-auth sessions on auth change
Gateway: avoid secret-ref auth disconnect churn
fix(gateway): skip restart when config.patch has no actual changes (#58502)
fix(gateway): return default scopes when trusted HTTP request has no scope header (#58603)
fix(gateway): revoke bootstrap tokens after handshake commit
fix: enforce node pairing approval scopes end-to-end (#60461)
fix: finalize device-pair scope hardening (#55996)
fix: harden paired-device management authz (#50627)
```

- **Session scope 強化**：session kill 操作強制 HTTP scope 檢查
- **Shared-auth 隔離**：auth 變更時斷開 shared-auth sessions
- **Bootstrap token**：handshake 完成後撤銷 bootstrap tokens
- **設備配對**：端到端配對 approval scope 強制執行
- **設備管理授權**：拒絕未核准的 token 角色

### 3.4 頻道安全

```
fix: bound discord inbound media downloads (#58593)
fix(qqbot): restrict structured payload local paths (#58453)
fix(qqbot): require explicit allowlist for /bot-logs (#58895)
Slack: filter thread context by allowlist (#58380)
Require owner access for /allowlist writes (#59836)
Limit connect snapshot metadata to admin-scoped clients (#58469)
```

### 3.5 Sandbox 安全

```
fix(sandbox): block home credential binds (#59157)
fix(sandbox): use browser image for browser runtime matching (#58759)
fix(sandbox): resolve pinned fs helper python without PATH (#58573)
```

### 3.6 其他安全修正

```
fix(skills): block UV_PYTHON in workspace dotenv (#59178)
fix(hooks): harden before_tool_call hook runner to fail-closed (#59822)
fix: harden embedded text normalization (#58555)
fix(security): prevent memory exhaustion in inline image decoding (#22325)
fix(agents): close remaining internal-context leak paths (#59649)
fix: wrap untrusted file inputs (#60277)
iOS: restrict A2UI action dispatch to trusted canvas URLs (#58471)
fix(android): require TLS for remote gateway endpoints (#58475)
```

---

## 4. Provider 與模型系統

### 4.1 Provider 架構大重構

```
refactor(providers): centralize request capabilities (#59636)
refactor(providers): centralize stream request headers (#59542)
refactor(providers): centralize media request shaping (#59469)
refactor(providers): centralize request attribution policy (#59433)
refactor(providers): centralize native provider detection (#60341)
refactor(providers): centralize compat endpoint detection (#60399)
refactor(providers): share transport stream helpers
refactor(providers): share anthropic payload policy
refactor(providers): share replay and tool compat helpers (#60637)
refactor(providers): compose provider stream wrappers
refactor(providers): add family replay and tool hooks
refactor(providers): add stream family hooks
refactor(providers): share native streaming compat helpers
refactor(providers): flatten passthrough provider hooks
refactor(providers): move defaults and error policy into plugins
refactor(providers): add internal request config seam (#59454)
refactor(providers): normalize transport policy wiring (#59682)
refactor(providers): unify request policy resolution (#59653)
```

Provider 系統經歷大規模集中化和模組化重構：
- **Request 集中化**：capabilities、headers、attribution、media shaping 統一
- **Stream 家族系統**：stream wrapper 組合化，共享 stream family hooks
- **Replay/Tool 相容**：跨 provider 共享 replay 和工具相容輔助
- **Transport 統一**：transport policy 正規化和統一解析
- **Plugin 所有權**：預設值和錯誤策略移入 plugin

### 4.2 Provider 懶加載

```
refactor(anthropic): lazy-load provider registration
refactor(openai): lazy-load provider registration
refactor(amazon-bedrock): lazy-load provider registration
refactor(openrouter): lazy-load provider registration
refactor(microsoft): lazy-load speech provider
refactor(browser): lazy-load plugin registration
refactor(github-copilot): lazy-load provider registration
refactor(vllm): lazy-load provider registration
refactor(acpx): lazy-load runtime service entry
```

所有 provider plugin 改為懶加載，顯著減少啟動時間和記憶體使用。

### 4.3 具體 Provider 修正

| Provider | 修正 |
|----------|------|
| Anthropic | 思考恢復 crash (#59062)、cache retention 語義 (#60370, #60375)、cache boundary gaps (#60691)、prompt cache fingerprints (#60731) |
| OpenAI | GPT personality overlay、codex gpt-5.4-mini、Responses phase 支援、websocket transport 強化 |
| Google/Gemini | CLI OAuth (#61260)、2.5 model ids (#61261)、managed prompt caching、image generation DNS fix (#59873)、cached content (#60757) |
| Ollama | Tool-call replay 強化 (#52253)、bundled web search (#59318) |
| MiniMax | TTS (#55921)、coding-plan usage (#52349)、M2.7 image capability (#54843)、reasoning leak 修正 (#55809)、API host 尊重 (#34524) |
| Moonshot/Kimi | Anthropic tool payloads (#59440, #60391)、K2.5 model fallback、web search 修正 (#59356)、tool args repair |
| xAI | x_search auth plugin-owned (#59691)、x_search config behind plugin boundary |
| Copilot | IDE auth headers on runtime requests (#60755) |
| Bedrock | Guardrails (#58588)、IAM auth (#61563)、inference profiles (#61299)、Mantle provider (#61296)、AWS SDK injection (#61194) |
| StepFun | 全新 bundled provider (#60032) |
| Fireworks | 全新 provider |
| Qwen | 全新 provider + video generation |
| OpenRouter | Cache markers (#60761)、403 billing fallback (#59892) |
| Microsoft Foundry | Image capability preservation |

### 4.4 Failover 改進

```
fix: escalate to model fallback after rate-limit profile rotation cap (#58707)
fix(agents): classify generic provider errors for failover (#59325)
fix(failover): scope openrouter-specific matchers (#60909)
fix(failover): classify AbortError / stream-abort as timeout (#58315)
fix(failover): OpenRouter 403 Key limit exceeded triggers billing fallback (#59892)
fix: differentiate overloaded vs rate-limit user-facing messages (#58562)
```

- Rate-limit profile rotation cap 後自動升級到 model fallback
- 通用 provider 錯誤分類為 failover
- AbortError/stream-abort 分類為 timeout
- OpenRouter 403 觸發 billing fallback
- 區分 overloaded 和 rate-limit 使用者訊息

---

## 5. 記憶系統與 Dreaming

### 5.1 Dreaming 系統（全新）

Dreaming 是本次版本最大的記憶系統創新：

- **Sleep Phases**：引入多階段記憶處理，模擬人類記憶鞏固
- **Weighted Recall Thresholds**：加權回憶閾值決定記憶提升
- **REM Preview**：安全預覽和重播即將提升的記憶
- **Aging Controls**：控制記憶隨時間的衰減速率
- **Verbose Logging**：詳細的 Dreaming 日誌
- **DREAMS Trail**：記憶追蹤從 session transcript 移至獨立 `dreams.md`
- **Dreaming UI**：Control UI 新增 Dreaming 控制（模式切換、階段統計、hover 說明）
- **Multilingual Promotion**：多語言記憶提升強化 (#60697)

### 5.2 記憶引擎改進

```
feat(memory): add Bedrock embedding provider (#61547)
memory: chunk daily dreaming ingestion (#61583)
memory: trim generic daily chunk headings (#61597)
fix(memory): preserve session indexing during full reindex (#39732)
fix(memory): prefer --mask over --glob for qmd collection pattern flag (#58736)
fix(memory): allow Gemini multimodal fallback before registry hydration (#61085)
fix(memory): avoid recursive provider discovery during register (#61402)
fix(memory): stabilize manager runtime lazy import
fix(memory-qmd): restore qmd compatibility defaults
fix(memory-qmd): streamline compatibility coverage
```

- **Bedrock embedding provider**：新增 Bedrock 嵌入提供者
- **Daily chunking**：每日 dreaming 攝入分塊處理
- **Session indexing**：全重新索引期間保留 session indexing
- **QMD 改進**：collection pattern flag 改用 `--mask`、相容性預設值恢復

### 5.3 記憶系統效能

大量 `perf(memory)` 提交將記憶相關的頻道模組改為懶加載：

```
perf(memory): lazy-load telegram/matrix/slack runtime surfaces (30+ commits)
perf(memory): trim channel import graphs
perf(memory): narrow channel config imports
```

---

## 6. 頻道改進

### 6.1 Matrix

**70+ 個 Matrix 相關提交**：

- **原生 exec approvals**（#58635）含 reaction 快捷鍵
- **Split partial/quiet preview streaming**（#61450）
- **IDB crypto persistence** advisory file locking（#59851）
- **Secret storage recreation** 修復啟動時驗證（#59846）
- **Room allow aliases** 遷移至 enabled（#60690）
- **Spec-compliant mentions** 發射（#59323）
- **Media size limit** 使用者可見標記（#60289）
- **Guided setup flow** 恢復（#59462）
- **Legacy mention edits** 保留
- **Pinned dispatcher runtime** 故障恢復（#61595）
- **Canonical private-network opt-in** 尊重
- **Account scoping** 和 default detection 收緊

### 6.2 Telegram

- **Reaction ownership** 跨重啟持久化（#59207）
- **Explicit topic targets** 回覆保留（#59634）
- **Media placeholder** 和 file_id 下載失敗時顯示（#59948）
- **Buffered apiRoot downloads** 覆蓋（#59544）
- **HTML formatting** model switch 訊息（#60042）
- **Native progress placeholder** plugin 命令 opt-in（#59300）
- **RFC2544 SSRF** vs fake-IP 指導文件澄清
- **Dangerous private-network media** opt-in
- **Local Bot API media roots** 信任（#60705）
- **Paginated commands** 強制 callbacks
- **Allow-always callback alias** 保留
- **Error suppression controls**（#51914）
- **Models picker** 完整 provider/model 比較（#60384）
- **Reasoning previews** 限制到 stream sessions（#61266）

### 6.3 Discord

- **Native command DM allowlist** 尊重
- **Component-only media** 行為保留（#60361）
- **@everyone mention gating**（#60081）
- **Default media cap** 提升
- **Thread starter resolution** forwarded message text（#60139）
- **Voice timeout** 拆分和 main gate 恢復（#60345）
- **Ack reaction auth** 和 gate fallback（#60081）
- **Proxy Carbon REST/webhook/monitor** fetch paths（#57465）
- **Carbon ratelimit signature** drift 支援
- **Explicit reply tags** 投遞尊重
- **Quiet Carbon reconcile** 日誌

### 6.4 Slack

- **Thread context** allowlist 過濾（#58380）
- **Scoped prompts** 和 mrkdwn hints（#59100）
- **Live DM replies** 路由到 channel
- **Pre-set shuttingDown** 防止孤立 ping intervals（#56646）
- **Groups:history scope** app manifest 新增
- **Typing status** indicator 移至 reaction fallback
- **App manifest** 擴展範例和 scope checklist

### 6.5 MS Teams

- **DM inline images** Graph API 下載（#52212）
- **Channel-list/channel-info actions**（#57529）
- **OpenClaw User-Agent** header（#51568）
- **Conversation reference** DM pairing 持久化（#60432）
- **Adaptive Card Action.Submit** invoke activities（#60431）
- **TypingIndicator config** 和防止重複 DM typing（#60771）
- **HttpPlugin** 替換為 httpServerAdapter（#60939）
- **Path-to-regexp crash** Express 5 修正（#55440）
- **DM threading** proactive fallback 保留（#55198）

### 6.6 WhatsApp

- **Reaction guidance levels**（#58622）
- **Block streaming config** 尊重（#60069）
- **Self-chat quoted replies** 群組忽略（#60148）
- **HTML/XML/CSS MIME map** + 未知媒體 fallback（#51562）
- **SelfChatMode** gate connect-time presence（#59410）
- **Watchdog timeout** 重連後重置（#60007）

### 6.7 其他頻道

| 頻道 | 改進 |
|------|------|
| Feishu | Comment event (#58497)、schema 2.0 card config (#53395)、reasoning preview gating (#61271) |
| Mattermost | Groups property config (#57618)、slash command token validation hardening、guard probe fetches |
| BlueBubbles | Action config dedup、monitor normalize、send markdown |
| Signal | UUID helper 拆分、lazy-load send/monitor |
| Line | Dist runtime contract path 修正、lazy-load runtime seams |
| QQ Bot | Silk-wasm lazy-load (#58829)、structured payload path restriction (#58453)、bot-logs allowlist (#58895) |
| Nostr | DM signature verification |
| Voice Call | STT connect timeout clear (#58586)、full config for realtime transcription (#61224) |
| IRC | Runtime API smoke coverage |

---

## 7. TaskFlow 任務系統

### 從 ClawFlow 到 TaskFlow

```
refactor(tasks): rename flow registry modules to task-flow
refactor(tasks): migrate task runtime callsites to task-flow
refactor(tasks): update plugin and acp task-flow consumers
docs: rename ClawFlow to TaskFlow and update references
```

背景任務控制平面正式從 ClawFlow 更名為 TaskFlow。

### TaskFlow 功能增強

```
TaskFlow: add managed child task execution (#59610)
TaskFlow: restore managed substrate (#58930)
Plugins: add bound TaskFlow runtime (#59622)
feat(tasks): add chat-native task board (#58828)
refactor(tasks): rename registry hooks to observers (#59829)
fix(tasks): recheck current state during maintenance sweep
fix(tasks): prevent synchronous task registry sweep from blocking event loop
fix(tasks): filter stale task rows from status cards (#58810)
fix(tasks): hide internal completion wake rows
fix(tasks): harden task-flow restore and maintenance
```

- **Managed child execution**：管理的子任務執行
- **Bound TaskFlow runtime**：plugin 綁定的 TaskFlow runtime
- **Chat-native task board**：聊天原生任務面板
- **Registry observers**：hooks 更名為 observers
- **Stale row 過濾**：status cards 中過濾過時行
- **Event loop 保護**：防止同步 sweep 阻塞 event loop

### 任務資料清理

```
fix: clean up stale cron and chat-backed tasks (#60310)
fix: reconcile stale cron and chat-backed tasks (#60310)
fix(tasks): distill task registry sweep scheduling
```

---

## 8. Plugin SDK 與架構重構

### 8.1 Plugin 系統改進

```
feat(plugins): auto-load provider plugins from model support
feat(plugins): add web fetch provider boundary (#59465)
feat(plugins): surface imported runtime state in status tooling (#59659)
refactor(plugins): separate activation from enablement (#59844)
Plugins: add install --force overwrite flag (#60544)
fix(plugins): keep auto-enabled channels behind allowlists
fix(plugins): preserve allowlist guard for auto-enabled bundled channels (#60233)
fix(plugins): enforce activation before shipped imports (#59136)
fix(plugins): guard runtime facade activation (#59412)
fix(plugins): constrain workspace discovery to .openclaw/extensions
```

- **Auto-load providers**：從模型支援自動載入 provider plugins
- **Activation/Enablement 分離**：activation 與 enablement 狀態分離
- **Install --force**：安裝時覆寫旗標
- **Allowlist 保護**：auto-enabled 頻道仍受 allowlist 保護
- **Workspace 發現**：限制至 `.openclaw/extensions`

### 8.2 Channel Runtime 懶加載

所有核心和 extension 頻道大規模改為懶加載：

```
refactor(discord): lazy-load action/audit/send/directory/resolver/subagent (10+ commits)
refactor(telegram): lazy-load send/action/audit/monitor (5+ commits)
refactor(slack): lazy-load action/send/webhook/directory/target/async (8+ commits)
refactor(whatsapp): lazy-load send/action/login/directory (5+ commits)
refactor(signal): lazy-load send/monitor (3+ commits)
refactor(feishu): split and lazy-load bot/comment/monitor/send/dedup/broadcast (15+ commits)
refactor(line): lazy-load channel/card (3+ commits)
refactor(zalo/zalouser): lazy-load setup/account/action (5+ commits)
refactor(msteams): narrow sdk/store/send/messenger/outbound imports (8+ commits)
```

### 8.3 Provider 插件化收官

```
refactor: move provider replay runtime ownership into plugins (#60126)
refactor(providers): move defaults and error policy into plugins
refactor: move provider discovery config into plugins
refactor(providers): centralize provider model policy
refactor(openai): move native transport policy into extension
refactor: move voice-call realtime providers into extensions
refactor(providers): add anthropic/google transport runtime
```

Provider runtime 完全遷移至 extension packages，核心不再擁有 provider-specific 行為。

### 8.4 Test 邊界清理（續）

```
test: centralize inbound/outbound/provider/channel contracts (20+ commits)
test: consolidate plugin/package manifest contracts (10+ commits)
test: move extension-owned coverage out of core (15+ commits)
test: split contract lanes (30+ commits)
test: localize suite helpers (20+ commits)
test: reduce partial mocks (80+ commits)
test: trim importOriginal/importActual usage (30+ commits)
```

繼上一版本的測試邊界清理，本次進一步將 extension-owned 測試覆蓋移至 extension 套件，大量消除 partial mocks 和 importOriginal 使用。

---

## 9. Exec Approvals 與安全閘門

### Exec Approval 改進

```
fix(exec): resume agent session after approval completion
fix(exec): resolve remote approval regressions (#58792)
fix(exec): strip invalid approval policy enums during config normalization (#59112)
Exec approvals: unify effective policy reporting and actions (#59283)
Exec approvals: fix policy source attribution (#59367)
fix(approvals): use canonical decision values in interactive button payloads
fix(approvals): suppress manual native approval narration
fix: decouple approval availability from native delivery enablement (#59620)
Approvals: scope foreign-channel account routing (#60417)
```

### Windows Exec 支援

```
fix(exec): implement Windows argPattern allowlist flow
fix(exec): make Windows exec hints accurate and dynamic
fix: land Windows exec allowlist (#56285)
fix(exec): ignore malformed drive-less windows exec paths
fix(exec): hide windows console windows
refactor: centralize Windows exec invocation
```

Windows 平台獲得完整的 exec allowlist 支援，包含 argPattern flow 和動態提示。

### 跨頻道 Approval 改善

```
Refactor channel approval capability seams (#58634)
fix(telegram): preserve allow-always callback alias
fix: cross-provider approval availability coverage (#59776)
fix(agents): honor cacheRetention for custom anthropic providers (#59049)
fix(exec): reuse gateway allow-always approvals
```

---

## 10. Prompt Cache 穩定性

本次版本將 Prompt Cache 穩定性提升為正式的正確性/效能關鍵領域：

```
fix(cache): sort MCP tools deterministically to stabilize prompt cache (#58037)
fix(cache): compact newest tool results first to preserve prompt cache prefix (#58036)
fix(cache): delay history image pruning to preserve prompt cache prefix (#58038)
fix(cache): preserve full 3-turn history image cache window (#60603)
fix(cache): enable prompt cache retention for Anthropic Vertex AI (#60888)
fix(cache): order stable project context before heartbeat (#61236)
fix(agents): split system prompt cache prefix by transport (#59054)
fix(agents): close remaining prompt cache boundary gaps (#60691)
fix(agents): stabilize prompt cache fingerprints (#60731)
feat(agents): add prompt cache break diagnostics (#60707)
docs: add prompt cache stability rules
```

- **MCP 工具排序**：確定性排序以穩定 prompt cache
- **壓縮策略**：優先壓縮最新 tool results 保留 cache prefix
- **圖片修剪延遲**：延遲歷史圖片修剪以保護 cache prefix
- **3-turn 圖片窗口**：保留完整 3-turn 歷史圖片 cache 窗口
- **Vertex AI cache**：Anthropic Vertex AI 啟用 prompt cache retention
- **穩定 context 排序**：project context 排在 heartbeat 之前
- **Transport 分離**：依 transport 分離 system prompt cache prefix
- **Break 診斷**：新增 prompt cache break 診斷工具

---

## 11. 平台應用

### Android

```
feat: add Android assistant role entrypoint
feat: route Android assistant launches into chat
feat: add Google Assistant App Actions entrypoint
feat: auto-send Android assistant prompts
feat(android): add talk.speak playback path
fix(android): require TLS for remote gateway endpoints (#58475)
fix(android): allow cleartext LAN gateways
fix: restore android talk mode reply tts (#60306)
fix: cancel in-flight Android talk playback on stop (#61164)
fix(android): delay operator bootstrap reconnect until stored auth
fix: use talk.speak for Android replies (#60954)
```

- **Google Assistant 整合**：App Actions 入口和自動發送
- **Talk.speak 播放路徑**：新增 talk.speak 播放和 reply speaker 路由
- **TLS 強制**：遠端 gateway endpoints 要求 TLS
- **LAN cleartext**：允許 LAN 明文 gateway
- **Talk mode TTS**：恢復 reply TTS

### iOS

```
feat(ios): add exec approval notification flow (#60239)
iOS: restrict A2UI action dispatch to trusted canvas URLs (#58471)
```

- **Exec approval 通知**：iOS 新增 exec approval 通知流程
- **A2UI 安全**：限制 A2UI action dispatch 到信任的 canvas URLs

### macOS

```
fix: recover unloaded macOS launch agents (#43766)
fix(gateway): use launchd KeepAlive restarts
fix(gateway): restart watch after child sigterm
fix: windows self-restart stale gateway cleanup (#60480)
fix: PID recycling detection in gateway lock (#59843)
fix: improve parallels smoke progress
```

- **Launch agent 恢復**：修復未載入的 macOS launch agents
- **LaunchD KeepAlive**：使用 launchd KeepAlive 重啟
- **PID 回收偵測**：gateway lock 中偵測 PID 回收
- **Windows 重啟**：self-restart stale gateway 清理

---

## 12. CLI 與 TUI

```
fix(cli): set non-zero exit code on argument errors (#60923)
fix(cli): narrow post-update root
fix(cli): route skills list output to stdout when --json is active (#60914)
fix(cli): add local logs fallback
fix(cli): preserve claude cache creation tokens
fix(cli): keep status json startup lean
fix(tui): preserve pending sends and busy-state visibility (#59800)
fix(tui): tolerate clock skew in pending-history reconciliation
fix: trim menu descriptions before dropping commands (#61129)
fix: exit after package-to-git handoff
fix: stop old cli after package-to-git switch
feat(config): add rich description fields to JSON Schema output (#60067)
```

- **Exit code**：argument 錯誤時設定非零 exit code
- **Skills list JSON**：`--json` 時輸出到 stdout
- **Local logs fallback**：新增本地日誌 fallback
- **Package-to-git**：切換後停止舊 CLI 並退出
- **TUI 可見性**：保持 pending sends 和 busy-state 可見
- **Config schema**：JSON Schema 新增 rich description

---

## 13. Gateway 改進

```
fix(gateway): prune empty node-pending-work state entries (#58179)
fix(gateway): stop pinning node commands to pairing state
Gateway: bound websocket shutdown close (#61565)
fix(gateway): fail closed on missing mode
fix: default gateway.mode to 'local' when unset (#54801)
fix: improve WS handshake reliability on slow-startup (#60075)
Gateway: refresh websocket auth after secrets reload (#60323)
Gateway: keep outbound session metadata in owner store
fix: avoid startup gateway reload loop (#58678)
fix: catch per-stage errors in HTTP request pipeline (#58689)
fix: skip failing gateway HTTP stages (#58746)
fix(gateway): prefer bootstrap auth over tailscale (#59232)
fix(gateway): emit before_reset on session reset (#53872)
fix: prevent duplicate gateway watchers
fix: prevent duplicate block reply delivery for text_end channels (#61530)
```

- **Memory leak 修正**：清理空的 node-pending-work state entries
- **WebSocket 改進**：shutdown close 邊界、慢啟動環境的握手可靠性
- **Auth 改進**：secrets reload 後刷新 websocket auth
- **預設 local 模式**：未設定時預設 gateway.mode 為 'local'
- **HTTP pipeline**：捕獲 per-stage 錯誤防止 cascade 500s
- **Bootstrap auth**：優先於 tailscale

---

## 14. 測試與 CI

**580 個 test 相關提交 + 89 個 perf 提交**：

### 測試架構革新

```
refactor: remove custom test planner runtime
test: use native vitest root projects
test: default vitest root projects to threads
test: split vitest setup for projects
test: default vitest lanes to isolated forks
fix(test): default local Vitest to one worker (#60281)
perf: split hooks, tui, extension, infra, tooling, provider lanes
perf: route targeted tests to scoped vitest configs
```

- **移除自定義 planner**：改用 Vitest 原生 root projects
- **Thread-first 預設**：預設使用 threads，部分套件例外使用 forks
- **單 worker 預設**：本地開發預設一個 worker
- **Scoped lanes**：按模組拆分 vitest 配置和測試 lane

### 大規模測試效能優化

```
perf(test): trim hotspot reload churn (30+ commits)
perf(test): isolate memory-heavy extension hotspots
perf(test): narrow sdk seams for channel hotspots
test: reduce partial mocks (80+ commits)
test: trim importOriginal/importActual usage (30+ commits)
test: localize suite helpers (20+ commits)
```

### Channel 和 Extension 測試懶加載

所有主要頻道的測試改為懶加載模式：

```
test(feishu): slim bot/comment/monitor/broadcast runtime fixtures (15+ commits)
test(bluebubbles): split webhook/monitor/action/status seams (10+ commits)
test(msteams): avoid loading graph module in tests (5+ commits)
test(matrix): avoid loading send/action modules in tests (5+ commits)
test(discord): defer provider runtime mocks (5+ commits)
```

### Contract 測試集中化

```
test(contracts): split provider/channel/plugin/setup/config lanes (40+ commits)
test(contracts): localize registry/surface/session helpers (15+ commits)
test(contracts): lazy-load test facades (10+ commits)
```

---

## 15. 文件與發布

### 文件改進

```
docs: add prompt cache stability rules
docs: add SOUL personality guide
docs(memory): add Dreaming concept page and overview links
docs: expand dreaming memory documentation
docs: restructure automation section as Automation & Tasks
docs: rewrite automation decision guide
docs: refresh 250+ doc mirror references
docs: expand cli/security/webhook/maintenance/sessions references
docs: add generated locale picker support
docs: clarify anthropic cli/migration/fallback guidance
docs: document music generation async flow
docs: document runway support
docs: add video generation provider pages
docs: expand model fallback guide
```

- **Prompt cache 規則**：新增 prompt cache 穩定性規則文件
- **SOUL 指南**：新增個性設定指南
- **Dreaming 文件**：概念頁面和概覽連結
- **自動化重組**：Automation & Tasks 段落重組
- **250+ mirror 刷新**：大量文件交叉參考更新
- **多語系**：locale picker 支援

### 版本發布

```
build: bump version to 2026.4.1 → 2026.4.2 → 2026.4.3 → 2026.4.4
chore: prepare 2026.4.1 release
chore: prepare 2026.4.6-beta.1 release
chore: release 2026.4.5
```

- 從 2026.3.31 至 2026.4.5 共 5 個版本
- 版本 2026.4.1 ~ 2026.4.5 stable + 2026.4.6-beta.1

---

## 16. 摘要統計

### 提交分類

| 類別 | 數量 | 百分比 |
|------|------|--------|
| fix | 1,155 | 37.3% |
| test | 580 | 18.7% |
| docs | 511 | 16.5% |
| refactor | 339 | 10.9% |
| feat | 108 | 3.5% |
| perf | 89 | 2.9% |
| chore | 76 | 2.5% |
| ci | 57 | 1.8% |
| style | 31 | 1.0% |
| build | 14 | 0.5% |
| 其他 | 138 | 4.5% |

### 檔案變更

- **變更檔案數**：6,437
- **新增行數**：403,870
- **刪除行數**：282,290
- **淨增行數**：121,580

### 社群貢獻

- **社群致謝提交**：113 個
- **社群貢獻者**：80 人

### 主要貢獻者

| 貢獻者 | 提交數 |
|--------|--------|
| Peter Steinberger | 1,552 |
| Vincent Koc | 662 |
| Shakker | 137 |
| Tak Hoffman | 131 |
| Ayaan Zaidi | 80 |
| Gustavo Madeira Santana | 64 |
| Agustin Rivera | 28 |
| Chinar Amrutkar | 12 |
| Mariano | 12 |
| @zimeg | 12 |
| Brad Groux | 11 |

### 關鍵技術趨勢

1. **Dreaming 記憶系統**：全新的多階段記憶促進系統（sleep phases、weighted recall、REM preview、aging controls）
2. **Provider 架構集中化**：request capabilities/headers/attribution/media/transport 統一；stream family hooks；所有 provider 懶加載
3. **TaskFlow 成熟**：ClawFlow → TaskFlow 更名、managed child execution、chat-native task board、bound runtime
4. **Prompt Cache 正式化**：升級為正確性/效能關鍵領域，新增診斷工具和穩定性規則
5. **多媒體生成**：音樂生成（ComfyUI）、影片生成（Runway/xAI/Alibaba/Qwen）
6. **大規模懶加載**：所有 provider plugins + 所有核心頻道 runtime 改為懶加載
7. **測試革新**：移除自定義 planner 改用 Vitest 原生 projects，80+ partial mock 消除
8. **Legacy 清理**：舊版配置別名從公開 schema 移除，遷移收斂至 doctor
9. **Windows exec**：完整 argPattern allowlist flow 和動態提示
10. **Control UI 多語系**：自動化 locale sync pipeline，新增 5+ 語系

### 版本里程碑

| 版本 | 日期 | 性質 |
|------|------|------|
| 2026.3.31 | 2026-03-31 | Stable（起始點） |
| 2026.4.1-beta.1 | — | Beta |
| 2026.4.1 | — | Stable |
| 2026.4.2-beta.1 | — | Beta |
| 2026.4.2 | — | Stable |
| 2026.4.3 | — | Internal |
| 2026.4.4 | — | Internal |
| 2026.4.5 | 2026-04-05 | Stable |
| 2026.4.6-beta.1 | — | Beta |
