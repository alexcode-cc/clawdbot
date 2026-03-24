# OpenClaw 專案技術文件（繁體中文）

本目錄包含 OpenClaw 專案的完整技術分析文件，以繁體中文撰寫，幫助開發者快速理解並繼續開發此專案。

## 文件索引

### 核心概念
- [專案概覽](./01-專案概覽.md) - 專案介紹、功能特色、技術棧
- [架構設計](./02-架構設計.md) - 整體架構、模組組織、設計模式

### 核心模組
- [CLI 命令系統](./03-CLI命令系統.md) - 命令註冊、懶加載、路由機制
- [Gateway 伺服器](./04-Gateway伺服器.md) - WebSocket、RPC、會話管理
- [Agent 系統](./05-Agent系統.md) - AI Agent、工具、系統提示詞
- [通訊頻道](./06-通訊頻道.md) - 內建頻道、擴充頻道、訊息路由

### 擴充系統
- [擴充套件開發](./07-擴充套件開發.md) - 插件架構、API、範例
- [應用程式](./08-應用程式.md) - iOS、Android、macOS 應用

### 開發指南
- [開發環境設定](./09-開發環境設定.md) - 安裝、建置、測試
- [配置系統](./10-配置系統.md) - 配置結構、驗證、環境變數
- [API 參考](./11-API參考.md) - Gateway RPC、Plugin SDK

### 指令與操作
- [CLI 指令詳解](./12-CLI指令詳解.md) - 所有 CLI 指令完整參考、選項說明、情境範例
- [大模型設定與驗證詳解](./13-大模型設定與驗證詳解.md) - 23 個提供者設定、認證流程、模型切換、驗證方法

### 安裝設定與系統
- [Onboard 安裝設定詳解](./17-Onboard安裝設定詳解.md) - `openclaw onboard` 完整指南（Skills、Hooks、Boot.md、Session Memory）
- [長期記憶系統詳解](./18-長期記憶系統詳解.md) - 記憶儲存、索引搜尋、自動沖洗、濃縮清除策略

### 實戰指南
- [從零開始設定實作](./14-從零開始設定實作.md) - 完整工作流程設定（Ollama/Claude Code + Discord + Gmail + GitHub）
- [從零開始設定實作（Mac Mini M4 16G）](./15-從零開始設定實作(Mac-Mini-M4-16G).md) - Mac Mini M4 16GB 專用設定指南（含硬體最佳化）
- [從零開始設定實作（Mac Mini M4 24G）](./16-從零開始設定實作(Mac-Mini-M4-24G).md) - Mac Mini M4 24GB 專用設定指南（32B 模型 + 記憶體深度優化）

### 提交分析
- [2026-03-03 提交分析](./commit-analyze-20260303.md) - 682 個提交分析（初始分析）
- [2026-03-04 提交分析](./commit-analyze-20260304.md) - 2026.3.2 → 2026.3.3 更新分析
- [2026-03-10 提交分析](./commit-analyze-20260310.md) - 1,120 個提交分析（2026.3.3 → 2026.3.9）
- [2026-03-13 提交分析](./commit-analyze-20260313.md) - 1,101 個提交分析（2026.3.9 → 2026.3.13）
- [2026-03-23 提交分析](./commit-analyze-20260323.md) - 2,694 個提交分析（2026.3.13 → 2026.3.23）

## 版本

目前版本：`2026.3.23`

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
│   ├── media/           # 媒體處理
│   ├── memory/          # 記憶系統
│   ├── secrets/         # Secrets 管理
│   ├── cron/            # 排程任務
│   ├── plugins/         # 插件系統
│   ├── plugin-sdk/      # 插件開發 SDK
│   ├── providers/       # LLM 提供者
│   ├── shared/          # 共用輔助函式
│   └── infra/           # 基礎設施
├── extensions/          # 擴充套件（插件）
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
