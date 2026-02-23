# CLI 命令系統

## 概述

OpenClaw CLI 基於 Commander.js 建構，採用分層註冊和懶加載機制，確保快速啟動和模組化管理。

## 入口流程

```
openclaw.mjs (shebang 入口)
    │
    ▼
src/entry.ts
    │ 處理環境變數
    │ Windows 參數標準化
    │ 警告過濾
    ▼
src/cli/run-main.ts (runCli)
    │
    ▼
src/cli/program.ts (buildProgram)
    │ 建構 Commander.js 程式
    │ 註冊命令
    ▼
執行命令
```

## 命令結構

### 核心命令（立即註冊）

| 命令 | 說明 |
|------|------|
| `setup` | 互動式設定精靈 |
| `onboard` | 引導設定流程 |
| `configure` | 快速配置 |
| `config` | 配置管理 |
| `agent` | Agent 執行 |
| `agents` | Agent 管理（list, add, delete） |
| `message` | 訊息操作（send, read, edit, delete） |
| `status` | 狀態查詢 |
| `health` | 健康檢查 |
| `sessions` | 會話管理 |
| `browser` | 瀏覽器控制 |

### 子 CLI（懶加載）

| 命令 | 說明 |
|------|------|
| `gateway` | Gateway 伺服器管理 |
| `daemon` | 背景服務 |
| `logs` | 日誌查詢 |
| `system` | 系統資訊 |
| `models` | 模型管理 |
| `approvals` | 執行審批 |
| `nodes` | 節點管理 |
| `devices` | 裝置管理 |
| `sandbox` | 沙箱執行 |
| `tui` | 終端 UI |
| `cron` | 排程任務 |
| `dns` | DNS 設定 |
| `docs` | 文件查詢 |
| `hooks` | 鉤子管理 |
| `webhooks` | Webhook 設定 |
| `pairing` | 裝置配對 |
| `plugins` | 插件管理 |
| `channels` | 頻道管理 |
| `directory` | 目錄操作 |
| `security` | 安全設定 |
| `skills` | 技能管理 |
| `update` | 更新檢查（支援 `--dry-run`） |
| `memory` | 記憶系統 |
| `acp` | ACP 協議 |
| `completion` | Shell 自動完成 |
| `qr` | QR 碼工具 |

## 命令註冊機制

### 命令註冊表

```typescript
// src/cli/program.ts
export interface CommandRegistration {
  name: string;
  aliases?: string[];
  description: string;
  register: (program: Command) => void | Promise<void>;
}

export const commandRegistry: CommandRegistration[] = [
  {
    name: 'setup',
    description: 'Interactive setup wizard',
    register: (program) => {
      program
        .command('setup')
        .description('Run the interactive setup wizard')
        .option('--channel <channel>', 'Configure specific channel')
        .action(async (options) => {
          // 執行設定精靈
        });
    },
  },
  // ... 更多命令
];
```

### 懶加載機制

```typescript
// 懶加載子命令
export const lazySubClis: LazySubCliEntry[] = [
  { name: 'gateway', register: registerGatewaySubCli },
  { name: 'models', register: registerModelsSubCli },
  // ...
];

function registerLazyCommand(program: Command, entry: LazySubCliEntry) {
  const placeholder = program.command(entry.name);
  placeholder.description(`${entry.name} commands (lazy loaded)`);
  
  placeholder.action(async (...args) => {
    removeCommand(program, placeholder);
    await entry.register(program);
    await program.parseAsync(process.argv);
  });
}
```

## 常用命令詳解

### setup - 設定精靈

```bash
# 互動式設定
openclaw setup

# 設定特定頻道
openclaw setup --channel telegram

# 快速設定模式
openclaw setup --quick
```

### config - 配置管理

```bash
# 查看所有配置
openclaw config list

# 取得特定配置（敏感值自動脫敏）
openclaw config get models.default

# 設定配置值
openclaw config set models.default claude-sonnet

# 編輯配置檔
openclaw config edit
```

