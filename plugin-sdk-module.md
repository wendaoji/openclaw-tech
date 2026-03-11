# OpenClaw Plugin-SDK 模块技术原理详解

## 1. 模块概述

Plugin-SDK 是 OpenClaw 的插件开发工具包，提供创建通道插件、技能插件和上下文引擎所需的 API。它采用适配器模式，允许开发者扩展 OpenClaw 功能。

**核心位置**: `src/plugin-sdk/` (114 个文件)

## 2. 架构设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Plugin SDK Layer                                │
├─────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    Plugin APIs                                │   │
│  │  core.ts | telegram.ts | discord.ts | slack.ts | whatsapp.ts │   │
│  └──────────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    Plugin Runtime                              │   │
│  │  runtime.ts | runtime/ | services.ts                         │   │
│  └──────────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    Security & Auth                            │   │
│  │  ssrf-policy.ts | webhook-guards.ts | auth.ts               │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

## 3. 核心 API

### 3.1 核心接口

```typescript
// core.ts
export type OpenClawPluginApi = {
  // 工具工厂
  tools: OpenClawPluginToolFactory[];
  
  // 钩子
  hooks?: HookEntry[];
  
  // 配置
  configSchema?: OpenClawPluginConfigSchema;
};

export type OpenClawPluginService = {
  // 服务名称
  name: string;
  // 启动
  start: () => Promise<void>;
  // 停止
  stop: () => Promise<void>;
};
```

### 3.2 工具上下文

```typescript
// types.ts
export type OpenClawPluginToolContext = {
  config?: OpenClawConfig;
  workspaceDir?: string;
  agentDir?: string;
  agentId?: string;
  sessionKey?: string;
  sessionId?: string;
  messageChannel?: string;
  agentAccountId?: string;
  requesterSenderId?: string;
  senderIsOwner?: boolean;
  sandboxed?: boolean;
};
```

### 3.3 工具工厂

```typescript
export type OpenClawPluginToolFactory = (
  ctx: OpenClawPluginToolContext,
) => AnyAgentTool | AnyAgentTool[] | null | undefined;
```

## 4. 认证与安全

### 4.1 认证类型

```typescript
export type ProviderAuthKind = 
  | "oauth" 
  | "api_key" 
  | "token" 
  | "device_code" 
  | "custom";
```

### 4.2 OAuth 处理

```typescript
// provider-auth-result.ts
export function buildOauthProviderAuthResult(
  provider: string,
  auth: OAuthCredential
): ProviderAuthResult;
```

### 4.3 SSRF 防护

```typescript
// ssrf-policy.ts
export type SSRFPolicy = {
  allowedIPs?: string[];
  blockedIPs?: string[];
  allowedHosts?: string[];
  blockedHosts?: string[];
};
```

### 4.4 Webhook 守卫

```typescript
// webhook-request-guards.ts
export function validateWebhookRequest(
  req: IncomingMessage,
  secret: string,
  policy: WebhookPolicy
): boolean;
```

## 5. 通道插件

### 5.1 Telegram

```typescript
// telegram.ts
export type TelegramPluginConfig = {
  botToken: string;
  allowedUpdates?: string[];
  pollingTimeout?: number;
};
```

### 5.2 Discord

```typescript
// discord.ts
export type DiscordPluginConfig = {
  botToken: string;
  intents: number[];
  guildId?: string;
};
```

### 5.3 Slack

```typescript
// slack.ts
export type SlackPluginConfig = {
  botToken: string;
  signingSecret: string;
  appToken?: string;
};
```

### 5.4 WhatsApp

```typescript
// whatsapp.ts
export type WhatsAppPluginConfig = {
  phoneNumber: string;
  authDir: string;
  qrTimeout?: number;
};
```

## 6. 运行时服务

### 6.1 插件运行时

```typescript
// runtime.ts
export type PluginRuntime = {
  // 工具执行
  tools: PluginToolsRuntime;
  
  // 配置
  config: PluginConfigRuntime;
  
  // 日志
  logger: PluginLogger;
  
  // 存储
  store: PluginStore;
};
```

### 6.2 存储

```typescript
// runtime-store.ts
export type PluginStore = {
  get: (key: string) => Promise<unknown>;
  set: (key: string, value: unknown) => Promise<void>;
  delete: (key: string) => Promise<void>;
};
```

## 7. 消息处理

### 7.1 回复载荷

```typescript
// reply-payload.ts
export type ReplyPayload = {
  text: string;
  mentions?: string[];
  buttons?: Button[];
  replyTo?: string;
};
```

### 7.2 通道发送结果

```typescript
// channel-send-result.ts
export type ChannelSendResult = {
  ok: boolean;
  messageId?: string;
  error?: string;
};
```

## 8. Webhook

### 8.1 Webhook 目标

```typescript
// webhook-targets.ts
export type WebhookTarget = {
  url: string;
  method: "GET" | "POST" | "PUT" | "DELETE";
  headers?: Record<string, string>;
  body?: string;
};
```

### 8.2 内存守卫

```typescript
// webhook-memory-guards.ts
export type WebhookMemoryGuard = {
  maxBodySize?: number;
  maxHeaders?: number;
  timeout?: number;
};
```

## 9. 工具函数

### 9.1 命令执行

```typescript
// run-command.ts
export async function runPluginCommandWithTimeout(
  command: string,
  args: string[],
  options: PluginCommandRunOptions
): Promise<PluginCommandRunResult>;
```

### 9.2 临时文件

```typescript
// temp-path.ts
export function resolvePreferredOpenClawTmpDir(): string;
export function createTempFile(prefix: string): string;
```

### 9.3 文本分块

```typescript
// text-chunking.ts
export function chunkText(text: string, maxChunkSize: number): string[];
```

## 10. 文件结构

| 文件 | 功能 |
|------|------|
| `core.ts` | 核心 API 导出 |
| `telegram.ts` | Telegram 插件 API |
| `discord.ts` | Discord 插件 API |
| `slack.ts` | Slack 插件 API |
| `whatsapp.ts` | WhatsApp 插件 API |
| `signal.ts` | Signal 插件 API |
| `runtime.ts` | 运行时接口 |
| `ssrf-policy.ts` | SSRF 防护 |
| `webhook-*.ts` | Webhook 相关 |
| `reply-payload.ts` | 回复载荷 |
| `allow-from.ts` | 允许列表 |
| `persistent-dedupe.ts` | 消息去重 |

## 11. 插件类型

| 类型 | 用途 |
|------|------|
| `channel` | 消息通道扩展 |
| `skill` | 技能扩展 |
| `memory` | 记忆后端 |
| `context-engine` | 上下文引擎 |

## 12. 总结

Plugin-SDK 模块是 OpenClaw 的扩展核心，其设计特点包括：

1. **统一接口**: 统一的插件开发 API
2. **通道适配**: 预置的 Telegram/Discord/Slack/WhatsApp 适配器
3. **安全防护**: SSRF、Webhook 守卫
4. **运行时**: 完整的工具执行和存储服务
5. **类型安全**: TypeScript 严格类型
6. **npm 发布**: 支持独立 npm 包发布
