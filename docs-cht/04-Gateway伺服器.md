# Gateway 伺服器

## 概述

Gateway 是 OpenClaw 的核心伺服器元件，提供 WebSocket 即時通訊、RPC 方法處理、會話管理等功能。所有客戶端（Web UI、行動應用、CLI）都透過 Gateway 與系統互動。

## 架構

```
                    ┌─────────────────────────────────────┐
                    │          Gateway Server             │
                    │                                     │
┌─────────┐         │  ┌──────────────────────────────┐  │
│ Web UI  │─────────┼─▶│     WebSocket Handler        │  │
└─────────┘         │  │  ┌────────────────────────┐  │  │
                    │  │  │ Connection Manager     │  │  │
┌─────────┐         │  │  │ - authenticate         │  │  │
│  iOS    │─────────┼─▶│  │ - route messages       │  │  │
│  App    │         │  │  │ - broadcast events     │  │  │
└─────────┘         │  │  └────────────────────────┘  │  │
                    │  └──────────────────────────────┘  │
┌─────────┐         │                                     │
│ Android │─────────┼─▶┌──────────────────────────────┐  │
│  App    │         │  │      RPC Methods            │  │
└─────────┘         │  │  - chat.*                    │  │
                    │  │  - config.*                  │  │
┌─────────┐         │  │  - channels.*               │  │
│ macOS   │─────────┼─▶│  - node.*                   │  │
│  App    │         │  │  - system.*                 │  │
└─────────┘         │  └──────────────────────────────┘  │
                    │                                     │
                    │  ┌──────────────────────────────┐  │
                    │  │     Runtime State           │  │
                    │  │  - chatRunState             │  │
                    │  │  - chatRunBuffers           │  │
                    │  │  - wsConnections            │  │
                    │  └──────────────────────────────┘  │
                    └─────────────────────────────────────┘
```

## 核心檔案

| 檔案 | 說明 |
|------|------|
| `src/gateway/server.impl.ts` | Gateway 伺服器主實作 |
| `src/gateway/server-runtime-state.ts` | 運行時狀態管理 |
| `src/gateway/server-methods.ts` | RPC 方法處理器 |
| `src/gateway/server-methods/*.ts` | 各類方法實作 |
| `src/gateway/server/ws-connection/` | WebSocket 連接處理 |
| `src/gateway/protocol/` | 協議定義（TypeBox） |

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
  wss.on('connection', (ws, req) => {
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
  id: string;           // 請求 ID
  method: string;       // 方法名稱
  params?: unknown;     // 參數
}

// 回應
interface RpcResponse {
  id: string;           // 對應請求 ID
  success: boolean;     // 是否成功
  result?: unknown;     // 結果
  error?: {             // 錯誤資訊
    code: string;
    message: string;
  };
}

// 事件
interface RpcEvent {
  type: string;         // 事件類型
  payload: unknown;     // 事件資料
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

### 聊天相關 (chat.*)

| 方法 | 說明 |
|------|------|
| `chat.send` | 發送使用者訊息 |
| `chat.abort` | 中止執行中的回應 |
| `chat.sessions.list` | 列出會話 |
| `chat.sessions.get` | 取得會話詳情 |
| `chat.sessions.delete` | 刪除會話 |
| `chat.history` | 取得對話歷史 |

### 配置相關 (config.*)

| 方法 | 說明 |
|------|------|
| `config.get` | 取得配置 |
| `config.set` | 設定配置 |
| `config.list` | 列出配置 |
| `config.reload` | 重新載入配置 |

### 頻道相關 (channels.*)

| 方法 | 說明 |
|------|------|
| `channels.status` | 頻道狀態 |
| `channels.connect` | 連接頻道 |
| `channels.disconnect` | 斷開頻道 |
| `channels.send` | 發送訊息 |

### 節點相關 (node.*)

| 方法 | 說明 |
|------|------|
| `node.register` | 註冊節點 |
| `node.invoke.request` | 呼叫節點能力 |
| `node.invoke.result` | 節點回傳結果 |
| `node.event` | 節點事件 |

### 系統相關 (system.*)

| 方法 | 說明 |
|------|------|
| `system.health` | 健康檢查 |
| `system.info` | 系統資訊 |
| `system.logs` | 日誌查詢 |

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

| 事件 | 說明 |
|------|------|
| `chat.message` | 新訊息（使用者/助手） |
| `chat.typing` | 正在輸入 |
| `chat.tool.start` | 工具開始執行 |
| `chat.tool.result` | 工具執行結果 |
| `chat.complete` | 回應完成 |
| `chat.error` | 錯誤發生 |
| `channels.status.changed` | 頻道狀態變更 |
| `node.connected` | 節點連接 |
| `node.disconnected` | 節點斷開 |

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
  sms?: boolean;  // Android
}
```

## HTTP 端點

| 端點 | 方法 | 說明 |
|------|------|------|
| `/health` | GET | 健康檢查 |
| `/api/chat` | POST | HTTP 聊天 API |
| `/api/config` | GET/POST | 配置 API |
| `/webhooks/*` | POST | Webhook 處理 |

## 配置選項

```typescript
interface GatewayServerOptions {
  port: number;               // 埠號（預設 18789）
  bind: 'loopback' | 'lan' | string;
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
        to: "user:USER_ID"
      }
    ]
  }
}
```

## 安全考量

1. **Token 認證**: 客戶端需提供有效 Token
2. **設備身份**: 支援設備簽名驗證（v2 簽名，已移除舊版 v1）
3. **TLS 加密**: 生產環境建議使用 WSS
4. **CORS 設定**: 控制跨域存取
5. **速率限制**: 防止濫用
6. **Sanitize**: 最終回覆清理不受信任的包裝標記
