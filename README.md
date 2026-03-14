# OpenClaw

**运行在本地设备上的多通道 AI 网关**

---

## 一句话概述

OpenClaw 是一个运行在用户本地设备上的多通道 AI 网关，能够在 WhatsApp、Telegram、Slack、Discord、Signal、iMessage 等 20+ 消息平台上提供统一的 AI 助手服务。

---

## 核心特性

| 特性 | 说明 |
|------|------|
| 多通道 | 支持 20+ 消息平台 |
| 本地运行 | 数据存储在本地，保护隐私 |
| AI 驱动 | 基于 Pi Agent Core |
| 可扩展 | 35+ 官方插件 |
| 开源 | MIT 许可证 |

---

## 技术原理

### 系统架构

```
用户 (WhatsApp/Telegram/Slack/...)
        │
        ▼
┌─────────────────────────────┐
│   Gateway (网关层)           │
│   ws://localhost:18789      │
│   • 会话管理  • 消息路由     │
│   • 认证授权  • 定时任务    │
└────────────┬────────────────┘
             │
    ┌────────┼────────┬──────────┐
    ▼        ▼        ▼          ▼
Channels   Agents   Memory    Config
(通道层)   (智能体)  (记忆)    (配置)
    │        │        │          │
    ▼        ▼        ▼          ▼
消息平台    LLM     向量存储   JSON5
```

### 核心模块

1. **Gateway** - 系统心脏，负责协调所有组件，监听 WebSocket 连接，管理会话和消息路由

2. **Channels** - 消息通道，支持 Telegram、Discord、Slack、WhatsApp 等 20+ 平台

3. **Agents** - AI 智能体，基于 Pi Agent Core 处理对话、调用工具、管理记忆

4. **Memory** - 记忆系统，结合向量搜索和全文搜索实现语义检索

5. **Plugin-SDK** - 插件开发套件，支持自定义通道、技能、记忆后端

### 消息处理流程

```
用户消息 → 通道接收 → Gateway 认证 → 消息解析 → 会话解析
    → Agent 处理 (构建提示词 → LLM推理 → 工具执行) 
    → 记忆检索 → 生成回复 → 发送到通道 → 消息确认
```

---

## 快速开始

```bash
# 安装
npm install -g openclaw@latest

# 初始化
openclaw onboard --install-daemon

# 启动 Gateway
openclaw gateway --port 18789

# 与 AI 对话
openclaw agent --message "你好"
```

---

## 文档目录

| 文档 | 说明 |
|------|------|
| [完整文档](docs/README.md) | 完整技术文档 |
| [系统架构](docs/architecture.md) | 技术架构总览 |
| [Gateway](docs/gateway-module.md) | 网关模块详解 |
| [Channels](docs/channels-module.md) | 通道模块详解 |
| [Agents](docs/agents-module.md) | 智能体模块详解 |
| [Memory](docs/memory-module.md) | 记忆模块详解 |
| [Plugin SDK](docs/plugin-sdk-module.md) | 插件 SDK 详解 |
| [Config](docs/config-module.md) | 配置模块详解 |
| [CLI](docs/cli-commands-module.md) | CLI 命令详解 |
| [Media](docs/media-module.md) | 媒体处理详解 |
| [Web TUI](docs/web-tui-module.md) | 用户界面详解 |

或访问在线文档：[https://wendaoji.github.io/openclaw-tech/](https://wendaoji.github.io/openclaw-tech/)

## Docsify 文档站点

项目已接入 Docsify，文档入口在 `docs/index.html`。

本地预览：

```bash
npx docsify-cli serve docs
```

默认访问地址：

```text
http://localhost:3000
```

---

## License

MIT
