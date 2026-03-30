# Gateway 伺服器

## 概述

Gateway 是 OpenClaw 的核心伺服器元件，提供 WebSocket 即時通訊、RPC 方法處理、會話管理等功能。所有客戶端（Web UI、行動應用、CLI）都透過 Gateway 與系統互動。

## 架構

```
                    ┌────────────────────────────────────┐
                    │          Gateway Server            │
                    │                                    │
┌─────────┐         │  ┌──────────────────────────────┐  │
│ Web UI  │─────────┼▶│     WebSocket Handler        │  │
└─────────┘         │  │  ┌────────────────────────┐  │  │
                    │  │  │ Connection Manager     │  │  │
┌─────────┐         │  │  │ - authenticate         │  │  │
│  iOS    │─────────┼▶│  │ - route messages       │  │  │
│  App    │         │  │  │ - broadcast events     │  │  │
└─────────┘         │  │  └────────────────────────┘  │  │
                    │  └──────────────────────────────┘  │
┌─────────┐         │                                    │
│ Android │─────────┼▶┌──────────────────────────────┐  │
│  App    │         │  │      RPC Methods             │  │
└─────────┘         │  │  - chat.*                    │  │
                    │  │  - config.*                  │  │
┌─────────┐         │  │  - channels.*                │  │
│ macOS   │─────────┼▶│  - node.*                    │  │
│  App    │         │  │  - system.*                  │  │
└─────────┘         │  └──────────────────────────────┘  │
                    │                                    │
                    │  ┌──────────────────────────────┐  │
                    │  │     Runtime State            │  │
                    │  │  - chatRunState              │  │
                    │  │  - chatRunBuffers            │  │
                    │  │  - wsConnections             │  │
                    │  └──────────────────────────────┘  │
                    └─────────────────────────────────────┘
```

## 核心檔案

| 檔案                                  | 說明                 |
| ------------------------------------- | -------------------- |
| `src/gateway/server.impl.ts`          | Gateway 伺服器主實作 |
| `src/gateway/server-runtime-state.ts` | 運行時狀態管理       |
| `src/gateway/server-methods.ts`       | RPC 方法處理器       |
| `src/gateway/server-methods/*.ts`     | 各類方法實作         |
| `src/gateway/server/ws-connection/`   | WebSocket 連接處理   |
| `src/gateway/protocol/`               | 協議定義（TypeBox）  |

## 啟動流程

```typescript
// src/gateway/server.impl.ts
export async function startGatewayServer(options: GatewayServerOptions) {
  // 1. 建立 HTTP 伺服器
  const app = express();
  const server = http.createServer(app);

  // 2. 建立 WebSocket 伺服器
  const wss = new WebSocketServer({ server });

  // 3. 初始化運行時狀態
  const state = createGatewayRuntimeState();

  // 4. 註冊 RPC 方法
  registerGatewayMethods(state, options);

  // 5. 設定 WebSocket 處理
  wss.on("connection", (ws, req) => {
    handleConnection(ws, req, state);
  });

  // 6. 啟動伺服器
  server.listen(options.port, options.bind);

  return { server, wss, state };
}
```

## WebSocket 協議

### 訊息格式

```typescript
// 請求
interface RpcRequest {
  id: string; // 請求 ID
  method: string; // 方法名稱
  params?: unknown; // 參數
}

// 回應
interface RpcResponse {
  id: string; // 對應請求 ID
  success: boolean; // 是否成功
  result?: unknown; // 結果
  error?: {
    // 錯誤資訊
    code: string;
    message: string;
  };
}

// 事件
interface RpcEvent {
  type: string; // 事件類型
  payload: unknown; // 事件資料
}
```

### 連接流程

```
Client                              Gateway
   │                                   │
   │──────── WebSocket Connect ───────▶│
   │                                   │
   │◀─────── Connection Accept ────────│
   │                                   │
   │──────── { method: "connect",      │
   │           params: {               │
   │             role: "control",      │
   │             token: "...",         │
   │             deviceId: "..."       │
   │           }                       │
   │         } ───────────────────────▶│
   │                                   │
   │◀─────── { success: true,          │
   │           result: {               │
   │             sessionId: "...",     │
   │             serverVersion: "..."  │
   │           }                       │
   │         } ────────────────────────│
   │                                   │
   │◀════════ Events (broadcast) ══════│
   │                                   │
```

## RPC 方法

### 聊天相關 (chat.\*)

