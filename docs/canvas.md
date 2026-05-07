# 画布（Production Canvas）逻辑分析

## 概述

画布是 Toonflow 的核心功能模块之一，负责从剧本到视频的完整制作流程。它是一个**无限画布**（Infinite Canvas），将**资产（Assets）、分镜（Storyboard）、视频轨道（VideoTrack）和视频（Video）** 组织在同一个工作区中，支持 AI Agent 自动化创作和用户手动编辑。

---

## 整体架构

```
前端 (Toonflow-web)
  │
  ├── Socket.IO (/api/socket/productionAgent)  ← AI Agent 实时交互
  │
  └── HTTP API (/api/production/...)            ← 数据 CRUD 操作
        │
        ├── /assets/*        ← 资产管理
        ├── /storyboard/*    ← 分镜管理
        ├── /workbench/*     ← 工作台（视频轨道 + 视频生成）
        ├── /editImage/*     ← 图片编辑工作流
        ├── /getFlowData     ← 获取完整画布数据
        └── /saveFlowData    ← 保存完整画布数据
              │
        ┌─────┴─────┐
        │  SQLite   │
        │ Database  │
        └───────────┘
```

---

## 核心数据模型

### 数据库表结构

| 表名 | 说明 | 关键字段 |
|------|------|----------|
| `o_storyboard` | 分镜 | id, scriptId, projectId, prompt, filePath, duration, state, trackId, track, videoDesc, shouldGenerateImage, flowId, index |
| `o_videoTrack` | 视频轨道 | id, videoId, projectId, scriptId, state, reason, prompt, duration |
| `o_video` | 视频 | id, filePath, errorReason, time, state, scriptId, projectId, videoTrackId |
| `o_assets` | 资产（角色/道具/场景） | id, name, type, prompt, describe, imageId, assetsId, projectId, flowId |
| `o_image` | 图片 | id, filePath, state, errorReason, assetsId, type, resolution, model |
| `o_assets2Storyboard` | 资产-分镜关联 | storyboardId, assetId (联合主键) |
| `o_scriptAssets` | 剧本-资产关联 | scriptId, assetId (联合主键) |
| `o_agentWorkData` | Agent 工作数据缓存 | id, projectId, episodesId, key, data |
| `o_imageFlow` | 图片编辑工作流 | id, flowData (JSON) |

### 实体关系

```
o_project ──1:N──> o_script (剧本)
                      │
                      ├── o_scriptAssets ──N:N──> o_assets (资产)
                      │                              │
                      │                              ├── o_assets2Storyboard ──N:N──> o_storyboard (分镜)
                      │                              │
                      │                              └── o_image (图片)
                      │
                      ├── o_storyboard ──N:1──> o_videoTrack (视频轨道)
                      │                            │
                      │                            └── o_videoTrack ──1:N──> o_video (视频)
                      │
                      └── o_agentWorkData (工作区缓存)
```

---

## 文件清单与职责

### 1. AI Agent 层

| 文件路径 | 职责 |
|----------|------|
| `src/agents/productionAgent/index.ts` | ProductionAgent 主入口，决策层 AI，管理子 Agent 调度 |
| `src/agents/productionAgent/tools.ts` | Agent 工具定义：获取工作区数据、增删衍生资产、生成资产图片、生成分镜图片 |
| `src/socket/routes/productionAgent.ts` | Socket.IO 命名空间处理器，管理连接认证、chat 消息、stop 停止 |
| `src/socket/resTool.ts` | Socket 消息构建工具，支持流式输出、thinking、搜索、图片等内容类型 |

### 2. HTTP 路由层 — 工作区数据

| 文件路径 | API 路径 | 职责 |
|----------|----------|------|
| `src/routes/production/getFlowData.ts` | POST `/api/production/getFlowData` | 获取完整画布数据（剧本、资产、分镜、工作台），组装所有关联数据 |
| `src/routes/production/saveFlowData.ts` | POST `/api/production/saveFlowData` | 保存画布数据到 `o_agentWorkData`，更新分镜排序 |
| `src/routes/production/getStoryboardData.ts` | POST `/api/production/getStoryboardData` | 获取分镜列表，附带关联资产信息（characters） |

### 3. HTTP 路由层 — 资产管理 (assets)

