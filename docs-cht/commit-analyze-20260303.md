# OpenClaw 提交分析報告 — 2026-03-03

> **分析範圍**：從 `fd0892329`（Merge branch 'develop' into cursor，2026-03-02）至 `4d04e1a41`（2026-03-03 最新），共 **682 個提交**，涉及 **1761 個檔案**、新增 92,383 行、刪除 51,372 行。

---

## 目錄

1. [重大破壞性變更 (Breaking Changes)](#1-重大破壞性變更)
2. [新功能 (New Features)](#2-新功能)
3. [安全強化 (Security)](#3-安全強化)
4. [頻道改進 (Channels)](#4-頻道改進)
5. [Agent 與 Session](#5-agent-與-session)
6. [Gateway 與 CLI](#6-gateway-與-cli)
7. [瀏覽器工具 (Browser)](#7-瀏覽器工具)
8. [媒體與音訊 (Media / Audio)](#8-媒體與音訊)
9. [Cron 與排程](#9-cron-與排程)
10. [效能優化 (Performance)](#10-效能優化)
11. [平台相容性 (Platform)](#11-平台相容性)
12. [測試與重構](#12-測試與重構)
13. [文件更新](#13-文件更新)
14. [摘要統計](#14-摘要統計)

---

## 1. 重大破壞性變更

> ⚠️ 升級前必讀，可能影響現有部署或外掛開發

### Plugin SDK：`registerHttpHandler` 移除

```
BREAKING: Plugin SDK removed api.registerHttpHandler(...)
```

插件必須改用 `api.registerHttpRoute({ path, auth, match, handler })`，動態 webhook 生命週期請用 `registerPluginHttpRoute(...)`。

**影響範圍**：所有使用 `api.registerHttpHandler` 的自訂外掛（channel plugins、webhook 外掛）。

---

### Zalouser：移除外部 CLI 依賴

```
BREAKING: @openclaw/zalouser no longer depends on external zca-compatible CLI binaries
```

Zalo Personal 外掛已完全改用 JS 原生 `zca-js` 整合，不再依賴 `openzca`、`zca-cli` 等外部二進位檔。升級後需執行：
```bash
openclaw channels login --channel zalouser
```

---

### Onboarding 預設 tools.profile 改為 messaging

```
BREAKING: Onboarding now defaults tools.profile to "messaging" for new local installs
```

新安裝不再預設具備廣泛的 coding/system 工具，必須明確配置才能啟用。現有安裝不受影響。

---

### ACP dispatch 預設啟用

```
BREAKING: ACP dispatch now defaults to enabled unless explicitly disabled
```

如需暫停 ACP 路由，設定：
```json5
{ acp: { dispatch: { enabled: false } } }
```

---

## 2. 新功能

### 2.1 PDF 分析工具

新增一等公民 `pdf` 工具，支援原生 Anthropic 與 Google 提供者，非原生模型自動降級提取。

**配置**：
```json5
{
  agents: {
    defaults: {
      pdfModel: "anthropic/claude-sonnet-4-5",
      pdfMaxBytesMb: 10,
      pdfMaxPages: 50
    }
  }
}
```

**相關 PR**：#31319（感謝 @tyler6204）

---

### 2.2 openclaw config validate

新增 CLI 指令用於在 Gateway 啟動前驗證配置：
```bash
openclaw config validate
openclaw config validate --json
```

無效鍵路徑和 Zod/AJV 驗證允許值均納入報告（`--json` 機器可讀輸出）。

**相關 PR**：#31220（感謝 @Sid-Qin）

---

### 2.3 CLI Banner Tagline 模式

新增配置控制 CLI 啟動橫幅的 tagline 行為：
```json5
{
  cli: {
    banner: {
      taglineMode: "random" // random | default | off
    }
  }
}
```

---

### 2.4 Diffs 工具 — PDF 輸出與品質控制

`diffs` 工具新增 PDF 檔案輸出支援及渲染品質控制：
```json5
{
  fileQuality: 90,
  fileScale: 2,
  fileMaxWidth: 1200
}
```

當訊息頻道壓縮圖片時，PDF 是較佳的輸出選項。

**相關 PR**：#31342（感謝 @gumadeiras）

---

### 2.5 MiniMax M2.5 Highspeed 支援

新增 `MiniMax-M2.5-highspeed` 至提供者目錄、onboarding 流程及 MiniMax OAuth 外掛預設，同時保持對 `MiniMax-M2.5-Lightning` 的向下相容。

---

### 2.6 Outbound Adapter sendPayload

跨 Discord、Slack、WhatsApp、Zalo、Zalouser 統一的 `sendPayload` 機制，支援多媒體迭代和區塊感知文字降級。

**相關 PR**：#30144（感謝 @nohat）

---

### 2.7 音頻轉錄回聲 (Audio Echo)

新增可選的 `echoTranscript` 功能，在代理處理前向原始聊天發送轉錄確認訊息。預設關閉。

```json5
{
  tools: {
    media: {
      audio: {
        echoTranscript: true,
        echoFormat: "blockquote" // 格式選項
      }
    }
  }
}
```

**相關 PR**：#32150（感謝 @AytuncYildizli）

---

### 2.8 Plugin STT API

外掛可透過 `api.runtime.stt.transcribeAudioFile(...)` 呼叫 OpenClaw 配置的媒體理解音頻提供者進行音頻轉錄。

**相關 PR**：#22402（感謝 @benthecarman）

---

### 2.9 sessions_spawn 內聯附件

`sessions_spawn` 子代理執行時支援 base64/utf8 編碼的內聯檔案附件，包含轉錄內容遮蔽、生命週期清理及可配置限制：
```json5
{
  tools: {
    sessions_spawn: {
      attachments: {
        maxSizeBytes: 1048576,
        maxCount: 5
      }
    }
  }
}
```

**相關 PR**：#16761（感謝 @napetrov）

---

### 2.10 Hooks 訊息生命週期擴展

新增內部 hook 事件：
- `message:transcribed` — 音頻轉錄完成後
- `message:preprocessed` — 訊息前處理完成後
- `message:sent` 增加 `isGroup`、`groupId` 欄位

**相關 PR**：#9859（感謝 @Drickon）

---

### 2.11 Telegram 串流預設改為 partial

`channels.telegram.streaming` 預設值從 `off` 改為 `partial`，新 Telegram 安裝開箱即得即時預覽串流，原生 draft 不可用時自動降級為 message-edit 預覽。

---

### 2.12 Telegram DM 私聊串流

使用 `sendMessageDraft` 作為私聊預覽串流，保持 reasoning/answer 預覽通道分離。

**相關 PR**：#31824（感謝 @obviyus）

---

### 2.13 Telegram disableAudioPreflight

群組/topic 配置中可加入 `disableAudioPreflight`，跳過入站語音訊息的提及偵測預飛轉錄：
```json5
{
  channels: {
    telegram: {
      groups: {
        "group_id": {
          disableAudioPreflight: true
        }
      }
    }
  }
}
```

**相關 PR**：#23067（感謝 @yangnim21029）

---

### 2.14 Cron 失敗目標 (Failure Destination)

排程任務執行失敗時可獨立路由通知：
```json5
{
  cron: {
    failureDestination: {
      channel: "discord",
      to: "user:ADMIN_USER_ID"
    },
    jobs: [
      {
        id: "my-job",
        // ...
        delivery: {
          failureDestination: {
            channel: "telegram",
            to: "user:BACKUP_ID"
          }
        }
      }
    ]
  }
}
```

**相關 PR**：#31059（感謝 @kesor）

---

### 2.15 Cron UI 進階控制

Control UI cron 編輯器新增：
- `run-if-due` 控制
- 路由配置
- Cron 模型自動補全（含 agent 模型預設）

---

### 2.16 Docker Sandbox 支援（opt-in）

Docker 部署可啟用 sandbox：
```json5
{
  agents: {
    defaults: {
      sandbox: {
        docker: {
          enabled: true,
          socketPath: "/var/run/docker.sock" // 自訂 socket: OPENCLAW_DOCKER_SOCKET
        }
      }
    }
  }
}
```

環境變數：`OPENCLAW_SANDBOX=1`（明確 opt-in）

**相關 PR**：#29974（感謝 @jamtujest、@vincentkoc）

---

### 2.17 OpenAI OAuth TLS 預飛檢查

加入 OpenAI Codex OAuth TLS 憑證鏈預飛檢查，並整合 doctor 前提條件探測（僅在 OpenAI Codex OAuth 配置時啟用，或 `doctor --deep`）。

**相關 PR**：#32051（感謝 @alexfilatov）

---

### 2.18 Zalouser 原生 zca-js 整合

Zalo Personal 外掛完全改用 JS 原生 `zca-js`，支援：
- reactions（表情回應）
- 群組上下文
- 收據確認（receipt acks）
- 帳號範圍安全（pairing store 按 accountId 隔離）

---

### 2.19 Tlon 頻道強化

Tlon (Urbit) 頻道/外掛功能大幅擴充。

**相關 PR**：#21208

---

### 2.20 Voice-Call 等待佇列

Twilio 入站通話新增通話等待佇列支援，可排隊同時到達的多通電話。

---

## 3. 安全強化

> 本次更新包含大量安全修復，以下為主要項目

### 3.1 Webhook Auth-Before-Body 強化

BlueBubbles、Google Chat、LINE 的 webhook 處理器現強制 **先驗證身份、再讀取請求 body**，加入嚴格的 pre-auth body/time 預算，防止未認證慢速 body DoS 攻擊。

**回報者**：@GCXWLP

---

### 3.2 Sandbox Bootstrap 邊界強化

拒絕透過 symlink/hardlink 別名讀取 workspace 外部的 bootstrap seed 文件（如 `AGENTS.md`），後壓縮時的 `AGENTS.md` 上下文讀取改用邊界驗證的安全開啟。

**回報者**：@tdjackey

---

### 3.3 Skills 工作區 Symlink 逃脫審計

`openclaw security audit` 新增 `skills.workspace.symlink_escape` 警告，當 `skills/**/SKILL.md` 解析到工作區外部時（如 symlink 鏈漂移）發出警告。

---

### 3.4 ACP Sandbox 繼承強化

從沙箱 session 發起的 `sessions_spawn` 加入 ACP runtime 失閉（fail-closed）防護：
- 拒絕沙箱 session 透過 `runtime: "acp"` 生成 ACP 子代理
- 拒絕 ACP runtime 設定 `sandbox: "require"`

防止透過主機端 ACP 初始化繞過沙箱邊界。

**相關 PR**：#32254（回報者：@tdjackey）

---

### 3.5 節點相機 URL 下載邊界強化

`camera.snap`/`camera.clip` URL payload 下載現綁定到已解析的節點主機，`remoteIp` 不可用時失閉，使用 SSRF 防護 fetch 及重定向主機/協議檢查。

**回報者**：@tdjackey

---

### 3.6 Gateway 外掛 HTTP 路徑正規化強化

對外掛路由路徑變體進行規範化解碼（bounded depth），對規範化異常失閉，強制深層編碼的 `/api/channels/*` 變體通過 gateway auth，防止透過外掛處理器的替代路徑 auth 繞過。

**回報者**：@tdjackey

---

### 3.7 Prompt Spoofing 防護

停止將排隊的執行時事件注入 user-role prompt 文字，改透過可信的 system-prompt 上下文路由，並中和入站訊息內容中的偽造標記（如 `[System Message]`、行首 `System:`）。

**相關 PR**：#30448

---

### 3.8 正規表達式 bound 輸入防護

為 session 過濾和日誌遮蔽的大型 regex 評估輸入加上邊界；強化量化模糊交替模式偵測（如 `(a|aa)+`）。

---

### 3.9 Skills Archive 提取安全

統一 tar.gz 和 tar.bz2 安裝流程的 tar 提取安全檢查，強制壓縮大小限制，tar.bz2 在預飛和提取間變更則失閉。

**回報者**：@GCXWLP

---

### 3.10 SSRF 防護強化 (Web Tools)

`web_fetch` 和 citation-redirect URL 在 proxy env 設定時保持 DNS pinning，受信任/操作者控制端點需明確危險 opt-in 才能繞過固定分發。

**回報者**：@tdjackey

---

### 3.11 macOS LaunchAgent Umask 強化

生成的 gateway launchd plist 中寫入 `Umask=63`（八進位 `077`），npm 更新後服務重新安裝保持 owner-only 檔案權限。

**相關 PR**：#32022（感謝 @liuxiaopai-ai）

---

### 3.12 Windows Spawn 正規化（ACPX）

ACPX Windows spawn 強化：透過 PATH/PATHEXT 解析 `.cmd/.bat` wrapper，在不需要 shell 解析時直接執行 Node/EXE 入口點，`strictWindowsCmdWrapper` 預設嚴格失閉。

**回報者**：@tdjackey

---

### 3.13 配置備份權限強化

輪換的配置備份強制 owner-only (`0600`) 權限，並清理受管備份環之外的孤立 `.bak.*` 檔案。

**相關 PR**：#31718（感謝 @YUJIE2002）

---

## 4. 頻道改進

### 4.1 Discord

| 改進項目 | 說明 |
|---------|------|
| 並行訊息分發 | 每頻道獨立訊息佇列，恢復並行代理分發，同頻道內保持順序 |
| Slack mrkdwn 正規化 | 串流和預覽路徑的 mrkdwn 轉換，確保格式一致 |
| 音頻前飛提及偵測 | 透過 `content_type` 偵測音頻附件，對文字提及進行前飛轉錄 |
| 論壇系統訊息排除 | 排除 forum topic 系統服務訊息（`forum_topic_*`、`general_forum_topic_*`）免於隱式提及偵測 |
| 語音 ffmpeg 路徑 | 強化語音 ffmpeg 路徑和 opus 快速路徑 |
| 音頻重採樣 | 語音訊息重採樣至 48kHz |
| 連線重連 watchdog | 加入 transport stall-watchdog，強化生命週期 force-stop |
| DM 指令 auth | 統一 DM allowlist + pairing-store 授權 |

**相關 PR**：#32298、#32136、#32262、#31927

---

### 4.2 Telegram

| 改進項目 | 說明 |
|---------|------|
| 串流預設 partial | 預設從 `off` 改為 `partial` |
| DM 私聊串流 | 使用 `sendMessageDraft` |
| disableAudioPreflight | 跳過入站語音前飛轉錄 |
| 長模型按鈕可選 | compact callback payload + provider 重新提示 |
| DM 路由 | 按 sender id 路由 DM session |
| 重複 token 防護 | guard token.trim() 防止 undefined 崩潰 |
| sticker getFile 重試 | 加入重試邏輯 |

---

### 4.3 Feishu / Lark

| 改進項目 | 說明 |
|---------|------|
| topic session 路由 | `thread_id` 作為 topic session scope 降級 |
| topic 根回覆 | 優先使用 `root_id` 作為 `replyTargetMessageId` |
| DM 配對回覆 | 發送至 `chat:<chat_id>` 而非 `user:<open_id>` |
| Lark 私聊路由 | `chat_type: "private"` 視為 DM 上下文 |
| 發件人查詢權限抑制 | 抑制 `contact:contact.base:readonly` 錯誤的使用者提示 |
| 訊息 debounce | 快速同聊天發送者連發 debounce 成一次有序分發 |
| 訊息序列化 | 每聊天序列化訊息處理，保持跨聊天並行 |
| dedup 重啟持久化 | monitor 啟動時熱載入持久化 dedup 狀態 |
| typing 通知抑制 | 跳過已啟動指示器的 keepalive 重新加入 |
| 系統預覽泄漏修復 | 停止將 Feishu 訊息預覽排入 system 事件 |

---

### 4.4 Slack

| 改進項目 | 說明 |
|---------|------|
| off-mode session 路由 | `replyToMode=off` 保持頂層頻道訊息在單一 session |
| inbound debounce 路由 | 按 message timestamp 隔離 debounce key |
| thread context 優化 | 現有 session 的 thread 上下文跳過 header/history，減少 token 浪費 |
| Bolt 4.6 相容 | 移除導致崩潰的 `message.channels`/`message.groups` handler 註冊，改用 `channel_type` 統一 handler |
| null-safe 提及 | 防止 undefined text 崩潰提及剝離 |

---

### 4.5 WhatsApp

- 入站 `fromMe` 上下文傳播，標記 owner 自發訊息為 `(self)`
- 主 DM 路由改為單一 owner（`ab0f534e90`）

---

### 4.6 Synology Chat

| 改進項目 | 說明 |
|---------|------|
| Webhook 相容 | 接受 JSON 及別名 payload 欄位，支援 body/query/header token 來源，以 `204` ACK |
| Bounded body reads | 防止未認證慢速 body 掛起 |
| 回覆交付 | 解析 webhook username → Chat API `user_id` |
| 啟動循環修復 | `startAccount` 等到 abort 防止 webhook 路由重啟循環 |

---

### 4.7 Signal

- `react` 工具 fallback 到 `toolContext.currentMessageId`（與 Telegram 行為對齊）
- Loop 防護：UUID-only `accountUuid` 配置下也評估 own-account 偵測

---

### 4.8 Zalo

- webhook 帳號範圍回歸測試
- Zalouser pairing store 按 `accountId` 隔離

---

### 4.9 Matrix

- Conduit 相容：非阻塞 sync 啟動、防重複 monitor listener 、移除 2 成員 DM 啟發式
- 接受無別名 `!room` ID

---

### 4.10 WebChat

- `NO_REPLY`-only 轉錄條目從 `chat.history` 過濾
- 串流最終化：在最終事件省略 `message` 時持久化串流文字

---

## 5. Agent 與 Session

### 5.1 Gemini Schema Null 屬性保護

在提供者驗證前，將畸形的 JSON Schema `properties` 值（`null`、陣列、原始值）強制轉為 `{}`，防止無效外掛/工具 schema 崩潰嚴格驗證器。

**相關 PR**：#32332（感謝 @webdevtodayjason）

---

### 5.2 OpenAI WS Tool Call ID 清理

正規化空白/空字串的串流 tool-call id，阻止空 `function_call_output.call_id` payload，防止 OpenAI 400 錯誤。

---

### 5.3 記憶沖洗強制預壓縮

```json5
{
  agents: {
    defaults: {
      compaction: {
        memoryFlush: {
          forceFlushTranscriptBytes: 2097152 // 2MB，預設值
        }
      }
    }
  }
}
```

長對話在 token 快照過期時自動恢復，無需手動刪除轉錄。

**相關 PR**：#30655

---

### 5.4 Session PID 鎖恢復

透過比對 lock-file `starttime` 與 `/proc/<pid>/stat` starttime，偵測被回收的 Linux PID，在容器化 PID 重用場景下立即回收過期鎖定檔案。

**相關 PR**：#26443（感謝 @HirokiKobayashi-R、@vincentkoc）

---

### 5.5 fsPolicy workspaceOnly 傳播

`tools.fs.workspaceOnly` 現正確傳播至 image/pdf 工具的 local-root 允許清單。

**相關 PR**：#31882（感謝 @justinhuangcode）

---

### 5.6 Exec Allowlist Regex 修復

逃脫 path-pattern 字面量中的 regex 元字元（保留 glob 萬用字元），防止包含 `/usr/bin/g++` 等字元的可執行檔崩潰和錯誤匹配。

**相關 PR**：#32162（感謝 @stakeswky）

---

### 5.7 Tool Call State 中斷清理

中斷時無論 synthetic tool 結果是否停用，一律清除 pending tool-call 狀態，防止孤立 tool-use 轉錄導致後續提供者請求失敗。

**相關 PR**：#32120（感謝 @jnMetaCode）

---

### 5.8 子代理公告清理

- 完成訊息執行在後代 settle 期間保持 pending
- 加入 30 分鐘強制過期防止無限期 pending 狀態

**相關 PR**：#23970（感謝 @tyler6204）

---

### 5.9 Moonshot / Kimi thinking 相容

套用原生 thinking payload 相容性，failover stop reason error 視為 timeout。

---

## 6. Gateway 與 CLI

### 6.1 Gateway 心跳模型熱重載

`models.*` 和 `agents.defaults.model` 配置更新現觸發心跳熱重載，無需完整 gateway 重啟。

**相關 PR**：#32046（感謝 @stakeswky）

---

### 6.2 Plugin HTTP 路由優先序修復

顯式外掛 HTTP 路由在 Control UI SPA catch-all 之前執行，確保已註冊的外掛 webhook/自訂路徑可到達。

**相關 PR**：#31885（感謝 @Sid-Qin）

---

### 6.3 Gateway 節點危險指令同步

`sms.send` 加入預設 onboarding 節點 `denyCommands`，與危險指令真相來源共享 onboarding 拒絕預設，phone-control `/phone arm writes` 也包含 `sms.send`。

**感謝**：@zpbrent

---

### 6.4 Windows 端口偵測

加入 Windows 相容端口偵測（使用 netstat fallback）及等待進程退出後再重啟 Gateway。

**相關 PR**：#29239（感謝 @ajay99511）、#27913（感謝 @tda1017）

---

### 6.5 Control UI basePath Webhook 穿透修復

配置了 `controlUiBasePath` 時，非 GET 請求正確穿透至外掛路由（不再回傳 405）。

**相關 PR**：#32311（感謝 @ademczuk）

---

### 6.6 Webchat NO_REPLY 串流抑制

抑制 `NO_REPLY` 前綴的助手 lead-fragment delta，防止在靜默回應執行時出現 `NO` 前綴洩漏。

**相關 PR**：#32073（感謝 @liuxiaopai-ai）

---

### 6.7 Daemon/Homebrew 執行時釘固

解析 Homebrew Cellar Node 路徑至穩定受管 symlink（包含 `node@22` 版本化 formula），Gateway 安裝在 brew 升級後保持預期執行時。

**相關 PR**：#32185（感謝 @scoootscooob）

---

### 6.8 CLI 啟動速度優化 (Raspberry Pi)

- 快速路由避免不必要的外掛預載
- 根目錄 `--version` 快速路徑 bootstrap 繞過
- 並行化 JSON/非 JSON 狀態掃描
- 啟動時啟用 Node 編譯快取（`NODE_COMPILE_CACHE`）

**相關 PR**：#5871（感謝 @BookCatKid）

---

### 6.9 Hooks 穩定性

hooks 內部處理器登錄表改為 `globalThis` singleton，在 bundle 拆分產生重複模組副本時保持 hook 登錄/分發一致性。

**相關 PR**：#32292（感謝 @Drickon）

---

### 6.10 Webhook ACK 從 202 改為 200

`/hooks/agent` 成功請求改回傳 `200`（原 `202`），與要求 200 的提供者（如 Forward Email）相容。

**相關 PR**：#28204（感謝 @Glucksberg）

---

## 7. 瀏覽器工具

| 改進項目 | 說明 | PR |
|---------|------|-----|
| CDP 啟動就緒等待 | 在回報 `cdpReady` 前要求成功的 `Browser.getVersion` 回應 | #29538 |
| 受管分頁上限 | 限制 loopback managed `openclaw` 頁面分頁至 8 個 | #29724 |
| 代理繞過 | CDP localhost HTTP/WS 連線強制直連，不透過 proxy | #31469 |
| 遠端 CDP 擁有者略過 | 非 loopback 遠端 CDP profile 跳過本地進程擁有者錯誤 | #28780 |
| 預設 openclaw profile | 無沙箱/headless 環境優先 `openclaw` profile | #14944 |
| 繼電器重連容忍 | 容忍短暫 MV3 worker 中斷，不關閉附加分頁 | #30232 |
| cdpReady 探針準確性 | 要求 CDP websocket 成功（不只 socket open） | #23427 |
| 啟動診斷 | Chrome stderr 和 Linux no-sandbox 提示納入啟動超時錯誤 | #29355 |
| Act 請求相容 | 接受傳統扁平 `action="act"` 參數 | #31359 |
| profile attach-only 支援 | `browser.profiles.<name>.attachOnly` 支援 | #31429 |

---

## 8. 媒體與音訊

### 8.1 MIME 正規化

正規化帶參數/大小寫變體的 MIME 字串（如 `Audio/Ogg; codecs=opus`），WhatsApp 語音訊息正確分類為 audio 並路由至轉錄。跨 Telegram/Signal/iMessage 媒體種類檢查通過統一的 `kindFromMime` 正規化。

**相關 PR**：#32280（感謝 @Lucenx9）

---

### 8.2 音頻轉錄小檔案保護

跳過小於 1024 bytes 的音頻檔案（tiny/空檔案），避免無效音頻錯誤。

**相關 PR**：#8388（感謝 @Glucksberg）

---

### 8.3 Proxy 感知媒體 HTTP

從 `HTTPS_PROXY`/`HTTP_PROXY` 環境變數傳遞 proxy 感知 fetch 函數至音頻/視頻提供者調用，轉錄/視頻請求遵守配置的出站 proxy。

**相關 PR**：#27093（感謝 @mcaxtr）

---

### 8.4 parakeet-mlx 輸出解析

從 `--output-dir/<media-basename>.txt` 讀取轉錄（stdout fallback 用於非 txt 格式）。

**相關 PR**：#9177（感謝 @mac-110）

---

### 8.5 OpenAI 音頻能力

`audio` 加入 OpenAI 提供者能力清單，音頻轉錄模型現可在媒體理解提供者選擇中被選中。

---

## 9. Cron 與排程

| 改進項目 | 說明 |
|---------|------|
| 遺留 store 格式遷移 | 正規化遺留 string `schedule` 和頂層 `command`/`timeout` 欄位 |
| HEARTBEAT_OK 抑制 | 抑制 fallback 主 session 排入，防止 `HEARTBEAT_OK` 出現在聊天 |
| 多 payload 心跳抑制 | 任何 payload 含心跳 ack token 且無 media 時視為可跳過 |
| Session reaper 可靠性 | Cron session reaper 移入 `onTimer` `finally`，定時器失敗時仍繼續清理 |
| 一次性任務重排 | 允許已完成的 `at` 任務在重排至更晚時間後再次執行 |
| 失敗目標路由 | `failureAlert.mode`、`cron.failureDestination`、per-job `delivery.failureDestination` |
| 排程 cwd 重新驗證 | `system.run` 執行前重新驗證 approval-bound `cwd` |
| 日誌時間戳本地時間 | 使用本地時間而非 UTC |

---

## 10. 效能優化

| 改進項目 | 說明 |
|---------|------|
| 路由/提及 regex 快取 | 快取路由和提及 regex 解析 |
| Agent ID 正規化快取 | 快取正規化 agent-id 查詢 |
| Allowlist/Account ID 正規化 | 快取 allowlist 和帳號 ID 正規化 |
| Config schema 工作跳過 | 跳過冗餘 schema 和 session-store 工作 |
| Slack 允許路徑記憶化 | `allow-from` 和提及路徑記憶化 |
| 安全掃描目錄走訪快取 | 快取 scanner 目錄走訪 |
| Cron 排程評估快取 | 快取 schedule 評估器和偏移量 |
| 頻道外掛查詢快取 | 縮短熱路徑分配 |
| CLI preaction 模組快取 | 快取 preaction lazy 模組導入 |
| 測試套件大幅提速 | 大量測試 fixture、計時器、序列化開銷優化 |

---

## 11. 平台相容性

### 11.1 Windows

- 跨 ACP client、QMD/mcporter、sandbox Docker 統一非核心 spawn 處理（#31750）
- PATH key 大小寫不敏感解析用於 `pathPrepend`（#31879）
- 節點執行輸出以 active code page 解碼（#30652）
- `Process.fix Windows .cmd spawn EINVAL`（#29759）
- PowerShell 執行策略處理（#24794）
- Windows Scheduled Task 過期執行狀態修復（#19504）
- `cron.run` EBUSY fallback 使用 `copyFile`（#16932）
- `exec` PATH key 大小寫不敏感（#25399）

### 11.2 macOS

- PeekabooBridge 為 `clawdbot`、`clawdis`、`moltbot` 加入相容 socket symlink（#6033）
- LaunchAgent umask 077 強化（#31919）
- Doctor iCloud/CloudStorage 狀態目錄警告（#31004）
- Homebrew Cellar Node 路徑解析（#32185）

### 11.3 Linux / Raspberry Pi

- CLI 啟動速度大幅提升（#5871）
- WSL2 開機自動啟動指南（#31616）
- Doctor SD/eMMC 狀態目錄警告（#31033）
- 容器中缺少 `systemctl` 的處理（#26089）

### 11.4 Docker

- Sandbox opt-in 環境變數明確（`OPENCLAW_SANDBOX=1|true|yes|on`）
- 自訂 Docker socket：`OPENCLAW_DOCKER_SOCKET`
- OCI base-image 標籤完善
- Docker Compose gateway 目標：共享信任邊界、`cap_drop`、`no-new-privileges`

---

## 12. 測試與重構

本次更新包含大量（約 **300+** 個）測試效能優化和重構提交：

- **測試效能**：fixture 快取、計時器緊縮、重複案例合併、臨時目錄重用
- **重構去重**：提取共享 fixture、helper、工廠函數，消除各模組（agents、browser、cli、gateway、channels）的重複設定
- **CI 修復**：型別錯誤修正、lint 修復、跨平台斷言相容
- **vitest 分流**：快速通道（unit）、頻道/gateway 套件、e2e 通道分開執行

---

## 13. 文件更新

- 提供者清單排序（A-Z）
- 工具連結排序（A-Z）
- 頻道清單跨語系排序
- WSL2 開機自動啟動指南
- Windows exec 編碼故障排除
- Docker 沙箱指南（常用映像與 bootstrap 說明）
- Android 節點配對與能力文件同步
- ACP 權限文件（`permissionMode` 預設值）
- 回應式 onboarding 工具 profile 預設說明
- 自適應 thinking 和 OpenAI WebSocket 文件澄清
- Moonshot thinking 和 failover stop-reason 澄清

---

## 14. 摘要統計

| 指標 | 數值 |
|------|------|
| 總提交數 | **682** |
| 影響檔案數 | **1,761** |
| 新增行數 | 92,383 |
| 刪除行數 | 51,372 |
| 重大破壞性變更 | **4 項** |
| 新功能 | **20+ 項** |
| 安全修復 | **15+ 項** |
| 效能優化提交 | **~100+** |
| 測試/重構提交 | **~300+** |

### 按模組分布

| 模組 | 變更強度 |
|------|---------|
| 安全 (Security) | ⬛⬛⬛⬛⬛ 極高 |
| 測試/效能 (Tests/Perf) | ⬛⬛⬛⬛⬛ 極高 |
| 頻道 - 飛書 (Feishu) | ⬛⬛⬛⬛ 高 |
| 瀏覽器 (Browser) | ⬛⬛⬛⬛ 高 |
| Agent/Session | ⬛⬛⬛ 中 |
| Gateway/CLI | ⬛⬛⬛ 中 |
| Telegram | ⬛⬛⬛ 中 |
| Cron | ⬛⬛ 中 |
| Media/Audio | ⬛⬛ 中 |
| Discord | ⬛⬛ 中 |
| Windows 平台 | ⬛⬛ 中 |
| Slack | ⬛⬛ 中 |
| Zalouser | ⬛ 低 |

---

*本文件由 AI 自動分析生成，分析基準點：`fd0892329` → `4d04e1a41`（2026-03-03）*
