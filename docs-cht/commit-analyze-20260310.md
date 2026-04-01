# OpenClaw 提交分析報告 — 2026-03-10

> **分析範圍**：從 `3d508ce01b`（docs: 修正線框圖位移錯位，2026-03-04）至 `91b38f7a7`（2026-03-10 最新），共 **1,120 個提交**，涉及 **2,778 個檔案**、新增 151,485 行、刪除 30,799 行。

---

## 目錄

1. [重大變更 (Breaking / Major Changes)](#1-重大變更)
2. [新功能 (New Features)](#2-新功能)
3. [安全強化 (Security)](#3-安全強化)
4. [頻道改進 (Channels)](#4-頻道改進)
5. [Agent 與 Session](#5-agent-與-session)
6. [ACP (Agent Communication Protocol)](#6-acp)
7. [Gateway 與 CLI](#7-gateway-與-cli)
8. [Cron 與排程](#8-cron-與排程)
9. [瀏覽器工具 (Browser)](#9-瀏覽器工具)
10. [記憶與壓縮 (Memory / Compaction)](#10-記憶與壓縮)
11. [模型與提供者 (Models / Providers)](#11-模型與提供者)
12. [Plugin SDK 與擴充系統](#12-plugin-sdk-與擴充系統)
13. [平台相容性 (Platform)](#13-平台相容性)
14. [效能優化與重構](#14-效能優化與重構)
15. [測試與 CI](#15-測試與-ci)
16. [文件更新](#16-文件更新)
17. [摘要統計](#17-摘要統計)

---

## 1. 重大變更

> 升級前必讀，可能影響現有部署或外掛開發

### Plugin SDK：全面遷移至 Scoped Subpath Imports

```
MAJOR: Plugin SDK bundled subpath wiring — all plugins/extensions migrated to scoped imports
```

全部 30+ 個官方外掛和擴充套件已從 `openclaw/plugin-sdk` 根匯入遷移至 scoped subpath imports（如 `openclaw/plugin-sdk/channel`、`openclaw/plugin-sdk/allowlist` 等），共涉及約 **60+ 個提交**。舊的根匯入透過 root-alias shim 維持向下相容，但新外掛應使用 scoped imports。

**影響範圍**：自訂外掛若使用 `openclaw/plugin-sdk` 根匯入仍可運作，但建議遷移至 scoped imports 以獲得更好的 tree-shaking 和載入效能。

---

### Context Engine 外掛系統

```
MAJOR: Extend plugin system to support custom context management (#22201)
```

新增 Context Engine 外掛類型，允許外掛註冊自訂上下文管理策略。外掛可透過 `kind: "context-engine"` manifest 聲明註冊，並使用 slash-delimited schema lookup paths 進行配置。

**相關 PR**：#22201

---

### Android 套件重新命名

```
BREAKING: Android app package renamed to ai.openclaw.app
```

Android 應用套件名稱從舊名稱變更為 `ai.openclaw.app`，同時移除了麥克風、螢幕前台服務、背景定位模式和自我更新安裝流程（Google Play 政策合規）。

---

## 2. 新功能

### 2.1 GPT-5.4 支援

新增 OpenAI `gpt-5.4` 模型支援（API 與 Codex OAuth），搭配 1M context window。包含 implicit provider snapshot、transport overrides 正規化和 baseUrl scope 修正。

**相關 PR**：#36590

---

### 2.2 Gemini 3.1 Flash-Lite 支援

新增 Google `gemini-3.1-flash-lite` 模型支援，包含前向相容性處理和 compat 正規化。

---

### 2.3 Vercel AI Gateway 目錄發現

`models` 系統新增對 Vercel AI Gateway catalog 的自動發現支援。

---

### 2.4 Brave Search LLM Context API

`web_search` 工具新增 Brave Search LLM Context API 模式，除了原有的 Web Search API，現可切換至針對 LLM 最佳化的 Context API 回應格式。

**相關 PR**：#33383（感謝 @thirumaleshp）

---

### 2.5 本地備份 CLI

新增 `openclaw backup` CLI 指令，支援本地配置備份與驗證：

```bash
openclaw backup create
openclaw backup verify
```

**相關 PR**：#40163

---

### 2.6 Compaction 模型覆寫

允許透過配置覆寫壓縮使用的模型：

```json5
{
  agents: {
    defaults: {
      compaction: {
        model: "anthropic/claude-sonnet-4-5"
      }
    }
  }
}
```

**相關 PR**：#38753

---

### 2.7 Compaction Safeguard 品質控制

新增壓縮安全護欄的進階配置：

- **preserve/quality 設定**（#25557）：可配置壓縮後保留的上下文區段和品質等級
- **摘要品質審計重試**（#25556）：壓縮摘要可觸發自動重試以確保品質
- **結構化摘要標題要求**（#25555）：強制壓縮摘要使用結構化標題

---

### 2.8 Compaction 生命週期 Hooks

新增壓縮相關的 hook 事件：

```
compaction:start  — 壓縮開始
compaction:end    — 壓縮完成
```

可在 `before_prompt_build` 中使用 `prependSystemContext` 和 `appendSystemContext` 注入自訂上下文。

**相關 PR**：#16788、#35177

---

### 2.9 Gateway Auth SecretRef 支援

`gateway.auth.token` 現支援 SecretRef，並加入 auth-mode guardrails：

```json5
{
  gateway: {
    auth: {
      token: { "$ref": "secret:gateway-token" }
    }
  }
}
```

**相關 PR**：#35094

---

### 2.10 Gateway 密碼檔案輸入

`gateway run` 新增 `--password-file` 選項，從檔案讀取密碼而非命令列引數：

```bash
openclaw gateway run --password-file /path/to/password
```

**相關 PR**：#39067

---

### 2.11 Channel-Backed 就緒探測

Gateway 新增 channel-backed readiness probes，可透過頻道連線狀態判斷 gateway 是否就緒：

```bash
openclaw channels status --probe
```

**相關 PR**：#38285

---

### 2.12 iOS Live Activity 連線狀態

iOS 應用新增 Live Activity 顯示連線狀態，包含過期清理機制。

**相關 PR**：#33591

---

### 2.13 iOS App Store Connect 發布資產

iOS 應用新增 App Store Connect 發布資產準備流程。

**相關 PR**：#38936（感謝 @ngutman）

---

### 2.14 ACP 持久化綁定

ACP session 支援持久化的 Discord channel 和 Telegram topic 綁定，以及 pin binding message：

```json5
{
  acp: {
    bindings: {
      discord: { channelId: "..." },
      telegram: { topicId: "..." }
    }
  }
}
```

**相關 PR**：#34873、#36683

---

### 2.15 TTS baseUrl 支援

OpenAI TTS 配置新增 `baseUrl` 支援，可指向自訂 TTS 端點：

```json5
{
  tools: {
    tts: {
      openai: {
        baseUrl: "https://custom-tts.example.com/v1"
      }
    }
  }
}
```

**相關 PR**：#34321

---

### 2.16 西班牙語區域支援

新增西班牙語（Spanish）locale 支援。

**相關 PR**：#35038（感謝 @DaoPromociones）

---

### 2.17 Mattermost 互動式按鈕與模型選擇器

Mattermost 頻道新增互動式按鈕支援和互動式模型選擇器（model picker）。

**相關 PR**：#19957、#38767

---

### 2.18 nano-banana-pro 增強

影像生成工具新增：
- `--aspect-ratio` 旗標（#28159）
- `--background` 和 `--style` 選項驗證（#36762、#36648）
- `--output-format` 驗證與正規化（#36648）
- 明確 `--resolution` 於編輯時優先（#36880）

---

### 2.19 TUI 工作區 Agent 推斷

TUI 啟動時自動推斷當前工作區的 agent：

**相關 PR**：#39591

---

### 2.20 CLI --version 顯示 Commit Hash

`openclaw --version` 輸出現包含 git commit hash。

**相關 PR**：#39712

---

### 2.21 UTC 時間併行顯示

系統提示詞中的 `Current time` 行現同時顯示本地時間和 UTC 時間。

**相關 PR**：#32423

---

### 2.22 Onboarding Web Search 整合

新安裝的 onboarding 流程中加入 web search 配置步驟。

**相關 PR**：#34009

---

### 2.23 Slim Docker Image

提供精簡版 Docker image，大幅減少 runtime image 大小。

**相關 PR**：#38479

---

### 2.24 Container 可選擴充依賴

Docker/Podman 建置新增 `OPENCLAW_EXTENSIONS` build arg，可選擇性包含擴充套件依賴：

```dockerfile
docker build --build-arg OPENCLAW_EXTENSIONS="feishu,msteams" .
```

**相關 PR**：#32223

---

### 2.25 macOS 遠端 Gateway Token 欄位

macOS 應用遠端模式新增 gateway token 欄位，支援 Tailscale 發現改善和 unsupported token 保留。

---

### 2.26 Browserbase 遠端 CDP 支援

瀏覽器工具新增 Browserbase 作為託管遠端 CDP 選項，支援直接 WebSocket CDP URL：

```json5
{
  browser: {
    profiles: {
      browserbase: {
        cdpUrl: "wss://connect.browserbase.com?apiKey=..."
      }
    }
  }
}
```

---

### 2.27 Post-Compaction 上下文區段配置

壓縮後的上下文區段可透過配置自訂：

**相關 PR**：#34556

---

### 2.28 Agent Reply Pipeline 沖洗

Agent 在壓縮等待前先沖洗回覆管線，確保回覆不會遺失：

**相關 PR**：#35489

---

### 2.29 Agent Poll-Vote Action

Agent 新增通用的 poll-vote action 支援。

---

### 2.30 Path-Scoped Config Schema Lookup

Gateway 新增 path-scoped 配置 schema 查詢，支援 slash-delimited 路徑：

**相關 PR**：#37266

---

## 3. 安全強化

> 本次更新包含大量安全修復

### 3.1 Marker 替換邊界強化

`replaceMarkers()` 強化以捕捉空格/底線邊界標記變體，防止 prompt spoofing。

**相關 PR**：#35983

---

### 3.2 瀏覽器 SSRF 重定向檢查

瀏覽器工具強化 redirect-hop SSRF 檢查，防止惡意重定向繞過 SSRF 防護。

---

### 3.3 跨域重定向標頭過濾

跨域重定向時自動移除自訂 auth headers，防止憑證洩漏。

---

### 3.4 Sandbox fs-bridge 強化

沙箱檔案系統橋接多項強化：
- `readfile` handles 固定（pin）防止路徑遍歷
- `mkdirp` 路徑錨定
- 破壞性操作路徑錨定
- 路徑安全拆分

---

### 3.5 安裝基礎漂移清理

安全強化安裝基礎（install base）的漂移偵測和清理機制。

---

### 3.6 fs-safe 複製寫入強化

檔案系統安全複製寫入操作的邊界強化。

---

### 3.7 Zip 提取寫入強化

Zip 檔案提取時的寫入路徑強化，防止 path traversal。

---

### 3.8 Workspace Skill 路徑限制

技能（skill）路徑限制在工作區內，防止透過 symlink 逃逸。

---

### 3.9 Windows ACL 語系無關審計

使用 `icacls /sid` 進行語系無關的 Windows ACL 安全審計（取代語系依賴的輸出解析）。

**相關 PR**：#38900

---

### 3.10 SecretRef 安全持久化

`models.json` 持久化時保護 SecretRef，防止明文 API key 寫入：

**相關 PR**：#38955

---

### 3.11 MIME Spoofing 防護

拒絕偽造的 `input_image` MIME payload，防止惡意媒體注入。

**相關 PR**：#38289

---

### 3.12 Hook Prompt 策略驗證

Plugin prompt hooks 加入 runtime policy 驗證，確保 hook 符合安全策略。

**相關 PR**：#36567

---

### 3.13 Exec 安全強化

- 辨識 PowerShell encoded commands
- 綁定已核准的腳本操作數
- 綁定 bun/deno 核准腳本
- 封鎖危險的 env override-only pivots
- Shell 註解在 allow-always 分析中正確處理
- 驗證 skill 下載根目錄

---

### 3.14 Gateway Auth 強化

- Chat config 寫入需要 admin 權限
- Plugin HTTP route auth 強化
- Dashboard gateway tokens 不存入 URL storage
- Hook auth lockout 前先進行方法閘道
- config-first secretrefs fail-closed
- Cached device token override fallback 封鎖
- Envfile auth provenance 保留

---

### 3.15 Config 驗證日誌注入防護

配置驗證日誌輸出進行控制字元消毒，防止 log injection。

**相關 PR**：#39116

---

### 3.16 Cron Store 權限

Cron store 和 run logs 強制 `0600` 權限（owner-only）。

**相關 PR**：#36078

---

### 3.17 Prototype Chain 帳號路徑檢查

避免透過 prototype chain 的帳號路徑檢查攻擊。

**相關 PR**：#34982

---

### 3.18 依賴安全更新

- `tar` 升級至 7.5.10
- `hono` transitive audit 漏洞修補
- `dompurify` 升級至 3.3.2

---

## 4. 頻道改進

### 4.1 Telegram

| 改進項目 | 說明 | PR |
|---------|------|-----|
| Exec 審核支援 | OpenCode/Codex 執行審核 | #37233 |
| 網路 fallback 修復 | 移至 resolver-scoped dispatchers | #40740 |
| 預覽超時防重複 | 預覽編輯超時時防止重複訊息 | #41662 |
| DM 訊息預覽 | DM 中使用訊息預覽 | — |
| DM 草稿串流恢復 | 恢復 DM draft streaming | #39398 |
| 原生指令 auth | 群組 allowFrom 驗證、sender-only 正規化 | #39310、#36753 |
| Topic 指令路由 | 原生 topic commands 路由至 active session | #38871 |
| Slash SessionKey 命名空間 | 按 agent 命名空間化 | #38680 |
| Webhook 健康恢復 | 重啟後恢復 webhook 模式健康 | — |
| 下載超時 | 防止 polling loop 掛起 | #40098 |
| 偏移量確認強化 | 持久化偏移量確認和停滯恢復 | — |
| DM 草稿清除 | materialize 後清除 DM draft | #36746 |
| Poll gating 強化 | 輪詢閘道和 schema 一致性 | #36547 |
| Topic/Pin 綁定 | ACP topic binding 和 pin binding message | #36683 |
| DM 去重 | 每 agent 去重入站 DM 回覆 | #40519 |
| 409 衝突重設 | webhook 清理鎖存器重設 | #39205 |
| 持久化 update id 防護 | null 值正規化防護 | — |
| SecretRef 狀態解析 | provider-safe env 檢查 | — |
| Direct delivery hooks | 橋接至 internal message:sent hooks | #40185 |
| 停機中止 | 停機時中止 getUpdates fetch | #23950 |
| Dispatch 失敗降級 | 分發失敗時顯示 fallback | #39209 |

---

### 4.2 Discord

| 改進項目 | 說明 | PR |
|---------|------|-----|
| maxLinesPerMessage | 即時回覆中套用有效值 | #40133 |
| 原生指令防衝突 | 避免 native plugin command collisions | — |
| 非阻塞監聽器 | message listener 改為非阻塞 | #39154 |
| Guild slash auth | 遵從 commands.allowFrom | #38794 |
| Session key 保留 | 原生指令 session keys 保留 | — |
| DM session key 正規化 | DM session keys 正規化 | — |
| Model picker 誤警告 | 避免 false mismatch warning | #39105 |
| Exec auth 傳遞 | 傳遞 gateway auth 至 exec approvals | — |
| agentComponents 驗證 | 驗證 config schema parity | #39378 |
| 原生指令 args | 預設缺失的 native command args | — |
| outbound mediaMaxMb | 遵從上傳大小限制 | #38065 |

---

### 4.3 Feishu / Lark

| 改進項目 | 說明 | PR |
|---------|------|-----|
| 本地圖片轉換 | sendText 中傳遞 mediaLocalRoots | #40623 |
| 媒體超時恢復 | 恢復明確的媒體請求超時 | — |
| Block streaming 修復 | 停用 block streaming 防止靜默回覆丟失 | #38422 |
| SDK 超時修復 | 移除無效的 timeout 屬性 | #38267 |
| 完整回覆機制 | outbound replyToId 轉發 + topic-aware reply targeting | #33789 |
| MP4 影片類型 | 使用 msg_type media 發送 MP4 | #33674 |
| Streaming card 錯誤處理 | 檢查 response.ok | #35628 |
| Bot 提及偵測 | 使用 probed botName 和 ID 提及 | #36391、#36333 |
| groupPolicy 別名 | 接受 "allowall" 作為 "open" 別名 | #36358 |
| HTTP 超時 | 防止每聊天佇列死鎖 | #36430 |
| 全域 HTTP 超時修復 | 避免媒體回歸 | #36500 |
| 群組 slash 探測 | 正規化群組 slash command 探測 | — |
| Streaming 合併語義 | 強化 streaming merge 和最終回覆去重 | #33245 |

---

### 4.4 Slack

| 改進項目 | 說明 | PR |
|---------|------|-----|
| App_mention 重試 key | dedup 檢查前記錄 retry key | #37033 |
| 系統事件路由 | 路由至綁定 agent sessions | #34045 |
| Dedup 恢復 | 保留 dedup 同時恢復 dropped app_mention | #34937 |
| mediaLocalRoots 傳播 | 透過 send path 傳播 | — |
| Reactions channel ID | 入站 context 中傳遞 channel ID | #34831 |
| mrkdwn 重複轉換 | 移除 native streaming path 的重複轉換 | — |
| Stale socket 清除 | 斷線後清除 stale socket state | #39083 |
| allowlist scoping | auto-reply allowlist store 按帳號 scope | #39015 |

---

### 4.5 Mattermost

| 改進項目 | 說明 | PR |
|---------|------|-----|
| replyTo 參數 | plugin handleAction send 中讀取 replyTo | #41176 |
| threaded replies | payload.replyToId 作為 root_id | #27744 |
| 互動式按鈕 | 新增 interactive buttons 支援 | #19957 |
| 互動式模型選擇器 | 新增 model picker | #38767 |
| 互動回調綁定 | 強化 interaction callback binding | #38057 |
| Action lookup sentinel | 修復 interaction action lookup | — |
| 提及覆寫 | 遵從 onmessage mention override | #27160 |

---

### 4.6 其他頻道

| 頻道 | 改進項目 | PR |
|------|---------|-----|
| Signal | 轉發所有入站附件 | #39212 |
| iMessage | 防止 echo loop 洩漏內部 metadata 和 NO_REPLY 溢出 | #33295 |
| WhatsApp | 遵從 outbound mediaMaxMb | #38097 |
| Matrix | 恢復不依賴 memberCount 的 DM 路由 | #19736 |
| LINE | 完成 requireMention gating 行為 | #35847 |
| MS Teams | 強制 sender allowlists 搭配 route allowlists | — |
| Google Chat | 多帳號 webhook auth 繼承共用預設 | #38492 |
| Synology Chat | webhook gate 中傳遞 command auth | — |

---

## 5. Agent 與 Session

### 5.1 錯誤觀察

Agent 系統新增 embedded error observations 和 fallback error observations，提供更豐富的錯誤診斷上下文。

**相關 PR**：#41336、#41337

---

### 5.2 記憶沖洗路徑修復

修復 memory flush write path 轉發問題。

**相關 PR**：#41761

---

### 5.3 Bootstrap 保護

記憶沖洗時保護 bootstrap 文件（如 `AGENTS.md`）不被清除。

**相關 PR**：#38574

---

### 5.4 Compaction 穩定性

- 壓縮重試等待加入邊界，重啟時排空 embedded runs（#40324）
- 壓縮閾值套用 contextTokens cap（#39099）
- 無真實訊息時跳過壓縮 API call（#36451）
- `model_context_window_exceeded` 分類為 context overflow 並觸發壓縮（#35934）
- 溢出觸發的壓縮正確遞增計數器（#39123）
- 惡意 thinking blocks 的 context pruning 防護（#35146）
- 惡意 content entries 的 promoteThinkingTagsToBlocks 防護（#35143）
- 請求失敗時保留 totalTokens（#34275）
- Reply pipeline 在壓縮等待前沖洗（#35489）
- 壓縮後恢復 post-compaction sections（#34556）

---

### 5.5 提供者相容性修復

- xAI/Grok tool call arguments HTML entity 解碼（#35276）
- Venice 代理 xAI/Grok 模型偵測（#35355）
- OpenAI strict turn ordering 消毒（#39252）
- WebSocket stream path 尊重 `compat.supportsStore`（#39113）
- WebSocket `maxTokens=0` 正確轉發
- Kimi coding anthropic tool payload format 正規化
- Ollama configured endpoints 不需 dummy api keys
- OpenAI completions undici stream timeout 強化

---

### 5.6 Billing / Rate Limit 分類

- 單提供者 billing cooldown 探測（#41422）
- 402 temporary-limit 偵測擴展（#38533）
- `insufficient_quota` 400 分類為 billing（#36783）
- Generic cooldown text 避免 false global rate-limit 分類（#32972）
- Service-unavailable 窄化至需要 overload indicator（#32828）
- Auth cooldown error counters 到期重設（#41028）
- Double websocket retry accounting 修復（#39133）
- Failover HTTP status 分類共用（#36615）
- HTTP 402 分類為 rate_limit（#30484）
- 不明 model ids 的 clear warning（#39215）
- Rate limit patterns 新增 'too many tokens' 和 'tokens per day'（#39377）

---

### 5.7 工具執行修復

- 限制性 profiles 下重新暴露配置的工具（#39215）
- 惡性 tool call names 在 dispatch 前正規化（#39328）
- Tool call state 中斷清理改善
- Idle-timeout cleanup 避免 synthetic tool-result writes
- Exec-approvals ask=off 在 gateway/node runs 中正確遵從
- Allow-always 分析中遵從 shell comments
- xAI web_search tool-name collision 避免
- Unsupported responses store payloads 移除

---

### 5.8 Session 管理

- Session contextTokens 在 model switch 時清除（#38044）
- Session key 正規化為小寫（#34013）
- TUI /new sessions 按 client 隔離
- Daily/scheduled reset 時歸檔舊 transcript（#35493）
- Webchat 直接 UI turns 優先 webchat routes（#37135）
- Direct-session webchat routing 匹配收緊（#37867）
- Auto-reply system events timeline 恢復（#34794）
- Source channel 用於 agent event surfacing（#36030）

---

### 5.9 子代理改善

- Subagent workspace 繼承（#39247）
- ACP spawned child session history 持久化（#40137）
- ACP sessions_spawn parent streaming relay（#34310）
- Stuck ACP child processes 啟動時 kill（#33699）
- Announce delivery with descendant gating（#35080）
- Announce cleanup after kill/complete race recovery
- Leaked `[[reply_to]]` tags strip from completion announces（#34503）

---

## 6. ACP

### 6.1 可靠性強化

- Follow-up 可靠性和附件處理強化（#41464）
- 附件轉發至 ACP runtime sessions（#41427）
- IDE 客戶端的 streaming updates 豐富化（#41442）
- Session context 和 controls 恢復（#41425）
- Bridge mode 誠實失敗報告（#41424）
- `setSessionMode` gateway 錯誤傳播至客戶端（#41185）
- Error states 映射至 end_turn 而非 unconditional refusal（#41187）
- Inline delivery 避免 oneshot run spawns（#39014）
- Sandboxed slash spawns 封鎖

---

### 6.2 Ingress Provenance Receipts

ACP 新增可選的 ingress provenance receipts 機制。

**相關 PR**：#40473

---

### 6.3 Session Lineage

允許 ACP `sessions.patch` 設定 lineage fields。

**相關 PR**：#40995

---

## 7. Gateway 與 CLI

### 7.1 Node Pending 機制

Gateway 新增 pending node work primitives 和 tightened drain semantics。

**相關 PR**：#41409、#41429

---

### 7.2 啟動穩定性

- 啟動失敗在 run loop 中捕捉，防止 process exit（#35862）
- 重啟前驗證 config 防止 crash + macOS 權限遺失（#35862）
- 重啟 shutdown timeout 時 exit non-zero
- 非管理的 listeners 停止並重啟（#39355）

---

### 7.3 Gateway Auth 改善

- 跨 systemd、Discord、node host 統一的 shared auth resolution（#39067）
- Password-file input 支援
- SecretRef 支援（#35094）
- Config-first secretrefs fail-closed
- Dashboard tokens 不進入 URL storage
- launchd supervision 偵測透過 XPC_SERVICE_NAME

---

### 7.4 Control UI 修復

- Global 404s 修復（symlinked wrappers 和 bundled package roots）（#40385）
- Root-mounted control ui 中 probe routes 保持可達（#38199）
- 實際版本傳遞至 Control UI client（#35230）
- Tool events 即時串流至 control chat（#39104）
- Auth 跨 refresh 保留（#40892）
- marked.js parse errors 捕捉防止 crash（#36445）
- Accounts schema node 正確渲染（#35380）

---

### 7.5 路由修復

- Chat.send 內部路由洩漏防止
- Webchat route 繼承在 channel sessions 上停止（#39175）
- Shared-main chat.send 不繼承 stale external routes（#38418）
- Chat delta 在 tool-start events 前 flush（#39128）
- Streamed prefixes 跨 tool boundaries 保留

---

### 7.6 其他 CLI 改善

- One-shot exit hangs 修復（cached memory managers teardown）（#40389）
- Update restart failures 避免 false attribution（#39508）
- Auto-create inherited agent override entries
- Read-only SecretRef status flows 安全降級（#37023）
- Config 無效載入 fail-closed（#39071）
- Light-background 終端色彩對比改善（#40345）
- Plugin discovery cache 安裝後清除（#39752）
- Bundled channel plugins 優先於 npm duplicates（#40094）
- Parallel tool calls gating 至相容 APIs（#39356）

---

## 8. Cron 與排程

| 改進項目 | 說明 | PR |
|---------|------|-----|
| lastErrorReason | 記錄 job state 中的錯誤原因 | #14382 |
| NO_REPLY 分類 | 不將 empty/NO_REPLY 誤分類為 interim ack | #41401 |
| 交付合約 | 強制 cron-owned delivery contract | #40998 |
| 重啟錯開 | missed jobs 重啟時 stagger 防止過載 | #18925 |
| Owner-only 工具 | isolated runs 恢復 owner-only 工具 | — |
| 雙重公告修復 | 消除 double-announce，改用 push-based flow | #39089 |
| Best-effort fallback | 公告失敗後恢復直接 fallback | #36177 |
| 隔離交付預設 | 恢復 isolated delivery defaults | — |
| 手動 timeout 保留 | 新增時保留 manual timeoutSeconds | — |
| 備份保護 | 備份保留 pre-edit snapshot | #35195 |
| Stale-run 恢復 | 統一 stale-run recovery + 手動 run every anchors | #35363 |
| 啟動 replay | 窄化啟動 replay backoff guard | #35391 |
| Catch-up 語義 | 穩定重啟 catch-up replay 語義 | #35351 |
| 跨頻道公告 | 跨頻道 announce fallback 回歸修復 | #36197 |
| 交付正規化 | ingress 時正規化遺留 delivery 格式 | — |
| Legacy provider 遷移 | 遷移遺留 provider delivery hints | — |
| Cron announce delivery | 修復 Telegram targets 的文字公告交付 | #40575 |

---

## 9. 瀏覽器工具

| 改進項目 | 說明 | PR |
|---------|------|-----|
| SSRF 重定向檢查 | 強化 redirect-hop SSRF 檢查 | — |
| Browserbase CDP | 新增 Browserbase 作為託管遠端 CDP 選項 | — |
| WebSocket CDP URL | 支援直接 WebSocket CDP URL | — |
| Relay bind 配置 | browser relay bind address 可配置 | #39364 |
| Extension tab 等待 | relay drop 後等待 extension tabs | #32331 |
| Wildcard CDP URL | 正規化 wildcard 遠端 CDP URL | #17760 |
| wss:// 保留 | legacy profile 解析中保留 wss:// | — |
| Dispatcher context | no-retry hints 中保持 dispatcher context | — |
| Tab cleanup | session cleanup 時關閉追蹤的 tabs | #36666 |
| Deprecated flag 移除 | 移除 deprecated --disable-blink-features flag | — |
| CDP sessions scoping | scope CDP sessions 和強化 stale target recovery | — |
| Context engine registry | 跨 bundled chunks 共享 registry | #40115 |
| WSL2 remote Chrome | 新增 WSL2 遠端 Chrome CDP 故障排除指南 | #39407 |

---

## 10. 記憶與壓縮

### 10.1 QMD 穩定性

- 搜尋結果無 docid 的處理
- Collection name conflicts 修復
- Duplicate document constraints 恢復
- 避免 destructive collection rebinds
- SecretRef keys 在 doctor embeddings 中處理
- Windows EINVAL spawn 重試

---

### 10.2 BM25 相關性排序

保留 BM25 相關性排序，修復先前可能被打亂的結果順序。

**相關 PR**：#33757（感謝 @lsdcc01）

---

### 10.3 SQLite busy_timeout

reopen 時恢復 sqlite busy_timeout 設定。

**相關 PR**：#39183（感謝 @MumuTW）

---

### 10.4 Bootstrap 保護

記憶沖洗時保護 bootstrap 文件不被清除，ban timestamped variant files。

**相關 PR**：#38574、#34951

---

### 10.5 Local Embedding 序列化

序列化 local embedding initialization 避免重複 model loads。

**相關 PR**：#15639

---

## 11. 模型與提供者

### 11.1 新模型支援

| 模型 | 說明 |
|------|------|
| OpenAI GPT-5.4 | API + Codex OAuth，1M context |
| Gemini 3.1 Flash-Lite | 前向相容 |
| Venice kimi-k2-5 | 預設模型切換 |
| Vercel AI Gateway | catalog 自動發現 |
| openai-codex | implicit provider + baseUrl scope |

---

### 11.2 提供者修復

| 提供者 | 修復 | PR |
|--------|------|-----|
| Kimi Coding | Anthropic tool schema 正規化 | #40008 |
| Kimi Coding | thinking signatures 保留策略 | #39798 |
| Venice | 發現限制和工具支援強化 | #38306 |
| Kilocode | 所有模型可用 | #32352 |
| MiniMax | portal coding plan VLM routing | #33953 |
| Ollama | compaction/summarization 自訂 API | #39332 |
| Ollama | local limits 和 native thinking fallback 保留 | #39292 |
| Ollama | provider headers 傳遞至 stream function | #24285 |
| OpenAI | usage streaming chunks 在 non-native 上停用 | — |
| OpenAI Codex | OAuth scopes 修復 | #24720 |
| OpenAI Codex | OAuth login/refresh 路徑強化 | — |
| GPT/Gemini | alias defaults 重新整理 | #38638 |
| Custom Provider | headers 傳播至 model objects | #27490 |
| General | HTTP 499 加入 transient error codes | #41468 |
| General | zhipuai 1310 Weekly/Monthly Limit failover | #33813 |

---

### 11.3 Transcript Policy

- Anthropic signature-excluded providers 使用 named Set
- `preserveSignatures` 設定為 `isAnthropic` 在 `resolveTranscriptPolicy` 中

---

## 12. Plugin SDK 與擴充系統

### 12.1 Scoped Subpath Wiring

Plugin SDK 新增完整的 bundled subpath wiring，提供如下 scoped import paths：

```typescript
import { ... } from "openclaw/plugin-sdk/channel"
import { ... } from "openclaw/plugin-sdk/allowlist"
import { ... } from "openclaw/plugin-sdk/resolve-target"
// 等等
```

Root alias shim 維持向下相容，但 lazily load。

---

### 12.2 Context Engine 外掛

新增 `context-engine` manifest kind，允許外掛註冊自訂 context management：

```json5
{
  kind: "context-engine",
  // ...
}
```

**相關 PR**：#22201

---

### 12.3 Model Auth API

Context-engine plugins 可透過 `api.runtime.modelAuth` 存取 model authentication API。

**相關 PR**：#41090

---

### 12.4 Prompt Hook Policy

Plugin prompt hooks 加入 runtime policy 驗證，確保 hooks 符合安全策略。

**相關 PR**：#36567

---

### 12.5 registerHttpHandler 遷移錯誤改善

`registerHttpHandler` 的遷移錯誤訊息更加清晰，指引使用者切換至 `registerHttpRoute`。

**相關 PR**：#36794

---

### 12.6 Plugin 安裝後清除快取

Plugin 安裝後自動清除 discovery cache。

**相關 PR**：#39752

---

### 12.7 Integrity Drift 偵測

Unpinned 更新時避免 false integrity drift 提示。

**相關 PR**：#37179

---

## 13. 平台相容性

### 13.1 macOS

| 改進項目 | 說明 | PR |
|---------|------|-----|
| Universal binaries | release 預設 universal binaries | #33891 |
| Tailscale 發現 | 改善 gateway discovery | #40167 |
| 遠端 token 欄位 | 遠端模式 gateway token | — |
| VoiceWake 修復 | OverlayController exclusivity violation | #39275 |
| XPC 偵測 | launchd 透過 XPC_SERVICE_NAME 偵測 | #20555 |
| LaunchAgent 強化 | install permissions 強化 | — |
| Speech hint | Talk Mode mic capture taskHint | — |
| 自我重啟修復 | launchd 移除 self-kickstart | — |
| Canvas 自動載入 | iOS 自動載入 scoped gateway canvas | #40282 |
| Foreground resume | iOS 前景恢復時安全重播佇列 actions | #40281 |
| iOS gateway reconnect | 前景返回時重連 | #41384 |
| iOS quick setup 跳過 | 已配置 gateway 時跳過 | #38964 |

---

### 13.2 Android

| 改進項目 | 說明 | PR |
|---------|------|-----|
| 套件重新命名 | 改為 ai.openclaw.app | #38712 |
| 移除前台服務 | 麥克風和螢幕（Play policy） | — |
| 移除背景定位 | 移除 background location mode | — |
| 移除自我更新 | 移除 self-update install flow | — |
| Legacy 遷移 | 持久化 legacy location mode 遷移 | — |
| 應用圖示更新 | 新版 app icon | — |
| Run command 對齊 | 對齊 app id | — |

---

### 13.3 Windows

| 改進項目 | 說明 | PR |
|---------|------|-----|
| 排程任務重啟 | 透過 Scheduled Task 重啟 gateway | #38825 |
| 語系無關偵測 | locale-invariant schtasks 偵測 | #39076 |
| PATH 凍結修復 | 避免凍結 PATH 在 task scripts | #39139 |
| icacls ACL | locale-independent ACL audit | #38900 |

---

### 13.4 Linux

| 改進項目 | 說明 | PR |
|---------|------|-----|
| systemd 單元檢查 | is-enabled 前先檢查 managed unit | #38819 |
| exit 4 處理 | is-enabled exit 4 (not-found) 處理 | #33634 |
| User bus fallback | systemd user bus env 缺失 fallback | #34884 |
| 重啟超時修復 | Debian/systemd false timeouts 修復 | #34874 |
| WSL2 systemctl | 強化 WSL2 systemctl install checks | #39294 |
| Degraded status | 處理 degraded systemd status checks | #39325 |

---

### 13.5 Docker / Podman

| 改進項目 | 說明 | PR |
|---------|------|-----|
| Build cache | 改善 build cache reuse | #40351 |
| Slim image | 精簡版 runtime image | #38479 |
| 可選擴充 | OPENCLAW_EXTENSIONS build arg | #32223 |
| Podman cwd | TMPDIR 避免 cwd 權限錯誤 | #39435 |
| Podman bind | OPENCLAW_GATEWAY_BIND env-file override | #38785 |
| SELinux | :Z mount option | #39449 |
| Multi-arch | pin multi-arch docker base digests | — |

---

## 14. 效能優化與重構

### 14.1 大規模重構

本次更新包含約 **200+ 個重構提交**，主要方向：

- **共用提取**：跨頻道的 allowlist resolver、config adapter、group policy、onboarding scaffolding、DM pairing 等統一提取為共用模組
- **測試去重**：大量測試 fixture、scaffolding、builder 的去重複化（agents、browser、cli、gateway、channels 各模組）
- **Plugin SDK scoped imports**：全部 30+ 個官方外掛遷移至 scoped subpath imports
- **Gateway auth**：統一 auth resolution 跨 systemd、Discord、node host
- **Models**：拆分 provider discovery helpers、list row builders、models.json planning
- **Cron**：提取 delivery tool policy、agent defaults merge、schedule resolver
- **Telegram/Discord/Slack**：拆分 lane delivery、native command、polling session 等模組

---

### 14.2 效能改善

| 項目 | 說明 |
|------|------|
| Chunking 防護 | 防止 quadratic scans |
| Models.json 原子寫入 | 原子化 publish |
| Bundled channel plugins 優先 | 避免 npm 重複 |
| Context engine registry 共享 | 跨 bundled chunks |
| Config schema cache key | 防止 RangeError |

---

## 15. 測試與 CI

### 15.1 CI 改善

| 項目 | 說明 |
|------|------|
| CodeQL | 新增 CodeQL workflow（JavaScript scope） |
| Swift 6.2 | CodeQL 使用 Swift 6.2 toolchain |
| Knip | 新增 dead-code 分析（report-only） |
| Secret scan | scope 至 changed files，main scan 恢復 |
| Docker cache | GHA cache 改善 |
| Windows cache | pnpm + Python stores 快取 |
| Shallow checkout | scope checkouts 最佳化 |
| Install smoke | Docker smoke 建置快取 |

---

### 15.2 detect-secrets

大量（20+ 個）detect-secrets baseline refresh 和 allowlist 調整提交，確保 CI secret scan 穩定。

---

### 15.3 測試穩定性

- 大量測試 fixture 去重和正規化
- Windows CI 相容性修復
- Timer、timeout、concurrency 測試穩定化
- Node 24+ test runner 相容性

---

## 16. 文件更新

| 文件主題 | 說明 |
|---------|------|
| Context Engine | 完整的 context engine 外掛文件 |
| Browserbase | 遠端 CDP 選項指南 |
| Brave Search API | Feb 2026 plan restructuring 更新 |
| WSL2 remote Chrome | CDP 故障排除指南 |
| Memory 指令 | 記憶指令範例改善 |
| Trusted operator | control surfaces 澄清 |
| Web tools | MDX 連結修復 |
| Changelog 管理 | 格式、attribution、placement 規範 |
| Security | agent owner trust defaults 澄清 |
| PR workflow | clickable commit SHAs、bot review ownership |
| Gateway config | Slack、TTS、heartbeat、cron 更新 |
| Control UI | locale 支援文件 |
| Telegram DM | single-user allowlist 建議 |
| Mac release | arch defaults、notarization 流程澄清 |

---

## 17. 摘要統計

| 指標 | 數值 |
|------|------|
| 總提交數 | **1,120** |
| 影響檔案數 | **2,778** |
| 新增行數 | 151,485 |
| 刪除行數 | 30,799 |
| 版本跨度 | `2026.3.3` → `2026.3.9` |
| 新功能 | **30+ 項** |
| 安全修復 | **18+ 項** |
| 頻道修復 | **60+ 項** |
| 重構提交 | **~200+** |
| 測試/CI 提交 | **~100+** |

### 按模組分布

| 模組 | 變更強度 |
|------|---------|
| Plugin SDK (Scoped Imports) | ■■■■■ 極高 |
| 重構 / 去重 | ■■■■■ 極高 |
| Telegram | ■■■■ 高 |
| Agent / Compaction | ■■■■ 高 |
| 安全 (Security) | ■■■■ 高 |
| Gateway / CLI | ■■■■ 高 |
| Feishu / Lark | ■■■■ 高 |
| Discord | ■■■ 中 |
| ACP | ■■■ 中 |
| Cron | ■■■ 中 |
| Browser | ■■■ 中 |
| Models / Providers | ■■■ 中 |
| 記憶 (Memory) | ■■ 中 |
| Slack | ■■ 中 |
| Mattermost | ■■ 中 |
| macOS / iOS | ■■ 中 |
| Android | ■■ 低 |
| Windows | ■■ 低 |
| Docker / Podman | ■■ 低 |
| Linux / systemd | ■ 低 |

---

*本文件由 AI 自動分析生成，分析基準點：`3d508ce01b` → `91b38f7a7`（2026-03-10）*
