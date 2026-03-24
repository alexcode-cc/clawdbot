# Agent 系統

## 概述

Agent 系統是 OpenClaw 的核心 AI 引擎，負責處理使用者訊息、呼叫工具、生成回應。系統基於 Pi Agent 框架，支援多種 LLM 提供者。

## 架構

```
                    ┌─────────────────────────────────────┐
                    │           Agent System              │
                    │                                     │
使用者訊息 ────────▶│  ┌──────────────────────────────┐   │
                    │  │      System Prompt Builder   │   │
                    │  │  - 身份設定                   │   │
                    │  │  - 工具描述                   │   │
                    │  │  - 上下文注入                 │   │
                    │  └──────────────────────────────┘   │
                    │              │                      │
                    │              ▼                      │
                    │  ┌──────────────────────────────┐   │
                    │  │        LLM Provider          │   │
                    │  │  - Anthropic (Claude)        │   │
                    │  │  - OpenAI (GPT)              │   │
                    │  │  - Mistral                   │   │
                    │  │  - Ollama (Local)            │   │
                    │  │  - Google Gemini             │   │
                    │  │  - OpenRouter                │   │
                    │  └──────────────────────────────┘   │
                    │              │                      │
                    │              ▼                      │
                    │  ┌──────────────────────────────┐   │
                    │  │       Tool Executor          │   │
                    │  │  - exec (Shell)              │   │
                    │  │  - read/write (File)         │   │
                    │  │  - browser                   │   │
                    │  │  - search                    │   │
                    │  │  - node.* (Device)           │   │
                    │  └──────────────────────────────┘   │
                    │              │                      │
                    │              ▼                      │
                    └──────────────┼──────────────────────┘
                                   │
                    助手回應 ◀─────┘
```

## 核心檔案

