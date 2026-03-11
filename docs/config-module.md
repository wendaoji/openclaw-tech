# OpenClaw Config 模块技术原理详解

## 1. 模块概述

Config 模块负责 OpenClaw 的配置管理，包括配置文件加载、验证、迁移和运行时更新。

**核心位置**: `src/config/` (208 个文件)

## 2. 核心组件

### 2.1 配置加载

```typescript
// io.ts
export async function loadConfig(): Promise<OpenClawConfig>;
export async function readConfigFileSnapshot(): Promise<ConfigSnapshot>;
```

### 2.2 配置验证

```typescript
// validation.ts
export function validateConfigObject(
  config: unknown
): { ok: true; value: OpenClawConfig } | { ok: false; errors: string[] };
```

### 2.3 配置迁移

```typescript
// legacy-migrate.ts
export function migrateLegacyConfig(
  config: unknown
): { config: OpenClawConfig | null; changes: string[] };
```

## 3. 配置结构

```typescript
// types.ts
export type OpenClawConfig = {
  // Gateway 配置
  gateway?: GatewayConfig;
  
  // 模型配置
  models?: ModelsConfig;
  
  // 通道配置
  channels?: ChannelsConfig;
  
  // 技能配置
  skills?: SkillsConfig;
  
  // 记忆配置
  memory?: MemoryConfig;
  
  // 安全配置
  security?: SecurityConfig;
  
  // 密钥
  secrets?: SecretsConfig;
};
```

## 4. 配置类型

### 4.1 Gateway 配置

```typescript
export type GatewayConfig = {
  bind?: "loopback" | "lan" | "tailnet" | "auto";
  port?: number;
  auth?: GatewayAuthConfig;
  controlUi?: ControlUiConfig;
};
```

### 4.2 模型配置

```typescript
export type ModelsConfig = {
  default?: string;
  providers?: Record<string, ProviderConfig>;
};
```

### 4.3 通道配置

```typescript
export type ChannelsConfig = Partial<Record<ChannelId, ChannelConfig>>;
```

## 5. 运行时配置

### 5.1 配置快照

```typescript
// runtime-overrides.ts
export function getRuntimeConfigSnapshot(): OpenClawConfig;
export function setRuntimeConfigSnapshot(config: OpenClawConfig): void;
```

### 5.2 热重载

```typescript
export function setRuntimeConfigSnapshotRefreshHandler(
  handler: () => Promise<void>
): void;
```

## 6. 配置验证

### 6.1 JSON Schema

配置使用 JSON Schema 进行验证：

```typescript
// types.ts
export const CONFIG_SCHEMA: TSchema;
```

### 6.2 允许值

```typescript
// allowed-values.ts
export const ALLOWED_VALUES = {
  gateway: {
    bind: ["loopback", "lan", "tailnet", "auto"],
    auth: {
      mode: ["none", "token", "token-or-anonymous", "passcode"],
    },
  },
};
```

## 7. 文件结构

| 文件 | 功能 |
|------|------|
| `config.ts` | 导出入口 |
| `io.ts` | 配置文件 I/O |
| `validation.ts` | 配置验证 |
| `legacy-migrate.ts` | 旧配置迁移 |
| `types.ts` | 类型定义 |
| `paths.ts` | 路径解析 |
| `runtime-overrides.ts` | 运行时覆盖 |

## 8. 总结

Config 模块负责 OpenClaw 的配置管理，支持 JSON5 格式、配置验证、迁移和热重载。
