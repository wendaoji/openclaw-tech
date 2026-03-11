# OpenClaw Infra 模块技术原理详解

## 1. 模块概述

Infra 模块是 OpenClaw 的基础设施层，提供系统级的底层功能，包括二进制管理、环境处理、备份、心跳、服务发现等。

**核心位置**: `src/infra/` (313 个文件)

## 2. 核心组件

### 2.1 环境管理

```typescript
// env.ts
export function normalizeEnv(): void;
export function getEnv(key: string): string | undefined;
```

### 2.2 二进制管理

```typescript
// binaries.ts
export function ensureBinary(name: string): Promise<string>;
```

### 2.3 心跳系统

```typescript
// heartbeat-runner.ts
export function startHeartbeatRunner(intervalMs: number): void;
```

### 2.4 服务发现 (Bonjour/mDNS)

```typescript
// bonjour.ts
export function startBonjourDiscovery(): void;
export function stopBonjourDiscovery(): void;
```

### 2.5 备份系统

```typescript
// backup-create.ts
export async function createBackup(params: BackupParams): Promise<BackupResult>;
```

### 2.6 归档管理

```typescript
// archive.ts
export class ArchiveManager {
  create(path: string): Promise<void>;
  extract(archive: string, dest: string): Promise<void>;
}
```

### 2.7 重启管理

```typescript
// restart.ts
export function setGatewaySigusr1RestartPolicy(policy: RestartPolicy): void;
```

### 2.8 密钥文件

```typescript
// secret-file.ts
export function loadSecretFileSync(path: string): SecretFile;
```

## 3. 文件结构

| 目录 | 功能 |
|------|------|
| `*.ts` | 核心基础设施 |
| `binaries/` | 二进制管理 |
| `bonjour*.ts` | mDNS 服务发现 |
| `backup*.ts` | 备份系统 |
| `archive*.ts` | 归档管理 |
| `secrets/` | 密钥管理 |
| `exec-approval*.ts` | 执行审批 |

## 4. 总结

Infra 模块提供 OpenClaw 运行所需的基础设施支持，包括环境管理、服务发现、备份、心跳和重启机制等系统级功能。
