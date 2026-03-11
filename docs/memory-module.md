# OpenClaw Memory 模块技术原理详解

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
┌─────────────────────────────────────────────────────────────┐
│                     MemorySearchManager                      │
│                   (search-manager.ts)                       │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐    ┌──────────────────────────────┐  │
│  │  QMD Backend    │    │    Builtin Backend          │  │
│  │ (qmd-manager)  │───▶│  (MemoryIndexManager)       │  │
│  └─────────────────┘    └──────────────────────────────┘  │
│                                     │                       │
│                                     ▼                       │
│                         ┌───────────────────────────┐       │
│                         │   Hybrid Search Engine    │       │
│                         ├───────────────────────────┤       │
│                         │  ┌─────────┐ ┌─────────┐  │       │
│                         │  │ Vector  │ │   FTS   │  │       │
│                         │  │ Search  │ │ Search  │  │       │
│                         │  └─────────┘ └─────────┘  │       │
│                         │        │        │         │       │
│                         │        ▼        ▼         │       │
│                         │   MMR + Temporal Decay   │       │
│                         └───────────────────────────┘       │
└─────────────────────────────────────────────────────────────┘
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
// embeddings.ts
export async function createEmbeddingProvider(options) {
  // 自动选择提供商
  if (options.provider === "auto") {
    // 尝试 OpenAI → Gemini → Voyage → Mistral
  }
  
  // 回退机制
  if (primaryProvider失败) {
    return fallbackProvider;
  }
}
```

### 4.3 本地嵌入模型

使用 `node-llama-cpp` 实现本地嵌入：

```typescript
// embeddings.ts
async function createLocalEmbeddingProvider(options) {
  const { getLlama } = await importNodeLlamaCpp();
  const llama = await getLlama();
  const model = await llama.loadModel({ modelPath });
  const context = await model.createEmbeddingContext();
  
  return {
    embedQuery: async (text) => {
      const embedding = await context.getEmbeddingFor(text);
      return normalize(embedding.vector);
    }
  };
}
```

### 4.4 向量归一化

```typescript
function sanitizeAndNormalizeEmbedding(vec: number[]): number[] {
  const sanitized = vec.map((value) => (Number.isFinite(value) ? value : 0));
  const magnitude = Math.sqrt(sanitized.reduce((sum, v) => sum + v * v, 0));
  if (magnitude < 1e-10) return sanitized;
  return sanitized.map((value) => value / magnitude);
}
```

## 5. 混合搜索机制

### 5.1 搜索策略

`hybrid.ts` 实现了混合搜索，结合向量相似度和 BM25 关键词匹配：

```typescript
// 混合搜索核心逻辑
score = vectorWeight * vectorScore + textWeight * textScore;
```

### 5.2 向量搜索

- 使用 `sqlite-vec` 存储向量
- 支持余弦相似度计算
- 批量嵌入处理

### 5.3 全文搜索 (FTS)

- 使用 SQLite FTS5
- 支持查询扩展
- 多语言停用词过滤

### 5.4 查询扩展

`query-expansion.ts` 实现智能关键词提取：

- 移除停用词 (英语、中文、西班牙语等)
- 提取有意义的关键词
- 支持多语言

```typescript
// STOP_WORDS_EN 示例
const STOP_WORDS_EN = new Set([
  "a", "an", "the", "this", "that",
  "i", "me", "my", "we", "our",
  "is", "are", "was", "were",
  // ... 更多停用词
]);
```

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

**实现细节**:
- 使用 Jaccard 相似度计算文本相似性
- 迭代选择最大 MMR 分数的结果

```typescript
export function jaccardSimilarity(setA: Set<string>, setB: Set<string>): number {
  if (setA.size === 0 && setB.size === 0) return 1;
  if (setA.size === 0 || setB.size === 0) return 0;
  
  let intersection = 0;
  for (const token of setA) {
    if (setB.has(token)) intersection++;
  }
  
  const union = setA.size + setB.size - intersection;
  return union === 0 ? 0 : intersection / union;
}
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
- 其他文件: 从 mtime 获取

```typescript
function isEvergreenMemoryPath(filePath: string): boolean {
  if (normalized === "MEMORY.md" || normalized === "memory.md") {
    return true;
  }
  if (!normalized.startsWith("memory/")) {
    return false;
  }
  return !DATED_MEMORY_PATH_RE.test(normalized);
}
```

## 7. 数据存储

### 7.1 SQLite 表结构

```sql
-- 向量表
CREATE VIRTUAL TABLE chunks_vec USING vec0(
  id TEXT PRIMARY KEY,
  path TEXT,
  start_line INTEGER,
  end_line INTEGER,
  source TEXT,
  content TEXT,
  embedding FLOAT[dimension]
);

-- FTS 表
CREATE VIRTUAL TABLE chunks_fts USING fts5(
  id,
  path,
  content,
  content='chunks',
  content_rowid='rowid'
);

-- 嵌入缓存
CREATE TABLE embedding_cache (
  hash TEXT PRIMARY KEY,
  embedding BLOB,
  created_at INTEGER
);
```

### 7.2 会话文件处理

`session-files.ts` 处理对话历史：

```typescript
// 解析 JSONL 会话文件
async function buildSessionEntry(absPath) {
  const lines = raw.split("\n");
  for (const line of lines) {
    const record = JSON.parse(line);
    if (record.type === "message") {
      const text = extractSessionText(record.message.content);
      // 敏感信息脱敏
      const safe = redactSensitiveText(text, { mode: "tools" });
    }
  }
}
```

