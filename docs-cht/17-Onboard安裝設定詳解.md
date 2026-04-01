# OpenClaw Onboard 安裝設定詳解

> 版本：`2026.3.31` | Node.js `>=22.16.0`
>
> 本文件完整說明 `openclaw onboard` 安裝精靈的所有步驟、Skills 系統、Hooks 系統，以及 Boot.md / Bootstrap 機制的詳細細節與設定方式。

---

## 目錄

1. [Onboard 概述](#1-onboard-概述)
2. [安裝前準備](#2-安裝前準備)
3. [Onboard 完整流程](#3-onboard-完整流程)
4. [Gateway 設定詳解](#4-gateway-設定詳解)
5. [頻道設定](#5-頻道設定)
6. [Web Search 搜尋設定](#6-web-search-搜尋設定)
7. [Skills 系統完整指南](#7-skills-系統完整指南)
8. [Hooks 系統完整指南](#8-hooks-系統完整指南)
9. [Bootstrap 與 Boot.md 機制](#9-bootstrap-與-bootmd-機制)
10. [Session Memory 詳解](#10-session-memory-詳解)
11. [完成與驗證](#11-完成與驗證)
12. [非互動式安裝](#12-非互動式安裝)
13. [設定檔完整參考](#13-設定檔完整參考)
14. [疑難排解](#14-疑難排解)

---

## 1. Onboard 概述

### 兩種安裝路徑

| 路徑 | 平台 | 介面 | 適用場景 | 指令 |
|------|------|------|---------|------|
| **CLI Onboard** | macOS / Linux / Windows (WSL2) | 終端精靈 | 伺服器、無頭環境、完整控制 | `openclaw onboard` |
| **macOS App** | macOS | 圖形化引導 | 桌面 Mac、視覺化設定 | 啟動 OpenClaw App |

### Onboard 設定的項目

1. **模型提供者與認證**（API Key / OAuth / Setup Token）
2. **工作區目錄**（預設 `~/.openclaw/workspace`）
3. **Gateway 設定**（連接埠、綁定位址、認證模式）
4. **通訊頻道**（WhatsApp / Telegram / Discord 等）
5. **Web Search 搜尋提供者**
6. **Skills 技能系統**
7. **Hooks 自動化鉤子**
8. **Daemon 背景服務安裝**

### 設定檔位置

所有設定寫入 `~/.openclaw/openclaw.json`（JSON5 格式，支援註解和尾隨逗號）。

---

## 2. 安裝前準備

### 系統需求

- **Node.js** >= 22.16.0
- **npm** / **pnpm** / **bun**（任一套件管理器）
- **Git**（建議）

### 安裝 OpenClaw

```bash
# 方法 A：安裝腳本（推薦）
curl -fsSL https://openclaw.ai/install.sh | bash

# 方法 B：npm 全域安裝
npm install -g openclaw@latest

# 方法 C：pnpm 全域安裝
pnpm add -g openclaw@latest
pnpm approve-builds -g

# 驗證安裝
openclaw --version
# 應顯示 2026.3.31 或更新版本
```

---

## 3. Onboard 完整流程

### 啟動精靈

```bash
openclaw onboard                    # 互動式（推薦）
openclaw onboard --flow quickstart  # 快速開始（最少提示）
openclaw onboard --flow manual      # 完整手動控制
openclaw onboard --mode remote      # 遠端 Gateway 模式
openclaw onboard --reset            # 重設後重新設定
```

### 步驟 1：安全風險確認

精靈首先顯示安全警告，說明 OpenClaw 預設為**個人使用模式**。如果要用於多人共享環境，需要額外的安全設定（配對、allowlists、沙箱）。

> 安全文件參考：https://docs.openclaw.ai/gateway/security

### 步驟 2：現有設定處理

若偵測到已存在的設定檔，提供三個選項：

| 選項 | 說明 |
|------|------|
| **Keep existing values** | 保留現有設定，僅更新缺失值 |
| **Update values** | 修改現有設定 |
| **Reset** | 重設設定 |

重設範圍選項：
- `config` — 僅重設設定檔
- `config+creds+sessions` — 設定 + 認證 + 會話
- `full` — 完全重設（含工作區）

### 步驟 3：流程選擇

| 流程 | 說明 |
|------|------|
| **QuickStart** | 最少提示、自動生成 token、建議預設值 |
| **Manual / Advanced** | 完整控制所有選項 |

> Remote Gateway 強制使用 Advanced 模式。

### 步驟 4：模型提供者與認證

支援的認證方式（按類別分組）：

#### 雲端提供者

| 認證選項 | 提供者 | 環境變數 | 說明 |
|---------|--------|---------|------|
| `setup-token` | Anthropic | `ANTHROPIC_API_KEY` | API Key |
| `openai-api-key` | OpenAI | `OPENAI_API_KEY` | API Key |
| `openai-codex` | OpenAI Codex | — | OAuth 流程 |
| `google-gemini-api-key` | Google Gemini | `GEMINI_API_KEY` | API Key |
| `google-vertex` | Google Vertex AI | — | Service Account |
| `amazon-bedrock` | Amazon Bedrock | — | AWS SDK |
| `azure-openai` | Azure OpenAI | — | API Key |
| `mistral-api-key` | Mistral AI | `MISTRAL_API_KEY` | API Key |
| `xai-api-key` | xAI (Grok) | `XAI_API_KEY` | API Key |
| `anthropic-vertex` | Anthropic Vertex | — | GCP Service Account |
| `github-copilot` | GitHub Copilot | — | OAuth |

#### 中國 / 亞洲提供者

| 認證選項 | 提供者 | 環境變數 |
|---------|--------|---------|
| `kimi-api-key` | Moonshot / Kimi | `KIMI_API_KEY` |
| `qwen-api-key` | 通義千問 | `QWEN_PORTAL_API_KEY` |
| `minimax-api-key` | MiniMax | `MINIMAX_API_KEY` |
| `modelstudio-api-key` | 阿里百煉 | `MODELSTUDIO_API_KEY` |
| `deepseek-api-key` | DeepSeek | `DEEPSEEK_API_KEY` |
| `xiaomi-api-key` | 小米 | `XIAOMI_API_KEY` |

#### 聚合 / 本地提供者

| 認證選項 | 提供者 | 說明 |
|---------|--------|------|
| `openrouter-api-key` | OpenRouter | 多模型聚合 |
| `together-api-key` | Together AI | 開源模型代管 |
| `ollama` | Ollama | 本地 LLM（自動下載模型） |
| `custom-api-key` | 自訂端點 | 任何 OpenAI/Anthropic 相容端點 |
| `skip` | 跳過 | 稍後再設定 |

#### Secret 輸入模式

| 模式 | 說明 |
|------|------|
| **plaintext**（預設） | 直接儲存 API Key 至設定檔 |
| **ref** | 儲存環境變數引用（SecretRef），運行時從 env 解析 |

```bash
# 使用 ref 模式
openclaw onboard --secret-input-mode ref
```

### 步驟 5：工作區設定

- **預設路徑**：`~/.openclaw/workspace`
- **可自訂**：`--workspace ~/my-workspace`
- 設定寫入：`agents.defaults.workspace`
- 首次建立時會 seed 初始 bootstrap 檔案（`AGENTS.md`、`BOOTSTRAP.md` 等）

### 步驟 6：Gateway 設定

詳見 [第 4 節](#4-gateway-設定詳解)。

### 步驟 7：頻道設定

詳見 [第 5 節](#5-頻道設定)。

### 步驟 8：Web Search 設定

詳見 [第 6 節](#6-web-search-搜尋設定)。

### 步驟 9：Skills 設定

詳見 [第 7 節](#7-skills-系統完整指南)。

### 步驟 10：Hooks 設定

詳見 [第 8 節](#8-hooks-系統完整指南)。

### 步驟 11：完成

詳見 [第 11 節](#11-完成與驗證)。

---

## 4. Gateway 設定詳解

### QuickStart 預設值

| 項目 | 預設值 |
|------|--------|
| 連接埠 | `18789` |
| 綁定 | `loopback`（127.0.0.1） |
| 認證 | Token（自動生成 48 字元 hex） |
| Tailscale | Off |
| 工具 Profile | `coding` |
| DM Scope | `per-channel-peer` |

### Manual 模式選項

#### 綁定模式

| 模式 | 說明 | 適用場景 |
|------|------|---------|
| `loopback` | 僅 127.0.0.1 | 本機使用 |
| `lan` | 0.0.0.0（所有介面） | 區域網路存取 |
| `tailnet` | Tailscale 網路 | 安全遠端存取 |
| `custom` | 自訂 IP | 特定網路介面 |
| `auto` | 自動偵測 | 進階用途 |

#### 認證模式

| 模式 | 說明 |
|------|------|
| `token` | Token 認證（推薦，預設自動生成） |
| `password` | 密碼認證 |
| `none` | 無認證（**不建議**，即使 loopback） |

#### Tailscale 整合

| 選項 | 說明 |
|------|------|
| `off` | 不使用 Tailscale |
| `serve` | 透過 Tailscale 提供服務（僅 tailnet 內） |
| `funnel` | 透過 Tailscale Funnel 公開（需付費方案） |

### Gateway 設定範例

```json5
{
  gateway: {
    port: 18789,
    bind: "127.0.0.1",
    auth: {
      mode: "token",
      token: "your-generated-token-here",
    },
    // 或使用 SecretRef
    // auth: { mode: "token", token: { source: "env", provider: "default", id: "OPENCLAW_GATEWAY_TOKEN" } },

    // Tailscale（可選）
    tailscale: {
      mode: "off",        // off | serve | funnel
      resetOnExit: false,
    },

    // 熱重載
    reload: {
      mode: "hybrid",     // hybrid | hot | restart | off
      debounceMs: 300,
    },
  },
}
```

---

## 5. 頻道設定

Onboard 精靈會列出所有已安裝的頻道插件，讓你逐一設定：

### 可設定的頻道

| 頻道 | 需要 | 說明 |
|------|------|------|
| WhatsApp | QR Code 掃描 | Baileys 協議 |
| Telegram | Bot Token | grammY Bot API |
| Discord | Bot Token | Carbon Bot |
| Slack | Bot Token + App Token | @slack/bolt |
| Signal | Signal CLI | 需要已註冊號碼 |
| iMessage | macOS + AppleScript | 僅 macOS |
| LINE | Channel Access Token | LINE Messaging API |
| Google Chat | Service Account | Google Workspace |
| Matrix | Homeserver URL + Token | 開放協議 |
| MS Teams | App ID + Secret | Bot Framework |
| 飛書 (Feishu) | App ID + Secret | 飛書 Bot API |
| Mattermost | Bot Token | 開源團隊協作 |
| Synology Chat | Webhook URL | Synology NAS（含設定精靈） |
| Nostr | 私鑰 | 去中心化協議 |

### DM Scope 設定

| 範圍 | 說明 |
|------|------|
| `per-channel-peer`（預設） | 每個頻道 + 對話者隔離 session |
| `per-channel` | 每個頻道共用 session |
| `per-sender` | 每個發送者隔離 session |
| `agent` | 所有 DM 共用同一 agent session |

### DM Policy 設定

| 政策 | 說明 |
|------|------|
| `pairing` | 需透過配對碼連接 |
| `allowlist` | 僅允許清單中的使用者 |
| `open` | 開放所有 DM |
| `disabled` | 停用 DM |

---

## 6. Web Search 搜尋設定

### 可用搜尋提供者

| 提供者 | 類型 | 需要 API Key | 說明 |
|--------|------|-------------|------|
| **Brave** | Bundled plugin（預設啟用） | 是 | 快速結構化結果 + LLM Context API |
| **DuckDuckGo** | Bundled plugin | 否（免費） | Experimental，不需 API key |
| **Exa** | Bundled plugin | 是 | 語意搜尋 + extract 功能 |
| **Tavily** | Bundled plugin | 是 | Search + extract 雙工具 |
| **Perplexity** | 內建 | 是 | 結構化結果 + 語言/地區/時間篩選 |
| **Gemini** | 內建 | 是 | Google Search grounding |
| **Grok** | 內建 | 是 | xAI web-grounded 回應 |
| **Kimi** | 內建 | 是 | Moonshot web search |

### 設定範例

```json5
{
  tools: {
    web: {
      search: {
        provider: "brave",              // 搜尋提供者 ID
        enabled: true,
        brave: {
          apiKey: "${BRAVE_API_KEY}",
        },
        // 或使用 Perplexity
        perplexity: {
          apiKey: "${PERPLEXITY_API_KEY}",
          searchLanguage: "zh-TW",       // 搜尋語言
          searchRegion: "TW",            // 搜尋地區
          searchRecency: "week",         // day | week | month | year
        },
      },
    },
  },
}
```

---

## 7. Skills 系統完整指南

### 概述

Skills（技能）是擴展 Agent 能力的模組化工具。每個 Skill 提供特定的工具命令，由 Agent 根據使用者請求自動呼叫。

### Onboard 中的 Skills 設定流程

1. 精靈掃描所有可用 Skills 及其狀態
2. 依狀態分組顯示：
   - **Eligible**（就緒）：所有需求已滿足
   - **Missing requirements**（缺少依賴）：需要安裝外部工具
   - **Unsupported OS**（不支援的系統）：當前 OS 不支援
   - **Blocked by allowlist**（被安全策略阻擋）
3. 提供安裝缺失依賴的選項（brew / npm / go / uv / download）
4. 提示輸入需要 API Key 的 Skills 的環境變數

### Skill 來源

| 來源 | 路徑 | 說明 |
|------|------|------|
| Bundled | 隨 OpenClaw 發布 | 核心 Skills |
| Managed | `~/.openclaw/skills/` | 跨工作區共用 |
| Workspace | `<workspace>/skills/` | 工作區專用 |
| Plugin | `extensions/*/skills/` | 插件提供 |

### 完整 Skill 清單

#### 生產力工具

| Skill | Emoji | 說明 | 需求 | 安裝方式 |
|-------|-------|------|------|---------|
| **github** | 🐙 | GitHub 完整操作（Issues、PRs、CI、Code Review） | `gh` CLI | `brew install gh` |
| **gh-issues** | 🐙 | GitHub Issues 操作（範圍限定） | `gh` CLI | `brew install gh` |
| **notion** | 📝 | Notion API（建立/管理頁面、資料庫、區塊） | `NOTION_API_KEY` env | 需 API Key |
| **trello** | 📋 | Trello 看板/列表/卡片管理 | `jq` + `TRELLO_API_KEY` + `TRELLO_TOKEN` | `brew install jq` |
| **obsidian** | 💎 | Obsidian vault 操作（純 Markdown） | `obsidian-cli` | `brew install obsidian-cli` |
| **bear-notes** | 🐻 | Bear 筆記建立/搜尋/管理（macOS） | `grizzly` | `go install` |
| **apple-notes** | 📝 | Apple Notes 管理（macOS） | `memo` | `brew install memo` |
| **apple-reminders** | ⏰ | Apple Reminders 管理（macOS） | `remindctl` | `brew install remindctl` |
| **things-mac** | ✅ | Things 3 任務管理（macOS） | `things` | `go install` |

#### 通訊工具

| Skill | Emoji | 說明 | 需求 | 安裝方式 |
|-------|-------|------|------|---------|
| **himalaya** | 📧 | CLI 郵件管理（IMAP/SMTP） | `himalaya` | `brew install himalaya` |
| **gog** | 🎮 | Google Workspace（Gmail、Calendar、Drive、Contacts、Sheets、Docs） | `gog` | `brew install gog` |
| **imsg** | 📨 | iMessage/SMS CLI（macOS Messages.app） | `imsg`（macOS） | `brew install imsg` |
| **slack** | 💬 | Slack 操作（reactions、pins、messages） | Slack 頻道已設定 | 設定 `channels.slack` |
| **discord** | 🎮 | Discord 操作 | Discord 頻道已設定 | 設定 `channels.discord.token` |
| **bluebubbles** | 🫧 | iMessage via BlueBubbles | BlueBubbles 已設定 | 設定 `channels.bluebubbles` |
| **wacli** | 📱 | WhatsApp 訊息和歷史同步 | `wacli` | `brew install wacli` |
| **xurl** | 🐦 | X (Twitter) API（貼文、回覆、搜尋、DM） | `xurl` | `brew install xurl` 或 `npm` |

#### 媒體與音訊

| Skill | Emoji | 說明 | 需求 | 安裝方式 |
|-------|-------|------|------|---------|
| **video-frames** | 🎬 | 從影片擷取幀/片段 | `ffmpeg` | `brew install ffmpeg` |
| **openai-whisper** | 🎤 | 本地語音轉文字（Whisper CLI，無 API Key） | `whisper` | `brew install whisper` |
| **openai-whisper-api** | 🌐 | OpenAI Whisper API 語音轉文字 | `curl` + `OPENAI_API_KEY` | 需 API Key |
| **sag** | 🔊 | ElevenLabs 文字轉語音 | `sag` + `ELEVENLABS_API_KEY` | `brew install sag` |
| **sherpa-onnx-tts** | 🔉 | 本地離線文字轉語音 | sherpa-onnx runtime | 需手動安裝 |
| **songsee** | 🌊 | 音訊頻譜圖和特徵視覺化 | `songsee` | `brew install songsee` |
| **camsnap** | 📸 | RTSP/ONVIF 攝影機擷取 | `camsnap` | `brew install camsnap` |
| **peekaboo** | 👀 | macOS UI 自動化和螢幕擷取 | `peekaboo`（macOS） | `brew install peekaboo` |
| **gifgrep** | 🧲 | GIF 搜尋和下載 | `gifgrep` | `brew install gifgrep` |

#### 智慧家庭與生活

| Skill | Emoji | 說明 | 需求 | 安裝方式 |
|-------|-------|------|------|---------|
| **openhue** | 💡 | Philips Hue 燈光/場景控制 | `openhue` | `brew install openhue` |
| **eightctl** | 🛌 | Eight Sleep Pod 控制 | `eightctl` | `go install` |
| **blucli** | 🫐 | BluOS 音響控制 | `blu` | `go install` |
| **spotify-player** | 🎵 | Spotify 播放/搜尋 | `spogo` 或 `spotify_player` | `brew install` |
| **weather** | ☔ | 天氣和預報（wttr.in / Open-Meteo） | `curl`（免費，無 API Key） | 已預裝 |
| **goplaces** | 📍 | Google Places API 查詢 | `goplaces` + `GOOGLE_PLACES_API_KEY` | `brew install goplaces` |
| **ordercli** | 🛵 | Foodora 訂單查詢 | `ordercli` | `brew install ordercli` |

#### 開發工具

| Skill | Emoji | 說明 | 需求 | 安裝方式 |
|-------|-------|------|------|---------|
| **coding-agent** | 🧩 | 委託 Codex / Claude Code / Pi agent 執行程式碼任務 | `claude` / `codex` / `opencode` / `pi`（任一） | 各自安裝 |
| **tmux** | 🧵 | 遠端控制 tmux session | `tmux` | `brew install tmux` |
| **session-logs** | 📜 | 搜尋和分析自己的 session 日誌 | `jq` + `rg` | `brew install jq ripgrep` |
| **mcporter** | 📦 | 直接操作 MCP servers | `mcporter` | `npm install -g mcporter` |
| **gemini** | ✨ | Gemini CLI Q&A、摘要、生成 | `gemini` | `brew install gemini` |
| **oracle** | 🧿 | Oracle CLI 最佳實踐 | `oracle` | `npm install -g oracle` |
| **nano-pdf** | 📄 | 自然語言 PDF 編輯 | `nano-pdf` | `uv install nano-pdf` |

#### 內容與研究

| Skill | Emoji | 說明 | 需求 | 安裝方式 |
|-------|-------|------|------|---------|
| **summarize** | 🧾 | 從 URL/播客/檔案摘要和提取 | `summarize` | `brew install summarize` |
| **blogwatcher** | 📰 | 監控部落格和 RSS/Atom feeds | `blogwatcher` | `go install` |

#### 安全與管理

| Skill | Emoji | 說明 | 需求 | 安裝方式 |
|-------|-------|------|------|---------|
| **1password** | 🔐 | 1Password CLI 操作 | `op` | `brew install 1password-cli` |
| **model-usage** | 📊 | 每模型使用量成本摘要 | `codexbar`（macOS） | `brew install --cask codexbar` |
| **healthcheck** | — | 主機安全強化和風險容忍度設定 | — | 內建 |

#### 系統 Skills

| Skill | 說明 | 需求 |
|-------|------|------|
| **canvas** | 在已連接的 OpenClaw 節點上顯示 HTML 內容 | Canvas host 設定 |
| **clawhub** | ClawHub 插件市場 CLI（搜尋/安裝/更新/發布 Skills） | `clawhub` via npm |
| **node-connect** | 診斷 OpenClaw 節點連線和配對故障 | — |
| **skill-creator** | 建立、編輯、改善或審計 AgentSkills | — |
| **voice-call** | 透過 voice-call 插件發起語音通話 | voice-call 插件已啟用 |

#### 擴充 Skills（來自 Extensions）

| Skill | 擴充套件 | 說明 |
|-------|---------|------|
| **feishu-doc** | feishu | 飛書文件讀寫操作 |
| **feishu-drive** | feishu | 飛書雲端硬碟檔案管理 |
| **feishu-perm** | feishu | 飛書文件/檔案權限管理 |
| **feishu-wiki** | feishu | 飛書知識庫導覽 |
| **diffs** | diffs | 可分享的差異渲染（viewer URL 或檔案） |
| **acp-router** | acpx | 路由請求至 Pi / Claude Code / Codex / OpenCode / Gemini CLI |
| **lobster** | lobster | Lobster 工作流程（含審核檢查點） |
| **prose** | open-prose | OpenProse VM 技能包（多 Agent 工作流程） |
| **tavily** | tavily | Tavily Web 搜尋、內容提取、研究工具 |

### Skills 設定

```json5
{
  skills: {
    // 允許的 bundled skills（白名單）
    allowBundled: ["github", "weather", "session-logs"],

    // 安裝偏好
    install: {
      preferBrew: true,                  // macOS 上優先使用 Homebrew
      nodeManager: "pnpm",              // npm | pnpm | yarn | bun
    },

    // 載入設定
    load: {
      extraDirs: [],                     // 額外的 skills 目錄
      watch: true,                       // 監視變更
      watchDebounceMs: 1000,
    },

    // 限制
    limits: {
      maxCandidatesPerRoot: 100,
      maxSkillsLoadedPerSource: 500,
      maxSkillsInPrompt: 100,
      maxSkillsPromptChars: 50000,
      maxSkillFileBytes: 1048576,        // 1MB
    },

    // 個別 Skill 設定
    entries: {
      "notion": {
        enabled: true,
        apiKey: "${NOTION_API_KEY}",     // 或 SecretRef
      },
      "goplaces": {
        enabled: true,
        env: { "GOOGLE_PLACES_API_KEY": "your-key-here" },
      },
      "weather": {
        enabled: true,                   // 無需 API Key
      },
    },
  },
}
```

### Skills 管理指令

```bash
# 列出所有 skills 及其狀態
openclaw skills list

# 列出就緒的 skills
openclaw skills list --eligible

# 檢查特定 skill 資訊
openclaw skills info github

# 安裝 skill 依賴
openclaw skills install github
```

---

## 8. Hooks 系統完整指南

### 概述

Hooks 是事件驅動的自動化機制，在特定事件（指令、啟動、訊息等）發生時觸發自訂處理程序。

### Onboard 中的 Hooks 設定流程

1. 精靈列出所有可用的 bundled hooks
2. 對每個 hook 顯示名稱、描述、監聽事件
3. 使用者選擇要啟用的 hooks
4. 精靈寫入設定

### Bundled Hooks 完整說明

#### 1. boot-md 🚀 — Gateway 啟動執行

**功能**：Gateway 啟動時自動執行工作區中的 `BOOT.md` 檔案。

**監聽事件**：`gateway:startup`

**運作機制**：
1. Gateway 啟動完成後觸發 `gateway:startup` 事件
2. Hook handler 載入 `<workspace>/BOOT.md`
3. 建立臨時 boot session（ID: `boot-<timestamp>-<uuid>`）
4. 用 BOOT.md 內容建構 prompt 發送給 Agent
5. Agent 執行 BOOT.md 中的指令（例如發送訊息、檢查狀態）
6. 執行完成後恢復主 session 狀態

**用途範例**：
- 每次 Gateway 重啟時發送狀態通知
- 自動檢查系統健康
- 執行定時初始化任務

**BOOT.md 範例**：

```markdown
# 啟動檢查清單

1. 使用 message 工具向 Telegram 傳送「Gateway 已啟動 ✅」
2. 檢查 WhatsApp 連線狀態
3. 如果有異常，使用 message 工具通知管理員
```

**設定**：

```json5
{
  hooks: {
    internal: {
      enabled: true,
      entries: {
        "boot-md": {
          enabled: true,
        },
      },
    },
  },
}
```

**需求**：`workspace.dir` 必須已設定

**狀態碼**：
- `skipped` + `missing` — BOOT.md 檔案不存在
- `skipped` + `empty` — BOOT.md 檔案為空
- `ran` — 成功執行
- `failed` — 執行失敗

---

#### 2. bootstrap-extra-files 📎 — 額外 Bootstrap 檔案注入

**功能**：在 Agent 初始化時注入額外的 bootstrap 檔案，適用於 monorepo 或多套件專案。

**監聽事件**：`agent:bootstrap`

**運作機制**：
1. Agent 初始化時觸發 `agent:bootstrap` 事件
2. Hook handler 讀取設定中的 glob/路徑模式
3. 在工作區中搜尋匹配的檔案
4. 將找到的檔案附加至 Agent 的 context files

**用途範例**：
- 在 monorepo 中載入所有子套件的 `AGENTS.md`
- 載入共用的工具文件
- 注入專案特定的指令

**設定**：

```json5
{
  hooks: {
    internal: {
      enabled: true,
      entries: {
        "bootstrap-extra-files": {
          enabled: true,
          // 使用 paths、patterns 或 files（三者為同義詞）
          paths: [
            "packages/*/AGENTS.md",
            "packages/*/TOOLS.md",
            "docs/bootstrap/*.md",
          ],
        },
      },
    },
  },
}
```

**支援的路徑格式**：
- Glob 模式：`packages/*/AGENTS.md`
- 目錄模式：`docs/bootstrap/`
- 相對路徑：`../shared/TOOLS.md`

**可載入的 Bootstrap 檔名**：
- `AGENTS.md` — Agent 行為規則
- `SOUL.md` — 個性描述
- `TOOLS.md` — 工具指引
- `IDENTITY.md` — 身份設定
- `USER.md` — 使用者資訊
- `HEARTBEAT.md` — 心跳檢查
- `BOOTSTRAP.md` — 啟動指令
- `MEMORY.md` / `memory.md` — 記憶檔案

**安全限制**：
- 所有路徑從工作區解析
- Realpath 驗證防止 symlink 逃脫
- 僅載入上述認可的 bootstrap 檔名

**需求**：`workspace.dir` 必須已設定

---

#### 3. command-logger 📝 — 指令日誌記錄

**功能**：記錄所有使用者指令（`/new`、`/reset`、`/stop` 等）至日誌檔案。

**監聽事件**：`command`（所有指令事件）

**運作機制**：
1. 使用者執行任何指令時觸發 `command` 事件
2. Hook handler 擷取事件資訊
3. 以 JSONL 格式附加至日誌檔案

**日誌位置**：`~/.openclaw/logs/commands.log`

**日誌格式**（JSONL，每行一筆）：

```json
{"timestamp":"2026-03-23T14:30:00.000Z","action":"new","sessionKey":"agent:main:telegram:direct:123456","senderId":"+886912345678","source":"telegram"}
{"timestamp":"2026-03-23T15:00:00.000Z","action":"reset","sessionKey":"agent:main:whatsapp:direct:+886987654321","senderId":"+886987654321","source":"whatsapp"}
{"timestamp":"2026-03-23T16:00:00.000Z","action":"stop","sessionKey":"agent:main:main","senderId":"local","source":"tui"}
```

**記錄的欄位**：
- `timestamp` — ISO 8601 時間戳
- `action` — 指令名稱（new、reset、stop 等）
- `sessionKey` — Session 識別碼
- `senderId` — 發送者 ID
- `source` — 來源頻道

**設定**：

```json5
{
  hooks: {
    internal: {
      enabled: true,
      entries: {
        "command-logger": {
          enabled: true,
          // 無額外設定選項
        },
      },
    },
  },
}
```

**注意事項**：
- 無自動日誌輪轉，需手動執行 `mv` 或設定 `logrotate`
- 靜默運作，不通知使用者
- 適合審計和除錯用途

---

#### 4. session-memory 💾 — Session 記憶自動儲存

**功能**：在使用者執行 `/new` 或 `/reset` 時，自動將當前 session 的對話摘要儲存為記憶檔案。

**監聽事件**：`command:new`、`command:reset`

**運作機制**：
1. 使用者執行 `/new` 或 `/reset` 時觸發事件
2. Hook handler 找到前一個 session 的 JSONL 檔案
3. 提取最後 N 則訊息（預設 15 則）
4. 使用 LLM 生成描述性檔名 slug
5. 建立日期化的記憶檔案：`<workspace>/memory/YYYY-MM-DD-<slug>.md`

**輸出格式**：

```markdown
# Session: 2026-03-23 14:30:00 UTC

- **Session Key**: agent:main:telegram:direct:123456
- **Session ID**: abc123def456
- **Source**: telegram

## Conversation Summary

[對話內容摘要]
```

**檔名範例**：
- `2026-03-23-vendor-pitch.md`
- `2026-03-23-api-design.md`
- `2026-03-23-bug-investigation.md`

**設定**：

```json5
{
  hooks: {
    internal: {
      enabled: true,
      entries: {
        "session-memory": {
          enabled: true,
          messages: 25,          // 包含的訊息數（預設 15）
          llmSlug: true,         // 使用 LLM 生成檔名（預設 true）
        },
      },
    },
  },
}
```

**需求**：`workspace.dir` 必須已設定

**注意事項**：
- LLM 不可用時 fallback 為時間戳 slug
- 自動建立 `memory/` 目錄
- 驗證 session 檔案路徑安全性
- 測試環境中跳過 LLM

---

### Hooks 管理指令

```bash
# 列出所有 hooks
openclaw hooks list

# 列出就緒的 hooks
openclaw hooks list --eligible

# 查看特定 hook 詳情
openclaw hooks info session-memory

# 檢查 hook 就緒狀態
openclaw hooks check

# 啟用 hook
openclaw hooks enable session-memory

# 停用 hook
openclaw hooks disable command-logger
```

### Hooks 設定完整範例

```json5
{
  hooks: {
    enabled: true,                      // 全域啟用
    internal: {
      enabled: true,                    // 內部 hooks 啟用
      entries: {
        "boot-md": { enabled: true },
        "bootstrap-extra-files": {
          enabled: true,
          paths: [
            "packages/*/AGENTS.md",
            "shared/TOOLS.md",
          ],
        },
        "command-logger": { enabled: true },
        "session-memory": {
          enabled: true,
          messages: 20,
        },
      },
      load: {
        extraDirs: [],                   // 額外的 hooks 目錄
      },
    },
  },
}
```

### Hook 事件完整列表

| 事件 | 觸發時機 |
|------|---------|
| `command` | 所有指令 |
| `command:new` | `/new` 指令 |
| `command:reset` | `/reset` 指令 |
| `command:stop` | `/stop` 指令 |
| `agent:bootstrap` | Agent 初始化 |
| `gateway:startup` | Gateway 啟動完成 |
| `message` | 所有訊息事件 |
| `message:received` | 收到使用者訊息 |
| `message:transcribed` | 音訊轉錄完成 |
| `message:preprocessed` | 訊息前處理完成 |
| `message:sent` | 回覆已送出（含 `isGroup`、`groupId`） |
| `session:compact:before` | 壓縮開始前 |
| `session:compact:after` | 壓縮完成後 |

### Hook 發現優先序

1. **Bundled hooks** — 隨 OpenClaw 發布
2. **Plugin hooks** — 已安裝插件提供
3. **Managed hooks** — `~/.openclaw/hooks/` + 額外目錄
4. **Workspace hooks** — `<workspace>/hooks/`（預設停用）

---

## 9. Bootstrap 與 Boot.md 機制

### 工作區檔案結構

```
<workspace>/
├── AGENTS.md           # Agent 行為規則和指示
├── SOUL.md             # Agent 個性和語調
├── USER.md             # 使用者身份資訊
├── IDENTITY.md         # Agent 名稱、風格、Emoji
├── TOOLS.md            # 工具使用指引（僅引導）
├── HEARTBEAT.md        # 心跳檢查清單（可選）
├── BOOT.md             # 啟動檢查清單（需啟用 boot-md hook）
├── BOOTSTRAP.md        # 首次執行引導（完成後自動刪除）
├── MEMORY.md           # 精選長期記憶（可選）
├── memory/             # 自動記憶檔案
│   ├── 2026-03-23-api-design.md
│   └── 2026-03-23-bug-fix.md
├── skills/             # 工作區專用 skills（可選）
├── hooks/              # 工作區專用 hooks（可選）
└── canvas/             # Canvas UI 檔案（可選）
```

### BOOTSTRAP.md — 首次執行引導

**用途**：首次啟動 Agent 時的對話式引導流程。

**運作機制**：
1. Agent 初始化時偵測到 `BOOTSTRAP.md` 存在
2. 執行 Q&A 流程（一次一個問題）
3. 與使用者一起建立 Agent 身份
4. 寫入 `IDENTITY.md`、`USER.md`、`SOUL.md`
5. 完成後**自動刪除** `BOOTSTRAP.md`（僅執行一次）

**典型內容**：
```markdown
# 首次啟動引導

Hey. I just came online. Who am I?

Let's figure this out together:
1. What's my name?
2. What's my nature/vibe?
3. What emoji represents me?

After we're done:
- Update IDENTITY.md with name, vibe, emoji
- Update USER.md with user preferences
- Update SOUL.md with personality guidelines
- Optionally set up a channel connection (WhatsApp, Telegram, web)
- Delete this BOOTSTRAP.md file
```

### BOOT.md — Gateway 啟動腳本

**用途**：每次 Gateway 啟動時自動執行的任務清單。

**需求**：必須啟用 `boot-md` hook。

**運作機制**：
1. Gateway 啟動並完成頻道連接後觸發
2. 載入 `BOOT.md` 內容
3. 在臨時 boot session 中執行
4. Agent 遵循 BOOT.md 指令操作
5. 如需發送訊息，使用 message tool（`action=send`）
6. 完成後恢復主 session

**BOOT.md 範例**：

```markdown
# 啟動任務

## 狀態通知
使用 message 工具向 Telegram @admin_user 傳送：
「🟢 Gateway 已啟動，時間：{current_time}」

## 健康檢查
1. 檢查所有已啟用頻道的連線狀態
2. 如果有頻道離線，通知管理員

## 每日摘要
如果是早上 8:00-9:00，發送昨天的活動摘要
```

### Bootstrap 載入順序

Agent 初始化時的檔案載入流程：

```
1. 載入核心 bootstrap 檔案
   ├── 如有 sessionKey → 使用 session 快取
   └── 否則 → 從工作區載入所有 bootstrap 檔案

2. 套用 bootstrap hooks（agent:bootstrap 事件）
   └── bootstrap-extra-files hook 新增額外模式匹配的檔案

3. 依 session 過濾
   └── 移除不符合 sessionKey 的覆寫

4. 套用 context mode 過濾
   ├── full → 包含所有檔案
   ├── lightweight + heartbeat → 僅 HEARTBEAT.md
   └── lightweight + cron/default → 空 context

5. 驗證檔案
   └── 確保路徑有效且為正規路徑
```

### Bootstrap 快取

- 以 sessionKey 為 key 快取 bootstrap 檔案
- 透過 inode/device/size/mtime 驗證有效性
- 首次存取時延遲載入
- 工作區狀態變更時清除
- 單檔上限：2MB

---

## 10. Session Memory 詳解

### 記憶搜尋設定

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        enabled: true,                 // 啟用記憶搜尋
        provider: "openai",            // openai | gemini | voyage | mistral | local
        model: "text-embedding-3-small",

        // 額外記憶路徑
        extraPaths: [
          "../team-docs",
          "/srv/shared-notes/overview.md",
        ],

        // 混合搜尋（BM25 + 向量）
        query: {
          hybrid: {
            enabled: true,
            vectorWeight: 0.7,
            textWeight: 0.3,
            candidateMultiplier: 4,
            mmr: {
              enabled: true,
              lambda: 0.7,             // 0=最大多樣性，1=最大相關性
            },
            temporalDecay: {
              enabled: true,
              halfLifeDays: 30,        // 提升近期記憶權重
            },
          },
        },
      },
    },
  },
}
```

### 記憶檔案格式

- **類型**：Markdown（`.md`）
- **位置**：`<workspace>/memory/*.md` 和 `<workspace>/MEMORY.md`
- **索引**：Per-agent SQLite，位於 `~/.openclaw/memory/<agentId>.sqlite`
- **監視**：檔案變更監視器（debounce 1.5 秒）
- **工具**：`memory_search` 和 `memory_get`（`memorySearch.enabled` 為 true 時啟用）

---

## 11. 完成與驗證

### 精靈完成步驟

1. **Systemd Linger 檢查**（Linux）：確保 session 在登出後存活
2. **Daemon 安裝**：
   - macOS → LaunchAgent
   - Linux → Systemd user unit
   - Windows → Scheduled Tasks 或 Startup folder
3. **Gateway 健康檢查**：等待 Gateway 可達（預設 15 秒逾時）
4. **Shell 自動完成**：偵測 shell 類型並安裝補全腳本
5. **Control UI 啟動**：開啟瀏覽器至 `http://localhost:18789`

### 驗證指令

```bash
# 版本確認
openclaw --version

# 系統診斷
openclaw doctor
openclaw doctor --fix      # 自動修復

# Gateway 狀態
openclaw gateway status
openclaw health

# 頻道狀態
openclaw channels status
openclaw channels status --probe    # 含連線探測
openclaw channels status --deep     # 深度探測

# Skills 狀態
openclaw skills list --eligible

# Hooks 狀態
openclaw hooks check
```

---

## 12. 非互動式安裝

適用於自動化部署和 CI/CD 環境：

```bash
# OpenAI + 預設設定
openclaw onboard --non-interactive \
  --accept-risk \
  --auth-choice openai-api-key \
  --openai-api-key "$OPENAI_API_KEY"

# Anthropic + 自訂工作區
openclaw onboard --non-interactive \
  --accept-risk \
  --auth-choice setup-token \
  --anthropic-api-key "$ANTHROPIC_API_KEY" \
  --workspace /opt/openclaw/workspace

# Ollama 本地模型
openclaw onboard --non-interactive \
  --accept-risk \
  --auth-choice ollama \
  --custom-base-url "http://localhost:11434" \
  --custom-model-id "qwen3.5:27b"

# 自訂端點（OpenAI 相容）
openclaw onboard --non-interactive \
  --accept-risk \
  --auth-choice custom-api-key \
  --custom-base-url "https://llm.example.com/v1" \
  --custom-model-id "my-model" \
  --custom-api-key "$MY_API_KEY" \
  --custom-compatibility openai \
  --secret-input-mode plaintext

# Secret Reference 模式（推薦用於生產環境）
openclaw onboard --non-interactive \
  --accept-risk \
  --auth-choice openai-api-key \
  --secret-input-mode ref

# 遠端 Gateway
openclaw onboard --non-interactive \
  --accept-risk \
  --mode remote \
  --remote-url "wss://gateway.example.com" \
  --remote-token "$GATEWAY_TOKEN"

# 跳過特定步驟
openclaw onboard --non-interactive \
  --accept-risk \
  --auth-choice setup-token \
  --anthropic-api-key "$KEY" \
  --skip-channels \
  --skip-skills \
  --skip-search \
  --skip-health \
  --skip-ui

# 指定 Daemon 設定
openclaw onboard --non-interactive \
  --accept-risk \
  --auth-choice openai-api-key \
  --openai-api-key "$KEY" \
  --install-daemon \
  --daemon-runtime node      # node | bun
```

### 必要旗標

| 旗標 | 說明 |
|------|------|
| `--non-interactive` | 啟用非互動模式 |
| `--accept-risk` | **必要**：確認安全風險（個人使用模式） |
| `--auth-choice <provider>` | 選擇認證提供者 |

---

## 13. 設定檔完整參考

### 完整設定結構

```json5
{
  // 模型與 Agent
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
      model: {
        primary: "anthropic/claude-sonnet-4-6",
        fallbacks: ["openai/gpt-5.2"],
      },
      thinking: "adaptive",            // adaptive | low | off
      skipBootstrap: false,
    },
  },

  // Session
  session: {
    dmScope: "per-channel-peer",
  },

  // 工具
  tools: {
    profile: "coding",                 // coding | full | sandbox | none
    web: {
      search: {
        provider: "brave",
        enabled: true,
      },
    },
  },

  // Gateway
  gateway: {
    port: 18789,
    bind: "127.0.0.1",
    auth: {
      mode: "token",
      token: "auto-generated-token",
    },
    reload: { mode: "hybrid" },
  },

  // 頻道
  channels: {
    telegram: {
      enabled: true,
      botToken: "BOT_TOKEN",
      dmPolicy: "allowlist",
      allowFrom: ["@username"],
    },
  },

  // Skills
  skills: {
    install: { nodeManager: "pnpm", preferBrew: true },
    entries: {
      github: { enabled: true },
      notion: { apiKey: "${NOTION_API_KEY}" },
    },
  },

  // Hooks
  hooks: {
    internal: {
      enabled: true,
      entries: {
        "boot-md": { enabled: true },
        "bootstrap-extra-files": {
          enabled: true,
          paths: ["packages/*/AGENTS.md"],
        },
        "command-logger": { enabled: true },
        "session-memory": { enabled: true, messages: 20 },
      },
    },
  },

  // 精靈元資料（自動寫入）
  wizard: {
    lastRunAt: "2026-03-23T10:00:00.000Z",
    lastRunVersion: "2026.3.31",
    lastRunCommand: "onboard",
    lastRunMode: "local",
  },
}
```

### 環境變數替換

設定值中支援 `${ENV_VAR}` 語法：

```json5
{
  gateway: { auth: { token: "${OPENCLAW_GATEWAY_TOKEN}" } },
  models: { providers: { openai: { apiKey: "${OPENAI_API_KEY}" } } },
}
```

規則：
- 僅匹配大寫名稱：`[A-Z_][A-Z0-9_]*`
- 缺失/空變數在載入時拋出錯誤
- 用 `$${VAR}` 轉義為文字輸出

### SecretRef 參考

```json5
{
  // 環境變數參考
  apiKey: { source: "env", provider: "default", id: "OPENAI_API_KEY" },

  // 檔案參考
  apiKey: { source: "file", provider: "filemain", id: "/skills/entries/notion/apiKey" },

  // 指令參考
  serviceAccount: { source: "exec", provider: "vault", id: "channels/googlechat/serviceAccount" },
}
```

---

## 14. 疑難排解

### 常見問題

| 問題 | 解決方案 |
|------|---------|
| Gateway 無法啟動 | `openclaw doctor --fix` |
| 頻道連線失敗 | `openclaw channels status --probe` 查看詳情 |
| Skills 不可用 | `openclaw skills list` 檢查依賴狀態 |
| Hooks 未觸發 | `openclaw hooks check` 確認啟用狀態 |
| 設定驗證失敗 | `openclaw config validate` 檢查語法 |
| BOOT.md 未執行 | 確認 `boot-md` hook 已啟用 |
| 記憶搜尋無結果 | 確認 `memorySearch.enabled: true` 且有記憶檔案 |

### 完整重設

```bash
# 僅重設設定
openclaw onboard --reset --reset-scope config

# 重設設定 + 認證 + 會話
openclaw onboard --reset --reset-scope config+creds+sessions

# 完全重設（含工作區）
openclaw onboard --reset --reset-scope full
```

### 診斷指令

```bash
# 完整診斷
openclaw doctor

# 自動修復
openclaw doctor --fix
openclaw doctor --yes          # 不確認直接修復

# Secrets 審計
openclaw secrets audit

# 設定驗證
openclaw config validate

# 查看設定值
openclaw config get agents.defaults.model
openclaw config get hooks.internal.entries
```

---

*本文件基於 OpenClaw `2026.3.31` 版本撰寫。完整英文文件請參考 https://docs.openclaw.ai*
