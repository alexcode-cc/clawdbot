# API 參考

## Gateway RPC API

Gateway 透過 WebSocket 提供 JSON-RPC 風格的 API。

### 連接

```javascript
const ws = new WebSocket('ws://localhost:18789');

// 連接後發送認證
ws.onopen = () => {
  ws.send(JSON.stringify({
    id: 'connect-1',
    method: 'connect',
    params: {
      role: 'control',  // control | node | chat
      token: 'your-token',
      deviceId: 'device-123',
    }
  }));
};
```

### 請求格式

```typescript
interface RpcRequest {
  id: string;         // 請求 ID（用於匹配回應）
  method: string;     // 方法名稱
  params?: unknown;   // 參數
}
```

### 回應格式

```typescript
interface RpcResponse {
  id: string;         // 對應請求 ID
  success: boolean;   // 是否成功
  result?: unknown;   // 成功時的結果
  error?: {           // 失敗時的錯誤
    code: string;
    message: string;
    details?: unknown;
  };
}
```

### 事件格式

```typescript
interface RpcEvent {
  type: string;       // 事件類型
  payload: unknown;   // 事件資料
}
```

## 聊天 API (chat.*)

### chat.send

發送使用者訊息。

```typescript
// 請求
{
  id: 'msg-1',
  method: 'chat.send',
  params: {
    sessionKey: 'session-123',      // 會話鍵值
    message: 'Hello, how are you?', // 訊息內容
    attachments?: [                  // 附件（可選）
      {
        type: 'image',
        url: 'https://...',
        mimeType: 'image/png'
      }
    ]
  }
}

// 回應
{
  id: 'msg-1',
  success: true,
  result: {
    messageId: 'user-msg-1'
  }
}
```

### chat.abort

中止執行中的回應。

```typescript
// 請求
{
  id: 'abort-1',
  method: 'chat.abort',
  params: {
    sessionKey: 'session-123'
  }
}

// 回應
{
  id: 'abort-1',
  success: true
}
```

### chat.sessions.list

列出所有會話。

```typescript
// 請求
{
  id: 'list-1',
  method: 'chat.sessions.list',
  params: {
    limit?: 50,
    offset?: 0
  }
}

// 回應
{
  id: 'list-1',
  success: true,
  result: {
    sessions: [
      {
        key: 'session-123',
        agentId: 'default',
        channelId: 'web',
        chatId: 'user-1',
        messageCount: 42,
        createdAt: '2024-01-01T00:00:00Z',
        updatedAt: '2024-01-02T12:00:00Z'
      }
    ],
    total: 100
  }
}
```

### chat.sessions.get

取得會話詳情。

```typescript
// 請求
{
  id: 'get-1',
  method: 'chat.sessions.get',
  params: {
    sessionKey: 'session-123'
  }
}

// 回應
{
  id: 'get-1',
  success: true,
  result: {
    session: {
      key: 'session-123',
      agentId: 'default',
      messages: [
        { role: 'user', content: 'Hello' },
        { role: 'assistant', content: 'Hi there!' }
      ]
    }
  }
}
```

### chat.sessions.delete

刪除會話。

```typescript
// 請求
{
  id: 'del-1',
  method: 'chat.sessions.delete',
  params: {
    sessionKey: 'session-123'
  }
}

// 回應
{
  id: 'del-1',
  success: true
}
```

## 配置 API (config.*)

### config.get

取得配置值。

```typescript
// 請求
{
  id: 'cfg-1',
  method: 'config.get',
  params: {
    key: 'models.default'
  }
}

// 回應
{
  id: 'cfg-1',
  success: true,
  result: {
    value: 'claude-sonnet'
  }
}
```

### config.set

設定配置值。

```typescript
// 請求
{
  id: 'cfg-2',
  method: 'config.set',
  params: {
    key: 'models.default',
    value: 'gpt-4o'
  }
}

// 回應
{
  id: 'cfg-2',
  success: true
}
```

### config.list

列出所有配置。

```typescript
// 請求
{
  id: 'cfg-3',
  method: 'config.list'
}

// 回應
{
  id: 'cfg-3',
  success: true,
  result: {
    config: {
      models: { ... },
      providers: { ... },
      channels: { ... }
    }
  }
}
```

## 頻道 API (channels.*)

### channels.status

取得頻道狀態。

