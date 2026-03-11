# OpenClaw CLI/Commands 模块技术原理详解

## 1. 模块概述

CLI/Commands 模块是 OpenClaw 的命令行界面，提供用户与系统交互的入口点。

**核心位置**: 
- `src/cli/` (168 个文件)
- `src/commands/` (299 个文件)

## 2. CLI 架构

### 2.1 程序入口

```typescript
// program.ts
export function buildProgram(): Command {
  return program
    .name("openclaw")
    .description("Multi-channel AI assistant")
    .version(version);
}
```

### 2.2 命令列表

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

## 3. 核心命令

### 3.1 Agent 命令

```typescript
// agent.ts
export async function runAgentCommand(
  message: string,
  options: AgentOptions
): Promise<void> {
  // 连接 Gateway
  // 发送消息
  // 接收响应
}
```

### 3.2 Send 命令

```typescript
// send.ts
export async function runSendCommand(
  to: string,
  message: string,
  options: SendOptions
): Promise<void> {
  // 解析目标
  // 发送消息
}
```

### 3.3 Onboard 命令

```typescript
// onboard.ts
export async function runOnboardWizard(
  options: OnboardOptions
): Promise<void> {
  // 欢迎界面
  // 配置 Gateway
  // 配置通道
  // 安装技能
}
```

## 4. 依赖注入

```typescript
// deps.ts
export function createDefaultDeps(): CliDeps {
  return {
    gatewayUrl: "http://localhost:18789",
    http: fetch,
    // ...
  };
}
```

## 5. 文件结构

### CLI 目录

| 文件 | 功能 |
|------|------|
| `program.ts` | 程序入口 |
| `deps.ts` | 依赖注入 |
| `banner.ts` | 启动横幅 |
| `argv.ts` | 参数解析 |

### Commands 目录

| 目录 | 功能 |
|------|------|
| `agent/` | Agent 命令 |
| `agents/` | 代理管理 |
| `channels/` | 通道管理 |
| `skills/` | 技能管理 |
| `config/` | 配置命令 |
| `onboard/` | 初始化向导 |

## 6. 总结

CLI/Commands 模块是 OpenClaw 的用户交互入口，提供完整的命令行界面来管理 Gateway、通道、技能和配置。