| 方法                   | 說明             |
| ---------------------- | ---------------- |
| `chat.send`            | 發送使用者訊息   |
| `chat.abort`           | 中止執行中的回應 |
| `chat.sessions.list`   | 列出會話         |
| `chat.sessions.get`    | 取得會話詳情     |
| `chat.sessions.delete` | 刪除會話         |
| `chat.history`         | 取得對話歷史     |

### 配置相關 (config.\*)

| 方法            | 說明         |
| --------------- | ------------ |
| `config.get`    | 取得配置     |
| `config.set`    | 設定配置     |
| `config.list`   | 列出配置     |
| `config.reload` | 重新載入配置 |

### 頻道相關 (channels.\*)

| 方法                  | 說明     |
| --------------------- | -------- |
| `channels.status`     | 頻道狀態 |
| `channels.connect`    | 連接頻道 |
| `channels.disconnect` | 斷開頻道 |
| `channels.send`       | 發送訊息 |

### 節點相關 (node.\*)

| 方法                  | 說明         |
| --------------------- | ------------ |
| `node.register`       | 註冊節點     |
| `node.invoke.request` | 呼叫節點能力 |
| `node.invoke.result`  | 節點回傳結果 |
| `node.event`          | 節點事件     |

### 系統相關 (system.\*)

| 方法            | 說明     |
| --------------- | -------- |
| `system.health` | 健康檢查 |
| `system.info`   | 系統資訊 |
| `system.logs`   | 日誌查詢 |

## 運行時狀態

```typescript
// src/gateway/server-runtime-state.ts
export interface GatewayRuntimeState {
  wsConnections: Map<string, WebSocket>;
  chatRunState: Map<string, ChatRunState>;
  chatRunBuffers: Map<string, ChatRunBuffer>;
  chatAbortControllers: Map<string, AbortController>;
  nodes: Map<string, NodeRegistration>;
  pluginState: Map<string, unknown>;
}
```

## 常見事件類型

| 事件                      | 說明                  |
| ------------------------- | --------------------- |
| `chat.message`            | 新訊息（使用者/助手） |
| `chat.typing`             | 正在輸入              |
| `chat.tool.start`         | 工具開始執行          |
| `chat.tool.result`        | 工具執行結果          |
| `chat.complete`           | 回應完成              |
| `chat.error`              | 錯誤發生              |
| `channels.status.changed` | 頻道狀態變更          |
| `node.connected`          | 節點連接              |
| `node.disconnected`       | 節點斷開              |

## 節點系統

### 節點角色

- **control**: 控制客戶端（Web UI、macOS App）
- **node**: 能力節點（iOS App、Android App）
- **chat**: 聊天客戶端

### 節點能力

```typescript
interface NodeCapabilities {
  camera?: boolean;
  screenshot?: boolean;
  location?: boolean;
  microphone?: boolean;
  tts?: boolean;
  sms?: boolean; // Android
}
```

## HTTP 端點

### 健康檢查探針（Container Probes）

| 端點       | 方法     | 類型      | 說明                        |
| ---------- | -------- | --------- | --------------------------- |
| `/health`  | GET/HEAD | Liveness  | 存活探針                    |
| `/healthz` | GET/HEAD | Liveness  | 存活探針（Kubernetes 慣例） |
| `/ready`   | GET/HEAD | Readiness | 就緒探針                    |
| `/readyz`  | GET/HEAD | Readiness | 就緒探針（Kubernetes 慣例） |

適用於 Docker `HEALTHCHECK` 和 Kubernetes probe 配置。Dockerfile 內建 `HEALTHCHECK` 使用 `GET /healthz`，不需認證。

### 應用 API

| 端點          | 方法     | 說明          |
| ------------- | -------- | ------------- |
| `/api/chat`   | POST     | HTTP 聊天 API |
| `/api/config` | GET/POST | 配置 API      |
| `/webhooks/*` | POST     | Webhook 處理  |

## 配置選項

```typescript
interface GatewayServerOptions {
  port: number; // 埠號（預設 18789）
  bind: "loopback" | "lan" | string;
  token?: string;
  allowUnconfigured?: boolean;
  tlsCert?: string;
  tlsKey?: string;
}
```

## 啟動命令

```bash
# 基本啟動
openclaw gateway run

# 指定埠號和綁定
openclaw gateway run --port 18789 --bind lan

# 強制啟動
openclaw gateway run --force

# 允許未配置
openclaw gateway run --allow-unconfigured
```

