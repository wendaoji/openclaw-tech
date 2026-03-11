# OpenClaw Gateway 模块技术原理详解

## 1. 模块概述

Gateway 是 OpenClaw 的核心控制平面，通过 WebSocket 协议管理所有客户端连接、会话、通道和消息路由。它是整个系统的中枢，负责协调各个模块的运行。

**核心位置**: `src/gateway/` (244 个文件)

## 2. 架构设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Gateway Server                                │
│                     (server.impl.ts)                                │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │   Config   │  │   Auth      │  │   Secrets   │  │  Plugins │ │
│  │  Manager   │  │  Manager    │  │  Manager    │  │  Manager │ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └──────────┘ │
├─────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    Server Methods                             │   │
│  │  agent | chat | sessions | channels | cron | tools | config  │   │
│  └──────────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │
│  │  Channel   │  │   Node      │  │   Canvas    │               │
│  │  Manager   │  │  Registry   │  │   Host      │               │
│  └─────────────┘  └─────────────┘  └─────────────┘               │
├─────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                 WebSocket Server                              │   │
│  │     ws://gateway-host:18789 (客户端连接)                    │   │
│  └──────────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                 HTTP Server (Control UI)                      │   │
│  │     http://gateway-host:18789 (Web UI)                      │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

## 3. 核心组件

### 3.1 Server Methods (服务器方法)

Gateway 通过方法调用处理客户端请求。核心方法分类：

| 分类 | 方法 | 功能 |
|------|------|------|
| **Agent** | `agent.*` | AI 智能体交互 |
| **Chat** | `chat.*` | 消息发送/接收 |
| **Sessions** | `sessions.*` | 会话管理 |
| **Channels** | `channels.*` | 通道控制 |
| **Cron** | `cron.*` | 定时任务 |
| **Tools** | `tools.*` | 工具目录 |
| **Config** | `config.*` | 配置管理 |
| **Secrets** | `secrets.*` | 密钥管理 |
| **Nodes** | `nodes.*` | 移动节点 |

**核心文件**: `src/gateway/server-methods/`

```typescript
// server-methods/types.ts
export type GatewayRequestHandler = (opts: GatewayRequestHandlerOptions) => Promise<void> | void;

export type GatewayRequestHandlers = Record<string, GatewayRequestHandler>;
```

### 3.2 通道管理 (Channel Manager)

负责消息通道的启动、停止和生命周期管理。

```typescript
// server-channels.ts
export type ChannelManager = {
  getRuntimeSnapshot: () => ChannelRuntimeSnapshot;
  startChannels: () => Promise<void>;
  startChannel: (channel: ChannelId, accountId?: string) => Promise<void>;
  stopChannel: (channel: ChannelId, accountId?: string) => Promise<void>;
  markChannelLoggedOut: (channelId: ChannelId, cleared: boolean, accountId?: string) => void;
  isManuallyStopped: (channelId: ChannelId, accountId: string) => boolean;
  resetRestartAttempts: (channelId: ChannelId, accountId: string) => void;
};
```

**通道重启策略**:
```typescript
const CHANNEL_RESTART_POLICY: BackoffPolicy = {
  initialMs: 5_000,
  maxMs: 5 * 60_000,
  factor: 2,
  jitter: 0.1,
};
```

### 3.3 节点注册 (Node Registry)

管理移动节点 (iOS/Android) 的连接和状态。

**核心文件**: `src/gateway/node-registry.ts`

### 3.4 WebSocket 运行时

处理客户端的 WebSocket 连接和消息协议。

```typescript
// server-ws-runtime.ts
export function attachGatewayWsHandlers(params: GatewayWsRuntimeParams) {
  attachGatewayWsConnectionHandler({
    wss: params.wss,
    clients: params.clients,
    // ...
  });
}
```

## 4. 通信协议

### 4.1 协议框架

Gateway 使用基于 JSON 的请求/响应帧协议：

```typescript
// protocol/index.ts
export type RequestFrame = {
  id: string;
  method: string;
  params?: Record<string, unknown>;
};

export type ResponseFrame = {
  id: string;
  ok: boolean;
  payload?: unknown;
  error?: ErrorShape;
};

export type EventFrame = {
  event: string;
  payload?: unknown;
};
```

### 4.2 协议版本

```typescript
export const PROTOCOL_VERSION = "2026.2.15";
```

### 4.3 连接参数

```typescript
export type ConnectParams = {
  protocolVersion: string;
  clientType: "desktop" | "mobile" | "web";
  clientVersion?: string;
  authToken?: string;
  // ...
};
```

## 5. 认证与授权

### 5.1 认证模式

Gateway 支持多种认证模式：

```typescript
// auth.ts
export type GatewayAuthConfig = {
  mode: "none" | "token" | "token-or-anonymous" | "passcode";
  token?: string;
  passcode?: string;
  allowedOrigins?: string[];
  tailscale?: GatewayTailscaleConfig;
};
```

### 5.2 速率限制

```typescript
// auth-rate-limit.ts
export function createAuthRateLimiter(config: AuthRateLimitConfig) {
  // 基于 IP 的请求限流
}
```

## 6. 启动流程

```typescript
// server.impl.ts
export async function startGatewayServer(
  port = 18789,
  opts: GatewayServerOptions = {},
): Promise<GatewayServer> {
  // 1. 加载配置
  let configSnapshot = await readConfigFileSnapshot();
  
  // 2. 迁移旧配置
  const { config: migrated } = migrateLegacyConfig(configSnapshot.parsed);
  
  // 3. 激活密钥
  const prepared = await prepareSecretsRuntimeSnapshot({ config });
  activateSecretsRuntimeSnapshot(prepared);
  
  // 4. 创建 WebSocket 服务器
  const wss = new WebSocketServer({ port });
  
  // 5. 初始化通道
  const channelManager = createChannelManager({ ... });
  
  // 6. 启动定时任务
  const cron = buildGatewayCronService({ ... });
  
  // 7. 附加处理器
  attachGatewayWsHandlers({ wss, clients, ... });
  
  return { close };
}
```

