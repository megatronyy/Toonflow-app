# Vendor（AI 供应商适配器）系统

## 概述

`data/vendor/` 目录存放了 Toonflow 的 AI 供应商适配器插件，每份 `.ts` 文件封装了一个 AI 平台的 API 调用逻辑。这些文件是**运行时加载**的 TypeScript 脚本，通过 VM2 沙箱执行，无需重新编译应用即可添加或修改 AI 供应商。

## 文件清单

| 文件 | 供应商 | 支持能力 | 版本 |
|------|--------|----------|------|
| `toonflow.ts` | Toonflow 官方中转平台 | 文本、图片、视频 | 2.0 |
| `grsai.ts` | Grsai AI 平台 | 文本、图片、视频 | 2.0 |
| `klingai.ts` | 可灵 AI（快手） | 视频（仅视频） | 2.0 |
| `minimax.ts` | MiniMax（海螺 AI） | 文本、图片、视频 | 2.1 |
| `volcengine.ts` | 火山引擎（豆包） | 文本、图片、视频 | 2.1 |
| `vidu.ts` | Vidu 开放平台 | 图片、视频 | — |
| `openai.ts` | OpenAI 标准接口 | 文本（仅文本） | 2.0 |
| `null.ts` | 空模板（开发用） | 文本（占位） | 2.0 |

## 在项目中的角色

### 插件式供应商架构

Vendor 文件构成了一个**可插拔的 AI 供应商系统**：

1. **`data/vendor/*.ts`** — 供应商适配器脚本（本目录）
2. **`src/utils/vendor.ts`** — 读取/写入/加载供应商代码
3. **`src/utils/vm.ts`** — VM2 沙箱执行引擎，注入运行时依赖
4. **`src/utils/ai.ts`** — AI 编排层，调用供应商导出函数

### 运行时加载流程

```
用户请求 → ai.ts 获取供应商代码 → vm.ts 沙箱执行 → 注入依赖（axios/createOpenAI/...）
→ 调用 exports.textRequest/imageRequest/videoRequest → 返回结果
```

具体步骤：
1. `getCode()` 读取 `data/vendor/{id}.ts` 文件
2. `sucrase.transform()` 将 TypeScript 编译为 JavaScript
3. `u.vm()` 在 VM2 沙箱中执行，注入全局依赖
4. 沙箱返回 `exports` 对象，其中包含供应商配置和适配器函数
5. `ai.ts` 根据请求类型调用对应的适配器函数

## 文件结构

每个 vendor 文件遵循统一的结构：

```typescript
// ============================================================
// 类型定义（每个文件都包含完整的类型声明）
// ============================================================
type VideoMode = "singleImage" | "startEndRequired" | "endFrameOptional" | ...;
interface TextModel { ... }
interface ImageModel { ... }
interface VideoModel { ... }
interface VendorConfig { ... }
interface ImageConfig { ... }
interface VideoConfig { ... }
// ...

// ============================================================
// 全局声明（运行时注入的依赖，在沙箱中可用）
// ============================================================
declare const axios: any;
declare const logger: (msg: string) => void;
declare const zipImage: (base64: string, size: number) => Promise<string>;
declare const urlToBase64: (url: string) => Promise<string>;
declare const pollTask: (fn, interval?, timeout?) => Promise<PollResult>;
declare const createOpenAI: any;
declare const createAnthropic: any;
// ... 其他 AI SDK 工厂函数
declare const exports: { ... };

// ============================================================
// 供应商配置
// ============================================================
const vendor: VendorConfig = {
  id: "唯一标识",
  version: "x.y",
  name: "显示名称",
  inputs: [...],       // 用户需要填写的配置项（API Key、Base URL 等）
  inputValues: {...},   // 默认值
  models: [...],        // 支持的模型列表
};

// ============================================================
// 辅助工具（可选，多个适配器函数共享的逻辑）
// ============================================================
// getHeaders(), getBaseUrl(), generateAuthToken() 等

// ============================================================
// 适配器函数
// ============================================================
const textRequest = (model, think, thinkLevel) => { ... };
const imageRequest = async (config, model) => { ... };
const videoRequest = async (config, model) => { ... };
const ttsRequest = async (config, model) => { ... };

// ============================================================
// 导出
// ============================================================
exports.vendor = vendor;
exports.textRequest = textRequest;
exports.imageRequest = imageRequest;
exports.videoRequest = videoRequest;
exports.ttsRequest = ttsRequest;
export {};
```

## 核心概念

### VendorConfig（供应商配置）

