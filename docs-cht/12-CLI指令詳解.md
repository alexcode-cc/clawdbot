# CLI 指令詳解

> 版本：`2026.3.23` | Node.js `>=22.16.0`
>
> 本文件涵蓋 OpenClaw CLI 的所有指令、選項和實際操作情境範例，幫助使用者從零開始安裝、設定和操作 OpenClaw 平台。

---

## 目錄

1. [安裝與快速開始](#1-安裝與快速開始)
2. [全域選項](#2-全域選項)
3. [初始設定指令](#3-初始設定指令)
4. [Gateway 指令](#4-gateway-指令)
5. [Agent 指令](#5-agent-指令)
6. [訊息指令](#6-訊息指令)
7. [頻道指令](#7-頻道指令)
8. [模型指令](#8-模型指令)
9. [配置指令](#9-配置指令)
10. [會話指令](#10-會話指令)
11. [記憶指令](#11-記憶指令)
12. [排程指令](#12-排程指令)
13. [瀏覽器指令](#13-瀏覽器指令)
14. [插件指令](#14-插件指令)
15. [節點指令](#15-節點指令)
16. [安全與維運指令](#16-安全與維運指令)
17. [其他指令](#17-其他指令)
18. [情境操作範例](#18-情境操作範例)

---

## 1. 安裝與快速開始

### 安裝方式

```bash
# 方式一：npm 全域安裝（推薦）
npm install -g openclaw

# 方式二：從原始碼建置
git clone https://github.com/openclaw/openclaw.git
cd openclaw
pnpm install
pnpm build

# 方式三：Docker
docker pull openclaw/openclaw
```

### 驗證安裝

```bash
openclaw --version
# 應顯示 2026.3.23 或更新版本
```

### 快速啟動流程

```bash
# 1. 執行引導精靈（互動式）
openclaw onboard

# 2. 啟動 Gateway
openclaw gateway run

# 3. 開啟終端介面
openclaw tui

# 4. 查看狀態
openclaw status
```

---

## 2. 全域選項

所有指令皆可使用以下全域選項：

| 選項 | 說明 |
|------|------|
| `--dev` | 開發模式：隔離狀態至 `~/.openclaw-dev`，預設 Gateway port 19001 |
| `--profile <name>` | 使用命名設定檔，隔離至 `~/.openclaw-<name>` |
| `--log-level <level>` | 全域日誌等級（`silent`/`fatal`/`error`/`warn`/`info`/`debug`/`trace`） |
| `--no-color` | 停用 ANSI 色彩輸出 |
| `-V, --version` | 顯示版本號 |
| `-h, --help` | 顯示幫助資訊 |

**範例**：

```bash
# 使用開發模式（隔離環境）
openclaw --dev gateway run

# 使用自訂設定檔
openclaw --profile staging status

# 除錯模式
openclaw --log-level debug agent --message "test"
```

---

## 3. 初始設定指令

### `openclaw onboard` — 互動式引導精靈

設定 Gateway、工作區和技能的完整引導流程。

```
openclaw onboard [選項]
```

**主要選項**：

| 選項 | 說明 |
|------|------|
| `--mode <mode>` | 模式：`local`（本地 Gateway）或 `remote`（遠端 Gateway） |
| `--flow <flow>` | 流程：`quickstart`（快速）、`advanced`（進階）、`manual`（手動） |
| `--non-interactive` | 非互動式執行（需搭配其他選項） |
| `--accept-risk` | 確認已瞭解 Agent 完整系統存取風險（非互動式必要） |
| `--install-daemon` | 安裝 Gateway 服務（launchd/systemd/schtasks） |
| `--no-install-daemon` | 跳過服務安裝 |
| `--skip-channels` | 跳過頻道設定 |
| `--skip-skills` | 跳過技能設定 |
| `--skip-search` | 跳過搜尋提供者設定 |
| `--skip-health` | 跳過健康檢查 |
| `--reset` | 執行前重設配置 |
| `--reset-scope <scope>` | 重設範圍：`config`、`config+creds+sessions`、`full` |
| `--json` | 輸出 JSON 摘要 |

**認證選項**（`--auth-choice` 搭配對應 API Key）：

| 選項 | 說明 |
|------|------|
| `--auth-choice <choice>` | 認證方式（見下方清單） |
| `--anthropic-api-key <key>` | Anthropic API Key |
| `--openai-api-key <key>` | OpenAI API Key |
| `--gemini-api-key <key>` | Google Gemini API Key |
| `--openrouter-api-key <key>` | OpenRouter API Key |
| `--ollama` | 使用 Ollama（`--auth-choice ollama`） |
| `--custom-base-url <url>` | 自訂提供者 Base URL |
| `--custom-api-key <key>` | 自訂提供者 API Key |
| `--custom-model-id <id>` | 自訂模型 ID |
| `--custom-compatibility <mode>` | API 相容模式：`openai` 或 `anthropic` |

**Gateway 選項**：

| 選項 | 說明 |
|------|------|
| `--gateway-port <port>` | Gateway 埠號 |
| `--gateway-bind <mode>` | 綁定模式：`loopback`/`lan`/`tailnet`/`auto`/`custom` |
| `--gateway-auth <mode>` | 認證模式：`token` 或 `password` |
| `--gateway-token <token>` | Gateway Token |
| `--gateway-password <password>` | Gateway 密碼 |
| `--secret-input-mode <mode>` | API Key 持久化模式：`plaintext` 或 `ref`（環境變數參考） |

**情境範例**：

```bash
# 互動式快速引導
openclaw onboard

# 使用 Anthropic API Key 非互動式設定
openclaw onboard --non-interactive --accept-risk \
  --auth-choice apiKey \
  --anthropic-api-key "sk-ant-..." \
  --gateway-bind loopback \
  --install-daemon

# 使用 Ollama 本地模型
openclaw onboard --non-interactive --accept-risk \
  --auth-choice ollama \
  --install-daemon

# 連接遠端 Gateway
openclaw onboard --mode remote \
  --remote-url "wss://my-gateway.example.com" \
  --remote-token "my-token"

# 使用環境變數參考存儲 API Key
openclaw onboard --non-interactive --accept-risk \
  --auth-choice apiKey \
  --anthropic-api-key "$ANTHROPIC_API_KEY" \
  --secret-input-mode ref
```

---

### `openclaw setup` — 初始化配置

建立初始配置檔和工作區。

```
openclaw setup [選項]
```

| 選項 | 說明 |
|------|------|
| `--mode <mode>` | 模式：`local` 或 `remote` |
| `--workspace <dir>` | 工作區目錄（預設：`~/.openclaw/workspace`） |
| `--remote-url <url>` | 遠端 Gateway URL |
| `--remote-token <token>` | 遠端 Gateway Token |
| `--non-interactive` | 非互動式 |
| `--wizard` | 執行互動式引導精靈 |

---

### `openclaw configure` — 互動式設定精靈

針對特定部分執行互動式設定。

```
openclaw configure [選項]
```

| 選項 | 說明 |
|------|------|
| `--section <section>` | 設定區段（可重複）：`workspace`/`model`/`web`/`gateway`/`daemon`/`channels`/`skills`/`health` |

```bash
# 只設定模型
openclaw configure --section model

# 設定頻道和 Gateway
openclaw configure --section channels --section gateway
```

---

### `openclaw reset` — 重設本地配置

```
openclaw reset [選項]
```

| 選項 | 說明 |
|------|------|
| `--scope <scope>` | 範圍：`config`/`config+creds+sessions`/`full` |
| `--dry-run` | 預覽操作，不實際刪除 |
| `--non-interactive` | 非互動式（需搭配 `--scope` 和 `--yes`） |
| `--yes` | 跳過確認提示 |

```bash
# 互動式重設
openclaw reset

# 僅重設配置（保留 sessions）
openclaw reset --scope config --yes

# 預覽完整重設
openclaw reset --scope full --dry-run
```

---

### `openclaw uninstall` — 解除安裝

解除安裝 Gateway 服務和本地資料（保留 CLI 本身）。

```bash
openclaw uninstall
```

---

## 4. Gateway 指令

Gateway 是 OpenClaw 的核心伺服器，所有客戶端透過 WebSocket 與其通訊。

### `openclaw gateway run` — 前景執行 Gateway

```
openclaw gateway run [選項]
```

| 選項 | 說明 |
|------|------|
| `--port <port>` | 埠號（預設 18789） |
| `--bind <mode>` | 綁定模式：`loopback`/`lan`/`tailnet`/`auto`/`custom` |
| `--auth <mode>` | 認證模式：`none`/`token`/`password`/`trusted-proxy` |
| `--token <token>` | 共享 Token（也讀取 `OPENCLAW_GATEWAY_TOKEN` 環境變數） |
| `--password <password>` | 密碼 |
| `--password-file <path>` | 從檔案讀取密碼 |
| `--force` | 強制啟動（殺死佔用 port 的進程） |
| `--dev` | 建立 dev 配置和工作區 |
| `--allow-unconfigured` | 允許未設定 `gateway.mode=local` 時啟動 |
| `--verbose` | 詳細日誌輸出 |
| `--compact` | 精簡 WebSocket 日誌（`--ws-log compact` 別名） |
| `--ws-log <style>` | WebSocket 日誌風格：`auto`/`full`/`compact` |
| `--raw-stream` | 記錄原始模型串流事件至 JSONL |
| `--raw-stream-path <path>` | JSONL 路徑 |
| `--tailscale <mode>` | Tailscale 模式：`off`/`serve`/`funnel` |
| `--tailscale-reset-on-exit` | 關閉時重設 Tailscale 設定 |
| `--claude-cli-logs` | 僅顯示 claude-cli 日誌 |
| `--reset` | 重設 dev 配置（需搭配 `--dev`） |

```bash
# 基本啟動
openclaw gateway run

# 區域網路存取，使用 token 認證
openclaw gateway run --bind lan --token "my-secret-token"

# 指定埠號，強制啟動
openclaw gateway run --port 8080 --force

# 開發模式
openclaw gateway run --dev --verbose

# 使用密碼檔案
openclaw gateway run --auth password --password-file /etc/openclaw/password

# 搭配 Tailscale
openclaw gateway run --bind tailnet --tailscale serve
```

---

### `openclaw gateway install` — 安裝為系統服務

```
openclaw gateway install [選項]
```

| 選項 | 說明 |
|------|------|
| `--port <port>` | Gateway 埠號 |
| `--token <token>` | Gateway Token |
| `--runtime <runtime>` | 服務 runtime：`node` 或 `bun` |
| `--force` | 強制重新安裝 |
| `--json` | 輸出 JSON |

```bash
# 安裝服務（macOS: launchd, Linux: systemd, Windows: schtasks）
openclaw gateway install

# 指定 token 和 port
openclaw gateway install --port 18789 --token "my-token"

# 強制重新安裝
openclaw gateway install --force
```

---

### `openclaw gateway status` — 查看 Gateway 狀態

```
openclaw gateway status [選項]
```

| 選項 | 說明 |
|------|------|
| `--deep` | 掃描系統級服務 |
| `--json` | 輸出 JSON |
| `--no-probe` | 跳過 RPC 探測 |
| `--require-rpc` | RPC 探測失敗時 exit non-zero（自動化用） |
| `--timeout <ms>` | 逾時（預設 10000ms） |
| `--token <token>` | Gateway Token |
| `--password <password>` | Gateway 密碼 |
| `--url <url>` | Gateway WebSocket URL |

```bash
# 基本狀態
openclaw gateway status

# 深度掃描
openclaw gateway status --deep

# 自動化檢查（失敗時 exit 1）
openclaw gateway status --require-rpc

# JSON 輸出
openclaw gateway status --json
```

---

### 其他 Gateway 子指令

| 指令 | 說明 |
|------|------|
| `openclaw gateway start` | 啟動 Gateway 服務 |
| `openclaw gateway stop` | 停止 Gateway 服務 |
| `openclaw gateway restart` | 重啟 Gateway 服務 |
| `openclaw gateway uninstall` | 解除安裝 Gateway 服務 |
| `openclaw gateway health` | 取得 Gateway 健康狀態 |
| `openclaw gateway probe` | 顯示 reachability + discovery + health 摘要 |
| `openclaw gateway discover` | 透過 Bonjour 發現 Gateway |
| `openclaw gateway call` | 呼叫 Gateway RPC 方法 |
| `openclaw gateway usage-cost` | 取得 session 使用成本摘要 |

```bash
# 停止服務
openclaw gateway stop

# 重啟服務
openclaw gateway restart

# 發現區域網路上的 Gateway
openclaw gateway discover
```

---

## 5. Agent 指令

### `openclaw agent` — 執行單次 Agent Turn

透過 Gateway（或本地 embedded）執行一次 Agent 對話輪次。

```
openclaw agent [選項]
```

| 選項 | 說明 |
|------|------|
| `-m, --message <text>` | 訊息內容 |
| `-t, --to <number>` | 接收者（E.164 格式），用於推導 session key |
| `--agent <id>` | Agent ID |
| `--channel <channel>` | 交付頻道（`last`/`telegram`/`discord` 等） |
| `--deliver` | 將回覆發送至選定頻道 |
| `--local` | 使用本地 embedded agent（需要 API Key 在環境中） |
| `--thinking <level>` | Thinking 等級：`off`/`minimal`/`low`/`medium`/`high`/`xhigh` |
| `--timeout <seconds>` | 逾時秒數（預設 600） |
| `--session-id <id>` | 明確指定 session ID |
| `--json` | 輸出 JSON |
| `--reply-to <target>` | 交付目標覆寫 |
| `--reply-channel <channel>` | 交付頻道覆寫 |
| `--reply-account <id>` | 交付帳號 ID 覆寫 |
| `--verbose <on\|off>` | 持久化 agent verbose 等級 |

```bash
# 簡單問答
openclaw agent --message "今天天氣如何？"

# 發送至特定頻道
openclaw agent --message "每日報告" --deliver --channel telegram --to "@mychat"

# 使用本地 Agent（無需 Gateway）
openclaw agent --local --message "分析這段程式碼"

# 高層次思考模式
openclaw agent --message "設計資料庫架構" --thinking high

# JSON 輸出
openclaw agent --message "回傳 JSON" --json
```

---

### `openclaw agents` — 管理隔離 Agent

```
openclaw agents [子指令]
```

| 子指令 | 說明 |
|--------|------|
| `list` | 列出已配置的 Agents |
| `add` | 新增隔離 Agent |
| `delete` | 刪除 Agent 和清理工作區 |
| `bind` | 新增路由綁定 |
| `unbind` | 移除路由綁定 |
| `bindings` | 列出路由綁定 |
| `set-identity` | 更新 Agent 身份 |

```bash
# 列出所有 Agents
openclaw agents list

# 新增 Agent
openclaw agents add

# 設定路由綁定（將特定使用者路由至特定 Agent）
openclaw agents bind
```

---

## 6. 訊息指令

### `openclaw message send` — 發送訊息

```
openclaw message send [選項]
```

| 選項 | 說明 |
|------|------|
| `-m, --message <text>` | 訊息內容 |
| `-t, --target <dest>` | 目標：E.164（WhatsApp/Signal）、Telegram chatId/@username、Discord channel/user |
| `--channel <channel>` | 頻道名稱 |
| `--account <id>` | 頻道帳號 ID |
| `--media <path-or-url>` | 附件（圖片/音訊/影片/文件） |
| `--reply-to <id>` | 回覆訊息 ID |
| `--thread-id <id>` | Telegram forum thread ID |
| `--silent` | 靜音發送（Telegram + Discord） |
| `--buttons <json>` | Telegram inline keyboard 按鈕 JSON |
| `--components <json>` | Discord components payload JSON |
| `--card <json>` | Adaptive Card JSON（支援的頻道） |
| `--gif-playback` | 影片視為 GIF（僅 WhatsApp） |
| `--dry-run` | 預覽不發送 |
| `--json` | 輸出 JSON |
| `--verbose` | 詳細日誌 |

```bash
# 發送 Telegram 訊息
openclaw message send --channel telegram --target "@mychat" --message "你好！"

# 發送 WhatsApp 訊息（附件圖片）
openclaw message send --channel whatsapp --target "+886912345678" \
  --message "看看這張照片" --media /tmp/photo.jpg

# 發送 Discord 訊息至特定頻道
openclaw message send --channel discord --target "channel_id" \
  --message "自動通知"

# 靜音發送
openclaw message send --channel telegram --target "@mychat" \
  --message "深夜通知" --silent

# 預覽不實際發送
openclaw message send --channel telegram --target "@mychat" \
  --message "測試" --dry-run
```

### 其他訊息子指令

| 子指令 | 說明 |
|--------|------|
| `read` | 讀取最近訊息 |
| `edit` | 編輯訊息 |
| `delete` | 刪除訊息 |
| `broadcast` | 廣播訊息至多個目標 |
| `react` | 新增/移除反應表情 |
| `reactions` | 列出訊息反應 |
| `pin` / `unpin` | 釘選/取消釘選 |
| `pins` | 列出釘選訊息 |
| `poll` | 發送投票 |
| `search` | 搜尋 Discord 訊息 |
| `thread` | 串接操作 |
| `ban` / `kick` / `timeout` | 成員管理 |
| `member` / `role` / `permissions` | 角色與權限 |
| `voice` | 語音操作 |
| `emoji` / `sticker` | 表情/貼圖操作 |
| `event` / `channel` | 事件/頻道操作 |

---

## 7. 頻道指令

### `openclaw channels` — 管理通訊頻道

```
openclaw channels [子指令]
```

| 子指令 | 說明 |
|--------|------|
| `status` | 顯示頻道狀態 |
| `list` | 列出已配置的頻道和認證設定檔 |
| `add` | 新增或更新頻道帳號 |
| `remove` | 停用或刪除頻道帳號 |
| `login` | 連結頻道帳號 |
| `logout` | 登出頻道 |
| `logs` | 顯示最近的頻道日誌 |
| `resolve` | 解析頻道/使用者名稱至 ID |
| `capabilities` | 顯示提供者能力 |

```bash
# 查看所有頻道狀態
openclaw channels status

# 探測頻道憑證
openclaw channels status --probe

# JSON 輸出
openclaw channels status --json

# 列出已配置頻道
openclaw channels list

# 登入 WhatsApp（掃描 QR Code）
openclaw channels login --channel whatsapp

# 解析使用者名稱
openclaw channels resolve --channel telegram --query "@username"
```

---

### `openclaw status` — 綜合狀態

```
openclaw status [選項]
```

| 選項 | 說明 |
|------|------|
| `--all` | 完整診斷（唯讀，可貼上） |
| `--deep` | 探測頻道（WhatsApp/Telegram/Discord/Slack/Signal） |
| `--json` | 輸出 JSON |
| `--usage` | 顯示模型提供者使用量/配額 |
| `--verbose` | 詳細日誌 |
| `--timeout <ms>` | 探測逾時（預設 10000ms） |

```bash
# 快速狀態
openclaw status

# 完整診斷
openclaw status --all

# 探測所有頻道
openclaw status --deep

# 顯示使用量
openclaw status --usage
```

---

## 8. 模型指令

### `openclaw models` — 模型管理

```
openclaw models [子指令]
```

| 子指令 | 說明 |
|--------|------|
| `list` | 列出模型（預設顯示已配置） |
| `set` | 設定預設模型 |
| `set-image` | 設定圖像模型 |
| `status` | 顯示模型狀態 |
| `scan` | 掃描 OpenRouter 免費模型 |
| `aliases` | 管理模型別名 |
| `fallbacks` | 管理 fallback 模型清單 |
| `image-fallbacks` | 管理圖像 fallback 模型 |
| `auth` | 管理模型認證設定檔 |

```bash
# 列出已配置的模型
openclaw models list

# 列出所有可用模型
openclaw models list --all

# 按提供者篩選
openclaw models list --provider anthropic

# 僅本地模型
openclaw models list --local

# 設定預設模型
openclaw models set anthropic/claude-sonnet-4-5

# 查看模型狀態
openclaw models status

# JSON 輸出
openclaw models status --json
```

### 模型認證管理

```bash
# 互動式新增認證
openclaw models auth add

# 執行 OAuth 認證流程
openclaw models auth login

# GitHub Copilot 登入
openclaw models auth login-github-copilot

# 貼上 Token
openclaw models auth paste-token

# 管理認證順序
openclaw models auth order
```

---

## 9. 配置指令

### `openclaw config` — 非互動式配置操作

```
openclaw config [子指令]
```

| 子指令 | 說明 |
|--------|------|
| `get <path>` | 取得配置值（dot notation） |
| `set <path> <value>` | 設定配置值（JSON5 或字串） |
| `unset <path>` | 移除配置值 |
| `file` | 顯示配置檔案路徑 |
| `validate` | 驗證配置（不啟動 Gateway） |

```bash
# 取得值
openclaw config get gateway.port
openclaw config get agents.defaults.model

# 設定值
openclaw config set gateway.port 18789
openclaw config set agents.defaults.model "anthropic/claude-sonnet-4-5"

# 設定複雜值（JSON5）
openclaw config set channels.telegram.token "BOT_TOKEN"

# 驗證配置
openclaw config validate

# JSON 輸出驗證結果
openclaw config validate --json

# 查看配置檔路徑
openclaw config file

# 移除值
openclaw config unset channels.whatsapp
```

---

## 10. 會話指令

### `openclaw sessions` — 會話管理

```
openclaw sessions [選項]
```

| 選項 | 說明 |
|------|------|
| `--agent <id>` | 指定 Agent ID |
| `--all-agents` | 聚合所有 Agent 的會話 |
| `--active <minutes>` | 僅顯示最近 N 分鐘內更新的會話 |
| `--json` | 輸出 JSON |
| `--store <path>` | 指定 session store 路徑 |
| `--verbose` | 詳細日誌 |

| 子指令 | 說明 |
|--------|------|
| `cleanup` | 執行 session store 維護 |

```bash
# 列出所有會話
openclaw sessions

# 僅顯示特定 Agent 的會話
openclaw sessions --agent work

# 聚合所有 Agent
openclaw sessions --all-agents

# 最近 2 小時的活躍會話
openclaw sessions --active 120

# JSON 輸出
openclaw sessions --json

# 執行維護清理
openclaw sessions cleanup
```

---

## 11. 記憶指令

### `openclaw memory` — 記憶搜尋與索引

```
openclaw memory [子指令]
```

| 子指令 | 說明 |
|--------|------|
| `search [query]` | 搜尋記憶檔案 |
| `index` | 重新索引記憶檔案 |
| `status` | 顯示記憶索引狀態 |

```bash
# 搜尋記憶
openclaw memory search "會議紀錄"

# 限制結果數量
openclaw memory search --query "部署計畫" --max-results 10

# 查看索引狀態
openclaw memory status

# 深度探測嵌入提供者
openclaw memory status --deep

# 強制完整重新索引
openclaw memory index --force

# JSON 輸出
openclaw memory status --json
```

---

## 12. 排程指令

### `openclaw cron` — 排程任務管理

```
openclaw cron [子指令]
```

| 子指令 | 說明 |
|--------|------|
| `list` | 列出排程任務 |
| `add` | 新增排程任務 |
| `edit` | 編輯排程任務 |
| `rm` | 移除排程任務 |
| `enable` | 啟用任務 |
| `disable` | 停用任務 |
| `run` | 手動執行（除錯用） |
| `runs` | 顯示執行歷史 |
| `status` | 顯示排程器狀態 |

**`cron add` 主要選項**：

| 選項 | 說明 |
|------|------|
| `--name <name>` | 任務名稱 |
| `--cron <expr>` | Cron 表達式（5 或 6 欄位） |
| `--every <duration>` | 間隔執行（如 `10m`、`1h`） |
| `--at <when>` | 單次執行（ISO 時間或 `+20m`） |
| `--message <text>` | Agent 訊息 |
| `--agent <id>` | Agent ID |
| `--channel <channel>` | 交付頻道 |
| `--to <dest>` | 交付目標 |
| `--announce` | 將摘要公告至聊天 |
| `--session <target>` | Session 目標：`main` 或 `isolated` |
| `--tz <iana>` | 時區（IANA 格式） |
| `--thinking <level>` | Thinking 等級 |
| `--model <model>` | 模型覆寫 |
| `--light-context` | 使用輕量 bootstrap 上下文 |
| `--delete-after-run` | 單次任務完成後刪除 |

```bash
# 列出所有排程任務
openclaw cron list

# 包含停用的任務
openclaw cron list --all

# 新增每日報告
openclaw cron add --name "daily-report" \
  --cron "0 9 * * *" \
  --message "產生每日狀態報告" \
  --announce --channel telegram --to "@mychat" \
  --tz "Asia/Taipei"

# 每 30 分鐘執行
openclaw cron add --name "health-check" \
  --every 30m \
  --message "檢查系統狀態"

# 20 分鐘後單次執行
openclaw cron add --name "remind" \
  --at "+20m" \
  --message "提醒：開會時間到了" \
  --announce --channel discord --to "channel_id" \
  --delete-after-run

# 手動執行任務（除錯）
openclaw cron run --name "daily-report"

# 停用任務
openclaw cron disable --name "health-check"

# 查看執行歷史
openclaw cron runs
```

---

## 13. 瀏覽器指令

### `openclaw browser` — 瀏覽器自動化

OpenClaw 提供完整的瀏覽器控制能力，支援 Chrome/Chromium。

```
openclaw browser [子指令]
```

**主要子指令**：

| 子指令 | 說明 |
|--------|------|
| `start` | 啟動瀏覽器 |
| `stop` | 停止瀏覽器 |
| `status` | 瀏覽器狀態 |
| `tabs` | 列出開啟的分頁 |
| `open <url>` | 在新分頁開啟 URL |
| `navigate <url>` | 導航至 URL |
| `focus <id>` | 聚焦分頁 |
| `close [id]` | 關閉分頁 |
| `screenshot` | 截圖 |
| `snapshot` | 頁面快照（AI / ARIA） |
| `click <ref>` | 點擊元素 |
| `type <ref> <text>` | 輸入文字 |
| `press <key>` | 按鍵 |
| `hover <ref>` | 懸停 |
| `drag <from> <to>` | 拖曳 |
| `select <ref> <options>` | 選擇下拉選項 |
| `fill` | 填寫表單 |
| `scroll­intoview <ref>` | 捲動至元素 |
| `resize <w> <h>` | 調整視窗大小 |
| `pdf` | 頁面匯出 PDF |
| `console` | 取得 console 訊息 |
| `errors` | 取得頁面錯誤 |
| `requests` | 取得網路請求 |
| `cookies` | 讀寫 cookies |
| `storage` | 讀寫 localStorage/sessionStorage |
| `evaluate` | 執行 JavaScript |
| `wait` | 等待條件 |
| `upload` | 檔案上傳 |
| `download` | 下載檔案 |
| `dialog` | 處理對話框 |
| `trace` | 錄製 Playwright trace |

**Profile 管理子指令**：

| 子指令 | 說明 |
|--------|------|
| `profiles` | 列出瀏覽器設定檔 |
| `create-profile` | 建立設定檔 |
| `delete-profile` | 刪除設定檔 |
| `reset-profile` | 重設設定檔 |

```bash
# 啟動瀏覽器
openclaw browser start

# 開啟網頁
openclaw browser open https://example.com

# 截圖
openclaw browser screenshot
openclaw browser screenshot --full-page

# 頁面快照（AI 格式）
openclaw browser snapshot
openclaw browser snapshot --format aria --limit 200

# 點擊元素
openclaw browser click 12 --double

# 輸入文字並送出
openclaw browser type 23 "hello" --submit

# 填寫表單
openclaw browser fill --fields '[{"ref":"1","value":"Ada"}]'

# 等待條件
openclaw browser wait --text "載入完成"

# 匯出 PDF
openclaw browser pdf

# 停止瀏覽器
openclaw browser stop
```

---

## 14. 插件指令

### `openclaw plugins` — 插件管理

```
openclaw plugins [子指令]
```

| 子指令 | 說明 |
|--------|------|
| `list` | 列出已發現的插件 |
| `install <path-or-spec>` | 安裝插件（路徑、封存檔或 npm spec） |
| `uninstall` | 解除安裝插件 |
| `update` | 更新已安裝的插件 |
| `enable` | 啟用插件 |
| `disable` | 停用插件 |
| `info` | 顯示插件詳情 |
| `doctor` | 報告插件載入問題 |

**`plugins install` 選項**：

| 選項 | 說明 |
|------|------|
| `-l, --link` | 連結本地路徑（不複製） |
| `--pin` | 記錄 npm 安裝為精確版本 |

```bash
# 列出所有插件
openclaw plugins list

# 從 npm 安裝
openclaw plugins install @openclaw/voice-call

# 從本地路徑安裝
openclaw plugins install ./my-plugin

# 連結開發中的插件
openclaw plugins install -l ./my-plugin

# 更新所有插件
openclaw plugins update

# 插件健康檢查
openclaw plugins doctor

# 啟用/停用
openclaw plugins enable my-plugin
openclaw plugins disable my-plugin
```

---

## 15. 節點指令

### `openclaw nodes` — 節點管理

管理 Gateway 擁有的節點配對和節點命令。

```
openclaw nodes [子指令]
```

| 子指令 | 說明 |
|--------|------|
| `list` | 列出待配對和已配對的節點 |
| `status` | 列出已知節點及連線狀態 |
| `pending` | 列出待配對請求 |
| `approve` | 批准待配對請求 |
| `reject` | 拒絕待配對請求 |
| `rename` | 重新命名節點 |
| `describe` | 描述節點能力 |
| `invoke` | 在節點上執行命令 |
| `run` | 在節點上執行 shell 命令（僅 mac） |
| `camera` | 從節點擷取相機媒體 |
| `screen` | 從節點錄製螢幕 |
| `location` | 從節點取得位置 |
| `canvas` | 擷取或渲染 canvas 內容 |
| `notify` | 在節點上發送通知（僅 mac） |
| `push` | 發送 APNs 測試推播至 iOS 節點 |

```bash
# 查看節點狀態
openclaw nodes status

# 列出待配對請求
openclaw nodes pending

# 批准配對
openclaw nodes approve

# 在 Mac 節點上執行命令
openclaw nodes run --node <id> --raw "uname -a"

# 拍照
openclaw nodes camera snap --node <id>
```

### `openclaw node` — Headless Node Host

在伺服器上執行無頭節點服務。

```
openclaw node [子指令]
```

| 子指令 | 說明 |
|--------|------|
| `run` | 前景執行 node host |
| `install` | 安裝 node host 服務 |
| `stop` | 停止服務 |
| `restart` | 重啟服務 |
| `status` | 查看狀態 |
| `uninstall` | 解除安裝 |

```bash
# 執行 node host
openclaw node run --host 127.0.0.1 --port 18789

# 安裝為服務
openclaw node install
```

### `openclaw qr` — 生成 iOS 配對 QR Code

```
openclaw qr [選項]
```

| 選項 | 說明 |
|------|------|
| `--json` | 輸出 JSON |
| `--no-ascii` | 跳過 ASCII QR 渲染 |
| `--setup-code-only` | 僅輸出 setup code |
| `--remote` | 使用 `gateway.remote.url` |
| `--public-url <url>` | 覆寫 Gateway 公開 URL |
| `--token <token>` | 覆寫 Gateway Token |

```bash
# 生成配對 QR Code
openclaw qr

# 僅顯示 setup code
openclaw qr --setup-code-only

# 遠端模式
openclaw qr --remote
```

---

## 16. 安全與維運指令

### `openclaw doctor` — 健康檢查與修復

```
openclaw doctor [選項]
```

| 選項 | 說明 |
|------|------|
| `--fix` | 套用建議修復 |
| `--force` | 積極修復（覆寫自訂服務配置） |
| `--deep` | 掃描系統服務 |
| `--non-interactive` | 非互動式（僅安全遷移） |
| `--generate-gateway-token` | 生成並配置 Gateway Token |
| `--no-workspace-suggestions` | 停用工作區建議 |
| `--yes` | 接受預設值 |

```bash
# 執行健康檢查
openclaw doctor

# 自動修復
openclaw doctor --fix

# 深度掃描 + 修復
openclaw doctor --deep --fix

# 生成 Gateway Token
openclaw doctor --generate-gateway-token
```

---

### `openclaw security` — 安全審計

```
openclaw security audit [選項]
```

| 選項 | 說明 |
|------|------|
| `--deep` | 包含 Gateway 即時探測 |
| `--fix` | 套用安全修復（檔案權限等） |
| `--json` | 輸出 JSON |

```bash
# 執行安全審計
openclaw security audit

# 深度審計（含 Gateway 探測）
openclaw security audit --deep

# 自動修復
openclaw security audit --fix
```

---

### `openclaw backup` — 備份管理

```
openclaw backup [子指令]
```

| 子指令 | 說明 |
|--------|------|
| `create` | 建立備份封存（config、credentials、sessions、workspaces） |
| `verify` | 驗證備份封存及其 manifest |

```bash
# 建立備份
openclaw backup create

# 驗證備份
openclaw backup verify
```

---

### `openclaw secrets` — 密鑰管理

```
openclaw secrets [子指令]
```

| 子指令 | 說明 |
|--------|------|
| `reload` | 重新解析 secret references 並原子交換 |
| `audit` | 審計明文密鑰、未解析的 refs 和優先級漂移 |
| `apply` | 套用先前生成的 secrets 計畫 |
| `configure` | 互動式 secrets 設定 |

```bash
# 重新載入 secrets
openclaw secrets reload

# 審計 secrets
openclaw secrets audit

# 互動式設定
openclaw secrets configure
```

---

### `openclaw approvals` — 執行審批管理

```
openclaw approvals [子指令]
```

| 子指令 | 說明 |
|--------|------|
| `get` | 取得 exec approvals 快照 |
| `set` | 用 JSON 檔案替換 exec approvals |
| `allowlist` | 編輯 per-agent allowlist |

```bash
# 查看目前的 exec approvals
openclaw approvals get

# 編輯 allowlist
openclaw approvals allowlist
```

---

### `openclaw update` — 更新管理

```
openclaw update [選項]
```

| 選項 | 說明 |
|------|------|
| `--channel <channel>` | 更新頻道：`stable`/`beta`/`dev` |
| `--tag <dist-tag\|version>` | 覆寫 npm dist-tag 或版本 |
| `--dry-run` | 預覽操作不執行 |
| `--no-restart` | 更新後不重啟 Gateway |
| `--yes` | 跳過確認 |
| `--timeout <seconds>` | 逾時秒數（預設 1200） |
| `--json` | 輸出 JSON |

| 子指令 | 說明 |
|--------|------|
| `status` | 顯示更新頻道和版本狀態 |
| `wizard` | 互動式更新精靈 |

```bash
# 更新至最新穩定版
openclaw update

# 切換至 beta 頻道
openclaw update --channel beta

# 預覽更新
openclaw update --dry-run

# 非互動式更新
openclaw update --yes

# 查看更新狀態
openclaw update status
```

---

## 17. 其他指令

### `openclaw tui` — 終端介面

```
openclaw tui [選項]
```

| 選項 | 說明 |
|------|------|
| `--message <text>` | 連接後自動發送訊息 |
| `--session <key>` | Session key（預設 `main`） |
| `--token <token>` | Gateway Token |
| `--password <password>` | Gateway 密碼 |
| `--url <url>` | Gateway URL |
| `--deliver` | 交付 assistant 回覆 |
| `--history-limit <n>` | 載入歷史筆數（預設 200） |
| `--thinking <level>` | Thinking 等級覆寫 |
| `--timeout-ms <ms>` | Agent 逾時（ms） |

```bash
# 開啟 TUI
openclaw tui

# 連接並自動發送訊息
openclaw tui --message "開始工作"

# 連接遠端 Gateway
openclaw tui --url "wss://remote.example.com" --token "my-token"
```

---

### `openclaw dashboard` — 開啟 Control UI

```bash
# 在瀏覽器中開啟 Dashboard
openclaw dashboard

# 僅列印 URL
openclaw dashboard --no-open
```

---

### `openclaw hooks` — Hooks 管理

```
openclaw hooks [子指令]
```

| 子指令 | 說明 |
|--------|------|
| `list` | 列出所有 hooks |
| `info` | 顯示 hook 詳情 |
| `enable` | 啟用 hook |
| `disable` | 停用 hook |
| `install` | 安裝 hook pack |
| `update` | 更新已安裝的 hooks |
| `check` | 檢查 hooks 資格狀態 |

---

### `openclaw skills` — 技能管理

```
openclaw skills [子指令]
```

| 子指令 | 說明 |
|--------|------|
| `list` | 列出所有可用技能 |
| `info` | 顯示技能詳情 |
| `check` | 檢查哪些技能已就緒/缺少需求 |

```bash
# 列出技能
openclaw skills list

# 檢查技能就緒狀態
openclaw skills check
```

---

### `openclaw sandbox` — 沙箱管理

```
openclaw sandbox [子指令]
```

| 子指令 | 說明 |
|--------|------|
| `list` | 列出沙箱容器及狀態 |
| `recreate` | 移除容器以強制重建 |
| `explain` | 解釋有效的 sandbox/tool 策略 |

```bash
# 列出沙箱容器
openclaw sandbox list

# 重建所有容器
openclaw sandbox recreate --all

# 解釋有效策略
openclaw sandbox explain
```

---

### `openclaw logs` — 日誌查看

```
openclaw logs [選項]
```

| 選項 | 說明 |
|------|------|
| `--follow` | 持續追蹤日誌 |
| `--limit <n>` | 最大行數（預設 200） |
| `--json` | 輸出 JSON 日誌行 |
| `--local-time` | 顯示本地時區 |
| `--plain` | 純文字輸出 |
| `--interval <ms>` | 輪詢間隔 |

```bash
# 查看最近日誌
openclaw logs

# 持續追蹤
openclaw logs --follow

# JSON 格式
openclaw logs --json --limit 50
```

---

### `openclaw acp` — Agent Control Protocol

```
openclaw acp [選項]
```

| 選項 | 說明 |
|------|------|
| `--session <key>` | Session key |
| `--session-label <label>` | Session label |
| `--token <token>` | Gateway Token |
| `--password <password>` | Gateway 密碼 |
| `--url <url>` | Gateway URL |
| `--provenance <mode>` | Provenance 模式：`off`/`meta`/`meta+receipt` |
| `--reset-session` | 使用前重設 session |
| `--require-existing` | 要求 session 已存在 |
| `-v, --verbose` | 詳細日誌 |

| 子指令 | 說明 |
|--------|------|
| `client` | 執行互動式 ACP 客戶端 |

```bash
# 執行 ACP bridge
openclaw acp --session "agent:main:main"

# 執行 ACP 客戶端
openclaw acp client
```

---

### `openclaw devices` — 裝置管理

```
openclaw devices [子指令]
```

| 子指令 | 說明 |
|--------|------|
| `list` | 列出待配對和已配對裝置 |
| `approve` | 批准待配對請求 |
| `reject` | 拒絕配對請求 |
| `remove` | 移除已配對裝置 |
| `clear` | 清除所有配對裝置 |
| `revoke` | 撤銷裝置 token |
| `rotate` | 輪換裝置 token |

---

### `openclaw directory` — 聯絡人目錄

```
openclaw directory [子指令]
```

| 子指令 | 說明 |
|--------|------|
| `self` | 顯示目前帳號身份 |
| `peers` | 聯絡人/使用者目錄 |
| `groups` | 群組目錄 |

```bash
# 查看 Slack 帳號身份
openclaw directory self --channel slack

# 搜尋聯絡人
openclaw directory peers list --channel slack --query "alice"

# 列出 Discord 群組
openclaw directory groups list --channel discord
```

---

### 其他工具指令

| 指令 | 說明 |
|------|------|
| `openclaw health` | 從運行中的 Gateway 取得健康狀態 |
| `openclaw docs [query]` | 搜尋線上 OpenClaw 文件 |
| `openclaw dns setup` | 設定 CoreDNS（Wide-Area Bonjour） |
| `openclaw system event` | 排入系統事件 |
| `openclaw system heartbeat` | 心跳控制 |
| `openclaw system presence` | 列出系統 presence |
| `openclaw webhooks gmail` | Gmail Pub/Sub hooks |
| `openclaw pairing` | 安全 DM 配對管理 |
| `openclaw completion` | 生成 shell 自動完成腳本 |
| `openclaw clawbot` | 舊版 clawbot 別名 |

```bash
# 安裝 shell 自動完成
openclaw completion --install

# 為 bash 生成
openclaw completion --shell bash

# 搜尋文件
openclaw docs "gateway setup"
```

---

## 18. 情境操作範例

### 情境一：從零開始設定（個人使用者，Anthropic API）

```bash
# 步驟 1：安裝
npm install -g openclaw

# 步驟 2：執行引導精靈
openclaw onboard

# 互動式選擇：
# - 模式：local
# - 認證：Anthropic API Key
# - Gateway 綁定：loopback
# - 安裝服務：是

# 步驟 3：驗證
openclaw status
openclaw gateway status

# 步驟 4：開始對話
openclaw tui
```

---

### 情境二：設定 Telegram Bot

```bash
# 步驟 1：確認 Gateway 運行中
openclaw gateway status

# 步驟 2：設定 Telegram Token
openclaw config set channels.telegram.token "123456:ABC-DEF..."

# 步驟 3：設定 allowlist
openclaw config set channels.telegram.allowlist '["@your_username"]'

# 步驟 4：啟用 Telegram
openclaw config set channels.telegram.enabled true

# 步驟 5：重啟 Gateway
openclaw gateway restart

# 步驟 6：驗證
openclaw channels status --probe
```

---

### 情境三：設定 Discord Bot

```bash
# 步驟 1：設定 Discord Token
openclaw config set channels.discord.token "MTIz..."

# 步驟 2：設定允許的伺服器
openclaw config set channels.discord.allowlist '["server_id"]'

# 步驟 3：啟用
openclaw config set channels.discord.enabled true

# 步驟 4：重啟並驗證
openclaw gateway restart
openclaw channels status --probe
```

---

### 情境四：使用 Ollama 本地模型

```bash
# 步驟 1：確認 Ollama 運行中
curl http://localhost:11434/api/tags

# 步驟 2：使用 Ollama 進行引導
openclaw onboard --non-interactive --accept-risk \
  --auth-choice ollama \
  --install-daemon

# 步驟 3：設定預設模型
openclaw models set ollama/qwen2.5-coder:32b

# 步驟 4：測試
openclaw agent --local --message "用 Python 寫一個 Hello World"
```

---

### 情境五：排程自動化（每日報告）

```bash
# 新增每日 9:00 執行的報告任務
openclaw cron add \
  --name "daily-summary" \
  --cron "0 9 * * *" \
  --tz "Asia/Taipei" \
  --message "產生今天的工作摘要，包含待辦事項和進度" \
  --announce \
  --channel telegram \
  --to "@mychat"

# 驗證
openclaw cron list

# 手動測試
openclaw cron run --name "daily-summary"
```

---

### 情境六：發送跨頻道訊息

```bash
# Telegram
openclaw message send --channel telegram --target "@chat_id" \
  --message "Telegram 通知"

# Discord
openclaw message send --channel discord --target "channel_id" \
  --message "Discord 通知"

# WhatsApp（附帶媒體）
openclaw message send --channel whatsapp --target "+886912345678" \
  --message "請查看附件" --media /tmp/report.pdf

# Slack
openclaw message send --channel slack --target "#general" \
  --message "Slack 通知"
```

---

### 情境七：Docker 部署

```bash
# 建立配置目錄
mkdir -p ~/.openclaw

# 設定配置
cat > ~/.openclaw/openclaw.json << 'EOF'
{
  "gateway": {
    "mode": "local",
    "port": 18789,
    "bind": "lan",
    "auth": {
      "mode": "token",
      "token": "my-secure-token"
    }
  }
}
EOF

# 執行 Docker
docker run -d \
  --name openclaw \
  -p 18789:18789 \
  -v ~/.openclaw:/root/.openclaw \
  -e OPENCLAW_TZ=Asia/Taipei \
  openclaw/openclaw gateway run --bind lan

# 從主機驗證
openclaw gateway status --url ws://localhost:18789 --token "my-secure-token"
```

---

### 情境八：備份與還原

```bash
# 建立備份
openclaw backup create

# 驗證備份
openclaw backup verify

# 安全審計
openclaw security audit --deep

# 健康檢查與自動修復
openclaw doctor --fix
```

---

### 情境九：iOS/Android 配對

```bash
# 生成配對 QR Code
openclaw qr

# 僅顯示 setup code（供手動輸入）
openclaw qr --setup-code-only

# 遠端模式（搭配公開 URL）
openclaw qr --remote --public-url "wss://my-gateway.example.com"

# 查看待配對裝置
openclaw devices list

# 批准配對
openclaw devices approve
```

---

### 情境十：非互動式 CI/CD 環境

```bash
# 非互動式安裝（CI 環境）
openclaw onboard --non-interactive --accept-risk \
  --auth-choice apiKey \
  --anthropic-api-key "$ANTHROPIC_API_KEY" \
  --secret-input-mode ref \
  --gateway-bind loopback \
  --gateway-auth token \
  --gateway-token "$OPENCLAW_GATEWAY_TOKEN" \
  --skip-channels \
  --skip-skills \
  --skip-search \
  --install-daemon

# 健康檢查（CI 中需要 exit code）
openclaw gateway status --require-rpc

# 執行 agent turn
openclaw agent --message "運行自動化測試" --json --timeout 300
```

---

*本文件基於 OpenClaw `2026.3.23` 版本撰寫。完整英文文件請參考 https://docs.openclaw.ai/cli*
