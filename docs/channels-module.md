# OpenClaw Channels 模块技术原理详解

## 1. 模块概述

Channels 模块是 OpenClaw 的消息通道抽象层，负责统一管理所有消息平台的接入。该模块定义了通道接口规范，支持内置通道和插件扩展通道。

**核心位置**: `src/channels/` (66 个文件 + `plugins/` 子目录)

## 2. 架构设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Channel Layer                                  │
├─────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    Channel Registry                           │  │
│  │              (src/channels/registry.ts)                      │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│         ┌──────────────────┼──────────────────┐                  │
│         ▼                  ▼                  ▼                  │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐          │
│  │  Built-in   │    │   Plugin   │    │   Custom    │          │
│  │  Channels   │    │  Channels   │    │  Extensions │          │
│  └─────────────┘    └─────────────┘    └─────────────┘          │
├─────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    Channel Adapters                            │  │
│  │  Messaging | Security | Groups | Outbound | Status | etc.    │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

## 3. 核心类型定义

### 3.1 通道 ID

```typescript
// types.core.ts
export type ChannelId = ChatChannelId | (string & {});
```

### 3.2 通道能力

```typescript
// types.core.ts
export type ChannelCapabilities = {
  chatTypes: Array<ChatType | "thread">;
  polls?: boolean;
  reactions?: boolean;
  edit?: boolean;
  unsend?: boolean;
  reply?: boolean;
  effects?: boolean;
  groupManagement?: boolean;
  threads?: boolean;
  media?: boolean;
  nativeCommands?: boolean;
  blockStreaming?: boolean;
};
```

### 3.3 通道插件接口

```typescript
// types.plugin.ts
export type ChannelPlugin<ResolvedAccount, Probe, Audit> = {
  id: ChannelId;
  meta: ChannelMeta;
  capabilities: ChannelCapabilities;
  
  // 核心适配器
  config: ChannelConfigAdapter;
  setup?: ChannelSetupAdapter;
  security?: ChannelSecurityAdapter;
  groups?: ChannelGroupAdapter;
  outbound?: ChannelOutboundAdapter;
  status?: ChannelStatusAdapter;
  messaging?: ChannelMessagingAdapter;
  streaming?: ChannelStreamingAdapter;
  threading?: ChannelThreadingAdapter;
  
  // 辅助适配器
  pairing?: ChannelPairingAdapter;
  mentions?: ChannelMentionAdapter;
  auth?: ChannelAuthAdapter;
  directory?: ChannelDirectoryAdapter;
  actions?: ChannelMessageActionAdapter;
  heartbeat?: ChannelHeartbeatAdapter;
  
  // 工具
  agentTools?: ChannelAgentToolFactory | ChannelAgentTool[];
};
```

## 4. 通道适配器

### 4.1 消息适配器 (Messaging Adapter)

```typescript
// types.adapters.ts
export type ChannelMessagingAdapter = {
  // 接收消息
  start: (params: {
    cfg: OpenClawConfig;
    accountId: string;
    ctx: ChannelMessagingContext;
  }) => Promise<() => Promise<void>>;
};

export type ChannelMessagingContext = {
  // 消息接收回调
  onInbound: (message: InboundMessage) => Promise<void>;
  // 发送回复
  sendReply: (target: MessagingReplyTarget, message: OutboundMessage) => Promise<void>;
};
```

### 4.2 安全适配器

```typescript
// types.core.ts
export type ChannelSecurityDmPolicy = {
  policy: string;  // "open" | "pairing" | "deny"
  allowFrom?: Array<string | number> | null;
};
```

### 4.3 出站适配器

```typescript
// types.adapters.ts
export type ChannelOutboundAdapter = {
  send: (params: {
    target: ChannelOutboundTarget;
    message: OutboundMessage;
  }) => Promise<ChannelOutboundResult>;
};
```

### 4.4 状态适配器

```typescript
// types.adapters.ts
export type ChannelStatusAdapter = {
  status: () => Promise<ChannelAccountState>;
  snapshot: () => Promise<ChannelAccountSnapshot>;
  probe?: () => Promise<BaseProbeResult>;
  audit?: () => Promise<Audit>;
};
```

## 5. 支持的通道

### 5.1 内置通道

| 通道 | 目录 | 框架 |
|------|------|------|
| Telegram | `src/telegram/` | GramJS |
| Discord | `src/discord/` | discord.js |
| Slack | `src/slack/` | @slack/bolt |
| Signal | `src/signal/` | libsignal |
| WhatsApp | `src/whatsapp/` | Baileys |
| iMessage | `src/imessage/` | BlueBubbles API |
| LINE | `src/line/` | @line/bot-sdk |
| Web | `src/web/` | WebSocket |

### 5.2 插件通道 (extensions/)

| 通道 | 包 |
|------|-----|
| Microsoft Teams | `@openclaw/msteams` |
| Matrix | `@openclaw/matrix` |
| Feishu | `@openclaw/feishu` |
| IRC | `@openclaw/irc` |
| Nostr | `@openclaw/nostr` |
| Twitch | `@openclaw/twitch` |
| Zalo | `@openclaw/zalo` |
| Mattermost | `@openclaw/mattermost` |

### 5.3 通道元数据

