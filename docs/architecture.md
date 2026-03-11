# OpenClaw 技术架构分析文档

## 项目概述

**OpenClaw** 是一个多通道 AI 网关（Multi-channel AI Gateway），用于在多个消息平台上运行个人 AI 助手。它允许用户通过 WhatsApp、Telegram、Slack、Discord、Google Chat、Signal、iMessage 等渠道与 AI 助手交互。

OpenClaw 采用本地优先（Local-first）架构，所有数据存储在用户本地设备，保护隐私。系统由 Gateway（控制平面）、多通道接入层、AI Agent 运行时和丰富的插件系统组成。

---

## 核心技术栈

| 类别 | 技术 |
|------|------|
| 运行时 | Node.js 22+ (ESM) |
| 包管理 | pnpm (workspace) |
| 语言 | TypeScript (严格模式) |
| 构建工具 | tsdown |
| 测试框架 | Vitest |
| 代码检查 | Oxlint, Oxfmt |

---

## 系统架构总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              OpenClaw System                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌──────────┐  │
│  │   macOS    │    │    iOS     │    │  Android   │    │  Web UI  │  │
│  │  (Swift)   │    │  (Swift)   │  │ (Kotlin)   │    │ (React)  │  │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘    └────┬─────┘  │
│         │                  │                  │                 │          │
│         └──────────────────┼──────────────────┼─────────────────┘          │
│                            │                  │                            │
│                     ┌──────▼──────────────────▼──────┐                     │
│                     │        Gateway Server           │                     │
│                     │     (WebSocket + HTTP)        │                     │
│                     │         src/gateway/           │                     │
│                     └──────────────┬────────────────┘                     │
│                                    │                                      │
│         ┌─────────────────────────┼─────────────────────────┐           │
│         │                         │                         │           │
│  ┌──────▼──────┐    ┌────────────▼────────────┐   ┌──────▼──────┐     │
│  │  Channels   │    │        Agents          │   │   Config    │     │
│  │  (Message) │    │    (AI Intelligence)    │   │  (Settings) │     │
│  └──────┬──────┘    └───────────┬───────────┘   └─────────────┘     │
│         │                        │                                       │
│  ┌──────▼──────┐    ┌──────────▼──────────┐                         │
│  │  Plugins    │    │       Memory         │                         │
│  │ (Extensions)│    │  (Vector Search)     │                         │
│  └─────────────┘    └─────────────────────┘                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 核心模块详解

### 1. Gateway (网关) - 控制平面

Gateway 是 OpenClaw 的核心控制平面，通过 WebSocket 协议管理所有客户端连接、会话、通道和消息路由。

**核心功能**：
- WebSocket 服务器 (`ws://localhost:18789`)
- HTTP 控制界面 (`http://localhost:18789`)
- 会话管理 (Session Management)
- 通道生命周期管理
- 认证与授权
- 定时任务 (Cron)
- 健康监控

**关键文件**：`src/gateway/` (244 个文件)

**详细分析**：[gateway-module.md](gateway-module.md)

---

### 2. Channels (通道层) - 消息接入

通道层提供统一的抽象接口，支持多种消息平台。采用适配器模式，每种消息平台通过适配器接入。

**内置通道**：
| 通道 | 框架 | 位置 |
|------|------|------|
| Telegram | GramJS | `src/telegram/` |
| Discord | discord.js | `src/discord/` |
| Slack | @slack/bolt | `src/slack/` |
| Signal | libsignal | `src/signal/` |
| WhatsApp | Baileys | `src/whatsapp/` |
| iMessage | BlueBubbles | `src/imessage/` |
| LINE | @line/bot-sdk | `src/line/` |
| Web | WebSocket | `src/web/` |

**扩展通道** (35+)：Matrix, Microsoft Teams, Feishu, IRC, Nostr, Twitch, Zalo 等

