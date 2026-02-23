# Agent 系統

## 概述

Agent 系統是 OpenClaw 的核心 AI 引擎，負責處理使用者訊息、呼叫工具、生成回應。系統基於 Pi Agent 框架，支援多種 LLM 提供者。

## 架構

```
                    ┌─────────────────────────────────────┐
                    │           Agent System              │
                    │                                     │
使用者訊息 ─────────▶│  ┌──────────────────────────────┐  │
                    │  │      System Prompt Builder    │  │
                    │  │  - 身份設定                    │  │
                    │  │  - 工具描述                    │  │
                    │  │  - 上下文注入                  │  │
                    │  └──────────────────────────────┘  │
                    │              │                      │
                    │              ▼                      │
                    │  ┌──────────────────────────────┐  │
                    │  │        LLM Provider          │  │
                    │  │  - Anthropic (Claude)        │  │
                    │  │  - OpenAI (GPT)              │  │
                    │  │  - Mistral                   │  │
                    │  │  - Ollama (Local)            │  │
                    │  │  - Google Gemini             │  │
                    │  │  - OpenRouter               │  │
                    │  └──────────────────────────────┘  │
                    │              │                      │
                    │              ▼                      │
                    │  ┌──────────────────────────────┐  │
                    │  │       Tool Executor          │  │
                    │  │  - exec (Shell)              │  │
                    │  │  - read/write (File)         │  │
                    │  │  - browser                   │  │
                    │  │  - search                    │  │
                    │  │  - node.* (Device)           │  │
                    │  └──────────────────────────────┘  │
                    │              │                      │
                    │              ▼                      │
                    └──────────────┼──────────────────────┘
                                   │
                    助手回應 ◀──────┘
```

## 核心檔案

| 檔案 | 說明 |
|------|------|
| `src/agents/agent-scope.ts` | Agent 配置解析、路徑解析 |
| `src/agents/system-prompt.ts` | 系統提示詞建構 |
| `src/agents/openclaw-tools.ts` | 核心工具建立 |
| `src/agents/pi-embedded-runner/` | Pi Agent 嵌入式執行器 |
| `src/agents/workspace.ts` | Agent 工作區管理 |
| `src/agents/tool-policy.ts` | 工具使用策略 |

## Agent 配置

### 配置結構

```typescript
interface AgentConfig {
  id: string;
  name?: string;
  identity?: AgentIdentity;
  model?: string;
  workspace?: string;
  tools?: ToolsConfig;
  hooks?: HooksConfig;
}

interface AgentIdentity {
  name?: string;
  soul?: string;   // SOUL.md
  boot?: string;   // BOOT.md / BOOTSTRAP.md
  user?: string;   // USER.md
}
```

### 配置範例

```json5
{
  "agents": {
    "default": {
      "id": "default",
      "name": "OpenClaw",
      "model": "claude-sonnet",
      "workspace": "~/Projects",
      "identity": {
        "name": "OpenClaw",
        "soul": "You are a helpful AI assistant."
      },
      "tools": {
        "exec": { "enabled": true },
        "browser": { "enabled": true },
        "node": { "enabled": true }
      }
    }
  }
}
```

## 系統提示詞

### 建構流程

```typescript
// src/agents/system-prompt.ts
export async function buildSystemPrompt(options: SystemPromptOptions) {
  const parts: string[] = [];
  
  // 1. 身份區塊
  parts.push(buildIdentitySection(options.identity));
  
  // 2. 工具描述
  parts.push(buildToolsSection(options.tools));
  
  // 3. 上下文資訊
  if (options.context) {
    parts.push(buildContextSection(options.context));
  }
  
  // 4. 使用者資訊
  if (options.user) {
    parts.push(buildUserSection(options.user));
  }
  
  // 5. 特殊指令
  if (options.boot) {
    parts.push(options.boot);
  }
  
  return parts.join('\n\n');
}
```

### 工作區目錄結構

Agent 工作區支援以下特殊檔案：

| 檔案 | 用途 |
|------|------|
| `AGENTS.md` | Agent 指示與行為規則 |
| `SOUL.md` | 個性描述 |
| `BOOTSTRAP.md` / `BOOT.md` | 啟動指令 |
| `IDENTITY.md` | 身份設定 |
| `USER.md` | 使用者資訊 |
| `TOOLS.md` | 工具使用指引 |

## 核心工具

### 工具清單

| 工具 | 說明 |
|------|------|
| `exec` | 執行 Shell 命令 |
| `read` | 讀取檔案 |
| `write` | 寫入檔案 |
| `edit` | 編輯檔案（patch） |
| `list` | 列出目錄 |
| `search` | 搜尋檔案 |
| `glob` | 模式匹配搜尋 |
| `browser` | 瀏覽器控制 |
| `web_search` | 網路搜尋 |
| `fetch` | HTTP 請求 |
| `node.*` | 裝置能力（相機、位置等） |
| `canvas` | Canvas A2UI |
| `cron` | 排程任務 |
| `sessions` | 會話管理 |

