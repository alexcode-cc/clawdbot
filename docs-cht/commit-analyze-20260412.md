# OpenClaw 提交分析報告 — 2026-04-12

> **分析範圍**：從 `7fbff4260f`（doc: 更新繁體中文文件版本號至 2026.4.8）至 `5040a25fed`（Merge branch '2026-04-12' into develop），共 **1,928 個非合併提交**，涉及 **4,697 個檔案**、新增 262,909 行、刪除 79,830 行。

---

## 目錄

1. [重大變更與安全修復](#1-重大變更與安全修復)
2. [新功能 (New Features)](#2-新功能)
3. [Active Memory 與 Dreaming 改善](#3-active-memory-與-dreaming-改善)
4. [Plugin 架構：Activation Planning](#4-plugin-架構activation-planning)
5. [Provider 與模型系統](#5-provider-與模型系統)
6. [頻道改進 (Channels)](#6-頻道改進)
7. [QA Lab 與 Parity Testing](#7-qa-lab-與-parity-testing)
8. [Exec Approvals 與安全閘門](#8-exec-approvals-與安全閘門)
9. [Gateway 改進](#9-gateway-改進)
10. [平台應用 (Apps)](#10-平台應用)
11. [CLI 與 TUI](#11-cli-與-tui)
12. [測試與 CI](#12-測試與-ci)
13. [文件與發布](#13-文件與發布)
14. [摘要統計](#14-摘要統計)

---

## 1. 重大變更與安全修復

### Exec 安全加固

```
fix(security): remove busybox/toybox from interpreter-like safe bins [AI-assisted] (#65713)
fix(security): broaden shell-wrapper detection and block env-argv assignment injection [AI-assisted] (#65717)
fix(approval-auth): prevent empty approver list from granting explicit approval authorization [AI] (#65714)
fix: Provider-supplied OAuth URLs inject Windows cmd.exe via openUrl (#64161)
fix: Implicit latest-device approval can pair the wrong requester (#64160)
```

本次版本的最重要安全修復：

- **移除 busybox/toybox**：從 interpreter-like safe bins 清單中移除 busybox 和 toybox，防止透過這些多功能工具繞過 exec 限制
- **Shell Wrapper 偵測擴大**：擴大 shell-wrapper 偵測範圍，封鎖 env-argv assignment injection 攻擊向量
- **空 Approver 清單漏洞**：修復空 approver 清單意外授予明確核准授權的漏洞
- **OAuth URL 注入**：修復 Provider-supplied OAuth URLs 可在 Windows 上注入 cmd.exe 的漏洞
- **設備配對漏洞**：修復 implicit latest-device approval 可能配對錯誤請求者的問題

### SSRF / 網路安全

```
fix(ssrf): validate hostname even when pinDns is disabled
fix(browser): keep legacy ssrf alias raw-config only
Feat/fix qq ssrf url list (#65788)
fix(sandbox): enforce CDP source-range restriction by default (#61404)
```

- **SSRF hostname 驗證**：即使 pinDns 停用時仍驗證 hostname
- **CDP 來源限制**：sandbox 預設強制 CDP source-range 限制
- **QQ SSRF 修復**：修復 QQ 頻道 SSRF URL 清單問題

### 環境安全

```
fix: scope pinDns override to multipart audio (#64766) (thanks @GodsBoy)
fix: preserve browser cdp ssrf policy
```

- pinDns override 限制範圍至 multipart audio
- 保留 browser CDP SSRF 策略

---

## 2. 新功能

### LM Studio 整合

```
feat: LM Studio Integration (#53248)
[codex] Fix LM Studio header-auth follow-ups (#65806)
```

新增 **LM Studio** 本地模型提供者整合。LM Studio 是熱門的本地 LLM 運行平台，現可直接作為 OpenClaw 的模型提供者使用。支援 header-auth 認證方式。

### Pluggable Agent Harness

```
feat: add pluggable agent harness registry
feat: add Codex app-server harness extension
feat: add Codex app-server controls
feat(agents): allow disabling PI harness fallback
```

新增**可插拔 Agent Harness 註冊表**，允許不同的 agent runtime 後端（如 Codex app-server）透過 registry 介面整合。這是支援多元 agent 執行環境的基礎架構。

### Local Exec-Policy CLI

```
feat: add local exec-policy CLI (#64050)
```

新增 `openclaw exec-policy` 本地指令，允許在本地管理和檢視 exec 執行策略。

### Plugin Text Transforms

```
feat: add plugin text transforms
```

插件現可註冊文字轉換能力，在訊息處理管線中對文字進行自訂轉換。

### Feishu QR Code Onboarding

```
feat: Streamline Feishu channel onboarding with QR code scan-to-create flow (#65680)
feat(feishu): improve document comment session, rich parsing, and typing feedback (#63785)
feat(feishu): standardize request UA and register bot as AI agent (#63835)
```

飛書頻道大幅改善：
- **QR Code 掃碼建立**：掃碼即可完成飛書頻道 onboarding
- **文件評論 Session**：改善文件評論互動、rich parsing、打字回饋
- **UA 標準化**：標準化請求 User-Agent 並註冊 bot 為 AI agent

### MS Teams 改善

```
feat(msteams): add federated credential support (certificate + managed identity) (#53615)
feat(msteams): auto-inject parent message context for thread replies (#54932) (#63945)
feat(msteams): handle signin/tokenExchange and signin/verifyState for SSO (#60956) (#64089)
```

MS Teams 頻道重大更新：
- **聯邦認證**：支援憑證（certificate）和受管理身份（managed identity）
- **Thread 回覆注入**：自動注入父訊息 context 於 thread 回覆中
- **SSO 支援**：處理 signin/tokenExchange 和 signin/verifyState 流程

### Matrix MSC4357 Live Streaming

```
feat(matrix): add MSC4357 live streaming markers to draft-stream edits (#63513)
```

Matrix 新增 MSC4357 即時串流標記，在 draft-stream 編輯中添加 live streaming markers。

### Video Generation 增強

```
feat: add Seedance 2 fal video models
feat(fal): add HeyGen video-agent model
video_generate: support url-only delivery (#61988)
video_generate: add providerOptions, inputAudios, and imageRoles (#61987)
```

影片生成工具大幅增強：
- **Seedance 2**：新增 Seedance 2 fal 影片模型
- **HeyGen Video Agent**：新增 HeyGen video-agent 模型
- **URL-only Delivery**：支援僅 URL 的影片交付方式
- **進階選項**：新增 providerOptions、inputAudios 和 imageRoles 參數

### Active Memory Recall Plugin

```
feat: Add Active Memory recall plugin (#63286)
feat: default active memory QMD recall to search (#65068)
```

新增 **Active Memory Recall** 插件，將記憶搜尋能力提升為一級功能。預設使用 QMD recall-to-search 策略。

### Dreaming UI

```
feat(ui): add dreaming diary controls and navigation (#63298)
chore: prep dreaming UI land (#64035) (thanks @davemorin)
```

Control UI 新增 Dreaming diary 控制介面和導航功能。

### Control UI 改善

```
feat(ui): render assistant directives and add embed tag (#64104)
feat(ui): render btw side results in control ui
Control UI: refresh slash commands from runtime command list (#65620)
fix(ui): hide synthetic transcript-repair history messages (#65458)
```

- 渲染 assistant directives 和 embed tag
- 顯示 btw 側邊結果
- 從 runtime command list 刷新 slash commands
- 隱藏 synthetic transcript-repair 歷史訊息

### /trace 切換與 Active Memory 診斷

```
Add /trace toggle and fix Active Memory diagnostics
```

新增 `/trace` 切換指令，搭配 Active Memory 診斷功能，方便開發者追蹤記憶系統行為。

---

## 3. Active Memory 與 Dreaming 改善

### Active Memory

```
feat: Add Active Memory recall plugin (#63286)
feat: default active memory QMD recall to search (#65068)
improve memory fallback lexical ranking (#65395)
Clarify Active Memory lexical fallback behavior
Clarify Active Memory embedding provider setup
Add /trace toggle and fix Active Memory diagnostics
fix(active-memory): remove built-in fallback model (#65047)
Fix active-memory recall runs when mx-claw is enabled (#65049)
```

Active Memory 系統重大改善：
- **Recall Plugin**：作為獨立插件提供，支援 QMD recall-to-search
- **Lexical Fallback 改善**：改進 fallback 詞彙排序，提升無嵌入環境下的搜尋品質
- **移除內建 Fallback 模型**：不再內建 fallback 模型，改由使用者明確配置
- **mx-claw 相容修復**：修復 mx-claw 啟用時 recall 運行問題
- **診斷工具**：/trace 切換和 Active Memory 診斷

### Dreaming 改善

```
feat(memory): harden grounded REM extraction (#63297)
feat(memory): add grounded REM backfill lane (#63273)
fix(memory-core): isolate dreaming narrative sessions per workspace (#61674)
fix: harden dreaming narrative session cleanup (#65320)
fix(memory-core): wake managed dreaming jobs immediately (#65053)
fix(memory-core): use all dreaming signals for light confidence
fix(memory-core): unblock dreaming-only promotion
fix(memory-core): match daily notes stored in memory/ subdirectories (#64682)
fix(memory-core): fix macOS chokidar glob issue by watching memory dir directly (#64711)
dreaming: isolate Belief-Layer Digest promotion from session-corpus pass
dreaming: allow session-corpus to run without search
```

Dreaming 記憶系統持續改善：
- **Grounded REM**：加強 REM extraction 的接地性，新增 backfill lane
- **Workspace 隔離**：dreaming narrative sessions 按 workspace 隔離
- **即時喚醒**：managed dreaming jobs 可立即喚醒
- **macOS 修復**：修復 macOS chokidar glob 問題
- **Belief-Layer 隔離**：Digest promotion 與 session-corpus pass 解耦
- **Session-corpus 解耦**：允許在無搜尋功能時運行 session-corpus

### Memory Wiki

```
fix(memory-wiki): support Unicode characters in slugifyWikiSegment (#64742)
docs(memory-wiki): add QMD bridge recipe (#63165)
```

- Memory Wiki slug 支援 Unicode 字元
- 新增 QMD bridge recipe 文件

---

## 4. Plugin 架構：Activation Planning

```
feat(plugins): narrow CLI loading via activation planning (#65120)
feat(plugins): narrow explicit provider loads from manifests (#65259)
feat(plugins): narrow channel loads from manifests (#65429)
feat(plugins): add manifest activation and setup descriptors (#64780)
refactor(plugins): centralize manifest owner trust policy (#65459)
feat(plugins): support provider auth aliases
fix(plugins): centralize explicit plugin scope handling (#65298)
fix(plugins): restore cached memory capability on cache hits (#65240)
fix(plugins): preserve empty provider scopes
fix(plugins): exempt dreaming engine from memory slot fast-path in loader (#65411)
fix(plugins): tolerate bundled peer resolution
fix(plugins): restore missing native runtime deps
```

本次版本引入 **Activation Planning**（啟動規劃）架構，是 Plugin 系統的重大演進：

- **Manifest-based Activation**：透過 manifest 描述符驅動啟動規劃，不再盲目載入所有插件
- **Narrow Loading**：CLI、Provider、Channel 三個維度都改為按需載入（narrow loading）
  - CLI loading：僅載入當前指令所需的插件
  - Provider loading：僅載入已配置的 provider 插件
  - Channel loading：僅載入已啟用的 channel 插件
- **Setup Descriptors**：manifest 新增 activation 和 setup 描述符，描述插件何時需要被載入
- **Owner Trust Policy**：集中化 manifest owner trust 策略
- **Provider Auth Aliases**：支援 provider auth 別名
- **Memory Capability 快取**：修復 cache hit 時遺失 memory capability 的問題

這大幅改善了啟動效能和記憶體使用量，特別是在安裝了大量插件的環境中。

---

## 5. Provider 與模型系統

### LM Studio

```
feat: LM Studio Integration (#53248)
[codex] Fix LM Studio header-auth follow-ups (#65806)
```

新增 LM Studio 本地模型提供者，支援 header-auth 認證。

### Codex / OpenAI

```
fix: preserve Codex OAuth scopes (#64713) (thanks @fuller-stack-dev)
fix: stabilize Codex runtime truthfulness (#64439) (thanks @100yenadmin)
fix: tighten codex app-server lifecycle
openai-codex: normalize streaming choices
OpenAI: strengthen heartbeat overlay guidance (#65148)
fix: return real usage for OpenAI-compatible chat completions (#62986) (thanks @Lellansin)
```

- 保留 Codex OAuth scopes
- 穩定 Codex runtime
- OpenAI heartbeat overlay guidance 加強
- 修正 OpenAI 相容聊天完成的真實 usage 回傳

### Gemini / Google

```
fix: sanitize Gemini tool schema required fields (#64284) (thanks @xxxxxmax)
fix: detect llama.cpp context overflow (#64196) (thanks @alexander-applyinnovations)
```

- 清理 Gemini tool schema required fields
- 偵測 llama.cpp context overflow

### OpenRouter

```
fix: continue fallback after OpenRouter no-endpoints 404 (#61472) (thanks @MonkeyLeeT)
```

OpenRouter 404 後繼續 fallback chain。

### Media Provider

```
feat: declare explicit media provider capabilities
feat: preserve media intent across provider fallback
fix(media): decouple capability registry from runtime loaders
fix(media): surface OpenAI audio transcription failures (#65096)
fix(media): use exported decision outcome type
```

- 明確宣告 media provider capabilities
- Media intent 跨 provider fallback 保留
- 能力 registry 與 runtime loaders 解耦
- OpenAI audio transcription 失敗上浮

---

## 6. 頻道改進

### Feishu（飛書）

```
feat: Streamline Feishu channel onboarding with QR code scan-to-create flow (#65680)
feat(feishu): improve document comment session, rich parsing, and typing feedback (#63785)
feat(feishu): standardize request UA and register bot as AI agent (#63835)
fix(feishu): guard app registration fetches
fix(feishu): break auth login barrel cycle
fix(feishu): keep channel auth on local api barrel
fix(feishu): avoid sdk facade cycles
```

飛書頻道獲得全面重構：QR Code onboarding、文件評論改善、UA 標準化、auth cycle 修復。

### MS Teams

```
feat(msteams): add federated credential support (#53615)
feat(msteams): auto-inject parent message context for thread replies (#54932)
feat(msteams): handle signin/tokenExchange and signin/verifyState for SSO (#60956)
fix(msteams): keep streaming alive during long tool chains via typing indicator (#59731)
fix(msteams): restore graph media diagnostics
```

MS Teams 重大更新：聯邦認證、thread context 注入、SSO 支援、長工具鏈串流保活。

### Matrix

```
feat(matrix): add MSC4357 live streaming markers to draft-stream edits (#63513)
fix(matrix): mirror runtime deps for docker builds
fix(matrix): sync runtime dependency lockfile
```

Matrix 新增 MSC4357 live streaming markers。

### Discord

```
fix(discord): clear stale heartbeat timers in SafeGatewayPlugin.connect() (#65087)
fix(discord): normalize legacy streaming aliases
fix(discord): declare gateway heartbeat timeout state
fix: route Discord auto TTS as voice notes (#64096) (thanks @LiuHuaize)
fix: compact discord allowlist resolution logs
fix(doctor): preserve discord streaming downgrade compatibility
```

Discord 修復：heartbeat timer 清理、streaming alias 正規化、auto TTS 路由為 voice notes。

### Telegram

```
fix: unblock Telegram approval callback deadlock (#64979) (thanks @nk3750)
```

修復 Telegram approval callback 死鎖問題。

### Slack

```
fix(slack): preserve auth on same-origin media redirects (#62996) (thanks @vincentkoc)
Slack: dedupe partial streaming replies (#62859)
```

- 同源 media 重定向保留 auth
- 去重部分串流回覆

### QQ

```
Feat/fix qq ssrf url list (#65788)
```

修復 QQ 頻道 SSRF URL 清單問題。

### iMessage

```
fix(imessage): retry watch.subscribe startup failures (#65482)
```

iMessage watch.subscribe 啟動失敗重試。

### WhatsApp

```
Fix WhatsApp media sends when mediaUrl is empty but mediaUrls is populated (#64394)
```

修復 WhatsApp 媒體發送在 mediaUrl 為空但 mediaUrls 有值時的問題。

---

## 7. QA Lab 與 Parity Testing

### QA Lab 擴展

```
feat(qa-lab): add Convex credential broker and admin CLI (#65596)
feat(qa-lab): Add proxy capture stack and QA Lab inspector (#64895)
feat(qa-lab): add control ui qa-channel roundtrip scenario
feat(qa-lab): support scenario-defined plugin runs
feat(qa-lab): add telegram mentioned-message scenario
feat(qa-lab): add telegram command demo scenarios
feat(qa-lab): add telegram mention-gating scenario
feat(qa-lab): add telegram live qa lane
feat(qa-channel): forward inbound media attachments
feat(browser): add qa web runtime support
feat: add multipass runner to qa suite
feat: add QA character eval reports
feat: add qa character vibes eval
feat: add character eval model options
feat: parallelize character eval runs
```

QA Lab 持續大幅擴展：
- **Convex Credential Broker**：Convex 認證代理和管理 CLI
- **Proxy Capture Stack**：代理擷取堆疊和 QA Lab inspector
- **多 Telegram 場景**：mentioned-message、command demo、mention-gating、live lane
- **Character Eval**：全新角色評估系統（vibes eval、model options、並行化執行、報告）
- **Multipass Runner**：multi-pass 場景執行器
- **Plugin Runs**：場景定義的 plugin 執行支援

### Parity Testing

```
qa-lab: gate parity on shared scenario coverage
qa-lab: scope parity metrics and harden fake-success detector
Treat skipped parity scenarios as uncovered
Tighten parity proof heuristics
fix: harden parity gate review findings
qa: salvage GPT-5.4 parity proof slice (#65664)
```

Parity testing 框架成熟化：
- **Shared Scenario Coverage**：parity 基於共享場景覆蓋率
- **Fake-success Detector**：偵測假成功場景
- **GPT-5.4 Parity Proof**：GPT-5.4 parity 驗證切片

---

## 8. Exec Approvals 與安全閘門

```
fix(approval-auth): prevent empty approver list from granting explicit approval authorization [AI] (#65714)
fix: Implicit latest-device approval can pair the wrong requester (#64160)
fix: unblock Telegram approval callback deadlock (#64979) (thanks @nk3750)
refactor: simplify exec approval booleans
cb19451132 refactor: drop legacy Codex approval support
```

- 空 approver 清單漏洞修復
- 設備配對 approval 修復
- Telegram approval 死鎖修復
- 精簡 exec approval 布林邏輯
- 移除 legacy Codex approval 支援

---

## 9. Gateway 改進

```
fix(gateway): defer cron AND heartbeat activation until sidecars are ready (#65322)
fix: defer gateway scheduled services (#65365) (thanks @lml2468)
fix: lazy-start gateway mcp loopback
fix: dedupe delivered subagent completion announces (#61525) (thanks @100yenadmin)
fix: don't broadcast state:error on per-attempt lifecycle errors
gateway: always send idempotencyKey on plugin subagent run (#65354)
fix: correct cron AND guidance (#64968) (thanks @BKF-Gitty)
fix: warn about orphaned agent dirs (#65113) (thanks @neeravmakwana)
```

- **Deferred Activation**：Gateway 延遲 cron 和 heartbeat 啟動，等待 sidecars 就緒
- **Scheduled Services 延遲**：gateway scheduled services 延遲到就緒狀態
- **MCP Loopback 懶啟動**：gateway MCP loopback 改為懶啟動
- **Subagent Completion 去重**：去重重複的 subagent completion 通知
- **IdempotencyKey**：plugin subagent run 總是發送 idempotencyKey
- **孤立 Agent 目錄警告**：偵測並警告孤立的 agent 目錄

---

## 10. 平台應用

### iOS

```
feat(ios): pin calver release versioning (#63001)
fix(talk): fix ensure permissions on first execution of Talk Mode in MacOS (#62459)
```

- iOS 釘定 CalVer 版本號策略
- 修復 Talk Mode 首次執行時的權限確保問題

### macOS

```
fix: improve macos parallels npm smoke installs
fix(imessage): retry watch.subscribe startup failures (#65482)
fix(memory-core): fix macOS chokidar glob issue by watching memory dir directly (#64711)
```

- 改善 macOS Parallels npm 安裝
- macOS chokidar glob 問題修復

### Android / Mobile

```
video_generate: add providerOptions, inputAudios, and imageRoles (#61987)
```

影片生成 API 新增進階選項。

### TUI

```
fix: reset TUI footer activity on session switch (#63988) (thanks @neeravmakwana)
fix: strip wrapped imsg rpc text fields (#64000) (thanks @neeravmakwana)
fix: require destination_caller_id for self-chat classification (#63989) (thanks @neeravmakwana)
```

- Session 切換時重置 TUI footer
- 清理 iMessage RPC 文字欄位包裝
- Self-chat 分類需要 destination_caller_id

---

## 11. CLI 與 TUI

```
feat: add local exec-policy CLI (#64050)
Add /trace toggle and fix Active Memory diagnostics
fix(tts): correct tagged TTS syntax guidance (#65573)
Control UI: refresh slash commands from runtime command list (#65620)
fix: surface Claude CLI API errors
```

- **exec-policy CLI**：本地 exec 執行策略管理
- **/trace**：Active Memory 追蹤切換
- **TTS 語法修正**：修正 tagged TTS 語法指引
- **Slash Commands 刷新**：Control UI 從 runtime 刷新 slash commands

---

## 12. 測試與 CI

### 測試共享化

本次版本包含 **538 個 test 提交**，大量測試進行共享化重構（share fixture/harness/helper）：

```
test(gateway): share transcript event waiters
test(gateway): share preauth hardening setup helpers
test(gateway): share search session transcript fixtures
test(agents): share sandbox spawn fixture helpers
test(agents): share approval abort helper
test(codex): share app-server client harness
test(msteams): share reaction handler harness
test(matrix): share client startup and backfill fixtures
test(discord): share guild interaction auth fixtures
... (100+ share/dedupe 提交)
```

系統性地將測試 fixture、harness、helper 共享化，大幅減少測試程式碼重複。

### CI 改善

```
CI: use 32vCPU Blacksmith release runners
Release: separate release checks workflow (#65552)
fix(ci): verify bundled plugin runtime deps
feat(test): use host-aware local full-suite defaults (#65264)
```

- 使用 32vCPU Blacksmith release runners
- 分離 release checks workflow
- 驗證 bundled plugin runtime deps
- Host-aware 測試預設值

---

## 13. 文件與發布

### 文件

```
docs(providers): fill undocumented capability gaps
docs(providers): improve 40+ provider docs with Mintlify components
docs(memory-wiki): add QMD bridge recipe (#63165)
docs: clarify strict-agentic and codex modes
docs: note GPT-5.4 parity harness landing
docs(agents): split scoped workflow guidance (#65241)
docs:add maintainer info (#65762)
docs: update 2026.4.12 changelog
```

- **Provider 文件大更新**：40+ 個 provider 文件使用 Mintlify 元件改善（TTS、media understanding、embeddings、xSearch、env vars）
- **Memory Wiki QMD 文件**：QMD bridge recipe
- **Agent 指南**：拆分 scoped workflow 指引
- **Maintainer 資訊**：新增 maintainer 資訊

### 版本發布

```
chore(release): prepare 2026.4.12
chore(release): prepare 2026.4.12-beta.1
chore(release): restore 2026.4.11 appcast
chore: prepare 2026.4.9 release
build(plugins): sync bundled versions
```

- 版本 2026.4.9 → 2026.4.10 → 2026.4.11 → 2026.4.12-beta.1 → 2026.4.12
- Appcast 和 bundled plugin 版本同步

---

## 14. 摘要統計

### 提交分類

| 類別 | 數量 | 百分比 |
|------|------|--------|
| fix | 735 | 38.1% |
| test | 538 | 27.9% |
| refactor | 146 | 7.6% |
| docs | 79 | 4.1% |
| chore | 74 | 3.8% |
| perf | 50 | 2.6% |
| feat | 45 | 2.3% |
| UI | 44 | 2.3% |
| ci | 21 | 1.1% |
| style | 14 | 0.7% |
| build | 14 | 0.7% |
| dreaming | 11 | 0.6% |
| 其他 | 157 | 8.1% |

### 檔案變更

- **變更檔案數**：4,697
- **新增行數**：262,909
- **刪除行數**：79,830
- **淨增行數**：183,079

### 社群貢獻

- **社群致謝提交**：30+ 個（thanks @...）
- **主要社群貢獻者**：@100yenadmin（4）、@neeravmakwana（4）、@ngutman（3）、@davemorin（UI）

### 主要貢獻者

| 貢獻者 | 提交數 |
|--------|--------|
| Peter Steinberger | 843 |
| Vincent Koc | 431 |
| Tak Hoffman | 100 |
| joshavant | 53 |
| Ayaan Zaidi | 45 |
| Shakker | 45 |
| Eva | 38 |
| Nimrod Gutman | 29 |
| Gustavo Madeira Santana | 21 |
| sudie-codes | 19 |
| Mariano | 18 |

### 關鍵技術趨勢

1. **Plugin Activation Planning**：manifest-driven 啟動規劃，CLI/Provider/Channel 三維 narrow loading
2. **LM Studio 整合**：新增熱門本地 LLM 平台提供者
3. **Active Memory Recall**：獨立 recall plugin + QMD recall-to-search + /trace 診斷
4. **Exec 安全加固**：busybox/toybox 移除、shell-wrapper 偵測擴大、approval 漏洞修復
5. **Feishu 全面重構**：QR Code onboarding、文件評論、UA 標準化
6. **MS Teams SSO + 聯邦認證**：企業級認證支援
7. **QA Parity Testing**：GPT-5.4 parity proof、character eval、multipass runner
8. **Pluggable Agent Harness**：可插拔 agent runtime 後端
9. **Video Generation 增強**：Seedance 2、HeyGen、providerOptions 支援
10. **測試共享化運動**：538+ test 提交，系統性 fixture/harness 共享

### 版本里程碑

| 版本 | 日期 | 性質 |
|------|------|------|
| 2026.4.8 | 2026-04-08 | Stable（起始點） |
| 2026.4.9 | — | Stable |
| 2026.4.10 | — | Internal |
| 2026.4.11 | — | Stable |
| 2026.4.12-beta.1 | — | Beta |
| 2026.4.12 | 2026-04-12 | Stable |
