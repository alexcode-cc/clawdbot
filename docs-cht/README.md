# OpenClaw 專案技術文件（繁體中文）

本目錄包含 OpenClaw 專案的完整技術分析文件，以繁體中文撰寫，幫助開發者快速理解並繼續開發此專案。

## 文件索引

### 核心概念
- [01 - 專案概覽](./01-專案概覽.md) - 專案介紹、功能特色、技術棧、版本資訊
- [02 - 架構設計](./02-架構設計.md) - 分層架構、模組組織、核心設計模式、Plugin Capability 系統

### 核心模組
- [03 - CLI 命令系統](./03-CLI命令系統.md) - 命令註冊、懶加載、路由機制
- [04 - Gateway 伺服器](./04-Gateway伺服器.md) - WebSocket、RPC、健康檢查、Kubernetes 支援、安全強化
- [05 - Agent 系統](./05-Agent系統.md) - AI Agent、26+ 提供者、工具策略、Context Engine、Compaction、Fast Mode、Plugin Capability
- [06 - 通訊頻道](./06-通訊頻道.md) - 8 內建 + 35+ 擴充頻道、Web Search 插件、Outbound 插件化

### 擴充與應用
- [07 - 擴充套件開發](./07-擴充套件開發.md) - 插件架構、Plugin Capability（Web Search/Image Gen/TTS/Media）、ClawHub 市場
- [08 - 應用程式](./08-應用程式.md) - iOS、Android（含記憶體/安全修復）、macOS 應用

### 設定與配置
- [09 - 開發環境設定](./09-開發環境設定.md) - 安裝、建置、測試、Mobile 開發
- [10 - 配置系統](./10-配置系統.md) - 配置結構、模型失效容錯備份（Failover）、環境變數、SecretRef
- [11 - API 參考](./11-API參考.md) - Gateway RPC、Plugin SDK

### 指令與操作
- [12 - CLI 指令詳解](./12-CLI指令詳解.md) - 所有 CLI 指令完整參考、選項說明、情境範例
- [13 - 大模型設定與驗證詳解](./13-大模型設定與驗證詳解.md) - 26+ 提供者設定、認證流程、模型切換、驗證方法

### 安裝設定與系統
- [17 - Onboard 安裝設定詳解](./17-Onboard安裝設定詳解.md) - `openclaw onboard` 完整指南（50+ Skills、4 Hooks、Boot.md、Gateway 設定）
- [18 - 長期記憶系統詳解](./18-長期記憶系統詳解.md) - 記憶儲存格式、6 嵌入提供者、混合搜尋、自動沖洗、濃縮清除策略
- [19 - TUI 斜線指令完整指南](./19-TUI斜線指令完整指南.md) - 60+ 斜線指令語法、參數、用途（含 /btw、/exec、/queue、/acp、/tts）

### 實戰指南
- [14 - 從零開始設定實作](./14-從零開始設定實作.md) - 完整工作流程設定（Ollama/Claude Code + Discord + Gmail + GitHub）
- [15 - 從零開始設定實作（Mac Mini M4 16G）](./15-從零開始設定實作(Mac-Mini-M4-16G).md) - Mac Mini M4 16GB 專用設定指南（含硬體最佳化）
- [16 - 從零開始設定實作（Mac Mini M4 24G）](./16-從零開始設定實作(Mac-Mini-M4-24G).md) - Mac Mini M4 24GB 專用設定指南（32B 模型 + 記憶體深度優化）

### 提交分析
- [2026-03-03 提交分析](./commit-analyze-20260303.md) - 682 個提交分析（初始分析）
- [2026-03-04 提交分析](./commit-analyze-20260304.md) - 2026.3.2 → 2026.3.3 更新分析
- [2026-03-10 提交分析](./commit-analyze-20260310.md) - 1,120 個提交分析（2026.3.3 → 2026.3.9）
- [2026-03-13 提交分析](./commit-analyze-20260313.md) - 1,101 個提交分析（2026.3.9 → 2026.3.13）
- [2026-03-23 提交分析](./commit-analyze-20260323.md) - 2,694 個提交分析（2026.3.13 → 2026.3.23）
- [2026-03-26 提交分析](./commit-analyze-20260326.md) - 463 個提交分析（2026.3.23 → 2026.3.24）
- [2026-03-28 提交分析](./commit-analyze-20260328.md) - 1,267 個提交分析（2026.3.24 → 2026.3.28）

## 版本

目前版本：`2026.3.28`

## 快速開始

```bash
# 安裝依賴
pnpm install

# 開發模式執行 CLI
pnpm openclaw --help

# 建置專案
pnpm build

# 執行測試
pnpm test

# 格式化與 lint 檢查
pnpm check
pnpm format
```

## 專案結構

```
openclaw/
├── src/                 # TypeScript 原始碼
│   ├── cli/             # CLI 入口與命令註冊
│   ├── commands/        # 命令實作
│   ├── agents/          # AI Agent 核心
│   ├── gateway/         # Gateway 伺服器
│   ├── config/          # 配置載入與驗證
│   ├── channels/        # 通訊頻道共用邏輯
│   ├── routing/         # 訊息路由
│   ├── telegram/        # Telegram 頻道
│   ├── discord/         # Discord 頻道
│   ├── slack/           # Slack 頻道
│   ├── signal/          # Signal 頻道
│   ├── imessage/        # iMessage 頻道
│   ├── web/             # WhatsApp (Baileys) 頻道
│   ├── line/            # LINE 頻道
│   ├── acp/             # ACP 協議
│   ├── browser/         # 瀏覽器控制
│   ├── tui/             # 終端 UI (TUI)
│   ├── process/         # 進程管理與監督
│   ├── media/           # 媒體處理（Plugin Capability）
│   ├── memory/          # 記憶系統（索引、搜尋、嵌入）
│   ├── secrets/         # Secrets 管理
│   ├── cron/            # 排程任務
│   ├── hooks/           # Hooks 系統（boot-md、session-memory 等）
│   ├── plugins/         # 插件系統（含懶加載邊界）
│   ├── plugin-sdk/      # 插件開發 SDK（Capability 註冊）
│   ├── providers/       # LLM 提供者（含 OpenAI WS 串流）
│   ├── shared/          # 共用輔助函式
│   └── infra/           # 基礎設施
├── extensions/          # 擴充套件（50+ 插件）
├── apps/                # 原生應用程式
│   ├── ios/             # iOS 應用 (Swift)
│   ├── android/         # Android 應用 (Kotlin)
│   └── macos/           # macOS 應用 (Swift)
├── packages/            # 內部套件
├── docs/                # 英文文件 (Mintlify)
├── docs-cht/            # 繁體中文技術文件
├── ui/                  # Web UI (Lit)
├── vendor/              # 第三方供應商程式碼
├── scripts/             # 建置與工具腳本
└── test/                # E2E 測試
```

## 相關連結

- [GitHub 儲存庫](https://github.com/openclaw/openclaw)
- [官方文件](https://docs.openclaw.ai)
- [Discord 社群](https://discord.gg/qkhbAGHRBT)
