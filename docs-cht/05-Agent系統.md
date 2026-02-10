# Agent 系統

## 概述

Agent 系統是 Moltbot 的核心 AI 引擎，負責處理使用者訊息、呼叫工具、生成回應。系統基於 Pi Agent 框架，支援多種 LLM 提供者。

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
                    │  │  - Ollama (Local)            │  │
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
| `src/agents/moltbot-tools.ts` | 核心工具建立 |
| `src/agents/pi-embedded-runner/` | Pi Agent 嵌入式執行器 |
| `src/agents/workspace.ts` | Agent 工作區管理 |
| `src/agents/tool-policy.ts` | 工具使用策略 |

## Agent 配置

### 配置結構

```typescript
interface AgentConfig {
  id: string;                    // Agent ID
  name?: string;                 // 顯示名稱
  identity?: AgentIdentity;      // 身份設定
  model?: string;                // 預設模型
  workspace?: string;            // 工作目錄
  tools?: ToolsConfig;           // 工具配置
  hooks?: HooksConfig;           // 鉤子配置
}

interface AgentIdentity {
  name?: string;                 // Agent 名稱
  soul?: string;                 // 個性描述（SOUL.md）
  boot?: string;                 // 啟動指令（BOOT.md）
  user?: string;                 // 使用者資訊（USER.md）
}
```

### 配置範例

```json5
{
  "agents": {
    "default": {
      "id": "default",
      "name": "Moltbot",
      "model": "claude-sonnet",
      "workspace": "~/Projects",
      "identity": {
        "name": "Moltbot",
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

### 提示詞結構

```
[身份區塊]
你是 Moltbot，一個有幫助的 AI 助手。
<SOUL>...</SOUL>

[工具區塊]
你有以下工具可用：
- exec: 執行 Shell 命令
- read: 讀取檔案
- write: 寫入檔案
- browser: 控制瀏覽器
- search: 網路搜尋

[上下文區塊]
<context>
目前工作目錄: /Users/user/Projects
作業系統: macOS
時區: Asia/Taipei
</context>

[使用者區塊]
<user>
名稱: User
偏好: 繁體中文
</user>

[啟動指令]
<BOOT>...</BOOT>
```

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

### 工具建立

```typescript
// src/agents/moltbot-tools.ts
export function createMoltbotTools(options?: MoltbotToolsOptions) {
  const tools: Tool[] = [];
  
  // 檔案操作工具
  tools.push(createReadTool(options));
  tools.push(createWriteTool(options));
  tools.push(createEditTool(options));
  tools.push(createListTool(options));
  tools.push(createSearchTool(options));
  
  // 執行工具
  tools.push(createExecTool(options));
  
  // 瀏覽器工具
  if (options?.browserEnabled) {
    tools.push(createBrowserTool(options));
  }
  
  // 節點工具
  if (options?.nodeEnabled) {
    tools.push(...createNodeTools(options));
  }
  
  return tools;
}
```

### 工具定義範例

```typescript
// 讀取檔案工具
const readTool: Tool = {
  name: 'read',
  description: 'Read the contents of a file',
  parameters: Type.Object({
    path: Type.String({ description: 'File path to read' }),
    encoding: Type.Optional(Type.String({ 
      description: 'File encoding (default: utf-8)' 
    })),
  }),
  execute: async (params, context) => {
    const { path, encoding = 'utf-8' } = params;
    const content = await fs.readFile(path, encoding);
    return { content };
  },
};
```

## 工具策略

### 審批機制

```typescript
// src/agents/tool-policy.ts
export interface ToolPolicy {
  // 需要審批的工具
  requireApproval: string[];
  
  // 允許的工具
  allowlist?: string[];
  
  // 禁止的工具
  denylist?: string[];
  
  // 自動審批規則
  autoApprove?: AutoApproveRule[];
}