### agent - Agent 執行

```bash
# 執行 Agent（互動模式）
openclaw agent

# 發送單一訊息
openclaw agent --message "Hello"

# 指定 Agent
openclaw agent --agent custom-agent

# 指定思考等級
openclaw agent --message "Hello" --thinking low

# RPC 模式
openclaw agent --mode rpc --json
```

### message - 訊息操作

```bash
# 發送訊息
openclaw message send --channel telegram --chat 123456 "Hello"

# 讀取訊息
openclaw message read --channel whatsapp --chat +1234567890

# 編輯訊息
openclaw message edit --channel discord --id 987654 "Updated"

# 刪除訊息
openclaw message delete --channel slack --id 123
```

### gateway - Gateway 管理

```bash
# 啟動 Gateway
openclaw gateway run

# 指定埠號和綁定
openclaw gateway run --port 18789 --bind lan

# 強制啟動（忽略已執行的實例）
openclaw gateway run --force

# 查看狀態
openclaw gateway status

# 停止 Gateway
openclaw gateway stop
```

### channels - 頻道管理

```bash
# 查看頻道狀態
openclaw channels status

# 深度探測
openclaw channels status --deep

# 全部頻道狀態（只讀/可貼上）
openclaw channels status --all

# 連接頻道
openclaw channels connect telegram

# 斷開頻道
openclaw channels disconnect whatsapp
```

### plugins - 插件管理

```bash
# 列出已安裝插件
openclaw plugins list

# 安裝插件
openclaw plugins install matrix

# 移除插件
openclaw plugins remove msteams

# 更新插件
openclaw plugins update
```

### update - 更新管理

```bash
# 檢查更新
openclaw update

# 預演模式（不實際更新）
openclaw update --dry-run
```

### doctor - 診斷

```bash
# 完整診斷
openclaw doctor
```

## 選項和旗標

### 全域選項

| 選項 | 說明 |
|------|------|
| `--config <path>` | 指定配置檔路徑 |
| `--verbose` | 詳細輸出 |
| `--json` | JSON 格式輸出 |
| `--debug` | 除錯模式 |
| `--profile <name>` | 使用指定配置檔案 |

### 環境變數

| 變數 | 說明 |
|------|------|
| `CLAWDBOT_CONFIG_PATH` | 配置檔路徑 |
| `CLAWDBOT_PROFILE` | 配置檔案名稱 |
| `CLAWDBOT_LOG_LEVEL` | 日誌等級 |
| `CLAWDBOT_SKIP_CHANNELS` | 跳過頻道載入 |
| `CLAWDBOT_LIVE_TEST` | 啟用實時測試 |

## 開發自訂命令

### 新增核心命令

```typescript
// src/cli/my-command-cli.ts
import { Command } from 'commander';

export function registerMyCommand(program: Command) {
  program
    .command('my-command')
    .description('My custom command')
    .option('-n, --name <name>', 'Name parameter')
    .action(async (options) => {
      console.log(`Hello, ${options.name || 'World'}!`);
    });
}
```

## 進度顯示

CLI 使用統一的進度顯示工具（`src/cli/progress.ts`），基於 `osc-progress` 和 `@clack/prompts` spinner：

```typescript
import { createSpinner } from 'src/cli/progress';

const spinner = createSpinner('Processing...');
spinner.start();

try {
  await doSomething();
  spinner.stop('Done!');
} catch (error) {
  spinner.stop('Failed!');
  throw error;
}
```

## 終端 UI

CLI 使用共享的色彩調色盤（`src/terminal/palette.ts`），適用於 onboarding/config 提示和其他 TTY UI 輸出。狀態輸出使用 `src/terminal/table.ts` 提供 ANSI 安全的表格包裝。

## 測試命令

```bash
# 執行 CLI 測試
pnpm test src/cli/**/*.test.ts

# 測試特定命令
pnpm test src/commands/config/**/*.test.ts

# E2E 測試
pnpm test:e2e
```