```typescript
{
  id: string;          // 唯一 ID，作为文件名存储，如 "minimax"
  version: string;     // 语义化版本 "x.y"
  name: string;        // 显示名称
  author: string;      // 作者
  description?: string; // 描述（支持 Markdown）
  icon?: string;       // 图标（Base64，128x128）
  inputs: [...];       // 配置表单字段定义
  inputValues: {...};  // 配置默认值
  models: [...];       // 模型列表
}
```

### 四种模型类型

1. **TextModel** — 文本/对话模型，`think` 标记是否支持推理模式
2. **ImageModel** — 图片生成模型，`mode` 声明支持 `text`（文生图）/ `singleImage`（图生图）/ `multiReference`（多参考）
3. **VideoModel** — 视频生成模型，`mode` 声明输入模式（文生视频、单图、首尾帧、多参考等）
4. **TTSModel** — 语音合成模型（暂未开放）

### VideoMode（视频输入模式）

| 模式 | 含义 |
|------|------|
| `"text"` | 纯文本生成视频 |
| `"singleImage"` | 单张图片作为首帧 |
| `"startEndRequired"` | 首尾帧都必须提供 |
| `"endFrameOptional"` | 尾帧可选 |
| `"startFrameOptional"` | 首帧可选 |
| `["imageReference:3", "videoReference:1"]` | 多模态参考，数字为最大数量 |

### 四个适配器函数

| 函数 | 返回值 | 说明 |
|------|--------|------|
| `textRequest(model, think, thinkLevel)` | AI SDK 的 chat model 实例 | 通过 `createOpenAI` 等工厂函数创建，返回值供 Vercel AI SDK 的 `streamText`/`generateText` 消费 |
| `imageRequest(config, model)` | `string`（有头 base64） | 图片生成，返回 `data:image/png;base64,...` 格式 |
| `videoRequest(config, model)` | `string`（有头 base64） | 视频生成，通常先提交任务再轮询，最终返回 `data:video/mp4;base64,...` 格式 |
| `ttsRequest(config, model)` | `string`（有头 base64） | 语音合成，暂未开放 |

### ReferenceList（统一引用类型）

```typescript
type ReferenceList =
  | { type: "image"; sourceType: "base64"; base64: string }
  | { type: "audio"; sourceType: "base64"; base64: string }
  | { type: "video"; sourceType: "base64"; base64: string };
```

所有媒体引用统一以 base64 形式传入，供应商适配器负责将其转换为各平台所需的格式。

## 运行时注入的依赖

以下全局变量在沙箱执行时自动注入，vendor 代码可直接使用：

| 依赖 | 说明 |
|------|------|
| `axios` | HTTP 请求库 |
| `logger(msg)` | 日志输出函数 |
| `jsonwebtoken` | JWT 签名/验证（可灵 AI 使用） |
| `zipImage(base64, sizeKB)` | 压缩图片体积 |
| `zipImageResolution(base64, w, h)` | 调整图片分辨率 |
| `mergeImages(base64Arr, maxSize?)` | 多图拼接为一张 |
| `urlToBase64(url)` | URL 下载后转 base64 |
| `pollTask(fn, interval?, timeout?)` | 异步任务轮询 |
| `createOpenAI(options)` | OpenAI SDK 工厂 |
| `createAnthropic(options)` | Anthropic SDK 工厂 |
| `createDeepSeek(options)` | DeepSeek SDK 工厂 |
| `createZhipu(options)` | 智谱 SDK 工厂 |
| `createQwen(options)` | 通义千问 SDK 工厂 |
| `createMinimax(options)` | MiniMax SDK 工厂 |
| `createGoogleGenerativeAI(options)` | Google Gemini SDK 工厂 |
| `createXai(options)` | xAI SDK 工厂 |
| `createOpenAICompatible(options)` | OpenAI 兼容格式 SDK 工厂 |

## 各供应商适配器详解

### 1. toonflow.ts — Toonflow 官方中转平台

- **认证**：API Key（Bearer Token）
- **文本模型**：Claude 系列、GPT 系列、MiniMax 系列（通过 OpenAI 兼容接口）
- **图片模型**：豆包 Seedream 系列（走 `images/generations` 接口）、Gemini/nano 系列（走 `chat/completions` 接口，从 Markdown 响应中提取图片）
- **视频模型**：万象 Wan2.6（走 `video/generations`，含首尾帧）、豆包 Seedance（含音频生成）、Vidu 系列
- **特殊逻辑**：根据模型名称自动选择不同的 API 适配路径

### 2. grsai.ts — Grsai AI 平台

- **认证**：API Key（Bearer Token）
- **文本模型**：通过 Google Generative AI SDK 适配（`/v1beta` 端点）
- **图片模型**：Nano Banana 系列（走 `/v1/draw/nano-banana` 提交 + `/v1/draw/result` 轮询）
- **视频模型**：通过 `/v1/video/veo` 提交 + `/v1/draw/result` 轮询，支持首尾帧和多图参考

