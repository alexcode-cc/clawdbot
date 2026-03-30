# OpenClaw 提交分析報告 — 2026-03-28

> **分析範圍**：從 `cff6dc94e3`（2026.3.24 release）至 `f9b1079283`（2026.3.28 stable），共 **1,267 個非合併提交**，涉及 **3,806 個檔案**、新增 217,667 行、刪除 92,006 行。

---

## 目錄

1. [重大變更 (Breaking / Major Changes)](#1-重大變更)
2. [新功能 (New Features)](#2-新功能)
3. [安全強化 (Security)](#3-安全強化)
4. [xAI 深度整合](#4-xai-深度整合)
5. [頻道改進 (Channels)](#5-頻道改進)
6. [記憶系統改善](#6-記憶系統改善)
7. [Gateway 與 CLI](#7-gateway-與-cli)
8. [模型與提供者](#8-模型與提供者)
9. [Plugin 與 Runtime 重構](#9-plugin-與-runtime-重構)
10. [Compaction 改善](#10-compaction-改善)
11. [平台相容性](#11-平台相容性)
12. [測試與 CI](#12-測試與-ci)
13. [文件與發布](#13-文件與發布)
14. [摘要統計](#14-摘要統計)

---

## 1. 重大變更

### 移除 Qwen OAuth 整合

```
Remove Qwen OAuth integration (qwen-portal-auth) (#52709)
```

移除 `qwen-portal-auth` 擴充套件（Qwen Portal OAuth 認證）。Qwen 使用者應改用 API Key 認證（`modelstudio-api-key`）。

### 核心 Provider Model 定義遷移

```
refactor: remove core provider model definitions compat
```

移除核心模組中的 provider model 定義相容層。所有模型定義現完全由 provider plugins 擁有。

---

## 2. 新功能

### 2.1 xAI x_search 與 code_execution

```
Tools: add x_search via xAI Responses
Tools: add xAI-backed code_execution
feat(xai): add plugin-owned x_search onboarding
```

xAI 新增兩大工具，完全由 xAI plugin 擁有：
- **x_search**：透過 xAI Responses API 的 web 搜尋，直接從 X（Twitter）獲取即時資訊
- **code_execution**：xAI 原生程式碼執行環境
- 完整的 plugin-owned onboarding 流程
- 詳見 [第 4 節](#4-xai-深度整合)

---

### 2.2 Microsoft Foundry Provider

```
feat: Add Microsoft Foundry provider with Entra ID authentication (#51973)
```

新增 Microsoft Foundry LLM 提供者，支援 Entra ID（Azure AD）認證：
- 提供者 ID：`microsoft-foundry`
- 認證：Entra ID JWT
- 透過 Microsoft AI 平台存取模型

---

### 2.3 Anthropic Claude CLI 遷移

```
feat: add anthropic claude cli migration
```

新增 Anthropic Claude CLI 歷史紀錄遷移工具：
- 將 Claude CLI 對話歷史匯入 OpenClaw
- 支援 tool messages 的統一轉換
- Stream-JSON 輸出防止 watchdog timeouts

---

### 2.4 OpenClaw Channel MCP Bridge

```
feat: add openclaw channel mcp bridge
```

新增 Channel MCP（Model Context Protocol）橋接器，將 OpenClaw 頻道暴露為 MCP 端點。

---

### 2.5 ACP Conversation Binds

```
feat(acp): add conversation binds for message channels
```

ACP 新增 conversation binds，讓 Agent 可以綁定至特定的訊息頻道對話。

---

### 2.6 Hooks before_tool_call 非同步審批

```
feat(hooks): add async requireApproval to before_tool_call (#55339)
```

`before_tool_call` hook 新增非同步 `requireApproval` 功能，讓外部系統在工具執行前進行核准或拒絕。

---

### 2.7 Video Generation 基礎建設

```
feat: add video generation core infrastructure and extend image generation parameters (#53681)
```

新增 video generation 核心基礎建設，擴展 image generation 參數。

> 注意：此功能在後續被 revert（#54943），然後在 2026.3.28 中重新引入。

---

### 2.8 CLI JSON Schema 工具

```
feat(cli): add json schema to cli tool (#54523)
```

CLI tool 工具新增 JSON Schema 支援。

---

### 2.9 CLI 推論後端插件化

```
feat: pluginize cli inference backends
```

CLI 推論後端遷移至 plugin 架構。

---

### 2.10 Plugin runId 暴露

```
feat(plugins): expose runId in agent hook context (#54265)
```

Agent hook context 中暴露 `runId`，讓 plugins 可以追蹤特定的 agent run。

---

### 2.11 Plugin System API: runHeartbeatOnce

```
plugin-runtime: expose runHeartbeatOnce in system API (#40299)
```

Plugin runtime system API 新增 `runHeartbeatOnce()`，讓 plugins 可以觸發單次心跳。

---

### 2.12 Tavily Extra Headers

```
feat: add support for extra headers in Tavily API requests (#55335)
```

Tavily API 請求支援自訂 extra headers。

---

## 3. 安全強化

### 3.1 路徑遍歷防護

```
fix: reject path traversal and home-dir patterns in media parse layer (#54642)
```

在媒體解析層拒絕路徑遍歷和 home-dir 模式。

### 3.2 CWD .env 過濾

```
Filter untrusted CWD .env entries before OpenClaw startup (#54631)
```

啟動前過濾不受信任的 CWD `.env` 條目。

### 3.3 Gateway 安全

| 修復 | 說明 | PR |
|------|------|-----|
| Reconnect scope escalation | 封鎖靜默重連的 scope 升級 | #54694 |
| Admin verbose defaults | 要求 admin 權限設定持久化 verbose | #55916 |
| Device session revocation | 中斷已撤銷裝置的 sessions | #55952 |
| Node pairing approvals | 限制 node pairing 審批 | #55951 |
| Reset-profile blocking | 封鎖較低權限的 reset-profile 請求 | #54618 |
| Host-env blocklist | daemon 安裝時套用 auth-profile env 封鎖清單 | #54627 |

### 3.4 頻道安全

| 修復 | 說明 | PR |
|------|------|-----|
| LINE HMAC timing-safe | 消除 HMAC 簽名驗證中的 timing side-channel | #55663 |
| Feishu webhook signatures | 解析前驗證 webhook 簽名 | #55083 |
| Telegram DM auth | 對 callbacks 強制 DM auth | #55112 |
| Telegram admin scope | writeback 需要 admin scope | #54561 |
| Feishu card commands | 拒絕 legacy raw card 指令 payload | #55130 |
| Google Chat group ids | 要求穩定的 group ids | #55131 |
| Matrix verification | gate verification notices on DM access | #55122 |
| Synology webhook throttle | 限流 webhook token 猜測 | #55141 |
| BlueBubbles webhook throttle | 限流 webhook auth 猜測 | #55133 |
| Feishu localRoots sandbox | docx 上傳檔案讀取強制 localRoots sandbox | #54693 |
| MS Teams feedback auth | 對齊 feedback invoke 授權 | #55108 |

### 3.5 Web Search 安全

```
fix(security): audit web search keys for all bundled providers (#56540)
```

審計所有 bundled web search provider 的金鑰安全。

### 3.6 其他安全

| 修復 | 說明 | PR |
|------|------|-----|
| ACP terminal titles | 消毒 terminal 工具標題 | #55137 |
| SSRF guard | extension fetch 呼叫路由通過 SSRF guard | #53929 |
| Talk-voice admin scope | /voice set config writes 需要 operator.admin | #54461 |
| OpenAI-compat scope | HTTP OpenAI 相容路由要求 operator.write | #56618 |
| Session_status guard | 強制 session_status guard after sessionId resolution | #55105 |

---

## 4. xAI 深度整合

本次更新包含 **30+ 個 xAI 相關提交**，是最大的單一功能區塊：

### 架構遷移

xAI provider 的 `x_search` 和 `code_execution` 完全遷移至 plugin 架構：

```
xAI: switch bundled provider defaults to Responses
refactor(xai): move x_search into plugin
refactor(xai): move code_execution into plugin
```

### x_search 工具

透過 xAI Responses API 提供即時 X（Twitter）搜尋：
- 路由通過公開 API seam
- 使用 fallback auth 解析
- Plugin key 重用
- 完整的 onboarding 流程

### code_execution 工具

xAI 原生程式碼執行：
- Plugin-owned config
- 窄化的 config typing
- SecretRef 路徑清理

### Auth 改善

- 集中化 fallback auth 解析
- Config-backed auth 在 provider bootstrap 時遵從
- Session auth 在 embedded runs 中保留
- Auth 解析診斷
- 不支援的 payload 欄位和 Responses reasoning params 自動剝離
- Stale Grok transport 正規化至 Responses

---

## 5. 頻道改進

### 5.1 Telegram

| 改進 | 說明 | PR |
|------|------|-----|
| Forum topic routing | 保留 /new 和 /reset 的 forum topic routing | #56654 |
| Empty text replies | 跳過空文字回覆避免 GrammyError 400 crash | #56620 |
| Word boundary split | 長訊息在字詞邊界分割，不再截斷字詞中間 | #56595 |
| ReplyToMessageId | 發送前驗證 replyToMessageId | #56587 |
| Model name display | 模型選擇器顯示名稱而非 ID | #56165 |
| Self-authored updates | 忽略自己發送的 DM message updates | #54530 |
| Forum topic verbose | 在 forum topics 中傳遞 verbose tool summaries | #43236 |
| Token fallback | 解析 binding-created accounts 的 token fallback | #54362 |
| Command error envelopes | 處理 telegram command error envelopes | — |
| Malformed reactions | guard malformed telegram reaction payloads | — |

### 5.2 Matrix

| 改進 | 說明 | PR |
|------|------|-----|
| E2EE 縮圖加密 | 在 E2EE rooms 中使用 thumbnail_file 加密縮圖 | #54711 |
| ESM __dirname | 使用 createRequire 修復 ESM 中的 __dirname | #54566 |
| Plugin load crash | 防止 matrix-js-sdk plugin 載入 crash | #56273 |
| Runtime deps | 包含 bundled 安裝的 runtime deps | — |
| Directory auth | 修復 directory auth 和 credentials fallback | — |
| Lazy-load setup | lazy-load matrix setup bootstrap surfaces | — |

### 5.3 MS Teams

| 改進 | 說明 | PR |
|------|------|-----|
| Entra JWT | 接受 strict Bot Framework 和 Entra service tokens | #56631 |
| Search message | 新增搜尋訊息 action | #54832 |
| Thread history | 透過 Graph API 擷取 channel reply thread 歷史 | #51643 |
| Feedback auth | 對齊 feedback invoke 授權 | #55108 |

### 5.4 WhatsApp

| 改進 | 說明 | PR |
|------|------|-----|
| Self-chat echoes | 使用 outbound ID tracking 丟棄 fromMe echoes | #54570 |
| Quoted messages | 解包 quoted wrapper messages | — |
| AllowFrom error | 釐清 allowFrom policy 錯誤訊息 | #54850 |

### 5.5 BlueBubbles

| 改進 | 說明 | PR |
|------|------|-----|
| Group participants | 用本地 Contacts 名稱豐富群組參與者 | #54984 |
| Private network | 本地 serverUrl 自動允許 private network | — |
| Inbound image | 恢復 CLI routed turns 的 inbound image embedding | #51373 |
| Debounce flush | guard debounce flush against null text | #56573 |

### 5.6 Feishu / Lark

| 改進 | 說明 | PR |
|------|------|-----|
| Card commands | 拒絕 legacy raw card 指令 payload | #55130 |
| Webhook signatures | 解析前驗證 webhook 簽名 | #55083 |
| LocalRoots sandbox | docx 上傳強制 localRoots sandbox | #54693 |

### 5.7 其他頻道

| 頻道 | 改進 | PR |
|------|------|-----|
| LINE | timing-safe HMAC 簽名驗證 | #55663 |
| LINE | 使用 configured field 進行 status checks | #45701 |
| Slack | 新增 upload-file action | #54987 |
| iMessage | 停止在 outbound text 中洩漏 reply tags | #39512 |
| Mattermost | thread resolved cfg through reply delivery | #48347 |
| Signal | 路由 runtime barrel off denied subpath | — |

---

## 6. 記憶系統改善

### 6.1 CJK 記憶分塊

```
fix(memory): account for CJK characters in QMD memory chunking (#40271)
fix: guard fine-split against breaking UTF-16 surrogate pairs
fix(memory): add CJK/Kana/Hangul support to MMR tokenize() (#29396)
```

重大中日韓語言支援改善：
- **QMD 分塊**：正確計算 CJK 字元（中/日/韓/越南語漢字）的 token 數
- **UTF-16 代理對保護**：fine-split 不再切斷 UTF-16 surrogate pairs
- **MMR 多樣性**：`tokenize()` 支援 CJK/Kana/Hangul 字元的正確分詞

### 6.2 Memory Flush 追加修復

```
fix: keep memory flush daily files append-only (#53725)
```

修復記憶沖洗可能覆寫（而非追加）每日記憶檔案的回歸。新增追加保護 guard 和回歸測試。

### 6.3 QMD 改善

- `embedInterval` 獨立於 `update interval` 運作
- 修復 slugified QMD search paths 解析

---

## 7. Gateway 與 CLI

### 7.1 Gateway 安全強化

| 項目 | 說明 | PR |
|------|------|-----|
| Reconnect scope | 封鎖靜默重連的 scope 升級 | #54694 |
| Admin verbose | 要求 admin 權限設定持久化 verbose | #55916 |
| Device revocation | 中斷已撤銷裝置的 sessions | #55952 |
| Node pairing | 限制 node pairing 審批 | #55951 |
| Chat.send reset | 對齊 chat.send reset scope 檢查 | #56009 |
| Synthetic origins | 支援 synthetic chat origins | — |

### 7.2 CLI 改善

| 項目 | 說明 | PR |
|------|------|-----|
| Remote gateway confirm | 發現遠端 gateway 前確認再存入 config | #55895 |
| JSON Schema tool | CLI tool 新增 JSON Schema | #54523 |
| Claude CLI migration | Anthropic Claude CLI 歷史匯入 | — |
| Codex responses WS | Codex responses 路由至 WebSocket | #53702 |

### 7.3 Compaction 觸發改善

```
Trigger preflight compaction from transcript estimates when usage is stale (#49479)
fix: trigger compaction on LLM timeout with high context usage (#46417)
fix(compaction): surface safeguard cancel reasons and clarify /compact skips (#51072)
```

- 使用量過時時從 transcript 估計觸發預飛 compaction
- LLM timeout 且 context 使用量高時觸發 compaction
- 顯示 safeguard cancel 原因並釐清 /compact 跳過

---

## 8. 模型與提供者

### 8.1 新增 / 變更

| 提供者 | 變更 | PR |
|--------|------|-----|
| Microsoft Foundry | 新增，Entra ID 認證 | #51973 |
| xAI | x_search + code_execution + Responses 預設 | — |
| Qwen Portal OAuth | **移除**（改用 API Key） | #52709 |
| Gemini 3.1 | 修復所有 Google provider aliases 的模型解析 | #56567 |
| Kimi Code | 恢復 Moonshot setup 下的 Kimi Code | #54619 |
| ModelStudio | 文件重命名、新增 Standard API endpoints | #54407 |

### 8.2 Provider Model 定義遷移

核心 provider model definitions 相容層已移除，所有模型定義現完全由 provider plugins 擁有。

### 8.3 Per-model Cooldown

```
fix: per-model cooldown scope, stepped backoff, and user-facing rate-limit message (#49834)
```

冷卻範圍改為 per-model（而非 per-provider），階梯式退避，用戶端顯示 rate-limit 訊息。

### 8.4 429 Rate Limit 靜默修復

```
fix: mid-turn 429 rate limit silent no-reply and context engine registration failure (#50930)
```

修復 mid-turn 429 rate limit 導致的靜默無回覆和 context engine 註冊失敗。

---

## 9. Plugin 與 Runtime 重構

本次包含 **259 個重構提交**，是大規模的 runtime 重構週期：

### 9.1 Channel Runtime 拆分

大量頻道的 runtime 功能被拆分並路由至獨立的 runtime barrel：

- Discord、Slack、Telegram、Matrix、IRC、Signal、Zalouser 等頻道的 runtime surfaces 分離
- Provider SDK runtime surfaces 穩定化
- 各頻道的 cold-runtime chunking 修復

### 9.2 Plugin Contract 系統

| 項目 | 說明 |
|------|------|
| Contract registries | 穩定化 provider contract loading |
| Extension contracts | lazy-load extension 和 web search contract registries |
| Capability providers | 保留 active capability providers |
| Channel metadata | 從 plugin manifests 衍生 channel metadata |

### 9.3 Regression 修復潮

本次包含 **20+ 個 regression 修復**，主要因 runtime 拆分引起：
- auto-enable channel/gateway/auth/directory/message/status 多項 selection 修復
- memory runtime 載入、channel status state、gateway plugin loads 修復
- 顯示此週期是一個積極的重構階段，伴隨密集的 regression 修復

---

## 10. Compaction 改善

| 項目 | 說明 | PR |
|------|------|-----|
| Preflight trigger | 使用量過時時從 transcript 估計觸發 | #49479 |
| LLM timeout trigger | 高 context 使用量時 LLM timeout 觸發 compaction | #46417 |
| Safeguard reasons | 顯示 cancel 原因並釐清 /compact 跳過 | #51072 |
| Session count reconcile | 晚期 compaction 成功後調和 session count | #45493 |
| NO_REPLY suppression | 壓制 JSON-wrapped NO_REPLY payloads | #56612 |

---

## 11. 平台相容性

### macOS
- 保持 local gateway attached
- 強化 mac gateway attach smoke
- Strip "OpenClaw " prefix before parsing gateway version

### Docker / Container
- WeChat channel 透過官方騰訊 iLink Bot plugin（#52131）

---

## 12. 測試與 CI

### 測試（~243 個 test/CI/chore 提交）

主要方向：
- **Contract 測試**：大量 extension contract registries 和 web search contract 測試
- **Parallels smoke**：MiniMax parallels、Anthropic parallels smoke lanes
- **Fixture 去重**：foundry auth、web search provider、redact/approval、pi compaction fixtures
- **Channel test 速度**：speed up channel test runs、enable planner parallelism

### CI
- 啟動 main required checks earlier
- Collapse preflight manifest routing
- Cap extension batch concurrency
- Balance shards

---

## 13. 文件與發布

### 發布

| 版本 | 時間 |
|------|------|
| `2026.3.25` unreleased | 中間準備 |
| `2026.3.28-beta.1` | Beta |
| `2026.3.28` stable | 正式發布 |

### 文件

- 新增 boundary AGENTS guides（#56647）
- xAI x_search onboarding 流程文件
- Anthropic Claude CLI migration 說明
- WeChat channel 文件（#52131）
- ModelStudio 重命名和 Standard API 端點文件
- CJK memory chunking changelog
- Secrets credential surface matrix 同步
- Beta blocker contributor guidance

---

## 14. 摘要統計

| 指標 | 數值 |
|------|------|
| 總提交數 | **1,267** |
| 影響檔案數 | **3,806** |
| 新增行數 | 217,667 |
| 刪除行數 | 92,006 |
| 淨增行數 | +125,661 |
| 版本跨度 | `2026.3.24` → `2026.3.28` |
| 新功能 | **12 項** |
| 安全修復 | **20+ 項** |
| Bug 修復 | **528** |
| 重構提交 | **~259** |
| 測試/CI 提交 | **~243** |

### 按模組分布

| 模組 | 變更強度 |
|------|---------|
| xAI 深度整合 | ■■■■■ 極高 |
| Runtime 重構 + Regression | ■■■■■ 極高 |
| 頻道安全強化 | ■■■■ 高 |
| Telegram | ■■■■ 高 |
| Matrix | ■■■ 中高 |
| Gateway 安全 | ■■■ 中高 |
| 記憶 CJK 支援 | ■■■ 中 |
| MS Teams | ■■■ 中 |
| Compaction | ■■ 中 |
| WhatsApp / BlueBubbles | ■■ 中 |
| Microsoft Foundry | ■■ 低中 |
| Claude CLI Migration | ■ 低 |
| Channel MCP Bridge | ■ 低 |

### 與前次分析比較

| 指標 | 2026-03-26 (3.24) | 2026-03-28 (3.28) | 變化 |
|------|-------------------|-------------------|------|
| 提交數 | 463 | 1,267 | +173% |
| 新功能 | 8 | 12 | +50% |
| 安全修復 | 12+ | 20+ | +67% |
| Bug 修復 | 227 | 528 | +133% |
| 重構 | ~42 | ~259 | +517% |

本次更新的突出特點：

1. **xAI 全面整合**：x_search + code_execution + Responses 預設 + plugin-owned auth，是最大的單一功能
2. **大規模 Runtime 重構**：channel/provider runtime 拆分，伴隨 20+ regression 修復
3. **CJK 記憶支援**：QMD chunking、UTF-16 保護、MMR tokenize 的中日韓語言修復
4. **頻道安全全面強化**：LINE timing-safe、Feishu webhook、Telegram DM auth 等 11 項
5. **Microsoft Foundry**：新增 Entra ID 認證的 LLM 提供者
6. **Memory Flush 追加修復**：修復可能覆寫每日記憶的回歸

---

*本文件由 AI 自動分析生成，分析基準點：`cff6dc94e3`（2026.3.24）→ `f9b1079283`（2026.3.28）*