| 文件路径 | API 路径 | 职责 |
|----------|----------|------|
| `src/routes/production/assets/getAssetsData.ts` | POST `.../assets/getAssetsData` | 获取项目所有资产（父级+衍生），含图片签名 URL |
| `src/routes/production/assets/batchGenerateAssetsImage.ts` | POST `.../assets/batchGenerateAssetsImage` | 批量生成资产图片，支持并发控制。先由 AI 生成 prompt，再调用图像模型生成 |
| `src/routes/production/assets/pollingImage.ts` | POST `.../assets/pollingImage` | 轮询资产图片生成状态（排除"生成中"的记录） |
| `src/routes/production/assets/updateAssetsUrl.ts` | POST `.../assets/updateAssetsUrl` | 手动更新资产图片 URL（来自图片工作流编辑后保存） |
| `src/routes/production/assets/deleteAssetsDireve.ts` | POST `.../assets/deleteAssetsDireve` | 删除衍生资产及其分镜关联 |

### 4. HTTP 路由层 — 分镜管理 (storyboard)

| 文件路径 | API 路径 | 职责 |
|----------|----------|------|
| `src/routes/production/storyboard/addStoryboard.ts` | POST `.../storyboard/addStoryboard` | 新增单个分镜，同时创建对应 videoTrack |
| `src/routes/production/storyboard/batchAddStoryboardInfo.ts` | POST `.../storyboard/batchAddStoryboardInfo` | 批量添加分镜，按 track 分组创建/复用 videoTrack |
| `src/routes/production/storyboard/editStoryboardInfo.ts` | POST `.../storyboard/editStoryboardInfo` | 编辑分镜的 prompt 和 videoDesc |
| `src/routes/production/storyboard/batchGenerateImage.ts` | POST `.../storyboard/batchGenerateImage` | 批量生成分镜图片，支持并发控制，带资产参考图 |
| `src/routes/production/storyboard/pollingImage.ts` | POST `.../storyboard/pollingImage` | 轮询分镜图片生成状态 |
| `src/routes/production/storyboard/updateStoryboardUrl.ts` | POST `.../storyboard/updateStoryboardUrl` | 手动更新分镜图片 URL（来自图片工作流编辑后保存） |
| `src/routes/production/storyboard/removeFrame.ts` | POST `.../storyboard/removeFrame` | 删除分镜，若为该 track 最后一个则同时删除 videoTrack |
| `src/routes/production/storyboard/previewImage.ts` | POST `.../storyboard/previewImage` | 生成分镜预览图（网格拼图，sharp 合成带编号标签） |
| `src/routes/production/storyboard/downPreviewImage.ts` | POST `.../storyboard/downPreviewImage` | 下载分镜预览图（PNG 格式，类似 previewImage 但返回文件下载） |
| `src/routes/production/storyboard/getStoryboardData.ts` | POST `.../storyboard/getStoryboardData` | 获取分镜详情，含关联资产的 name/type/avatar |

### 5. HTTP 路由层 — 工作台 (workbench)

| 文件路径 | API 路径 | 职责 |
|----------|----------|------|
| `src/routes/production/workbench/getGenerateData.ts` | POST `.../workbench/getGenerateData` | 获取工作台生成数据：trackList（含分镜媒体 + 资产媒体 + 视频列表），检测模型是否支持多参 |
| `src/routes/production/workbench/addTrack.ts` | POST `.../workbench/addTrack` | 新增视频轨道 |
| `src/routes/production/workbench/deleteTrack.ts` | POST `.../workbench/deleteTrack` | 删除视频轨道 |
| `src/routes/production/workbench/generateVideo.ts` | POST `.../workbench/generateVideo` | 生成视频：收集参考图 → 转 base64 → 调 AI 视频模型 → 保存 |
| `src/routes/production/workbench/generateVideoPrompt.ts` | POST `.../workbench/generateVideoPrompt` | AI 生成视频 prompt：分析分镜和资产信息，调用 LLM 生成提示词并保存到 videoTrack |
| `src/routes/production/workbench/updateVideoPrompt.ts` | POST `.../workbench/updateVideoPrompt` | 手动更新视频轨道的 prompt 和 duration |
| `src/routes/production/workbench/getVideoList.ts` | POST `.../workbench/getVideoList` | 获取视频列表（按 scriptId 查询所有 storyboard 对应的视频） |
| `src/routes/production/workbench/selectVideo.ts` | POST `.../workbench/selectVideo` | 为 videoTrack 选择/切换已生成的视频 |
| `src/routes/production/workbench/delVideo.ts` | POST `.../workbench/delVideo` | 删除视频记录，并清除 videoTrack 上的 videoId 引用 |

