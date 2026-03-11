# OpenClaw Media 模块技术原理详解

## 1. 模块概述

Media 模块负责 OpenClaw 的媒体处理，包括图片、音频、视频的转码、压缩、缩放和提取。

**核心位置**: `src/media/` (42 个文件)

## 2. 核心组件

### 2.1 图片处理

```typescript
// image-ops.ts
export async function resizeImage(
  input: Buffer,
  options: ResizeOptions
): Promise<Buffer>;

export async function compressImage(
  input: Buffer,
  quality: number
): Promise<Buffer>;

export async function convertImageFormat(
  input: Buffer,
  format: "jpeg" | "png" | "webp"
): Promise<Buffer>;
```

### 2.2 音频处理

```typescript
// audio.ts
export async function transcodeAudio(
  input: Buffer,
  format: AudioFormat
): Promise<Buffer>;

export async function extractAudio(
  videoPath: string
): Promise<Buffer>;
```

### 2.3 FFmpeg 集成

```typescript
// ffmpeg-exec.ts
export async function runFFmpeg(
  args: string[],
  input: Buffer
): Promise<Buffer>;
```

### 2.4 媒体获取

```typescript
// fetch.ts
export async function fetchMedia(
  url: string,
  options: MediaFetchOptions
): Promise<MediaPayload>;
```

### 2.5 MIME 类型检测

```typescript
// mime.ts
export function detectMimeType(buffer: Buffer): string;
export function isSupportedMedia(mimeType: string): boolean;
```

### 2.6 临时文件管理

```typescript
// temp-path.ts
export function createTempMediaPath(extension: string): string;
export function cleanupTempMedia(maxAgeMs: number): Promise<void>;
```

## 3. 文件结构

| 文件 | 功能 |
|------|------|
| `image-ops.ts` | 图片处理 |
| `audio.ts` | 音频处理 |
| `ffmpeg-exec.ts` | FFmpeg 执行器 |
| `fetch.ts` | 媒体获取 |
| `mime.ts` | MIME 类型检测 |
| `input-files.ts` | 输入文件处理 |
| `parse.ts` | 媒体解析 |

## 4. 总结

Media 模块处理 OpenClaw 中的所有媒体文件，支持图片压缩、音频转码、视频提取等。
