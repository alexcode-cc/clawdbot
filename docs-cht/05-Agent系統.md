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
| `video_generate` | 影片生成（Runway/xAI/Alibaba/Qwen providers，非同步任務追蹤）                                            |
| `music_generate` | 音樂生成（ComfyUI workflow 支援）                                                                         |
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
| Bedrock       | Claude (AWS)（含 Guardrails、IAM auth、Inference Profiles）                 |
| Qwen          | 通義千問 + 影片生成                                                          |
| xAI           | Grok（含 fast mode、x_search plugin-owned auth）                            |
| MiniMax       | MiniMax Portal（OAuth 認證，含 fast mode、native TTS、bundled web search）   |
| Google Gemini | Gemini 模型（CLI OAuth、managed prompt caching、cached content）；含 Gemini 3.1 Flash-Lite |
| Vercel AI     | Vercel AI Gateway catalog 自動發現                                          |
| Venice        | 預設 kimi-k2-5，發現限制和工具支援強化                                       |
| Kilocode      | 所有模型可用                                                                |
| Copilot       | Copilot Proxy（支援 token 過期自動重新整理）                                |
| Moonshot/Kimi | 原生 thinking payload 相容，failover stop reason error 視為 timeout         |
| Opencode Go   | Opencode Go provider                                                       |
| ModelStudio   | 阿里巴巴百煉（Coding Plan + 標準 DashScope 端點）                          |
| Poe           | Poe 模型（402 'used up your points' billing 識別）                          |
| Anthropic Vertex | Claude via GCP Vertex AI（含 prompt cache retention）                    |
| GitHub Copilot | 動態 model ID 解析（含 IDE auth headers）                                   |
| Xiaomi MiMo   | MiMo V2 Pro / Omni（OpenAI completions API）                               |
| Microsoft Foundry | Microsoft AI（Entra ID 認證）                                           |
| Bedrock Mantle | OpenAI-compatible Bedrock 提供者                                          |
| StepFun       | StepFun bundled provider                                                   |
| Fireworks     | Fireworks 推論提供者                                                        |

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

## xAI 深度整合（2026.3.28+）

### x_search 工具

透過 xAI Responses API 提供即時 X（Twitter）搜尋。完全由 xAI plugin 擁有，包含：
- Plugin-owned onboarding 流程
- 路由通過公開 API seam
- Fallback auth 解析
- Plugin key 重用

### code_execution 工具

xAI 原生程式碼執行環境：
- Plugin-owned codeExecution config
- 窄化的 config typing

### 架構變更

- xAI provider 預設切換至 Responses API
- `x_search` 和 `code_execution` 遷移至 plugin 架構
- 不支援的 payload 欄位和 Responses reasoning params 自動剝離
- Stale Grok transport 正規化至 Responses

## 新提供者（2026.3.28+）

- **Microsoft Foundry**：Entra ID 認證的 LLM 提供者
- **Qwen Portal OAuth 移除**：改用 ModelStudio API Key
- **Provider Model 定義遷移**：核心 model definitions 相容層移除，由 plugins 擁有

## Per-model Cooldown（2026.3.28+）

冷卻範圍改為 per-model（而非 per-provider），階梯式退避，用戶端顯示 rate-limit 訊息。

## Compaction 改善（2026.3.28+）

- 使用量過時時從 transcript 估計觸發預飛 compaction
- LLM timeout 且 context 使用量高時觸發 compaction
- 顯示 safeguard cancel 原因並釐清 /compact 跳過
- 壓制 JSON-wrapped NO_REPLY payloads

## CJK 記憶支援（2026.3.28+）

- QMD chunking 正確計算 CJK 字元（中/日/韓/越南語漢字）
- Fine-split 保護 UTF-16 surrogate pairs
- MMR `tokenize()` 支援 CJK/Kana/Hangul 字元
- Memory flush 追加修復（防止覆寫每日記憶檔案）

## Agent 改善（2026.3.31）

### LLM Idle Timeout

新增可配置的閒置串流超時，讓停滯的 model streams 乾淨地中止：

```json5
{
  agents: {
    defaults: {
      idleTimeout: 30000, // 毫秒，閒置超時
    },
  },
}
```

### Overload Failover 可配置

```json5
{
  auth: {
    cooldowns: {
      retryCount: 1,       // 同 provider 重試次數
      retryDelayMs: 0,     // 重試延遲
    },
  },
}
```

- HTTP 410、500 加入容錯分類
- Per-model cooldown 範圍和階梯式退避

### 背景任務控制平面 (ClawFlow)

任務系統從 ACP-only 簿記演進為 SQLite-backed 共享控制平面：

- **SQLite task ledger**：原子寫入確保資料一致性
- **統一執行**：ACP、subagent、cron、背景 CLI 統一管理
- **Task Flow Control**：`openclaw flows list|show|cancel`
- **Audit/Maintenance**：`openclaw tasks audit`/`maintenance`/`status`
- **Owner-key 存取邊界**：session-scoped task 更新
- **Blocked flow retry**：blocked 狀態持久化和 retry

### Pi/Codex Native Web Search

