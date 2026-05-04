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

## Gateway 安全強化（2026.3.31）

### 認證與授權
- **Trusted-proxy 混合 token 拒絕**：`trusted-proxy` 現在拒絕混合的 shared-token 配置
- **Token rotation session 撤銷**：token rotation 後立即撤銷 active sessions
- **Mixed handshake rate limiting**：mixed handshake 下保持 shared-auth rate limiting
- **HTTP tool-invoke 授權收緊**：owner-only 工具不可透過 HTTP invoke
- **Embeddings HTTP write scope**：強制 embeddings HTTP write scope
- **Pre-auth websocket 上限**：並行 pre-auth websocket upgrade 上限
- **Bearer scope 忽略**：忽略 bearer-declared HTTP operator scopes
- **Trusted-proxy HTTP origin**：拒絕不匹配的 browser Origin headers
- **Plugin HTTP scope**：plugin-auth HTTP 路由限 read-only，gateway-auth 路由保持 write

### Node 命令與事件
- **Node 命令需配對核准**：node 命令在配對核准前保持停用
- **Node 事件信任縮減**：node 觸發執行保持在縮減的信任表面
- **Node pairing 前禁用 commands**：device pairing 不再足以暴露 node commands

### 背景任務系統 (ClawFlow)
- **SQLite-backed task ledger**：任務帳本從檔案式存儲遷移到 SQLite
- **Task flow control**：`openclaw flows list|show|cancel` 線性任務流程控制
- **Task audit/maintenance/status**：任務健康監控與維護命令
- **Session-scoped task updates**：task 更新按 session scope
- **Atomic task-store writes**：原子寫入確保資料一致性
- **Owner-key task access boundaries**：存取邊界限制

### OpenAI 相容性增強
- **Flat Responses tool definitions**：`/v1/responses` 支援 flat function tool 定義
- **Strict tool enforcement**：保留 `strict` 正規化
- **OpenAI version header**：新增 OpenAI version attribution header
- **Default operator scopes**：恢復 bearer-authenticated 請求的預設 operator scopes

### 其他改善
- **Session death loop 防護**：防止過載 fallback 的 session 死循環
- **SecretRef restart drift**：解決重啟 token 漂移偵測
- **Plugin route runtime scopes**：縮限 plugin 路由 runtime 範圍
- **Shared-secret HTTP auth**：標記為設計行為
- **Gateway startup plugin loading**：收緊 startup plugin loading
- **Config open without shell interpolation**：無 shell 插值開啟 config 檔案

## Gateway 改善（2026.4.5）

### 安全強化
- **Exec Fail-Open 修復**：修復 exec approval 在連線斷開時的 fail-open 漏洞，改為 fail-closed
- **Browser SSRF 防護**：瀏覽器工具新增 SSRF 保護（私有 IP 限制）
- **Sandbox Home Credential Binds**：sandbox 限制 home 目錄認證綁定
- **Bootstrap Token Revocation**：bootstrap token 使用後立即撤銷
- **Device Pairing Scope 收緊**：裝置配對 scope 進一步限縮
- **Hook Fail-closed**：hooks 執行失敗時 fail-closed 而非靜默跳過
- **Context Leak 修復**：修復多處 context 洩漏問題
- **Untrusted File Wrapping**：不受信任檔案內容加入安全包裝標記

### 穩定性
- **記憶體洩漏修復**：修復多處 Gateway 記憶體洩漏
- **WebSocket 改善**：WebSocket 連線穩定性改善
- **預設 Local Mode**：Gateway 預設為 local mode
- **HTTP Pipeline 錯誤處理**：改善 HTTP pipeline 錯誤處理

### 背景任務系統（TaskFlow）
- **ClawFlow 更名為 TaskFlow**：背景任務系統正式更名
- **Managed Child Execution**：TaskFlow 支援受管理的子任務執行
- **Chat-native Task Board**：聊天原生任務面板
- **Bound Runtime**：TaskFlow 綁定 runtime 執行環境
- **Task Ledger 改善**：任務帳本穩定性改善

### Prompt Cache 穩定性
- **確定性 MCP Tool 排序**：MCP 工具以確定性排序組裝，避免快取失效
- **壓縮策略優化**：壓縮策略從最新內容開始變異，保留快取前綴
- **3-turn Image Cache 視窗**：圖片快取固定 3 turn 視窗
- **Break 診斷**：新增 prompt cache break 診斷工具