## Cron 排程

Gateway 內建 Cron 排程系統，支援定時任務：

```json5
{
  cron: {
    enabled: true,
    sessionRetention: "24h",
    jobs: [
      {
        id: "daily-summary",
        schedule: "0 9 * * *",
        agent: "main",
        message: "每日摘要",
        channel: "discord",
        to: "user:USER_ID",
      },
    ],
  },
}
```

## Secrets 管理

Gateway 支援 Secrets 引用機制，配置中的敏感值可使用 `SecretRef` 引用：

```json5
{
  providers: {
    anthropic: {
      keyRef: "secrets/anthropic-key",
    },
  },
}
```

CLI 指令：

```bash
openclaw secrets reload    # 重新載入 secrets
openclaw secrets audit     # 審計 secrets 使用
openclaw secrets configure # 配置 secrets
```

## SecretRef 管理

Gateway 支援完整的 SecretRef 機制，涵蓋 64 個使用者提供的憑證目標：

- **執行時快照模型**：eager resolution + atomic swap
- **活動表面篩選**：未解析的 ref 在活動表面 fail fast，非活動表面報告非阻塞診斷
- **`openclaw secrets`**：`plan`（規劃）、`apply`（套用）、`audit`（審計）、`configure`（配置）流程
- **Onboarding SecretInput UX**：引導設定 SecretRef

## Agent 發現

`listConfiguredAgentIds` 現在也包含磁碟掃描的 agent IDs（即使 `agents.list` 已配置），確保磁碟上的 ACP agent session 在 session 聚合和列表中可見。

## Gateway Auth SecretRef

`gateway.auth.token` 現支援 SecretRef 引用，並加入 auth-mode guardrails：

```json5
{
  gateway: {
    auth: {
      token: { "$ref": "secret:gateway-token" }
    }
  }
}
```

## Gateway 密碼檔案輸入

`gateway run` 新增 `--password-file` 選項：

```bash
openclaw gateway run --password-file /path/to/password
```

## Channel-Backed Readiness Probes

Gateway 新增 channel-backed readiness probes，可透過頻道連線狀態判斷就緒：

```bash
openclaw channels status --probe
```

## Node Pending 機制

Gateway 新增 pending node work primitives 和 tightened drain semantics，確保節點工作在 Gateway 停止前完成排空。

## 啟動穩定性改善

- 啟動失敗在 run loop 中捕捉，防止 process exit
- 重啟前驗證 config 防止 crash 和 macOS 權限遺失
- 重啟 shutdown timeout 時 exit non-zero
- 非管理的 listeners 停止並重啟
- Config 無效載入 fail-closed

## 安全考量

1. **Token 認證**: 客戶端需提供有效 Token
2. **設備身份**: 支援設備簽名驗證（v2 簽名，已移除舊版 v1）
3. **TLS 加密**: 生產環境建議使用 WSS
4. **WS 安全策略**: 預設僅允許 loopback `ws://`，私網需 `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1` 明確 opt-in
5. **CORS 設定**: 控制跨域存取（Control UI 支援 `allowedOrigins: ["*"]` 萬用字元）
6. **速率限制**: 防止濫用（包括重複未授權請求洪水防護）
7. **Sanitize**: 最終回覆清理不受信任的包裝標記
8. **Secrets 引用**: inline SecretRef 自動規範化為 canonical `tokenRef`/`keyRef`
9. **HTTP 安全標頭**: 預設 `Permissions-Policy: camera=(), microphone=(), geolocation=()` + `X-Content-Type-Options: nosniff`（media route）
10. **Auth 標籤安全**: `/status` 和 `/models` 不再暴露 token/API key 片段
11. **denyCommands 審計**: 建議有效的 `gateway.nodes.denyCommands` 條目

## Dashboard v2（2026.3.12+）

Control UI dashboard 經過全面翻新，新增：
- 模組化 overview、chat、config、agent、session 視圖
- Command palette
- 行動版底部 tabs
- 豐富的聊天工具（slash commands、搜尋、匯出、釘選訊息）
- Tool events 即時串流至 control chat

## Gateway 改善（2026.3.13+）

