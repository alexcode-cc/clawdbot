# OpenClaw 提交分析報告 — 2026-05-07

> **分析範圍**：以上一輪繁體文件對齊點 **release `v2026.5.5`** 為起點，分析 **`v2026.5.5..v2026.5.7`** 的新增／更新提交，並以英文 **`CHANGELOG.md` → `## 2026.5.6`** 與 **`## 2026.5.7`** 作為產品行為主軸。
> **版本對齊**：`package.json` 目前為 **`2026.5.7`**；`git tag v2026.5.7` 指向 **`eeef4864494f859838fec1586bedbab1f8fa5702`**。
> **前一輪文件**：`docs-cht/commit-analyze-20260505.md` 已對齊 **`v2026.5.5`**。
> **提交量**：`v2026.5.5..v2026.5.7` 共 **93** 筆提交，且 **93** 筆皆為非合併提交；其中 **`v2026.5.5..v2026.5.6`** 為 **17** 筆，**`v2026.5.6..v2026.5.7`** 為 **76** 筆；檔案差異約 **372** 個檔案，約 **9,668** 行新增、**1,163** 行刪除。

## 目錄

1. [版本切點與發行脈絡](#1-版本切點與發行脈絡)
2. [2026.5.6 — 小型回補修復](#2-202656--小型回補修復)
3. [2026.5.7 — 主要修復主題](#3-202657--主要修復主題)
4. [受影響文件與程式碼面](#4-受影響文件與程式碼面)
5. [繁體技術文件同步策略](#5-繁體技術文件同步策略)
6. [本輪對齊統計](#6-本輪對齊統計)

---

## 1. 版本切點與發行脈絡

```text
v2026.5.5
  → v2026.5.6
  → 2026.5.7 beta 1
  → v2026.5.7
```

本輪不是單一大功能版，而是 **兩個連續穩定化 release**。`2026.5.6` 是小型修補版，重點是撤回／限制會影響 Codex/OpenAI config 的 doctor rewrite，並修復 runtime fetch／web fetch timeout。`2026.5.7` 則擴大到 release/plugin publishing、Cron/Channels CLI、通道投遞、Agent context/compaction、Codex approvals、Provider transcript repair、插件安裝與外部通道 setup。

目前 checkout 的 `git describe` 顯示在 `v2026.5.7` 之後仍有後續提交，但本次使用者指定「上次更新到 release v2026.5.5」且 `package.json`／`CHANGELOG` 對齊 **`2026.5.7`**，因此本文件以 **`v2026.5.5..v2026.5.7`** 作為主分析邊界。

## 2. 2026.5.6 — 小型回補修復

`2026.5.6` 主要是防止 2026.5.5 附近的修復過度影響既有設定：

- **Doctor/OpenAI config**：保持 release branch 不帶 legacy Codex route rewrite，避免 `doctor --fix` 改寫仍有效的 OpenAI model config；只有支援的 repair path 才動設定。
- **Plugins/runtime fetch**：在 native `fetch` 或 `Headers` 前移除第三方 symbol metadata，避免 SDK／guarded/proxy fetch path 拒絕原本有效的插件請求。
- **Debug proxy**：replay captured fetch request 前正規化 header dictionaries，避免 caller-owned header object 的 symbol metadata 讓 proxy fetch 失敗。
- **Web fetch timeout**：request timeout 後 bounded cleanup guarded dispatcher，讓 timed-out fetch 回工具錯誤，而不是留下 Gateway tool lane active。

## 3. 2026.5.7 — 主要修復主題

### 3.1 Release／插件發布與供應鏈

- **ClawHub 發布修復**：release/plugin publishing 會重試暫時性 ClawHub CLI dependency install 失敗；單一 preview cell flake 不阻止已通過 preview 的插件發布；發布後驗證每個預期 ClawHub package version。
- **Managed npm lifecycle shell**：managed plugin install、rollback、repair、uninstall 使用與 staged package updates 相同的 absolute POSIX npm lifecycle shell，避免受限 PATH shell 破壞 cleanup。
- **外部通道 setup runtime**：非 bundled external plugin setup entries 會 forward `setChannelRuntime`，確保 deferred external channel runtime initializer 在 startup polling 前安裝。
- **managed npm uninstall**：plugin package 目錄已缺失時仍執行 managed npm cleanup，避免 stale package manifests 在下一次 npm install 復活已移除插件。

### 3.2 CLI／Cron／Channels／Tasks

- **Cron JSON status**：`cron list --json` 與 `cron show --json` 輸出 computed `status`，外部工具可直接讀取 disabled/running/ok/error/skipped/idle，不必重作 cron 狀態推導。
- **Channels CLI 收斂**：`openclaw channels list` 改成 channel-only；新增 `--all` 顯示 bundled/catalog channels；輸出 installed/configured/enabled state；model auth／usage 移到 `models auth list`、`status`、`models list`。
- **Cron doctor repair**：`openclaw doctor --fix` 會修復 persisted cron jobs 中 `payload.model` 為 `"default"`、`"null"`、空字串或 JSON `null` 的壞 override，同時維持 runtime model validation 嚴格。
- **Cron isolated delivery**：`delivery.channel=last` 沒有 previous route 時，implicit announce delivery 在 model execution 前即失敗，避免 recurring job 先花 token 才遇到永久交付目標錯誤。
- **Tasks reload blocker**：Gateway task reconciliation 會清理 stale CLI run-context tasks 與 channel hot-reload deferrals，避免 stale task records 永久阻塞 Discord／Slack／Telegram reload。

### 3.3 Agent／Context／Compaction／Skills

- **Context engine cache invalidation**：source history shrink 或 assembly failure 時會 invalid cached assembled context views，避免 reset 後仍沿用 stale pre-reset history。
- **Compaction reserve clamp**：compaction summary reserve tokens 會 clamp 到各模型 output limit，高上下文 compaction 不再要求無效 `max_tokens`。
- **Skills snapshot reset**：`/new` 與 `sessions.reset` 清掉 cached skills snapshots，讓長生命週期 channel session 在 skills 變更後重建可見 skill list。
- **Inline skill authorization**：auto-reply inline skill tool dispatch 會經 before-tool-call authorization hooks gate，不再繞過工具授權。
- **Subagent registry retention**：completed session-mode subagent registry rows 尊重 `agents.defaults.subagents.archiveAfterMinutes`，不再硬寫 5 分鐘 TTL。

### 3.4 通道與訊息交付

- **Discord target parsing**：`discord:channel:<id>` 這類 provider-prefixed target 會解析成 channel send，而不是 legacy Discord DM target，避免 cross-channel `message.send` 誤報 `Unknown Channel`。
- **Discord voice**：`channels capabilities` 與 `channels status --probe` 會稽核 voice-channel 權限，包含 auto-join target；voice capture 預設 post-speech silence grace 延到 2.5s，並可用 `voice.captureSilenceGraceMs` 調整。
- **Telegram allowlist／polling**：Telegram DMs、groups、native commands、callback authorization 會先遵守 `accessGroup:*` sender allowlists；polling watchdog 綁定 `getUpdates` liveness，避免 unrelated Bot API outbound calls 掩蓋 wedged inbound poller。
- **Telegram delivery fallback**：inbound Telegram turn 中同 chat 的 `message` tool outbound send 成功時，會視為已交付，避免再送 rewritten silent reply fallback。
- **WhatsApp LID／MEDIA**：proactive phone-number sends 會用 Baileys LID forward mappings；captioned `MEDIA:` auto-replies 只送一次，不再先送空 media 再送 captioned media。
- **Agent delivery truthfulness**：outbound delivery 沒有 adapter result 時回報 `deliverySucceeded=false`，避免 claimed/empty delivery path 假裝成功。

### 3.5 Codex／Approvals／Doctor

- **Codex native approvals**：Codex approval modes 預設不安裝 pre-guardian native `PermissionRequest` hook，讓 Codex reviewer 先核准安全指令；相同 native `PermissionRequest` payload 的 `allow-always` decision 會在 active session window 記住。
- **Plugin approval rendering**：plugin approval requests 會驗證並渲染實際 allowed decisions，避免 Telegram 等 native approval UI 顯示 stale actions。
- **Doctor/Codex OAuth**：`doctor --fix` 會保留可用的 `openai-codex/*` PI routes，並在只有 Codex OAuth auth 可用時恢復 2026.5.5 rewrite 過的 `openai/*` GPT-5 routes。
- **Codex workspace bootstrap**：OpenClaw workspace bootstrap block 透過 Codex `developerInstructions` 送入 app-server，而不是 `config.instructions`，讓 persona/style guidance 進入行為塑形通道。
- **PDF/Codex**：`openai-codex/*` PDF tool request 補 extraction-fallback instructions，使 Codex Responses 收到必要 system prompt。

### 3.6 Provider／模型／工具

- **OpenAI `chat-latest`**：支援 `openai/chat-latest` 作為 explicit direct API-key model override，可試用 moving ChatGPT Instant API alias 而不改 stable default model。
- **Tavily SecretRef**：`tavily_search` 與 `tavily_extract` 從 active runtime config snapshot 解析 credential，避免 exec SecretRef-backed API keys 未解析就到工具。
- **Provider transcript repair**：APNG sniffed PNG upload 正規化；Gemini 3 tool-call thought-signature replay 保留 fallback signatures；接受 legacy `__env__:VAR` custom-provider keys；修復 snake_case tool-call transcript sanitization。
- **Telegram models callback**：`/models` callback buttons 可解析含 dot 的 provider id，例如 `hf.co` model list inline keyboard。
- **Anthropic／OpenAI edge cases**：release 範圍內也補強 uppercase dynamic model id 拒絕、embedding output dimensions、HEIC model-run file normalization、OpenRouter auto selector refs。

### 3.7 安全與權限

- **Native command owner enforcement**：native command handlers 會遵守 owner enforcement。
- **Active Memory global toggles**：global memory toggles 需要 admin scope。
- **BTW usage placeholder**：`/btw` 缺問題時的 usage placeholder 加上括號，避免 outbound channel sanitization 把提示吃掉。
- **Slack startup allowlist**：Slack startup user allowlist resolution 增加 gate，降低啟動期間錯誤授權風險。

## 4. 受影響文件與程式碼面

本輪差異橫跨約 **372** 個檔案，主要集中在：

- **英文文件**：`docs/automation/cron-jobs.md`、`docs/automation/tasks.md`、`docs/channels/discord.md`、`docs/cli/channels.md`、`docs/cli/cron.md`、`docs/cli/doctor.md`、`docs/cli/tasks.md`、`docs/plugins/codex-harness.md`、`docs/providers/openai.md`、`docs/tools/web.md`。
- **通道插件**：`extensions/discord/**`、`extensions/telegram/**`、`extensions/whatsapp/**`、`extensions/tavily/**`、`extensions/active-memory/**`。
- **Agent 與 runtime**：`src/agents/**`、`src/auto-reply/**`、`src/gateway/server-methods/agent.ts`、`src/gateway/session-reset-service.ts`。
- **CLI／commands／cron**：`src/commands/channels/list.ts`、`src/commands/doctor-cron*`、`src/commands/tasks.ts`、`src/cron/service/**`、`src/cli/cron-cli/**`。
- **插件與 npm 管理**：`src/plugins/**`、`src/infra/npm-managed-root.ts`、`src/infra/npm-install-env.ts`、release publishing scripts。
- **網路與 fetch**：`src/infra/fetch-headers.ts`、`src/infra/net/fetch-guard.ts`、`src/infra/net/proxy-fetch.ts`、`src/proxy-capture/runtime.ts`。

## 5. 繁體技術文件同步策略

本次已將繁體技術文件的主版本標記由 **2026.5.5** 推進到 **2026.5.7**，並加入以下對齊補充：

- **專案概覽**：新增 2026.5.6／2026.5.7 release 摘要與本輪分析入口。
- **Gateway**：補充 web fetch timeout cleanup、optional plugin startup warning、task reload blocker、session rollover transcript persistence。
- **Agent**：補充 context cache invalidation、compaction reserve clamp、skills snapshot reset、inline skill authorization、subagent retention。
- **通訊頻道**：補充 Discord target／voice audit、Telegram allowlist／polling、WhatsApp LID／MEDIA dedupe、agent delivery success semantics。
- **插件開發**：補充 ClawHub publish hardening、managed npm lifecycle shell、external setup runtime forwarding、runtime fetch header normalization。
- **應用程式**：補充 iOS/macOS version sync 與 Control UI／TUI 受 session reset／skills snapshot 修復的行為。
- **配置系統**：補充 cron bad model sentinel repair、optional plugin config warning、Tavily SecretRef runtime snapshot、Active Memory admin scope。
- **CLI／Onboard**：補充 `channels list --all`、Cron JSON status、doctor cron repair、externalized channel plugin stale config recovery。
- **大模型**：補充 `openai/chat-latest`、Codex OAuth doctor route preservation、provider transcript repair、Gemini thought signatures、APNG/PNG handling。
- **長期記憶／TUI**：補充 Active Memory global toggles admin scope、skills snapshot reset、BTW placeholder、session reset 與 TUI 顯示關聯。

## 6. 本輪對齊統計

| 項目 | 結果 |
|------|------|
| **產品行為權威來源** | **`CHANGELOG.md` → `## 2026.5.6` + `## 2026.5.7`** |
| **`package.json` version** | **`2026.5.7`** |
| **起點 tag** | **`v2026.5.5`** → **`b1abf9d8ae`** |
| **中間 tag** | **`v2026.5.6`** → **`c97b9f79ec`** |
| **終點 tag** | **`v2026.5.7`** → **`eeef486449`** |
| **提交範圍** | **`v2026.5.5..v2026.5.7`** |
| **提交數** | **93** |
| **非合併提交數** | **93** |
| **檔案差異** | 約 **372** 個檔案 |
| **主要變更類型** | Release/plugin publishing、Cron/Channels CLI、通道投遞、Agent context/compaction、Codex approvals、Provider transcript repair、fetch/network cleanup |
| **本輪繁體文件主版本** | **`2026.5.7`** |

*若本地 checkout 或 tag 拓撲不同，請以實際 `git describe`、`git rev-parse "v2026.5.7^{}"` 與 `CHANGELOG.md` 頂段為準修正表頭。*