嵌入式 Pi 執行新增原生 Codex web search：
- Model-level API 的原生搜尋相關性
- Web search off 時壓制搜尋摘要

### MCP 增強

- **SSE/HTTP Transport**：遠端 MCP 伺服器 URL 配置支援
- **Tool Namespacing**：`serverName__toolName` 命名空間
- **Plugin Tools Bridge**：ACP session 的 MCP 工具橋接
- **Channel MCP Bridge**：頻道事件 MCP 橋接
- **Connection Timeout**：per-server connection timeouts
- **Schema 正規化**：修正 OpenAI Responses 的 MCP tool schema

### 記憶改善

- **QMD Extra Collections**：per-agent `memorySearch.qmd.extraCollections` 跨 agent 搜尋
- **CJK FTS5 Tokenizer**：可配置的 CJK/Kana/Hangul 全文搜尋 tokenizer
- **Collection 復原**：集合漂移重新綁定
- **維護排程**：跨 agent 嵌入維護交錯執行
- **Degraded Status**：向量搜尋降級狀態顯示
- **Session Transcripts Export**：匯出已封存 QMD session transcripts
- **QMD Sync Parity Hooks**：同步一致性 hooks

### Exec Approvals 強化

- **Command Carrier 偵測**：strict inline eval 中的指令載體偵測
- **Shell Init 攔截**：拒絕 shell 初始化檔案腳本匹配
- **Interpreter 持久化防護**：防止 interpreter allow-always 持久化
- **Wrapper 解包**：解包 `arch`、`xcrun`、`caffeinate`、`sandbox-exec`
- **safeBins 收緊**：`awk` 和 `sed` 移出低風險快速路徑
- **統一頻道 Approvals**：跨頻道統一 approval 路由和授權
- **Slack 原生 Approvals**：exec approval 提示留在 Slack

### Provider 更新

- **Azure**：停用推理負載省略
- **OpenAI**：text verbosity 轉發、disabled reasoning 省略
- **Ollama**：`think=false` 支援、串流事件發射、WSL2 網路診斷
- **Google/Gemini**：3.1 模型解析、空 `required` 陣列剔除
- **Mistral**：跨 proxy transport 相容性
- **Moonshot/Kimi**：Kimi Code 恢復、stream wrapper 解耦
- **Copilot**：auth refresh overflow 修正
- **Anthropic**：`api_error` transient 分類、service tier params

### Hooks 增強

- **before_install Hook**：新增 plugin install 時掃描器 hook
- **requireApproval**：`before_tool_call` hook 支援非同步 approval 要求
- **runId**：agent hook context 中暴露 `runId`
- **Inbound Attachments**：hooks 傳遞 inbound attachment arrays
- **Session Key Rebind**：hook-triggered session keys 重新綁定到目標 agent

### 其他改善

- **Anthropic Claude CLI Migration**：CLI 歷史匯入管線
- **Subagent Model Precedence**：修正 subagent model 優先序
- **Compaction Retry Timeout**：修正 compaction retry timeout unhandled rejection
- **Silent Turn Fail Closed**：silent turns fail closed
- **Codex server_error**：fail over 和 sanitize
- **Edit Tool edits[]**：支援 edit tool edits[] payloads
- **Pi TUI Reply Flush**：message_end 時 flush message-boundary block replies

## Agent 改善（2026.4.5）

### Dreaming 記憶促進系統

全新的多階段記憶促進系統，模擬人類睡眠中的記憶鞏固：

- **Sleep Phases**：多階段記憶處理（引入 → 處理 → 提升 → 鞏固）
- **Weighted Recall Thresholds**：加權回憶閾值決定哪些記憶值得提升
- **REM Preview**：安全預覽和重播即將提升的記憶，避免錯誤提升
- **Aging Controls**：控制記憶隨時間的衰減速率
- **DREAMS Trail**：記憶追蹤日誌從 session transcript 移至獨立 `dreams.md`
- **Dreaming UI**：Control UI 新增 Dreaming 控制面板（模式切換、階段統計、hover 說明）
- **Multilingual Promotion**：多語言記憶提升強化
- **Daily Chunking**：每日 dreaming 攝入分塊處理
- **Bedrock Embedding Provider**：新增 AWS Bedrock 嵌入提供者

### TaskFlow 任務系統（原 ClawFlow，已更名）

- **更名**：ClawFlow 正式更名為 **TaskFlow**
- **Managed Child Execution**：managed 子任務執行
- **Chat-native Task Board**：聊天原生任務面板（`/tasks` 命令）
- **Bound TaskFlow Runtime**：plugin 綁定的 TaskFlow runtime
- **Registry Observers**：hooks 更名為 observers
- **Stale Row 過濾**：status cards 中過濾過時行
- **Event Loop 保護**：防止同步 sweep 阻塞 event loop
- **Stale Task 清理**：清理過時的 cron 和 chat-backed tasks

### Prompt Cache 穩定性

Prompt cache 穩定性升級為正式的正確性/效能關鍵領域：

```json5
{
  // Prompt cache 相關配置
  agents: {
    defaults: {
      compaction: {
        // 優先壓縮最新 tool results 保留 cache prefix
        // 延遲歷史圖片修剪以保護 cache prefix
        // 3-turn 歷史圖片 cache 窗口
      },
    },
  },
}
```