**核心接口**：
```typescript
type ChannelPlugin = {
  id: ChannelId;
  meta: ChannelMeta;
  capabilities: ChannelCapabilities;
  messaging?: ChannelMessagingAdapter;
  security?: ChannelSecurityAdapter;
  outbound?: ChannelOutboundAdapter;
  // ...
};
```

**详细分析**：[channels-module.md](channels-module.md)

---

### 3. Agents (智能体) - AI 核心

Agent 模块基于 Pi Agent Core，提供完整的 AI 对话能力。

**核心功能**：
- 系统提示词生成
- 工具调用管理
- 模型目录
- 认证配置
- 工作区管理
- 子代理支持

**支持的模型**：
| 提供商 | 模型 |
|--------|------|
| OpenAI | GPT-4o, GPT-4o mini, GPT-5 series |
| Anthropic | Claude 3.5, Claude 3 |
| Google | Gemini 1.5, Gemini 2.0 |
| GitHub Copilot | Codex models |

**核心工具**：
| 工具 | 功能 |
|------|------|
| `message` | 发送消息 |
| `browser` | 浏览器自动化 |
| `execute` | 执行命令 |
| `memory_search` | 记忆搜索 |
| `read_file` | 读取文件 |
| `write_file` | 写入文件 |

**详细分析**：[agents-module.md](agents-module.md)

---

### 4. Memory (记忆系统) - 知识管理

Memory 模块提供持久化和向量搜索能力，采用混合搜索策略。

**架构**：
```
MemorySearchManager
    │
    ├─ QMD Backend (外部 CLI)
    │
    └─ Builtin Backend
            │
            ├─ Hybrid Search
            │   ├─ Vector (sqlite-vec)
            │   └─ FTS (SQLite FTS5)
            │
            ├─ Embedding Providers
            │   ├─ OpenAI
            │   ├─ Google Gemini
            │   ├─ Voyage AI
            │   ├─ Mistral
            │   ├─ Ollama
            │   └─ Local (node-llama-cpp)
            │
            └─ Ranking
                ├─ MMR (多样性)
                └─ Temporal Decay (时间衰减)
```

**核心特性**：
- 混合搜索 (向量 + 关键词)
- MMR 重排序
- 时间衰减
- 查询扩展
- 嵌入缓存

**详细分析**：[memory-module.md](memory-module.md)

---

### 5. Plugin-SDK (插件系统) - 扩展机制

Plugin-SDK 提供插件开发所需的 API，支持通道插件、技能插件和上下文引擎。

**导出模块**：
| 模块 | 用途 |
|------|------|
| `plugin-sdk/core` | 核心 API |
| `plugin-sdk/telegram` | Telegram 适配器 |
| `plugin-sdk/discord` | Discord 适配器 |
| `plugin-sdk/slack` | Slack 适配器 |
| `plugin-sdk/whatsapp` | WhatsApp 适配器 |
| `plugin-sdk/memory-core` | 记忆核心 |
| `plugin-sdk/memory-lancedb` | LanceDB 向量存储 |

**安全特性**：
- SSRF 防护
- Webhook 守卫
- 认证类型：OAuth、API Key、Token、Device Code

**详细分析**：[plugin-sdk-module.md](plugin-sdk-module.md)

---

### 6. Config (配置管理) - 系统设置

Config 模块负责配置文件的加载、验证、迁移和运行时更新。

**核心功能**：
- JSON5 配置解析
- Schema 验证
- 旧配置迁移
- 热重载

**配置结构**：
```typescript
type OpenClawConfig = {
  gateway?: GatewayConfig;
  models?: ModelsConfig;
  channels?: ChannelsConfig;
  skills?: SkillsConfig;
  memory?: MemoryConfig;
  security?: SecurityConfig;
  secrets?: SecretsConfig;
};
```

**详细分析**：[config-module.md](config-module.md)

---

### 7. Infra (基础设施) - 底层支持

Infra 模块提供系统级的底层功能。