| 檔案                             | 說明                     |
| -------------------------------- | ------------------------ |
| `src/agents/agent-scope.ts`      | Agent 配置解析、路徑解析 |
| `src/agents/system-prompt.ts`    | 系統提示詞建構           |
| `src/agents/openclaw-tools.ts`   | 核心工具建立             |
| `src/agents/pi-embedded-runner/` | Pi Agent 嵌入式執行器    |
| `src/agents/workspace.ts`        | Agent 工作區管理         |
| `src/agents/tool-policy.ts`      | 工具使用策略             |

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
  soul?: string; // SOUL.md
  boot?: string; // BOOT.md / BOOTSTRAP.md
  user?: string; // USER.md
}
```

### 配置範例

```json5
{
  agents: {
    default: {
      id: "default",
      name: "OpenClaw",
      model: "claude-sonnet",
      workspace: "~/Projects",
      identity: {
        name: "OpenClaw",
        soul: "You are a helpful AI assistant.",
      },
      tools: {
        exec: { enabled: true },
        browser: { enabled: true },
        node: { enabled: true },
      },
    },
  },
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

  return parts.join("\n\n");
}
```

### 工作區目錄結構

Agent 工作區支援以下特殊檔案：

| 檔案                       | 用途                 |
| -------------------------- | -------------------- |
| `AGENTS.md`                | Agent 指示與行為規則 |
| `SOUL.md`                  | 個性描述             |
| `BOOTSTRAP.md` / `BOOT.md` | 啟動指令             |
| `IDENTITY.md`              | 身份設定             |
| `USER.md`                  | 使用者資訊           |
| `TOOLS.md`                 | 工具使用指引         |

## 核心工具

### 工具清單

| 工具          | 說明                                                                                                        |
| ------------- | ----------------------------------------------------------------------------------------------------------- |
| `exec`        | 執行 Shell 命令                                                                                             |
| `read`        | 讀取檔案                                                                                                    |
| `write`       | 寫入檔案                                                                                                    |
| `edit`        | 編輯檔案（patch）                                                                                           |
| `list`        | 列出目錄                                                                                                    |
| `search`      | 搜尋檔案                                                                                                    |
| `glob`        | 模式匹配搜尋                                                                                                |
| `browser`     | 瀏覽器控制                                                                                                  |
| `web_search`  | 網路搜尋（Plugin Capability 系統）                                                                          |
| `image_generate` | 圖片生成（Plugin Capability 系統，Google/fal providers）                                                  |
| `fetch`       | HTTP 請求                                                                                                   |
| `node.*`      | 裝置能力（相機、位置等）                                                                                    |
| `canvas`      | Canvas A2UI                                                                                                 |
| `cron`        | 排程任務                                                                                                    |
| `sessions`    | 會話管理                                                                                                    |
| `diffs`       | 差異渲染（before/after 或 unified patch，支援 PDF 輸出、品質自訂 `fileQuality`/`fileScale`/`fileMaxWidth`） |
| `pdf`         | PDF 文件分析（原生 Anthropic/Google 支援，非原生自動降級）                                                  |
| `agents_list` | 代理清單                                                                                                    |
| `message`     | 訊息發送                                                                                                    |
| `gateway`     | Gateway 操作                                                                                                |

### 工具配置檔

| 配置檔      | 包含工具                                     |
| ----------- | -------------------------------------------- |
| `minimal`   | 基本檔案操作                                 |
| `coding`    | 檔案 + exec + search                         |
| `messaging` | 頻道互動（新安裝預設值，含 Onboarding 預設） |
| `full`      | 所有工具                                     |

> **注意**：自 2026-03-03 起，新安裝的 `tools.profile` 預設改為 `messaging`。需要 coding/system 工具需明確配置。

### 按提供商限制工具

```json5
{
  tools: {
    byProvider: {
      anthropic: ["exec", "read", "write", "browser"],
      ollama: ["read", "write"],
    },
  },
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
  allowedPaths: [options.workspaceDir, "/tmp"],
  deniedCommands: ["rm -rf /", "sudo"],
  timeout: 30000,
};
```

## LLM 提供者

### 支援的提供者

| 提供者        | 模型                                                                        |
| ------------- | --------------------------------------------------------------------------- |
| Anthropic     | Claude 4.6（預設 adaptive thinking）、Claude Opus/Sonnet/Haiku 4.5          |
| OpenAI        | GPT-4o, GPT-5.2, GPT-5.4（預設 WebSocket 串流；`transport: "auto"`，SSE 降級，1M context） |
| Mistral       | Mistral 模型                                                                |
| Ollama        | Llama 3.3, Qwen 2.5, DeepSeek R1 等                                         |
| OpenRouter    | 多種模型（x-ai/grok 自動跳過 `reasoning.effort` 注入）                      |
| Together AI   | Llama、DeepSeek、Kimi 等開源模型                                            |
| Bedrock       | Claude (AWS)（包含 Bedrock Claude 4.6 refs）                                |
| Qwen          | 通義千問                                                                    |
| xAI           | Grok（含 fast mode）                                                        |
| MiniMax       | MiniMax Portal（OAuth 認證，含 fast mode）；支援 `MiniMax-M2.5-highspeed`   |
| Google Gemini | Gemini 模型（CLI Auth / Antigravity Auth；null properties 自動強制為 `{}`）；含 Gemini 3.1 Flash-Lite |
| Vercel AI     | Vercel AI Gateway catalog 自動發現                                          |
| Venice        | 預設 kimi-k2-5，發現限制和工具支援強化                                       |
| Kilocode      | 所有模型可用                                                                |
| Copilot       | Copilot Proxy（支援 token 過期自動重新整理）                                |
| Moonshot/Kimi | 原生 thinking payload 相容，failover stop reason error 視為 timeout         |
| Opencode Go   | Opencode Go provider                                                       |
| ModelStudio   | 阿里巴巴百煉（Coding Plan + 標準 DashScope 端點）                          |
| Poe           | Poe 模型（402 'used up your points' billing 識別）                          |
| Anthropic Vertex | Claude via GCP Vertex AI                                                |
| GitHub Copilot | 動態 model ID 解析                                                         |
| Xiaomi MiMo   | MiMo V2 Pro / Omni（OpenAI completions API）                               |

### 模型配置

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-sonnet-4-5",
        fallbacks: ["anthropic/claude-opus-4-5", "ollama/qwen2.5-coder:32b"],
      },
      models: {
        "anthropic/claude-opus-4-5": { alias: "Opus" },
        "anthropic/claude-sonnet-4-5": { alias: "Sonnet" },
      },
    },
  },
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
- OpenAI Responses 支援伺服器端 compaction（`context_management`，per-model opt-out/threshold 覆寫）

### 會話儲存

會話日誌儲存在 `~/.openclaw/agents/<agentId>/sessions/<SessionId>.jsonl`。

## Thinking 模式

### Claude 4.6 自適應思考

Claude 4.6 模型（包含 Bedrock Claude 4.6 refs）預設使用 `adaptive` 思考等級。其他支援推理的模型預設為 `low`。

```json5
{
  agents: {
    defaults: {
      thinking: "adaptive", // adaptive | low | off
    },
  },
}
```

當提供者不支援指定的 thinking level 時，系統會自動降級為 `think=off` 以避免硬失敗。

### OpenAI WebSocket 串流

OpenAI Responses API 預設使用 WebSocket 串流（`transport: "auto"`，SSE 降級），並支援可選的 WebSocket 預熱（`openaiWsWarmup`）。

## 多 Agent 支援（子代理）

### 子代理概覽

OpenClaw 支援完整的子代理系統，允許 Agent 生成並管理其他 Agent 來並行處理任務。子代理完成事件使用結構化 `task_completion` 內部事件，統一直接和佇列通知路徑。

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
            deny: ["browser"],
          },
        },
      },
    },
  },
}
```