- **MCP 工具排序**：確定性排序以穩定 prompt cache
- **壓縮策略**：優先壓縮最新 tool results 保留 cache prefix
- **圖片修剪延遲**：延遲歷史圖片修剪以保護 cache prefix
- **3-turn 圖片窗口**：保留完整 3-turn 歷史圖片 cache 窗口
- **Vertex AI Cache**：Anthropic Vertex AI 啟用 prompt cache retention
- **穩定 Context 排序**：project context 排在 heartbeat 之前
- **Transport 分離**：依 transport 分離 system prompt cache prefix
- **Break 診斷**：新增 prompt cache break 診斷工具

### 影片/音樂生成

- **影片生成**：`video_generate` 工具，支援 Runway、xAI、Alibaba、Qwen providers
- **音樂生成**：`music_generate` 工具，透過 ComfyUI workflow
- **非同步任務追蹤**：影片/音樂生成使用非同步任務追蹤和完成分離
- **Async Media Delivery**：非同步媒體直接投遞旗標

### Provider 架構集中化

Provider 系統經歷大規模重構：
- **Request 集中化**：capabilities、headers、attribution、media shaping 統一
- **Stream Family 系統**：stream wrapper 組合化，共享 stream family hooks
- **Provider 懶加載**：所有 provider plugins（Anthropic、OpenAI、Bedrock、Google 等）改為懶加載
- **Transport 統一**：transport policy 正規化和統一解析
- **Plugin 所有權**：provider 預設值、錯誤策略、replay runtime 全部移入 plugin

### 新提供者（2026.4.5）

| Provider | 說明 |
|----------|------|
| Bedrock Mantle | OpenAI-compatible Bedrock 提供者 |
| Bedrock Guardrails | AWS Bedrock Guardrails 支援 |
| Bedrock Inference Profiles | 推論 profile 發現和 region injection |
| Bedrock IAM Auth | IAM credential 認證 |
| StepFun | StepFun bundled provider plugin |
| Fireworks | Fireworks 推論提供者 |
| Qwen Video | Qwen 影片生成 |
| MiniMax TTS | MiniMax native TTS speech provider |
| MiniMax Web Search | MiniMax bundled web search |
| SearXNG | SearXNG bundled web search |
| Ollama Web Search | Ollama bundled web search |
| Vydra | 媒體生成提供者 |

### Failover 改進（2026.4.5）

- Rate-limit profile rotation cap 後自動升級到 model fallback
- 通用 provider 錯誤分類為 failover
- AbortError/stream-abort 分類為 timeout
- OpenRouter 403 觸發 billing fallback
- 區分 overloaded 和 rate-limit 使用者訊息

### Exec Approvals 改進（2026.4.5）

- **Windows Exec Allowlist**：完整 argPattern allowlist flow 和動態提示
- **原生聊天 Approvals**：自動啟用原生聊天 approval
- **Matrix Exec Approvals**：Matrix 頻道新增原生 exec approval + reaction 快捷鍵
- **Host Exec 預設 Yolo**：host exec 預設模式變更為 yolo
- **Policy Reporting**：統一有效策略報告和操作
- **Fail-Open 修復**：exec script preflight fail-open 繞過關閉
- **Reuse Gateway Approvals**：gateway allow-always approvals 可重用

### 新配置選項（2026.4.5）

```json5
{
  agents: {
    defaults: {
      params: {}, // 全域預設 provider params
      compaction: {
        notifyUser: false, // 壓縮是否通知使用者
      },
    },
  },
  // Cron per-job tool allowlist
  // cron: { jobs: [{ tools: ["read", "write"] }] }
  // Gateway chat history max chars
  // gateway: { webchat: { chatHistoryMaxChars: 50000 } }
}
```

### Hooks 增強（2026.4.5）

- **before_agent_reply**：reply claiming pattern hook
- **Reply dispatch hook**：plugin reply dispatch hook
- **session_end enrichment**：session_end lifecycle hooks 資料豐富化
- **Passive hooks**：mention-skipped group messages 發射 passive hooks

### 其他改善（2026.4.5）

- **Anthropic Thinking Recovery**：crash 後恢復 Anthropic thinking
- **Claude CLI Auth**：claude-cli runtime auth、login profiles 持久化、session id 持久化
- **Configurable Context Visibility**：可配置的 context visibility
- **Wildcard Peer Bindings**：`peer.id="*"` 多 agent routing
- **Default Subagent Allowlists**：預設 subagent allowlists 支援
- **Inherited Agent Skill Allowlists**：Agent skill 繼承允許清單
- **Plugin Config TUI**：Plugin 配置 TUI 整合到 onboard/configure
- **ClawHub Skill Search**：Control UI 中的 ClawHub 搜尋
- **Structured Plan Updates**：實驗性結構化計畫更新
- **Provider System Prompt Contributions**：Provider 擁有的 system prompt 貢獻
- **Heartbeat Task Batching**：透過 HEARTBEAT.md 的任務批次支援
- **GPT Personality Overlay**：OpenAI 可選 GPT 個性覆蓋
- **Lobster Managed TaskFlow**：Lobster 管理的 TaskFlow 模式
- **Lobster In-Process Workflows**：Lobster 程序內工作流