### 6. HTTP 路由层 — 图片编辑工作流 (editImage)

| 文件路径 | API 路径 | 职责 |
|----------|----------|------|
| `src/routes/production/editImage/getImageFlow.ts` | POST `.../editImage/getImageFlow` | 获取图片编辑工作流数据（nodes + edges），含签名 URL |
| `src/routes/production/editImage/saveImageFlow.ts` | POST `.../editImage/saveImageFlow` | 新建图片工作流，保存到 `o_imageFlow` |
| `src/routes/production/editImage/updateImageFlow.ts` | POST `.../editImage/updateImageFlow` | 更新已有图片工作流 |
| `src/routes/production/editImage/generateFlowImage.ts` | POST `.../editImage/generateFlowImage` | 通过工作流生成图片（调用 AI 图像模型） |
| `src/routes/production/editImage/uploadImage.ts` | POST `.../editImage/uploadImage` | 上传图片（base64 → OSS 存储），仅支持图片格式 |
| `src/routes/production/editImage/getImageDefaultModle.ts` | POST `.../editImage/getImageDefaultModle` | 获取项目默认图像模型和质量配置 |

---

## 核心流程详解

### 流程 1: AI Agent 驱动的自动化创作

ProductionAgent 采用**三层 Agent 架构**：

```
决策层 (decisionAI)
  ├── 读取系统 Prompt: data/skills/production_agent_decision.md
  ├── 注入记忆 (Memory): 短期/长期/RAG
  ├── 注入项目模型信息
  │
  └── 可调度的子 Agent (tools):
        ├── run_sub_agent_derive_assets    → 衍生资产分析与写入
        │     └── prompt: production_execution_derive_assets.md
        ├── run_sub_agent_generate_assets  → 衍生资产图片生成
        │     └── prompt: production_execution_generate_assets.md
        ├── run_sub_agent_director_plan    → 导演规划（拍摄计划）
        │     └── prompt: production_execution_director_plan.md
        ├── run_sub_agent_storyboard_gen   → 分镜图生成
        │     └── prompt: production_execution_storyboard_gen.md
        ├── run_sub_agent_storyboard_panel → 分镜面板写入
        │     └── prompt: production_execution_storyboard_panel.md
        ├── run_sub_agent_storyboard_table → 分镜表构建
        │     └── prompt: production_execution_storyboard_table.md
        └── run_sub_agent_supervision      → 监督层（质量检查）
              └── prompt: production_agent_supervision.md
```

**Agent 工具列表**（定义在 `tools.ts`）：

| 工具名 | 功能 | 通过 Socket 事件 |
|--------|------|-------------------|
| `get_flowData` | 获取工作区指定 key 的数据 | `getFlowData` |
| `add_deriveAsset` | 新增/更新衍生资产 | `addDeriveAsset` |
| `del_deriveAsset` | 删除衍生资产 | `delDeriveAsset` |
| `generate_deriveAsset` | 触发衍生资产图片生成 | `generateDeriveAsset` |
| `generate_storyboard` | 触发分镜图片生成 | `generateStoryboard` |

**技能系统 (Skills)**：
- 美术风格技能：`data/skills/art_skills/{artStyle}/driector_skills/*.md`
- 故事风格技能：`data/skills/story_skills/{storyName}/driector_skills/*.md`
- 制作技能：`data/skills/production_skills/*.md`

### 流程 2: 画布数据加载 (`getFlowData`)

```
前端请求 getFlowData(projectId, episodesId)
  │
  ├── 查 o_agentWorkData → 有缓存数据？
  │     │
  │     ├── 无缓存：
  │     │     ├── 查 o_script → 获取剧本内容
  │     │     ├── 查 o_scriptAssets → 获取资产 ID 列表
  │     │     ├── 查 o_assets + o_image → 组装资产数据（含衍生资产）
  │     │     └── 返回初始 FlowData（script/scriptPlan/assets/storyboardTable/storyboard/workbench）
  │     │
  │     └── 有缓存：
  │           ├── 查 o_storyboard → 分镜列表（含 filePath 签名 URL）
  │           ├── 查 o_assets2Storyboard → 分镜-资产关联映射
  │           ├── 查 o_assets + o_image → 资产数据（含衍生资产）
  │           └── 合并缓存数据 + 数据库最新数据，返回完整 FlowData
```