## 開發指南

### 新增工具

```typescript
const myTool: Tool = {
  name: "my_tool",
  description: "My custom tool",
  parameters: Type.Object({
    param1: Type.String(),
    param2: Type.Optional(Type.Number()),
  }),
  execute: async (params, context) => {
    return { result: "..." };
  },
};
```

**工具 Schema 注意事項**：

- 避免 `Type.Union`；使用 `stringEnum`/`optionalStringEnum` 代替
- 頂層 schema 必須是 `type: "object"` 搭配 `properties`
- 使用 `Type.Optional(...)` 而非 `... | null`
- 避免使用 `format` 作為屬性名稱（部分驗證器視為保留字）

## 模型失敗降級

Agent 系統支援多層降級策略：

- **網路錯誤降級**: `ECONNREFUSED`、`ENETUNREACH`、`EHOSTUNREACH`、`ENETRESET`、`EAI_AGAIN` 觸發 fallback
- **速率限制分類**: 精確匹配 TPM 作為獨立 token/片語，避免子字串誤判
- **認證過期重試**: Copilot token 過期後自動刷新並重新執行
- **Thinking 降級**: 不支援的 thinking level 自動降級為 `off`

## Shell 環境標記

所有 shell-like 執行環境（`exec`、`acp`、`acp-client`、`tui-local`）會注入 `OPENCLAW_SHELL` 環境變數，讓 shell startup/config 規則可以識別 OpenClaw 執行上下文。

## PDF 分析工具

新增一等公民 `pdf` 工具，支援原生 Anthropic 與 Google 提供者，非原生模型自動降級為提取模式。

```json5
{
  agents: {
    defaults: {
      pdfModel: "anthropic/claude-sonnet-4-5", // PDF 分析首選模型
      pdfMaxBytesMb: 10, // 最大 PDF 大小（MB）
      pdfMaxPages: 50, // 最大頁數
    },
  },
}
```

## 音頻轉錄回聲

```json5
{
  tools: {
    media: {
      audio: {
        echoTranscript: true, // 在代理處理前向聊天發送轉錄
        echoFormat: "blockquote", // plain | blockquote | code
      },
    },
  },
}
```

## sessions_spawn 內聯附件

子代理執行時可傳遞內聯檔案附件：

```json5
{
  tools: {
    sessions_spawn: {
      attachments: {
        maxSizeBytes: 1048576, // 每附件最大大小（bytes）
        maxCount: 5, // 最大附件數
      },
    },
  },
}
```

## 記憶搜尋 Ollama 嵌入

```json5
{
  memorySearch: {
    provider: "ollama", // 使用 Ollama 作為嵌入向量提供者
    fallback: "ollama", // 嵌入 fallback 也使用 Ollama
  },
}
```

尊重 `models.providers.ollama` 設定。

## 記憶沖洗強制預壓縮

防止長對話在 token 快照過期時卡住：

```json5
{
  agents: {
    defaults: {
      compaction: {
        memoryFlush: {
          forceFlushTranscriptBytes: 2097152, // 2MB，預設值
        },
      },
    },
  },
}
```

## 工具迴圈偵測

可選的護欄，防止 agent 卡在重複工具呼叫迴圈中。預設**停用**。

```json5
{
  tools: {
    loopDetection: {
      enabled: false, // 啟用偵測
      historySize: 30, // 追蹤歷史大小
      warningThreshold: 10, // 警告門檻
      criticalThreshold: 20, // 嚴重門檻
      globalCircuitBreakerThreshold: 30, // 全域斷路器
      detectors: {
        genericRepeat: true, // 通用重複偵測
        knownPollNoProgress: true, // 已知輪詢無進展
        pingPong: true, // 乒乓模式
      },
    },
  },
}
```

支援 per-agent 覆寫（`agents.list[].tools.loopDetection`）。

## 工具結果截斷

對過大的工具結果使用 head+tail 截斷，保留重要的尾部診斷資訊，同時保持可配置的截斷選項。

