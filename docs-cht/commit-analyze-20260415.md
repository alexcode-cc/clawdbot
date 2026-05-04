# OpenClaw 提交分析報告 — 2026-04-15

> **分析範圍**：從 `30eb0ec085a41cec4f07cca7caa43827e49d02d8`（doc: 更新繁體中文文件版本號至 2026.4.14）至最新提交（`041266a669`，chore: prepare 2026.4.15 release），共 **417 個非合併提交**，涵蓋 2026-04-14 至 2026-04-15。

---

## 目錄

1. [版本發布](#1-版本發布)
2. [重大新功能](#2-重大新功能)
3. [安全修復](#3-安全修復)
4. [Dreaming / Memory 重大改善](#4-dreaming--memory-重大改善)
5. [Agent 與 Context Engine](#5-agent-與-context-engine)
6. [Codex / OpenAI 改善](#6-codex--openai-改善)
7. [頻道修復](#7-頻道修復)
8. [Gateway 改善](#8-gateway-改善)
9. [Control UI 改善](#9-control-ui-改善)
10. [效能優化](#10-效能優化)
11. [Plugin SDK 與架構](#11-plugin-sdk-與架構)
12. [QA / Matrix Live Lane](#12-qa--matrix-live-lane)
13. [CI / 建置改善](#13-ci--建置改善)
14. [文件更新](#14-文件更新)
15. [摘要統計](#15-摘要統計)

---

## 1. 版本發布

```
cb790c858b build(release): bump core app versions to 2026.4.15-beta.1
c635efd233 chore: prepare 2026.4.15-beta.2 release
041266a669 chore: prepare 2026.4.15 release
```

版本路徑：`2026.4.14` → `2026.4.15-beta.1` → `2026.4.15-beta.2` → **`2026.4.15`（穩定版）**

---

## 2. 重大新功能

### Anthropic 預設模型升級至 Opus 4.7

```
628b454eff feat: default Anthropic to Opus 4.7
```

**Anthropic 所有預設選擇全面升級至 Claude Opus 4.7**，包括：
- `opus` 別名
- Claude CLI 預設
- 捆綁影像理解（bundled image understanding）

這是本版本最重要的新功能之一，使用者無需手動更改配置即可使用最新的 Opus 4.7 模型。

### Google Gemini TTS

```
bf59917cd1 fix: add Google Gemini TTS provider (#67515) (thanks @barronlroth)
```

新增 **Google Gemini TTS（文字轉語音）**支援：
- 透過捆綁的 `google` 插件提供
- 支援聲音選擇
- WAV 回覆輸出
- PCM 電話語音輸出
- 包含設定和文件指引

### Local Model Lean Mode（實驗性）

```
4015138df9 Agents: add lean local model mode (#66495)
f09a4d9ba0 fix(agents): move lean local-model mode behind experimental flag
```

新增 **`agents.defaults.experimental.localModelLean: true`** 設定：
- 為較弱的本地模型設定移除重量級預設工具（如 `browser`、`cron`、`message`）
- 減少 prompt 大小，不影響正常路徑
- 目前為實驗性功能，已移至 `experimental` 旗標下

### Memory LanceDB 雲端儲存

```
df918c4de5 feat(memory-lancedb): add cloud storage support to memory-lancedb (#63502)
```

`memory-lancedb` 插件新增雲端儲存支援，持久化記憶索引可在遠端物件儲存（S3/GCS 等）而非僅限本地磁碟運行。

### GitHub Copilot 記憶搜尋 Embedding

```
88d3620a85 feat(github-copilot): add embedding provider for memory search (#61718)
```

GitHub Copilot 新增 embedding provider 用於記憶搜尋，並公開專用的 Copilot embedding host helper，讓插件可以重用 transport 同時遵守遠端覆寫、token 刷新和更安全的 payload 驗證。

### Model Auth Status 卡片（Control UI）

```
507b718917 feat(ui): add Model Auth status card to Overview dashboard (#66211)
```

Control UI Overview 新增 **Model Auth 狀態卡片**：
- 顯示 OAuth token 健康狀態和 provider rate-limit 壓力
- 當 OAuth token 即將過期或已過期時顯示提示
- 由新的 `models.authStatus` gateway 方法支援（快取 60 秒）

### QA Matrix Live Lane

```
82a2db71e8 refactor(qa): split Matrix QA into optional plugin (#66723)
4db162db7f QA: split lab runtime and extend Matrix coverage (#67430)
```

Matrix live QA 拆分為獨立的 `qa-matrix` runner，並進行大量 Matrix 測試覆蓋擴展。

### Skills 快取失效修復（重要）

```
b23d59a522 Gateway/skills: invalidate session skills snapshot on config write
```

重要修復：當配置寫入觸及 `skills.*` 時（例如 `skills.allowBundled`、`skills.entries.<id>.enabled`、`skills.profile`），現在會更新快取的 skills-snapshot 版本。修復了移除工具後 agent 繼續呼叫已停用工具的問題。

---

## 3. 安全修復

### P1 安全強化（7 項修復）

```
f624b1d246 fix(security): 7 P1 hardening fixes — scan-paths, windows-acl, audit-extra (#67003)
```

7 項 P1 安全強化修復，涵蓋 scan-paths、Windows ACL 和額外的 audit 項目。

### Compaction 憑證洩漏修復

```
8b7d76bfbb fix(compaction): stop retaining credential-like values (#67801)
```

修復 compaction（壓縮）保留憑證類似值的問題，防止 API keys、bearer tokens 等敏感資料在 compaction 後被保留在 context 中。

### Exec Approval 提示詞遮蔽

```
e99a24d645 fix(security): redact secrets in exec approval prompts (#61077) (#64790)
```

在 exec approval 提示詞中遮蔽 secrets，確保 inline approval review 不會在聊天頻道中洩漏憑證資料。

### Matrix 安全

```
f8705f512b fix(matrix): block DM pairing-store entries from authorizing room control commands [AI-assisted] (#67294)
450c3a8ed2 fix(security): include Matrix avatar params in sandbox media normalization + preserve mxc:// URLs + log gmail watcher stop failures [AI-assisted] (#64701)
```

- 阻止 DM pairing-store 條目授權 room control commands
- Matrix avatar params 納入沙箱媒體正規化；保留 `mxc://` URL

### Webchat 安全

```
1470de5d3e fix(webchat): reject remote-host file:// URLs in media embedding path [AI-assisted] (#67293)
6e58f1f9f5 fix(gateway): enforce localRoots containment on webchat audio embedding path [AI-assisted] (#67298)
```

- 拒絕 webchat 媒體嵌入路徑中的遠端 `file://` URL
- 在 webchat 音訊嵌入路徑上強制執行 localRoots 邊界檢查

### Agent 工作區文件安全

```
472bcbbccc fix(agents): tighten workspace file opens (#66636)
```

透過 `fs-safe` helpers 路由 `agents.files.get/set`，拒絕 allowlisted 文件的符號連結別名，修復 `open` 和 `realpath` 之間符號連結置換的競態條件。

### MCP Loopback 強化

```
62430d9f3a Harden MCP loopback request validation (#66665)
```

MCP loopback bearer 比對改用常量時間 `safeEqualSecret`，並拒絕非 loopback 瀏覽器來源請求。

### Gateway HTTP Auth 每請求解析

```
acd4e0a32f fix(gateway): re-resolve HTTP auth per-request to honor credential rotation [AI] (#66651)
```

Gateway bearer 現在每請求重新解析，修復 `secrets.reload` 或 config hot-reload 後舊認證仍可在 HTTP 路徑有效的問題。

### Codex Sandbox 恢復

```
4c66978591 security(codex): restore sandbox protections for resumed CLI sessions
```

恢復 resumed CLI sessions 的沙箱保護。

---

## 4. Dreaming / Memory 重大改善

### Dreaming 儲存模式預設改變

```
8c392f0019 fix(dreaming): default storage.mode to "separate" so phase blocks stop polluting daily memory files (#66412)
```

**重大變更**：Dreaming 儲存模式預設從 `inline` 改為 **`separate`**：
- Dreaming phase blocks（`## Light Sleep`、`## REM Sleep`）現在儲存在 `memory/dreaming/{phase}/YYYY-MM-DD.md`
- 不再注入到 `memory/YYYY-MM-DD.md` 中
- 每日記憶文件不再被大量 phase-block 行主導
- 若需舊行為，可設定 `plugins.entries.memory-core.config.dreaming.storage.mode: "inline"`

### Dreaming 自我攝入阻止

```
0c4e0d7030 memory: block dreaming self-ingestion (#66852)
```

修復普通對話記錄引用 dream-diary prompt 被錯誤分類為內部 dreaming runs 並從 session recall 攝入中靜默丟棄的問題。

### Dreaming 每日桶（dayBucket）修復

```
4de56b18ba fix(dreaming): use ingestion date for dayBucket instead of file date (#67091)
```

使用攝入日期而非來源文件日期作為每日召回去重，修復同一每日筆記的重複掃描無法跨日增加 `dailyCount` 的問題。

### Dreaming Transcript 攝入修復

```
a1b01f0281 fix(memory-core): skip dreaming transcript ingestion via session store (#67315)
```

在 session-store 元資料 bootstrap 記錄到來前跳過 dreaming narrative transcripts，防止 dream diary 提示/散文行污染 session 攝入。

### Session Corpus Metadata 清理

```
82a19b0f4b memory: strip inbound metadata envelopes from user messages in session corpus (#66548)
```

在正規化前從 session-corpus 使用者 turn 中去除 AI 面向的入站 metadata 信封，確保 REM topic 萃取看到使用者的實際訊息文字。

### Memory/QMD 路徑對齊

```
37d5971db3 Align QMD memory reads with canonical memory paths (#66026)
```

QMD 記憶讀取對齊規範記憶路徑，限制 `memory_get` 只能讀取 canonical 記憶文件，不能作為通用工作區文件讀取繞過工具政策。

---

## 5. Agent 與 Context Engine

### Context Engine Prompt Cache 穩定性

```
a327b6750d fix: stabilize context engine prompt cache touches (#67767)
687ede50a5 fix(agents): add prompt cache compatibility opt-out
eb10803691 fix(prompt): keep inbound chat ids out of system prefix
```

- 保持 loop-hook 和 final `afterTurn` prompt-cache touch 元資料與當前 assistant turn 對齊
- 新增 `models.providers.*.models.*.compat.supportsPromptCacheKey` opt-out 設定
- 從穩定系統提示中排除可變的入站 chat IDs，讓 task-scoped adapters 可以跨 runs 重用 prompt cache

### Context Window 收緊

```
4f00b76925 fix(context-window): Tighten context limits and bound memory excerpts (#67277)
```

收緊預設啟動/skills prompt 預算，cap `memory_get` excerpts，保持 QMD 讀取與同等有界 excerpt 合約對齊。

### Skills Prompt Cache 排序修復

```
1d31b1d1df fix: stabilize skills prompt ordering (#64198) (thanks @Bartok9)
a4b94f77b9 fix(skills): sort available_skills alphabetically for prompt cache stability
c4488d5ef5 fix: pin localeCompare to 'en' locale for cross-environment stability
```

修復 `available_skills` 條目因來源合并順序變化而改變 prompt-cache 前綴的問題，現在按技能名稱字母排序，並釘定 `localeCompare` 使用 `'en'` locale 確保跨環境一致性。

### Unknown-Tool Guard 預設啟用

```
36ed36768c Agents/tool-loop: enable unknown-tool stream guard by default
```

**`resolveUnknownToolGuardThreshold`** 現在預設啟用（不再需要明確設定 `tools.loopDetection.enabled: true`）。修復幻覺或已移除的工具（例如從 `skills.allowBundled` 移除的工具）在 embedded-run timeout 前持續觸發「Tool X not found」迴圈的問題。

### TUI 串流 Watchdog

```
36ed36768c TUI/streaming: add watchdog that resets the activity indicator after delta silence
```

在 `tui-event-handlers` 中新增客戶端串流 watchdog，在 30 秒 delta 靜默後自動重置 `streaming · Xm Ys` 活動指示器為 `idle`，防止 TUI 永久卡在 `streaming` 狀態。可透過 `streamingWatchdogMs` 配置（設為 `0` 禁用）。

### Agent Prompt Fallback 保留

```
d74533c718 Docs: remove QA Matrix changelog entry
71f3b8c9c9 fix(agents): preserve original prompt on model fallback retry (#65760) (#66029)
```

在包含 session history 的模型 fallback 重試中保留原始 prompt body，確保重試模型繼續執行 active task 而非只看到通用的 continue 訊息。

### CLI Transcript 持久化

```
b8ef507cc0 fix(agents): persist cli transcript turns
3a3fae0eac fix(agents): normalize cli transcript api field
c1817c62e3 docs(changelog): move cli transcript entry
```

將成功的 CLI-backed turns 持久化到 OpenClaw session transcript，讓 google-gemini-cli 回覆再次出現在 session history 和 Control UI 中。

### Delivery Mirror 去重

```
d842ec4179 fix: tighten delivery mirror dedupe (#67185) (thanks @andyylin)
e95efa4373 fix(sessions): dedupe redundant delivery mirrors
```

修復 Codex-backed turns 重複 visible reply 的問題，只跳過最新 assistant message 具有相同可見文字的 `delivery-mirror` transcript appends。

---

## 6. Codex / OpenAI 改善

### Codex HTML Challenge 處理

```
5262757f9a fix: handle Codex HTML challenge errors (#67704) (thanks @chris-yyau)
59caf03d67 Avoid rescanning HTML challenge pages during error formatting
36dd58ac2a Prevent Codex HTML challenge pages from looking like DNS failures
de129a6530 fix: restrict HTML timeout short-circuit to transient statuses
e588e904a7 fix: classify HTML provider error pages correctly (#67642) (thanks @stainlu)
```

全面修復 Codex/provider HTML 錯誤頁面處理：
- 在傳輸 DNS 分類前先偵測獨立的 Cloudflare/CDN HTML challenge 頁面
- 防止 provider block pages 顯示為本地 DNS 查找失敗
- 在錯誤格式化時不再重新掃描 HTML challenge 頁面
- 只對短暫狀態限制 HTML timeout 短路

### Codex 傳輸元資料正規化

```
90801ba400 fix(openai-codex): normalize stale transport metadata in resolution and discovery (#67635)
```

在 runtime 解析和 discovery/listing 中正規化過期的原生傳輸元資料，修復缺少 `api` 或 `https://chatgpt.com/backend-api/v1` 的 legacy `openai-codex` 行自動恢復為規範 Codex 傳輸。

### Codex Resume 路徑

```
461d0050d9 fix: keep codex resume runs non-interactive (#67666) (thanks @plgonzalezrx8)
4c66978591 security(codex): restore sandbox protections for resumed CLI sessions
```

保持 resumed `codex exec resume` 在安全非交互路徑，同時恢復沙箱保護。

### Codex Desktop User Agent

```
728295c046 Codex: parse Desktop app-server user agents
99dfc1b616 docs: add changelog for Codex Desktop user agents (#64666) (thanks @cyrusaf)
```

解析 Desktop 來源的 app-server user agents（如 `Codex Desktop/0.118.0`），讓版本閘門在 Codex CLI 繼承多詞 originator 時仍能運作。

### Codex Harness 自動啟用

```
69ba924b53 fix(codex): activate harness plugin for forced runtime
86f108401b fix: share agent harness runtime activation (#67474)
```

當 `codex` 被選為 embedded agent harness runtime 時自動啟用 Codex 插件。

### Ollama Chat Model ID 修復

```
6f5459364a fix: restore Ollama chat model IDs (#67457) (thanks @suboss87)
```

從 Ollama chat request model ids 中去除 `ollama/` provider 前綴，修復配置的 refs（如 `ollama/qwen3:14b-q8_0`）向 Ollama API 發送 404 的問題。

### OpenRouter Qwen3

```
e0bf756b50 fix: handle OpenRouter Qwen3 reasoning_details streams (#66905) (thanks @bladin)
```

解析 `reasoning_details` stream deltas 為 thinking content，修復 Qwen3 在 OpenRouter 上 reply 為空的問題。

---

## 7. 頻道修復

### Telegram

```
734bb9c2e7 Telegram/documents: sanitize binary payloads to prevent prompt input inflation (#66877)
3a371a32e2 fix: filter telegram binary caption text (#66663) (thanks @joelnishanth)
b1d03b4057 fix: keep Telegram command sync process-local (#66730) (thanks @nightq)
ff4edd0559 fix: restore Telegram native auto defaults (#66843) (thanks @kashevk0)
```

- **二進制 Caption 過濾**：丟棄入站 Telegram 文字中洩漏的二進制 caption bytes，防止 `.mobi` 或 `.epub` 文件上傳爆炸 prompt token 數量
- **命令同步本地化**：保持 Telegram 命令同步快取 process-local，防止重啟後信任過期的磁碟快取
- **原生 Auto 預設恢復**：恢復 native commands 和 native skills 的插件 registry-backed auto 預設

### Discord

```
78df859e15 fix: strip standalone <function> tool call tags from visible text (#67318) (thanks @joelnishanth)
ef98bcf630 fix(discord): raise carbon slow listener threshold
```

- 去除 assistant 文字中獨立的 Gemma 風格 `<function>...</function>` tool-call payloads，不截斷散文範例或尾部回覆
- 提高 Carbon slow listener 閾值

### Slack

```
d2a219ea44 fix(media): allow host-local CSV and Markdown uploads via Slack (#67047)
bd288e7683 fix(slack): fix slash commands with button arg menu errors
```

- 允許主機本地 CSV 和 Markdown 上傳（當 fallback buffer 確實解碼為文字時）
- 修復 Slash commands 按鈕 arg menu 錯誤

### BlueBubbles

```
77d9fd697f fix(bluebubbles): restore inbound image attachments and accept updated-message events (#67510)
4af7641350 BlueBubbles/catchup: per-message retry cap for wedged messages (#66870) (#67426)
6f1d321aab feat(bluebubbles): replay missed webhook messages after gateway restart (#66857)
58d0c179d7 fix(bluebubbles): dedupe inbound webhooks across restarts (#19176, #12053) (#66816)
```

BlueBubbles 大量改善：
- **入站圖片修復**：在 Node 22+ 上恢復入站圖片附件下載
- **Catchup 重試上限**：新增每訊息重試上限（`catchup.maxFailureRetries`，預設 10），防止格式錯誤的訊息永遠阻塞 catchup cursor
- **Restart Replay**：Gateway 重啟後透過持久化 per-account cursor 重播遺漏的 webhook 訊息
- **跨重啟去重複**：持久化文件支援的 GUID dedupe，防止 BB Server 重啟或重連後重複回覆

### WhatsApp

```
405c63fb32 fix: flush creds queue before reconnect socket open (#67464) (thanks @neeravmakwana)
b878d50e0e fix(whatsapp): write creds.json atomically (#63577)
d86527d8c6 fix(whatsapp): harden Baileys media upload hotfix (#65966)
```

- 重新開啟 socket 前先排空待處理的 per-auth creds save queue，防止重連時 `creds.json` 寫入競態
- 原子寫入 `creds.json`
- 強化 Baileys media upload 熱修復

### Feishu

```
c8003f1b33 Harden Feishu webhook replay guards (#66707)
```

強化 webhook transport 和 card-action replay guards，缺少 `encryptKey` 時 fail closed，拒絕無 key 時的未簽名請求。

### Matrix

```
2bfd808a83 fix(matrix): skip pairing-store reads for room auth (#67325)
b2753fd0de fix(matrix): fix E2EE SSSS bootstrap for passwordless token-auth bots (#66228)
3425823dfb fix(regression): avoid sync startup for matrix status reads
Fix Matrix media alias normalization
```

- 跳過 room 流量的 DM pairing-store 讀取
- 修復 passwordless token-auth bots 的 E2EE SSSS bootstrap
- 修復 Matrix media alias 正規化

### Voice / Audio

```
0c4e0d7030 fix: restore allowPrivateNetwork for self-hosted STT endpoints (#66692)
```

恢復自架 STT 端點的 `models.providers.*.request.allowPrivateNetwork`，修復 v2026.4.14 後私有 STT 端點觸發 SSRF 封鎖的問題。

### Speech/TTS

```
6ea3cddf0d fix: register bundled TTS providers and route overrides correctly (#62846) (thanks @stainlu)
```

自動啟用捆綁的 Microsoft 和 ElevenLabs speech providers，並透過明確或活動的 provider 優先路由 generic TTS 指令 tokens，修復 `[[tts:speed=1.2]]` 覆寫落在錯誤 provider 的問題。

### Cron / Announce

```
16c608e393 fix: harden cron announce NO_REPLY suppression (#65016) (thanks @BKF-Gitty)
fb4395c1fe fix(cron): preserve all fields in announce delivery by removing summarization instruction (#65638)
```

- 保持孤立 cron announce `NO_REPLY` 剝離大小寫不敏感
- 在 announce delivery 中保留所有欄位

---

## 8. Gateway 改善

### Gateway 重啟迴圈修復

```
8c11210fe5 fix(gateway): capture config hash after plugin auto-enable to prevent restart loop (#67557)
```

重要修復：修復 Linux/systemd 上 plugin auto-enable 是唯一啟動配置寫入時觸發虛假 SIGUSR1 重啟迴圈的問題，防止 chokidar 將每次啟動寫入視為外部變更並觸發 reload → restart 循環。

### Gateway Auth 改善

```
acd4e0a32f fix(gateway): re-resolve HTTP auth per-request to honor credential rotation [AI] (#66651)
804fb3a7de fix(configure): re-read config hash after persist to avoid stale-hash race (#64188) (#66528)
```

- Gateway bearer 每請求重新解析，honor credential rotation
- 配置寫入後重讀持久化 config hash，避免過期 hash 競態

### Gateway Compaction 保留 Floor

```
4bc46ccfed fix(gateway): cap compaction reserve floor to context window for small models (#65671)
```

將 compaction 保留 token floor 限制在模型 context window，修復小 context 本地模型（如 16K tokens 的 Ollama）在每次 prompt 上觸發 context-overflow 錯誤或無限 compaction 迴圈的問題。

### Gateway Exec Events 去重複

```
5dcf526a43 fix: dedupe replayed exec.finished node events (#67281)
```

以規範 session key + `runId` 去重複重播的 `exec.finished` node events，防止重複非同步完成重播注入重複完成 turn。

### Skills 快取失效

```
b23d59a522 Gateway/skills: invalidate session skills snapshot on config write
d7f489f85e Gateway/skills: dedupe skills prefix-match + drop dead fallback on log
```

- 配置寫入觸及 `skills.*` 時更新快取版本
- Skills prefix-match 去重複，移除 log 中的死 fallback

---

## 9. Control UI 改善

### Model Auth 狀態卡片

```
507b718917 feat(ui): add Model Auth status card to Overview dashboard (#66211)
f2fdb9d125 models.authStatus: normalize provider ids + tighten env-backed escape hatch (#67253)
```

- Overview dashboard 新增 Model Auth 狀態卡片
- 修復已別名化 providers、具有 auth.profiles 的 env-backed OAuth 和無法解析 env SecretRefs 的假陽性「缺少」警示

### Chat 期間保留使用者訊息

```
7734a40a56 fix(ui): skip chat history reload during active sends to prevent mess (#66997)
```

保留樂觀使用者訊息卡片在 active sends 期間，延遲同 session history reloads 直到 active run 結束。

### Exec Approval Modal 修復

```
053c5b05c1 [Dashboard] Fix exec approval modal overflow for long command content (#67082)
```

約束 exec approval modal 在桌面的 overflow，防止長命令內容將操作按鈕推出視野。

---

## 10. 效能優化

本版本包含大量效能優化，主要集中在 Plugin SDK 共用快取和 CLI 冷路徑：

### Plugin SDK 共用快取

```
8f0628d43b fix(plugin-sdk): route shared stream helpers through one barrel
e121889a9f fix(plugins): share native anthropic replay hooks
135c3848b9 fix(plugin-sdk): share provider stream wrapper composer
fix(plugin-sdk): share anthropic replay hook constants
fix(plugin-sdk): share canonical replay hook families
fix(openai): share responses stream hook family
fix(google): share gemini provider hook bundle
fix(minimax): share provider hook bundle
fix(openai): share responses transport hooks
fix(openai): share responses transport defaults
...
```

多個 Plugin SDK 路徑共用快取輔助工具，減少重複的 jiti 載入開銷。

### CLI 冷路徑優化

```
e31dfa9897 perf(cli): avoid runtime config loads in gateway discover
4c090accd3 perf(cli): avoid eager gateway call config loads
604a5e07d0 perf(cli): lazy-resolve daemon stop fallback port
f8610da4c5 perf(cli): narrow daemon and gateway cold paths
25efa8cf81 perf(cli): lazy-load daemon service runners
4e46488d1b perf(cli): lazy-load gateway registration deps
```

CLI 路徑大量懶載入優化：避免在 gateway discover、gateway call config 等路徑急切載入重量級模組。

### 測試效能

```
8a37bb4ed6 perf: speed up security audit test imports
372c0051ba test: speed up slow import-boundary tests
2285429aa2 test(vitest): cut unit-ui startup overhead
```

多個測試套件效能改善。

### Doctor 效能

```
66e06b50ba perf(doctor): fast-path bundled channel compat migrations
1169dd7039 perf(doctor): skip blocker scans without plugin disablement
3362cccc20 perf(doctor): skip redundant plugin legacy rescans
```

Doctor 命令在無插件停用時跳過阻塞掃描，跳過冗余的 plugin legacy 重掃。

### 其他效能

```
97ee0c6fd3 perf(migrations): trim legacy migration and bind cold paths
perf(tests): avoid bundled channel cold-loads in hot paths
perf(commands): trim sessions cold-path imports
```

---

## 11. Plugin SDK 與架構

### Bundled Plugin Runtime 本地化

```
c727388f93 fix(plugins): localize bundled runtime deps to extensions (#67099)
7c6f2c0a5a Build: prune packaged runtime test cargo (#67275)
```

- 捆綁插件 runtime 依賴本地化到擁有的 extensions
- 裁剪發布的 docs payload 和安裝/package-manager guardrails
- npm release 驗證現在會在打包的 test cargo 重新出現時失敗

### Plugin Source Alias 修復

```
1898b2093f fix(plugin-sdk): widen root alias source candidates
2cab81d9a7 fix(plugins): widen plugin-sdk source alias candidates
6821b8bfaa fix(plugins): widen extension-api source alias candidates
0a9616caa8 fix(matrix): align extension-api source aliases
```

擴寬各種 plugin SDK 來源別名候選，修復 dist facade 覆寫回退問題。

### Setup Entry Contract

```
1558a352f8 fix(plugins): support bundled setup-entry contract in loader (#66261)
```

支援 loader 中的捆綁 setup-entry contract，保留分割插件 + secrets 入口點的 channel secret contract。

### Bundled Channel Metadata 刷新

```
ec7635256b build: refresh bundled channel metadata
```

---

## 12. QA / Matrix Live Lane

```
82a2db71e8 refactor(qa): split Matrix QA into optional plugin (#66723)
0f7c40e508 Matrix: expose E2EE QA verification hooks
21d500a65f test: expose bundled plugin QA test APIs
4db162db7f QA: split lab runtime and extend Matrix coverage (#67430)
963ad1df06 QA: extend Matrix live contract coverage
3aae0fb16d QA: tighten Matrix substrate coverage
778ac4330a QA: finish Matrix P1 harness coverage
5e77cbd9ec QA: add Matrix transport substrate
```

Matrix QA 大幅擴展：
- Matrix live QA 拆分為獨立的 source-linked `qa-matrix` runner
- 保持 repo-private `qa-*` 表面不在打包和發布版本中
- 新增 E2EE QA verification hooks
- 大量 Matrix P1 harness 覆蓋完成

---

## 13. CI / 建置改善

### CI 權限強化

```
01b7516a95 CI: add explicit permissions to all workflow jobs (fixes code-scanning #40-#57) (#67612)
```

為所有 workflow jobs 添加明確權限，修復 code-scanning 安全問題。

### 依賴安全更新

```
fbccc18e74 fix(deps): bump hono to 4.12.14 and @hono/node-server to 1.19.14 (GHSA-458j-xx4x-4375) (#67613)
2c2dc00fb4 fix(deps): bump dompurify to 3.4.0 (#67614)
a848ddaa7e fix(deps): patch follow-redirects vulnerability
```

- Hono 升級至 4.12.14（修復 GHSA-458j-xx4x-4375）
- DOMPurify 升級至 3.4.0
- follow-redirects 安全漏洞修補

### Android 現代化

```
44a6e50fcc Android: modernize WebView and discovery API usage (#67627)
```

Android WebView 和 discovery API 現代化。

### Blacksmith Runner 支援

```
ad9da24317 ci: register blacksmith macos runner labels
42d100c390 fix(ci): move macOS jobs to blacksmith
```

macOS CI jobs 遷移至 Blacksmith runner。

### 其他 CI 修復

```
98c681e033 CI: mount writable Docker cache homes (#67825)
3ae5d95bfd CI: fix live Docker auth mounts (#67812)
fc137ec5e3 CI: fix live docker vite temp overlay
51606e9889 CI: fix release-check caller permissions (#67787)
f6eb671d62 docs: cap release e2e lanes
```

---

## 14. 文件更新

### Plugin SDK API Baseline

```
015d7ca0a0 docs: update plugin sdk api baseline
277885f0a4 build: refresh plugin sdk api baseline
```

### Experimental Features 頁面

```
a780151fd1 docs: add experimental-features page and de-experimentalize dreaming
```

新增 experimental-features 說明頁，並將 dreaming 從 experimental 狀態移出。

### Active Memory 速度建議

```
b10ae0bf13 fix(docs): add active memory speed recommendations
```

新增 Active Memory 速度建議文件。

### Showcase 頁面現代化

```
dae060390b docs: modernize showcase page (#48493) (thanks @jchopard69)
```

### Gateway Protocol 文件修正

```
489404d75e docs(gateway): correct protocol.md schema path, hello-ok example, auth precedence, and add client constants table (#67372)
```

### QQ 文件

```
dd90297dfc doc:add qq support to README (#67039)
```

### Contributor 更新

```
dafc71c913 Update contributor details for Josh Lehman (#67824)
0aea99883c Add Mason Huang as maintainer (#66974)
```

---

## 15. 摘要統計

### 提交分類

| 類別 | 數量 | 百分比 |
|------|------|--------|
| fix | 200+ | ~48% |
| perf | 50+ | ~12% |
| test | 60+ | ~14% |
| feat | 15 | ~4% |
| docs/chore/build | 40+ | ~10% |
| CI | 30+ | ~7% |
| refactor | 15+ | ~4% |
| 其他 | 7+ | ~2% |

### 版本里程碑

| 版本 | 日期 | 性質 |
|------|------|------|
| 2026.4.14 | 2026-04-14 | Stable（起始點） |
| 2026.4.15-beta.1 | 2026-04-14 | Beta |
| 2026.4.15-beta.2 | 2026-04-15 | Beta |
| 2026.4.15 | 2026-04-15 | Stable |

### 社群貢獻

| 貢獻者 | 主要貢獻 |
|--------|---------|
| @barronlroth | Google Gemini TTS |
| @omarshahine | BlueBubbles 大量修復 |
| @stainlu | 多個 failover/TTS/replay 修復 |
| @xantorres | Skills snapshot invalidation + unknown-tool guard + TUI watchdog |
| @Bartok9 | Skills prompt cache 排序 |
| @chris-yyau | Codex HTML challenge 處理 |
| @andyylin | Delivery mirror 去重複 |
| @plgonzalezrx8 | Codex resume 路徑 |
| @ImLukeF | Lean local model mode |
| @rugvedS07 | Memory LanceDB 雲端儲存 |
| @feiskyer | GitHub Copilot embeddings |
| @suboss87 | Ollama chat model ID 修復 |
| @joelnishanth | Discord/Telegram 文字清理 |
| @neeravmakwana | WhatsApp creds, approvals |
| @jalehman | Context engine, dreaming |

### 關鍵技術趨勢

1. **Anthropic Opus 4.7 作為新預設**：全面升級，使用者零配置受益
2. **Google Gemini TTS**：新增語音合成能力
3. **Dreaming 儲存模式改革**：`separate` 模式避免每日記憶文件被污染
4. **Skills Snapshot 失效修復**：解決 disabled skills 仍被模型呼叫的重要問題
5. **Unknown-Tool Guard 預設啟用**：防止工具 loop 不再需要明確配置
6. **TUI Streaming Watchdog**：防止 TUI 永久卡在 streaming 狀態
7. **Memory/Context 收緊**：bounded excerpts 和更小的 prompt 預算
8. **Prompt Cache 穩定性**：skills 排序、chat ID 隔離、context engine 元資料對齊
9. **Plugin SDK 共用快取**：大量去重複的 jiti 載入路徑
10. **BlueBubbles 大修**：入站圖片、catchup 重試上限、restart replay、跨重啟去重複
11. **依賴安全更新**：hono、DOMPurify、follow-redirects
12. **CI 權限強化**：所有 workflow jobs 明確權限