## 7. 核心功能

### 7.1 会话管理

```typescript
// sessions.ts
export type GatewayRequestHandlers = {
  "sessions.list": (opts) => Promise<SessionsListResult>;
  "sessions.patch": (opts) => Promise<SessionsPatchResult>;
  "sessions.delete": (opts) => Promise<void>;
  "sessions.compact": (opts) => Promise<void>;
};
```

### 7.2 消息路由

```
Inbound Message (Channel)
    │
    ▼
Channel Manager
    │
    ▼
Session Key Resolver
    │
    ├──▶ Main Session (direct chat)
    │
    ├──▶ Group Session (group chat)
    │
    └──▶ Agent Session (AI routing)
```

### 7.3 定时任务 (Cron)

```typescript
// server-cron.ts
export function buildGatewayCronService(params: GatewayCronServiceParams): CronService {
  return {
    add: (job) => { ... },
    remove: (id) => { ... },
    list: () => { ... },
    run: (id) => { ... },
  };
}
```

### 7.4 配置热重载

```typescript
// config-reload.ts
export function startGatewayConfigReloader(params: ConfigReloaderParams) {
  // 监视配置文件变化
  // 平滑重载配置
}
```

### 7.5 健康检查

```typescript
// health-state.ts
export function getHealthCache(): HealthSummary | null;
export function refreshGatewayHealthSnapshot(opts?: { probe?: boolean }): Promise<HealthSummary>;
```

## 8. 安全特性

### 8.1 密钥管理

```typescript
// secrets/runtime.ts
export async function prepareSecretsRuntimeSnapshot(config: OpenClawConfig) {
  // 从环境变量和配置文件加载密钥
}
```

### 8.2 执行审批

对于敏感操作（如执行外部命令），Gateway 支持审批流程：

```typescript
// exec-approval-manager.ts
export class ExecApprovalManager {
  request: (params) => Promise<void>;
  resolve: (params) => Promise<void>;
}
```

### 8.3 Tailscale 支持

```typescript
// server-tailscale.ts
export function startGatewayTailscaleExposure(config: GatewayTailscaleConfig) {
  // 通过 Tailscale 网络暴露 Gateway
}
```

## 9. 广播系统

Gateway 支持向所有客户端或特定客户端广播事件：

```typescript
// server-broadcast.ts
broadcast: (
  event: string,
  payload: unknown,
  opts?: { dropIfSlow?: boolean },
) => void;

broadcastToConnIds: (
  connIds: string[],
  event: string,
  payload: unknown,
) => void;
```

## 10. 维护与监控

### 10.1 通道健康监控

```typescript
// channel-health-monitor.ts
export function startChannelHealthMonitor(params: ChannelHealthMonitorParams) {
  // 定期检查通道状态
  // 自动重启故障通道
}
```

### 10.2 日志管理

```typescript
// ws-log.ts
export function createWsLog(params: WsLogParams) {
  // 记录 WebSocket 消息
}
```

## 11. 核心 API

### 11.1 启动 Gateway

```typescript
import { startGatewayServer } from "./gateway/server.js";

const gateway = await startGatewayServer(18789, {
  bind: "loopback",
  controlUiEnabled: true,
  auth: { mode: "token", token: "secret" }
});

await gateway.close({ reason: "restart" });
```

### 11.2 客户端连接

```typescript
const ws = new WebSocket("ws://localhost:18789");

ws.onopen = () => {
  ws.send(JSON.stringify({
    id: "1",
    method: "hello",
    params: { protocolVersion: "2026.2.15", clientType: "desktop" }
  }));
};

ws.onmessage = (event) => {
  const frame = JSON.parse(event.data);
  // 处理响应
};
```

## 12. 文件结构

| 目录/文件 | 功能 |
|----------|------|
| `server.impl.ts` | Gateway 启动和核心逻辑 |
| `server-methods/` | 请求处理器 (60+ 方法) |
| `server-channels.ts` | 通道生命周期管理 |
| `server-ws-runtime.ts` | WebSocket 连接处理 |
| `protocol/` | 协议定义和验证 |
| `auth.ts` | 认证逻辑 |
| `config-reload.ts` | 配置热重载 |
| `server-cron.ts` | 定时任务服务 |
| `node-registry.ts` | 移动节点管理 |
| `channel-health-monitor.ts` | 通道健康监控 |

## 13. 设计亮点

1. **双协议支持**: WebSocket + HTTP (Control UI)
2. **方法驱动**: 基于 JSON-RPC 风格的方法调用
3. **平滑重载**: 配置变更无需重启
4. **自动恢复**: 通道故障自动重启
5. **速率限制**: 防止恶意请求
6. **密钥分离**: 运行时密钥快照
7. **移动节点**: 支持 iOS/Android 客户端
8. **Tailscale**: 支持内网穿透

## 14. 总结

Gateway 模块是 OpenClaw 系统的核心控制平面，其设计特点包括：

1. **统一入口**: 所有客户端通过 Gateway 访问系统
2. **方法驱动**: 清晰的方法分类和请求处理
3. **状态管理**: 完整的会话、通道、节点生命周期
4. **安全认证**: 多种认证模式和速率限制
5. **可观测性**: 健康检查、日志、监控
6. **高可用**: 自动恢复、配置热重载、平滑重启