**核心功能**：
| 功能 | 说明 |
|------|------|
| 环境管理 | `env.ts` |
| 二进制管理 | `binaries.ts` |
| 心跳系统 | `heartbeat-runner.ts` |
| 服务发现 | `bonjour.ts` (mDNS) |
| 备份系统 | `backup-create.ts` |
| 归档管理 | `archive.ts` |
| 重启机制 | `restart.ts` |

**详细分析**：[infra-module.md](infra-module.md)

---

### 8. Media (媒体管道) - 内容处理

Media 模块处理图片、音频、视频等媒体文件。

**核心功能**：
| 功能 | 技术 |
|------|------|
| 图片处理 | `sharp` |
| 音频转码 | `ffmpeg` |
| 视频提取 | `ffmpeg` |
| PDF 解析 | `pdfjs-dist` |

**详细分析**：[media-module.md](media-module.md)

---

### 9. CLI/Commands (命令行) - 用户交互

CLI/Commands 模块提供用户与系统交互的入口。

**核心命令**：
| 命令 | 功能 |
|------|------|
| `openclaw gateway` | 启动 Gateway |
| `openclaw agent` | 与 AI 交互 |
| `openclaw send` | 发送消息 |
| `openclaw onboard` | 初始化向导 |
| `openclaw doctor` | 诊断问题 |
| `openclaw config` | 配置管理 |

**详细分析**：[cli-commands-module.md](cli-commands-module.md)

---

### 10. Web/TUI (用户界面)

Web/TUI 模块提供两种用户界面。

**Web UI**：
- 基于浏览器的控制界面
- 账户管理
- 消息发送
- 通道状态监控

**TUI**：
- 终端用户界面
- 命令处理
- 事件处理
- 主题支持

**详细分析**：[web-tui-module.md](web-tui-module.md)

---

### 11. Providers (模型提供商)

Providers 模块负责 LLM 提供商的认证集成。

**支持的提供商**：
| 提供商 | 认证方式 |
|--------|----------|
| GitHub Copilot | OAuth |
| Google | OAuth |
| Qwen Portal | OAuth |

**详细分析**：[providers-module.md](providers-module.md)

---

## 架构特点

### 1. 本地优先 (Local-first)

- Gateway 运行在用户本地设备
- 会话数据本地存储 (`~/.openclaw/sessions/`)
- 密钥本地管理
- 隐私优先的设计

### 2. 多通道聚合

- 统一的通道抽象层 (适配器模式)
- 跨平台消息聚合
- 统一的授权和配额管理
- 20+ 消息平台支持

### 3. 插件系统

- 基于 npm 的插件发布
- Plugin SDK 提供丰富的 API
- 支持通道、技能、记忆等多种扩展
- 35+ 官方插件

### 4. Session 模型

- 主会话 (`main`) 用于直接对话
- 分组隔离
- 多种激活模式
- 队列模式和回复机制

### 5. 安全特性

- DM 配对机制 (`dmPolicy="pairing"`)
- 允许列表控制
- SSRF 防护
- Webhook 守卫
- 执行审批流程

---

## 客户端应用

### macOS 应用
- **语言**: Swift + SwiftUI
- **位置**: `apps/macos/`
- **功能**: 菜单栏应用、Canvas 渲染、语音唤醒

### iOS 应用
- **语言**: Swift + SwiftUI
- **位置**: `apps/ios/`
- **功能**: 移动端助手

### Android 应用
- **语言**: Kotlin + Jetpack Compose
- **位置**: `apps/android/`
- **功能**: 移动端助手、语音通话

---

## 关键技术决策

### 1. 为什么选择 Node.js 22+？

- 原生 ESM 支持
- 优秀的异步性能
- 丰富的生态系统
- V8 引擎的高性能

### 2. 为什么使用 WebSocket？

- 双向通信
- 实时消息推送
- 低延迟
- 支持流式响应

### 3. 为什么用 SQLite？

- 零配置
- 单文件存储
- 向量搜索支持 (sqlite-vec)
- FTS5 全文搜索