### 3. klingai.ts — 可灵 AI（快手）

- **认证**：Access Key + Secret Key → JWT Token（HS256 签名，`jsonwebtoken`）
- **仅支持视频**：全系列可灵模型（v1 ~ video-o1）
- **三种 API 路径**：
  - Omni 模型（video-o1, v3-omni）→ `/v1/videos/omni-video`
  - 多图参考（v1-6）→ `/v1/videos/multi-image2video`
  - 文生视频 → `/v1/videos/text2video`
  - 图生视频 → `/v1/videos/image2video`
- **公共逻辑**：`submitAndPoll()` 统一处理提交+轮询

### 4. minimax.ts — MiniMax（海螺 AI）

- **认证**：API Key（Bearer Token）
- **文本模型**：MiniMax-M2 全系列（通过 OpenAI 兼容接口，支持推理模式 `reasoning_split`）
- **图片模型**：海螺图像 V1/V1-Live（走 `/v1/image_generation`，支持角色参考图）
- **视频模型**：海螺 2.3/02（走 `/v1/video_generation` 提交 → `/v1/query/video_generation` 轮询 → `/v1/files/retrieve` 获取下载地址）
- **额外导出**：`uploadReference` 前置处理器，压缩图片后以 base64 传递

### 5. volcengine.ts — 火山引擎（豆包）

- **认证**：API Key（Bearer Token）
- **文本模型**：Doubao-Seed 全系列 + 第三方托管模型（DeepSeek、GLM、Qwen 等），通过 OpenAI 兼容接口，自定义 `fetch` 注入 `thinking` 和 `reasoning_effort` 参数
- **图片模型**：Seedream 3.0/4.0/4.5/5.0 系列（走 `images/generations` 接口），含详细的像素尺寸映射表
- **视频模型**：Seedance 全系列（走 `contents/generations/tasks` 提交 + 轮询），支持多模态参考（图片/视频/音频）

### 6. vidu.ts — Vidu 开放平台

- **认证**：API Key（`Token` 前缀）
- **图片模型**：ViduQ1/Q2（走 `/reference2image` 提交 + 轮询）
- **视频模型**：ViduQ 全系列 + Vidu 2.0（走 `/start-end2video` 提交 + `/tasks/{id}/creations` 轮询）
- **特殊点**：较早版本的适配器，类型定义略有差异（`VideoMode` 用 `"videoReference"` 而非 `"videoReference:N"`），使用 `fetch` 而非 `axios`

### 7. openai.ts — OpenAI 标准接口

- **认证**：API Key（Bearer Token）
- **仅文本模型**：GPT-4o/4.1/5.x 系列
- **通用适配器**：支持任何 OpenAI 兼容的 API 端点，用户可自定义 baseUrl

### 8. null.ts — 空模板（开发模板）

- **用途**：新供应商开发的起始模板，包含完整的类型定义和代码生成指南
- **底部附有详细的 AI 代码生成规则**（13 条规则），指导 AI 如何基于此模板生成新的供应商适配器

## 异步任务处理模式

图片和视频生成通常采用**提交-轮询**模式：

```typescript
// 1. 提交任务
const submitResp = await axios.post(submitUrl, requestBody, { headers });
const taskId = submitResp.data.task_id;

// 2. 轮询结果
const result = await pollTask(async () => {
  const resp = await axios.get(queryUrl, { headers });
  if (status === "SUCCESS") return { completed: true, data: url };
  if (status === "FAILED") return { completed: true, error: "..." };
  return { completed: false };
}, 5000, 600000); // 每5秒轮询，10分钟超时

// 3. 转换结果
return await urlToBase64(result.data);
```

## 关键文件引用

| 文件 | 作用 |
|------|------|
| `data/vendor/*.ts` | 供应商适配器脚本（本文档描述的目录） |
| `src/utils/vendor.ts` | 读写和加载供应商代码 |
| `src/utils/vm.ts` | VM2 沙箱执行引擎 |
| `src/utils/ai.ts` | AI 编排层，调用供应商适配器函数 |
| `src/agents/productionAgent/` | 使用供应商模型进行图片/视频生成 |

## 扩展指南

添加新供应商的步骤：

1. 复制 `null.ts` 模板，重命名为 `{vendorId}.ts`
2. 填写 `VendorConfig`（id、name、inputs、models）
3. 实现需要的适配器函数（textRequest/imageRequest/videoRequest）
4. 未实现的函数保留空实现 `return ""`
5. 遵守模板底部的代码规则（禁止引入外部包、小驼峰命名、通过 `exports` 导出）
6. 文件放入 `data/vendor/` 目录，在设置界面配置 API 密钥即可使用