**FlowData 结构**：

```typescript
{
  script: string;           // 剧本内容
  scriptPlan: string;       // 拍摄计划
  assets: AssetItem[];      // 资产列表（含衍生资产）
  storyboardTable: string;  // 分镜表（文本）
  storyboard: StoryboardItem[];  // 分镜面板
}
```

### 流程 3: 分镜批量添加 (`batchAddStoryboardInfo`)

```
前端提交分镜数据数组
  │
  ├── 逐条 insert o_storyboard
  ├── 逐条 insert o_assets2Storyboard（资产关联）
  │
  └── 按 track 字段分组:
        ├── 已存在同 track 名称的 trackId → 复用，更新 duration
        └── 不存在 → insert o_videoTrack，新建轨道
        └── update o_storyboard.trackId = 对应轨道 ID
```

### 流程 4: 分镜图片批量生成 (`batchGenerateImage`)

```
前端提交 storyboardIds + projectId + scriptId
  │
  ├── 更新状态: shouldGenerateImage=0 → "未生成", =1 → "生成中"
  ├── 先返回当前状态给前端（不等待生成完成）
  │
  └── 后台按 concurrentCount 分批并发:
        ├── 查询每个分镜关联的资产图片 → getAssetsImageBase64()
        ├── 调用 u.Ai.Image(model).run() → 生成图片
        ├── imageCls.save(savePath) → 保存到 OSS
        ├── 更新 o_storyboard: filePath + state="已完成"
        └── 失败: 更新 state="生成失败" + reason
```

### 流程 5: 资产图片批量生成 (`batchGenerateAssetsImage`)

```
前端提交 assetIds + projectId + scriptId
  │
  ├── 为每个资产创建 o_image 记录（state="生成中"）
  ├── 更新 o_assets.imageId
  ├── 先返回 "开始生成资产图片"
  │
  └── 后台按 concurrentCount 分批并发:
        ├── 查询父级资产图片 → 获取参考图 base64
        ├── 获取美术风格 Prompt (art_character_derivative/art_prop_derivative/art_scene_derivative)
        ├── 调用 AI Text → 生成图像 prompt
        ├── 调用 AI Image → 生成图片
        ├── imageCls.save() → OSS
        ├── 更新 o_assets.prompt + o_image.state="已完成"
        └── 失败: o_image.state="生成失败" + errorReason
```

### 流程 6: 视频生成 (`generateVideo`)

```
前端提交 prompt + uploadData + model + duration + resolution + trackId
  │
  ├── 查询参考图: storyboard → o_storyboard.filePath / assets → o_image.filePath
  ├── 图片转 base64
  ├── insert o_video（state="生成中"）
  ├── 先返回 videoId
  │
  └── 后台异步:
        ├── 调用 u.Ai.Video(model).run() → 生成视频
        │     参数: prompt, referenceList(base64图片), mode, duration, aspectRatio, resolution, audio
        ├── aiVideo.save(videoPath) → 保存到 OSS
        └── 更新 o_video.state = "生成成功" / "生成失败"
```

### 流程 7: 视频 Prompt 生成 (`generateVideoPrompt`)

```
前端提交 trackId + info(分镜/资产引用) + model
  │
  ├── 查询分镜信息 (videoDesc, prompt, track, duration) + 关联资产
  ├── 查询项目美术风格
  ├── 查询视频 Prompt 生成模板 (o_prompt where type="videoPromptGeneration")
  ├── 调用 AI Text → 生成视频 prompt
  ├── 更新 o_videoTrack.prompt = 生成的 prompt
  └── 返回生成的 prompt 文本
```

### 流程 8: 工作台数据加载 (`getGenerateData`)