### 4. 为什么自研插件系统？

- 统一接口
- 类型安全
- 灵活的扩展机制
- npm 发布支持

---

## 项目结构

```
openclaw/
├── src/                    # 核心源代码 (119 个目录)
│   ├── gateway/           # Gateway 控制平面
│   ├── channels/          # 通道抽象层
│   ├── agents/           # AI 智能体
│   ├── memory/           # 记忆系统
│   ├── plugin-sdk/       # 插件 SDK
│   ├── plugins/          # 插件运行时
│   ├── config/           # 配置管理
│   ├── infra/           # 基础设施
│   ├── media/           # 媒体处理
│   ├── cli/             # CLI 入口
│   ├── commands/        # CLI 命令
│   ├── web/             # Web UI
│   ├── tui/             # 终端 UI
│   ├── telegram/         # Telegram 集成
│   ├── discord/         # Discord 集成
│   ├── slack/           # Slack 集成
│   ├── signal/          # Signal 集成
│   ├── whatsapp/        # WhatsApp 集成
│   └── ...
├── apps/                  # 客户端应用
│   ├── macos/           # macOS 应用
│   ├── ios/             # iOS 应用
│   ├── android/         # Android 应用
│   └── shared/          # 共享代码
├── extensions/            # 插件)
├── ui/扩展 (35+                   # 前端 UI
├── packages/            # 独立包
├── docs/                # 文档
├── scripts/             # 构建/部署脚本
└── test/               # 测试文件
```

---

## 关键依赖

### 运行时依赖

| 包 | 版本 | 用途 |
|---|------|------|
| `@mariozechner/pi-agent-core` | 0.57.1 | Agent 核心 |
| `grammy` | ^1.41.1 | Telegram 框架 |
| `@slack/bolt` | ^4.6.0 | Slack 框架 |
| `@whiskeysockets/baileys` | 7.0.0-rc.9 | WhatsApp 框架 |
| `hono` | 4.12.7 | Web 服务器 |
| `sqlite-vec` | 0.1.7-alpha.2 | 向量搜索 |
| `sharp` | ^0.34.5 | 图片处理 |
| `playwright-core` | 1.58.2 | 浏览器自动化 |

### 开发依赖

| 包 | 版本 | 用途 |
|---|------|------|
| `typescript` | ^5.9.3 | 类型系统 |
| `vitest` | ^4.0.18 | 测试框架 |
| `oxlint` | ^1.51.0 | 代码检查 |
| `oxfmt` | 0.36.0 | 代码格式化 |
| `tsdown` | 0.21.0 | 构建工具 |

---

## 构建和运行

### 开发环境

```bash
# 安装依赖
pnpm install

# 构建
pnpm build

# 运行 Gateway
pnpm gateway:dev

# 运行 CLI
pnpm openclaw onboard
```

### 测试

```bash
# 单元测试
pnpm test

# E2E 测试
pnpm test:e2e

# 实时测试
pnpm test:live

# Docker 测试
pnpm test:docker:all
```

---

## 总结

OpenClaw 是一个**成熟的大型多平台 AI 助手项目**，具有以下核心特点：

1. **多通道支持**: 支持 20+ 消息平台，采用适配器模式统一管理
2. **多平台客户端**: macOS、iOS、Android 原生应用
3. **可扩展架构**: 35+ 官方插件，Plugin SDK 支持自定义开发
4. **本地优先**: 保护用户隐私，所有数据存储在本地
5. **模块化设计**: 清晰的职责分离，易于维护和扩展

**技术亮点**：
- 混合搜索 (向量 + 关键词 + MMR + 时间衰减)
- 双后端支持 (Builtin + QMD)
- 6 种嵌入提供商
- 配置热重载
- 通道自动恢复

项目采用现代 TypeScript 开发实践，使用 monorepo 结构管理多个子模块，具有完善的测试覆盖和代码质量保障。
