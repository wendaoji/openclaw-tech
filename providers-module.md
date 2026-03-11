# OpenClaw Providers 模块技术原理详解

## 1. 模块概述

Providers 模块负责 LLM 提供商的认证和模型配置。主要包括 GitHub Copilot 和其他 OAuth 认证流程。

**核心位置**: `src/providers/` (13 个文件)

## 2. 核心组件

### 2.1 GitHub Copilot Provider

```typescript
// github-copilot-auth.ts
export type GitHubCopilotAuth = {
  token: string;
  refreshToken: string;
  expiresAt: number;
};
```

### 2.2 模型列表

```typescript
// github-copilot-models.ts
export const GITHUB_COPILOT_MODELS = [
  "gpt-4o",
  "gpt-4o-mini",
  // ...
];
```

### 2.3 Google Shared

```typescript
// google-shared.test-helpers.ts
export type GoogleAuthConfig = {
  projectId: string;
  location: string;
  credentials: GoogleCredentials;
};
```

## 3. 认证流程

### 3.1 OAuth 认证

```typescript
// qwen-portal-oauth.ts
export async function authenticateWithOAuth(
  provider: string,
  clientId: string,
  clientSecret: string
): Promise<AuthToken>;
```

## 4. 总结

Providers 模块相对较小，主要负责第三方 LLM 服务的认证集成。