## 8. QMD 后端 (可选)

`qmd-manager.ts` 提供 QMD 格式支持：

- **QMD**: Query-Memory-Digest 格式
- 支持 `qmd` 或 `mcporter` CLI 工具
- 更高级的搜索模式 (query, search, chat)

## 9. 配置选项

```typescript
// 记忆搜索配置
interface ResolvedMemorySearchConfig {
  provider: "auto" | "openai" | "gemini" | "voyage" | "mistral" | "ollama" | "local";
  model: string;
  vector: { enabled: boolean };
  fts: { enabled: boolean };
  hybrid: { vectorWeight: number; textWeight: number };
  mmr: { enabled: boolean; lambda: number };
  temporalDecay: { enabled: boolean; halfLifeDays: number };
  qmd?: ResolvedQmdConfig;
}
```

### 默认配置

```typescript
const DEFAULT_HYBRID = {
  vectorWeight: 0.7,
  textWeight: 0.3,
};

const DEFAULT_MMR_CONFIG = {
  enabled: false,
  lambda: 0.7,
};

const DEFAULT_TEMPORAL_DECAY_CONFIG = {
  enabled: false,
  halfLifeDays: 30,
};
```

## 10. 核心 API

```typescript
interface MemorySearchManager {
  // 语义搜索
  search(
    query: string,
    opts?: { maxResults?: number; minScore?: number; sessionKey?: string }
  ): Promise<MemorySearchResult[]>;
  
  // 读取文件内容
  readFile(params: { relPath: string; from?: number; lines?: number }): Promise<{ text: string }>;
  
  // 获取状态
  status(): MemoryProviderStatus;
  
  // 同步索引
  sync(params?: { force?: boolean; progress?: (update) => void }): Promise<void>;
  
  // 检查嵌入可用性
  probeEmbeddingAvailability(): Promise<{ ok: boolean; error?: string }>;
  
  // 检查向量可用性
  probeVectorAvailability(): Promise<boolean>;
  
  // 关闭管理器
  close?(): Promise<void>;
}
```

## 11. 设计亮点

### 11.1 双后端支持

- **Builtin 后端**: 内置 SQLite 向量搜索
- **QMD 后端**: 外部 qmd/mcporter CLI
- **自动降级**: QMD 失败时自动切换到 Builtin

### 11.2 混合搜索

- 向量相似度 + BM25 关键词匹配
- 可配置权重比例
- MMR 多样性重排序
- 时间衰减

### 11.3 多提供商

- 支持 6 种嵌入提供商
- 自动选择和回退
- 本地模型支持 (node-llama-cpp)

### 11.4 嵌入缓存

```typescript
const EMBEDDING_CACHE_TABLE = "embedding_cache";

// 避免重复计算嵌入
const cached = await getEmbeddingCache(hash);
if (cached) return cached;

const embedding = await provider.embedQuery(text);
await setEmbeddingCache(hash, embedding);
```

### 11.5 增量同步

- 文件监视 (chokidar)
- 定时同步
- 差量更新

## 12. 核心流程图

```
用户查询
    │
    ▼
Query Expansion (提取关键词)
    │
    ├─ Vector Search ──▶ 向量相似度评分
    │
    └─ FTS Search ─────▶ BM25 关键词评分
            │
            ▼
    Hybrid Merge (混合评分)
    score = vectorWeight × vectorScore + textWeight × textScore
            │
            ▼
    Temporal Decay (时间衰减)
    score = score × exp(-λ × ageInDays)
            │
            ▼
    MMR Re-rank (多样性排序)
    选择: λ × relevance - (1-λ) × maxSimilarity
            │
            ▼
    返回 Top-K 结果
```

## 13. 文件清单

### 核心文件

| 文件 | 行数 | 功能 |
|------|------|------|
| `manager.ts` | 802 | 核心管理器 |
| `qmd-manager.ts` | 2000+ | QMD 后端 |
| `embeddings.ts` | 322 | 嵌入提供者 |
| `hybrid.ts` | 155 | 混合搜索 |
| `mmr.ts` | 214 | MMR 重排序 |
| `temporal-decay.ts` | 167 | 时间衰减 |
| `query-expansion.ts` | 810 | 查询扩展 |
| `search-manager.ts` | 252 | 搜索抽象 |

### 嵌入提供者

| 文件 | 提供商 |
|------|--------|
| `embeddings-openai.ts` | OpenAI |
| `embeddings-gemini.ts` | Google Gemini |
| `embeddings-voyage.ts` | Voyage AI |
| `embeddings-mistral.ts` | Mistral |
| `embeddings-ollama.ts` | Ollama |
| `node-llama.ts` | Local (llama.cpp) |

### 批处理

| 文件 | 功能 |
|------|------|
| `batch-openai.ts` | OpenAI 批量嵌入 |
| `batch-gemini.ts` | Gemini 批量嵌入 |
| `batch-voyage.ts` | Voyage 批量嵌入 |
| `batch-runner.ts` | 批处理运行器 |

## 14. 总结

Memory 模块是 OpenClaw 系统中功能完善的记忆检索引擎，其设计特点包括：

1. **双后端架构**: Builtin + QMD，支持自动降级
2. **混合搜索**: 向量 + 关键词 + MMR + 时间衰减
3. **多提供商**: 6 种嵌入提供商，自动回退
4. **本地优先**: 支持本地模型，保护隐私
5. **增量同步**: 文件监视 + 定时更新 + 差量处理
6. **容错机制**: 嵌入缓存 + 失败重试 + 回退策略