### 工具配置檔

| 配置檔 | 包含工具 |
|--------|---------|
| `minimal` | 基本檔案操作 |
| `coding` | 檔案 + exec + search |
| `messaging` | 頻道互動 |
| `full` | 所有工具 |

### 按提供商限制工具

```json5
{
  tools: {
    byProvider: {
      "anthropic": ["exec", "read", "write", "browser"],
      "ollama": ["read", "write"]
    }
  }
}
```

## 工具策略

### 審批機制

```typescript
// src/agents/tool-policy.ts
export interface ToolPolicy {
  requireApproval: string[];
  allowlist?: string[];
  denylist?: string[];
  autoApprove?: AutoApproveRule[];
}
```

### 沙箱執行

```typescript
const sandbox: SandboxOptions = {
  workdir: options.workspaceDir,
  allowedPaths: [options.workspaceDir, '/tmp'],
  deniedCommands: ['rm -rf /', 'sudo'],
  timeout: 30000,
};
```

## LLM 提供者

### 支援的提供者

| 提供者 | 模型 |
|--------|------|
| Anthropic | Claude Opus 4.5, Claude Sonnet 4.5, Claude Haiku 4.5 |
| OpenAI | GPT-4o, GPT-5.2 |
| Mistral | Mistral 模型（新增） |
| Ollama | Llama 3.3, Qwen 2.5, DeepSeek R1 等 |
| OpenRouter | 多種模型 |
| Together AI | Llama、DeepSeek、Kimi 等開源模型 |
| Bedrock | Claude (AWS) |
| Qwen | 通義千問 |
| xAI | Grok |
| MiniMax | MiniMax Portal（OAuth 認證） |
| Google Gemini | Gemini 模型（CLI Auth / Antigravity Auth） |
| Copilot | Copilot Proxy |

### 模型配置

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-sonnet-4-5",
        fallbacks: [
          "anthropic/claude-opus-4-5",
          "ollama/qwen2.5-coder:32b"
        ]
      },
      models: {
        "anthropic/claude-opus-4-5": { alias: "Opus" },
        "anthropic/claude-sonnet-4-5": { alias: "Sonnet" }
      }
    }
  }
}
```

## Agent 執行流程

```
1. 接收使用者訊息
        │
        ▼
2. 載入 Agent 配置
   - 解析 agentId
   - 載入身份設定
   - 初始化工具
        │
        ▼
3. 建構系統提示詞
   - 身份區塊
   - 工具描述
   - 上下文注入
        │
        ▼
4. 組裝對話歷史
   - 載入會話
   - 應用壓縮 (compaction)
        │
        ▼
5. 呼叫 LLM
   - 選擇模型
   - 發送請求
        │
        ▼
6. 處理回應
   ├─ 文字回應 → 輸出
   └─ 工具呼叫 → 執行工具 → 回到步驟 5
        │
        ▼
7. 儲存對話
   - 更新會話
   - 觸發鉤子
```

## 會話管理

### 會話結構

```typescript
interface Session {
  key: string;
  agentId: string;
  channelId: string;
  chatId: string;
  messages: Message[];
  metadata: SessionMetadata;
  createdAt: Date;
  updatedAt: Date;
}
```

### 上下文壓縮 (Compaction)

當對話過長時，系統自動執行 compaction：
- 保留最近訊息
- 摘要舊訊息
- 只計算已完成的自動壓縮次數
- 忽略超大檢查中的工具結果細節
- 對齊壓縮下限指引

### 會話儲存

會話日誌儲存在 `~/.openclaw/agents/<agentId>/sessions/<SessionId>.jsonl`。

## 多 Agent 支援（子代理）

### 子代理概覽

OpenClaw 支援完整的子代理系統，允許 Agent 生成並管理其他 Agent 來並行處理任務。

### 子代理管理命令

```bash
# 列出所有子代理
/subagents list

# 停止子代理
/subagents stop <id>

# 查看子代理日誌
/subagents log <id>

# 向子代理發送訊息
/subagents send <id> <message>
```

### 子代理配置

```json5
{
  agents: {
    defaults: {
      tools: {
        subagents: {
          allowAgents: ["main", "code-expert"],
          archiveAfterMinutes: 60,
          tools: {
            allow: ["read", "write", "exec"],
            deny: ["browser"]
          }
        }
      }
    }
  }
}
```

## 開發指南

### 新增工具

```typescript
const myTool: Tool = {
  name: 'my_tool',
  description: 'My custom tool',
  parameters: Type.Object({
    param1: Type.String(),
    param2: Type.Optional(Type.Number()),
  }),
  execute: async (params, context) => {
    return { result: '...' };
  },
};
```

**工具 Schema 注意事項**：
- 避免 `Type.Union`；使用 `stringEnum`/`optionalStringEnum` 代替
- 頂層 schema 必須是 `type: "object"` 搭配 `properties`
- 使用 `Type.Optional(...)` 而非 `... | null`
- 避免使用 `format` 作為屬性名稱（部分驗證器視為保留字）