## Agent 改善（2026.4.8）

### 新功能

- **Infer CLI**：新增 `openclaw infer` 一級推理工作流指令，支援腳本化批次推理
- **Pluggable Compaction Provider**：compaction 透過 registry 支援可插拔 LLM provider
- **Compaction Checkpoints**：Gateway compaction 檢查點，斷線後可從檢查點恢復
- **Prompt Cache Context Engine**：prompt-cache runtime context 暴露給 context engines
- **Memory Prompt Helper**：context engine 新增 memory prompt helper
- **Agent Prompt Override**：runtime 動態覆蓋 agent 提示詞
- **Heartbeat Controls**：支援停用 heartbeat guidance，heartbeat 確保回到主 session

### 新提供者

- **Arcee AI**：新增 Arcee AI 提供者插件（OpenRouter 後端）
- **Google Gemma 4**：新增 Gemma 4 模型支援
- **Z.AI GLM-5.1**：Z.AI 預設模型更新至 GLM-5.1
- **Ollama Vision Auto-detect**：從 /api/show 自動偵測視覺能力

### Memory Wiki 系統

- **Belief-layer Digests**：信念層摘要，記憶片段聚合為結構化知識摘要
- **Claim Health Reports**：知識聲明健康報告
- **Compiled Digest Retrieval**：編譯摘要檢索
- **Slot-aware Dreaming**：多 agent 環境下 dreaming 配置互不干擾
- **Session Ingestion 改善**：時間戳分桶和游標式 session 攝取

### 安全修復

- **Proxy/SSRF 全面修復**：Slack proxy、SSRF guard 誤判、DNS pinning、NO_PROXY 語義
- **Env Blocklist 擴充**：Java、Rust、Cargo 工具鏈敏感變數封鎖
- **Secret Rotation Session Invalidation**：secret rotation 後失效 WS sessions
- **Node Reconnect Re-pairing**：node 重連 command 升級需重新配對

### 其他改善

- **Compaction 無限迴圈修復**：tool use 中止後 compaction 不再無限迴圈
- **Tool-result Overflow Recovery**：工具結果溢出恢復加強
- **Phase-aware History**：加強 phase-aware 歷史記錄
- **LightContext Subagents**：衍生 subagent 遵從 lightContext 設定
- **Heartbeat Transcript Race Fix**：修復 heartbeat transcript 截斷競態

## Agent 改善（2026.4.12）

### 新功能

- **LM Studio 整合**：新增 LM Studio 本地模型提供者，支援 header-auth 認證（#53248）
- **Pluggable Agent Harness**：可插拔 agent harness registry，支援 Codex app-server 等多元 runtime 後端
- **Local Exec-Policy CLI**：新增 `openclaw exec-policy` 本地指令管理 exec 執行策略（#64050）
- **Plugin Text Transforms**：插件可註冊文字轉換能力
- **Active Memory Recall Plugin**：獨立 recall plugin + QMD recall-to-search 策略（#63286）
- **/trace 切換**：Active Memory 追蹤診斷指令
- **Character Eval**：QA 角色評估系統（vibes eval、model options、並行化、報告）

### Plugin Activation Planning

- **Manifest-based Activation**：透過 manifest 描述符驅動啟動規劃
- **Narrow Loading**：CLI、Provider、Channel 三維按需載入（#65120, #65259, #65429）
- **Setup Descriptors**：manifest 新增 activation 和 setup 描述符（#64780）
- **Owner Trust Policy**：集中化 manifest owner trust 策略（#65459）

### Active Memory 與 Dreaming

- **Grounded REM**：加強 REM extraction 接地性，新增 backfill lane（#63273, #63297）
- **Workspace 隔離**：dreaming narrative sessions 按 workspace 隔離（#61674）
- **Lexical Fallback 改善**：改進無嵌入環境下的搜尋品質（#65395）
- **Memory Wiki Unicode**：slug 支援 Unicode 字元（#64742）
- **Belief-Layer 解耦**：Digest promotion 與 session-corpus pass 解耦

### Video Generation 增強

- **Seedance 2**：新增 Seedance 2 fal 影片模型
- **HeyGen Video Agent**：新增 HeyGen video-agent 模型
- **URL-only Delivery**：支援僅 URL 的影片交付（#61988）
- **進階選項**：providerOptions、inputAudios、imageRoles（#61987）

### 安全修復

- **busybox/toybox 移除**：從 interpreter-like safe bins 移除（#65713）
- **Shell Wrapper 偵測擴大**：封鎖 env-argv assignment injection（#65717）
- **Approval 空 Approver 漏洞**：修復空 approver 清單意外授權問題（#65714）
- **OAuth URL 注入**：修復 Windows cmd.exe 注入（#64161）
- **CDP Source-range**：sandbox 預設強制 CDP source-range 限制（#61404）

### 其他改善