### OpenAI 相容性
- **Responses API 改善**：`/v1/responses` 端點持續改善
- **Tool definitions 正規化**：工具定義正規化對齊 OpenAI 規範

### 其他
- **Video/Music 生成 Gateway 支援**：Gateway 支援 video_generate 和 music_generate 工具事件轉發
- **Cron --tools**：cron job 支援 per-job 工具白名單
- **webchat.chatHistoryMaxChars**：新增 webchat 歷史最大字元數配置

## Gateway 改善（2026.4.8）

### 安全

- **Secret Rotation Session 失效**：shared-token/password WS sessions 在 secret rotation 後立即失效（#62350）
- **SSRF Guard 修正**：SSRF guard 不再誤拒操作者配置的合法代理主機名（#62312）
- **Exec Approval Config 保護**：Gateway exec approval 配置路徑保護（#62001）
- **Node Reconnect Re-pairing**：node 重連 command 升級需重新配對（#62658）
- **Container Bind 收緊**：收緊容器綁定預設值（#61818）
- **Browser SSRF Redirect**：瀏覽器 SSRF 重定向防護加強（#62355）
- **Cross-origin Redirect Body Drop**：跨域 unsafe-method 重定向移除 request body（#62357）

### 穩定性

- **Compaction Checkpoints**：compaction 檢查點支援，斷線後可恢復
- **Compaction 無限迴圈修復**：tool use 中止後 compaction 不再無限迴圈（#62600）
- **/tts Audio in Webchat**：Control UI webchat 顯示 /tts 音訊（#61598）
- **Daemon Permission 處理**：permission-denied bus 錯誤時跳過 machine-scope fallback（#62337）
- **ThreadId Announce 傳遞**：sessions_send announce 正確傳遞 threadId（#62758）

### 功能

- **Pluggable Compaction**：可插拔壓縮 provider registry（#56224）
- **Webhooks TaskFlow Bridge**：新增 bundled webhooks TaskFlow bridge 插件（#61892）

---

## Gateway 改善（2026.4.12）

### 啟動與排程

- **Deferred Cron/Heartbeat Activation**：延遲 cron 與 heartbeat 啟動，直到 gateway 完全就緒
- **Scheduled Services Defer**：排程服務延遲啟動以避免競爭條件
- **MCP Loopback Lazy Start**：MCP loopback server 改為惰性啟動，減少冷啟動開銷

### 穩定性

- **Subagent Completion Dedupe**：子代理完成事件去重，避免重複處理（completion dedupe）
- **IdempotencyKey**：引入冪等鍵機制防止重複操作
- **Orphaned Agent Dir Warning**：偵測並警告孤立的 agent 目錄

### macOS Gateway

- **Chokidar Glob 修復**：修復 macOS 上 chokidar glob 監視問題
- **Parallels Gateway Fallback**：macOS Parallels 環境 gateway 回退機制改善

---

## Gateway 改善（2026.4.13–2026.4.14）

### Session 與路由

- **共用 Session Route 保留**：在系統事件期間保留共用 session route（#66073）
- **Stale Thread Route 清除**：在系統事件時清除過期的 thread route
- **Heartbeat Telegram Topic Routing**：保留孤立 heartbeats 的 Telegram topic routing（#66035）

### Cron 穩定性

- **Next-Run Refire Loop 修復**：停止未解析的 next-run 導致的重複觸發迴圈（#66083）
- **Next-Run Backoff 保留**：保留未解析的 next-run backoff（#66113）

### Gateway Config 安全防護

新增重要安全防護（#62006）：拒絕模型端 gateway 工具呼叫啟用危險安全旗標：
- `dangerouslyDisableDeviceAuth`、`allowInsecureAuth`
- `dangerouslyAllowHostHeaderOriginFallback`
- `hooks.gmail.allowUnsafeExternalContent`
- `tools.exec.applyPatch.workspaceOnly: false`

已啟用的旗標允許通過；直接認證的操作員 RPC 行為不受影響。

### Hooks 修復

- **workspaceDir 傳遞**：在 gateway session reset internal hook context 中傳遞 `workspaceDir`（#64735）
- **Ollama Slug Timeout**：讓 LLM-backed slug 生成遵守 `agents.defaults.timeoutSeconds`（#66237）

### Service Entrypoint

- **Service Entrypoint 加強**：加強 service entrypoint resolution（#65984）

