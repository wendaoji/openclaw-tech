# OpenClaw Web/TUI 模块技术原理详解

## 1. 模块概述

Web/TUI 模块提供 OpenClaw 的两种用户界面：
- **Web UI**: 基于浏览器的控制界面
- **TUI**: 终端用户界面

**核心位置**: 
- `src/web/` (42 个文件)
- `src/tui/` (33 个文件)

## 2. Web UI

### 2.1 控制界面

```typescript
// accounts.ts
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
// auto-reply.ts
export async function startAutoReply(
  config: AutoReplyConfig
): Promise<void> {
  // 监听消息
  // 调用 AI
  // 发送回复
}
```

### 2.3 入站处理

```typescript
// inbound/index.ts
export async function handleInboundMessage(
  message: InboundMessage
): Promise<void> {
  // 解析消息
  // 路由到 AI
  // 发送回复
}
```

## 3. TUI (终端界面)

### 3.1 命令处理

```typescript
// tui-command-handlers.ts
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
// tui-event-handlers.ts
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
// tui-formatters.ts
export function formatMessage(message: Message): string {
  // 格式化消息显示
}

export function formatChannelStatus(status: ChannelStatus): string {
  // 格式化通道状态
}
```

### 3.4 主题支持

```typescript
// theme/index.ts
export type Theme = {
  colors: ColorPalette;
  fonts: FontConfig;
};

export function loadTheme(name: string): Theme;
```

## 4. 文件结构

### Web 目录

| 文件 | 功能 |
|------|------|
| `accounts.ts` | 账户管理 |
| `auto-reply.ts` | 自动回复 |
| `inbound/` | 入站处理 |
| `auth-store.ts` | 认证存储 |

### TUI 目录

| 文件 | 功能 |
|------|------|
| `commands.ts` | 命令处理 |
| `tui-command-handlers.ts` | 命令处理器 |
| `tui-event-handlers.ts` | 事件处理器 |
| `tui-formatters.ts` | 格式化器 |
| `theme/` | 主题支持 |
| `components/` | UI 组件 |

## 5. 总结

Web/TUI 模块提供 OpenClaw 的两种用户界面：Web UI 用于浏览器控制，TUI 用于终端交互。两者都通过 Gateway 进行通信。
