# 從零開始設定實作指南（Mac Mini M4 16GB）

本指南針對 **Mac Mini M4 16GB** 環境，詳細說明如何從無到有設定 OpenClaw，實現以下完整工作流程：

1. **使用 Ollama 本地模型或 Claude Code 訂閱** 作為 AI 引擎
2. **使用 Discord 機器人** 作為通訊頻道
3. **自動收發 Gmail** 並觸發 AI 處理
4. **透過 Discord 機器人撰寫專案** 並自動上傳到 GitHub
5. **完成後發送 Gmail 通知** 並透過 Discord 告知

> **硬體環境**：Mac Mini M4 (Apple Silicon arm64) / 16 GB 統一記憶體 / macOS 15+

---

## 目錄

- [Mac Mini M4 16GB 硬體考量](#mac-mini-m4-16gb-硬體考量)
- [前置需求](#前置需求)
- [第一步：安裝 OpenClaw](#第一步安裝-openclaw)
- [第二步：AI 模型設定](#第二步ai-模型設定)
  - [選項 A：使用本地 Ollama（Mac Mini 本機）](#選項-a使用本地-ollamaMac-Mini-本機)
  - [選項 B：使用遠端 Mac Mini M4 24GB 作為 AI 引擎（推薦）](#選項-b使用遠端-mac-mini-m4-24gb-作為-ai-引擎推薦)
  - [選項 C：使用 LM Studio（本地替代方案）](#選項-c使用-lm-studio本地替代方案)
  - [選項 D：使用 Claude Code 訂閱](#選項-d使用-claude-code-訂閱)
  - [混合模式：遠端 Ollama + 本地 + 雲端（最佳方案）](#混合模式遠端-ollama--本地--雲端最佳方案)
- [第三步：Discord 機器人設定](#第三步discord-機器人設定)
- [第四步：Gmail 自動收發設定](#第四步gmail-自動收發設定)
- [第五步：GitHub 自動上傳設定](#第五步github-自動上傳設定)
- [第六步：整合工作流程設定](#第六步整合工作流程設定)
- [第七步：啟動與測試](#第七步啟動與測試)
- [第八步：進階配置](#第八步進階配置)
- [故障排除](#故障排除)

---

## Mac Mini M4 16GB 硬體考量

### 記憶體限制與模型選擇

Mac Mini M4 配備 16 GB 統一記憶體（CPU/GPU 共享），對本地 LLM 運行有以下限制：

| 項目 | 說明 |
|------|------|
| 可用 GPU 記憶體 | 約 10-12 GB（扣除系統與應用程式佔用） |
| 適合的模型大小 | 7B-14B 參數（Q4/Q5 量化） |
| 不建議的本機模型 | 32B 以上模型（會導致記憶體交換，嚴重降速） |
| 推薦策略 | **遠端 + 本地混合模式**：遠端 24GB 大模型 + 本地小模型 + 雲端備援 |

> **重要提示**：OpenClaw 官方建議本地模型需要至少 24 GB GPU 記憶體才能有良好體驗。16 GB 的 Mac Mini 只能在本機運行較小的模型（7B-14B），且延遲較高。**推薦方案**：搭配一台 Mac Mini M4 24GB 作為遠端 Ollama 伺服器，透過區域網路存取 32B 大模型（詳見[選項 B](#選項-b使用遠端-mac-mini-m4-24gb-作為-ai-引擎推薦)），同時使用 `models.mode: "merge"` 保持雲端備援。

### 效能最佳化建議

```bash
# 啟用 Node.js 模組編譯快取（加速 CLI 啟動）
cat >> ~/.zprofile <<'EOF'
export NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
mkdir -p /var/tmp/openclaw-compile-cache
export OPENCLAW_NO_RESPAWN=1
EOF
source ~/.zprofile
```

---

## 前置需求

### 系統需求

| 項目 | 需求 |
|------|------|
| 作業系統 | macOS 15.0+（Sequoia）或 macOS 13.0+ |
| 處理器 | Apple M4（arm64） |
| RAM | 16 GB 統一記憶體 |
| Node.js | >= 22.16.0 |
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

OpenClaw 支援多種 AI 模型提供者。針對 Mac Mini M4 16GB，以下提供本地與雲端的設定方式。

---

### 選項 A：使用本地 Ollama（Mac Mini 本機）

#### A.1 安裝 Ollama

```bash
# 使用 Homebrew 安裝
brew install ollama

# 或下載官方安裝程式
# https://ollama.com/download/mac
```

#### A.2 啟動 Ollama 並下載適合 16GB 的模型

```bash
# 啟動 Ollama 服務（macOS 會自動在背景運行）
ollama serve

# 下載適合 16GB 記憶體的模型（推薦）
ollama pull llama3.2:3b          # 3B 參數，輕量快速
ollama pull qwen2.5-coder:7b    # 7B 程式碼模型，編碼任務推薦
ollama pull llama3.3             # 8B 通用模型
ollama pull deepseek-r1:7b      # 7B 推理模型

# 驗證已下載的模型
ollama list

# 確認 Ollama API 正常運行
curl http://127.0.0.1:11434/api/tags
```

> **16GB 記憶體模型建議**：
> - 日常對話：`llama3.3`（8B）或 `llama3.2:3b`（3B，最快）
> - 程式碼撰寫：`qwen2.5-coder:7b`（7B，編碼品質最佳）
> - 推理任務：`deepseek-r1:7b`（7B，帶推理能力）
> - **避免使用**：32B 以上的模型（記憶體不足會導致嚴重交換延遲）

#### A.3 設定 OpenClaw 使用本地 Ollama

使用 CLI 快速設定：

```bash
# 設定 Ollama API Key（任意值即可，觸發自動探索）
openclaw config set models.providers.ollama.apiKey '"ollama-local"' --json
```

上面的命令會自動建立完整的 Ollama 提供者配置（`baseUrl`、`api`、`models`）。OpenClaw 啟動時會自動從 Ollama 探索可用模型。

或者手動編輯 `~/.openclaw/openclaw.json`：

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://127.0.0.1:11434",
        apiKey: "ollama-local",
        api: "ollama",
        models: [
          {
            id: "qwen2.5-coder:7b",
            name: "Qwen 2.5 Coder 7B",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 8192,
            maxTokens: 8192
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
            id: "deepseek-r1:7b",
            name: "DeepSeek R1 7B",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 8192,
            maxTokens: 8192
          }
        ]
      }
    }
  },
  agents: {
    defaults: {
      model: {
        primary: "ollama/qwen2.5-coder:7b",
        fallbacks: ["ollama/llama3.3", "ollama/deepseek-r1:7b"]
      }
    }
  }
}
```

> **注意**：
> - 本機 Ollama 的 `baseUrl` 為 `http://127.0.0.1:11434`（不含 `/v1`），`api` 設為 `"ollama"`（原生協議）。
> - 若使用 OpenAI 相容端點，則設定 `baseUrl` 為 `http://127.0.0.1:11434/v1`，`api` 設為 `"openai-completions"`。
> - 若不預先定義 `models` 陣列，OpenClaw 會在啟動時自動從 Ollama 探索可用模型。

#### A.4 驗證本地 Ollama 設定

```bash
# 列出所有已配置的模型
openclaw models list

# 檢查模型狀態
openclaw models status
```

---

### 選項 B：使用遠端 Mac Mini M4 24GB 作為 AI 引擎（推薦）

若你有另一台 **Mac Mini M4 24GB**（或其他高效能機器），可以將它作為專用 AI 推理伺服器，讓 16GB 的 Mac Mini 透過區域網路存取 32B 級別的大模型。這是 16GB 機器獲得高品質 AI 能力的最佳方案。

#### 架構概覽

```
┌─────────────────────┐         區域網路          ┌─────────────────────────┐
│  Mac Mini M4 16GB   │  ◄──── HTTP API ────►  │  Mac Mini M4 24GB       │
│  (OpenClaw Gateway) │      192.168.x.x:11434  │  (Ollama AI 伺服器)     │
│                     │                          │                         │
│  ● OpenClaw Gateway │                          │  ● Ollama 服務          │
│  ● Discord Bot      │                          │  ● qwen2.5-coder:32b   │
│  ● Gmail Webhook    │                          │  ● deepseek-r1:32b     │
│  ● 本地 7B 模型     │                          │  ● llama3.3            │
│    (快速簡單任務)    │                          │  ● llama3.2:3b         │
└─────────────────────┘                          └─────────────────────────┘
```

> **優勢**：
> - 16GB 機器專注運行 OpenClaw Gateway 和服務，記憶體不被大模型佔用
> - 24GB 機器專注 AI 推理，可全力運行 32B 模型
> - 兩台機器各司其職，整體效能優於單台
> - 本地仍保留 7B 小模型作為快速回應和離線備援

#### B.1 遠端伺服器設定（Mac Mini M4 24GB 端）

在 24GB 的 Mac Mini 上完成以下設定：

##### 安裝並啟動 Ollama

```bash
# 安裝 Ollama
brew install ollama

# 下載推薦的 32B 級別模型
ollama pull qwen2.5-coder:32b    # 主力編碼模型
ollama pull deepseek-r1:32b      # 主力推理模型
ollama pull llama3.3              # 快速對話模型
ollama pull llama3.2:3b           # 極速輕量模型
ollama pull qwen2.5-coder:14b    # 14B 編碼備援

# 驗證模型
ollama list
```

##### 設定 Ollama 監聽區域網路

Ollama 預設只監聽 `127.0.0.1`（本機），需要設定為監聽區域網路 IP：

```bash
# 方法一：使用環境變數（推薦）
# 將以下內容加入 ~/.zprofile（永久生效）
cat >> ~/.zprofile <<'EOF'

# Ollama 伺服器設定
export OLLAMA_HOST=0.0.0.0:11434      # 監聽所有網路介面
export OLLAMA_MAX_LOADED_MODELS=1     # 同時載入模型數（32B 建議 1）
export OLLAMA_KEEP_ALIVE=30m          # 模型保留在記憶體的時間
export OLLAMA_NUM_PARALLEL=2          # 並行推理請求數
EOF
source ~/.zprofile

# 方法二：使用 launchd plist（macOS 服務方式）
# 若 Ollama 透過 macOS 應用程式運行，需要修改 plist：
# 編輯 ~/Library/LaunchAgents/com.ollama.ollama.plist
# 加入環境變數 OLLAMA_HOST=0.0.0.0:11434

# 重啟 Ollama 服務
pkill ollama
ollama serve &
```

> **安全提示**：`OLLAMA_HOST=0.0.0.0` 會讓 Ollama 在所有網路介面上監聽。若你的區域網路不受信任，建議：
> - 改用具體 IP：`OLLAMA_HOST=192.168.x.x:11434`（只監聽區域網路 IP）
> - 使用防火牆限制存取來源
> - 或使用 Tailscale 建立安全的點對點連線

##### 確認遠端 IP 並驗證

```bash
# 查看 24GB Mac Mini 的區域網路 IP
ipconfig getifaddr en0
# 記下輸出的 IP，例如 192.168.1.100

# 在 24GB Mac Mini 本機驗證 Ollama 正在監聽
curl http://127.0.0.1:11434/api/tags

# 查看 Ollama 監聽狀態
lsof -i :11434
```

##### 效能優化（24GB 遠端伺服器）

```bash
# 加入 ~/.zprofile
cat >> ~/.zprofile <<'EOF'

# Node.js 效能優化（若同時運行其他服務）
export NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
mkdir -p /var/tmp/openclaw-compile-cache
EOF
source ~/.zprofile
```

#### B.2 本地端設定（Mac Mini M4 16GB 端）

##### 驗證網路連通性

```bash
# 將 REMOTE_IP 替換為 24GB Mac Mini 的實際 IP
REMOTE_IP=192.168.1.100

# 測試網路連通
ping -c 3 $REMOTE_IP

# 測試 Ollama API 可達
curl http://$REMOTE_IP:11434/api/tags

# 查看遠端可用的模型列表
curl http://$REMOTE_IP:11434/api/tags | python3 -m json.tool
```

##### 設定 OpenClaw 指向遠端 Ollama

使用 CLI 快速設定：

```bash
# 設定遠端 Ollama（將 192.168.1.100 替換為實際 IP）
openclaw config set models.providers.ollama-remote.baseUrl '"http://192.168.1.100:11434"' --json
openclaw config set models.providers.ollama-remote.apiKey '"ollama-remote"' --json
openclaw config set models.providers.ollama-remote.api '"ollama"' --json
```

或者手動編輯 `~/.openclaw/openclaw.json`（推薦，更精確控制）：

```json5
{
  models: {
    mode: "merge",  // 合併本地、遠端和雲端模型
    providers: {
      // 遠端 24GB Mac Mini 上的 Ollama（大模型）
      "ollama-remote": {
        baseUrl: "http://192.168.1.100:11434",  // 替換為實際 IP
        apiKey: "ollama-remote",
        api: "ollama",
        models: [
          {
            id: "qwen2.5-coder:32b",
            name: "Qwen 2.5 Coder 32B (Remote)",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 32768,
            maxTokens: 32768
          },
          {
            id: "deepseek-r1:32b",
            name: "DeepSeek R1 32B (Remote)",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 32768,
            maxTokens: 32768
          },
          {
            id: "qwen2.5-coder:14b",
            name: "Qwen 2.5 Coder 14B (Remote)",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 16384,
            maxTokens: 16384
          },
          {
            id: "llama3.3",
            name: "Llama 3.3 (Remote)",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 8192,
            maxTokens: 8192
          }
        ]
      },
      // 本機 16GB 上的 Ollama（輕量模型，快速回應 + 離線備援）
      ollama: {
        baseUrl: "http://127.0.0.1:11434",
        apiKey: "ollama-local",
        api: "ollama",
        models: [
          {
            id: "llama3.3",
            name: "Llama 3.3 (Local)",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 8192,
            maxTokens: 8192
          },
          {
            id: "llama3.2:3b",
            name: "Llama 3.2 3B (Local)",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 8192,
            maxTokens: 8192
          },
          {
            id: "qwen2.5-coder:7b",
            name: "Qwen 2.5 Coder 7B (Local)",
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
  agents: {
    defaults: {
      model: {
        // 遠端 32B 為主力，本地 7B 為備援
        primary: "ollama-remote/qwen2.5-coder:32b",
        fallbacks: [
          "ollama-remote/deepseek-r1:32b",
          "ollama-remote/llama3.3",
          "ollama/qwen2.5-coder:7b",   // 本地備援（遠端不可達時）
          "ollama/llama3.3"             // 本地備援
        ]
      }
    }
  }
}
```

> **提供者命名說明**：
> - `ollama-remote` — 遠端 24GB Mac Mini 上的 Ollama 實例
> - `ollama` — 本機 16GB Mac Mini 上的 Ollama 實例
> - 兩者可同時配置，OpenClaw 會分別連線到各自的 `baseUrl`
> - 模型引用格式為 `<provider-id>/<model-id>`（如 `ollama-remote/qwen2.5-coder:32b`）

#### B.3 使用 Tailscale 建立安全連線（選用但建議）

若兩台 Mac Mini 不在同一個可信賴的區域網路，建議使用 Tailscale 建立加密的點對點連線：

```bash
# 兩台 Mac Mini 都需要安裝並登入 Tailscale
brew install --cask tailscale
# 在兩台機器上分別登入同一個 Tailscale 帳號

# 查看 Tailscale IP
tailscale ip -4
# 例如：100.64.0.2（24GB Mac Mini 的 Tailscale IP）

# 使用 Tailscale IP 設定 Ollama（更安全）
# 在 24GB Mac Mini 上設定 Ollama 監聽 Tailscale 介面
export OLLAMA_HOST=0.0.0.0:11434

# 在 16GB Mac Mini 上使用 Tailscale IP 連線
# baseUrl: "http://100.64.0.2:11434"
```

#### B.4 驗證遠端連線

```bash
# 從 16GB Mac Mini 測試遠端 Ollama
curl http://192.168.1.100:11434/api/tags

# 測試推理是否正常
curl http://192.168.1.100:11434/api/generate -d '{
  "model": "qwen2.5-coder:32b",
  "prompt": "Hello, write a simple Python function.",
  "stream": false
}'

# 查看遠端目前載入的模型
curl http://192.168.1.100:11434/api/ps

# 驗證 OpenClaw 可以看到遠端模型
openclaw models list
openclaw models status
```

#### B.5 網路斷線備援策略

遠端伺服器可能因網路問題不可達。OpenClaw 的 fallback 機制會自動切換到本地模型：

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "ollama-remote/qwen2.5-coder:32b",  // 遠端 32B 優先
        fallbacks: [
          "ollama-remote/qwen2.5-coder:14b",          // 遠端 14B 備援
          "ollama/qwen2.5-coder:7b",                   // 本地 7B（網路斷線時）
          "ollama/llama3.3",                            // 本地 8B
          "anthropic/claude-sonnet-4-6"                 // 雲端最終備援
        ]
      }
    }
  }
}
```

確保本機 16GB Mac Mini 也安裝並運行 Ollama，下載輕量級備援模型：

```bash
# 在 16GB Mac Mini 本機安裝備援模型
brew install ollama
ollama pull llama3.3             # 8B 通用
ollama pull llama3.2:3b          # 3B 極速
ollama pull qwen2.5-coder:7b    # 7B 編碼
```

---

### 選項 C：使用 LM Studio（本地替代方案）

LM Studio 提供圖形化介面管理本地模型，並內建 OpenAI 相容 API 伺服器，是 Mac 上的本地模型方案。

#### C.1 安裝 LM Studio

從 [lmstudio.ai](https://lmstudio.ai) 下載 macOS 版本並安裝。

#### C.2 下載並載入模型

1. 開啟 LM Studio
2. 搜尋並下載適合 16GB 的模型（例如 `MiniMax M2.5` 的較小量化版本）
3. 載入模型並啟動本地伺服器（預設為 `http://127.0.0.1:1234`）
4. 驗證伺服器運行中：

```bash
curl http://127.0.0.1:1234/v1/models
```

#### C.3 設定 OpenClaw 使用 LM Studio

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
        primary: "lmstudio/minimax-m2.5-gs32"
      }
    }
  }
}
```

> **提示**：根據下載的實際模型 ID 調整 `id` 和 `contextWindow`/`maxTokens` 值。確保模型保持載入狀態，冷啟動會增加延遲。

---

### 選項 D：使用 Claude Code 訂閱

Claude Code 提供強大的雲端 AI 能力，適合需要大上下文、高品質程式碼生成的場景。

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

### 混合模式：遠端 Ollama + 本地 + 雲端（最佳方案）

針對 Mac Mini M4 16GB，**最佳方案是三層混合模式**：遠端 24GB 的大模型為主力 → 本地小模型快速回應 → 雲端模型最終備援。

#### 方案一：遠端 + 本地 + 雲端（推薦）

```json5
{
  models: {
    mode: "merge",
    providers: {
      // 遠端 24GB Mac Mini 大模型
      "ollama-remote": {
        baseUrl: "http://192.168.1.100:11434",  // 替換為實際 IP
        apiKey: "ollama-remote",
        api: "ollama",
        models: []  // 自動探索遠端模型
      },
      // 本地 16GB 輕量模型
      ollama: {
        baseUrl: "http://127.0.0.1:11434",
        apiKey: "ollama-local",
        api: "ollama",
        models: []  // 自動探索本地模型
      }
    }
  },
  agents: {
    defaults: {
      model: {
        // 遠端 32B → 本地 7B → 雲端
        primary: "ollama-remote/qwen2.5-coder:32b",
        fallbacks: [
          "ollama-remote/llama3.3",
          "ollama/qwen2.5-coder:7b",
          "anthropic/claude-sonnet-4-6"
        ]
      },
      models: {
        "ollama-remote/qwen2.5-coder:32b": { alias: "Qwen 32B Remote" },
        "ollama-remote/deepseek-r1:32b": { alias: "DeepSeek R1 Remote" },
        "ollama/qwen2.5-coder:7b": { alias: "Qwen 7B Local" },
        "ollama/llama3.3": { alias: "Llama Local" },
        "anthropic/claude-sonnet-4-6": { alias: "Sonnet" }
      }
    }
  }
}
```

#### 方案二：純本地 + 雲端（無遠端伺服器）

若沒有第二台機器，使用本地小模型 + 雲端：

```json5
{
  models: {
    mode: "merge",
    providers: {
      ollama: {
        baseUrl: "http://127.0.0.1:11434",
        apiKey: "ollama-local",
        api: "ollama",
        models: []
      }
    }
  },
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-sonnet-4-6",
        fallbacks: ["ollama/qwen2.5-coder:7b", "anthropic/claude-haiku-4-5"]
      },
      models: {
        "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
        "anthropic/claude-haiku-4-5": { alias: "Haiku" },
        "ollama/qwen2.5-coder:7b": { alias: "Qwen Local" }
      }
    }
  }
}
```

#### 方案三：任務分流（遠端不同模型處理不同任務）

透過多 Agent 讓不同任務使用最適合的模型：

```json5
{
  agents: {
    list: [
      {
        // 編碼任務：使用遠端 32B 模型
        id: "coding",
        name: "Code Assistant",
        default: true,
        model: {
          primary: "ollama-remote/qwen2.5-coder:32b",
          fallbacks: ["ollama/qwen2.5-coder:7b", "anthropic/claude-sonnet-4-6"]
        },
        workspace: "~/Projects/code"
      },
      {
        // 推理分析：使用遠端 DeepSeek R1
        id: "analysis",
        name: "Analysis Agent",
        model: {
          primary: "ollama-remote/deepseek-r1:32b",
          fallbacks: ["anthropic/claude-opus-4-6"]
        },
        workspace: "~/Projects/analysis"
      },
      {
        // 快速回應：使用本地輕量模型（不依賴網路）
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

## 第三步：Discord 機器人設定

### 3.1 建立 Discord 應用程式

1. 前往 [Discord Developer Portal](https://discord.com/developers/applications)
2. 點擊 **New Application**
3. 輸入應用程式名稱（例如：`OpenClaw Assistant`）

### 3.2 建立 Bot 並取得 Token

1. 在左側選單點擊 **Bot**
2. 設定 Bot 的 **Username**（即顯示名稱）
3. 點擊 **Reset Token**（首次使用，實際上是產生第一個 token）
4. **立即複製並安全保存 Bot Token**

> **警告**：Bot Token 等同於密碼，絕對不要公開、提交到版本控制，或在聊天中傳送。

### 3.3 啟用必要的 Gateway Intents

在 Bot 頁面往下捲動到 **Privileged Gateway Intents**，啟用：

| Intent | 必要性 | 說明 |
|--------|--------|------|
| Message Content Intent | **必要** | 讀取訊息內容 |
| Server Members Intent | **建議** | 成員查詢、allowlist 比對、角色路由 |
| Presence Intent | 選用 | 僅在需要接收在線狀態更新時啟用 |

### 3.4 設定 OAuth2 權限並邀請 Bot

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

### 3.5 取得 Discord ID

1. 在 Discord 應用程式中，進入 **User Settings**（齒輪圖示）→ **Advanced** → 開啟 **Developer Mode**
2. 右鍵點擊你的**伺服器圖示** → **Copy Server ID**（記為 `YOUR_GUILD_ID`）
3. 右鍵點擊你的**頭像** → **Copy User ID**（記為 `YOUR_USER_ID`）

### 3.6 允許 Bot 發送私訊

右鍵點擊你的**伺服器圖示** → **Privacy Settings** → 開啟 **Direct Messages**。這讓伺服器中的 Bot 可以向你傳送私訊（pairing 配對需要此功能）。

### 3.7 設定 OpenClaw Discord 配置

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

      // 串流預覽（選用）
      streaming: "partial",

      // 事件佇列逾時（對長時間 LLM 回應有用）
      eventQueue: {
        listenerTimeout: 120000
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

### 3.8 啟動 Gateway 並完成配對

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

### 3.9 設定 Guild 工作區（建議）

配對完成後，為每個頻道建立獨立的 Agent 會話：

```bash
# 若還沒在配置中設定 guild，可以直接告訴 Agent：
# "Add my Discord Server ID <YOUR_GUILD_ID> to the guild allowlist"
# "Allow my agent to respond on this server without having to be @mentioned"
```

現在可以在 Discord 伺服器上建立不同頻道（如 `#coding`、`#research`、`#home`），每個頻道會有獨立的 Agent 會話上下文。

---

## 第四步：Gmail 自動收發設定

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

### 4.1 使用 OpenClaw 設定精靈（推薦）

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

### 4.2 手動設定 Gmail（進階）

若需要手動控制每個步驟：

#### 4.2.1 GCP 一次性設定

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

#### 4.2.2 啟動 Gmail 監聽

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

#### 4.2.3 設定 OpenClaw Gmail Hook 配置

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

      // Gmail 處理使用的模型（選用，可用較便宜的模型）
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

### 4.3 測試 Gmail 收發

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

## 第五步：GitHub 自動上傳設定

### 5.1 安裝並登入 GitHub CLI

```bash
# 安裝 GitHub CLI（若前面未安裝）
brew install gh

# 登入 GitHub
gh auth login
# 選擇 GitHub.com → HTTPS → 使用瀏覽器登入
```

### 5.2 設定 Git 全域配置

```bash
git config --global user.name "Your Name"
git config --global user.email "your-email@gmail.com"
```

### 5.3 取得 GitHub Token（選用）

若需要透過 API 操作 GitHub：

```bash
# 產生 Personal Access Token
# 前往 https://github.com/settings/tokens → Generate new token (classic)
# 勾選：repo, workflow, read:org

# 儲存到環境變數
echo 'GITHUB_TOKEN=ghp_your-github-token' >> ~/.openclaw/.env
```

### 5.4 啟用 Agent 的程式撰寫工具

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

## 第六步：整合工作流程設定

### 6.1 完整配置檔

以下是整合所有功能的完整 `~/.openclaw/openclaw.json` 配置：

```json5
{
  // === AI 模型設定（遠端 + 本地 + 雲端三層架構） ===
  models: {
    mode: "merge",  // 合併遠端、本地與雲端模型
    providers: {
      // 遠端 Mac Mini M4 24GB 上的 Ollama（大模型主力）
      "ollama-remote": {
        baseUrl: "http://192.168.1.100:11434",  // 替換為 24GB Mac Mini 實際 IP
        apiKey: "ollama-remote",
        api: "ollama",
        models: []  // 自動探索遠端模型
      },
      // 本機 16GB 上的 Ollama（輕量模型 + 離線備援）
      ollama: {
        baseUrl: "http://127.0.0.1:11434",
        apiKey: "ollama-local",
        api: "ollama",
        models: []  // 自動探索本機模型
      }
    }
  },

  // === Agent 設定（遠端優先） ===
  agents: {
    defaults: {
      model: {
        primary: "ollama-remote/qwen2.5-coder:32b",    // 遠端 32B 主力
        fallbacks: [
          "ollama-remote/llama3.3",                      // 遠端快速回應
          "ollama/qwen2.5-coder:7b",                     // 本地備援
          "anthropic/claude-sonnet-4-6"                   // 雲端最終備援
        ]
      },
      models: {
        "ollama-remote/qwen2.5-coder:32b": { alias: "Qwen 32B Remote" },
        "ollama-remote/deepseek-r1:32b": { alias: "DeepSeek R1 Remote" },
        "ollama/qwen2.5-coder:7b": { alias: "Qwen 7B Local" },
        "ollama/llama3.3": { alias: "Llama Local" },
        "anthropic/claude-sonnet-4-6": { alias: "Sonnet" }
      },
      thinking: "adaptive",
      workspace: "~/Projects",
      tools: {
        exec: { enabled: true },
        browser: { enabled: true }
      }
    },
    list: [
      {
        id: "main",
        name: "OpenClaw",
        workspace: "~/Projects",
        identity: {
          name: "OpenClaw",
          soul: "你是一個專業的程式設計助手，運行在 Mac Mini M4 上，透過遠端 24GB Mac Mini 存取 32B 大模型。你會使用繁體中文回應。當被要求建立專案時，你會撰寫程式碼、建立 Git 儲存庫、上傳到 GitHub，並在完成後透過 Gmail 發送通知。"
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
        listenerTimeout: 120000
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

      // Gmail 處理使用本地輕量模型（不佔用遠端 32B 資源）
      model: "ollama/llama3.3",
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

### 6.2 環境變數檔案

建立 `~/.openclaw/.env`：

```bash
# Discord Bot Token（從 Discord Developer Portal 取得）
DISCORD_BOT_TOKEN=your-discord-bot-token

# Webhook 認證 Token（自行產生的隨機字串）
CLAWDBOT_HOOK_TOKEN=your-webhook-secret-token

# GitHub Token（從 GitHub Settings 取得）
GITHUB_TOKEN=ghp_your-github-token

# （選用）若使用 Claude Code 訂閱
# ANTHROPIC_API_KEY=sk-ant-your-key
```

產生安全的隨機 token：

```bash
# 產生 webhook token
openssl rand -hex 32
```

### 6.3 驗證配置

```bash
# 驗證配置檔語法與完整性
openclaw config validate

# 完整診斷
openclaw doctor
```

---

## 第七步：啟動與測試

### 7.1 啟動 Gateway

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

### 7.2 驗證各項服務

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
```

### 7.3 測試 Discord 連線

在 Discord 中向 Bot 傳送私訊：

```
你好，請確認連線正常
```

若 Bot 回應，表示 Discord 頻道已正常運作。

### 7.4 測試 Gmail 收發

```bash
# 發送測試郵件給自己
gog gmail send \
  --account your-email@gmail.com \
  --to your-email@gmail.com \
  --subject "OpenClaw Gmail 測試" \
  --body "這是一封自動化測試郵件"
```

檢查 Discord 是否收到 Gmail 通知。

### 7.5 測試完整工作流程

在 Discord 中傳送以下訊息，測試從專案建立到 GitHub 上傳再到 Gmail 通知的完整流程：

```
請幫我建立一個簡單的 Python Hello World 專案，包含 README.md，
初始化 Git 儲存庫，上傳到 GitHub，然後發送郵件通知我完成了。
```

預期流程：
1. Agent 收到 Discord 訊息
2. 使用 AI 模型撰寫程式碼
3. 建立 Git 儲存庫並推送到 GitHub
4. 發送 Gmail 通知
5. 在 Discord 中回報完成狀態

---

## 第八步：進階配置

### 8.1 排程任務（Cron Jobs）

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
        delivery: {
          failureDestination: {
            channel: "discord",
            to: "user:YOUR_DISCORD_USER_ID"
          }
        }
      }
    ]
  }
}
```

### 8.2 多 Agent 設定

根據不同任務類型使用不同的 Agent 和模型：

```json5
{
  agents: {
    list: [
      {
        id: "coding",
        name: "Code Assistant",
        model: {
          primary: "anthropic/claude-sonnet-4-6"
        },
        workspace: "~/Projects/code",
        identity: {
          soul: "你是一個專業的程式設計助手。專注於程式碼撰寫、審查和除錯。"
        }
      },
      {
        id: "docs",
        name: "Documentation Writer",
        model: {
          primary: "ollama/llama3.3"  // 文件任務用本地模型即可
        },
        workspace: "~/Projects/docs",
        identity: {
          soul: "你是一個技術文件撰寫專家。專注於撰寫清晰、完整的技術文件。"
        }
      }
    ]
  }
}
```

### 8.3 配置檔拆分（$include）

當配置變得龐大時，可以拆分成多個檔案：

```json5
// ~/.openclaw/openclaw.json
{
  gateway: { port: 18789, bind: "loopback" },
  agents: { $include: "./agents.json5" },
  channels: { $include: "./channels.json5" },
  hooks: { $include: "./hooks.json5" },
  timezone: "Asia/Taipei",
  locale: "zh-TW"
}
```

### 8.4 macOS 特定日誌查看

```bash
# 使用 OpenClaw 日誌工具查看 macOS 統一日誌
./scripts/clawlog.sh

# 查看 Gateway 日誌
tail -f /tmp/openclaw-gateway.log

# 跟蹤即時日誌
openclaw logs --follow
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
# 5. 長時間回應逾時 → 增加 eventQueue.listenerTimeout
```

### Ollama 相關（本地）

```bash
# 確認本地 Ollama 服務運行中
curl http://127.0.0.1:11434/api/tags

# 常見原因與解法：
# 1. Ollama 未啟動 → 執行 ollama serve 或重啟 Ollama 應用
# 2. 模型未下載 → 執行 ollama pull <model-name>
# 3. 記憶體不足 → 切換到較小的模型（3B-7B）
# 4. baseUrl 錯誤 → 原生 API 不含 /v1（http://127.0.0.1:11434）
# 5. apiKey 未設定 → 設定任意值（如 "ollama-local"）
```

### 遠端 Ollama 相關（Mac Mini M4 24GB）

```bash
# 確認遠端 Ollama 服務可達（替換為實際 IP）
curl http://192.168.1.100:11434/api/tags

# 查看遠端目前載入的模型
curl http://192.168.1.100:11434/api/ps

# 常見原因與解法：
# 1. 連線被拒（Connection refused）
#    → 確認遠端 Ollama 設定了 OLLAMA_HOST=0.0.0.0:11434
#    → 確認遠端 Ollama 服務正在運行：ssh user@192.168.1.100 'curl 127.0.0.1:11434/api/tags'
#    → 重啟遠端 Ollama：ssh user@192.168.1.100 'pkill ollama && ollama serve &'
#
# 2. 網路不通（No route to host / Connection timed out）
#    → 確認兩台機器在同一區域網路：ping 192.168.1.100
#    → 確認遠端 IP 正確：ssh user@192.168.1.100 'ipconfig getifaddr en0'
#    → 檢查防火牆：確認 macOS 防火牆允許 port 11434
#
# 3. 遠端模型回應極慢
#    → 查看遠端記憶體壓力：ssh user@192.168.1.100 'memory_pressure'
#    → 降低 contextWindow 或使用較小模型
#    → 確認 OLLAMA_MAX_LOADED_MODELS=1（32B 模型時）
#
# 4. 自動探索找不到遠端模型
#    → 確認 provider 的 api 設為 "ollama"（不是 "openai-completions"）
#    → 手動指定 models 陣列，不依賴自動探索
#
# 5. Fallback 未生效（遠端斷線後未切換到本地）
#    → 確認 fallbacks 陣列包含本地模型（ollama/... 開頭的）
#    → 確認本地 Ollama 也在運行
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
| 測試遠端 Ollama | `curl http://REMOTE_IP:11434/api/tags` |
| 查看遠端載入模型 | `curl http://REMOTE_IP:11434/api/ps` |
| 配置路徑 | `~/.openclaw/openclaw.json` |
| 環境變數 | `~/.openclaw/.env` |

---

## 總結

完成以上設定後，你的 Mac Mini M4 16GB 將成為一個功能完整的 AI 開發助手平台：

1. **透過 Discord 接收開發請求** — 在 Discord 頻道或私訊中與 Agent 互動
2. **透過遠端 Mac Mini M4 24GB 存取 32B 大模型** — 本地 16GB + 遠端 24GB 雙機協作
3. **本地輕量模型作為備援** — 網路斷線時自動切換到本地 7B 模型
4. **使用 Claude Code 作為最終備援** — 雲端模型處理超複雜任務
5. **自動接收 Gmail 並觸發 AI 處理** — 新郵件自動通知到 Discord
6. **撰寫程式碼並自動上傳到 GitHub** — Agent 可建立專案、初始化 Git、推送到 GitHub
7. **發送 Gmail 通知完成狀態** — 完成任務後自動發送郵件通知
8. **在 Discord 中回報進度** — 全程在 Discord 追蹤任務進展

如有任何問題，請參考：
- [官方文件](https://docs.openclaw.ai)
- [GitHub Issues](https://github.com/openclaw/openclaw/issues)
- [Discord 社群](https://discord.gg/qkhbAGHRBT)
