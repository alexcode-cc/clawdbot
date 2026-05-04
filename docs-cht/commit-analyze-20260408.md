# OpenClaw 提交分析報告 — 2026-04-08

> **分析範圍**：從 `1ec1af7916`（doc: 更新繁體中文文件版本號至 2026.4.5）至 `abe7b2c49d`（chore: sync 2026.4.8 config docs baseline），共 **1,705 個非合併提交**，涉及 **4,982 個檔案**、新增 209,588 行、刪除 105,400 行。

---

## 目錄

1. [重大變更與安全修復](#1-重大變更與安全修復)
2. [新功能 (New Features)](#2-新功能)
3. [Memory Wiki 與 Dreaming 改善](#3-memory-wiki-與-dreaming-改善)
4. [Provider 與模型系統](#4-provider-與模型系統)
5. [頻道改進 (Channels)](#5-頻道改進)
6. [QA Lab 強化](#6-qa-lab-強化)
7. [Plugin SDK 與架構](#7-plugin-sdk-與架構)
8. [Exec Approvals 與安全閘門](#8-exec-approvals-與安全閘門)
9. [Gateway 改進](#9-gateway-改進)
10. [平台應用 (Apps)](#10-平台應用)
11. [CLI 與 TUI](#11-cli-與-tui)
12. [大規模程式碼清理](#12-大規模程式碼清理)
13. [測試與 CI](#13-測試與-ci)
14. [文件與發布](#14-文件與發布)
15. [摘要統計](#15-摘要統計)

---

## 1. 重大變更與安全修復

### 網路安全強化（Proxy / SSRF）

```
fix(slack): honor HTTPS_PROXY for Socket Mode WebSocket connections
fix: address review — honor NO_PROXY, guard malformed URLs
fix: handle leading-dot NO_PROXY entries matching apex domain
fix(net): skip DNS pinning before trusted env proxy dispatch
fix(gateway): stop SSRF guard rejecting operator-configured proxy hostnames (#62312)
fix(browser): harden SSRF redirect guard against non-navigation document hops [AI] (#62355)
fix: centralize HTTP/1.1 SSRF dispatchers (#61777) (thanks @zozo123)
fix(fetch-guard): drop request body on cross-origin unsafe-method redirects [AI-assisted] (#62357)
```

這是本次版本的一大安全主題。修復了多處網路安全問題：

- **Slack Proxy 支援**：Slack Socket Mode WebSocket 現正確遵從 `HTTPS_PROXY` 環境變數，企業環境中的 Slack 連線不再被防火牆阻擋
- **NO_PROXY 語義修正**：正確處理以 `.` 開頭的 NO_PROXY 條目匹配 apex 域名，以及格式異常的 URL
- **SSRF 防護增強**：
  - DNS pinning 在可信代理環境中跳過，避免代理主機名被誤判
  - Gateway SSRF guard 不再拒絕操作者配置的合法代理主機名
  - 瀏覽器 SSRF 重定向防護針對非導航文件跳轉加強
  - 跨域 unsafe-method 重定向時移除請求 body
  - HTTP/1.1 SSRF 分發器集中化管理

### 環境變數過濾強化

```
fix(env): align inherited host exec env filtering (#59119)
fix(git): expand host env denylist coverage (#62002)
fix: expand host-exec env blocklist for Java, Rust, and Cargo toolchains [AI-assisted] (#62291)
```

- 擴大 host exec 環境變數封鎖清單，新增 Java、Rust、Cargo 工具鏈的敏感變數
- 對齊子進程繼承的 host exec 環境變數過濾邏輯

### Base64 / 資料完整性

```
Guard missed base64 decode paths (#62007)
fix: unify assistant visible text sanitizers (#61729)
fix: suppress commentary history leaks (#61747) (thanks @afurm)
```

- 修補遺漏的 base64 解碼路徑安全防護
- 統一 assistant 可見文字清理器，防止內部標記洩漏
- 修復 commentary 歷史記錄洩漏問題

### Gateway 安全修復

```
fix(gateway): invalidate shared-token/password WS sessions on secret rotation [AI] (#62350)
Protect gateway exec approval config paths (#62001)
Require re-pairing for node reconnect command upgrades (#62658)
fix: tighten container bind defaults for landing (#61818) (thanks @openperf)
```

- Secret rotation 後立即失效 shared-token/password WebSocket sessions
- Gateway exec approval 配置路徑保護
- Node 重連 command 升級需重新配對
- 收緊容器綁定預設值

---

## 2. 新功能

### Infer CLI — 一級推理工作流

```
feat: Add first-class infer CLI for inference workflows (#62129)
feat: unify live cli backend probes
docs: rename and improve infer docs
docs infer cli examples and alias note
```

新增 `openclaw infer` 指令，提供一級推理工作流介面。允許直接對模型發送單次推理請求，無需啟動完整 Agent 對話循環。適合腳本化批次處理、管線整合、和快速模型測試。

### Pluggable Compaction Provider

```
feat: add pluggable compaction provider registry (#56224)
feat(gateway): add compaction checkpoints (#62146)
fix: compaction after tool use abortion cause agent infinite loop calls (#62600)
```

- **可插拔壓縮提供者**：compaction 現透過 registry 支援不同的 LLM provider 進行上下文壓縮，不再限於主要模型
- **Compaction 檢查點**：Gateway 新增 compaction checkpoints，斷線後可從檢查點恢復
- **修復工具中斷後壓縮無限迴圈**：tool use 被中止後的 compaction 不再觸發 agent 無限迴圈呼叫

### Prompt Cache Context Engine

```
feat: expose prompt-cache runtime context to context engines (#62179)
feat(context-engine): add memory prompt helper
```

- 將 prompt-cache 執行時期 context 暴露給 context engines，讓 context engine 可以利用快取資訊優化內容組裝
- 新增 memory prompt helper，讓 context engine 更有效地注入記憶內容

### Agent Prompt Override / Heartbeat Controls

```
feat(agents): add prompt override and heartbeat controls
fix(agents): heartbeat always targets main session — prevent routing to active subagent sessions (#61803)
fix: respect disabled heartbeat guidance
```

- 新增 Agent prompt override 機制，允許在 runtime 動態覆蓋 agent 提示詞
- Heartbeat 控制：支援停用 heartbeat guidance，heartbeat 確保總是回到主 session 而非被路由到活躍的 subagent session

### Discord Event Cover Image

```
feat: add cover image support to Discord event create (#60883)
```

Discord 事件建立現支援封面圖片（cover image），豐富事件展示。

### Slack Thread Explicit Mention

```
feat(slack): add thread.requireExplicitMention config option (#58276)
```

新增 Slack 設定 `thread.requireExplicitMention`，控制在 thread 中是否需要明確 @mention 才觸發回覆。

### Media Provider Capabilities

```
feat: declare explicit media provider capabilities
feat: preserve media intent across provider fallback
```

- 明確宣告每個 media provider 的能力（audio/video/image），讓系統在 fallback 時能保留使用者的媒體意圖
- Provider fallback 時保留媒體意圖，不再因 fallback 而遺失使用者指定的媒體格式

### Video Mode-Aware Generation

```
feat(video): add mode-aware generation capabilities
```

影片生成新增 mode-aware 能力，根據上下文自動選擇適當的生成模式。

### Ollama Vision Detection

```
feat(ollama): detect vision capability from /api/show and set image i… (#62193)
```

Ollama 現可透過 `/api/show` API 自動偵測模型是否支援視覺能力，並自動設定 image input 支援。

### GitHub App Helper

```
feat: add gh-read GitHub app helper
```

新增 `gh-read` GitHub App helper，簡化 GitHub 整合操作。

---

## 3. Memory Wiki 與 Dreaming 改善

### Memory Wiki 系統

```
feat(memory-wiki): restore llm wiki stack
feat(memory-wiki): add belief-layer digests and compat migration
feat(memory-wiki): add claim health reports
feat(memory-wiki): use digests for retrieval
feat(memory-wiki): gate compiled digest prompts
fix(memory-wiki): stabilize compiled digest prompts
docs: add memory wiki docs
```

本次版本大幅發展 **Memory Wiki** 系統，這是在 Dreaming 記憶基礎上建立的知識管理層：

- **Belief-layer Digests**：引入信念層摘要（digests），將記憶片段聚合為結構化的知識摘要，支援向後相容遷移
- **Claim Health Reports**：新增知識聲明（claim）健康報告，追蹤知識片段的可信度和新鮮度
- **Digest 檢索**：搜尋時使用 compiled digests 提升檢索精確度
- **Compiled Digest Prompts**：經過穩定化的編譯摘要提示詞，可在 context engine 中使用

### Dreaming 改善

```
fix: slot-aware dreaming config paths (#62275) (thanks @SnowSky1)
fix(memory): respect memory slot in dreaming config
fix(memory-core): ignore managed dreaming blocks during daily ingestion (#61720) (thanks @MonkeyLeeT)
memory-core: add timestamp bucketing and cursored session ingest
memory-core: harden dreaming session ingestion privacy and idempotence
memory-core: decouple dreaming session ingest from memorySearch flags
Dreaming UI: use slot-aware configured state
```

- **Slot-aware 配置**：Dreaming 配置正確區分不同 memory slot，多 agent 環境下配置互不干擾
- **Session 攝取改善**：新增時間戳分桶（timestamp bucketing）和游標式 session 攝取，提升處理大量 session 的效率
- **隱私與冪等性**：加強 dreaming session 攝取的隱私保護和冪等性
- **解耦 memorySearch 旗標**：dreaming session 攝取不再依賴 memorySearch 設定，獨立運作
- **Daily ingestion 修復**：日常記憶攝取忽略 managed dreaming blocks，避免重複處理

---

## 4. Provider 與模型系統

### Arcee AI 提供者

```
feat: add Arcee AI provider plugin
feat: enhance Arcee AI provider with OpenRouter support and update onboarding instructions
fix: finalize arcee provider landing (#62068) (thanks @arthurbr11)
fix: separate arcee auth envs from openrouter
fix: remove provider hardcoding and fix arcee openrouter
fix: update maxTokens for Arcee model catalog entries
```

新增 **Arcee AI** 提供者插件，支援 OpenRouter 後端路由。感謝社群貢獻者 @arthurbr11 的協助。

### Google Gemma 4

```
feat(google): add support for Gemma 4 models and fix cross-provider resolution
fix(google): preserve Gemma 4 thinking-off semantics (#62411) (thanks @BunsDev)
```

- 新增 Gemma 4 模型支援，修復跨提供者解析
- 保留 Gemma 4 thinking-off 語義

### Z.AI / GLM-5.1

```
fix: align Z.AI endpoint detection with GLM-5.1 default (#61998) (thanks @serg0x)
fix(zai): default to GLM-5.1 instead of GLM-5
fix(zai): update stale glm-5 ref in docs/cli/onboard.md
```

Z.AI 提供者預設模型更新至 GLM-5.1，修正端點偵測邏輯。

### NVIDIA Endpoints

```
refactor(nvidia-endpoints): updated language & default models (#59866)
```

NVIDIA 端點更新語言和預設模型。

### Ollama

```
chore(ollama): update suggested onboarding models (#62626)
feat(ollama): detect vision capability from /api/show and set image i… (#62193)
fix: quiet unconfigured ollama discovery
```

- Ollama onboarding 建議模型更新
- 自動偵測視覺能力
- 靜默未配置的 Ollama 探索

### OpenAI TTS / Codex

```
fix: openai tts groq wav (#62233) (thanks @neeravmakwana)
fix(tts): carry OpenAI talk response format
fix: prefer codex gpt-5.4 runtime metadata (#62694) (thanks @ruclaw7)
```

- OpenAI TTS 修復 Groq wav 格式支援
- 偏好 Codex gpt-5.4 runtime metadata

### Auth 改善

```
fix: honor explicit auth profile selection (#62744)
fix: expose runtime-ready provider auth to plugins (#62753)
fix(doctor): warn when stale Codex overrides shadow OAuth (#40143)
fix(auth): resolve custom env markers dynamically
fix(anthropic): restore OAuth guard in service-tier stream wrappers (#60356)
```

- 遵從明確的 auth profile 選擇
- 向插件暴露 runtime-ready provider auth
- Doctor 在 stale Codex 覆蓋遮蔽 OAuth 時發出警告
- 動態解析自訂環境標記
- 恢復 Anthropic service-tier stream 的 OAuth guard

### 其他模型改善

```
fix: classify HTTP 404 errors for model fallback chain (#62244) (thanks @neeravmakwana)
fix: allowlist compat for capability provider fallback (#62234) (thanks @neeravmakwana)
fix: keep runtime model lookup on configured workspace
fix: preserve fallback error details
```

- HTTP 404 錯誤正確分類以進行 fallback
- 能力提供者 fallback 的白名單相容性
- 保留 fallback 錯誤詳情

---

## 5. 頻道改進

### Slack

```
fix: pass resolved Slack download tokens (#62097) (thanks @martingarramon)
fix(slack): forward resolved botToken to downloadSlackFile
fix(slack): honor HTTPS_PROXY for Socket Mode WebSocket connections
fix: honor Slack Socket Mode env proxies (#62878) (thanks @mjamiv)
Slack: clarify native streaming config hint
Docs: clarify Slack streaming thread behavior
feat(slack): add thread.requireExplicitMention config option (#58276)
fix: preserve Slack guarded media transport (#62239) (thanks @openperf)
fix: preserve slack fallback thread classification (#61835)
fix(config): normalize channel streaming config shape (#61381)
```

- **Proxy 完整支援**：Socket Mode 連線遵從 HTTPS_PROXY / NO_PROXY 環境變數
- **Download Token 修復**：正確傳遞已解析的 Slack download tokens
- **Thread Mention 配置**：新增 `thread.requireExplicitMention` 選項
- **串流配置釐清**：更新文件釐清 Slack native streaming 在 thread 中的行為
- **Media Transport 保護**：Slack guarded media 傳輸修復

### Discord

```
feat: add cover image support to Discord event create (#60883)
fix: harden discord voice receive recovery (#41536) (thanks @wit-oc)
fix(discord): add voice listener compat shim
fix(discord): narrow binding runtime imports
```

- Discord 事件封面圖片支援
- 加強 Discord 語音接收恢復機制
- Voice listener 相容墊片

### Telegram

```
fix(telegram): restore outbound message splitting for long messages (#57816)
fix(telegram): bound startup request timeouts (#61601) (thanks @neeravmakwana)
fix: repair Telegram setup package entry
fix: preserve telegram default auth promotion
```

- 恢復長訊息的出站分割
- Startup 請求超時限制
- 修復 Telegram setup package entry

### Matrix

```
fix(matrix): harden startup auth bootstrap (#61383)
fix(matrix): pass deviceId through health probe to prevent storage-meta overwrite (#61317) (#61581)
Docs: clarify Matrix autoJoin invite scope
Matrix: prompt invite auto-join during onboarding (#62168)
```

- 強化 Matrix 啟動 auth bootstrap
- Health probe 傳遞 deviceId 防止 storage-meta 覆寫
- Onboarding 中提示 invite auto-join

### 其他頻道

```
fix(windows): preserve plugin loader alias resolution (#61832) (thanks @Zeesejo)
fix: normalize cron jobId load path (#62251) (thanks @neeravmakwana)
fix(allowlist): gate write commands behind owner check before channel resolution [AI] (#62383)
```

- Windows plugin loader alias 解析保留
- Cron jobId 載入路徑正規化
- Write commands 在 channel 解析前需 owner 檢查

---

## 6. QA Lab 強化

```
feat(qa-lab): redesign UI with sidebar layout, Slack-like chat, and light/dark mode
feat: add one-command qa lab docker launcher
feat: add interactive qa lab suite runner
feat(qa-lab): add Clawfather/Claw avatars and live-watch mode for scenario runs
feat(qa): add manual harness lane
feat(qa): add frontier harness bakeoff loop
feat(qa): execute ten new repo-backed scenarios
feat(qa): add attachment understanding scenario
feat: auto-reload qa lab fast refresh
feat: add fast qa lab ui refresh mode
feat: streamline qa lab live runs
refactor: split qa scenarios into per-file markdown defs
refactor: move qa suite definitions into markdown
```

QA Lab 在本週期獲得大幅強化：

- **全新 UI 設計**：Sidebar 佈局、Slack 風格聊天介面、Light/Dark 模式切換
- **一鍵 Docker 啟動**：`openclaw qa-lab` 一鍵啟動完整 Docker 環境
- **互動式 Suite Runner**：互動式測試套件執行器
- **Avatar 與 Live-watch**：Clawfather/Claw 角色頭像，場景執行即時觀察模式
- **Manual Harness Lane**：手動測試通道
- **Frontier Bakeoff Loop**：前沿模型對比評估迴圈
- **Fast Refresh**：UI 快速刷新模式，自動重載場景
- **Markdown 場景定義**：場景定義從程式碼遷移至獨立 Markdown 檔案，更易維護
- **10+ 新場景**：新增附件理解等多個測試場景

---

## 7. Plugin SDK 與架構

### Plugin SDK 變更

```
Plugin SDK: split approval adapter seams
perf(plugin-sdk): split channel secret runtime helpers
perf(plugin-sdk): narrow provider contract config types
perf(plugin-sdk): slim provider contract enable path
docs: update plugin sdk api baseline
build: exclude plugin sdk build info from npm pack
```

- 拆分 approval adapter seams，將核准適配器介面模組化
- 拆分 channel secret runtime helpers，降低模組間耦合
- 收窄 provider contract config types，減少不必要的類型暴露
- 精簡 provider contract enable path

### Plugin 系統改善

```
Plugins: verify ClawHub archive integrity (#60517)
plugins: add bundled webhooks TaskFlow bridge (#61892)
fix: preserve plugin runtime registry state
perf(plugins): parallelize boundary canaries
perf(plugins): cache shared boundary freshness scans
perf(plugins): skip fresh boundary plugin compiles
perf(plugins): cache package boundary dts
perf(plugins): parallelize boundary artifact prep
perf(plugins): enable incremental boundary dts prep
perf(plugins): abort failed boundary compile siblings
```

- **ClawHub 完整性驗證**：下載 ClawHub 套件時驗證 archive 完整性
- **Webhooks TaskFlow Bridge**：新增 bundled webhooks TaskFlow bridge 插件
- **Boundary 效能大幅提升**：
  - 平行化 boundary canaries 和 artifact 準備
  - 快取 boundary freshness 掃描結果和 package boundary dts
  - 跳過已新鮮的 boundary plugin 編譯
  - 增量式 boundary dts 準備
  - 失敗時中止同級 boundary 編譯

### Approval 系統重構

```
Approvals: replay pending requests on startup
refactor: centralize native approval lifecycle assembly (#62135)
Extensions: align approval plugin typing
Approvals: align native runtime tests
fix(exec): harden stale/replay/live requests
```

- Startup 時重播待處理的核准請求
- 集中化 native approval 生命週期組裝
- 加強 stale/replay/live 請求的處理

---

## 8. Exec Approvals 與安全閘門

```
Protect gateway exec approval config paths (#62001)
fix(exec): harden stale/replay/live requests
fix: align exec default reporting with runtime
fix: guide exec timeouts to registered background sessions
Approvals: replay pending requests on startup
Require re-pairing for node reconnect command upgrades (#62658)
```

- **配置路徑保護**：Gateway exec approval 配置路徑現受保護
- **Request 類型加強**：區分 stale、replay、live 請求並分別處理
- **Exec 超時引導**：exec 超時正確引導到已註冊的背景 session
- **重連升級需重配對**：node 重連時的 command 升級需要重新配對核准

---

## 9. Gateway 改進

### 穩定性

```
fix(gateway): invalidate shared-token/password WS sessions on secret rotation [AI] (#62350)
fix(gateway): stop SSRF guard rejecting operator-configured proxy hostnames (#62312)
fix(gateway): show /tts audio in Control UI webchat (#61598) (thanks @neeravmakwana)
fix(daemon): skip machine-scope fallback on permission-denied bus errors (#62337)
fix: pass threadId through sessions_send announce delivery (#62758)
feat(gateway): add compaction checkpoints (#62146)
```

- Secret rotation 後立即失效相關 sessions
- SSRF guard 不再誤拒操作者代理主機名
- Control UI webchat 顯示 /tts 音訊
- Daemon 在 permission-denied bus 錯誤時跳過 machine-scope fallback
- sessions_send announce 正確傳遞 threadId
- Compaction 檢查點支援

### Auto-reply 改善

```
Auto-reply: preserve compacted transcript subpaths
perf(auto-reply): trim duplicate heavy coverage
perf(auto-reply): extract followup delivery seam
fix: don't broadcast state:error on per-attempt lifecycle errors (#60043) (thanks @jwchmodx)
fix: honor lightContext in spawned subagents (#62264) (thanks @theSamPadilla)
```

- 壓縮後保留 transcript 子路徑
- 提取 followup delivery seam
- 不再廣播 per-attempt 生命週期錯誤
- 衍生 subagent 遵從 lightContext 設定

### 其他

```
fix: harden tool-result overflow recovery (#61651)
fix: compact update_plan tool result
fix: normalize cached prompt token accounting
fix(agents): classify Anthropic extra-usage billing (#61608) (thanks @neeravmakwana)
fix(agents): skip redundant partial compaction summarization (#61603) (thanks @neeravmakwana)
fix(agents): narrow phase-aware history hardening (#61829) (thanks @100yenadmin)
fix: stop heartbeat transcript truncation races (#60998) (thanks @nxmxbbd)
```

- 工具結果溢出恢復加強
- 壓縮 update_plan 工具結果
- 正規化 cached prompt token 計算
- Anthropic extra-usage 計費分類
- 跳過冗餘的 partial compaction 摘要
- 加強 phase-aware 歷史記錄
- 修復 heartbeat transcript 截斷競態

---

## 10. 平台應用

### iOS

```
feat(ios): improve gateway connection error ux (#62650)
fix(ios): harden watch exec approval review (#61757)
```

- 改善 Gateway 連線錯誤的使用者體驗
- 加強 Apple Watch exec approval 審核流程

### macOS

```
fix(macos): strip commit hash from CLI version output (#61111)
```

- CLI 版本輸出移除 commit hash，顯示更簡潔

### Android

```
fix: add max-height, flex layout, and scrollable command preview for mobile approval card
fix: add bottom safe-area-inset for mobile approval overlay
```

- 行動裝置核准卡片新增最大高度、彈性佈局和可捲動命令預覽
- 行動裝置核准覆蓋層新增底部安全區域 inset

### TUI

```
fix: restore terminal keyboard state on tui exit (#49130) (thanks @biefan)
fix(tui): strip inbound metadata from command messages before rendering (#59985) (thanks @MoerAI)
fix: tighten TUI phase handling and heartbeat session guards (#61463) (thanks @100yenadmin)
fix: preserve legacy replay phase boundaries (#61529) (thanks @100yenadmin)
fix: seed SSE history state from one snapshot (#61855) (thanks @100yenadmin)
```

- TUI 退出時恢復終端鍵盤狀態
- 命令訊息渲染前移除 inbound metadata
- 收緊 TUI phase 處理和 heartbeat session guards
- 保留 legacy replay phase 邊界
- SSE 歷史狀態從單一快照播種

---

## 11. CLI 與 TUI

### Infer CLI

```
feat: Add first-class infer CLI for inference workflows (#62129)
docs: rename and improve infer docs
docs infer cli examples and alias note
```

新增 `openclaw infer` 指令作為一級推理工作流介面，詳見 [第 2 節](#2-新功能)。

### 日誌修復

```
fix(logging): correct levelToMinLevel mapping and related filter logic for tslog v4 (#44646)
```

修正 tslog v4 的日誌等級映射和過濾邏輯。

### 其他 CLI 改善

```
fix: surface Claude CLI API errors
fix: harden tahoe version check
fix: harden parallels upgrade flows
fix: harden parallels upgrade launchers
fix: harden parallels upgrade checks
```

- 上浮 Claude CLI API 錯誤
- 加強 Tahoe 版本檢查和 Parallels 升級流程

---

## 12. 大規模程式碼清理

本次版本包含 **579 個**以上的 dedupe / trim / speed up / split 提交，是一次大規模的程式碼清理運動。

### Trimmed String Readers 去重

```
refactor: dedupe trimmed string readers
refactor: dedupe core trimmed string readers
refactor: dedupe gateway trimmed readers (×3)
refactor: dedupe browser trimmed readers (×2)
refactor: dedupe agent trimmed readers (×2)
refactor: dedupe command trimmed readers
refactor: dedupe channel trimmed readers
refactor: dedupe telegram trimmed readers
refactor: dedupe discord trimmed readers
refactor: dedupe matrix trimmed readers
refactor: dedupe plugin trimmed readers
... (共 30+ 個 trimmed reader 去重)
```

在整個 codebase 中系統性地消除重複的 trimmed string reader 實作，將散布在各模組中的相似實作集中化。

### Lowercase Helpers 去重

```
refactor: dedupe provider lowercase helpers
refactor: dedupe extension lowercase helpers
refactor: dedupe plugin lowercase helpers
refactor: dedupe canvas lowercase helpers
refactor: dedupe normalization lowercase helpers
refactor: dedupe browser and memory lowercase helpers
refactor: dedupe telegram matrix lowercase helpers
refactor: dedupe security lowercase helpers
refactor: dedupe sandbox lowercase helpers
... (共 30+ 個 lowercase helper 去重)
```

系統性消除散布在各模組中的重複 lowercase 輔助函式，統一字串正規化邏輯。

### 測試效能優化

```
test: speed up agent auth config tests
test: speed up provider usage auth tests
test: speed up browser plugin entry tests
test: speed up model config provider tests
test: speed up plugin cli tests
test: speed up channel setup entry tests
test: speed up line and nostr channel tests
test: speed up irc channel seam tests
... (共 30+ 個 speed up 提交)
```

大量測試套件進行效能優化，包含：
- 將重型測試分割為更小的獨立測試檔案
- 消除不必要的模組重載
- 重用 workspace 設定
- 精簡 mock 設定
- 平行化獨立測試路徑

### Secrets / Plugin 效能

```
perf(secrets): narrow dry-run auth store preflight
perf(secrets): cache channel security artifact lookups
perf(secrets): split explicit bundled web provider artifacts
perf(secrets): narrow legacy web search compat providers
perf(secrets): fast-path bundled channel contract loads
perf(secrets): skip idle plugin origin discovery
perf(secrets): lazy-load apply test runtime
perf(secrets): scope compat migration scans
perf(secrets): skip no-op write runtime preflight
```

大幅優化 secrets 子系統效能，包含快取、懶加載、作用域限縮和跳過無效操作。

---

## 13. 測試與 CI

### 測試分割與隔離

```
perf(test): split reply command coverage
perf(test): split subagent command coverage
perf(test): split allowlist and models command coverage
perf(test): split support boundary shard
perf(test): parallelize extension boundary compile
perf(test): trim infra provider and approval suites
```

大量測試覆蓋率分割為獨立的測試路徑，改善平行執行效率。

### 測試穩定性

```
test: stabilize provider auth alias test imports
test: stabilize contract loader seams
test: stabilize scoped runners and qa ports
Tests: stabilize memory dreaming time windows
Tests: stabilize dream diary case assertion (#62275)
Tests: stabilize auth profile and subagent resume specs
Tests: fix nostr package boundary drift
```

多項測試穩定性修復，特別是 dreaming 時區漂移、boundary drift 和 auth profile 測試。

### CI

```
ci: prepare extension lint artifacts
fix: resolve ci type regressions
fix: resolve rebase regressions for ci landing
fix(ci): repair channel type drift
```

CI 管線改善，包含擴展 lint artifacts 準備和類型回歸修復。

---

## 14. 文件與發布

### 文件更新

```
Docs: refresh schema, slash commands, and TTS refs
Docs: clarify Slack streaming thread behavior
Docs: clarify Matrix autoJoin invite scope
Docs: document approval adapter subpaths
docs: add memory wiki docs
docs: rename and improve infer docs
docs: make README model guidance provider-agnostic
docs: document shared mention policy
docs: update config baseline
docs: update plugin sdk api baseline
chore: sync 2026.4.8 config docs baseline
```

- 刷新 schema、slash commands、TTS 參考文件
- 新增 Memory Wiki 文件
- Infer CLI 文件
- README 模型指引改為提供者無關
- Config baseline 和 Plugin SDK API baseline 更新

### 版本發布

```
chore: prepare 2026.4.8 npm release
chore: prepare 2026.4.7-1 npm release
chore: update appcast for 2026.4.7
docs: stamp 2026.4.7 changelog
build(plugins): align package versions to 2026.4.6
build(plugins): sync bundled versions to 2026.4.6
```

- 版本 2026.4.6 → 2026.4.7 → 2026.4.7-1 → 2026.4.8
- Appcast 更新、changelog 標記、bundled plugin 版本同步

---

## 15. 摘要統計

### 提交分類

| 類別 | 數量 | 百分比 |
|------|------|--------|
| refactor | 499 | 29.3% |
| fix | 452 | 26.5% |
| test | 360 | 21.1% |
| perf | 145 | 8.5% |
| chore | 56 | 3.3% |
| feat | 34 | 2.0% |
| docs | 34 | 2.0% |
| style | 18 | 1.1% |
| build | 15 | 0.9% |
| ci | 4 | 0.2% |
| 其他 | 88 | 5.2% |

### 檔案變更

- **變更檔案數**：4,982
- **新增行數**：209,588
- **刪除行數**：105,400
- **淨增行數**：104,188

### 社群貢獻

- **社群致謝提交**：36 個（thanks @...）
- **主要社群貢獻者**：@neeravmakwana（6）、@100yenadmin（4）、@openperf（3）

### 主要貢獻者

| 貢獻者 | 提交數 |
|--------|--------|
| Peter Steinberger | 1,160 |
| Vincent Koc | 329 |
| Gustavo Madeira Santana | 21 |
| Tak Hoffman | 19 |
| Ayaan Zaidi | 16 |
| Eva | 14 |
| Neerav Makwana | 10 |
| Vignesh Natarajan | 10 |
| Agustin Rivera | 9 |
| jjjojoj | 8 |
| pgondhi987 | 6 |

### 關鍵技術趨勢

1. **大規模程式碼清理**：579+ 個 dedupe/trim/split 提交，系統性消除 trimmed reader 和 lowercase helper 重複
2. **Memory Wiki**：在 Dreaming 基礎上建立知識管理層（belief-layer digests、claim health、compiled retrieval）
3. **Proxy / SSRF 安全加固**：全面修復 Slack proxy 支援、SSRF guard 誤判、DNS pinning、NO_PROXY 語義
4. **QA Lab 大幅進化**：全新 UI（Sidebar + Slack 風格）、一鍵 Docker、frontier bakeoff、Markdown 場景
5. **Pluggable Compaction**：可插拔壓縮 provider registry + compaction checkpoints
6. **Infer CLI**：一級推理工作流指令
7. **Plugin Boundary 效能**：平行化、快取、增量 dts 大幅加速 boundary 編譯
8. **Arcee AI + Gemma 4**：新增 Arcee AI 提供者、Google Gemma 4 模型支援
9. **Ollama Vision 自動偵測**：從 /api/show 自動設定 image input 能力
10. **Agent 控制增強**：prompt override、heartbeat controls、compaction 檢查點

### 版本里程碑

| 版本 | 日期 | 性質 |
|------|------|------|
| 2026.4.5 | 2026-04-05 | Stable（起始點） |
| 2026.4.6 | — | Internal |
| 2026.4.7 | — | Stable |
| 2026.4.7-1 | — | Hotfix |
| 2026.4.8 | 2026-04-08 | Stable |