```typescript
// types.core.ts
export type ChannelMeta = {
  id: ChannelId;
  label: string;
  selectionLabel: string;
  docsPath: string;
  blurb: string;
  aliases?: string[];
  systemImage?: string;
  showConfigured?: boolean;
  forceAccountBinding?: boolean;
};
```

## 6. 核心功能

### 6.1 会话管理

```typescript
// session.ts
export type Session = {
  key: string;
  channelId: ChannelId;
  accountId: string;
  chatType: ChatType;
  threadId?: string;
  participants: string[];
};
```

### 6.2 消息路由

```
Inbound Message
    │
    ▼
┌─────────────────┐
│   Security      │ ─── 允许列表检查
│   (allowFrom)  │
└─────────────────┘
    │
    ▼
┌─────────────────┐
│  Message Type   │ ─── 过滤
│  Detection      │
└─────────────────┘
    │
    ▼
┌─────────────────┐
│  Thread        │ ─── 话题绑定
│  Binding       │
└─────────────────┘
    │
    ▼
┌─────────────────┐
│    Queue       │ ─── 消息排队
│    Debounce    │
└─────────────────┘
    │
    ▼
   AI Agent
```

### 6.3 消息队列

```typescript
// draft-stream-loop.ts
export type DraftStreamLoop = {
  start: (sessionKey: string) => void;
  stop: (sessionKey: string) => void;
  flush: (sessionKey: string) => Promise<void>;
};
```

### 6.4 打字指示器

```typescript
// typing.ts
export type TypingIndicator = {
  start: (target: MessagingReplyTarget) => Promise<void>;
  stop: (target: MessagingReplyTarget) => Promise<void>;
};
```

### 6.5 已读回执

```typescript
// ack-reactions.ts
export type AckReaction = "sent" | "delivered" | "read" | "played";
```

## 7. 消息处理

### 7.1 入站消息

```typescript
export type InboundMessage = {
  channelId: ChannelId;
  accountId: string;
  senderId: string;
  senderName?: string;
  content: MessageContent;
  timestamp: number;
  threadId?: string;
  replyTo?: string;
  attachments?: Attachment[];
};
```

### 7.2 出站消息

```typescript
export type OutboundMessage = {
  content: MessageContent;
  mentions?: string[];
  replyTo?: string;
  attachments?: Attachment[];
};
```

### 7.3 消息动作

```typescript
// message-actions.ts
export type ChannelMessageActionAdapter = {
  // 支持的消息动作
  send: (message: OutboundMessage) => Promise<void>;
  edit: (messageId: string, message: OutboundMessage) => Promise<void>;
  delete: (messageId: string) => Promise<void>;
  react: (messageId: string, reaction: string) => Promise<void>;
};
```

## 8. 安全机制

### 8.1 DM 策略

```typescript
// types.core.ts
type DmPolicy = 
  | "open"      // 开放接受
  | "pairing"  // 需要配对
  | "deny";    // 拒绝
```

### 8.2 允许列表

```typescript
// allow-from.ts
export type AllowFromConfig = {
  allowFrom?: string[];
  denyFrom?: string[];
};
```

### 8.3 提及限制

```typescript
// mention-gating.ts
export type MentionGatingConfig = {
  required?: string[];
  blocked?: string[];
};
```

## 9. 配置管理

### 9.1 通道配置

```typescript
// channel-config.ts
export type ChannelConfig = {
  enabled: boolean;
  accountId: string;
  dmPolicy?: DmPolicy;
  allowFrom?: string[];
  modelOverrides?: Record<string, string>;
};
```

### 9.2 运行时配置

```typescript
// plugins/config-helpers.ts
export function resolveChannelAccountConfig(
  cfg: OpenClawConfig,
  channelId: ChannelId,
  accountId: string
): ResolvedChannelConfig;
```

## 10. 文件结构

| 文件 | 功能 |
|------|------|
| `registry.ts` | 通道注册表 |
| `session.ts` | 会话管理 |
| `targets.ts` | 消息目标解析 |
| `typing.ts` | 打字指示器 |
| `ack-reactions.ts` | 已读回执 |
| `dock.ts` | 通道对接 |
| `thread-bindings-policy.ts` | 话题绑定策略 |
| `draft-stream-controls.ts` | 草稿流控制 |
| `model-overrides.ts` | 模型覆盖 |
| `plugins/types.core.ts` | 核心类型 |
| `plugins/types.plugin.ts` | 插件接口 |
| `plugins/types.adapters.ts` | 适配器接口 |

## 11. 设计亮点

1. **统一接口**: 抽象出适配器模式，兼容所有消息平台
2. **插件化**: 支持外部插件扩展新通道
3. **安全优先**: DM 策略、配对机制、允许列表
4. **能力声明**: 每个通道声明自己的能力
5. **会话隔离**: 支持分组会话隔离
6. **消息队列**: 入站消息去重和防抖
7. **实时反馈**: 打字指示器、已读回执

## 12. 总结

Channels 模块是 OpenClaw 实现多平台消息统一管理的核心，其设计特点包括：

1. **适配器模式**: 统一的通道接口，适配不同平台
2. **插件系统**: 支持 npm 包形式扩展
3. **安全策略**: DM 配对、允许列表、提及限制
4. **会话管理**: 支持多会话、分组、话题
5. **消息队列**: 去重、防抖、实时处理
6. **状态监控**: 健康检查、审计日志