interface AutoApproveRule {
  tool: string;
  pattern?: string;  // 參數模式
  maxCount?: number; // 最大次數
}
```

### 沙箱執行

```typescript
// 執行工具時的安全限制
const sandbox: SandboxOptions = {
  workdir: options.workspaceDir,
  allowedPaths: [
    options.workspaceDir,
    '/tmp',
  ],
  deniedCommands: [
    'rm -rf /',
    'sudo',
  ],
  timeout: 30000,
};
```

## LLM 提供者

### 支援的提供者

| 提供者 | 模型 |
|--------|------|
| Anthropic | Claude Opus 4.5, Claude Sonnet 4.5, Claude Haiku 4.5 |
| OpenAI | GPT-4o, GPT-5.2 |
| Ollama | Llama 3.3, Qwen 2.5, DeepSeek R1 等 |
| OpenRouter | 多種模型 |
| Together AI | Llama、DeepSeek、Kimi 等開源模型 |
| Bedrock | Claude (AWS) |
| Qwen | 通義千問 |
| xAI | Grok |
| MiniMax | MiniMax Portal（OAuth 認證） |

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
        "anthropic/claude-sonnet-4-5": { alias: "Sonnet" },
        "together/zai-org/GLM-4.7": { alias: "GLM" }
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
   - 應用壓縮
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
  key: string;              // 會話鍵值
  agentId: string;          // Agent ID
  channelId: string;        // 頻道 ID
  chatId: string;           // 聊天 ID
  messages: Message[];      // 訊息歷史
  metadata: SessionMetadata;
  createdAt: Date;
  updatedAt: Date;
}

interface Message {
  role: 'user' | 'assistant' | 'system';
  content: string;
  toolCalls?: ToolCall[];
  timestamp: Date;
}
```

### 上下文壓縮

```typescript
// 當對話過長時自動壓縮
async function compactSession(session: Session, options: CompactOptions) {
  if (session.messages.length < options.threshold) {
    return session;
  }
  
  // 保留最近訊息
  const recentMessages = session.messages.slice(-options.keepRecent);
  
  // 摘要舊訊息
  const summary = await summarizeMessages(
    session.messages.slice(0, -options.keepRecent)
  );
  
  return {
    ...session,
    messages: [
      { role: 'system', content: `Previous context: ${summary}` },
      ...recentMessages,
    ],
  };
}
```

## 多 Agent 支援（子代理）

### 子代理概覽

Moltbot 支援完整的子代理系統，允許 Agent 生成並管理其他 Agent 來並行處理任務。

### Agent 之間通訊

```typescript
// 子 Agent 呼叫
const subagentTool: Tool = {
  name: 'subagent',
  description: 'Delegate task to another agent',
  parameters: Type.Object({
    agentId: Type.String(),
    task: Type.String(),
  }),
  execute: async (params, context) => {
    const result = await invokeAgent(params.agentId, params.task);
    return result;
  },
};
```

### Agent 協作模式

```
主 Agent
   │
   ├──▶ 子 Agent A (程式碼專家)
   │       └── 執行程式碼任務
   │
   ├──▶ 子 Agent B (研究專家)
   │       └── 執行搜尋任務
   │
   └──▶ 子 Agent C (寫作專家)
           └── 執行文件任務
```

### 子代理管理命令

```bash
# 列出所有子代理
/subagents list

# 停止子代理
/subagents stop <id>

# 查看子代理日誌
/subagents log <id>

# 查看子代理資訊
/subagents info <id>

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
          // 跨代理生成（允許其他 Agent 啟動子代理）
          allowAgents: ["main", "code-expert"],
          
          // 自動歸檔（分鐘）
          archiveAfterMinutes: 60,
          
          // 子代理工具策略
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
// 1. 定義工具
const myTool: Tool = {
  name: 'my_tool',
  description: 'My custom tool',
  parameters: Type.Object({
    param1: Type.String(),
    param2: Type.Optional(Type.Number()),
  }),
  execute: async (params, context) => {
    // 實作邏輯
    return { result: '...' };
  },
};

// 2. 註冊工具
export function createMoltbotTools(options) {
  const tools = [];
  // ...
  tools.push(myTool);
  return tools;
}
```

### 自訂 Agent

```typescript
// 在配置中定義
{
  "agents": {
    "code-expert": {
      "id": "code-expert",
      "name": "Code Expert",
      "identity": {
        "soul": "You are an expert programmer..."
      },
      "tools": {
        "exec": { "enabled": true },
        "browser": { "enabled": false }
      }
    }
  }
}
```
