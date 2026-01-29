# Moltbot 專案技術文件（繁體中文）

本目錄包含 Moltbot 專案的完整技術分析文件，以繁體中文撰寫，幫助開發者快速理解並繼續開發此專案。

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

## 快速開始

```bash
# 安裝依賴
pnpm install

# 開發模式執行 CLI
pnpm moltbot --help

# 建置專案
pnpm build

# 執行測試
pnpm test

# 格式化與 lint
pnpm lint
pnpm format
```

## 專案結構

```
moltbot/
├── src/                 # TypeScript 原始碼
│   ├── cli/             # CLI 入口與命令註冊
│   ├── commands/        # 命令實作
│   ├── agents/          # AI Agent 核心
│   ├── gateway/         # Gateway 伺服器
│   ├── config/          # 配置載入與驗證
│   ├── channels/        # 通訊頻道共用邏輯
│   ├── telegram/        # Telegram 頻道
│   ├── discord/         # Discord 頻道
│   ├── slack/           # Slack 頻道
│   ├── signal/          # Signal 頻道
│   ├── imessage/        # iMessage 頻道
│   ├── web/             # WhatsApp (Baileys) 頻道
│   └── infra/           # 基礎設施
├── extensions/          # 擴充套件（插件）
├── apps/                # 原生應用程式
│   ├── ios/             # iOS 應用 (Swift)
│   ├── android/         # Android 應用 (Kotlin)
│   └── macos/           # macOS 應用 (Swift)
├── docs/                # 英文文件 (Mintlify)
├── docs-cht/            # 繁體中文技術文件
├── test/                # E2E 測試
└── ui/                  # Web UI (Lit)
```

## 相關連結

- [GitHub 儲存庫](https://github.com/moltbot/moltbot)
- [官方文件](https://docs.molt.bot)
- [Discord 社群](https://discord.gg/qkhbAGHRBT)