- **用戶端請求限時**：限時並清理懸掛的 RPC calls，防止 stalled connections 無限期洩漏
- **殘留 socket 清理**：強制停止殘留的 gateway client sockets
- **--require-rpc**：`gateway status --require-rpc` 強制 RPC 探測失敗（自動化用途）
- **Scope-limited probe**：視為 degraded reachability
- **Session reset 路由保留**：保留 lastAccountId/lastThreadId 跨 reset
- **Main-session routing**：TUI/UI mode 保持在內部 surface
- **會話分離**：分離 conversation reset 和 admin reset
- **Token fallback**：強化 token fallback/reconnect 行為
- **Config validation issues**：dashboard 顯示 config validation issues
- **Runtime version**：狀態中暴露 runtime 版本
- **Before_tool_call**：HTTP tools 運行 before_tool_call hooks

## Windows Gateway 改善（2026.3.13+）

- schtasks 呼叫限時 + Startup-folder fallback
- 解析 fallback listeners 的正確 port 進行 stop
- 重用 service command env 讀取 runtime status
- 停止 loopback stale device signature 噪音
- windowsHide 隱藏 detached spawn console windows

## Kubernetes 支援（2026.3.12+）

新增入門級 K8s 安裝路徑：
- Raw manifests
- Kind 設定腳本
- 部署文件
- 健康檢查端點（`/healthz`、`/readyz`）適用於 K8s probe 配置

## Gateway 改善（2026.3.23+）

### 安全強化
- **Canvas route auth**：Canvas 路由現在要求身份驗證
- **Agent session reset admin**：Agent session reset 需要 admin 權限
- **Control UI operator.read**：修復 operator.read scope 處理
- **Discovery fail closed**：未解析的 discovery endpoints fail closed
- **Internal command gating**：限制內部指令的持久化變異
- **Hook ingress provenance**：保留非同步 hook ingress 來源資訊
- **OpenRouter pricing guard**：防止 auto pricing 遞迴

### 穩定性
- **SIGTERM 關閉**：強化 gateway SIGTERM 關閉行為
- **Probe auth**：完成 gateway probe auth landing
- **Status 韌性**：Status helpers 在 netif 失敗時保持韌性
- **Supervised lock**：強化 supervised lock 和 browser attach 就緒檢查
- **Probe false negatives**：避免連接後的 probe false negatives

### 功能
- **Plugin context lazy**：延遲解析 fallback plugin context
- **Discovery target**：集中化 discovery target 處理
- **.env 變數**：安裝時包含 .env 變數在 gateway service 環境中
- **Talk speak RPC**：新增 talk speak RPC 方法
- **Webchat 圖片**：inbound 圖片持久化至磁碟

### Doctor 改善
- **Plugin allowlist 清理**：清理過期的 plugin allowlist 和 entry refs
- **Plugin capability 摘要**：workspace status 新增 bundle plugin capability 摘要
- **Orphan transcript**：釐清孤立 transcript archive 提示

### Windows Gateway（2026.3.23+）
- npm 更新後正確重啟 gateway
- Windows media path 防護

## Gateway 改善（2026.3.24+）

### OpenAI 相容端點
- 新增 `/v1/models` 和 `/v1/embeddings` OpenAI 相容端點
- **Agent-first**：相容層以 Agent 為中心，回傳的模型列表對齊當前 Agent 設定
- 使第三方工具可透過 OpenAI 相容 API 與 OpenClaw Gateway 互動

### 穩定性
- **Connect challenge timeout**：提升預設 connect challenge timeout
- **Channel registry pin**：啟動時 pin channel registry 以 survive registry swaps
- **Restart sentinel**：重啟後喚醒 session 並保留 thread routing
- **Runtime state**：啟動 abort 時正確關閉 runtime state
- **Channel cache**：re-pin 時失效 channel caches
- **Minimal registry**：保持最小 gateway channel registry live

### Daemon
- `gateway start` 會 bootstrap 已停止的 service
- 集中化 daemon service 啟動狀態流程

### Doctor
- 更新期間跳過 service config repairs
- 非互動模式下遵從 --fix
- 新增 config clobber 鑑識

## Gateway 安全強化（2026.3.28+）

- **Reconnect scope**：封鎖靜默重連的 scope 升級
- **Admin verbose**：要求 admin 權限設定持久化 verbose defaults
- **Device revocation**：中斷已撤銷裝置的 sessions
- **Node pairing**：限制 node pairing 審批
- **Chat.send reset**：對齊 chat.send reset scope 檢查
- **Synthetic origins**：支援 synthetic chat origins
- **Session workspace reuse**：HTTP tool loading 重用 session workspace
- **Remote gateway confirm**：發現遠端 gateway 前確認再存入 config
