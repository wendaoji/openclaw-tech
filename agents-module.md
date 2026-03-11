# OpenClaw Agents 模块技术原理详解

## 1. 模块概述

Agents 模块是 OpenClaw 的 AI 智能体核心，负责 L与LM 的交互、工具调用、对话管理和系统提示词生成。它基于 Pi Agent Core 构建，提供完整的 AI 对话能力。

**核心位置**: `src/agents/` (550 个文件)

## 2. 架构设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Agents Layer                                   │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │
│  │    Pi Agent    │  │   Model        │  │   Tools        │    │
│  │     Core       │  │   Catalog       │  │   Catalog       │    │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘    │
├─────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    System Prompt Builder                       │   │
│  │  identity | skills | memory | tools | messaging | time      │   │
│  └──────────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │
│  │   Subagent     │  │   Auth         │  │   Workspace    │    │
│  │   Spawn        │  │   Profiles     │  │   Management   │    │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

## 3. 核心组件

### 3.1 模型目录 (Model Catalog)

管理支持的 LLM 模型列表：

```typescript
// model-catalog.ts
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

### 3.2 系统提示词 (System Prompt)

`system-prompt.ts` 构建 AI 助手的系统提示词：

```typescript
// 提示词模式
export type PromptMode = "full" | "minimal" | "none";
```

**提示词组成**:

```typescript
// 1. 身份部分
"## Authorized Senders"
"Authorized senders: +1234567890, +0987654321"

// 2. 技能部分
"## Skills (mandatory)"
"- Read SKILL.md when applicable"
"- Follow skill constraints"

// 3. 记忆部分
"## Memory Recall"
"- Run memory_search before answering about prior work"

// 4. 工具部分
"## Tools"
"- message: Send messages to channels"

// 5. 消息部分
"## Messaging"
"- Reply in current session"
"- Cross-session messaging"
```

### 3.3 子代理 (Subagents)

```typescript
// subagent-spawn.ts
export type SubagentSpawn = {
  action: "list" | "steer" | "kill";
  subagentId?: string;
  model?: string;
  message?: string;
};
```

### 3.4 认证配置 (Auth Profiles)

```typescript
// auth-profiles.ts
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

## 4. 工具系统

### 4.1 工具目录

```typescript
// tool-catalog.ts
export type AgentTool<TParam = unknown, TResult = unknown> = {
  name: string;
  description: string;
  parameters: TParam;
  execute: (params: TParam) => Promise<TResult>;
};
```

### 4.2 内置工具

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

### 4.3 工具策略

```typescript
// tool-policy.ts
export type ToolPolicy = {
  allow: string[];
  deny: string[];
  readonly?: string[];
  filesystem?: FilesystemPolicy;
};
```

### 4.4 工具循环检测

```typescript
// tool-loop-detection.ts
export function detectToolLoop(
  toolCalls: ToolCall[],
  config: LoopDetectionConfig
): boolean;
```

## 5. 对话管理

### 5.1 会话

```typescript
// session.ts
export type AgentSession = {
  key: string;
  agentId: string;
  channelId: string;
  history: Message[];
  context: SessionContext;
};
```

### 5.2 消息历史

```typescript
export type Message = {
  id: string;
  role: "user" | "assistant" | "system";
  content: MessageContent;
  timestamp: number;
  toolCalls?: ToolCall[];
  toolResults?: ToolResult[];
};
```

### 5.3 转录策略

```typescript
// transcript-policy.ts
export type TranscriptPolicy = {
  mode: "full" | "minimal" | "off";
  includeThinking?: boolean;
  includeToolCalls?: boolean;
};
```

## 6. 工作区管理

### 6.1 工作区结构

```typescript
// workspace.ts
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

### 6.3 技能 (Skills)

```typescript
// skills/types.ts
export type Skill = {
  id: string;
  name: string;
  description: string;
  trigger: string | string[];
  action: SkillAction;
};
```

## 7. 认证与模型选择

### 7.1 认证轮换

```typescript
// auth-profiles/resolve-auth-profile-order.ts
export function resolveAuthProfileOrder(
  profiles: AuthProfile[],
  config: ResolveConfig
): AuthProfile[];
```

### 7.2 模型回退

```typescript
// models-config.ts
export type ModelFallback = {
  primary: string;
  fallback: string[];
  provider?: string;
};
```

## 8. 文件结构

| 目录/文件 | 功能 |
|----------|------|
| `model-catalog.ts` | 模型目录管理 |
| `system-prompt.ts` | 系统提示词生成 |
| `tool-catalog.ts` | 工具目录 |
| `tool-policy.ts` | 工具策略 |
| `subagent-spawn.ts` | 子代理管理 |
| `auth-profiles/` | 认证配置 |
| `workspace.ts` | 工作区管理 |
| `skills/` | 技能系统 |
| `pi-model-discovery.ts` | 模型发现 |
| `pi-agent-core/` | Agent Core 集成 |

## 9. 核心流程

### 9.1 对话流程

```
User Message
    │
    ▼
Security Check (allowFrom)
    │
    ▼
Resolve Session Key
    │
    ▼
Build System Prompt
    │
    ├─ Identity
    ├─ Skills
    ├─ Memory
    ├─ Tools
    └─ Messaging
    │
    ▼
LLM Inference
    │
    ▼
Tool Execution Loop
    │
    ├─ Browser
    ├─ Execute
    ├─ Message
    └─ Memory
    │
    ▼
Response Generation
    │
    ▼
Deliver to Channel
```

### 9.2 工具调用流程

```
LLM Request (tool_call)
    │
    ▼
Tool Policy Check
    │
    ├─ Allowlist
├─ Denylist
└─ Filesystem Policy
    │
    ▼
Loop Detection
    │
    ▼
Tool Execution
    │
    ▼
Result Formatting
    │
    ▼
Return to LLM
```

## 10. 设计亮点

1. **Pi Agent Core**: 基于成熟的 Agent 运行时
2. **模块化提示词**: 可配置的提示词模式 (full/minimal/none)
3. **工具策略**: 细粒度的工具控制
4. **子代理**: 支持多代理协作
5. **认证轮换**: 多认证配置自动切换
6. **记忆集成**: 与 Memory 模块深度集成
7. **技能系统**: 可插拔的技能机制

## 11. 总结

Agents 模块是 OpenClaw 的 AI 智能体核心，其设计特点包括：

1. **Pi Core 集成**: 基于 `pi-agent-core` 的成熟运行时
2. **系统提示词**: 模块化、可配置的提示词生成
3. **工具系统**: 丰富的内置工具 + 策略控制
4. **多模型支持**: OpenAI/Anthropic/Google 等多提供商
5. **认证管理**: 认证配置轮换和回退
6. **工作区**: 本地工作区 + 技能 + 记忆
7. **子代理**: 支持多代理协作和编排