```typescript
// 請求
{
  id: 'ch-1',
  method: 'channels.status',
  params: {
    channelId?: 'telegram',  // 可選，不指定則返回全部
    deep?: false              // 是否深度探測
  }
}

// 回應
{
  id: 'ch-1',
  success: true,
  result: {
    channels: [
      {
        id: 'telegram',
        name: 'Telegram',
        connected: true,
        authenticated: true,
        lastActivity: '2024-01-02T12:00:00Z'
      },
      {
        id: 'whatsapp',
        name: 'WhatsApp',
        connected: false,
        error: 'Not logged in'
      }
    ]
  }
}
```

### channels.connect

連接頻道。

```typescript
// 請求
{
  id: 'ch-2',
  method: 'channels.connect',
  params: {
    channelId: 'telegram'
  }
}

// 回應
{
  id: 'ch-2',
  success: true,
  result: {
    connected: true
  }
}
```

### channels.disconnect

斷開頻道。

```typescript
// 請求
{
  id: 'ch-3',
  method: 'channels.disconnect',
  params: {
    channelId: 'telegram'
  }
}

// 回應
{
  id: 'ch-3',
  success: true
}
```

### channels.send

透過頻道發送訊息。

```typescript
// 請求
{
  id: 'ch-4',
  method: 'channels.send',
  params: {
    channelId: 'telegram',
    chatId: '@username',
    message: {
      content: 'Hello!',
      format: 'markdown'
    }
  }
}

// 回應
{
  id: 'ch-4',
  success: true,
  result: {
    messageId: 'msg-12345'
  }
}
```

## 節點 API (node.*)

### node.register

註冊節點（由 App 呼叫）。

```typescript
// 請求
{
  id: 'node-1',
  method: 'node.register',
  params: {
    deviceId: 'iphone-123',
    platform: 'ios',
    capabilities: {
      camera: true,
      microphone: true,
      location: true,
      tts: true
    },
    commands: [
      {
        name: 'camera.capture',
        description: 'Take a photo',
        parameters: { ... }
      }
    ]
  }
}

// 回應
{
  id: 'node-1',
  success: true,
  result: {
    nodeId: 'node-456'
  }
}
```

### node.invoke.request

呼叫節點能力（由 Gateway 發送給 Node）。

```typescript
// 事件（Gateway → Node）
{
  type: 'node.invoke.request',
  payload: {
    id: 'invoke-1',
    command: 'camera.capture',
    params: {
      resolution: 'high'
    }
  }
}
```

### node.invoke.result

節點回傳結果（由 Node 發送給 Gateway）。

```typescript
// 請求（Node → Gateway）
{
  id: 'result-1',
  method: 'node.invoke.result',
  params: {
    id: 'invoke-1',  // 對應請求 ID
    success: true,
    result: {
      imageUrl: 'https://...',
      mimeType: 'image/jpeg'
    }
  }
}
```

## 系統 API (system.*)

### system.health

健康檢查。

```typescript
// 請求
{
  id: 'sys-1',
  method: 'system.health'
}

// 回應
{
  id: 'sys-1',
  success: true,
  result: {
    status: 'healthy',
    version: '2024.1.27',
    uptime: 3600,
    memory: {
      used: 128000000,
      total: 512000000
    }
  }
}
```

### system.info

系統資訊。

```typescript
// 請求
{
  id: 'sys-2',
  method: 'system.info'
}

// 回應
{
  id: 'sys-2',
  success: true,
  result: {
    version: '2024.1.27',
    nodeVersion: '22.0.0',
    platform: 'darwin',
    arch: 'arm64',
    configPath: '/Users/user/.clawdbot/config.json5'
  }
}
```

## 事件類型

### 聊天事件

```typescript
// 新訊息
{
  type: 'chat.message',
  payload: {
    sessionKey: 'session-123',
    message: {
      id: 'msg-1',
      role: 'assistant',
      content: 'Hello!'
    }
  }
}

// 正在輸入
{
  type: 'chat.typing',
  payload: {
    sessionKey: 'session-123',
    isTyping: true
  }
}

// 工具開始執行
{
  type: 'chat.tool.start',
  payload: {
    sessionKey: 'session-123',
    toolCallId: 'call-1',
    toolName: 'read',
    params: { path: '/file.txt' }
  }
}

// 工具執行結果
{
  type: 'chat.tool.result',
  payload: {
    sessionKey: 'session-123',
    toolCallId: 'call-1',
    result: { content: '...' }
  }
}

// 回應完成
{
  type: 'chat.complete',
  payload: {
    sessionKey: 'session-123',
    usage: {
      inputTokens: 100,
      outputTokens: 50
    }
  }
}

// 錯誤
{
  type: 'chat.error',
  payload: {
    sessionKey: 'session-123',
    error: {
      code: 'RATE_LIMIT',
      message: 'Rate limit exceeded'
    }
  }
}
```

