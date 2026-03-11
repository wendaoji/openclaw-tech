# OpenClaw 技术完全解析

**OpenClaw** 是一个运行在用户本地设备上的多通道 AI 网关，能够在 WhatsApp、Telegram、Slack、Discord、Signal、iMessage 等 20+ 消息平台上提供统一的 AI 助手服务。

---

# 目录

1. [系统概览](#系统概览)
2. [核心技术栈](#核心技术栈)
3. [项目结构](#项目结构)
4. [Gateway 模块](#gateway-模块)
5. [Channels 模块](#channels-模块)
6. [Memory 模块](#memory-模块)
7. [Agents 模块](#agents-模块)
8. [Plugin-SDK 模块](#plugin-sdk-模块)
9. [Config 模块](#config-模块)
10. [Infra 模块](#infra-模块)
11. [Media 模块](#media-模块)
12. [CLI/Commands 模块](#clicommands-模块)
13. [Web/TUI 模块](#webtui-模块)
14. [Providers 模块](#providers-模块)
15. [架构特点](#架构特点)
16. [安全设计](#安全设计)
17. [快速开始](#快速开始)

---

# 系统概览

## 什么是 OpenClaw？

OpenClaw 是一个**个人 AI 助手**，它不是云服务，而是运行在你自己的设备上（Mac、Windows、Linux、Raspberry Pi）。你可以通过多种渠道与它对话，它会在所有渠道上回复你。

## 核心特性

| 特性 | 说明 |
|------|------|
| 多通道 | 支持 20+ 消息平台 |
| 本地运行 | 数据存储在本地，保护隐私 |
| AI 驱动 | 基于 Pi Agent Core |
| 可扩展 | 35+ 官方插件 |
| 开源 | MIT 许可证 |

---

# 核心技术栈

| 类别 | 技术 |
|------|------|
| 运行时 | Node.js 22+ (ESM) |
| 包管理 | pnpm (workspace) |
| 语言 | TypeScript (严格模式) |
| 构建工具 | tsdown |
| 测试框架 | Vitest |
| 代码检查 | Oxlint, Oxfmt |

---

# 项目结构

```
openclaw/
├── src/                    # 核心源代码
│   ├── gateway/           # Gateway WebSocket 控制平面
│   ├── channels/          # 通道抽象层
│   ├── agents/           # Agent 运行时
│   ├── memory/           # 记忆/向量存储
│   ├── plugin-sdk/       # 插件 SDK
│   ├── config/           # 配置管理
│   ├── infra/           # 基础设施
│   ├── media/           # 媒体处理管道
│   ├── cli/             # CLI 入口
│   ├── commands/        # CLI 命令
│   ├── web/             # Web UI
│   ├── tui/             # 终端 UI
│   ├── telegram/        # Telegram 集成
│   ├── discord/         # Discord 集成
│   ├── slack/           # Slack 集成
│   ├── signal/          # Signal 集成
│   └── whatsapp/        # WhatsApp 集成
├── apps/                # 客户端应用
│   ├── macos/          # macOS 应用 (Swift/SwiftUI)
│   ├── ios/            # iOS 应用 (Swift/SwiftUI)
│   ├── android/        # Android 应用 (Kotlin)
│   └── shared/         # 共享代码
├── extensions/          # 插件扩展 (35+)
└── docs/               # 文档
```

---

# Gateway 模块

## 1. 模块概述

Gateway 是 OpenClaw 的核心控制平面，通过 WebSocket 协议管理所有客户端连接、会话、通道和消息路由。它是整个系统的中枢，负责协调各个模块的运行。

**核心位置**: `src/gateway/` (244 个文件)

## 2. 架构设计

```
+---------------------------------------------------------------------+
|                        Gateway Server                                |
|                     (server.impl.ts)                                |
+---------------------------------------------------------------------+
|  +-----------+  +-----------+  +-----------+  +----------+          |
|  |  Config   |  |   Auth    |  |  Secrets  |  | Plugins  |          |
|  |  Manager  |  |  Manager  |  |  Manager  |  | Manager  |          |
|  +-----------+  +-----------+  +-----------+  +----------+          |
+---------------------------------------------------------------------+
|  +--------------------------------------------------------------+   |
|  |                    Server Methods                             |   |
|  |  agent | chat | sessions | channels | cron | tools | config  |   |
|  +--------------------------------------------------------------+   |
+---------------------------------------------------------------------+
|  +-----------+  +-----------+  +-----------+                       |
|  |  Channel  |  |   Node    |  |  Canvas   |                       |
|  |  Manager  |  | Registry  |  |   Host    |                       |
|  +-----------+  +-----------+  +-----------+                       |
+---------------------------------------------------------------------+
|  +--------------------------------------------------------------+   |
|  |                 WebSocket Server                             |   |
|  |     ws://gateway-host:18789 (客户端连接)                   |   |
|  +--------------------------------------------------------------+   |
+---------------------------------------------------------------------+
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

### 3.2 通道管理 (Channel Manager)

负责消息通道的启动、停止和生命周期管理。

```typescript
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

## 4. 通信协议

### 4.1 协议框架

Gateway 使用基于 JSON 的请求/响应帧协议：

```typescript
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

## 5. 认证与授权

### 5.1 认证模式

Gateway 支持多种认证模式：

```typescript
export type GatewayAuthConfig = {
  mode: "none" | "token" | "token-or-anonymous" | "passcode";
  token?: string;
  passcode?: string;
  allowedOrigins?: string[];
  tailscale?: GatewayTailscaleConfig;
};
```

## 6. 启动流程

```typescript
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
- 列表/补丁/删除/压缩

### 7.2 消息路由
```
Inbound Message (Channel)
    |
    v
Channel Manager
    |
    v
Session Key Resolver
    |
    +---> Main Session (direct chat)
    +---> Group Session (group chat)
    +---> Agent Session (AI routing)
```

### 7.3 定时任务 (Cron)
```typescript
export function buildGatewayCronService(params: GatewayCronServiceParams): CronService {
  return {
    add: (job) => { ... },
    remove: (id) => { ... },
    list: () => { ... },
    run: (id) => { ... },
  };
}
```

## 8. 安全特性

### 8.1 密钥管理
```typescript
export async function prepareSecretsRuntimeSnapshot(config: OpenClawConfig) {
  // 从环境变量和配置文件加载密钥
}
```

### 8.2 执行审批
对于敏感操作（如执行外部命令），Gateway 支持审批流程：
```typescript
export class ExecApprovalManager {
  request: (params) => Promise<void>;
  resolve: (params) => Promise<void>;
}
```

### 8.3 Tailscale 支持
```typescript
export function startGatewayTailscaleExposure(config: GatewayTailscaleConfig) {
  // 通过 Tailscale 网络暴露 Gateway
}
```

## 9. 设计亮点

1. **双协议支持**: WebSocket + HTTP (Control UI)
2. **方法驱动**: 基于 JSON-RPC 风格的方法调用
3. **平滑重载**: 配置变更无需重启
4. **自动恢复**: 通道故障自动重启
5. **速率限制**: 防止恶意请求
6. **密钥分离**: 运行时密钥快照
7. **移动节点**: 支持 iOS/Android 客户端

---

# Channels 模块

## 1. 模块概述

Channels 模块是 OpenClaw 的消息通道抽象层，负责统一管理所有消息平台的接入。该模块定义了通道接口规范，支持内置通道和插件扩展通道。

**核心位置**: `src/channels/` (66 个文件 + plugins 子目录)

## 2. 核心组件

### 2.1 通道插件接口

```typescript
export type ChannelPlugin<ResolvedAccount, Probe, Audit> = {
  id: ChannelId;
  meta: ChannelMeta;
  capabilities: ChannelCapabilities;
  config: ChannelConfigAdapter;
  setup?: ChannelSetupAdapter;
  security?: ChannelSecurityAdapter;
  groups?: ChannelGroupAdapter;
  mentions?: ChannelMentionAdapter;
  outbound?: ChannelOutboundAdapter;
  status?: ChannelStatusAdapter;
  messaging?: ChannelMessagingAdapter;
  streaming?: ChannelStreamingAdapter;
  threading?: ChannelThreadingAdapter;
  pairing?: ChannelPairingAdapter;
  directory?: ChannelDirectoryAdapter;
  actions?: ChannelMessageActionAdapter;
  heartbeat?: ChannelHeartbeatAdapter;
  agentTools?: ChannelAgentToolFactory | ChannelAgentTool[];
};
```

### 2.2 通道能力

```typescript
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

## 3. 支持的通道

### 3.1 内置通道

| 通道 | 框架 | 位置 |
|------|------|------|
| Telegram | GramJS | src/telegram/ |
| Discord | discord.js | src/discord/ |
| Slack | @slack/bolt | src/slack/ |
| Signal | libsignal | src/signal/ |
| WhatsApp | Baileys | src/whatsapp/ |
| iMessage | BlueBubbles | src/imessage/ |
| LINE | @line/bot-sdk | src/line/ |
| Web | WebSocket | src/web/ |

### 3.2 扩展通道 (35+)

Matrix, Microsoft Teams, Feishu, IRC, Nostr, Twitch, Zalo, BlueBubbles, Voice Call 等

## 4. 安全机制

### 4.1 DM 策略

```typescript
type DmPolicy = "open" | "pairing" | "deny";
```

### 4.2 允许列表

```typescript
export type AllowFromConfig = {
  allowFrom?: string[];
  denyFrom?: string[];
};
```

### 4.3 提及限制

```typescript
export type MentionGatingConfig = {
  required?: string[];
  blocked?: string[];
};
```

## 5. 消息处理

### 5.1 入站消息

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

### 5.2 出站消息

```typescript
export type OutboundMessage = {
  content: MessageContent;
  mentions?: string[];
  replyTo?: string;
  attachments?: Attachment[];
};
```

## 6. 设计亮点

1. **统一接口**: 抽象出适配器模式，兼容所有消息平台
2. **插件化**: 支持外部插件扩展新通道
3. **安全优先**: DM 策略、配对机制、允许列表
4. **能力声明**: 每个通道声明自己的能力
5. **会话隔离**: 支持分组会话隔离

---

# Memory 模块

## 1. 模块概述

Memory 模块是 OpenClaw 的核心记忆系统，负责持久化和检索用户的对话上下文和知识库。该模块采用混合搜索策略，结合向量嵌入和全文搜索，提供高效的语义检索能力。

**核心位置**: `src/memory/` (98 个源文件)

## 2. 核心组件

| 组件 | 文件 | 功能 |
|------|------|------|
| MemoryIndexManager | `manager.ts` | 核心索引管理器 |
| Embedding Providers | `embeddings*.ts` | 多提供商嵌入支持 |
| Hybrid Search | `hybrid.ts` | 向量+关键词混合搜索 |
| Search Manager | `search-manager.ts` | 搜索抽象层 |
| QMD Manager | `qmd-manager.ts` | QMD 格式管理 |
| MMR Ranking | `mmr.ts` | 多样性重排序 |
| Temporal Decay | `temporal-decay.ts` | 时间衰减 |
| Query Expansion | `query-expansion.ts` | 查询扩展 |

## 3. 架构设计

```
+-------------------------------------------------------------------+
|                     MemorySearchManager                            |
|                   (search-manager.ts)                              |
+-------------------------------------------------------------------+
|  +-----------------+    +---------------------------+           |
|  |  QMD Backend   |    |    Builtin Backend       |           |
|  | (qmd-manager)  |--->|  (MemoryIndexManager)    |           |
|  +-----------------+    +---------------------------+           |
|                                     |                             |
|                                     v                             |
|                         +---------------------------+             |
|                         |   Hybrid Search Engine  |             |
|                         +---------------------------+             |
|                         |  +---------+ +---------+ |             |
|                         |  | Vector  | |   FTS   | |             |
|                         |  | Search  | | Search  | |             |
|                         |  +---------+ +---------+ |             |
|                         |        |        |       |             |
|                         |        v        v       |             |
|                         |   MMR + Temporal Decay |             |
|                         +---------------------------+             |
+-------------------------------------------------------------------+
```

## 4. 嵌入向量支持

### 4.1 支持的嵌入提供商

| 提供商 | 模型 | 配置键 |
|--------|------|--------|
| OpenAI | text-embedding-3-small, ada-002 | `openai` |
| Google Gemini | gemini-embedding-001 | `gemini` |
| Voyage AI | voyage-large-2, voyage-code-2 | `voyage` |
| Mistral | mistral-embed | `mistral` |
| Ollama | 本地模型 | `ollama` |
| Local (node-llama-cpp) | embeddinggemma-300m | `local` |

### 4.2 嵌入创建流程

```typescript
export async function createEmbeddingProvider(options) {
  // 自动选择提供商
  if (options.provider === "auto") {
    // 尝试 OpenAI -> Gemini -> Voyage -> Mistral
  }
  
  // 回退机制
  if (primaryProvider失败) {
    return fallbackProvider;
  }
}
```

## 5. 混合搜索机制

### 5.1 搜索策略

`hybrid.ts` 实现了混合搜索，结合向量相似度和 BM25 关键词匹配：

```typescript
// 混合搜索核心逻辑
score = vectorWeight * vectorScore + textWeight * textScore;
```

### 5.2 查询扩展

`query-expansion.ts` 实现智能关键词提取：

- 移除停用词 (英语、中文、西班牙语等)
- 提取有意义的关键词
- 支持多语言

## 6. 搜索结果排序

### 6.1 MMR (最大边际相关性)

`mmr.ts` 实现结果多样性排序：

```typescript
// MMR 公式
MMR = λ * relevance - (1-λ) * max_similarity_to_selected

// 参数
lambda: 0-1 (默认 0.7)
- λ = 1: 完全按相关性排序
- λ = 0: 完全按多样性排序
```

### 6.2 时间衰减

`temporal-decay.ts` 实现近因偏好：

```typescript
// 半衰期衰减
score = originalScore * Math.exp(-λ * ageInDays);
λ = Math.LN2 / halfLifeDays;  // 默认 30 天半衰期
```

**特殊处理**:
- `memory/MEMORY.md`: 永不过期
- `memory/YYYY-MM-DD.md`: 从文件名解析日期

## 7. 数据存储

### 7.1 SQLite 表结构

```sql
-- 向量表
CREATE VIRTUAL TABLE chunks_vec USING vec0(
  id TEXT PRIMARY KEY,
  path TEXT,
  content TEXT,
  embedding FLOAT[dimension]
);

-- FTS 表
CREATE VIRTUAL TABLE chunks_fts USING fts5(
  id, path, content
);

-- 嵌入缓存
CREATE TABLE embedding_cache (
  hash TEXT PRIMARY KEY,
  embedding BLOB,
  created_at INTEGER
);
```

## 8. 设计亮点

1. **双后端支持**: Builtin + QMD，自动降级
2. **混合搜索**: 向量权重 + 关键词权重组合评分
3. **6 种嵌入提供商**: OpenAI/Gemini/Voyage/Mistral/Ollama/Local
4. **自动回退**: 主提供商失败自动切换
5. **嵌入缓存**: 避免重复计算

---

# Agents 模块

## 1. 模块概述

Agents 模块是 OpenClaw 的 AI 智能体核心，基于 Pi Agent Core，提供完整的 AI 对话能力。

**核心位置**: `src/agents/` (550 个文件)

## 2. 核心组件

### 2.1 模型目录 (Model Catalog)

```typescript
export type ModelCatalogEntry = {
  id: string;
  name: string;
  provider: string;
  contextWindow?: number;
  reasoning?: boolean;
  input?: ModelInputType[];  // "text" | "image" | "document"
};
```

**支持的主要模型**:

| 提供商 | 模型 |
|--------|------|
| OpenAI | GPT-4o, GPT-4o mini, GPT-5 series |
| Anthropic | Claude 3.5, Claude 3 |
| Google | Gemini 1.5, Gemini 2.0 |
| Github Copilot | Codex models |
| KiloCode | KiloCode models |

### 2.2 系统提示词 (System Prompt)

```typescript
export type PromptMode = "full" | "minimal" | "none";
```

**提示词组成**:

- **Identity**: 身份部分
- **Skills**: 技能部分
- **Memory**: 记忆部分
- **Tools**: 工具部分
- **Messaging**: 消息部分

## 3. 工作流程

```
用户消息
    |
    v
安全检查 (allowFrom)
    |
    v
构建系统提示词
    |
    +-- Identity (身份)
    +-- Skills (技能)
    +-- Memory (记忆)
    +-- Tools (工具)
    +-- Messaging (消息)
    |
    v
LLM 推理
    |
    v
工具执行循环
    |
    +-- Browser (浏览器)
    +-- Execute (命令)
    +-- Message (消息)
    +-- Memory (记忆)
    |
    v
生成回复
```

## 4. 工具系统

### 4.1 核心工具

| 工具 | 功能 |
|------|------|
| `message` | 发送消息到通道 |
| `browser` | 浏览器自动化 |
| `execute` | 执行命令 |
| `memory_search` | 记忆搜索 |
| `memory_get` | 获取记忆内容 |
| `read_file` | 读取文件 |
| `write_file` | 写入文件 |
| `list_directory` | 列出目录 |
| `mcp_*` | MCP 协议工具 |

### 4.2 工具策略

```typescript
export type ToolPolicy = {
  allow: string[];
  deny: string[];
  readonly?: string[];
  filesystem?: FilesystemPolicy;
};
```

### 4.3 工具循环检测

```typescript
export function detectToolLoop(
  toolCalls: ToolCall[],
  config: LoopDetectionConfig
): boolean;
```

## 5. 认证配置 (Auth Profiles)

```typescript
export type AuthProfile = {
  id: string;
  provider: string;
  priority: number;
  apiKey?: string;
  token?: string;
  baseUrl?: string;
  models?: string[];
};
```

### 5.1 认证轮换

```typescript
export function resolveAuthProfileOrder(
  profiles: AuthProfile[],
  config: ResolveConfig
): AuthProfile[];
```

## 6. 工作区管理

### 6.1 工作区结构

```typescript
export type Workspace = {
  dir: string;
  skills: Skill[];
  memory: MemoryFile[];
  bootstrap: BootstrapFile[];
};
```

### 6.2 目录布局

```
workspace/
├── .openclaw/
│   └── skills/
│       ├── skill-1/
│       │   ├── SKILL.md
│       │   └── config.json
│       └── skill-2/
├── memory/
│   ├── MEMORY.md
│   └── 2024-01-01.md
├── .bootstrap/
│   ├── system.yaml
│   └── prompts/
└── .cache/
```

## 7. 子代理 (Subagents)

```typescript
export type SubagentSpawn = {
  action: "list" | "steer" | "kill";
  subagentId?: string;
  model?: string;
  message?: string;
};
```

## 8. 设计亮点

1. **Pi Agent Core**: 基于成熟的 Agent 运行时
2. **模块化提示词**: 可配置的提示词模式 (full/minimal/none)
3. **工具策略**: 细粒度的工具控制
4. **子代理**: 支持多代理协作
5. **认证轮换**: 多认证配置自动切换
6. **记忆集成**: 与 Memory 模块深度集成

---

# Plugin-SDK 模块

## 1. 模块概述

Plugin-SDK 是 OpenClaw 的插件开发工具包，提供创建通道插件、技能插件和上下文引擎所需的 API。

**核心位置**: `src/plugin-sdk/` (114 个文件)

## 2. 核心 API

### 2.1 核心接口

```typescript
export type OpenClawPluginApi = {
  tools: OpenClawPluginToolFactory[];
  hooks?: HookEntry[];
  configSchema?: OpenClawPluginConfigSchema;
};

export type OpenClawPluginService = {
  name: string;
  start: () => Promise<void>;
  stop: () => Promise<void>;
};
```

### 2.2 工具上下文

```typescript
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

### 2.3 工具工厂

```typescript
export type OpenClawPluginToolFactory = (
  ctx: OpenClawPluginToolContext,
) => AnyAgentTool | AnyAgentTool[] | null | undefined;
```

## 3. 认证与安全

### 3.1 认证类型

```typescript
export type ProviderAuthKind = "oauth" | "api_key" | "token" | "device_code" | "custom";
```

### 3.2 OAuth 处理

```typescript
export function buildOauthProviderAuthResult(
  provider: string,
  auth: OAuthCredential
): ProviderAuthResult;
```

### 3.3 SSRF 防护

```typescript
export type SSRFPolicy = {
  allowedIPs?: string[];
  blockedIPs?: string[];
  allowedHosts?: string[];
  blockedHosts?: string[];
};
```

### 3.4 Webhook 守卫

```typescript
export function validateWebhookRequest(
  req: IncomingMessage,
  secret: string,
  policy: WebhookPolicy
): boolean;
```

## 4. 通道插件

### 4.1 通道适配器

```typescript
export type ChannelPlugin = {
  id: ChannelId;
  meta: ChannelMeta;
  capabilities: ChannelCapabilities;
  config: ChannelConfigAdapter;
  setup?: ChannelSetupAdapter;
  security?: ChannelSecurityAdapter;
  messaging?: ChannelMessagingAdapter;
  outbound?: ChannelOutboundAdapter;
  // ...
};
```

### 4.2 认证适配器

```typescript
export type ChannelAuthAdapter = {
  login?: () => Promise<void>;
  logout?: () => Promise<void>;
  refreshToken?: () => Promise<void>;
};
```

## 5. 运行时服务

### 5.1 插件运行时

```typescript
export type PluginRuntime = {
  tools: PluginToolsRuntime;
  config: PluginConfigRuntime;
  logger: PluginLogger;
  store: PluginStore;
};
```

### 5.2 存储

```typescript
export type PluginStore = {
  get: (key: string) => Promise<unknown>;
  set: (key: string, value: unknown) => Promise<void>;
  delete: (key: string) => Promise<void>;
};
```

## 6. 消息处理

### 6.1 回复载荷

```typescript
export type ReplyPayload = {
  text: string;
  mentions?: string[];
  buttons?: Button[];
  replyTo?: string;
};
```

### 6.2 通道发送结果

```typescript
export type ChannelSendResult = {
  ok: boolean;
  messageId?: string;
  error?: string;
};
```

## 7. 设计亮点

1. **统一接口**: 统一的插件开发 API
2. **通道适配**: 预置的 Telegram/Discord/Slack/WhatsApp 适配器
3. **安全防护**: SSRF、Webhook 守卫
4. **运行时**: 完整的工具执行和存储服务
5. **npm 发布**: 支持独立 npm 包发布

---

# Config 模块

## 1. 模块概述

Config 模块负责 OpenClaw 的配置管理，包括配置文件加载、验证、迁移和运行时更新。

**核心位置**: `src/config/` (208 个文件)

## 2. 核心组件

### 2.1 配置加载

```typescript
export async function loadConfig(): Promise<OpenClawConfig>;
export async function readConfigFileSnapshot(): Promise<ConfigSnapshot>;
```

### 2.2 配置验证

```typescript
export function validateConfigObject(
  config: unknown
): { ok: true; value: OpenClawConfig } | { ok: false; errors: string[] };
```

### 2.3 配置迁移

```typescript
export function migrateLegacyConfig(
  config: unknown
): { config: OpenClawConfig | null; changes: string[] };
```

## 3. 配置结构

```typescript
export type OpenClawConfig = {
  gateway?: GatewayConfig;
  models?: ModelsConfig;
  channels?: ChannelsConfig;
  skills?: SkillsConfig;
  memory?: MemoryConfig;
  security?: SecurityConfig;
  secrets?: SecretsConfig;
};
```

### 3.1 Gateway 配置

```typescript
export type GatewayConfig = {
  bind?: "loopback" | "lan" | "tailnet" | "auto";
  port?: number;
  auth?: GatewayAuthConfig;
  controlUi?: ControlUiConfig;
};
```

### 3.2 模型配置

```typescript
export type ModelsConfig = {
  default?: string;
  providers?: Record<string, ProviderConfig>;
};
```

### 3.3 通道配置

```typescript
export type ChannelsConfig = Partial<Record<ChannelId, ChannelConfig>>;
```

## 4. 运行时配置

### 4.1 配置快照

```typescript
export function getRuntimeConfigSnapshot(): OpenClawConfig;
export function setRuntimeConfigSnapshot(config: OpenClawConfig): void;
```

### 4.2 热重载

```typescript
export function setRuntimeConfigSnapshotRefreshHandler(
  handler: () => Promise<void>
): void;
```

## 5. 设计亮点

1. **JSON5 支持**: 灵活的配置文件格式
2. **Schema 验证**: 配置校验
3. **旧配置迁移**: 向后兼容
4. **热重载**: 运行时更新

---

# Infra 模块

## 1. 模块概述

Infra 模块是 OpenClaw 的基础设施层，提供系统级的底层功能。

**核心位置**: `src/infra/` (313 个文件)

## 2. 核心功能

### 2.1 环境管理

```typescript
export function normalizeEnv(): void;
export function getEnv(key: string): string | undefined;
```

### 2.2 二进制管理

```typescript
export function ensureBinary(name: string): Promise<string>;
```

### 2.3 心跳系统

```typescript
export function startHeartbeatRunner(intervalMs: number): void;
```

### 2.4 服务发现 (Bonjour/mDNS)

```typescript
export function startBonjourDiscovery(): void;
export function stopBonjourDiscovery(): void;
```

### 2.5 备份系统

```typescript
export async function createBackup(params: BackupParams): Promise<BackupResult>;
```

### 2.6 归档管理

```typescript
export class ArchiveManager {
  create(path: string): Promise<void>;
  extract(archive: string, dest: string): Promise<void>;
}
```

### 2.7 重启管理

```typescript
export function setGatewaySigusr1RestartPolicy(policy: RestartPolicy): void;
```

### 2.8 密钥文件

```typescript
export function loadSecretFileSync(path: string): SecretFile;
```

## 3. 设计亮点

1. **环境管理**: 统一的环境变量处理
2. **服务发现**: Bonjour/mDNS 支持
3. **备份**: 自动化备份
4. **重启**: 优雅重启

---

# Media 模块

## 1. 模块概述

Media 模块负责 OpenClaw 的媒体处理，包括图片、音频、视频的转码、压缩、缩放和提取。

**核心位置**: `src/media/` (42 个文件)

## 2. 核心组件

### 2.1 图片处理

```typescript
export async function resizeImage(
  input: Buffer,
  options: ResizeOptions
): Promise<Buffer>;

export async function compressImage(
  input: Buffer,
  quality: number
): Promise<Buffer>;

export async function convertImageFormat(
  input: Buffer,
  format: "jpeg" | "png" | "webp"
): Promise<Buffer>;
```

### 2.2 音频处理

```typescript
export async function transcodeAudio(
  input: Buffer,
  format: AudioFormat
): Promise<Buffer>;

export async function extractAudio(
  videoPath: string
): Promise<Buffer>;
```

### 2.3 FFmpeg 集成

```typescript
export async function runFFmpeg(
  args: string[],
  input: Buffer
): Promise<Buffer>;
```

### 2.4 媒体获取

```typescript
export async function fetchMedia(
  url: string,
  options: MediaFetchOptions
): Promise<MediaPayload>;
```

### 2.5 MIME 类型检测

```typescript
export function detectMimeType(buffer: Buffer): string;
export function isSupportedMedia(mimeType: string): boolean;
```

## 3. 处理流程

```
上传媒体
    |
    v
MIME 类型检测
    |
    v
大小检查 (>50MB 拒绝)
    |
    +-- 图片 --> 压缩/缩放
    +-- 音频 --> 转码
    +-- 视频 --> 提取音频
    +-- PDF  --> 文本提取
    |
    v
临时存储
    |
    v
发送到 AI / 通道
```

## 4. 设计亮点

1. **sharp**: 高效图片处理
2. **ffmpeg**: 强大音视频处理
3. **pdfjs-dist**: PDF 解析

---

# CLI/Commands 模块

## 1. 模块概述

CLI/Commands 模块是 OpenClaw 的命令行界面，提供用户与系统交互的入口点。

**核心位置**: 
- `src/cli/` (168 个文件)
- `src/commands/` (299 个文件)

## 2. 核心命令

| 命令 | 功能 |
|------|------|
| `openclaw gateway` | 启动 Gateway |
| `openclaw agent` | 与 AI 交互 |
| `openclaw send` | 发送消息 |
| `openclaw onboard` | 初始化向导 |
| `openclaw doctor` | 诊断问题 |
| `openclaw config` | 配置管理 |
| `openclaw channels` | 通道管理 |
| `openclaw skills` | 技能管理 |

## 3. 使用示例

```bash
# 初始化
openclaw onboard

# 启动 Gateway
openclaw gateway --port 18789

# 与 AI 对话
openclaw agent --message "你好"

# 发送消息
openclaw send --to "+1234567890" --message "Hello"

# 检查状态
openclaw status
```

## 4. 依赖注入

```typescript
export function createDefaultDeps(): CliDeps {
  return {
    gatewayUrl: "http://localhost:18789",
    http: fetch,
    // ...
  };
}
```

---

# Web/TUI 模块

## 1. 模块概述

Web/TUI 模块提供 OpenClaw 的两种用户界面。

**核心位置**: 
- `src/web/` (42 个文件)
- `src/tui/` (33 个文件)

## 2. Web UI

### 2.1 控制界面

```typescript
export async function loadWebAccounts(): Promise<Account[]> {
  // 从 Gateway 获取账户列表
}

export async function sendWebMessage(
  accountId: string,
  target: string,
  message: string
): Promise<void> {
  // 通过 Gateway 发送消息
}
```

### 2.2 自动回复

```typescript
export async function startAutoReply(
  config: AutoReplyConfig
): Promise<void> {
  // 监听消息
  // 调用 AI
  // 发送回复
}
```

## 3. TUI (终端界面)

### 3.1 命令处理

```typescript
export function registerCommandHandlers(
  tui: TUI
): void {
  tui.onCommand("send", handleSend);
  tui.onCommand("agent", handleAgent);
  tui.onCommand("channels", handleChannels);
}
```

### 3.2 事件处理

```typescript
export function registerEventHandlers(
  tui: TUI
): void {
  tui.on("message", handleMessage);
  tui.on("typing", handleTyping);
  tui.on("presence", handlePresence);
}
```

### 3.3 格式化

```typescript
export function formatMessage(message: Message): string {
  // 格式化消息显示
}

export function formatChannelStatus(status: ChannelStatus): string {
  // 格式化通道状态
}
```

### 3.4 主题支持

```typescript
export type Theme = {
  colors: ColorPalette;
  fonts: FontConfig;
};

export function loadTheme(name: string): Theme;
```

---

# Providers 模块

## 1. 模块概述

Providers 模块负责 LLM 提供商的认证和模型配置。

**核心位置**: `src/providers/` (13 个文件)

## 2. 支持的提供商

### 2.1 GitHub Copilot Provider

```typescript
export type GitHubCopilotAuth = {
  token: string;
  refreshToken: string;
  expiresAt: number;
};
```

### 2.2 模型列表

```typescript
export const GITHUB_COPILOT_MODELS = [
  "gpt-4o",
  "gpt-4o-mini",
  // ...
];
```

### 2.3 Google Shared

```typescript
export type GoogleAuthConfig = {
  projectId: string;
  location: string;
  credentials: GoogleCredentials;
};
```

## 3. 认证流程

### 3.1 OAuth 认证

```typescript
export async function authenticateWithOAuth(
  provider: string,
  clientId: string,
  clientSecret: string
): Promise<AuthToken>;
```

---

# 架构特点

## 1. 本地优先 (Local-first)

- Gateway 运行在用户本地设备
- 会话数据本地存储 (`~/.openclaw/sessions/`)
- 密钥本地管理
- 隐私优先的设计

## 2. 多通道聚合

- 统一的通道抽象层 (适配器模式)
- 跨平台消息聚合
- 统一的授权和配额管理
- 20+ 消息平台支持

## 3. 插件系统

- 基于 npm 的插件发布
- Plugin SDK 提供丰富的 API
- 支持通道、技能、记忆等多种扩展
- 35+ 官方插件

## 4. Session 模型

- 主会话 (`main`) 用于直接对话
- 分组隔离
- 多种激活模式
- 队列模式和回复机制

## 5. 模块化设计

- 清晰的职责分离
- 依赖注入
- 可测试性
- 可维护性

---

# 安全设计

## 安全层次

```
+----------------------------------------------------------+
|           认证层 (Authentication)                          |
|  * Token 认证                                            |
|  * OAuth 认证                                            |
|  * 设备配对                                               |
+----------------------------------------------------------+
                        |
+-----------------------v--------------------------------+
|           授权层 (Authorization)                          |
|  * DM 策略 (open/pairing/deny)                         |
|  * 允许列表 (allowFrom)                                |
|  * 角色权限                                             |
+----------------------------------------------------------+
                        |
+-----------------------v--------------------------------+
|           防护层 (Protection)                            |
|  * SSRF 防护                                          |
|  * Webhook 签名验证                                   |
|  * 输入清理                                            |
|  * 速率限制                                            |
+----------------------------------------------------------+
                        |
+-----------------------v--------------------------------+
|           审计层 (Audit)                                 |
|  * 操作日志                                            |
|  * 消息记录                                            |
|  * 审计追踪                                            |
+----------------------------------------------------------+
```

---

# 快速开始

## 安装

```bash
# 使用 npm
npm install -g openclaw@latest

# 使用 pnpm
pnpm add -g openclaw@latest
```

## 初始化

```bash
# 启动引导向导
openclaw onboard --install-daemon
```

## 运行

```bash
# 启动 Gateway
openclaw gateway --port 18789

# 在另一个终端，与 AI 对话
openclaw agent --message "你好"

# 发送消息
openclaw send --to "+1234567890" --message "Hello"
```

## 配置通道

```bash
# 查看可用通道
openclaw channels status

# 启动 Telegram
openclaw channels start telegram

# 启动 WhatsApp
openclaw channels start whatsapp
```

---

# 关键依赖

## 运行时依赖

| 包 | 版本 | 用途 |
|---|------|------|
| @mariozechner/pi-agent-core | 0.57.1 | Agent 核心 |
| grammy | ^1.41.1 | Telegram 框架 |
| @slack/bolt | ^4.6.0 | Slack 框架 |
| @whiskeysockets/baileys | 7.0.0-rc.9 | WhatsApp 框架 |
| hono | 4.12.7 | Web 服务器 |
| sqlite-vec | 0.1.7-alpha.2 | 向量搜索 |
| sharp | ^0.34.5 | 图片处理 |
| playwright-core | 1.58.2 | 浏览器自动化 |

## 开发依赖

| 包 | 版本 | 用途 |
|---|------|------|
| typescript | ^5.9.3 | 类型系统 |
| vitest | ^4.0.18 | 测试框架 |
| oxlint | ^1.51.0 | 代码检查 |
| oxfmt | 0.36.0 | 代码格式化 |
| tsdown | 0.21.0 | 构建工具 |

---

# 客户端应用

## macOS 应用
- **语言**: Swift + SwiftUI
- **位置**: `apps/macos/`
- **功能**: 菜单栏应用、Canvas 渲染、语音唤醒

## iOS 应用
- **语言**: Swift + SwiftUI
- **位置**: `apps/ios/`
- **功能**: 移动端助手

## Android 应用
- **语言**: Kotlin + Jetpack Compose
- **位置**: `apps/android/`
- **功能**: 移动端助手、语音通话

---

# 总结

OpenClaw 是一个**功能完整的个人 AI 助手框架**，它：

1. **统一多平台** - 一个界面管理 20+ 消息渠道
2. **本地运行** - 数据隐私有保障
3. **高度可扩展** - 插件系统支持自定义开发
4. **模块化设计** - 每个模块职责清晰
5. **生产级质量** - 完善的测试和安全机制

**技术亮点**：

- 混合搜索 (向量 + 关键词 + MMR + 时间衰减)
- 双后端支持 (Builtin + QMD)
- 6 种嵌入提供商
- 配置热重载
- 通道自动恢复

无论你是想快速搭建一个个人 AI 助手，还是想开发自定义插件，OpenClaw 都能满足你的需求。

---

*文档生成时间: 2026-03-11*
