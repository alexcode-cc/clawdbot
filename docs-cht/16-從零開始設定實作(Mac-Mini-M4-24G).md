# 從零開始設定實作指南（Mac Mini M4 24GB）

本指南針對 **Mac Mini M4 24GB** 環境，詳細說明如何從無到有設定 OpenClaw，實現以下完整工作流程：

1. **使用 Ollama 本地模型或 Claude Code 訂閱** 作為 AI 引擎
2. **使用 Discord 機器人** 作為通訊頻道
3. **自動收發 Gmail** 並觸發 AI 處理
4. **透過 Discord 機器人撰寫專案** 並自動上傳到 GitHub
5. **完成後發送 Gmail 通知** 並透過 Discord 告知

> **硬體環境**：Mac Mini M4 (Apple Silicon arm64) / 24 GB 統一記憶體 / macOS 15+

---

## 目錄

- [Mac Mini M4 24GB 硬體優勢與模型策略](#mac-mini-m4-24gb-硬體優勢與模型策略)
- [前置需求](#前置需求)
- [第一步：安裝 OpenClaw](#第一步安裝-openclaw)
- [第二步：AI 模型設定](#第二步ai-模型設定)
  - [選項 A：使用本地 Ollama（推薦本機優先）](#選項-a使用本地-ollama推薦本機優先)
  - [選項 B：使用 LM Studio](#選項-b使用-lm-studio)
  - [選項 C：使用 Claude Code 訂閱](#選項-c使用-claude-code-訂閱)
  - [混合模式：本地 + 雲端](#混合模式本地--雲端)
- [第三步：記憶體與效能深度優化](#第三步記憶體與效能深度優化)
- [第四步：Discord 機器人設定](#第四步discord-機器人設定)
- [第五步：Gmail 自動收發設定](#第五步gmail-自動收發設定)
- [第六步：GitHub 自動上傳設定](#第六步github-自動上傳設定)
- [第七步：整合工作流程設定](#第七步整合工作流程設定)
- [第八步：啟動與測試](#第八步啟動與測試)
- [第九步：進階配置](#第九步進階配置)
- [故障排除](#故障排除)

---

## Mac Mini M4 24GB 硬體優勢與模型策略

### 24GB vs 16GB：關鍵差異

24 GB 統一記憶體正好達到 OpenClaw 官方建議的「單一 24 GB GPU」門檻，相比 16GB 版本有顯著的提升：

| 項目 | 16 GB | 24 GB |
|------|-------|-------|
| 可用 GPU 記憶體 | 約 10-12 GB | 約 16-18 GB |
| 適合的模型大小 | 7B-14B（Q4/Q5） | **14B-32B**（Q4/Q5） |
| 可同時載入模型數 | 1 個中型模型 | 1 個大型或 2 個中型模型 |
| 推薦主力模型 | qwen2.5-coder:7b | **qwen2.5-coder:32b** / deepseek-r1:32b |
| 上下文窗口 | 受限（4K-8K） | 可達 **16K-32K** |
| 本地優先可行性 | 建議混合模式 | **可獨立運行本地模型** |

### 24GB 模型選擇矩陣

以下根據模型大小、量化等級和預估記憶體佔用，整理適合 24GB Mac Mini 的模型：

#### 推薦模型（可流暢運行）

| 模型 | 參數量 | 量化 | 預估記憶體 | 用途 | 推薦度 |
|------|--------|------|-----------|------|--------|
| `qwen2.5-coder:32b` | 32B | Q4_K_M | ~18 GB | 程式碼撰寫 | ★★★★★ |
| `deepseek-r1:32b` | 32B | Q4_K_M | ~18 GB | 推理與分析 | ★★★★★ |
| `llama3.3` | 8B | Q4_K_M | ~5 GB | 通用對話 | ★★★★☆ |
| `qwen2.5-coder:14b` | 14B | Q4_K_M | ~9 GB | 程式碼（輕量） | ★★★★☆ |
| `deepseek-r1:14b` | 14B | Q4_K_M | ~9 GB | 推理（輕量） | ★★★★☆ |
| `mistral:7b` | 7B | Q4_K_M | ~4.5 GB | 快速回應 | ★★★☆☆ |
| `llama3.2:3b` | 3B | Q4_K_M | ~2 GB | 極速簡單任務 | ★★★☆☆ |
| `codellama:34b` | 34B | Q4_K_M | ~19 GB | 程式碼（替代） | ★★★☆☆ |
| `gemma2:27b` | 27B | Q4_K_M | ~16 GB | 通用（Google） | ★★★☆☆ |
| `phi-4:14b` | 14B | Q4_K_M | ~8.5 GB | 推理（Microsoft） | ★★★☆☆ |

#### 可運行但需注意記憶體的模型

| 模型 | 參數量 | 量化 | 預估記憶體 | 注意事項 |
|------|--------|------|-----------|----------|
| `qwen2.5:32b` | 32B | Q4_K_M | ~18 GB | 記憶體接近上限，建議關閉其他應用 |
| `mixtral:8x7b` | 46.7B (MoE) | Q4_K_M | ~26 GB | 超出記憶體，會使用 swap，延遲較高 |
| `llama3.1:70b` | 70B | Q4_K_M | ~40 GB | 不建議，嚴重 swap |

#### 多模型同時載入方案

24GB 的優勢之一是可以同時載入多個小型模型：

| 組合方案 | 模型 A | 模型 B | 總記憶體 | 說明 |
|----------|--------|--------|----------|------|
| 編碼 + 對話 | qwen2.5-coder:14b (9 GB) | llama3.3 (5 GB) | ~14 GB | 推薦日常組合 |
| 編碼 + 推理 | qwen2.5-coder:14b (9 GB) | deepseek-r1:7b (4.5 GB) | ~13.5 GB | 開發 + 分析 |
| 快速 + 重量級 | llama3.2:3b (2 GB) | qwen2.5-coder:32b (18 GB) | ~20 GB | 簡單任務快回 + 複雜任務深度處理 |

> **效能提示**：Apple M4 的統一記憶體架構讓 CPU 和 GPU 共享同一記憶體空間，模型載入後不需要額外的資料搬移，推理延遲比同等 VRAM 的獨立 GPU 方案更低。

---

## 前置需求

### 系統需求

| 項目 | 需求 |
|------|------|
| 作業系統 | macOS 15.0+（Sequoia）或 macOS 13.0+ |
| 處理器 | Apple M4（arm64） |
| RAM | 24 GB 統一記憶體 |
| Node.js | >= 22.16.0 |
| 儲存空間 | 建議 50 GB 以上可用空間（存放模型檔案） |
| 網路 | 需要穩定的網際網路連線 |

### 必要帳號

1. **Discord 帳號** — 用於建立機器人
2. **Gmail 帳號** — 用於自動收發郵件
3. **GitHub 帳號** — 用於程式碼託管
4. **Google Cloud Platform (GCP) 帳號** — 用於 Gmail Pub/Sub 推播
5. **Tailscale 帳號** — 用於公開 HTTPS 端點（Gmail 推播需要）
6. （選用）**Anthropic / Claude 訂閱帳號** — 若使用 Claude Code 作為 AI 引擎

### 安裝必要工具

```bash
# 安裝 Homebrew（若尚未安裝）
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 安裝 Node.js 22+
brew install node@22

# 驗證 Node.js 版本
node --version  # 應為 v22.x.x 或更高

# 安裝 Git
brew install git
git --version

# 安裝 GitHub CLI
brew install gh

# 安裝 Tailscale
brew install --cask tailscale
```

---

## 第一步：安裝 OpenClaw

### 方法 A：使用安裝腳本（推薦）

```bash
curl -fsSL https://openclaw.ai/install.sh | bash

# 驗證安裝
openclaw --version
# 應顯示 2026.3.23 或更新版本
```

### 方法 B：使用 npm 全域安裝

```bash
npm install -g openclaw@latest

# 驗證安裝
openclaw --version
```

### 方法 C：從原始碼建置

```bash
# 安裝 pnpm
npm install -g pnpm@10.23.0

git clone https://github.com/openclaw/openclaw.git
cd openclaw

pnpm install
pnpm build

# 驗證
pnpm openclaw --version
```

### 初始設定

```bash
# 使用引導設定精靈（推薦首次使用）
openclaw onboard --install-daemon

# 查看配置檔路徑（macOS 預設為 ~/.openclaw/openclaw.json）
openclaw config file

# 驗證配置（啟動前）
openclaw config validate
```

> **注意**：自 2026-03-03 起，新安裝的 `tools.profile` 預設為 `messaging`（僅頻道互動工具）。若需要程式撰寫與系統操作工具，請在配置中明確設定：
>
> ```bash
> openclaw config set tools.profile '"coding"' --json
> # 或使用完整工具集
> openclaw config set tools.profile '"full"' --json
> ```

---

## 第二步：AI 模型設定

24GB 記憶體讓 Mac Mini M4 可以充分發揮本地模型的潛力。以下提供多種設定方式。

---

### 選項 A：使用本地 Ollama（推薦本機優先）

24GB 記憶體讓 Ollama 成為可靠的本地 AI 引擎，能夠運行 32B 級別的模型。

#### A.1 安裝 Ollama

```bash
# 使用 Homebrew 安裝
brew install ollama

# 或下載官方安裝程式
# https://ollama.com/download/mac
```

#### A.2 啟動 Ollama 並下載推薦模型

```bash
# 啟動 Ollama 服務（macOS 會自動在背景運行）
ollama serve

# === 推薦核心模型（24GB 最佳配置） ===

# 主力程式碼模型（32B，編碼品質接近雲端模型）
ollama pull qwen2.5-coder:32b

# 主力推理模型（32B，支援 Chain-of-Thought）
ollama pull deepseek-r1:32b

# 通用對話模型（8B，快速回應，可同時載入）
ollama pull llama3.3

# === 選用補充模型 ===

# 14B 編碼模型（可與 llama3.3 同時載入）
ollama pull qwen2.5-coder:14b

# 14B 推理模型
ollama pull deepseek-r1:14b

# 極速輕量模型（Gmail 處理等簡單任務）
ollama pull llama3.2:3b

# 27B 通用模型（Google Gemma 2）
ollama pull gemma2:27b

# 驗證已下載的模型
ollama list

# 確認 Ollama API 正常運行
curl http://127.0.0.1:11434/api/tags
```

> **24GB 記憶體模型建議**：
> - **主力編碼**：`qwen2.5-coder:32b`（32B，本地編碼品質最佳）
> - **深度推理**：`deepseek-r1:32b`（32B，推理鏈分析）
> - **快速對話**：`llama3.3`（8B，秒級回應）
> - **簡單任務**：`llama3.2:3b`（3B，Gmail 處理、摘要等）
> - **同時載入**：可同時載入 `qwen2.5-coder:14b` + `llama3.3`（共約 14 GB）

#### A.3 設定 OpenClaw 使用本地 Ollama

使用 CLI 快速設定：

```bash
# 設定 Ollama API Key（任意值即可，觸發自動探索）
openclaw config set models.providers.ollama.apiKey '"ollama-local"' --json
```

上面的命令會自動建立完整的 Ollama 提供者配置（`baseUrl`、`api`、`models`）。OpenClaw 啟動時會自動從 Ollama 探索可用模型。

或者手動編輯 `~/.openclaw/openclaw.json`，發揮 24GB 的完整潛力：

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://127.0.0.1:11434",
        apiKey: "ollama-local",
        api: "ollama",
        models: [
          // 主力編碼模型
          {
            id: "qwen2.5-coder:32b",
            name: "Qwen 2.5 Coder 32B",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 32768,
            maxTokens: 32768
          },
          // 主力推理模型
          {
            id: "deepseek-r1:32b",
            name: "DeepSeek R1 32B",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 32768,
            maxTokens: 32768
          },
          // 快速對話模型
          {
            id: "llama3.3",
            name: "Llama 3.3",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 8192,
            maxTokens: 8192
          },
          // 輕量任務模型
          {
            id: "llama3.2:3b",
            name: "Llama 3.2 3B",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 8192,
            maxTokens: 8192
          },
          // 14B 編碼備援
          {
            id: "qwen2.5-coder:14b",
            name: "Qwen 2.5 Coder 14B",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 16384,
            maxTokens: 16384
          }
        ]
      }
    }
  },
  agents: {
    defaults: {
      model: {
        primary: "ollama/qwen2.5-coder:32b",
        fallbacks: ["ollama/deepseek-r1:32b", "ollama/qwen2.5-coder:14b", "ollama/llama3.3"]
      }
    }
  }
}
```

> **注意**：
> - 本機 Ollama 的 `baseUrl` 為 `http://127.0.0.1:11434`（不含 `/v1`），`api` 設為 `"ollama"`（原生協議）。
> - 若使用 OpenAI 相容端點，則設定 `baseUrl` 為 `http://127.0.0.1:11434/v1`，`api` 設為 `"openai-completions"`。
> - 若不預先定義 `models` 陣列，OpenClaw 會在啟動時自動從 Ollama 探索可用模型。
> - 32B 模型使用 32K 上下文窗口是 24GB 記憶體的合理上限；若遇到記憶體壓力可降至 16K。

#### A.4 使用遠端 Ollama 伺服器（選用）

若有另一台高效能機器運行 Ollama（例如桌機配備獨立 GPU），可指向遠端以使用更大的模型：

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://192.168.168.168:11434",
        apiKey: "ollama-local",
        api: "ollama",
        // models 留空讓 OpenClaw 自動探索遠端模型
        models: []
      }
    }
  }
}
```

#### A.5 驗證 Ollama 設定

```bash
# 列出所有已配置的模型
openclaw models list

# 檢查模型狀態
openclaw models status
```

---

### 選項 B：使用 LM Studio

LM Studio 提供圖形化介面管理本地模型，並內建 OpenAI 相容 API 伺服器（Responses API），是 OpenClaw 官方推薦的本地模型方案。

#### B.1 安裝 LM Studio

從 [lmstudio.ai](https://lmstudio.ai) 下載 macOS 版本並安裝。

#### B.2 下載並載入模型

1. 開啟 LM Studio
2. 搜尋並下載推薦模型：
   - **MiniMax M2.5**（OpenClaw 官方推薦的本地模型，下載最大的可用版本）
   - 或其他 GGUF 格式模型
3. 載入模型並啟動本地伺服器（預設為 `http://127.0.0.1:1234`）
4. 驗證伺服器運行中：

```bash
curl http://127.0.0.1:1234/v1/models
```

> **24GB 模型選擇**：LM Studio 可載入完整的 MiniMax M2.5 GS32 模型，搭配 Responses API 將推理過程與最終回覆分離，避免在 WhatsApp 等頻道發送推理過程。

#### B.3 設定 OpenClaw 使用 LM Studio

編輯 `~/.openclaw/openclaw.json`：

```json5
{
  models: {
    mode: "merge",  // 保留雲端模型作為 fallback
    providers: {
      lmstudio: {
        baseUrl: "http://127.0.0.1:1234/v1",
        apiKey: "lmstudio",
        api: "openai-responses",
        models: [
          {
            id: "minimax-m2.5-gs32",
            name: "MiniMax M2.5 GS32",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 196608,
            maxTokens: 8192
          }
        ]
      }
    }
  },
  agents: {
    defaults: {
      model: {
        primary: "lmstudio/minimax-m2.5-gs32",
        fallbacks: ["anthropic/claude-sonnet-4-6"]
      },
      models: {
        "lmstudio/minimax-m2.5-gs32": { alias: "MiniMax Local" },
        "anthropic/claude-sonnet-4-6": { alias: "Sonnet" }
      }
    }
  }
}
```

> **提示**：根據下載的實際模型 ID 調整 `id` 和 `contextWindow`/`maxTokens` 值。確保模型保持載入狀態，冷啟動會增加延遲。

---

### 選項 C：使用 Claude Code 訂閱

Claude Code 提供強大的雲端 AI 能力，適合需要超大上下文、最高品質程式碼生成的場景。即使有 24GB 本地能力，搭配 Claude 可處理本地模型無法勝任的複雜任務。

#### C.1 安裝 Claude Code CLI

```bash
# macOS 安裝
curl -fsSL https://claude.ai/install.sh | bash
```

#### C.2 登入並產生 Setup Token

```bash
# 啟動 Claude CLI，按照提示完成瀏覽器登入
claude

# 產生 setup token
claude setup-token
# 複製產生的 token
```

#### C.3 在 OpenClaw 中設定 Claude

```bash
# 方法一：使用設定精靈
openclaw onboard --auth-choice setup-token

# 方法二：使用命令列直接貼上 token
openclaw models auth paste-token --provider anthropic
```

#### C.4 驗證 Claude 設定

```bash
# 檢查模型認證狀態
openclaw models status
```

成功後，可在配置中使用 Anthropic 模型：

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-sonnet-4-6",
        fallbacks: ["anthropic/claude-haiku-4-5", "anthropic/claude-opus-4-6"]
      }
    }
  }
}
```

---

### 混合模式：本地 + 雲端

24GB Mac Mini 可以選擇以**本地優先**的混合模式，大幅降低雲端 API 費用：

#### 本地優先 + 雲端備援（推薦）

```json5
{
  models: {
    mode: "merge",
    providers: {
      ollama: {
        baseUrl: "http://127.0.0.1:11434",
        apiKey: "ollama-local",
        api: "ollama",
        models: []  // 自動探索
      }
    }
  },
  agents: {
    defaults: {
      model: {
        // 本地 32B 主力，雲端作為備援
        primary: "ollama/qwen2.5-coder:32b",
        fallbacks: ["anthropic/claude-sonnet-4-6", "ollama/llama3.3"]
      },
      models: {
        "ollama/qwen2.5-coder:32b": { alias: "Qwen 32B" },
        "ollama/deepseek-r1:32b": { alias: "DeepSeek R1" },
        "ollama/llama3.3": { alias: "Llama Fast" },
        "anthropic/claude-sonnet-4-6": { alias: "Sonnet" }
      }
    }
  }
}
```

#### 任務分流策略

透過多 Agent 配置，針對不同任務使用最適合的模型：

```json5
{
  agents: {
    list: [
      {
        // 編碼 Agent：使用本地 32B 模型
        id: "coding",
        name: "Code Assistant",
        default: true,
        model: {
          primary: "ollama/qwen2.5-coder:32b",
          fallbacks: ["anthropic/claude-sonnet-4-6"]
        },
        workspace: "~/Projects/code"
      },
      {
        // 推理 Agent：使用 DeepSeek R1
        id: "analysis",
        name: "Analysis Agent",
        model: {
          primary: "ollama/deepseek-r1:32b",
          fallbacks: ["anthropic/claude-opus-4-6"]
        },
        workspace: "~/Projects/analysis"
      },
      {
        // 快速回應 Agent：使用輕量模型
        id: "quick",
        name: "Quick Responder",
        model: {
          primary: "ollama/llama3.3",
          fallbacks: ["ollama/llama3.2:3b"]
        },
        workspace: "~/Projects/quick"
      }
    ]
  }
}
```

---

## 第三步：記憶體與效能深度優化

24GB 記憶體需要精細管理才能同時運行 OpenClaw Gateway + Ollama + 系統服務。以下是針對 Mac Mini M4 24GB 的深度優化方案。

### 3.1 Ollama 記憶體管理

#### 控制同時載入模型數量

```bash
# 設定 Ollama 同時載入的最大模型數（預設為 1）
# 24GB 建議：1（使用 32B 模型時）或 2（使用 14B 以下模型時）
export OLLAMA_MAX_LOADED_MODELS=1

# 設定模型在記憶體中的保留時間（預設 5 分鐘）
# 延長保留以避免頻繁載入/卸載
export OLLAMA_KEEP_ALIVE="30m"

# 將設定加入 shell 設定檔
cat >> ~/.zprofile <<'EOF'
export OLLAMA_MAX_LOADED_MODELS=1
export OLLAMA_KEEP_ALIVE="30m"
EOF
source ~/.zprofile
```

#### 上下文窗口大小調整

上下文窗口直接影響記憶體佔用。OpenClaw 透過 `num_ctx` 參數傳遞 `contextWindow` 值給 Ollama。

| 模型大小 | 建議 contextWindow | 預估額外記憶體 |
|----------|-------------------|---------------|
| 32B (Q4) | 16384-32768 | +2-4 GB |
| 14B (Q4) | 16384-32768 | +1-2 GB |
| 7B-8B (Q4) | 8192-16384 | +0.5-1 GB |
| 3B (Q4) | 4096-8192 | +0.2-0.5 GB |

```json5
// 保守設定（確保穩定）
{
  models: {
    providers: {
      ollama: {
        models: [
          {
            id: "qwen2.5-coder:32b",
            contextWindow: 16384,  // 16K 較保守，記憶體充裕
            maxTokens: 16384
          }
        ]
      }
    }
  }
}

// 激進設定（發揮最大效能）
{
  models: {
    providers: {
      ollama: {
        models: [
          {
            id: "qwen2.5-coder:32b",
            contextWindow: 32768,  // 32K 上下文，需密切監控記憶體
            maxTokens: 32768
          }
        ]
      }
    }
  }
}
```

#### GPU 層級卸載（Metal 加速）

Apple Silicon 的 Metal API 支援 GPU 加速推理。Ollama 預設會自動判斷 GPU 層數，但可手動調整：

```bash
# 查看模型的層數資訊
ollama show qwen2.5-coder:32b --modelfile

# 強制所有層在 GPU 上運行（24GB 應可負擔 32B Q4 模型）
# 在 Modelfile 中設定：
# PARAMETER num_gpu 99
```

### 3.2 macOS 記憶體壓力監控

#### 即時監控工具

```bash
# 方法一：使用 Activity Monitor（活動監視器）
# 開啟 Activity Monitor → Memory 標籤頁 → 觀察「Memory Pressure」圖表
# 綠色 = 正常 | 黃色 = 需注意 | 紅色 = 記憶體不足

# 方法二：終端機即時監控
# 查看記憶體壓力（每 2 秒更新）
while true; do
  memory_pressure 2>/dev/null | head -5
  echo "---"
  sleep 2
done

# 方法三：查看 swap 使用量
sysctl vm.swapusage

# 方法四：使用 vm_stat 查看頁面統計
vm_stat | head -10
```

#### 記憶體壓力警告腳本

建立 `~/.openclaw/scripts/memory-check.sh`：

```bash
#!/bin/bash
# 檢查記憶體壓力並在高壓時警告
SWAP_USED=$(sysctl -n vm.swapusage | awk '{print $6}' | sed 's/M//')
if (( $(echo "$SWAP_USED > 2048" | bc -l) )); then
  echo "WARNING: High swap usage (${SWAP_USED}M). Consider unloading models."
  echo "Run: curl http://127.0.0.1:11434/api/generate -d '{\"model\":\"unused-model\",\"keep_alive\":0}'"
fi
```

#### 主動卸載閒置模型

```bash
# 立即卸載指定模型（釋放記憶體）
curl http://127.0.0.1:11434/api/generate -d '{"model":"deepseek-r1:32b","keep_alive":0}'

# 查看目前載入的模型
curl http://127.0.0.1:11434/api/ps
```

### 3.3 Node.js 與 OpenClaw 效能優化

```bash
# 啟用 Node.js 模組編譯快取（加速 CLI 啟動）
cat >> ~/.zprofile <<'EOF'
export NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
mkdir -p /var/tmp/openclaw-compile-cache
export OPENCLAW_NO_RESPAWN=1
EOF
source ~/.zprofile
```

### 3.4 macOS 系統層級優化

#### 減少背景記憶體佔用

```bash
# 停用 Spotlight 索引（節省約 200-500 MB 記憶體和 CPU）
# 僅在確定不需要 Spotlight 時使用
sudo mdutil -a -i off

# 若需要恢復 Spotlight
# sudo mdutil -a -i on
```

#### SSD swap 效能

Mac Mini M4 配備高速 NVMe SSD，swap 效能比傳統硬碟好得多，但仍應避免過度依賴：

```bash
# 查看 swap 大小和使用量
sysctl vm.swapusage

# 如果 swap 持續高於 4 GB，考慮：
# 1. 降低 contextWindow
# 2. 減少 OLLAMA_MAX_LOADED_MODELS
# 3. 切換到較小的模型
```

### 3.5 模型載入策略

根據使用情境選擇最佳的模型載入策略：

#### 策略一：單一大模型（最高品質）

```bash
# 適合：專注編碼或深度分析
export OLLAMA_MAX_LOADED_MODELS=1
# 使用 qwen2.5-coder:32b 或 deepseek-r1:32b
# 記憶體佔用：~18 GB 模型 + ~4 GB 系統 = ~22 GB
```

#### 策略二：雙模型並行（平衡效能）

```bash
# 適合：同時處理不同類型任務
export OLLAMA_MAX_LOADED_MODELS=2
# 使用 qwen2.5-coder:14b (9 GB) + llama3.3 (5 GB)
# 記憶體佔用：~14 GB 模型 + ~4 GB 系統 = ~18 GB
```

#### 策略三：輕量快速（最低延遲）

```bash
# 適合：Gmail 處理、簡單對話、快速回應
export OLLAMA_MAX_LOADED_MODELS=2
# 使用 llama3.3 (5 GB) + llama3.2:3b (2 GB)
# 記憶體佔用：~7 GB 模型 + ~4 GB 系統 = ~11 GB
# 剩餘記憶體可用於瀏覽器、IDE 等應用
```

### 3.6 Gmail 處理專用模型優化

Gmail 處理通常只需要簡單的摘要和分類，使用輕量模型即可節省記憶體：

```json5
{
  hooks: {
    gmail: {
      // 使用最輕量的本地模型處理 Gmail
      model: "ollama/llama3.2:3b",
      thinking: "off"
    }
  }
}
```

---

## 第四步：Discord 機器人設定

### 4.1 建立 Discord 應用程式

1. 前往 [Discord Developer Portal](https://discord.com/developers/applications)
2. 點擊 **New Application**
3. 輸入應用程式名稱（例如：`OpenClaw Assistant`）

### 4.2 建立 Bot 並取得 Token

1. 在左側選單點擊 **Bot**
2. 設定 Bot 的 **Username**（即顯示名稱）
3. 點擊 **Reset Token**（首次使用，實際上是產生第一個 token）
4. **立即複製並安全保存 Bot Token**

> **警告**：Bot Token 等同於密碼，絕對不要公開、提交到版本控制，或在聊天中傳送。

### 4.3 啟用必要的 Gateway Intents

在 Bot 頁面往下捲動到 **Privileged Gateway Intents**，啟用：

| Intent | 必要性 | 說明 |
|--------|--------|------|
| Message Content Intent | **必要** | 讀取訊息內容 |
| Server Members Intent | **建議** | 成員查詢、allowlist 比對、角色路由 |
| Presence Intent | 選用 | 僅在需要接收在線狀態更新時啟用 |

### 4.4 設定 OAuth2 權限並邀請 Bot

1. 點擊左側 **OAuth2** → 捲動到 **OAuth2 URL Generator**
2. **Scopes** 勾選：
   - `bot`
   - `applications.commands`
3. **Bot Permissions** 勾選：
   - View Channels
   - Send Messages
   - Read Message History
   - Embed Links
   - Attach Files
   - Add Reactions（選用）
4. 複製底部產生的 URL，在瀏覽器開啟
5. 選擇你的私人伺服器，點擊 **Continue** 完成邀請

> **建議**：將 Bot 加入自己的私人伺服器。如果還沒有，先[建立一個](https://support.discord.com/hc/en-us/articles/204849977-How-do-I-create-a-server)（選擇 **Create My Own → For me and my friends**）。

### 4.5 取得 Discord ID

1. 在 Discord 應用程式中，進入 **User Settings**（齒輪圖示）→ **Advanced** → 開啟 **Developer Mode**
2. 右鍵點擊你的**伺服器圖示** → **Copy Server ID**（記為 `YOUR_GUILD_ID`）
3. 右鍵點擊你的**頭像** → **Copy User ID**（記為 `YOUR_USER_ID`）

### 4.6 允許 Bot 發送私訊

右鍵點擊你的**伺服器圖示** → **Privacy Settings** → 開啟 **Direct Messages**。這讓伺服器中的 Bot 可以向你傳送私訊（pairing 配對需要此功能）。

### 4.7 設定 OpenClaw Discord 配置

**安全地設定 Bot Token**（不要在聊天中傳送）：

```bash
openclaw config set channels.discord.token '"YOUR_BOT_TOKEN"' --json
openclaw config set channels.discord.enabled true --json
```

或者使用環境變數（在 `~/.openclaw/.env` 中設定）：

```bash
echo 'DISCORD_BOT_TOKEN=your-actual-bot-token' >> ~/.openclaw/.env
```

**完整 Discord 配置**（編輯 `~/.openclaw/openclaw.json`）：

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "${DISCORD_BOT_TOKEN}",  // 引用環境變數

      // DM（私訊）策略
      dmPolicy: "allowlist",
      allowFrom: ["YOUR_USER_ID"],

      // 伺服器（Guild）策略
      groupPolicy: "allowlist",
      guilds: {
        "YOUR_GUILD_ID": {
          slug: "my-server",
          requireMention: false,  // 私人伺服器建議 false，不需 @mention
          users: ["YOUR_USER_ID"],
          channels: {
            "YOUR_CHANNEL_ID": {
              allow: true,
              requireMention: false
            }
          }
        }
      },

      // 串流預覽（推薦開啟，32B 模型回應較慢時改善體驗）
      streaming: "partial",

      // 事件佇列逾時（32B 模型回應時間較長，建議增加）
      eventQueue: {
        listenerTimeout: 180000  // 3 分鐘，比 16GB 版本更寬裕
      },

      // 動作權限
      actions: {
        reactions: true,
        messages: true,
        threads: true
      }
    }
  }
}
```

### 4.8 啟動 Gateway 並完成配對

```bash
# 啟動或重啟 Gateway
openclaw gateway

# 在 Discord 中向你的 Bot 發送私訊
# Bot 會回應一組配對碼

# 在終端機核准配對
openclaw pairing list discord
openclaw pairing approve discord <配對碼>
```

配對碼在 1 小時後過期。配對成功後，你就可以在 Discord 中與 Agent 對話了。

### 4.9 設定 Guild 工作區（建議）

配對完成後，為每個頻道建立獨立的 Agent 會話：

```bash
# 若還沒在配置中設定 guild，可以直接告訴 Agent：
# "Add my Discord Server ID <YOUR_GUILD_ID> to the guild allowlist"
# "Allow my agent to respond on this server without having to be @mentioned"
```

現在可以在 Discord 伺服器上建立不同頻道（如 `#coding`、`#research`、`#home`），每個頻道會有獨立的 Agent 會話上下文。

---

## 第五步：Gmail 自動收發設定

### 前置需求

需要安裝以下工具：

```bash
# 安裝 Google Cloud SDK
brew install --cask google-cloud-sdk

# 安裝 gogcli（Gmail CLI 工具）
# 參考 https://gogcli.sh/ 取得安裝方式

# 確認 Tailscale 已安裝並登入
tailscale status
```

### 5.1 使用 OpenClaw 設定精靈（推薦）

```bash
openclaw webhooks gmail setup \
  --account your-email@gmail.com
```

此精靈會自動完成：
- 安裝必要依賴（macOS 透過 Homebrew）
- 建立 GCP Pub/Sub topic 和 subscription
- 設定 Tailscale Funnel 作為公開 HTTPS 端點
- 寫入 `hooks.gmail` 配置
- 啟用 Gmail hook preset（`hooks.presets: ["gmail"]`）

### 5.2 手動設定 Gmail（進階）

若需要手動控制每個步驟：

#### 5.2.1 GCP 一次性設定

```bash
# 登入 Google Cloud
gcloud auth login
gcloud config set project <YOUR_PROJECT_ID>

# 啟用必要 API
gcloud services enable gmail.googleapis.com pubsub.googleapis.com

# 建立 Pub/Sub topic
gcloud pubsub topics create gog-gmail-watch

# 授權 Gmail 推播到 topic
gcloud pubsub topics add-iam-policy-binding gog-gmail-watch \
  --member=serviceAccount:gmail-api-push@system.gserviceaccount.com \
  --role=roles/pubsub.publisher
```

> **注意**：Gmail watch 要求 Pub/Sub topic 必須在 OAuth 客戶端所屬的同一個 GCP 專案中。

#### 5.2.2 啟動 Gmail 監聽

```bash
# 授權 gogcli
gog auth login --account your-email@gmail.com

# 啟動 Gmail watch
gog gmail watch start \
  --account your-email@gmail.com \
  --label INBOX \
  --topic projects/<YOUR_PROJECT_ID>/topics/gog-gmail-watch
```

記下輸出的 `history_id`（除錯時使用）。

#### 5.2.3 設定 OpenClaw Gmail Hook 配置

編輯 `~/.openclaw/openclaw.json`：

```json5
{
  hooks: {
    enabled: true,
    token: "${CLAWDBOT_HOOK_TOKEN}",  // webhook 認證 token
    path: "/hooks",
    presets: ["gmail"],

    gmail: {
      account: "your-email@gmail.com",
      label: "INBOX",
      topic: "projects/YOUR_PROJECT_ID/topics/gog-gmail-watch",
      subscription: "gog-gmail-watch-push",
      pushToken: "shared-push-token",  // Pub/Sub 推播認證 token
      hookUrl: "http://127.0.0.1:18789/hooks/gmail",
      includeBody: true,
      maxBytes: 20000,
      renewEveryMinutes: 720,  // 每 12 小時自動更新 watch

      // 本地推播伺服器設定
      serve: {
        bind: "127.0.0.1",
        port: 8788,
        path: "/"
      },

      // Tailscale Funnel 公開端點
      tailscale: {
        mode: "funnel",
        path: "/gmail-pubsub"
      },

      // Gmail 處理使用輕量本地模型（24GB 可用更好的模型）
      model: "ollama/llama3.3",
      thinking: "off"
    },

    // 訊息路由映射
    mappings: [
      {
        id: "gmail-to-discord",
        match: { path: "gmail" },
        action: "agent",
        wakeMode: "now",
        name: "Gmail Handler",
        sessionKey: "hook:gmail:{{messages[0].id}}",
        messageTemplate: "New email from {{messages[0].from}}\nSubject: {{messages[0].subject}}\n{{messages[0].snippet}}\n{{messages[0].body}}",
        deliver: true,
        channel: "discord",
        to: "user:YOUR_DISCORD_USER_ID"
      }
    ]
  }
}
```

> **重要提示**：
> - 當 `hooks.enabled=true` 且 `hooks.gmail.account` 已設定時，Gateway 啟動時會自動啟動 Gmail watcher 並自動更新 watch。
> - **不要同時**手動運行 `openclaw webhooks gmail run` 和 Gateway 自動 watcher，否則會出現 `address already in use` 錯誤。
> - 若需要手動管理 watcher，設定環境變數 `OPENCLAW_SKIP_GMAIL_WATCHER=1` 停用自動啟動。

### 5.3 測試 Gmail 收發

```bash
# 發送測試郵件
gog gmail send \
  --account your-email@gmail.com \
  --to your-email@gmail.com \
  --subject "OpenClaw 測試" \
  --body "這是一封測試郵件"

# 檢查 watch 狀態
gog gmail watch status --account your-email@gmail.com
```

收到郵件後，Agent 會透過 Discord 通知你。

---

## 第六步：GitHub 自動上傳設定

### 6.1 安裝並登入 GitHub CLI

```bash
# 安裝 GitHub CLI（若前面未安裝）
brew install gh

# 登入 GitHub
gh auth login
# 選擇 GitHub.com → HTTPS → 使用瀏覽器登入
```

### 6.2 設定 Git 全域配置

```bash
git config --global user.name "Your Name"
git config --global user.email "your-email@gmail.com"
```

### 6.3 取得 GitHub Token（選用）

若需要透過 API 操作 GitHub：

```bash
# 產生 Personal Access Token
# 前往 https://github.com/settings/tokens → Generate new token (classic)
# 勾選：repo, workflow, read:org

# 儲存到環境變數
echo 'GITHUB_TOKEN=ghp_your-github-token' >> ~/.openclaw/.env
```

### 6.4 啟用 Agent 的程式撰寫工具

確保 Agent 有足夠的工具權限來建立專案和操作 Git：

```json5
{
  agents: {
    defaults: {
      tools: {
        exec: { enabled: true },    // 執行 shell 命令（git, gh 等）
        browser: { enabled: true }  // 瀏覽器控制（選用）
      }
    }
  }
}
```

並確認 `tools.profile` 設為 `"coding"` 或 `"full"`：

```bash
openclaw config set tools.profile '"coding"' --json
```

---

## 第七步：整合工作流程設定

### 7.1 完整配置檔（24GB 優化版）

以下是針對 24GB Mac Mini 優化的完整 `~/.openclaw/openclaw.json` 配置：

```json5
{
  // === AI 模型設定（24GB 優化） ===
  models: {
    mode: "merge",  // 合併本地與雲端模型
    providers: {
      ollama: {
        baseUrl: "http://127.0.0.1:11434",
        apiKey: "ollama-local",
        api: "ollama",
        models: [
          {
            id: "qwen2.5-coder:32b",
            name: "Qwen 2.5 Coder 32B",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 32768,
            maxTokens: 32768
          },
          {
            id: "deepseek-r1:32b",
            name: "DeepSeek R1 32B",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 32768,
            maxTokens: 32768
          },
          {
            id: "llama3.3",
            name: "Llama 3.3",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 8192,
            maxTokens: 8192
          },
          {
            id: "llama3.2:3b",
            name: "Llama 3.2 3B",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 8192,
            maxTokens: 8192
          }
        ]
      }
    }
  },

  // === Agent 設定（本地優先） ===
  agents: {
    defaults: {
      model: {
        primary: "ollama/qwen2.5-coder:32b",
        fallbacks: ["ollama/deepseek-r1:32b", "anthropic/claude-sonnet-4-6", "ollama/llama3.3"]
      },
      thinking: "adaptive",
      workspace: "~/Projects",
      tools: {
        exec: { enabled: true },
        browser: { enabled: true }
      },
      models: {
        "ollama/qwen2.5-coder:32b": { alias: "Qwen 32B" },
        "ollama/deepseek-r1:32b": { alias: "DeepSeek R1" },
        "ollama/llama3.3": { alias: "Llama Fast" },
        "ollama/llama3.2:3b": { alias: "Llama Tiny" },
        "anthropic/claude-sonnet-4-6": { alias: "Sonnet" }
      }
    },
    list: [
      {
        id: "main",
        name: "OpenClaw",
        workspace: "~/Projects",
        identity: {
          name: "OpenClaw",
          soul: "你是一個專業的程式設計助手，運行在 Mac Mini M4 24GB 上。你會使用繁體中文回應。你擁有本地 32B 級別的 AI 模型，能夠高品質地撰寫程式碼。當被要求建立專案時，你會撰寫程式碼、建立 Git 儲存庫、上傳到 GitHub，並在完成後透過 Gmail 發送通知。"
        }
      }
    ]
  },

  // === 工具配置 ===
  tools: {
    profile: "coding"
  },

  // === Discord 頻道設定 ===
  channels: {
    discord: {
      enabled: true,
      token: "${DISCORD_BOT_TOKEN}",

      dmPolicy: "allowlist",
      allowFrom: ["YOUR_DISCORD_USER_ID"],

      groupPolicy: "allowlist",
      guilds: {
        "YOUR_GUILD_ID": {
          slug: "my-workspace",
          requireMention: false,
          users: ["YOUR_DISCORD_USER_ID"],
          channels: {
            "YOUR_CHANNEL_ID": {
              allow: true,
              requireMention: false
            }
          }
        }
      },

      streaming: "partial",
      eventQueue: {
        listenerTimeout: 180000  // 3 分鐘，32B 模型需要更多時間
      }
    }
  },

  // === Webhook / Gmail 設定 ===
  hooks: {
    enabled: true,
    token: "${CLAWDBOT_HOOK_TOKEN}",
    path: "/hooks",
    presets: ["gmail"],

    gmail: {
      account: "your-email@gmail.com",
      label: "INBOX",
      topic: "projects/YOUR_PROJECT_ID/topics/gog-gmail-watch",
      subscription: "gog-gmail-watch-push",
      pushToken: "shared-push-token",
      hookUrl: "http://127.0.0.1:18789/hooks/gmail",
      includeBody: true,
      maxBytes: 20000,
      renewEveryMinutes: 720,

      serve: { bind: "127.0.0.1", port: 8788, path: "/" },
      tailscale: { mode: "funnel", path: "/gmail-pubsub" },

      // 使用輕量模型處理 Gmail，不佔用 32B 模型資源
      model: "ollama/llama3.2:3b",
      thinking: "off"
    },

    mappings: [
      {
        id: "gmail-to-discord",
        match: { path: "gmail" },
        action: "agent",
        wakeMode: "now",
        name: "Gmail Handler",
        sessionKey: "hook:gmail:{{messages[0].id}}",
        messageTemplate: "New email from {{messages[0].from}}\nSubject: {{messages[0].subject}}\n{{messages[0].snippet}}\n{{messages[0].body}}",
        deliver: true,
        channel: "discord",
        to: "user:YOUR_DISCORD_USER_ID"
      }
    ]
  },

  // === Gateway 設定 ===
  gateway: {
    mode: "local",
    port: 18789,
    bind: "loopback",
    reload: {
      mode: "hybrid",
      debounceMs: 300
    }
  },

  // === 地區設定 ===
  timezone: "Asia/Taipei",
  locale: "zh-TW"
}
```

### 7.2 環境變數檔案

建立 `~/.openclaw/.env`：

```bash
# Discord Bot Token（從 Discord Developer Portal 取得）
DISCORD_BOT_TOKEN=your-discord-bot-token

# Webhook 認證 Token（自行產生的隨機字串）
CLAWDBOT_HOOK_TOKEN=your-webhook-secret-token

# GitHub Token（從 GitHub Settings 取得）
GITHUB_TOKEN=ghp_your-github-token

# Ollama 記憶體管理
OLLAMA_MAX_LOADED_MODELS=1
OLLAMA_KEEP_ALIVE=30m

# （選用）若使用 Claude Code 訂閱
# ANTHROPIC_API_KEY=sk-ant-your-key
```

產生安全的隨機 token：

```bash
# 產生 webhook token
openssl rand -hex 32
```

### 7.3 Shell 環境設定

將以下內容加入 `~/.zprofile`（或 `~/.zshrc`）：

```bash
# OpenClaw 效能優化
export NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
mkdir -p /var/tmp/openclaw-compile-cache
export OPENCLAW_NO_RESPAWN=1

# Ollama 記憶體管理
export OLLAMA_MAX_LOADED_MODELS=1
export OLLAMA_KEEP_ALIVE=30m
```

### 7.4 驗證配置

```bash
# 驗證配置檔語法與完整性
openclaw config validate

# 完整診斷
openclaw doctor
```

---

## 第八步：啟動與測試

### 8.1 啟動 Gateway

**macOS 桌面環境**（推薦）：

在 macOS 上，Gateway 以 **選單列應用程式（Menubar App）** 運行。安裝 OpenClaw Mac 應用程式後，從選單列啟動 Gateway。

**命令列啟動**（適合無 GUI 環境或測試）：

```bash
# 前景執行（方便觀察日誌）
openclaw gateway run

# 背景執行
nohup openclaw gateway run > /tmp/openclaw-gateway.log 2>&1 &

# 指定埠號與綁定
openclaw gateway run --port 18789 --bind loopback

# 強制啟動（忽略現有實例）
openclaw gateway run --force
```

### 8.2 驗證各項服務

```bash
# 完整診斷（檢查所有元件）
openclaw doctor

# 檢查頻道狀態（含探測）
openclaw channels status --probe

# 檢查模型認證
openclaw models status

# 列出可用模型
openclaw models list

# 檢查 Gateway 健康狀態
curl http://localhost:18789/healthz
curl http://localhost:18789/readyz

# 檢查 Ollama 模型載入狀態
curl http://127.0.0.1:11434/api/ps
```

### 8.3 測試 Discord 連線

在 Discord 中向 Bot 傳送私訊：

```
你好，請確認連線正常
```

若 Bot 回應，表示 Discord 頻道已正常運作。

### 8.4 測試本地模型效能

在 Discord 中測試 32B 模型的編碼能力：

```
請用 Python 寫一個 FastAPI 應用程式，包含：
1. 使用者 CRUD API
2. JWT 認證
3. SQLite 資料庫
4. Pydantic 模型驗證
```

觀察回應速度和品質。32B 模型在 24GB Mac Mini 上應能在 1-3 分鐘內完成此類任務。

### 8.5 測試 Gmail 收發

```bash
# 發送測試郵件給自己
gog gmail send \
  --account your-email@gmail.com \
  --to your-email@gmail.com \
  --subject "OpenClaw Gmail 測試" \
  --body "這是一封自動化測試郵件"
```

檢查 Discord 是否收到 Gmail 通知。

### 8.6 測試完整工作流程

在 Discord 中傳送以下訊息，測試從專案建立到 GitHub 上傳再到 Gmail 通知的完整流程：

```
請幫我建立一個簡單的 Python Hello World 專案，包含 README.md，
初始化 Git 儲存庫，上傳到 GitHub，然後發送郵件通知我完成了。
```

預期流程：
1. Agent 收到 Discord 訊息
2. 使用本地 32B 模型撰寫程式碼
3. 建立 Git 儲存庫並推送到 GitHub
4. 發送 Gmail 通知
5. 在 Discord 中回報完成狀態

---

## 第九步：進階配置

### 9.1 排程任務（Cron Jobs）

設定每日自動摘要等排程任務：

```json5
{
  cron: {
    enabled: true,
    sessionRetention: "24h",
    maxConcurrentRuns: 2,
    jobs: [
      {
        id: "daily-gmail-summary",
        schedule: "0 9 * * *",  // 每天早上 9 點
        agent: "main",
        message: "請檢查昨天的 Gmail 摘要並報告重要郵件",
        channel: "discord",
        to: "user:YOUR_DISCORD_USER_ID",
        // 使用輕量模型執行排程任務
        model: "ollama/llama3.3",
        delivery: {
          failureDestination: {
            channel: "discord",
            to: "user:YOUR_DISCORD_USER_ID"
          }
        }
      },
      {
        id: "weekly-code-review",
        schedule: "0 10 * * 1",  // 每週一早上 10 點
        agent: "main",
        message: "請檢查 ~/Projects 下所有 Git 儲存庫的最近一週變更，產生摘要報告",
        channel: "discord",
        to: "user:YOUR_DISCORD_USER_ID",
        // 程式碼審查使用 32B 模型
        model: "ollama/qwen2.5-coder:32b"
      }
    ]
  }
}
```

### 9.2 多 Agent 設定（任務分流）

根據不同任務類型使用不同的 Agent 和模型，最大化 24GB 記憶體效益：

```json5
{
  agents: {
    list: [
      {
        id: "coding",
        name: "Code Assistant",
        default: true,
        model: {
          primary: "ollama/qwen2.5-coder:32b",
          fallbacks: ["anthropic/claude-sonnet-4-6"]
        },
        workspace: "~/Projects/code",
        identity: {
          soul: "你是一個專業的程式設計助手。專注於程式碼撰寫、審查和除錯。使用繁體中文回應。"
        }
      },
      {
        id: "research",
        name: "Research Agent",
        model: {
          primary: "ollama/deepseek-r1:32b",
          fallbacks: ["anthropic/claude-opus-4-6"]
        },
        workspace: "~/Projects/research",
        identity: {
          soul: "你是一個研究分析專家。專注於資料分析、技術調研和深度推理。使用繁體中文回應。"
        }
      },
      {
        id: "docs",
        name: "Documentation Writer",
        model: {
          primary: "ollama/llama3.3",
          fallbacks: ["ollama/qwen2.5-coder:14b"]
        },
        workspace: "~/Projects/docs",
        identity: {
          soul: "你是一個技術文件撰寫專家。專注於撰寫清晰、完整的技術文件。使用繁體中文回應。"
        }
      }
    ]
  },

  // Agent 路由綁定（根據 Discord 頻道分流）
  bindings: [
    {
      agentId: "coding",
      match: {
        channel: "discord",
        guildId: "YOUR_GUILD_ID",
        channelName: "coding"
      }
    },
    {
      agentId: "research",
      match: {
        channel: "discord",
        guildId: "YOUR_GUILD_ID",
        channelName: "research"
      }
    },
    {
      agentId: "docs",
      match: {
        channel: "discord",
        guildId: "YOUR_GUILD_ID",
        channelName: "docs"
      }
    }
  ]
}
```

### 9.3 配置檔拆分（$include）

當配置變得龐大時，可以拆分成多個檔案：

```json5
// ~/.openclaw/openclaw.json
{
  gateway: { port: 18789, bind: "loopback" },
  models: { $include: "./models.json5" },
  agents: { $include: "./agents.json5" },
  channels: { $include: "./channels.json5" },
  hooks: { $include: "./hooks.json5" },
  cron: { $include: "./cron.json5" },
  timezone: "Asia/Taipei",
  locale: "zh-TW"
}
```

### 9.4 macOS 特定日誌查看

```bash
# 使用 OpenClaw 日誌工具查看 macOS 統一日誌
./scripts/clawlog.sh

# 查看 Gateway 日誌
tail -f /tmp/openclaw-gateway.log

# 跟蹤即時日誌
openclaw logs --follow
```

### 9.5 安全性考量

本地模型跳過雲端提供者的安全過濾，需要額外注意：

```json5
{
  // 限制 Agent 工具範圍
  tools: {
    profile: "coding"  // 不使用 "full" 除非必要
  },

  // Gmail hook 內容安全邊界（預設開啟）
  hooks: {
    gmail: {
      allowUnsafeExternalContent: false  // 保持 false
    }
  }
}
```

---

## 故障排除

### Discord 相關

```bash
# 檢查 Discord 連線狀態
openclaw channels status --probe

# 常見原因與解法：
# 1. Message Content Intent 未啟用 → 在 Developer Portal 開啟
# 2. Bot 權限不足 → 重新產生邀請 URL 並重新邀請
# 3. allowlist / allowFrom 設定錯誤 → 確認 User ID 為字串格式
# 4. Guild messages 被封鎖 → 確認 groupPolicy 和 guilds 設定
# 5. 長時間回應逾時 → 增加 eventQueue.listenerTimeout（建議 180000）
```

### Ollama 相關

```bash
# 確認 Ollama 服務運行中
curl http://127.0.0.1:11434/api/tags

# 查看目前載入的模型和記憶體佔用
curl http://127.0.0.1:11434/api/ps

# 常見原因與解法：
# 1. Ollama 未啟動 → 執行 ollama serve 或重啟 Ollama 應用
# 2. 模型未下載 → 執行 ollama pull <model-name>
# 3. 32B 模型載入緩慢 → 正常現象，首次載入需要 30-60 秒
# 4. 記憶體壓力過高 → 降低 contextWindow 或切換到 14B 模型
# 5. baseUrl 錯誤 → 原生 API 不含 /v1（http://127.0.0.1:11434）
# 6. apiKey 未設定 → 設定任意值（如 "ollama-local"）
# 7. Swap 使用過高 → 檢查 OLLAMA_MAX_LOADED_MODELS 設定
```

### 記憶體相關

```bash
# 檢查記憶體壓力
memory_pressure

# 檢查 swap 使用量
sysctl vm.swapusage

# 查看 Ollama 載入的模型
curl http://127.0.0.1:11434/api/ps

# 卸載閒置模型
curl http://127.0.0.1:11434/api/generate -d '{"model":"deepseek-r1:32b","keep_alive":0}'

# 常見原因與解法：
# 1. Swap > 4 GB → 降低 OLLAMA_MAX_LOADED_MODELS 或 contextWindow
# 2. Memory Pressure 紅色 → 卸載非必要模型，關閉背景應用
# 3. 模型回應極慢 → 可能在使用 swap，切換到較小模型
# 4. 系統變得遲鈍 → 卸載所有模型後重新載入需要的模型
```

### Claude Code 相關

```bash
# 重新產生 setup-token
claude setup-token

# 重新設定認證
openclaw models auth paste-token --provider anthropic

# 檢查認證狀態
openclaw models status
```

### Gmail / Webhook 相關

```bash
# 檢查 Gmail watch 狀態
gog gmail watch status --account your-email@gmail.com

# 常見原因與解法：
# 1. Pub/Sub topic 不在正確的 GCP 專案 → Invalid topicName
# 2. Gmail API 未啟用 → gcloud services enable gmail.googleapis.com
# 3. Port 8788 被佔用 → 不要同時運行手動 watcher 和 Gateway 自動 watcher
# 4. Tailscale Funnel 未設定 → tailscale funnel status
# 5. Watch 過期 → renewEveryMinutes 設定自動更新
```

### Gateway 相關

```bash
# 檢查 Gateway 是否運行
openclaw gateway status

# 強制重啟
openclaw gateway run --force

# macOS 上檢查 Gateway 進程
launchctl print gui/$UID | grep openclaw

# 完整診斷
openclaw doctor

# 查看日誌
tail -n 200 /tmp/openclaw-gateway.log
```

### 一般診斷指令

```bash
# 綜合診斷（會檢查所有元件並建議修復）
openclaw doctor

# 配置驗證
openclaw config validate

# 查看目前配置
openclaw config file

# 查看完整狀態
openclaw channels status --probe
openclaw models status
openclaw models list
```

---

## 快速參考卡

| 操作 | 命令 |
|------|------|
| 安裝 OpenClaw | `curl -fsSL https://openclaw.ai/install.sh \| bash` |
| 初始設定 | `openclaw onboard --install-daemon` |
| 啟動 Gateway | `openclaw gateway run` |
| 完整診斷 | `openclaw doctor` |
| 頻道狀態 | `openclaw channels status --probe` |
| 模型狀態 | `openclaw models status` |
| 配置驗證 | `openclaw config validate` |
| 查看日誌 | `openclaw logs --follow` |
| Discord 配對 | `openclaw pairing approve discord <CODE>` |
| Gmail 設定 | `openclaw webhooks gmail setup --account email` |
| 查看載入模型 | `curl http://127.0.0.1:11434/api/ps` |
| 卸載模型 | `curl -d '{"model":"name","keep_alive":0}' http://127.0.0.1:11434/api/generate` |
| 記憶體壓力 | `memory_pressure` |
| Swap 使用量 | `sysctl vm.swapusage` |
| 配置路徑 | `~/.openclaw/openclaw.json` |
| 環境變數 | `~/.openclaw/.env` |

---

## 24GB vs 16GB 設定差異摘要

| 項目 | 16GB 版本 | 24GB 版本 |
|------|-----------|-----------|
| 主力模型 | qwen2.5-coder:7b | **qwen2.5-coder:32b** |
| 推理模型 | deepseek-r1:7b | **deepseek-r1:32b** |
| contextWindow | 8192 | **32768** |
| eventQueue timeout | 120000 ms | **180000 ms** |
| 模型策略 | 混合模式（雲端優先） | **本地優先** |
| OLLAMA_MAX_LOADED_MODELS | 1 | 1（32B）或 2（14B） |
| Gmail 處理模型 | ollama/llama3.3 | **ollama/llama3.2:3b** |
| 多 Agent | 基本（2 個） | **進階（3+ 個，含路由綁定）** |
| 記憶體監控 | 基本 | **深度（壓力監控 + 主動卸載）** |

---

## 總結

完成以上設定後，你的 Mac Mini M4 24GB 將成為一個強大的本地 AI 開發助手平台：

1. **透過 Discord 接收開發請求** — 在 Discord 頻道或私訊中與 Agent 互動
2. **使用本地 32B 模型處理 AI 任務** — qwen2.5-coder:32b 提供接近雲端的編碼品質
3. **雲端模型作為備援** — Claude/GPT 處理本地模型無法勝任的超複雜任務
4. **自動接收 Gmail 並觸發 AI 處理** — 輕量模型高效處理郵件，不影響主力模型
5. **撰寫程式碼並自動上傳到 GitHub** — Agent 可建立專案、初始化 Git、推送到 GitHub
6. **發送 Gmail 通知完成狀態** — 完成任務後自動發送郵件通知
7. **在 Discord 中回報進度** — 全程在 Discord 追蹤任務進展
8. **精細的記憶體管理** — 充分發揮 24GB 統一記憶體的每一分效能

如有任何問題，請參考：
- [官方文件](https://docs.openclaw.ai)
- [GitHub Issues](https://github.com/openclaw/openclaw/issues)
- [Discord 社群](https://discord.gg/qkhbAGHRBT)
