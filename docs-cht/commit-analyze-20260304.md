# docs-cht 更新分析記錄 (2026-03-04)

## 更新基準

- **上次更新 commit**: `5190e8536cc06dc10929e274ec99aed4daddd6bd`
- **本次更新 commit**: `5a7ea87af` (Merge branch 'develop' into cursor)
- **版本變更**: `2026.3.2` → `2026.3.3`

## 主要變更摘要

### 新功能
1. **Perplexity Search API** — web_search 從舊版切換到原生 Search API，支援語言/地區/時間篩選
2. **Telegram per-topic agent routing** — forum groups 和 DM topics 支援個別 agentId 覆寫
3. **Slack typingReaction** — Socket Mode DM 反應表情處理狀態
4. **Discord allowBots** — `allowBots: "mentions"` 僅接受提及的 bot 訊息
5. **Plugin SDK channel subpaths** — core/telegram/discord/slack/signal/imessage/whatsapp/line
6. **SecretRef 擴展** — 64 個憑證目標完整覆蓋
7. **工具迴圈偵測** — `tools.loopDetection` 防重複工具呼叫
8. **工具結果截斷** — head+tail 截斷保留尾部診斷
9. **Bootstrap 截斷警告** — `bootstrapPromptTruncationWarning` (off|once|always)
10. **記憶搜尋 Ollama 嵌入** — `memorySearch.provider = "ollama"`
11. **Plugin Runtime APIs** — system.requestHeartbeatNow, events.onAgentEvent/onSessionTranscriptUpdate
12. **Plugin hooks session lifecycle** — sessionKey in session_start/session_end
13. **channelRuntime** — ChannelGatewayContext 暴露共用 runtime helpers

### 安全/修復
- HTTP 安全標頭：Permissions-Policy + X-Content-Type-Options nosniff
- Auth 標籤安全：移除 token/API key 片段
- denyCommands 審計建議
- iOS 安全堆疊：Voice timing、Concurrency locks、Runtime guards、TTS fallback
- iOS keychain hardening
- LINE auth 邊界強化（fail-closed、replay/dedup）
- Discord 多項修復（thread reset、presence、slash commands、voice decoding）
- Telegram draft-stream 邊界穩定性
- Routing session duplicate suppression
- Legacy route inheritance guards

### 架構
- 懶加載邊界（.runtime.ts）取代無效動態匯入
- Plugin SDK 拆分啟動匯入
- 建置時 [INEFFECTIVE_DYNAMIC_IMPORT] 警告檢查

## 更新的文件

| 文件 | 更新內容 |
|------|---------|
| README.md | 版本號 |
| 01-專案概覽.md | 版本號、新功能（web search、loop detection、truncation、memory ollama、SecretRef 64、HTTP headers） |
| 02-架構設計.md | Plugin SDK subpaths、安全（HTTP headers、lazy boundaries） |
| 03-CLI命令系統.md | （無需變更，已包含 config validate） |
| 04-Gateway伺服器.md | SecretRef 64 targets、Agent 發現、安全標頭、auth 標籤 |
| 05-Agent系統.md | 工具迴圈偵測、工具結果截斷、bootstrap 截斷警告、web search 提供者、Ollama 嵌入、Diffs PDF 品質、compaction 擴充、session 日期校準 |
| 06-通訊頻道.md | Telegram topic routing/multi-account/hooks、Discord allowBots/thread/presence/mentions/slash/voice、Slack typingReaction、LINE auth/media/routing/status |
| 07-擴充套件開發.md | SDK subpaths、runtime APIs（system/events/channelRuntime）、hooks sessionKey、lazy loading |
| 08-應用程式.md | iOS 安全堆疊（voice/concurrency/keychain/security/TTS fallback） |
| 09-開發環境設定.md | 動態匯入護欄、記憶體壓力測試環境變數 |
| 10-配置系統.md | loopDetection、web search、typingReaction、allowBots、topic agentId、bootstrapPromptTruncationWarning、memorySearch |
| 11-API參考.md | 版本號、Plugin Runtime APIs、Hooks 事件（session lifecycle）、SDK subpaths |
| 13-Mac-Mini-M4-16G.md | 版本號 |
| 14-Mac-Mini-M4-24G.md | 版本號 |