- **Codex OAuth Scopes 保留**：保留 Codex OAuth scopes（#64713）
- **OpenAI Heartbeat Overlay**：加強 heartbeat overlay guidance（#65148）
- **Subagent Completion 去重**：去重重複的 completion 通知（#61525）
- **Gemini Tool Schema 清理**：清理 required fields（#64284）
- **llama.cpp Overflow 偵測**：偵測 context overflow（#64196）

## Agent 改善（2026.4.13–2026.4.14）

### Context Engine 重大改善

- **Per-Iteration Ingest/Assemble**：Context engine 從第一個 tool-loop delta 開始進行 compact，並在 `afterTurn` 缺失時保留 ingest fallback，使長時間執行的 tool loops 保持有界（#63555）
- **Idle-Aware 背景維護**：將 opt-in turn 維護作為 idle-aware 背景工作執行，下一個前景 turn 不再需要等待主動維護（#65233）
- **合約驗證**：拒絕 `info.id` 與已註冊 slot id 不符的 plugin engines（#63222）

### OpenAI / Codex 改善

- **GPT-5.4-Pro 前向相容**：新增 `gpt-5.4-pro` 前向相容支援，含 Codex 定價/限制和清單/狀態可見性（#66453）
- **gpt-5.4-codex 別名標準化**：將 legacy `openai-codex/gpt-5.4-codex` runtime 別名規範化為 `openai-codex/gpt-5.4`（#43060）
- **apiKey 欄位修復**：在 codex provider catalog 輸出中加入 `apiKey`，修復 Pi ModelRegistry 驗證器靜默丟棄所有 `models.json` 自訂模型的問題（#66180）
- **Auth 診斷**：保持 Codex CLI auth-file 診斷在 debug logger 而非 stdout（#66451）
- **Reasoning-Only OpenAI Turns**：修復 reasoning-only turns 的恢復問題（#66167）

### Ollama 改善

- **Streaming Usage**：為 Ollama streaming completions 發送 `stream_options.include_usage`，修復虛假 prompt-token 計數觸發提前 compaction（#64568）
- **Embedded Timeout**：將配置的 embedded-run timeout 正確傳入全域 undici stream timeout（#63175）
- **Slug Timeout**：讓 LLM-backed session-memory slug 生成遵守 `agents.defaults.timeoutSeconds` 覆蓋設定（#66237）

### GitHub Copilot

- **xhigh 推理**：允許 `github-copilot/gpt-5.4` 使用 `xhigh` 推理等級，與 GPT-5.4 系列一致（#50168）

### Google / Gemini

- **Vertex Flash-Lite IDs**：正規化 Google Vertex flash-lite 模型 ID
- **Gemini Base Suffix**：原生 Gemini image API 呼叫時剝離尾部 `/openai` suffix（#66445）

### 效能優化大波次

本版本包含 50+ 效能優化提交，核心策略為懶載入和窄化匯入：

- **Agent 啟動加速**：session helpers、delivery runtime、CLI runner seams、thinking default 等改為懶載入
- **Cron 優化**：20+ 項 cron 效能提交，全面冷路徑懶載入
- **Config 優化**：dry-run 驗證、daemon token 寫入、legacy registry 讀取等縮短冷路徑
- **Channels/Sessions/Outbound**：熱路徑訊息正規化拆分、target parsing 隔離
- **結果**：agent 啟動、setup、cron 執行、config 讀取均有明顯加速

### 安全修復（2026.4.14）

- **Gateway Config Mutation Guard**：拒絕模型端 gateway 工具呼叫啟用危險旗標（#62006）
- **Heartbeat Owner Downgrade**：強制降級不受信任 hook:wake 系統事件的 owner 等級（#66031）
- **Config Redaction**：redactConfigSnapshot 遮蔽 sourceConfig 和 runtimeConfig 別名欄位（#66030）
- **Systemd Dotenv 安全**：doctor --repair 不再嵌入 dotenv secrets 到 systemd units（#66249）
- **Browser SSRF 調整**：預設放寬 + 嚴格模式保留；快照/截圖/分頁路由強制 SSRF 政策

### 其他修復（2026.4.14）

- **Subagent Registry**：修復 subagent registry lazy-runtime stub 在 dist 輸出中的路徑問題（#66420, #66205）
- **Windows Path Join**：修復 Windows drive path join 用於 read/sandbox 工具（#54039）
- **Local Model Context Preflight**：明確本地模型低 context preflight 提示（#66236）
- **Unknown-Tool Loops**：停止重複的 unknown-tool 迴圈（#65922）
- **Approvals Timeout**：加強 approvals get timeout 處理（#66239）
- **Store Limits**：遵守配置的 store limits（#66240）

## Agent 改善（2026.4.15）

### Anthropic 預設升級至 Opus 4.7

- 所有 Anthropic 預設選擇升級至 **Claude Opus 4.7**，包括 `opus` 別名、Claude CLI 預設、捆綁影像理解
- 使用者無需手動修改配置即可受益

### Local Model Lean Mode（實驗性）

- 新增 **`agents.defaults.experimental.localModelLean: true`**
- 為較弱的本地模型設定移除 `browser`、`cron`、`message` 等重量級預設工具
- 減少 prompt 大小，不影響正常路徑
- 目前為實驗性功能（旗標下）

### Prompt Cache 穩定性改善

