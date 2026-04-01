# OpenClaw TUI 斜線指令完整指南

> 版本：`2026.3.31` | Node.js `>=22.16.0`
>
> 本文件完整說明 `openclaw tui` 終端管理介面中所有斜線指令（Slash Commands）的用途、語法、參數和注意事項。

---

## 目錄

1. [概述](#1-概述)
2. [Session 與生命週期](#2-session-與生命週期)
3. [模型與推論選項](#3-模型與推論選項)
4. [狀態與資訊](#4-狀態與資訊)
5. [工具與執行](#5-工具與執行)
6. [子代理管理](#6-子代理管理)
7. [ACP 代理控制](#7-acp-代理控制)
8. [訊息佇列控制](#8-訊息佇列控制)
9. [TTS 語音合成](#9-tts-語音合成)
10. [設定與管理](#10-設定與管理)
11. [頻道切換（Dock）](#11-頻道切換dock)
12. [TUI 專屬操作](#12-tui-專屬操作)
13. [指令語法規則](#13-指令語法規則)
14. [權限與授權](#14-權限與授權)
15. [設定旗標參考](#15-設定旗標參考)

---

## 1. 概述

### 啟動 TUI

```bash
openclaw tui              # 啟動終端管理介面
openclaw tui --agent main # 指定 agent
```

### 指令格式

所有斜線指令以 `/` 開頭，作為獨立訊息發送：

```
/help                     # 無參數
/model anthropic/claude-sonnet-4-6    # 帶參數
/think: high              # 冒號分隔（可選）
/exec host=gateway security=allowlist # 鍵值參數
```

### 指令總覽

| 類別 | 指令數 | 範例 |
|------|--------|------|
| Session 與生命週期 | 5 | `/new`、`/reset`、`/stop`、`/compact`、`/abort` |
| 模型與推論 | 8 | `/model`、`/think`、`/fast`、`/verbose`、`/reasoning`、`/elevated`、`/usage`、`/exec` |
| 狀態與資訊 | 6 | `/help`、`/status`、`/context`、`/whoami`、`/commands`、`/export-session` |
| 工具與執行 | 4 | `/tools`、`/skill`、`/btw`、`/bash` |
| 子代理 | 5 | `/subagents`、`/focus`、`/unfocus`、`/kill`、`/steer` |
| ACP | 1 | `/acp`（含 15 個子指令） |
| 佇列 | 1 | `/queue`（含 5 種模式） |
| TTS | 1 | `/tts`（含 10 個子指令） |
| 設定管理 | 7 | `/config`、`/mcp`、`/plugins`、`/debug`、`/allowlist`、`/send`、`/activation` |
| 頻道切換 | 動態 | `/dock-telegram`、`/dock-discord` 等 |
| TUI 專屬 | 6 | `/agent`、`/session`、`/settings`、`/exit` 等 |

---

## 2. Session 與生命週期

### /new — 新建 Session

```
/new
/new [model-hint]
```

**功能**：建立全新 session，清除所有歷史紀錄。

**參數**：
- `model-hint`（可選）：新 session 使用的模型（provider/model 或 provider 名稱）

**行為**：
- 建立新的 session ID
- 清除對話歷史
- 觸發 `command:new` hook（如 session-memory 會儲存前一 session）
- 保留 agent 和頻道設定

**範例**：
```
/new
/new anthropic/claude-opus-4-6
```

---

### /reset — 重設 Session

```
/reset
/reset [optional-content]
```

**功能**：重設當前 session 的訊息歷史。

**參數**：
- `optional-content`（可選）：重設後傳遞的內容

**行為**：
- 清除當前 session 的訊息歷史
- 不建立新的 session ID（與 `/new` 不同）
- 觸發 `command:reset` hook

**與 /new 的差異**：

| 項目 | /new | /reset |
|------|------|--------|
| Session ID | 新建 | 保留 |
| 歷史紀錄 | 清除 | 清除 |
| Hook 觸發 | `command:new` | `command:reset` |

---

### /stop — 停止當前運行

```
/stop
```

**功能**：停止當前正在執行的 agent run。

**行為**：
- 立即停止活動的 LLM 呼叫和工具執行
- 觸發 `command:stop` hook

---

### /abort — 中止活動 Run

```
/abort
```

**功能**：中止當前活動的 run（TUI 專屬，立即生效）。

---

### /compact — 手動壓縮

```
/compact
/compact [instructions]
```

**功能**：手動觸發 session 上下文壓縮（compaction）。

**參數**：
- `instructions`（可選）：額外的壓縮指示，引導 LLM 如何摘要

**行為**：
- 將較舊的對話歷史摘要為緊湊的摘要條目
- 近期訊息保持完整
- 摘要持久化至 session JSONL 中
- 壓縮前可能觸發靜默的記憶沖洗（memory flush）

**用途**：
- 長對話接近 context window 時手動壓縮
- 配合自訂指示保留特定內容

**範例**：
```
/compact
/compact 請保留所有 API 端點的細節
```

---

## 3. 模型與推論選項

### /model — 設定模型

```
/model                              # 顯示當前模型
/model list                         # 顯示模型列表
/model <#>                          # 按編號選擇
/model <provider/model>             # 設定特定模型
/model status                       # 顯示詳細狀態
```

**功能**：查看或切換 LLM 模型。

**範例**：
```
/model                              # 顯示：anthropic/claude-sonnet-4-6
/model anthropic/claude-opus-4-6    # 切換至 Opus
/model openai/gpt-5.4               # 切換至 GPT-5.4
/model list                         # 顯示所有可用模型
/model 3                            # 選擇列表中的第 3 個
```

---

### /think — 設定 Thinking 等級

```
/think <level>
/thinking <level>
/t <level>
```

**別名**：`/thinking`、`/t`

**等級**（依模型/提供者而異）：

| 等級 | 說明 |
|------|------|
| `off` | 關閉推理思考 |
| `minimal` | 最少推理 |
| `low` | 低度推理 |
| `medium` | 中度推理 |
| `high` | 高度推理 |
| `xhigh` | 最大推理深度 |

**注意**：
- Claude 4.6 預設 `adaptive`
- 不是所有模型都支援所有等級
- 不支援的等級會自動降級為 `off`

**範例**：
```
/think high
/t off
/thinking medium
```

---

### /fast — 快速模式

```
/fast                    # 顯示當前狀態
/fast status             # 同上
/fast on                 # 啟用
/fast off                # 停用
```

**功能**：切換 session-level 快速模式。

**效果**（依提供者）：
- **Anthropic**：`service_tier: "auto"`（優先處理佇列）
- **OpenAI**：降低 reasoning effort + verbosity + 優先處理
- **xAI / MiniMax**：provider-specific 快速處理

---

### /verbose — 詳細模式

```
/verbose                 # 切換
/verbose on              # 啟用
/verbose off             # 停用
/v on                    # 別名
```

**別名**：`/v`

**功能**：控制工具執行結果的詳細程度。
- **on**：顯示完整工具輸出和錯誤詳情
- **off**：僅顯示摘要

---

### /reasoning — 推理可見性

```
/reasoning on            # 顯示推理過程
/reasoning off           # 隱藏推理過程
/reasoning stream        # 串流顯示推理
/reason on               # 別名
```

**別名**：`/reason`

**功能**：控制模型推理過程（thinking blocks）是否顯示在回覆中。

---

### /elevated — 提升模式

```
/elevated on             # 啟用提升
/elevated off            # 停用
/elevated ask            # 每次詢問
/elevated full           # 完全提升（跳過審批）
/elev full               # 別名
```

**別名**：`/elev`

**功能**：控制工具執行的權限等級。

| 等級 | 行為 |
|------|------|
| `off` | 基本權限，受限工具 |
| `on` | 提升權限，但仍需審批 |
| `ask` | 每個操作都詢問 |
| `full` | 完全提升，跳過所有審批提示 |

> **警告**：`full` 模式跳過所有安全審批，僅在信任的環境中使用。

---

### /usage — 使用量顯示

```
/usage                   # 循環切換模式
/usage off               # 關閉
/usage tokens            # 顯示 token 數
/usage full              # 完整使用量
/usage cost              # 成本摘要
```

**功能**：控制每次回覆後顯示的使用量資訊。

---

## 4. 狀態與資訊

### /help — 顯示幫助

```
/help
```

**功能**：顯示所有可用斜線指令的摘要說明。

---

### /status — 顯示狀態

```
/status
```

**功能**：顯示 Gateway / Session 的目前狀態摘要，包含：
- 連線狀態
- 當前模型
- 壓縮次數
- Session 資訊

---

### /context — 上下文說明

```
/context                 # 概覽說明
/context list            # 列出上下文組成
/context detail          # 每個檔案/工具/技能的大小明細
/context json            # JSON 格式輸出
```

**功能**：說明當前 agent 的上下文是如何建構的，包含：
- Bootstrap 檔案（AGENTS.md、SOUL.md 等）
- 可用工具
- 載入的 Skills
- System prompt 大小

**注意**：結果是 session-scoped，切換 agent/頻道/模型可能改變輸出。

---

### /whoami — 顯示發送者 ID

```
/whoami
/id                      # 別名
```

**別名**：`/id`

**功能**：顯示當前發送者的 ID（用於 allowlist 設定）。

---

### /commands — 列出所有指令

```
/commands
```

**功能**：列出所有已註冊的斜線指令。

---

### /export-session — 匯出 Session

```
/export-session
/export-session [path]
/export [path]           # 別名
```

**別名**：`/export`

**功能**：將當前 session 匯出為 HTML 檔案，包含完整的 system prompt。

**參數**：
- `path`（可選）：輸出檔案路徑（預設為工作區目錄）

**輸出內容**：
- 完整對話歷史
- 完整 system prompt
- Session metadata
- 所有上下文資訊

---

## 5. 工具與執行

### /tools — 列出可用工具

```
/tools                   # 精簡列表
/tools compact           # 同上
/tools verbose           # 含描述的完整列表
```

**功能**：列出當前 session 中所有可用的 runtime 工具。

---

### /skill — 執行技能

```
/skill <name> [input]
```

**功能**：依名稱執行特定技能。

**參數**：
- `name`（必要）：技能名稱
- `input`（可選）：技能輸入

**範例**：
```
/skill weather 台北天氣
/skill github list issues
/skill summarize https://example.com/article
```

---

### /btw — 附帶問題

```
/btw <question>
```

**功能**：在不影響 session 歷史的情況下問一個快速的附帶問題。

**行為**：
- 快照當前 session 上下文作為背景
- 執行獨立的一次性 LLM 呼叫（無工具）
- 回答不寫入 session 歷史
- 以「side result」形式呈現（與一般回覆視覺區分）
- TUI 中可用 Enter/Esc 關閉
- 不會在重新載入時回放

**用途**：
- 長時間 run 進行中時的快速釐清
- 不想污染 session 歷史的臨時問題

**範例**：
```
/btw 我們正在編輯哪個檔案？
/btw 這個錯誤是什麼意思？
/btw 用一句話總結目前的任務
```

---

### /bash — Shell 執行

```
/bash <command>
! <command>              # 別名（Bang 語法）
!poll [sessionId]        # 檢查輸出/狀態
!stop [sessionId]        # 停止執行中的工作
```

**功能**：在 host 上執行 shell 命令（非沙箱）。

**⚠️ 需要啟用**：`commands.bash: true`

**參數**：
- `command`（必要）：要執行的 shell 命令

**行為**：
- 前景模式：等待 `bashForegroundMs`（預設 2000ms）完成
- 超時後切換為背景模式
- 使用 `!poll` 檢查背景工作的輸出
- 使用 `!stop` 停止執行中的工作

**範例**：
```
/bash ls -la ~/workspace
! git status
! docker ps
!poll
!stop
```

**安全注意**：
- 僅限 host 執行（非沙箱）
- 需要 `commands.bash: true` **且** `tools.elevated` 允許清單授權
- 審批提示仍然適用（除非 `elevated=full`）

---

### /exec — 執行預設值

```
/exec                                    # 顯示當前設定
/exec host=<value> security=<value> ask=<value> node=<id>
```

**功能**：設定當前 session 的 exec 工具預設值（不寫入設定檔）。

**參數**：

| 參數 | 值 | 預設 | 說明 |
|------|-----|------|------|
| `host` | `sandbox` / `gateway` / `node` | `sandbox` | 命令執行位置 |
| `security` | `deny` / `allowlist` / `full` | `deny`（sandbox）/ `allowlist`（其他） | 安全策略 |
| `ask` | `off` / `on-miss` / `always` | `on-miss` | 審批模式 |
| `node` | 節點 ID/名稱 | 未設定 | 目標節點（host=node 時） |

**Host 說明**：
- **sandbox**：在隔離容器中執行
- **gateway**：在 gateway 主機上執行（需審批）
- **node**：在已配對的節點上執行（macOS app 或 headless host）

**範例**：
```
/exec                                    # 查看當前設定
/exec host=gateway                       # 切換到 gateway 執行
/exec host=gateway security=full ask=off # 完全權限，無審批
/exec host=node node=mac-mini            # 在特定節點執行
```

---

### /approve — 審批執行請求

```
/approve <id> allow-once
/approve <id> allow-always
/approve <id> deny
```

**功能**：回應 exec 工具的審批請求。

| 動作 | 行為 |
|------|------|
| `allow-once` | 僅核准此次執行 |
| `allow-always` | 核准並加入永久允許清單 |
| `deny` | 拒絕執行 |

**審批儲存**：`~/.openclaw/exec-approvals.json`

---

## 6. 子代理管理

### /subagents — 管理子代理

```
/subagents list
/subagents kill <id|#|all>
/subagents log <id|#> [limit] [tools]
/subagents info <id|#>
/subagents send <id|#> <message>
/subagents steer <id|#> <message>
/subagents spawn <agentId> <task> [--model <model>] [--thinking <level>]
```

**動作說明**：

| 動作 | 說明 |
|------|------|
| `list` | 列出所有從當前 session 生成的子代理 |
| `kill <id\|#\|all>` | 停止特定或所有子代理（級聯至子層） |
| `log <id\|#> [limit] [tools]` | 查看完成後的輸出詳情 |
| `info <id\|#>` | 顯示 run metadata（狀態、時間、session ID、transcript 路徑） |
| `send <id\|#> <message>` | 向執行中的子代理發送訊息 |
| `steer <id\|#> <message>` | 向執行中的子代理發送引導指令 |
| `spawn <agentId> <task>` | 啟動背景子代理（非阻塞，立即返回 run ID） |

**Spawn 選項**：
- `--model <provider/model>`：覆寫模型
- `--thinking <level>`：覆寫推理等級

**目標格式**：
- Run ID（完整 UUID）
- `#<number>`（索引編號，如 `#1`）
- `all`（僅限 kill）

**範例**：
```
/subagents list
/subagents spawn main "分析 src/ 目錄的架構" --model anthropic/claude-opus-4-6
/subagents steer #1 "請專注於安全相關的程式碼"
/subagents kill #1
/subagents kill all
/subagents log #1
```

---

### /focus — 綁定 Thread

```
/focus <target>
```

**功能**：將當前 thread/topic 綁定至特定的 session 目標（Discord/Telegram）。

**目標格式**：子代理 label/index、session key/id/label

**需求**：thread bindings 必須啟用。

---

### /unfocus — 解除綁定

```
/unfocus
```

**功能**：移除當前 thread/topic 的 session 綁定。

---

### /kill — 終止子代理

```
/kill <id|#|all>
```

**功能**：立即終止指定的子代理（無確認提示）。

**行為**：
- 立即中止
- 級聯至巢狀子層
- 確定性指令（不經過模型）

---

### /steer — 引導子代理

```
/steer <id|#> <message>
/tell <id|#> <message>     # 別名
```

**別名**：`/tell`

**功能**：向執行中的子代理發送引導指令。

**行為**：
- 可能注入到進行中的 run
- 否則中止當前工作並以 steer 訊息重啟

---

## 7. ACP 代理控制

### /acp — ACP Session 管理

ACP（Agent Control Protocol）允許與外部 agent 後端互動。

```
/acp spawn <harness> [options]
/acp cancel [session-target]
/acp steer [--session target] <message>
/acp close [session-target]
/acp status [session-target]
/acp set-mode <mode> [session-target]
/acp set <key> <value> [session-target]
/acp cwd <path> [session-target]
/acp permissions <profile> [session-target]
/acp timeout <seconds> [session-target]
/acp model <provider/model> [session-target]
/acp reset-options [session-target]
/acp sessions
/acp doctor
/acp install
/acp help
```

**Spawn 選項**：
- `--mode persistent|oneshot`
- `--bind here|off`（綁定當前對話）
- `--thread auto|here|off`（建立/綁定 thread）
- `--cwd <path>`（工作目錄）
- `--label <name>`（標籤）

**內建 harness 別名**：
`claude`、`codex`、`copilot`、`cursor`、`droid`、`gemini`、`iflow`、`kilocode`、`kimi`、`kiro`、`openclaw`、`opencode`、`pi`、`qwen`

**範例**：
```
/acp spawn codex --bind here --cwd /home/user/project
/acp steer 專注在效能優化
/acp status
/acp model anthropic/claude-opus-4-6
/acp close
/acp doctor
```

**Session target 解析順序**：
1. 明確的 target 參數
2. 當前 thread 綁定
3. 當前 requester session fallback

---

## 8. 訊息佇列控制

### /queue — 佇列設定

```
/queue <mode>
/queue <mode> debounce:<time> cap:<number> drop:<policy>
/queue default              # 重設為預設
/queue reset                # 同上
```

**模式**：

| 模式 | 行為 |
|------|------|
| `steer` | 立即注入當前 run（取消下一個工具邊界後的待處理工具呼叫） |
| `interrupt` | 中止活動 run，執行最新訊息（legacy） |
| `followup` | 排入下一個 agent turn |
| `collect`（預設） | 合併所有排隊訊息為單一 followup turn |
| `steer-backlog` | 立即 steer **且** 保留訊息供 followup（可能重複回覆） |

**選項**（適用於 followup/collect/steer-backlog）：

| 選項 | 格式 | 預設 | 說明 |
|------|------|------|------|
| `debounce` | `500ms` / `2s` | `1000ms` | 靜默等待時間 |
| `cap` | 數字 | `20` | 每 session 最大排隊數 |
| `drop` | `old` / `new` / `summarize` | `summarize` | 溢出策略 |

**Drop 策略**：
- **old**：丟棄最舊的訊息
- **new**：丟棄最新的訊息
- **summarize**：保留已丟棄訊息的項目符號摘要

**範例**：
```
/queue steer                             # 切換到 steer 模式
/queue collect debounce:2s cap:25        # 收集模式，2 秒 debounce
/queue followup drop:old                 # followup 模式，丟棄最舊
/queue default                           # 重設為預設
```

---

## 9. TTS 語音合成

### /tts — 文字轉語音控制

```
/tts off                    # 停用 TTS
/tts always                 # 總是啟用（別名：/tts on）
/tts inbound                # 僅在收到語音訊息後啟用
/tts tagged                 # 僅在回覆含 [[tts]] 標籤時啟用
/tts status                 # 顯示當前設定
/tts provider [id]          # 顯示/設定語音提供者
/tts limit <chars>          # 設定最大字元數
/tts summary off|on         # 切換長文自動摘要
/tts audio <text>           # 從自訂文字生成 TTS（一次性）
/tts help                   # 顯示使用指南
```

**提供者**：

| 提供者 | 需要 API Key | 說明 |
|--------|-------------|------|
| `elevenlabs` | 是 | 全功能，高品質語音 |
| `microsoft` | 否 | Microsoft Edge 神經 TTS |
| `openai` | 是 | OpenAI TTS API |

**輸出格式**（依頻道自動選擇）：
- **Telegram/WhatsApp**：Opus 語音訊息（48kHz/64kbps）
- **其他頻道**：MP3（44.1kHz/128kbps）

**範例**：
```
/tts always                              # 啟用所有回覆的語音
/tts provider elevenlabs                 # 切換到 ElevenLabs
/tts audio 你好，這是測試語音            # 一次性語音生成
/tts limit 500                           # 限制最大 500 字元
/tts summary on                          # 長文自動摘要
```

---

## 10. 設定與管理

### /config — 設定管理

```
/config                     # 顯示設定
/config show                # 同上
/config get <path>          # 取得特定值
/config set <path> <value>  # 設定值
/config unset <path>        # 移除值
```

**⚠️ 需要**：`commands.config: true`（預設停用）；僅限 owner

**範例**：
```
/config get agents.defaults.model
/config set agents.defaults.thinking high
/config unset channels.telegram.streaming
```

---

### /mcp — MCP Server 管理

```
/mcp                        # 顯示所有 MCP servers
/mcp show                   # 同上
/mcp get <name>             # 取得特定 server 設定
/mcp set <name> <json>      # 設定 server
/mcp unset <name>           # 移除 server
```

**⚠️ 需要**：`commands.mcp: true`（預設停用）；僅限 owner

---

### /plugins — 插件管理

```
/plugins                    # 列出插件
/plugins list               # 同上
/plugins show <id>          # 顯示插件詳情
/plugins enable <id>        # 啟用插件
/plugins disable <id>       # 停用插件
/plugin <id>                # 別名
```

**⚠️ 需要**：`commands.plugins: true`（預設停用）；寫入操作僅限 owner

---

### /debug — 除錯覆寫

```
/debug                      # 顯示當前覆寫
/debug show                 # 同上
/debug set <path> <value>   # 設定 runtime 覆寫
/debug unset <path>         # 移除覆寫
/debug reset                # 清除所有覆寫
```

**⚠️ 需要**：`commands.debug: true`（預設停用）；僅限 owner

**注意**：僅存在 runtime 記憶體中，不持久化至設定檔。

---

### /allowlist — 允許清單管理

```
/allowlist                  # 列出當前允許清單
/allowlist add <entry>      # 新增條目
/allowlist remove <entry>   # 移除條目
```

**⚠️ 需要**：`commands.config: true`

---

### /send — 發送政策

```
/send on                    # 啟用發送
/send off                   # 停用發送
/send inherit               # 繼承上層設定
```

**功能**：控制當前 session 的訊息發送政策。

**⚠️ 僅限 owner**

---

### /activation — 群組啟動模式

```
/activation mention          # 需要提及 bot 才啟動
/activation always           # 總是啟動
```

**功能**：控制群組訊息的啟動條件。

---

### /restart — 重啟 Gateway

```
/restart
```

**功能**：重啟 OpenClaw Gateway。

**注意**：預設啟用，設定 `commands.restart: false` 可停用。

---

## 11. 頻道切換（Dock）

### /dock-\<provider\> — 切換回覆頻道

```
/dock-telegram               # 回覆切換至 Telegram
/dock-discord                # 回覆切換至 Discord
/dock-slack                  # 回覆切換至 Slack
/dock_whatsapp               # 底線變體也支援
```

**功能**：將回覆路由至特定的頻道/提供者。

**行為**：
- 根據已設定的頻道插件動態生成
- 下劃線和連字號變體均支援

---

## 12. TUI 專屬操作

以下指令僅在 TUI 中可用，由 TUI 客戶端直接處理：

### /agent — 切換 Agent

```
/agent                      # 開啟 agent 選擇器
/agent <id>                 # 切換至指定 agent
```

### /agents — Agent 列表

```
/agents                     # 開啟 agent 選擇器
```

### /session — 切換 Session

```
/session                    # 開啟 session 選擇器
/session <key>              # 切換至指定 session
```

### /sessions — Session 列表

```
/sessions                   # 開啟可篩選的 session 列表
```

### /models — 模型列表

```
/models                     # 開啟可搜尋的模型列表
```

### /settings — 設定面板

```
/settings                   # 開啟 TUI 設定面板
```

可調整項目：
- 工具輸出（展開/收合）
- 推理顯示（開/關）

### /exit 或 /quit — 離開 TUI

```
/exit
/quit
```

---

## 13. 指令語法規則

### 基本規則

1. 指令以 `/` 開頭，作為**獨立訊息**發送
2. 冒號分隔可選：`/think: high` 等同 `/think high`
3. 參數以空格分隔
4. 鍵值參數格式：`key=value`（如 `/exec host=gateway`）

### 內聯指令（Inline）

以下指令可嵌入一般訊息中，執行後從訊息中剝離（模型不會看到）：

- `/help`、`/commands`、`/status`、`/whoami`（僅限授權發送者）

### 指令提示（Directives）

以下指令既可獨立發送（持久化設定），也可嵌入訊息（不持久化）：

- `/think`、`/fast`、`/verbose`、`/reasoning`、`/elevated`、`/exec`、`/model`、`/queue`

**獨立發送**：設定持久化至 session
**嵌入訊息**：僅影響該次回覆，不持久化

### 原生指令表面

| 頻道 | 支援方式 |
|------|---------|
| Discord | 完整原生指令註冊 |
| Telegram | 完整原生指令註冊 |
| Slack | 有限（Slack 管理斜線指令，文字回退） |
| 其他（Signal、WhatsApp 等） | 文字解析回退 |

---

## 14. 權限與授權

### 授權模型

- 指令遵從 `commands.allowFrom` 設定
- 回退至頻道 allowlists / pairing + `commands.useAccessGroups`
- 未授權的發送者的指令會被靜默忽略

### Owner-only 指令

以下指令僅限 owner 使用：
- `/config`、`/mcp`、`/plugins`（寫入）、`/debug`
- `/send`、`/allowlist`

### 需要明確啟用的指令

| 指令 | 設定旗標 | 預設 |
|------|---------|------|
| `/bash` | `commands.bash` | `false` |
| `/config` | `commands.config` | `false` |
| `/mcp` | `commands.mcp` | `false` |
| `/plugins` | `commands.plugins` | `false` |
| `/debug` | `commands.debug` | `false` |
| `/restart` | `commands.restart` | `true` |

---

## 15. 設定旗標參考

```json5
{
  commands: {
    text: true,              // 啟用文字指令解析（預設 true）
    native: "auto",          // 註冊原生指令（auto/true/false）
    bash: false,             // 啟用 /bash（預設 false）
    bashForegroundMs: 2000,  // /bash 前景等待時間（ms）
    config: false,           // 啟用 /config（預設 false）
    mcp: false,              // 啟用 /mcp（預設 false）
    plugins: false,          // 啟用 /plugins（預設 false）
    debug: false,            // 啟用 /debug（預設 false）
    restart: true,           // 啟用 /restart（預設 true）
    allowFrom: [],           // 指令允許的發送者
    useAccessGroups: false,  // 使用存取群組
  },
}
```

---

*本文件基於 OpenClaw `2026.3.31` 版本撰寫。完整英文文件請參考 https://docs.openclaw.ai/tools/slash-commands*