### Subagent Registry

- **Registry Dist Output**：修復 subagent registry lazy-runtime stub 在 dist 輸出中的路徑問題（#66420, #66205）

### Docker 文件

- **Docker-out-of-Docker Paradox**：新增文件說明在 Docker 容器內執行 Gateway 的限制（#65473）

---

## Gateway 改善（2026.4.24）

### Restart continuation 與傳遞

- **持久交接**：將 restart continuation **先入 session-delivery 佇列**再移除 sentinel，避免競態丟失信號。  
- **崩潰恢復**：重啟後自佇列**恢復未送達 continuation** 工作。  
- **Wake 後援**：若重啟後無任何 channel route 存活，回退為 **session-only wake**（#70780）。

### MCP 與關機

- **Shutdown 清理**：Gateway 停止時 **dispose bundled MCP**，使已移除的 `mcp.servers` 子程序可被回收。  
- **熱重載**：偵測 `mcp.*` 設定變更時 **釋放已快取的 session MCP runtimes**。  
- **閒置 TTL**：設定 **`mcp.sessionIdleTtlMs`** 以驅逐洩漏的 session runtime。

### Exec 診斷

- **Exec 程序 telemetry**：診斷路徑可發射 **exec 子程序**相關遙測（見 OpenTelemetry／診斷文件）。

### Session 與瀏覽器

- **Session rollover**：閒置、每日換日、`/new`、`/reset` 封存舊 transcript 時，**關閉先前追蹤的 browser tabs**，避免跨 session 分頁洩漏。

---

## Gateway 改善（2026.4.25）

### 探針與 HTTP 舞台

- **`/healthz`／`/readyz`**：在插件、canvas、Control UI 等較重路由之前 **預先註冊**，後段 handler 卡住時 **liveness／readiness** 仍可回應（#69674）。

### Bonjour／mDNS

- **Docker Compose**：bridge 網路預設 **關閉 Bonjour 廣播**；host／macvlan 需 **`OPENCLAW_DISABLE_BONJOUR=0`** 才維持 opt-in（#71879）。  
- **容器**：偵測容器環境時可 **自動關閉 mDNS 廣告**（見 changelog）。  
- **失敗退避**：ciao watchdog／probing **不再無限重試**拖垮行程（#69011）。

### 配對／身分與更新

- **Tailscale**：已認證之 Control UI operator（瀏覽器裝置身分）可 **略過部分 device pairing** 往返（#71986）。  
- **LaunchAgent**：服務安裝紀錄若偵測 **過期內嵌 gateway auth**，改 **刷新**而非誤報 already-installed（#70752）。  
- **套件更新**：重啟後 Gateway **版本不符**視為更新失敗（含 fallback／JSON 模式）（#71835）。  
- **混版二進位**：較舊 openclaw **不可**對較新 config 環境做 **process／service mutation**（#57079）。

### 環境 Proxy

- **啟動**：自 Gateway **直接啟動**即套用 **`HTTPS_PROXY`／`HTTP_PROXY`**，在第一個 embedded agent 網路請求前生效（#71833）。
- **`ALL_PROXY`／`all_proxy`**：一併進入全域 Undici／provider fetch path（與 changelog SSRF 政策並存）（#43821）。

---

## Gateway 改善（2026.4.26）

### 握手與啟動路徑

- **`hello-ok`**：初始快照含 **連線客戶端** 與 **presence 版本**。  
- **前景 Gateway**：略過 CLI **自我 respawn**，降低低記憶體／Node 24 **啟動前 hang**（#72720）。  
- **無效插件 schema**：單一插件設定 schema 錯誤時改 **degraded 啟動**；**`openclaw doctor --fix`** 可 **隔離**問題設定（#62976、#70371）。

### Metadata 重用

- **Config snapshot 內 manifests**：驅動 **auto-enable**、驗證與 bootstrap **規劃**，減少開機 **重複掃描**。

### WebChat／連線衛生

- **Handshake**：綁定 **作用中 socket**；連線 close 後 **拒絕遲到註冊**，避免 zombie client／誤導重複連線日誌（#72753）。

---

## Gateway 改善（2026.4.27）

> 對齊 **`CHANGELOG.md`**：**下文 `## Gateway 改善（2026.5.2）`** 對 **`## 2026.5.2`**；本節標題 **`（2026.4.27）`** 保留為先前 **Unreleased** 時期條目的時間標記——**繁體文件全域版本**請以 **`2026.5.2`**（README／頁尾／API `version`）為準。