- **Skills 排序修復**：`available_skills` 按技能名稱字母排序，`localeCompare` 釘定 `'en'` locale
- **Chat ID 隔離**：volatile 的入站 chat IDs 從穩定系統提示移除，task-scoped adapters 可跨 runs 重用 cache
- **Context Engine 元資料對齊**：loop-hook 和 final `afterTurn` prompt-cache touch 元資料與當前 assistant turn 對齊
- **Prompt Cache Opt-out**：新增 `models.providers.*.models.*.compat.supportsPromptCacheKey` 設定

### Unknown-Tool Guard 預設啟用

- `resolveUnknownToolGuardThreshold` 現在**預設啟用**（不再需要明確設定 `tools.loopDetection.enabled: true`）
- 工具不存在時觸發，防止幻覺工具呼叫迴圈耗盡 timeout
- 可透過 `tools.loopDetection.unknownToolThreshold` 自訂閾值（預設 10）

### Skills Snapshot 失效修復

- 配置寫入觸及 `skills.*` 時更新快取的 skills-snapshot 版本
- 修復移除 bundled skill 後 model 仍繼續呼叫已停用工具的問題

### Context Window 收緊

- 裁減預設啟動/skills prompt 預算
- `memory_get` excerpts 預設 cap，包含 continuation metadata
- QMD 讀取對齊同等有界 excerpt 合約

### Failover 改善

- **HTML Error Page 修復**：正確分類 CDN 風格 5xx HTML 頁面為 upstream transport failures
- **Codex HTML Challenge**：在 DNS 分類前先偵測 Cloudflare/CDN HTML challenge 頁面
- **Model Fallback 保留**：fallback retry 時保留原始 prompt body 和 session history
- **Replay Recovery**：`401 input item ID does not belong to this connection` 分類為 replay-invalid

### CLI Transcript 持久化

- 成功的 CLI-backed turns 持久化至 OpenClaw session transcript
- google-gemini-cli 回覆再次出現在 session history 和 Control UI 中

### 其他 Agent 改善（2026.4.15）

- **Compaction Reserve Floor**：cap 至模型 context window，修復小 context 模型（16K tokens）無限 compaction
- **Delivery Mirror 去重複**：修復 Codex-backed turns 重複 visible reply
- **Codex Resume 路徑**：保持 resumed codex exec 在安全非交互路徑並恢復沙箱保護
- **Codex Harness 自動啟用**：選擇 codex 為 embedded harness 時自動啟用 Codex 插件
- **Anthropic Max Tokens 修復**：忽略非正整數 max_tokens override，防止無效值到達 API
- **Context Engine 降級**：第三方 context engine 插件失敗時優雅降級至 legacy engine
- **Exec Events 去重複**：以 session key + runId 去重複重播的 exec.finished events

## Agent 改善（2026.4.18–2026.4.20）

### 系統提示強化

- **默認系統提示**和 OpenAI GPT-5 overlay 強化：更清晰的 completion bias、live-state checks、weak-result recovery 和 verification-before-final guidance

### Session 管理改善

- **Session Maintenance 預設強制執行**（重要！）：Session 內建條目上限和年齡修剪現在預設強制執行，並在載入時修剪過大 stores，防止累積的 cron/executor session backlog 在寫入路徑執行前造成 gateway OOM
- **Auto-failover Override 清除**：`/new` 和 `/reset` 時清除 auto-sourced model、provider 和 auth-profile overrides；每次 turn 前清除 transient auto-failover session overrides，確保恢復的 primary models 立即重試
- **Session Cost Snapshot 修復**：修復 `estimatedCostUsd` 在多次 persist 路徑中被累積（最高 x 幾十倍）的問題

### Agent Bootstrap 改善

- **Bootstrap 從 workspace 真實狀態解析**：不從過期的 session transcript markers
- **Bootstrap 指令隱藏**：放在 hidden user-context prelude，不在主提示中
- **抑制啟動問候語**：`BOOTSTRAP.md` 待處理時抑制 `/new` 和 `/reset` 問候語

### Context Engine / Compaction

- **Compaction Settings 重載修復**：沒有 extension factories 的 runs 不再靜默丟失 compaction settings
- **Compaction 通知**（opt-in）：compaction 開始和完成時可選發送通知
- **Compaction 重命名**：embedded Pi compaction lifecycle events 改名至 `compaction_start/end`，對齊 pi-coding-agent 0.66.1

### Pi Runner / Failover

- **Pi Runner 靜默 Error Turn 重試**：沒有 side effects 時重試靜默 `stopReason=error` turns
- **Context Engine 第三方插件修復**：停止拒絕 `info.id` 與 registered slot id 不同的第三方 context engines（如 `lossless-claw`）

### Agent Channels / Subagents

- **子代理跨 Agent 通道繫結修復**：cross-agent subagent spawns 通過目標代理的 bound channel account 路由，不繼承 caller 的 account
- **嵌套 Lane 隔離**：嵌套 agent 工作按目標 session 隔離，不跨 gateway 阻塞無關 sessions

### Exec / YOLO 修復

- **YOLO Exec 修復**：修復 `security=full` + `ask=off` 模式下 Python/Node script preflight 阻止 heredoc 形式的問題

