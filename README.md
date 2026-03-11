# OpenClaw 技术完全解析

## 一句话概述

**OpenClaw** 是一个运行在用户本地设备上的多通道 AI 网关，能够在 WhatsApp、Telegram、Slack、Discord、Signal、iMessage 等 20+ 消息平台上提供统一的 AI 助手服务。

---

## 目录

1. [系统概览](#系统概览)
2. [核心架构](#核心架构)
3. [模块详解](#模块详解)
4. [数据流分析](#数据流分析)
5. [关键技术选型](#关键技术选型)
6. [安全设计](#安全设计)
7. [扩展机制](#扩展机制)
8. [快速开始](#快速开始)

---

## 系统概览

### 什么是 OpenClaw？

OpenClaw 是一个**个人 AI 助手**，它不是云服务，而是运行在你自己的设备上（Mac、Windows、Linux、Raspberry Pi）。你可以通过多种渠道与它对话，它会在所有渠道上回复你。

### 核心特性

| 特性 | 说明 |
|------|------|
| 多通道 | 支持 20+ 消息平台 |
| 本地运行 | 数据存储在本地，保护隐私 |
| AI 驱动 | 基于 Pi Agent Core |
| 可扩展 | 35+ 官方插件 |
| 开源 | MIT 许可证 |

### 典型使用场景

```
用户 (在 WhatsApp)
    │
    ▼
OpenClaw Gateway (本地运行)
    │
    ├──▶ AI Agent (处理对话)
    │         │
    │         ├──▶ Memory (检索记忆)
    │         ├──▶ Tools (执行工具)
    │         └──▶ Models (调用 LLM)
    │
    ▼
用户 (收到回复)
```

---

## 核心架构

### 分层架构

```
┌────────────────────────────────────────────┐
│           用户界面层 (Presentation)          │
│  ┌─────────┐  ┌─────────┐  ┌──────────┐  │
│  │  Web UI │  │   TUI   │  │  Mobile  │  │
│  └────┬────┘  └────┬────┘  └─────┬────┘  │
└───────┼─────────────┼──────────────┼────────┘
        │             │              │
┌───────▼─────────────▼──────────────▼────────┐
│           网关层 (Gateway)                    │
│  ┌────────────────────────────────────────┐ │
│  │     WebSocket Server (ws://:18789)     │ │
│  │     HTTP Server (http://:18789)         │ │
│  └────────────────────────────────────────┘ │
│  • 会话管理  • 认证授权  • 消息路由           │
│  • 定时任务  • 健康监控  • 配置热重载          │
└────────────────────┬─────────────────────────┘
                     │
    ┌────────────────┼────────────────┐
    │                │                │
┌───▼────┐    ┌─────▼─────┐    ┌───▼──────┐
│通道层   │    │  智能体层  │    │  记忆层   │
│Channels │    │  Agents   │    │  Memory  │
└────┬────┘    └─────┬─────┘    └────┬─────┘
     │               │                │
     ▼               ▼                ▼
 消息平台          LLM              向量存储
```

### 核心模块关系

```
Gateway (网关)
    │
    ├── Channels (通道层)
    │       ├── Telegram ──────┐
    │       ├── Discord ───────┤
    │       ├── Slack ─────────┼──▶ 消息平台
    │       ├── WhatsApp ──────┤
    │       └── [35+ 扩展] ───┘
    │
    ├── Agents (智能体)
    │       ├── Pi Agent Core
    │       ├── System Prompt
    │       ├── Tools Catalog
    │       └── Auth Profiles
    │
    ├── Memory (记忆)
    │       ├── Vector Search (sqlite-vec)
    │       ├── FTS Search (SQLite FTS5)
    │       ├── Embedding Providers
    │       └── Hybrid Ranking
    │
    ├── Config (配置)
    │       ├── JSON5 Config
    │       ├── Schema Validation
    │       ├── Legacy Migration
    │       └── Hot Reload
    │
    └── Plugin-SDK (插件)
            ├── Channel Adapters
            ├── Tool Factories
            ├── Auth Handlers
            └── Security Guards
```

---

## 模块详解

### 1. Gateway - 控制中枢

Gateway 是整个系统的心脏，负责协调所有组件。

**核心职责**：
- 监听 WebSocket 连接 (`ws://localhost:18789`)
- 管理客户端会话
- 路由消息到正确的通道
- 调度 Agent 处理请求
- 定时任务执行

**关键组件**：
```typescript
// 启动 Gateway
const gateway = await startGatewayServer(18789, {
  bind: "loopback",
  controlUiEnabled: true,
  auth: { mode: "token", token: "secret" }
});
```

**详细文档**：[gateway-module.md](gateway-module.md)

---

### 2. Channels - 消息通道

Channels 模块让 OpenClaw 能够与各种消息平台对话。

**架构**：
```typescript
// 通道插件接口
type ChannelPlugin = {
  id: ChannelId;           // 通道唯一标识
  meta: ChannelMeta;       // 元信息
  capabilities: ChannelCapabilities;  // 能力声明
  messaging?: ChannelMessagingAdapter;  // 消息收发
  security?: ChannelSecurityAdapter;     // 安全策略
  outbound?: ChannelOutboundAdapter;    // 出站消息
};
```

**支持的通道**：
- **内置**: Telegram, Discord, Slack, Signal, WhatsApp, iMessage, LINE, Web
- **扩展**: Matrix, Microsoft Teams, Feishu, IRC, Nostr, Twitch, Zalo, BlueBubbles 等 35+

**详细文档**：[channels-module.md](channels-module.md)

---

### 3. Agents - AI 智能体

Agents 模块是 OpenClaw 的大脑，处理用户输入并生成回复。

**工作流程**：
```
用户消息
    │
    ▼
安全检查 (allowFrom)
    │
    ▼
构建系统提示词
    │
    ├── Identity (身份)
    ├── Skills (技能)
    ├── Memory (记忆)
    ├── Tools (工具)
    └── Messaging (消息)
    │
    ▼
LLM 推理
    │
    ▼
工具执行循环
    │
    ├── Browser (浏览器)
    ├── Execute (命令)
    ├── Message (消息)
    └── Memory (记忆)
    │
    ▼
生成回复
    │
    ▼
发送到通道
```

**核心组件**：
- **Pi Agent Core**: AI 运行时
- **System Prompt**: 动态生成提示词
- **Tool Catalog**: 工具目录
- **Auth Profiles**: 认证配置

**详细文档**：[agents-module.md](agents-module.md)

---

### 4. Memory - 记忆系统

Memory 模块让 AI 能够记住之前的对话和知识。

**搜索架构**：
```
查询
    │
    ▼
┌─────────────────────┐
│  Query Expansion    │  (提取关键词)
└──────────┬──────────┘
           │
     ┌─────┴─────┐
     ▼           ▼
┌────────┐  ┌────────┐
│ Vector │  │  FTS   │
│ Search │  │ Search │
└───┬────┘  └───┬────┘
    │           │
    └─────┬─────┘
          │
          ▼
┌─────────────────────┐
│  Hybrid Merge       │  (评分混合)
│  score = v*0.7 + t*0.3
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Temporal Decay     │  (时间衰减)
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  MMR Re-rank       │  (多样性排序)
└──────────┬──────────┘
           │
           ▼
    Top-K 结果
```

**嵌入提供商**：
- OpenAI (text-embedding-3-small)
- Google Gemini (gemini-embedding-001)
- Voyage AI (voyage-large-2)
- Mistral (mistral-embed)
- Ollama (本地模型)
- Local (node-llama-cpp)

**详细文档**：[memory-module.md](memory-module.md)

---

### 5. Plugin-SDK - 扩展机制

Plugin-SDK 让你能够开发自己的插件来扩展 OpenClaw。

**插件类型**：
| 类型 | 用途 |
|------|------|
| `channel` | 添加新消息平台 |
| `skill` | 添加自定义技能 |
| `memory` | 添加新记忆后端 |
| `context-engine` | 添加上下文引擎 |

**开发示例**：
```typescript
// 创建通道插件
import { ChannelPlugin } from "@openclaw/plugin-sdk";

const myChannelPlugin: ChannelPlugin = {
  id: "my-channel",
  meta: { label: "My Channel" },
  capabilities: { chatTypes: ["direct", "group"] },
  messaging: { start: async ({ onInbound }) => { /* ... */ } },
  // ...
};
```

**详细文档**：[plugin-sdk-module.md](plugin-sdk-module.md)

---

### 6. Config - 配置管理

Config 模块管理系统所有配置。

**配置文件位置**：
- 默认: `~/.openclaw/config.json`
- 环境变量覆盖

**配置结构**：
```json
{
  "gateway": {
    "bind": "loopback",
    "port": 18789,
    "auth": { "mode": "token" }
  },
  "models": {
    "default": "gpt-4o"
  },
  "channels": {
    "telegram": { "enabled": true },
    "slack": { "enabled": true }
  }
}
```

**详细文档**：[config-module.md](config-module.md)

---

### 7. Media - 媒体处理

Media 模块处理图片、音频、视频。

**处理流程**：
```
上传媒体
    │
    ▼
MIME 类型检测
    │
    ▼
大小检查 (>50MB 拒绝)
    │
    ├─ 图片 ──▶ 压缩/缩放
    ├─ 音频 ──▶ 转码
    ├─ 视频 ──▶ 提取音频
    └─ PDF ──▶ 文本提取
    │
    ▼
临时存储
    │
    ▼
发送到 AI / 通道
```

**技术栈**：
- `sharp`: 图片处理
- `ffmpeg`: 音视频处理
- `pdfjs-dist`: PDF 解析

**详细文档**：[media-module.md](media-module.md)

---

### 8. CLI - 命令行

CLI 模块提供用户与 OpenClaw 交互的入口。

**常用命令**：
```bash
# 初始化
openclaw onboard

# 启动 Gateway
openclaw gateway

# 与 AI 对话
openclaw agent --message "你好"

# 发送消息
openclaw send --to +1234567890 --message "Hello"

# 检查状态
openclaw status

# 诊断问题
openclaw doctor
```

**详细文档**：[cli-commands-module.md](cli-commands-module.md)

---

## 数据流分析

### 完整消息流程

```
1. 用户发送消息
   │
   ▼
2. 通道接收 (Telegram/Discord/WhatsApp/...)
   │
   ▼
3. Gateway 认证检查
   │   - Token 验证
   │   - 允许列表检查
   │   - DM 策略检查
   ▼
4. 消息解析
   │   - 文本提取
   │   - 附件处理
   │   - 提及解析
   ▼
5. 会话解析
   │   - 确定 sessionKey
   │   - 确定 channelId
   │   - 确定 accountId
   ▼
6. 入站处理
   │   - 消息去重
   │   - 防抖
   │   - 排队
   ▼
7. Agent 处理
   │   - 构建提示词
   │   - LLM 调用
   │   - 工具执行
   ▼
8. 记忆检索
   │   - 向量搜索
   │   - 关键词搜索
   │   - 结果混合
   ▼
9. 生成回复
   │   - 流式输出
   │   - 特殊标签处理
   │   - 格式化
   ▼
10. 发送回复
    │   - 路由到原通道
    │   - 附件上传
    │   - 按钮/模板处理
    ▼
11. 消息确认
        - 已读回执
        -  реакции (反应)
        ▼
12. 完成
```

---

## 关键技术选型

### 为什么选择这些技术？

| 技术 | 选型理由 |
|------|----------|
| **Node.js 22+** | 原生 ESM、V8 高性能、异步 I/O |
| **WebSocket** | 双向通信、实时推送、低延迟 |
| **SQLite** | 零配置、单文件、向量支持 |
| **TypeScript** | 类型安全、IDE 支持、重构友好 |
| **Pi Agent Core** | 成熟的 AI 运行时 |
| **Hono** | 轻量、高性能、类型友好 |

---

## 安全设计

### 安全层次

```
┌─────────────────────────────────────────┐
│           认证层 (Authentication)        │
│  • Token 认证                           │
│  • OAuth 认证                           │
│  • 设备配对                            │
└────────────────────────────────────────┘
                    │
┌───────────────────▼────────────────────┐
│           授权层 (Authorization)        │
│  • DM 策略 (open/pairing/deny)         │
│  • 允许列表 (allowFrom)                │
│  • 角色权限                            │
└────────────────────────────────────────┘
                    │
┌───────────────────▼────────────────────┐
│           防护层 (Protection)           │
│  • SSRF 防护                          │
│  • Webhook 签名验证                    │
│  • 输入清理                            │
│  • 速率限制                            │
└────────────────────────────────────────┘
                    │
┌───────────────────▼────────────────────┐
│           审计层 (Audit)                │
│  • 操作日志                            │
│  • 消息记录                            │
│  • 审计追踪                            │
└────────────────────────────────────────┘
```

### DM 策略

```typescript
// 配置 DM 策略
channels: {
  telegram: {
    dmPolicy: "pairing"  // 需要配对
  },
  slack: {
    dmPolicy: "open",    // 开放
    allowFrom: ["*"]     // 允许所有人
  },
  discord: {
    dmPolicy: "deny"     // 拒绝 DM
  }
}
```

---

## 扩展机制

### 开发自定义通道

```typescript
// 1. 创建插件
// my-channel/package.json
{
  "name": "@openclaw/my-channel",
  "plugin-sdk": "2026.2.19"
}

// 2. 实现通道接口
import { ChannelPlugin } from "@openclaw/plugin-sdk";

export const myChannel: ChannelPlugin = {
  id: "my-channel",
  meta: {
    label: "My Channel",
    blurb: "Custom messaging channel"
  },
  capabilities: {
    chatTypes: ["direct", "group"]
  },
  async start({ cfg, accountId, ctx }) {
    // 启动消息监听
    ctx.onInbound(message => {
      // 处理入站消息
    });
    return () => {
      // 清理函数
    };
  }
};
```

### 开发自定义技能

```typescript
// skills/my-skill/SKILL.md
# My Skill

## Triggers
- /my-skill
- 帮我做...

## Action
1. 读取配置
2. 执行操作
3. 返回结果
```

---

## 快速开始

### 安装

```bash
# 使用 npm
npm install -g openclaw@latest

# 使用 pnpm
pnpm add -g openclaw@latest
```

### 初始化

```bash
# 启动引导向导
openclaw onboard --install-daemon
```

### 运行

```bash
# 启动 Gateway
openclaw gateway --port 18789

# 在另一个终端，与 AI 对话
openclaw agent --message "你好"
```

### 配置通道

```bash
# 查看可用通道
openclaw channels status

# 启动 Telegram
openclaw channels start telegram

# 启动 WhatsApp
openclaw channels start whatsapp
```

---

## 相关文档

| 文档 | 说明 |
|------|------|
| [architecture.md](architecture.md) | 技术架构总览 |
| [gateway-module.md](gateway-module.md) | Gateway 模块详解 |
| [channels-module.md](channels-module.md) | 通道模块详解 |
| [agents-module.md](agents-module.md) | 智能体模块详解 |
| [memory-module.md](memory-module.md) | 记忆模块详解 |
| [plugin-sdk-module.md](plugin-sdk-module.md) | 插件 SDK 详解 |
| [config-module.md](config-module.md) | 配置模块详解 |
| [infra-module.md](infra-module.md) | 基础设施详解 |
| [media-module.md](media-module.md) | 媒体处理详解 |
| [cli-commands-module.md](cli-commands-module.md) | CLI 命令详解 |
| [web-tui-module.md](web-tui-module.md) | 用户界面详解 |
| [providers-module.md](providers-module.md) | LLM 提供商详解 |

---

## 总结

OpenClaw 是一个功能完整的个人 AI 助手框架，它：

1. **统一多平台** - 一个界面管理 20+ 消息渠道
2. **本地运行** - 数据隐私有保障
3. **高度可扩展** - 插件系统支持自定义开发
4. **模块化设计** - 每个模块职责清晰
5. **生产级质量** - 完善的测试和安全机制

无论你是想快速搭建一个个人 AI 助手，还是想开发自定义插件，OpenClaw 都能满足你的需求。