### 設定與啟動效能

- **無效設定**：啟動／熱重載 **fail closed**，**不**再自動還原損壞或無法驗證之設定；**`openclaw doctor --fix`** 負責安全遷移（even其他驗證仍失敗時亦尽量套用已知 legacy 清理）（#76798、#76800）。  
- **延後載入**：**cron runtime**、部分 **discovery**、**shutdown／maintenance timer** 延至 **ready 之後**；縮減 **plugin auto-enable** 重複工作；**native 插件路徑**盡量 **避免 jiti**。  
- **剖析**：Gateway startup benchmark 支援 **`--cpu-prof-dir`**；**`pnpm gateway:watch --benchmark-no-force`** 略過預設埠清理以利基準。

### Sessions／用量／模型列表

- **`sessions.list`**：**預設有界**回應並附 **truncation metadata**（#77062）；列表列 **thinking／cost** 計算 **memo** 與 **skip** 冗餘工作（#76931）。  
- **用量**：**`usage.cost`／`sessions.usage`** 改以 **transcript aggregate cache** 服務，附 **stale** 標示（#76650）。  
- **`models.list` 唯讀路徑**：減少不必要 **runtime provider discovery**（延續 #76382 一族）。

### Canvas／日誌／安裝

- **Canvas URL**：保留 **Gateway TLS scheme** 於瀏覽器 canvas 連結與 **bind 後**之 mount 日誌。  
- **`logging.file`**：路徑前導 **`~`** 正規化後再建檔（#73587）。  
- **LaunchAgent**：保留 **`.env` 託管鍵**於 env 檔；**managed keys** 與 operator secrets **跨 re-stage** 行為調整（#75374、#76860）。

---

## Gateway 改善（2026.5.2）

> 對齊 **`CHANGELOG.md` → `## 2026.5.2`**；完整主題見 **`docs-cht/commit-analyze-20260502.md`**。

### 啟動／RPC／連線／診斷

- **Secrets preflight**：可跳過插件背書 **auth-profile overlay**，縮短 readiness（reload／OAuth 恢復仍可用 overlay）。  
- **`gateway restart`**：**`--force`**、**`--wait`**；記錄作用中 task run id；逾時視為強制重啟（#68327 脈絡）。  
- **`$include`**：**`OPENCLAW_INCLUDE_ROOTS`** 核准目錄可讀檔，預設仍限制設定目錄。  
- **通道啟動 fan-out**：至多 **四個** channel／account handoff；Bonjour **ciao self-probe** 競態恢復（#75687）。  
- **定價 fetch**：延後至 ready 路徑後；shutdown **中止**進行中定價請求並避免 shutdown 後寫快取（#74128、#72208）。  
- **穩定性 bundle**：bounded **redacted startup error**（#75797）。

### Sessions／tasks／transcripts／chat history

- **`sessions.list`**：大型 store **輕量 compaction checkpoint／索引**，維持輪詢可負擔。  
- **Session-store writer**：集中寫入槽與快取借用，降 **`sessions.json`** 鎖與重讀（#68554）。  
- **Transcript 鎖**：**`acquireTimeoutMs`** 統一政策；預設等待 **60s**（#75894）。  
- **Bounded／async transcript IO**：詳情、歷史、artifact、compaction、subscribe 等路徑 **流式／分段讀**，降低 OOM（見 changelog 多條）。  
- **Chat history**：依請求視窗 bound transcript read，避免「只要一小頁卻載入整份超大 log」。

### Status／CLI／macOS 服務

- **`gateway start`**：修復 **過時 managed service**（指到舊版／缺 binary／暫存 installer path）。  
- **Probe 失敗**：輸出具體 **下一步**（service、設定、listener-owner、log）（#75944 脈絡）。  
- **LaunchAgent PATH**：寫入 **canonical system PATH**，不再保留舊 plist PATH（Volta／asdf／fnm／pnpm 不再污染子程序 Node）（#75233）。

### Sidecars／pricing／models

- **Early RPC**：sessions.create／send／abort、agent.wait、tools.effective 等在 **可重試 startup-sidecars** 錯誤時回同一語意（#76012）。  
- **`models.list`**：catalog discovery stall 時維持預設視圖響應；**`--all`** 仍等待完整目錄（#75297）。