### Active Memory 改善

- **優雅降級**：recall 在 prompt 建置期間失敗時優雅降級，記錄 warning 並讓 reply 繼續而非失敗整個 turn
- **Timeout 提升**：阻塞 recall timeout 上限從預設提升至 120 秒

### OpenAI / Codex 改善

- **Codex /backend-api/codex 端點**：修復 `openai-codex/gpt-5.4` 在 `/backend-api/responses` 別名被移除後的 404
- **Codex OAuth Runtime-only**：外部 CLI OAuth imports 保持 runtime-only，不寫入 `auth-profiles.json`
- **Codex App-Server 預設 On-request Approval**：不再使用 over-permissive 預設
- **Codex Token Totals 修復**：cumulative app-server token totals 不再被視為 fresh context usage

### Google / Gemini 改善

- **Gemini 2.5 Pro thinkingBudget=0 修復**：修復 `thinkingBudget=0` 請求導致的 `Budget 0 is invalid` 錯誤
- **Gemini 3.1 Pro 前向相容**：解析 Gemini 3.1 Pro custom-tools 和 Flash variants

### OpenAI / Responses 改善

- **Reasoning Effort 修復**：`/think off` 時省略 disabled reasoning payloads，不再發送不支持的 `reasoning.effort: "none"`
- **GPT-5 Prompt Contract**：使用 tagged GPT-5 prompt contract，改善 GPT-5 系列的 prompt overlay

## Agent 改善（2026.4.22–2026.5.2）

### 2026.4.22（精選）

- **TUI 無 Gateway 模式**：終端內嵌聊天可不連 Gateway（插件核准仍受限）
- **`/models add`**：對話中註冊 provider 模型無需重啟
- **`sessions_list`**：label／agent／search 篩選與標題／最後訊息預覽
- **Trajectory**：預設本機捕獲 + **`/export-trajectory`** 除錯匯出
- **Runner 可見性**：`/status` 顯示 **`Runner:`**（Pi embedded、CLI、`codex (acp)` 等）
- **Thinking 預設**：具推理模型在使用者未設定時解析為安全 **medium** 等價預設
- **Cron／MCP 生命週期**：bundled MCP 退場路徑統一、transport close 時清理 stdio 樹
- **Gemini／OpenAI 等**：見 CHANGELOG Fixes（逾時 classification、websocket 路由分類器等）

### 2026.4.23（精選）

- **`sessions_spawn` forked context**（可選）：子會話可繼承請求端 transcript（預設隔離）
- **生成工具 `timeoutMs`**：影像／影片／音樂／TTS 支援每呼叫逾時
- **OpenAI／Codex replay**：Codex／OpenAI transcript replay **停止合成缺失 tool results**（其他 transport 規則不變）
- **Embedded undici**：不再降低全域 stream 逾時，避免長影像請求誤繼承短逾時
- **容量／模型失敗**：PI、Codex、auto-reply 路徑對選模型容量失敗給予 **換模型提示**
- **`--agent` + `--session-id`**：lookup **限於指定 agent store**（#70985）
- **Codex harness**：`request_user_input` 回到來源 chat；ctx-engine 組裝錯誤 log **脫敏**；結構化 harness 選擇 **debug log**
- **Dreaming**：managed dreaming cron **與 heartbeat 解耦**，以輕量隔離 agent turn 執行；doctor 可遷移舊 cron 形狀
- **Memory 本機搜尋**：**`memorySearch.local.contextSize`**（預設 4096）可調本機 embedding 上下文
- **Pi 0.70、gpt-5.5 目錄**：bundled Pi 升級；OpenAI／Codex 目錄來自 pi；**gpt-5.5-pro** 僅本機 forward-compat

### 2026.4.24（精選）

- **Heartbeat 提示隔離**：heartbeat 系統提示僅注入 **heartbeat 執行**，不混入一般回合。  
- **DeepSeek V4**：**V4 Flash／Pro**；預設與 **thinking／replay（後續 tool turn）** 行為修正。  
- **即時語音**：Talk／Voice Call／Google Meet 可將 **full agent（含工具）** 納入即時語音顧問迴圈。  
- **Subagent**：傳輸中斷後 **恢復等待**語意強化（transport drop recovery）。  
- **串流回合**：assistant chunk 僅空白時仍可 **回填最終文字**（#71454）。  
- **Codex**：對外暴露 **runtime context caps**；Responses 路徑系統提示走 **`instructions`**。  
- **Copilot**：保留加密 **reasoning item IDs** 於 replay（#71448）。  
- **Memory**：**hybrid search** 可取得 **分量分數**；**llama** runtime **可選載入**，避免強依賴拉檔錯誤（#71425）。  
- **Tool-result 修剪**：畸形 `{ type: "text" }` 區塊與估算硬化，避免繞過修剪（#34979）。

### 2026.4.25（精選）