## Bootstrap 截斷警告

```json5
{
  agents: {
    defaults: {
      bootstrapPromptTruncationWarning: "once", // off | once | always
    },
  },
}
```

統一 bootstrap 預算/截斷分析（embedded + CLI runtime、`/context`、`openclaw doctor`），持久化警告簽名元資料，確保截斷警告跨 turn 一致且不重複。

## Hooks 訊息生命週期

新增的 hook 事件：

- `message:transcribed` — 音頻轉錄完成後觸發
- `message:preprocessed` — 訊息前處理完成後觸發
- `message:sent` — 新增 `isGroup`（是否為群組）、`groupId`（群組 ID）欄位
- `session_start` / `session_end` — 新增 `sessionKey` 欄位供路由身份關聯

## Web 搜尋提供者

`web_search` 工具現在透過 **Plugin Capability 系統** 運作，支援以下提供者：

| 提供者                | 類型            | 特色                                       |
| --------------------- | --------------- | ------------------------------------------ |
| Brave Search          | Bundled plugin（預設啟用） | 快速結構化結果 + LLM Context API 模式 |
| DuckDuckGo            | Bundled plugin（experimental） | 免費，不需 API key             |
| Exa                   | Bundled plugin  | 語意搜尋 + extract 功能                    |
| Tavily                | Bundled plugin  | search + extract 雙工具                    |
| Perplexity Search API | 內建            | 結構化結果 + 語言/地區/時間篩選 + 內容提取 |
| Gemini                | 內建            | Google Search grounding                    |
| Grok                  | 內建            | xAI web-grounded 回應                      |
| Kimi                  | 內建            | Moonshot web search                        |

```json5
{
  tools: {
    web: {
      search: {
        provider: "perplexity", // 明確指定提供者
        perplexity: {
          apiKey: "${PERPLEXITY_API_KEY}",
          searchLanguage: "zh-TW", // 搜尋語言
          searchRegion: "TW", // 搜尋地區
          searchRecency: "week", // 時間範圍：day | week | month | year
        },
      },
    },
  },
}
```

## Compaction 擴充

壓縮交接指示已擴展，保留活動任務狀態、批次進度、最新使用者請求和後續承諾，確保壓縮後保留進行中的工作上下文。

### Compaction 模型覆寫

可使用獨立模型進行壓縮：

```json5
{
  agents: {
    defaults: {
      compaction: {
        model: "anthropic/claude-sonnet-4-5"
      }
    }
  }
}
```

### Compaction Safeguard 品質控制

- **preserve/quality 設定**：可配置壓縮後保留的上下文區段和品質等級
- **摘要品質審計重試**：壓縮摘要觸發自動重試確保品質
- **結構化摘要標題要求**：強制壓縮摘要使用結構化標題
- **Post-compaction 上下文區段配置**：可自訂壓縮後的上下文區段

### Compaction 生命週期 Hooks

```
compaction:start  — 壓縮開始
compaction:end    — 壓縮完成
```

可在 `before_prompt_build` 中使用 `prependSystemContext` 和 `appendSystemContext` 注入自訂上下文。

### Compaction 穩定性改善

- 壓縮重試等待加入邊界，重啟時排空 embedded runs
- 壓縮閾值套用 contextTokens cap
- 無真實訊息時跳過壓縮 API call
- `model_context_window_exceeded` 分類為 context overflow 並觸發壓縮
- Reply pipeline 在壓縮等待前沖洗

## Billing / Rate Limit 分類

- 單提供者 billing cooldown 探測
- 402 temporary-limit 偵測擴展
- `insufficient_quota` 400 分類為 billing
- Generic cooldown text 避免 false global rate-limit 分類
- Service-unavailable 窄化至需要 overload indicator
- Auth cooldown error counters 到期重設
- HTTP 499 加入 transient error codes（模型 fallback）
- zhipuai 1310 Weekly/Monthly Limit failover
- Rate limit patterns 新增 'too many tokens' 和 'tokens per day'

## Session 日期校準

啟動/post-compaction 的 AGENTS context 中 `YYYY-MM-DD` 佔位符會替換為實際日期，`/new` 和 `/reset` 提示追加執行時當前時間行。

## Fast Mode（2026.3.12+）

session-level 快速模式切換，支援 Anthropic 和 OpenAI：

- **OpenAI**: 透過 `/fast`、TUI、Control UI、ACP 切換，per-model config defaults 和 request shaping
- **Anthropic**: 映射至 API `service_tier` 請求，即時驗證

