# API 參考

## Gateway RPC API

Gateway 透過 WebSocket 提供 JSON-RPC 風格的 API。

### 連接

```javascript
const ws = new WebSocket('ws://localhost:18789');

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
  id: string;
  method: string;
  params?: unknown;
}
```

### 回應格式

```typescript
interface RpcResponse {
  id: string;
  success: boolean;
  result?: unknown;
  error?: {
    code: string;
    message: string;
    details?: unknown;
  };
}
```

### 事件格式

```typescript
interface RpcEvent {
  type: string;
  payload: unknown;
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
    sessionKey: 'session-123',
    message: 'Hello, how are you?',
    attachments?: [{
      type: 'image',
      url: 'https://...',
      mimeType: 'image/png'
    }]
  }
}

// 回應
{
  id: 'msg-1',
  success: true,
  result: { messageId: 'user-msg-1' }
}
```

### chat.abort

中止執行中的回應。

### chat.sessions.list

列出所有會話。

### chat.sessions.get

取得會話詳情。

### chat.sessions.delete

刪除會話。

## 配置 API (config.*)

### config.get

取得配置值（敏感值自動脫敏）。

### config.set

設定配置值。

### config.list

列出所有配置。

### config.reload

重新載入配置。

## 頻道 API (channels.*)

### channels.status

取得頻道狀態。

```typescript
{
  id: 'ch-1',
  method: 'channels.status',
  params: {
    channelId?: 'telegram',
    deep?: false
  }
}
```

### channels.connect / channels.disconnect

連接/斷開頻道。

### channels.send

透過頻道發送訊息。

## 節點 API (node.*)

### node.register

註冊節點（由 App 呼叫）。

```typescript
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
    commands: [{
      name: 'camera.capture',
      description: 'Take a photo',
      parameters: { ... }
    }]
  }
}
```

### node.invoke.request / node.invoke.result

呼叫節點能力並接收結果。

## 系統 API (system.*)

### system.health

```typescript
{
  id: 'sys-1',
  success: true,
  result: {
    status: 'healthy',
    version: '2026.2.22',
    uptime: 3600,
    memory: { used: 128000000, total: 512000000 }
  }
}
```

### system.info

系統資訊。

## 事件類型

### 聊天事件

| 事件 | 說明 |
|------|------|
| `chat.message` | 新訊息（使用者/助手） |
| `chat.typing` | 正在輸入 |
| `chat.tool.start` | 工具開始執行 |
| `chat.tool.result` | 工具執行結果 |
| `chat.complete` | 回應完成 |
| `chat.error` | 錯誤發生 |

### 頻道事件

| 事件 | 說明 |
|------|------|
| `channels.status.changed` | 頻道狀態變更 |

### 節點事件

| 事件 | 說明 |
|------|------|
| `node.connected` | 節點連接 |
| `node.disconnected` | 節點斷開 |

## 子代理 API (subagents.*)

### subagents.list

列出所有子代理。

### subagents.stop

停止指定子代理。

### subagents.send

向子代理發送訊息。

## 設備配對 API (device.*)

### device.pair.list

列出待配對的設備。

### device.pair.approve

批准設備配對請求。

### device.pair.reject

拒絕設備配對請求。

## Plugin SDK API

### OpenClawPluginApi

```typescript
interface OpenClawPluginApi {
  runtime: PluginRuntime;
  config: PluginConfig;
  logger: Logger;
  
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
  parameters: TSchema;
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

function sendRequest(method: string, params?: unknown): Promise<unknown> {
  return new Promise((resolve, reject) => {
    const id = Math.random().toString(36).substring(7);
    const handler = (event: MessageEvent) => {
      const data = JSON.parse(event.data);
      if (data.id === id) {
        ws.removeEventListener('message', handler);
        data.success ? resolve(data.result) : reject(new Error(data.error.message));
      }
    };
    ws.addEventListener('message', handler);
    ws.send(JSON.stringify({ id, method, params }));
  });
}

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
        await ws.send(json.dumps({
            'id': str(uuid.uuid4()),
            'method': 'connect',
            'params': {'role': 'control', 'token': 'xxx'}
        }))
        response = json.loads(await ws.recv())
        print(response)
        
        await ws.send(json.dumps({
            'id': str(uuid.uuid4()),
            'method': 'chat.send',
            'params': {'sessionKey': 'session-1', 'message': 'Hello'}
        }))
        
        while True:
            data = json.loads(await ws.recv())
            if 'type' in data:
                print(f"Event: {data['type']}")
            else:
                print(f"Response: {data}")

asyncio.run(main())
```