### 頻道事件

```typescript
// 頻道狀態變更
{
  type: 'channels.status.changed',
  payload: {
    channelId: 'telegram',
    connected: true,
    authenticated: true
  }
}
```

### 節點事件

```typescript
// 節點連接
{
  type: 'node.connected',
  payload: {
    nodeId: 'node-123',
    deviceId: 'iphone-456',
    platform: 'ios'
  }
}

// 節點斷開
{
  type: 'node.disconnected',
  payload: {
    nodeId: 'node-123',
    reason: 'Connection lost'
  }
}
```

## Plugin SDK API

### MoltbotPluginApi

```typescript
interface MoltbotPluginApi {
  // 運行時
  runtime: PluginRuntime;
  config: PluginConfig;
  logger: Logger;
  
  // 註冊方法
  registerChannel(opts: ChannelRegistration): void;
  registerTool(tool: ToolDefinition, opts?: ToolOptions): void;
  registerGatewayMethod(name: string, handler: RpcHandler): void;
  registerHttpHandler(handler: HttpHandler): void;
  registerCli(registrar: CliRegistrar, opts?: CliOptions): void;
  registerService(service: ServiceDefinition): void;
  registerProvider(provider: ProviderDefinition): void;
}
```

### ToolDefinition

```typescript
interface ToolDefinition {
  name: string;
  description: string;
  parameters: TSchema;  // TypeBox Schema
  execute: (
    toolCallId: string,
    params: Record<string, unknown>,
    context: ToolContext
  ) => Promise<ToolResult>;
}
```

### ChannelPlugin

```typescript
interface ChannelPlugin {
  id: string;
  name: string;
  
  initialize(config: ChannelConfig): Promise<void>;
  startMonitor(handler: MessageHandler): Promise<void>;
  sendMessage(chatId: string, message: OutgoingMessage): Promise<void>;
  getStatus(): Promise<ChannelStatus>;
  shutdown(): Promise<void>;
}
```

### RpcHandler

```typescript
type RpcHandler = (context: {
  params: unknown;
  respond: (success: boolean, data: unknown) => void;
  broadcast: (event: string, payload: unknown) => void;
  connectionId: string;
}) => Promise<void>;
```

## 錯誤碼

| 錯誤碼 | 說明 |
|--------|------|
| `INVALID_REQUEST` | 無效的請求格式 |
| `UNAUTHORIZED` | 未授權 |
| `NOT_FOUND` | 資源不存在 |
| `RATE_LIMIT` | 超過速率限制 |
| `INTERNAL_ERROR` | 內部錯誤 |
| `CHANNEL_ERROR` | 頻道錯誤 |
| `MODEL_ERROR` | 模型錯誤 |
| `TOOL_ERROR` | 工具執行錯誤 |

## 使用範例

### JavaScript/TypeScript

```typescript
const ws = new WebSocket('ws://localhost:18789');

// 發送請求的輔助函數
function sendRequest(method: string, params?: unknown): Promise<unknown> {
  return new Promise((resolve, reject) => {
    const id = Math.random().toString(36).substring(7);
    
    const handler = (event: MessageEvent) => {
      const data = JSON.parse(event.data);
      if (data.id === id) {
        ws.removeEventListener('message', handler);
        if (data.success) {
          resolve(data.result);
        } else {
          reject(new Error(data.error.message));
        }
      }
    };
    
    ws.addEventListener('message', handler);
    ws.send(JSON.stringify({ id, method, params }));
  });
}

// 使用
await sendRequest('connect', { role: 'control', token: 'xxx' });
await sendRequest('chat.send', { sessionKey: 'session-1', message: 'Hello' });
```

### Python

```python
import asyncio
import websockets
import json
import uuid

async def main():
    async with websockets.connect('ws://localhost:18789') as ws:
        # 連接
        await ws.send(json.dumps({
            'id': str(uuid.uuid4()),
            'method': 'connect',
            'params': {'role': 'control', 'token': 'xxx'}
        }))
        response = json.loads(await ws.recv())
        print(response)
        
        # 發送訊息
        await ws.send(json.dumps({
            'id': str(uuid.uuid4()),
            'method': 'chat.send',
            'params': {
                'sessionKey': 'session-1',
                'message': 'Hello'
            }
        }))
        
        # 接收事件
        while True:
            data = json.loads(await ws.recv())
            if 'type' in data:
                print(f"Event: {data['type']}")
            else:
                print(f"Response: {data}")

asyncio.run(main())
```