- **Auto-reply dedupe**：replay 不安全失敗後 **標記 dedupe**，避免進度／副作用後 **重複發話**（#69303；見 changelog）。  
- **bundle-mcp allowlist**：決定 bundled MCP 可用性時 **尊重 allowlist token**。  
- **Live session fallback**：已知後續 redirect **捷徑跳轉**，少走無關候選（#57471）。  
- **OpenAI Responses**：**web search** 與 **最小 thinking** 相容修正。  
- **Subagent**：無 thread 請求者於父 announce **無可見回覆**時，以 **direct fallback** 交付子代理完成結果。  
- **Cron／exec／heartbeat wake**：以 **transient runtime context** 提交，**不當成使用者可見 prompt** 進 transcript（#66496、#66814）。  
- **Codex app-server**：native MCP **PreToolUse／PostToolUse／PermissionRequest** 經 **hook relay**；版本需求 **≥ 0.125.0**。  
- **`agents_list`／預設 slash**：優先露出 **原生 Codex app-server**，勝過模糊選 **Codex ACP**（除非明確 acpx）。  
- **`agentRuntime.id`**：canonical 設定鍵；doctor 遷移 legacy；Anthropic canonical **走 `claude-cli`** 且不將 CLI backend 別名餵給 embedded harness（#71957）。  
- **`--thinking minimal`**：對 **gpt-5.5／5.4／5.4-mini／5.2** 等現代 Codex 於請求組裝時對應為 **`low`**，避免浪費首輪（#71946）。

### 2026.4.26（精選）

- **LSP**：dispose 時 **終止 bundled stdio LSP 樹**（含 `tsserver`）（#72357）。  
- **`subagents.allowAgents`**：**顯式** `sessions_spawn(agentId=…)` **須列入 allowlist**（#72827）。  
- **`sessions_spawn(runtime="acp")`**：`acp.dispatch.enabled=false` 時仍允許 **手動** bootstrap turn（#63591）。  
- **Compaction**：可選 **`agents.defaults.compaction.maxActiveTranscriptBytes`** **預檢**，觸發正常本機 compaction 並 **rotation successor** transcript。  
- **Tool loop**：偵測視窗 **綁定 active run**；**exec volatile metadata** 不比對；**空 object tool args** → `{}`。  
- **Claude CLI stream-json**：live session **輸入／輸出格式成對**（#72206）。  
- **Session lock**：**cold bootstrap／插件／工具**完成後才 acquire **session write lock**，避免前置卡住阻塞 fallback。

### 2026.4.27（精選）

- **`/steer`**：session **閒置**時 **佇列外轉向**目前 run（#76934）；**`/side`** 為 **`/btw`** 別名。  
- **Verbose／progress**：預設 **精簡**工具摘要；**`agents.defaults.toolProgressDetail: "raw"`** 還原 raw command／detail（多通道 progress draft 通用）。  
- **Subagent announce**：direct completion fallback **保留完整分組子結果**（#77120 思路）。  
- **Message tool**：同目標多筆送出後仍可帶 **獨立結尾評論**；**string thread id** 去重路由 **避免數字精度遺失**（#76915 等）。  
- **Compaction／WebChat**：overflow retry 僅在使用者 turn **確寫入**後；**Pi 管理**助理 turn **不重複寫入**（#76424、#77033）。  
- **Bootstrap**：待處理 **`BOOTSTRAP.md`** 與截斷提示留於 **Project Context**，不混進 WebChat user context（#76946）。  
- **Presentation**：**理由文字**自 **rich presentation** **剔淨**再經 message tool 發送（防隱藏規劃外洩）。  
- **PDF／media factories**：遇 **`tools.deny`** 時 **提早跳過建立**（#76997）。

### 2026.5.2（精選）

- **Runtime registry 重用**：請求時刻重用 **啟動載入插件 registry**（providers、tools、channel actions、web／capability／memory／migration）；**memo** transcript replay-policy 與 provider extra-params（維持 transport hook／custom-env）。  
- **`agents.defaults.skipOptionalBootstrapFiles`**：略過選用 workspace bootstrap 檔而不關閉必填 workspace setup（#62110）。  
- **`heartbeat_respond`**：**結構化** heartbeat 工具輸出（#75765）。  
- **Codex**：動態工具 **native-first**；可見管道未設定時預設 **`message` tool**（#75308、#75765）。  
- **GPT-5／SSE**：預設 API-key session **SSE Responses**，除非明確選 WebSocket（修復無模型事件類問題）。  
- **Failover**：工具執行中觸發的 **run-level timeout** **不**再誤觸發模型 fallback／timeout compaction（#75873）。  
- **Compaction**：**z.ai／openrouter z-ai／GLM gateway** 連續回合保留前文（#76056）。  
- **Tool loop**：critical **circuit breaker** 以 **blocked tool result** 回傳，避免模型無限重試。  
- **Prompt／dispatch**：快取 **stable system prompt prefix**；重用 prompt-report tool schema 統計（#75999）。  
- **Sessions**：heartbeat 後保留既有 runtime model／context window（#75452）；**malformed tool args repair** 擴及 Codex／Azure OpenAI Responses（#75154）。  
- **Commitments**：heartbeat target none 時 follow-up **內部化**、due heartbeat **禁用工具**、佇列與過期策略硬化（見 changelog）。  
- **Embedded runner**：`abortable` wrapper 抽出以避免 hang 後閉包留住 transcript／subscription（#74182）。