```json5
{
  agents: {
    defaults: {
      params: {
        fastMode: true, // 啟用 fast mode
      }
    }
  }
}
```

## Sessions Yield（2026.3.12+）

新增 `sessions_yield` 工具，讓 orchestrator 可以：
- 立即結束當前 turn
- 跳過排隊的 tool work
- 攜帶隱藏的 follow-up payload 進入下一個 session turn

## Provider Plugin 架構（2026.3.12+）

Ollama、vLLM、SGLang 遷移至 provider-plugin 架構：
- Provider 擁有自己的 onboarding、discovery、model-picker setup、post-selection hooks
- 核心 provider wiring 更加模組化
- 支援非互動式 provider plugin setup

## Compaction 改善（2026.3.13+）

- Token 計數比較使用全 session pre-compaction totals
- Safeguard 壓縮摘要語言連續性保持（persona drift 減少）
- 壓縮期間顯示狀態 reaction
- Image-only tool-result 修剪支援

## Billing / Rate Limit 改善（2026.3.13+）

- Venice 402 billing error 識別
- Poe 402 'used up your points' 識別
- OpenRouter HTTP 422 分類為 format，credits 分類為 billing
- Gemini MALFORMED_RESPONSE 分類為可重試的 timeout
- 避免同一 provider 的重複 cooldown probes
- 在 context overflow 啟發式之前檢查 billing errors
- 防止 false billing error 取代有效回應文字

## 記憶改善（2026.3.13+）

- Gemini Embedding 2 Preview 支援
- Gemini embeddings 正規化
- 多模態圖片和音訊索引
- Bootstrap 載入偏好 MEMORY.md（fallback memory.md）
- Case-insensitive Docker mounts 不再注入重複記憶
- 記憶工具獨立註冊，防止耦合失敗
- LanceDB runtime 按需 bootstrap
- Pluggable system prompt section for memory plugins

## Context Engine 增強（2026.3.23+）

### Transcript 維護

新增自動 transcript 維護機制（`context engine transcript maintenance`），保持 session transcript 的健康狀態。

### assemble() 增強

`assemble()` 方法現在接收更多上下文資訊：
- **incoming prompt**：傳入 incoming prompt，改善上下文組裝品質
- **modelId**：傳入 modelId，支援 per-model 上下文策略
- **compaction delegate**：暴露 compaction delegate helper

### Compaction 改善（2026.3.23+）

- **JSONL 截斷**：compaction 後截斷 session JSONL，防止無限制增長
- **使用者通知**：compaction 開始和完成時通知使用者
- **Content-aware guard**：compaction guard 具備內容感知，防止 heartbeat sessions 的誤取消
- **Memory flush dedup**：使用 content hash 替代 compactionCount 進行記憶 flush 去重
- **Safeguard summary budget**：修復 compaction safeguard summary budget
- **Transcript 指標**：保持 session transcript 指標在 compaction 後保持最新
- **Skills compact fallback**：compaction 前透過 compact fallback 保留所有 skills

## Plugin Capability 系統（2026.3.23+）

核心能力已遷移至 Plugin Capability 系統，透過插件 manifest 中的 capability 註冊：

| 能力 | 說明 |
| ---- | ---- |
| Web Search | Provider 和 runtime 完全由插件控制 |
| Image Generation | `image_generate` 工具，Google/fal.ai providers |
| Speech / TTS | Deepgram、Groq、Microsoft providers，記憶體內語音合成 |
| Media Understanding | 圖片/音訊/影片分析由 vendor plugins 提供 |

## Image Generation（2026.3.23+）

```json5
{
  agents: {
    defaults: {
      tools: {
        imageGeneration: {
          enabled: true,
          provider: "google", // google | fal
        },
      },
    },
  },
}
```

## Fast Mode 擴展（2026.3.23+）

Fast mode 支援擴展至更多提供者：
- **xAI (Grok)**：支援 fast mode
- **MiniMax**：支援 fast mode + pi defaults 同步

## 新提供者（2026.3.23+）

- **Anthropic Vertex**：Claude via GCP Vertex AI
- **xAI**：Grok 完整整合
- **GitHub Copilot**：動態 model ID 解析
- **Xiaomi MiMo V2**：MiMo V2 Pro 和 Omni 模型
- **Mistral**：新增 curated catalog models
- **ModelStudio**：新增標準（按量付費）DashScope 端點

## Billing / Fallback 改善（2026.3.23+）

- 泛化 api_error 偵測以觸發 fallback model
- Moonshot compat 和 xAI fast remaps 集中化
- Anthropic thinking block 排序保留