```
前端请求 getGenerateData(projectId, scriptId)
  │
  ├── 查 o_project → 获取视频模型配置
  ├── 查 o_videoTrack → 获取所有轨道
  ├── 查 o_storyboard → 按 trackId 分组分镜媒体
  ├── 查 o_assets + o_image → 获取资产媒体（多参模型时）
  ├── 查 o_video → 获取所有已生成视频
  │
  └── 组装 trackList:
        每个 track 包含:
        ├── id, duration, prompt, state, reason, selectVideoId
        ├── medias: [...资产媒体, ...分镜媒体] (去重)
        └── videoList: [{id, src, state}]
```

### 流程 9: 图片编辑工作流

```
用户在图片编辑器中操作节点流（nodes + edges）
  │
  ├── 保存: saveImageFlow / updateImageFlow → o_imageFlow
  │     节点类型: "upload"(上传图) / "generated"(生成图)
  │     图片 URL 转为相对路径存储
  │
  ├── 加载: getImageFlow → 还原图片签名 URL
  │
  ├── 生成: generateFlowImage → 调 AI 图像模型生成
  │     输入: model, references(参考图URL), quality, ratio, prompt
  │     输出: 生成的图片 URL
  │
  └── 应用结果:
        ├── 资产: updateAssetsUrl → 更新 o_assets.imageId + o_image
        └── 分镜: updateStoryboardUrl → 更新 o_storyboard.filePath + state
```

---

## Socket.IO 通信

### 命名空间: `/api/socket/productionAgent`

**连接认证**：
- `token`: JWT 令牌验证
- `isolationKey`: 隔离键（区分不同会话）
- `projectId`: 项目 ID
- `scriptId`: 剧本 ID

**事件**：

| 事件名 | 方向 | 说明 |
|--------|------|------|
| `chat` | Client → Server | 发送消息给 Agent，触发 decisionAI |
| `stop` | Client → Server | 中止当前 AI 生成 |
| `updateContext` | Client → Server | 更新隔离键/项目/剧本上下文 |
| `message` | Server → Client | 新消息通知（含 id, role, name, status） |
| `message:update` | Server → Client | 消息状态更新（pending/complete/stop/error） |
| `content:add` | Server → Client | 添加消息内容块（text/markdown/thinking/image/toolcall 等） |
| `content:update` | Server → Client | 更新内容块数据（流式追加/合并） |

**Agent 内部 Socket 事件**（Agent 通过 Socket 与前端实时交互）：

| 事件名 | 说明 |
|--------|------|
| `getFlowData` | Agent 获取前端工作区数据 |
| `addDeriveAsset` | Agent 通知前端新增衍生资产 |
| `delDeriveAsset` | Agent 通知前端删除衍生资产 |
| `generateDeriveAsset` | Agent 触发前端衍生资产图片生成 |
| `generateStoryboard` | Agent 触发前端分镜图片生成 |

---

## 状态流转

### 分镜图片状态
```
未生成 → 生成中 → 已完成
                 → 生成失败
```

### 视频状态
```
未生成 → 生成中 → 已完成 (生成成功)
                 → 生成失败
```

### 资产图片状态（o_image.state）
```
生成中 → 已完成
       → 生成失败
```

---

## 关键设计特点

1. **缓存机制**: `o_agentWorkData` 缓存 Agent 工作区数据，避免每次重新计算。数据变更时通过 `saveFlowData` 更新缓存。

2. **Track 分组**: 分镜通过 `track` 字段分组，每组分镜对应一个 `o_videoTrack`。添加分镜时自动检测同 track 是否已有 videoTrack 并复用。

3. **并发控制**: 批量生成图片时通过 `concurrentCount` 参数控制并发数（默认 5），避免 API 过载。

4. **异步生成**: 图片和视频生成均采用"先返回、后异步"模式，前端通过轮询（polling）接口获取最新状态。

5. **工作流编辑**: 图片编辑工作流（`o_imageFlow`）支持基于节点流的图片编辑，编辑结果可回写到资产或分镜。

6. **多模型支持**: 通过 `o_project.imageModel` / `o_project.videoModel` 配置，格式为 `vendorId:modelName`。工作台会检测模型是否支持多参（`mode` 是否为数组的数组）来决定是否附加资产参考图。

7. **Agent 记忆**: ProductionAgent 使用 Memory 系统（短期/长期/RAG），跨会话保持创作上下文。

8. **技能外部化**: 所有 Agent Prompt 以 Markdown 文件形式存储在 `data/skills/` 目录下，支持运行时编辑而无需代码变更。
