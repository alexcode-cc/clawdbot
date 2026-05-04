# OpenClaw 提交分析報告 — 2026-04-14

> **分析範圍**：從 `dd88f6462911efafdafc6f2875604b70a509716e`（doc: 更新繁體中文文件版本號至 2026.4.12）至最新提交（`323493fa1b`，chore: prepare 2026.4.14 release），共 **267 個非合併提交**，涵蓋 2026-04-12 至 2026-04-14。

---

## 目錄

1. [版本發布](#1-版本發布)
2. [安全修復](#2-安全修復)
3. [效能優化大波次](#3-效能優化大波次)
4. [Telegram 大幅改善](#4-telegram-大幅改善)
5. [Browser / CDP / SSRF 修復](#5-browser--cdp--ssrf-修復)
6. [OpenAI / Codex 改善](#6-openai--codex-改善)
7. [頻道重播與重試標準化](#7-頻道重播與重試標準化)
8. [Context Engine 改善](#8-context-engine-改善)
9. [記憶系統修復](#9-記憶系統修復)
10. [Gateway 與配置系統](#10-gateway-與配置系統)
11. [其他頻道修復](#11-其他頻道修復)
12. [UI / Control UI](#12-ui--control-ui)
13. [QA Lab 改善](#13-qa-lab-改善)
14. [文件更新](#14-文件更新)
15. [測試與 CI](#15-測試與-ci)
16. [摘要統計](#16-摘要統計)

---

## 1. 版本發布

```
chore(release): prepare 2026.4.14 beta
chore: prepare 2026.4.14 release
build: refresh a2ui bundle hash
test: refresh release verification baselines
```

本次週期的最終版本為 **2026.4.14**（穩定版），從 beta 版經測試後正式發布。主要亮點包括：效能優化大波次、Telegram topic 名稱改善、OpenAI gpt-5.4-pro 前向相容、Browser SSRF 修復、以及大規模頻道重播/重試標準化。

---

## 2. 安全修復

### Browser SSRF 策略修復

```
fix(browser): relax default hostname SSRF guard
fix(browser): use loopback policy for json-new fallback
fix(browser): preserve explicit strict SSRF config
fix(browser): preserve legacy strict SSRF alias
fix(browser): enforce SSRF policy on snapshot, screenshot, and tab routes [AI] (#66040)
fix(browser): unblock loopback CDP readiness under strict SSRF defaults (#66354)
fix: add browser SSRF follow-up changelog entry (#66386)
fix(browser): detect local attachOnly loopback CDP sessions (#66080)
fix(browser): unblock managed loopback CDP startup and control (#66043)
```

瀏覽器 SSRF 策略經過重大調整：
- **預設策略放寬**：恢復預設 hostname 導航能力，僅保留嚴格模式為可選配置
- **嚴格模式保留**：使用 `browser.ssrfPolicy.allowPrivateNetwork: false` 的使用者設定會被正確標準化為嚴格標記
- **Loopback 政策**：`/json/new` fallback 請求使用本地 CDP 控制政策
- **快照/截圖/分頁路由**：對這些路由強制執行 SSRF 政策
- **Loopback CDP 可達性**：修復嚴格 SSRF 預設下 loopback CDP 就緒檢查無法連接的問題

### Gateway Config 安全防護

```
Guard dangerous gateway config mutations (#62006)
```

新增重要安全防護：拒絕模型端 gateway 工具呼叫中會啟用 `openclaw security audit` 列舉危險旗標的 `config.patch`/`config.apply` 操作，例如：
- `dangerouslyDisableDeviceAuth`
- `allowInsecureAuth`
- `dangerouslyAllowHostHeaderOriginFallback`
- `hooks.gmail.allowUnsafeExternalContent`
- `tools.exec.applyPatch.workspaceOnly: false`

已啟用的旗標允許通過（以避免阻斷非危險編輯），直接認證的操作員 RPC 行為不受影響。

### Heartbeat 安全

```
fix(heartbeat): force owner downgrade for untrusted hook:wake system events [AI-assisted] (#66031)
```

強制降級不受信任 `hook:wake` 系統事件的 owner 等級。

### MS Teams SSO 安全

```
fix(msteams): enforce sender allowlist checks on SSO signin invokes [AI] (#66033)
```

對 SSO signin 呼叫強制執行發送者 allowlist 檢查。

### Config 安全

```
fix(config): redact sourceConfig and runtimeConfig alias fields in redactConfigSnapshot [AI] (#66030)
```

在 `redactConfigSnapshot` 中遮蔽 `sourceConfig` 和 `runtimeConfig` 別名欄位。

### Doctor/Systemd 修復

```
fix: avoid inline dotenv secrets in systemd unit during service repair (#66249) (thanks @tmimmanuel)
```

修復 `openclaw doctor --repair` 和服務重裝不再將 dotenv-backed secrets 重新嵌入使用者 systemd units，同時保留較新的 inline overrides 優先於舊的 state-dir `.env` 值。

### 媒體附件安全

```
fix(media): fail closed on attachment canonicalization (#66022)
```

當本地附件路徑無法透過 `realpath` 規範化解析時，改為 fail-closed（安全失敗），防止 `realpath` 錯誤將規範根目錄 allowlist 檢查降級為非規範比較。

---

## 3. 效能優化大波次

本次版本包含超過 **50 個效能優化提交**，形成系統性的效能改善計畫，主要策略為「懶載入（lazy-load）」和「窄化匯入（narrow imports）」。

### Agents 效能

```
perf(agents): narrow session helper imports
perf(agents): lazy-load session store updates
perf(agents): lazy-load delivery runtime
perf(agents): keep attempt execution runtime cold
perf(agents): lazy-load cli runner seams
perf(agents): isolate agent scope config helpers
perf(agents): isolate thinking default helper
perf(agents): use lightweight model fallback selection helpers
perf(agents): narrow failover helper imports
perf(agents): keep model fallback auth runtime cold
perf(agents): keep fallback auth store cold without sources
```

Agent 系統大量路徑改為懶載入，包括 session 輔助工具、delivery runtime、CLI runner seams、thinking 預設輔助工具、模型 fallback 選擇輔助工具等。

### Commands 效能

```
perf(commands): lazy-load agent secret resolution
perf(commands): narrow agent config imports
perf(commands): narrow session test imports
```

命令系統縮小冷路徑的 import 範圍。

### Cron 效能

```
perf(cron): keep skill filter runtime lazy
perf(cron): lazy-load delivery runtime helpers
perf(cron): lazy-load external content runtime
perf(cron): lazy-load run executor runtime
perf(cron): use narrow verbose-level runtime seam
perf(cron): use lightweight model selection resolver
perf(cron): drop stale skill snapshot runtime exports
perf(cron): use read-only allow-from store seam
perf(cron): lazy-load delivery logger runtime
perf(cron): lazy-load skills snapshot runtime
perf(cron): lazy-load delivery subagent registry
perf(cron): use session store read path
perf(cron): lazy-load context and catalog lookups
perf(cron): narrow execution and skill runtime imports
perf(cron): lazy-load embedded runtime branch
perf(cron): isolate runtime-heavy seams
perf(cron): narrow session runtime imports
perf(cron): keep auth profile runtime cold
perf(cron): lazy-load isolated cli runner runtime
perf(cron): use narrow bound-account lookup
perf(cron): trim unused runtime barrel exports
```

Cron 系統超過 20 個效能提交，形成全面的冷路徑懶載入改造。

### Config 效能

```
perf(config): keep runtime compat migrations lightweight
perf(config): scope dry-run legacy validation
perf(config): narrow channel legacy rule loading
perf(config): skip cold runtime refresh on one-shot writes
perf(config): use generated SecretRef policy metadata
perf(config): use direct writes for gateway token persistence
perf(config): defer legacy web search registry reads
perf(config): reuse prepared snapshots for daemon token writes
perf(config): skip shell env fallback for explicit empty vars
perf(config): reuse validated best-effort snapshots
```

配置系統同樣進行大幅效能優化，涵蓋 dry-run 驗證、daemon token 寫入、legacy registry 讀取等。

### Channels / Sessions / Outbound / Secrets 效能

```
perf(channels): split hot-path message channel normalization
perf(channels): read bundled channel metadata directly
perf(channels): isolate loaded target parsing
perf(sessions): isolate reset policy helpers
perf(sessions): use loaded thread-info seam
perf(outbound): narrow loaded target channel reads
perf(outbound): use loaded-only channel plugin reads
perf(outbound): use read-only channel registry seam
perf(outbound): isolate id-like target resolution
perf(secrets): lazy-load provider env var exports
perf(secrets): fast-path explicit channel target lookup
perf(auth-profiles): narrow source check path imports
perf(plugins): isolate manifest registry cache state
perf(utils): isolate message channel normalization
perf(daemon): import install config helpers directly
perf(daemon): slim gateway install token imports
perf(daemon): lazy-load auth profile install helpers
perf(daemon): keep install auth env path cold
perf(infra): cache login shell env probes
perf(cli): skip redundant schema passes for structured dry runs
perf(wizard): keep explicit skip auth path cold
```

### 整體效能影響

這波效能優化的核心目標是：
1. **Agent 啟動加速**：Agent 啟動不再等待重量級 import，provider 選擇和 agent 啟動停止停滯在重量級 imports 上
2. **Cron 執行加速**：直接頻道交付和相關 cron 執行減少載入無關的 auth、plugin 和 channel runtime
3. **Setup/Config 加速**：setup、config dry-runs 和 daemon install 不再急切啟動 auth-profile 和 plugin repair runtime
4. **冷路徑隔離**：測試可以 mock 較薄的 seam，而不是意外觸發真實工作

---

## 4. Telegram 大幅改善

### Forum Topic 名稱功能

```
fix: expose telegram topic names in agent context (#65973) (thanks @ptahdunbar)
fix(telegram): persist topic-name cache
fix(telegram): allow topic cache without session runtime
fix(telegram): persist topic cache via default runtime
fix: move telegram topic-cache changelog to unreleased (#66107)
test(telegram): cover topic-name cache reload
docs(changelog): note telegram topic-name persistence
```

新增 **Telegram Forum Topic 名稱支援**：
- **Topic 名稱學習**：從 Telegram forum service messages 學習人類可讀的 topic 名稱
- **Agent Context 整合**：在 agent context、prompt metadata 和 plugin hook metadata 中展示人類可讀的 topic 名稱
- **持久化快取**：將已學習的 topic 名稱持久化到 Telegram session sidecar store，重啟後無需重新學習
- **無 Session Runtime**：topic 快取在沒有 session runtime 的情況下也可運作

### Telegram 重試機制

```
fix(telegram): retry failed approval callbacks
fix(telegram): retry failed model selections
fix(telegram): retry failed model browser callbacks
fix(telegram): retry failed pagination preflight
fix(telegram): retry failed plugin binding callbacks
fix(telegram): retry failed commands pagination callbacks
fix(telegram): retry failed model callbacks
fix(telegram): retry failed reaction updates
fix(telegram): retry failed group migration updates
fix(telegram): block watermark advancement past failed updates
fix(telegram): defer replay commit until update succeeds
fix(telegram): avoid leaking thread binding persist cleanup
fix(telegram): swallow update watermark persistence failures
```

Telegram 頻道全面加強重試機制：
- **回呼重試**：approval callbacks、model selection callbacks、model browser callbacks、plugin binding callbacks、model callbacks 全部支援重試
- **分頁重試**：pagination preflight 和 commands pagination callbacks 支援重試
- **更新重試**：reaction updates 和 group migration updates 支援重試
- **水印管理**：阻止 watermark 在失敗更新後前進；延遲重播 commit 直到更新成功

### Telegram Status 指令

```
[codex] Telegram: unblock status commands behind busy turns (#66226)
```

修復忙碌 turns 期間 status 指令被阻塞的問題。

---

## 5. Browser / CDP / SSRF 修復

（詳見第 2 節「安全修復」中的 Browser SSRF 部分）

額外修復：
```
fix(browser): detect local attachOnly loopback CDP sessions (#66080)
fix(browser): unblock managed loopback CDP startup and control (#66043)
fix(browser): unblock loopback CDP readiness under strict SSRF defaults (#66354)
```

- **Local AttachOnly 偵測**：偵測本地 attachOnly loopback CDP sessions
- **Managed Loopback**：解除 managed loopback CDP 啟動和控制的阻塞
- **嚴格模式下的 Loopback**：確保嚴格 SSRF 預設下本地受管 Chrome 仍可重新連接

---

## 6. OpenAI / Codex 改善

### GPT-5.4-Pro 前向相容

```
feat(codex): add gpt-5.4-pro forward compat (#66453)
test(codex): cover exact gpt-5.4 registry upgrades (#66454)
```

新增 `gpt-5.4-pro` 前向相容支援，包含 Codex 定價/限制和清單/狀態可見性，在上游 catalog 跟上之前可先使用。

### Codex 別名與目錄修復

```
fix(codex): canonicalize the gpt-5.4-codex alias (#66438)
fix: include apiKey in codex provider catalog to unblock models.json loading (#66180)
fix(codex): keep auth read diagnostics off stdout (#66451)
```

- **gpt-5.4-codex 別名**：將 legacy `openai-codex/gpt-5.4-codex` runtime 別名規範化為 `openai-codex/gpt-5.4`
- **apiKey 欄位**：在 codex provider catalog 輸出中加入 `apiKey`，修復 Pi ModelRegistry 驗證器拒絕條目並靜默丟棄所有 `models.json` 中自訂模型的問題
- **Auth 診斷**：保持 Codex CLI auth-file 診斷在 debug logger 而非 stdout

### GitHub Copilot

```
fix(github-copilot): enable xhigh for gpt-5.4 (#66437)
```

允許 `github-copilot/gpt-5.4` 使用 `xhigh` 推理等級，與 GPT-5.4 系列其他成員一致。

### OpenAI Reasoning 修復

```
fix: recover reasoning-only OpenAI turns (#66167)
fix: normalize OpenAI minimal reasoning
```

修復 reasoning-only OpenAI turns 的恢復問題，正規化 OpenAI minimal reasoning。

### Ollama 改善

```
fix(ollama): enable streaming usage for openai-compat (#66439)
```

為 Ollama streaming completions 發送 `stream_options.include_usage`，使本地 Ollama 報告真實 usage 而非備用的虛假 prompt-token 計數（後者會觸發提前 compaction）。

---

## 7. 頻道重播與重試標準化

```
fix(discord): make inbound retries explicit
fix(slack): make inbound retries explicit
fix(matrix): make delivery replay retries explicit
fix(line): make webhook replay retries explicit
fix(feishu): make card action retries explicit
fix(feishu): make bot menu retries explicit
fix(feishu): keep comment replay closed after generic failures
fix(mattermost): make replay retries explicit
fix(mattermost): dedupe repeated model picker selects
fix(whatsapp): make inbound retries explicit
fix(zalo): make replay retries explicit
fix(nextcloud-talk): make replay retries explicit
fix(nextcloud-talk): release replay claims on handler failure
fix(nostr): dedupe deterministic rejected events
fix(nostr): retry inbound events after handler failures
fix(tlon): release replay claims after handler failures
fix(voice-call): keep retryable errors replayable
fix(voice-call): retry rejected inbound hangups
fix(voice-call): keep unknown-call replays retryable
fix(plugins): serialize interactive callback dedupe
fix(plugin-sdk): add claimable dedupe helper
fix(plugin-sdk): serialize claimable dedupe races
refactor(feishu): share synthetic event dedupe claims
refactor(feishu): reuse persistent dedupe lookups
refactor(line): share replay dedupe guard
```

這是本次版本最重要的架構性改善之一：**標準化跨所有頻道的重播重複資料刪除（replay dedupe）、可重試失敗釋放和成功後提交行為**，確保：
- 成功後保持 reply-once
- 預交付失敗後乾淨地重試

覆蓋的頻道包括：Telegram、Discord、Slack、Mattermost、WhatsApp、Matrix、LINE、Feishu、Zalo、Nextcloud Talk、TLON、Nostr、Voice Call 以及共用的 plugin interactive callbacks。

Plugin SDK 新增 `claimable dedupe helper`，序列化競態條件，提供統一的去重複基礎設施。

---

## 8. Context Engine 改善

### Per-Iteration Ingest 與 Compaction

```
fix(agents) context-engine: per-iteration ingest and assemble for compaction (#63555)
```

Context engine 從第一個 tool-loop delta 開始進行 compact，並在 `afterTurn` 缺失時保留 ingest fallback，使長時間執行的 tool loops 保持有界而不丟失 engine state。

### Idle-Aware 背景維護

```
Run context-engine turn maintenance as idle-aware background work (#65233)
```

將 opt-in turn 維護作為 idle-aware 背景工作執行，使下一個前景 turn 不再需要等待主動維護。

### Context Engine 合約驗證

```
fix: validate resolved context engine contracts (#63222)
```

拒絕報告的 `info.id` 與其已註冊的 slot id 不符的 plugin engines，使格式錯誤的 engines 在基於 id 的 runtime 分支出現問題之前快速失敗。

---

## 9. 記憶系統修復

### Embedding 修復

```
fix(memory): preserve embedding proxy provider prefixes (#66452)
fix(memory): restore ollama embedding adapter (#66269)
```

- **Proxy Provider 前綴**：正規化 OpenAI 相容 embedding 模型引用時保留非 OpenAI provider 前綴，修復 proxy-backed memory providers 失敗顯示 `Unknown memory embedding provider` 的問題
- **Ollama Embedding Adapter**：恢復 Ollama embedding adapter

### Dreaming / Memory-Core 修復

```
fix(memory-core): run Dreaming once per cron schedule (#66139)
fix(memory): unify default root memory handling (#66141)
fix(active-memory): Move active memory recall into the hidden prompt prefix (#66144)
```

- **Dreaming 一次/排程**：修復 Dreaming 在每個 cron schedule 中只執行一次
- **預設根記憶體**：統一預設根記憶體處理
- **Active Memory Recall 位置**：將 active memory recall 移入隱藏的 prompt prefix（避免出現在可見歷史中）

---

## 10. Gateway 與配置系統

### Session / Route 修復

```
Gateway/sessions: preserve shared session route on system events (#66073)
fix(session): clear stale thread route on system events
fix(heartbeat): preserve Telegram topic routing for isolated heartbeats (#66035)
```

- **共用 Session Route**：在系統事件期間保留共用 session route
- **Stale Thread Route**：在系統事件時清除過期的 thread route
- **Heartbeat Telegram Topic**：保留孤立 heartbeats 的 Telegram topic routing

### Cron 修復

```
fix(cron): stop unresolved next-run refire loops (#66083)
fix(cron): preserve unresolved next-run backoff (#66113)
```

修復未解析的 next-run 導致的重複觸發迴圈，保留未解析的 next-run backoff。

### Hooks 修復

```
fix(hooks): pass workspaceDir in gateway session reset internal hook context (#64735)
fix(hooks): honor configured ollama slug timeout (#66455)
```

- 在 gateway session reset internal hook context 中傳遞 `workspaceDir`
- 讓 LLM-backed session-memory slug 生成遵守明確的 `agents.defaults.timeoutSeconds` 覆蓋設定

### Subagent Registry 修復

```
fix: preserve subagent registry runtime import path across source and dist (#66420)
fix(build): include subagent-registry.runtime.js in dist output (#66205)
```

修復 subagent registry lazy-runtime stub 未正確包含在 dist 輸出中，導致 `ERR_MODULE_NOT_FOUND` 錯誤。

### Gateway Config 防護

```
fix(gateway): scope reset hook assertion
fix(gateway): harden service entrypoint resolution (#65984)
```

---

## 11. 其他頻道修復

### Feishu（飛書）

```
Feishu: tighten allowlist target canonicalization (#66021)
```

加強 allowlist target 規範化，提升安全性。

### Slack

```
fix(slack): align interaction auth with allowlists (#66028)
fix: allow plugin commands on Slack when channel supports native commands (#64578)
fix(slack): isolate doctor contract API (#63192)
```

- **互動認證**：將全域 `allowFrom` owner allowlist 套用到 channel block-action 和 modal interactive 事件
- **Plugin Commands**：當頻道支援 native commands 時允許 plugin commands 在 Slack 上運行
- **Doctor Contract API**：隔離 doctor contract API

### WhatsApp

```
fix(whatsapp): await write stream finish before returning encFilePath (#65896)
fix: mirror baileys root dependency
fix: keep baileys plugin-local
```

- 等待 write stream 完成後才回傳 encFilePath，修復加密媒體文件被讀取前尚未完全寫入的問題
- 保持 Baileys 作為 plugin-local 依賴

### Voice Call / Stream

```
fix(stream): tighten voice stream ingress guards (#66027)
fix(voice-call): keep retryable errors replayable
fix(voice-call): retry rejected inbound hangups
fix(voice-call): keep unknown-call replays retryable
```

語音通話錯誤處理加強，各類錯誤情況支援重試。

### Queue / Outbound

```
fix(queue): split collect batches by auth context (#66024)
fix(outbound): replay queued session context (#66025)
fix(outbound): suppress relay status placeholder leaks
fix: sendPolicy deny should suppress delivery, not inbound processing (#53328) (#65461)
```

- **批次分割**：按 auth context 分割 collect 批次
- **Session Context 重播**：重播排隊的 session context
- **sendPolicy 修復**：`sendPolicy: "deny"` 只抑制交付，不阻止入站訊息處理，observer-style 設定仍可執行 agent turn

### Models 修復

```
fix(models): normalize google-vertex flash-lite ids
fix(google): strip Gemini compat base suffixes (#66445)
fix(tools): normalize media model lookups (#66422)
```

- **Google Vertex Flash-Lite**：正規化 Google Vertex flash-lite 模型 ID
- **Gemini Base Suffix**：從配置的 Google base URLs 中僅在呼叫原生 Gemini image API 時剝離尾部 `/openai` suffix
- **Media Model Lookup**：正規化 configured provider/model refs，修復 Ollama vision models 被拒絕為未知的問題

### Onboarding 修復

```
fix(onboard): cap compat probe max_tokens (#66450)
```

使用 `max_tokens=16` 進行 OpenAI 相容驗證探測，修復嚴格的自訂端點拒絕 onboarding 檢查的問題。

### Media 修復

```
fix(media): remap AAC uploads to M4A (#66446)
fix(media-understanding): auto-upgrade provider HTTP helper to trusted env proxy mode (#66458)
fix(tts): allow OpenClaw temp directory paths in reply media normalizer (#63511)
```

- **AAC → M4A**：將 `.aac` 文件名重映射為 `.m4a` 用於 OpenAI 相容音頻上傳
- **Media-Understanding Proxy**：當 `HTTP_PROXY`/`HTTPS_PROXY` 啟用且目標未被 `NO_PROXY` 繞過時，自動升級 provider HTTP helper 到受信任的 env-proxy 模式
- **TTS 路徑**：允許 OpenClaw temp 目錄路徑在 reply media normalizer 中

### Telegram Proxy / Media

```
fix(telegram): trust explicit proxy DNS for media downloads (#66461)
```

讓 Telegram media fetches 在 hostname-policy 檢查後信任操作員配置的明確 proxy 進行目標 DNS 解析。

### Mattermost

```
fix(mattermost): dedupe repeated model picker selects
```

去重複的 model picker 選擇。

### BlueBubbles

```
fix(bluebubbles): lazy-refresh Private API status on send (#43764) (#65447)
```

當請求 reply threading 或 message effects 但狀態未知時，在傳送時懶刷新 Private API server-info 快取。

### Nostr

```
fix(nostr): dedupe deterministic rejected events
fix(nostr): retry inbound events after handler failures
```

### Plugins

```
fix(plugins): serialize interactive callback dedupe
fix(plugins): treat context-engine plugins as capabilities in status/inspect (#58766)
143c1e82 fix(plugins): cache external plugin catalog lookups in auto-enable
```

- **序列化去重複**：序列化 interactive callback 去重複競態
- **Context-Engine 能力**：在 `plugins inspect` 中報告已註冊的 context-engine ID 而非擁有插件 ID
- **外部目錄快取**：快取外部 `preferOver` 目錄查找，修復大型 `agents.list` 配置導致 CPU 峰值的問題

### Runtime

```
fix(runtime): avoid leaking detached cleanup promises
fix: stop repeated unknown-tool loops (#65922)
fix: count unknown-tool retries only when streamed (#66145)
fix: classify openrouter json 404 model errors
```

- 避免洩漏 detached cleanup promises
- 停止重複的 unknown-tool 迴圈
- 分類 OpenRouter JSON 404 模型錯誤

### Lobster

```
Lobster: import published core runtime (#64755)
```

在 process 中載入已發布的 `@clawdbot/lobster/core` runtime。

### QR Code

```
fix(qr): lazy load terminal ascii renderer
fix(qr): lazy load terminal runtime modules
```

QR code 終端 ASCII 渲染器和相關模組改為懶載入。

### Agents

```
agents: stop strict mode from hijacking chat turns
Agents: clarify local model context preflight (#66236)
Agents: fix Windows drive path join for read/sandbox tools (#54039) (#66193)
```

- 停止嚴格模式搶占 chat turns
- 明確本地模型 context preflight 提示，停止在 `agents.defaults.contextTokens` 是實際限制時建議使用更大的模型
- 修復 Windows drive path join 用於 read/sandbox 工具

---

## 12. UI / Control UI

### ReDoS 安全修復

```
fix(ui): replace marked.js with markdown-it to fix ReDoS UI freeze (#46707) thanks @zhangfnf
```

將 marked.js 替換為 markdown-it，修復惡意製作的 markdown 可能透過 ReDoS 凍結 Control UI 的漏洞。

### Session 保留

```
fix(ui): preserve user-selected session on reconnect and tab switch (#59611) thanks @loong0306
```

在重連和 tab 切換時保留使用者選擇的 session。

### Dreaming UI 防護

```
[codex] fix(ui): guard dreaming wiki plugin calls (#66140)
```

防護 dreaming wiki plugin 呼叫。

---

## 13. QA Lab 改善

```
test(qa-lab): seed broken-turn recovery scenarios (#66416)
test(qa-lab): cover GPT-style broken turns
test: harden qa-lab concurrent web scenarios
test: harden video live provider release gate
```

新增 QA 場景：
- **Broken Turn Recovery**：seed 損壞 turn 恢復場景
- **GPT-Style Broken Turns**：GPT 風格損壞 turn 測試
- **Concurrent Web**：加強 concurrent web 場景測試
- **Video Release Gate**：加強影片 live provider 發布閘

---

## 14. 文件更新

### 新增 Hostinger 安裝指南

```
feat(docs): add Hostinger installation guide and link in VPS document (#65904)
```

新增 Hostinger VPS 安裝指南（`docs/install/hostinger.md`）並在 VPS 文件中添加連結。

### Docker-out-of-Docker 文件

```
docs(gateway): Document Docker-out-of-Docker Paradox and constraint (#65473)
```

文件化 Docker-out-of-Docker Paradox 和約束，讓使用者了解在 Docker 容器內執行 Gateway 時的限制。

### Changelog 改善

```
docs(changelog): note telegram topic-name persistence
docs(changelog): add 2026.4.12 dedupe note
docs(changelog): note sendPolicy suppressDelivery + BB Private API cache fixes (#66220)
docs(changelog): note perf fixes
docs(changelog): tidy unreleased entries
docs: clarify npm dist-tag auth
docs: prepare changelog for 2026.4.14
```

### NPM Dist-Tag

```
ci: add stable npm dist-tag sync
docs: clarify npm dist-tag auth
```

新增穩定 npm dist-tag 同步 CI，並澄清 dist-tag auth 文件。

---

## 15. 測試與 CI

### 測試改善

```
test(telegram): add inbound retry regressions (#66075)
test(telegram): cover topic-name cache reload
test(cron): fix #66019 maintenance regression coverage (#66122)
test(cron): mock skills snapshot runtime seam
test(wizard): mock auth profile runtime seam
test: enforce npm pack budget in install smoke
test: remove timer dependency from telegram topic cache tests
test: bound canvas auth helper waits
```

### CI 修復

```
fix(ci): avoid frozen hook test clock hangs
fix(ci): repair telegram ui and watch regressions
fix(ci): repair telegram topic cache typing
fix(ci): repair agent test mocks
fix(ci): align cron tests with default model
fix(ci): restore plugin-local whatsapp deps
fix(ci): repair baileys lockfile snapshot
fix(ci): clear residual tsgo blockers
fix(ci): align cron and session tests with runtime
fix(ci): mirror whatsapp runtime dependency
fix(ci): repair extension boundary contracts
fix(ci): unblock discord boundary typing
fix(ci): verify bundled plugin runtime deps (延續前版)
```

大量 CI 修復確保測試套件在大規模重構後保持穩定。

---

## 16. 摘要統計

### 提交分類

| 類別 | 數量 | 百分比 |
|------|------|--------|
| perf | 55+ | ~21% |
| fix | 110+ | ~41% |
| test | 40+ | ~15% |
| feat | 5 | ~2% |
| docs | 12 | ~4.5% |
| chore/build/ci | 20+ | ~7.5% |
| refactor | 8 | ~3% |
| 其他 | 17+ | ~6% |

### 版本里程碑

| 版本 | 日期 | 性質 |
|------|------|------|
| 2026.4.12 | 2026-04-12 | Stable（起始點） |
| 2026.4.14-beta | 2026-04-13 | Beta |
| 2026.4.14 | 2026-04-14 | Stable |

### 社群貢獻

| 貢獻者 | 提交數/主要貢獻 |
|--------|----------------|
| @ptahdunbar | Telegram topic names |
| @neeravmakwana | Store limits, approval timeout |
| @zhangfnf | ReDoS fix (marked.js → markdown-it) |
| @loong0306 | UI session persistence |
| @tmimmanuel | Systemd dotenv security |
| @yfge | Plugin catalog cache |
| @obviyus | Multiple fixes (browser, telegram) |
| @ben-z | AAC → M4A media fix |
| @100yenadmin | Context engine idle-aware work |
| @fuller-stack-dev | Context engine validation |
| @pgondhi987 | Security AI fixes |

### 關鍵技術趨勢

1. **效能優化大波次**：50+ perf 提交系統性重構懶載入和窄化匯入，全面降低冷路徑開銷
2. **頻道重播標準化**：統一 15+ 頻道的 replay dedupe/retry 行為，Plugin SDK 提供 claimable dedupe helper
3. **Telegram Forum Topics**：Topic 名稱在 agent context 中可見，持久化快取重啟後可用
4. **Browser SSRF 調整**：預設放寬 + 嚴格模式保留，修復多個 loopback CDP 問題
5. **GPT-5.4-Pro 前向相容**：在上游 catalog 更新前提前支援新模型
6. **安全防護**：Gateway config mutation 防護、heartbeat owner 降級、MS Teams SSO allowlist
7. **ReDoS 修復**：Control UI 從 marked.js 遷移至 markdown-it
8. **Context Engine 改善**：per-iteration ingest + idle-aware 背景維護
9. **Ollama 改善**：streaming usage、slug timeout、embedding adapter 恢復
10. **記憶系統穩定**：Embedding proxy prefixes、Dreaming 頻率修復、Active Memory 位置調整
