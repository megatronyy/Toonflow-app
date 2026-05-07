# ProductionAgent 智能体分析

## 一、概述

ProductionAgent 是 Toonflow 项目中的**视频制作智能体**，负责将剧本转化为可执行的分镜和视频素材。它采用**三层智能体架构**（决策层/执行层/监督层），通过 Socket.IO 与前端实时通信，支持资产衍生管理、分镜表构建、分镜面板写入和图片生成等完整视频制作流程。

### 核心能力

- 基于剧本与资产生成导演创作规划（六维度）
- 分析并管理衍生资产（角色服装变体、场景角度变体等）
- 将剧本拆分为结构化分镜表
- 分镜面板写入（支持三种模式）
- 分镜图片生成
- 自动质量审核与红线校验

---

## 二、文件结构

```
src/agents/productionAgent/
├── index.ts              # 主入口：决策层 + 子Agent调度 + 技能系统 + 流式响应处理
└── tools.ts              # 工具定义：工作区数据读写 + 资产操作 + 图片生成

src/socket/routes/
└── productionAgent.ts    # Socket.IO 命名空间处理：认证、上下文更新、消息路由

src/routes/production/
├── getFlowData.ts                              # POST: 读取完整工作区状态
├── saveFlowData.ts                             # POST: 保存工作区数据
├── assets/
│   ├── batchGenerateAssetsImage.ts             # 批量生成资产图片
│   ├── deleteAssetsDireve.ts                   # 删除衍生资产
│   ├── pollingImage.ts                         # 轮询图片生成状态
│   └── updateAssetsUrl.ts                      # 更新资产URL
├── storyboard/
│   ├── addStoryboard.ts                        # 新增分镜
│   ├── batchAddStoryboardInfo.ts               # 批量新增分镜
│   ├── batchDelete.ts                          # 批量删除分镜
│   ├── batchGenerateImage.ts                   # 批量生成分镜图片
│   ├── editStoryboardInfo.ts                   # 编辑分镜信息
│   ├── getStoryboardData.ts                    # 获取分镜数据
│   ├── pollingImage.ts                         # 轮询分镜图片状态
│   ├── previewImage.ts / downPreviewImage.ts   # 预览/下载分镜图
│   └── updateStoryboardUrl.ts                  # 更新分镜图片URL
├── editImage/                                  # 图片编辑相关（流程图、上传、生成）
└── workbench/                                  # 视频工作台（生成、轨道、音频绑定等）

src/utils/agent/
├── memory.ts              # 记忆系统（与ScriptAgent共享）
├── embedding.ts           # 文本向量化
└── skillsTools.ts         # 技能系统：技能扫描、frontmatter解析、激活/读取工具

data/skills/
├── production_agent_decision.md           # 决策层提示词（6阶段流水线）
├── production_execution_director_plan.md  # 执行层-导演规划
├── production_execution_derive_assets.md  # 执行层-衍生资产分析
├── production_execution_generate_assets.md# 执行层-衍生资产生成
├── production_execution_storyboard_table.md # 执行层-分镜表构建
├── production_execution_storyboard_panel.md # 执行层-分镜面板写入
├── production_execution_storyboard_gen.md   # 执行层-分镜图生成
└── production_agent_supervision.md          # 监督层提示词
```

---

## 三、与 ScriptAgent 的关键差异

ProductionAgent 在架构上与 ScriptAgent 类似（三层 + Socket.IO + Memory），但有以下显著差异：

| 维度 | ScriptAgent | ProductionAgent |
|------|-------------|-----------------|
| **Socket 上下文** | `projectId` | `projectId` + `scriptId`（按集操作） |
| **上下文动态切换** | 不支持 | 支持 `updateContext` 事件动态切换集数 |
| **技能系统** | 静态加载技能文件 | 动态技能系统（`activate_skill` + `read_skill_file`） |
| **风格技法** | 无 | 根据项目的 `artStyle` 和 `directorManual` 加载风格专属技法 |
| **子Agent数量** | 3个执行 + 1个监督 | 6个执行 + 1个监督 |
| **工作区数据** | 通过 Socket 获取 | 通过 Socket 获取，但数据结构更复杂（含资产层级） |
| **模型信息注入** | 无 | 注入图像模型/视频模型/多参信息 |
| **审核范围** | 骨架 + 策略 | 导演规划 + 分镜表（含6条绝对红线） |

---

## 四、三层架构详解

### 4.1 决策层（Decision Layer）

**文件**: `src/agents/productionAgent/index.ts` → `runDecisionAI()`
**提示词**: `data/skills/production_agent_decision.md`

决策层是唯一与用户直接交互的层，核心原则是**只决策不执行**。

#### 核心职责

1. **需求分析** — 解析用户请求，判断属于流水线哪个阶段
2. **任务拆解** — 将复杂请求分解为可执行的子任务
3. **调度执行** — 通过阶段专用调度工具派发到执行层
4. **质量管控** — 调用监督层审核关键阶段产出
5. **记忆检索** — 通过 `deepRetrieve` 获取历史进度

#### 6阶段流水线

```
阶段1: 导演规划(含衍生预划) → 阶段2: 衍生资产分析 → 阶段3: 衍生资产生成(可选)
    → 阶段4: 构建分镜表 → 阶段5: 分镜面板写入 → 阶段6: 分镜图生成
```

| 阶段 | 子Agent工具 | 输出 | 需审核 |
|------|------------|------|--------|
| 1 导演规划 | `run_sub_agent_director_plan` | 导演拍摄计划 + 衍生预划清单 | 是 |
| 2 衍生资产分析 | `run_sub_agent_derive_assets` | 衍生资产信息写入 | 否 |
| 3 衍生资产生成 | `run_sub_agent_generate_assets` | 图片生成启动（异步） | 否 |
| 4 构建分镜表 | `run_sub_agent_storyboard_table` | 结构化分镜表 | 是 |
| 5 分镜面板写入 | `run_sub_agent_storyboard_panel` | 分镜面板 XML | 否 |
| 6 分镜图生成 | `run_sub_agent_storyboard_gen` | 分镜图片生成启动（异步） | 否 |

#### 阶段间决策逻辑

**阶段1 → 阶段2**：执行层返回导演规划后，自动触发监督层审核，等待用户确认。

**阶段2 → 阶段3**：
- 执行层返回"无需衍生资产" → 直接进入阶段4
- 执行层返回衍生清单 → 展示给用户确认是否生成图片
  - 确认全部 → 阶段3
  - 部分生成 → 传子集到阶段3
  - 跳过 → 直接阶段4
  - 调整 → 重新派发分析

**阶段3 → 阶段4**：图片生成是异步的，返回后询问用户是否同时进入阶段4。

**阶段4 → 阶段5**：审核通过后，根据模型参数决定写入模式：
- `多参=是` → 询问用户选择"纯文本多参模式"或"分镜图辅助多参模式"
- `多参=否` → 直接以"首位帧模式"派发

**阶段5 → 阶段6**：
- 纯文本多参模式 → 提醒用户进入视频工作台
- 其他模式 → 询问是否生成分镜图

### 4.2 执行层（Execution Layer）

执行层由6个专职子Agent组成，每个子Agent加载独立的技能文件，并可动态激活风格技法技能。

#### 导演规划 Agent

**提示词**: `data/skills/production_execution_director_plan.md`
**AI Key**: `productionAgent:directorPlanAgent`
**输出格式**: `<scriptPlan>...</scriptPlan>`

制定六维度导演创作规划：

| 维度 | 内容 |
|------|------|
| ① 主题立意与叙事核心 | 核心主题、情感主线、离场感受、表达策略 |
| ② 视觉风格与画面基调 | 构图风格（对称/三分/对角线/框中框映射情绪）、镜头运动偏好 |
| ③ 叙事结构与节奏规划 | 叙事模式选型、段落划分表、情绪曲线、转折点、过渡方式 |
| ④ 分场景情绪与画面意图 | 逐场的情绪目标、氛围、镜头意图、空间叙事、距离感设计 |
| ⑤ 声音方向 | 环境音设计（具体声源）、沉默运用 |
| ⑥ 转场与视觉连续性 | 转场策略、空镜过渡、视觉连续性锚点 |
| ⑦ 衍生资产预划清单 | 资产名·衍生状态·原因/出现段落 |

核心创作原则：
- **导演具象化原则**：所有描述以"摄像机能拍到什么"为标准，禁止抽象情绪词
- **动作具体化**：连续物理动作链，禁止"感到疲惫"
- **情绪靠身体**：肢体微表情传达，如"指尖发颤、瞳孔收缩"
- **Agent层不规划光影/色调**：由视频模型从场景图自动推导

衍生资产预划规则：
- 角色：仅①服装变体 ②结构性特征变体（变身/异化）
- 场景：四类并列——①角度变体 ②时段变体 ③天候变体 ④破坏/状态变体
- 判定门槛：稳定、可复用、资产级，瞬时表情/单镜头不入清单
- 每个父资产 0~5 条，宁缺勿滥

#### 衍生资产分析 Agent

**提示词**: `data/skills/production_execution_derive_assets.md`
**AI Key**: `productionAgent:deriveAssetsAgent`

严格以导演规划⑦预划清单为硬约束：
1. 读取剧本、资产、导演规划
2. 逐条按预划清单生成完整 `name`/`desc`/`type` 字段
3. 逐条调用 `add_deriveAsset` 写入数据库

**不得超出预划，不得缺漏预划**——这是阶段间的硬约束传递。

#### 衍生资产生成 Agent

**提示词**: `data/skills/production_execution_generate_assets.md`
**AI Key**: `productionAgent:generateAssetsAgent`

简单直接的图片生成任务：
1. 获取资产列表
2. 收集需要生成图片的衍生资产ID
3. 调用 `generate_deriveAsset` 异步发起图片生成

#### 分镜表构建 Agent

**提示词**: `data/skills/production_execution_storyboard_table.md`
**AI Key**: `productionAgent:storyboardTableAgent`
**输出格式**: `<storyboardTable>...</storyboardTable>`

将剧本拆分为结构化分镜表，包含13个字段：

| 字段 | 说明 |
|------|------|
| 序号 | 分镜编号 |
| 画面描述 | 15~50字具体视觉描述 |
| 场景 | 当前场景名称 |
| 关联资产名称 | 出现的角色/道具/场景名称列表 |
| 时长 | 秒数（含台词时长计算公式） |
| 景别 | 大特写/特写/近景/中景/中远景/远景/大远景 |
| 运镜 | 静止/缓推/缓拉/环绕等 |
| 角色动作 | 连续物理动作链（含衔接说明） |
| 朝向 | 角色面朝方向（独立列） |
| 空间关系 | 9站位（左前/中前/右前/左中/中中/右中/左后/中后/右后） |
| 情绪 | 具象情绪描述 |
| 台词 | 剧本原文照搬 |
| 音效 | 仅环境音+动作音，禁配乐 |
| 关联资产ID | 实际资产ID列表（衍生优先） |

关键约束：
- **导演规划两层对齐**：段落锚点 + 场次镜头意图
- **台词原文锁定**：一字不差，禁改写/省略/意译
- **视觉连续性铁律**：动作连续/景别递进/视轴守恒/朝向逻辑/信息控制/节拍密度/头尾安全区
- **任意字段禁光影/色调**：由场景图自动承担
- **音效列禁配乐**：仅允许具体物理声源

#### 分镜面板写入 Agent

**提示词**: `data/skills/production_execution_storyboard_panel.md`
**AI Key**: `productionAgent:storyboardPanelAgent`
**输出格式**: `<storyboardItem ...></storyboardItem>`

支持三种写入模式：

| 模式 | 适用场景 | prompt | shouldGenerateImage | track规则 |
|------|---------|--------|---------------------|-----------|
| 纯文本多参模式 | 多参=是，用户选择 | 空字符串 | false | 累计≤15s |
| 分镜图辅助多参模式 | 多参=是，用户选择 | 正常生成 | true | 累计≤15s |
| 首位帧模式 | 多参=否 | 正常生成 | true | 每行独立递增 |

执行步骤：
1. 获取分镜表、识别写入模式、加载对应技法
2. 确定分组与时长规则
3. **人物空间位置与朝向预分析**（建立全局基准表）
4. **图像资产标注与正文绑定**（`@图N` 格式）
5. 生成 `videoDesc`（所有模式）
6. 生成 `prompt` 并忠实性校验（非纯文本模式）
7. 逐行写入分镜面板 XML

#### 分镜图生成 Agent

**提示词**: `data/skills/production_execution_storyboard_gen.md`
**AI Key**: `productionAgent:storyboardGenAgent`

简单直接的图片生成：
1. 获取分镜面板数据
2. 提取真实分镜ID列表
3. 调用 `generate_storyboard` 异步发起图片生成

### 4.3 监督层（Supervision Layer）

**文件**: `src/agents/productionAgent/index.ts` → `run_sub_agent_supervision` 工具
**提示词**: `data/skills/production_agent_supervision.md`
**AI Key**: `productionAgent:supervisionAgent`

监督层维护 **6条绝对红线（R1~R6）**，违反任一直接判严重：

| 红线 | 内容 |
|------|------|
| R1 资产引用合法 | ID在assets中存在、可见角色必须关联、场景资产必须引用、禁止父子同镜 |
| R2 剧本忠实 | 台词一字不差、不遗漏场次/事件、不新增情节 |
| R3 风格不冲突 | 构图/节奏/声音方向与风格技法一致（光影/色调不参与比对） |
| R4 必填字段不缺失 | 导演规划七维度完整、分镜表必填字段齐全 |
| R5 具象可感 | 禁止抽象笼统词、禁光影/色温描述、声音具体到声源 |
| R6 父子资产选择正确 | 衍生状态匹配时必须用衍生ID |

审核对象：

**导演规划审核**：
- 内容质量维度（14项）：资产匹配、风格一致性、七维度完整、具象化表达、衍生预划合法/完整/判定门槛、剧情覆盖度、叙事模式、节奏、转折点、构图、声音、场景角度覆盖
- 工程检查：依赖关系正确

**分镜表审核**：
- 20个审核维度，涵盖资产关联、台词完整性、剧本覆盖度、视觉连续性七律、朝向/空间关系稳定、拆分粒度、时长合理、音效禁配乐、字段禁光影等

---

## 五、动态技能系统

ProductionAgent 引入了 ScriptAgent 没有的**动态技能加载机制**。

### 技能层级

```
art_skills/{artStyle}/driector_skills/*.md    # 画风专属技法
story_skills/{storyName}/driector_skills/*.md  # 风格专属技法
production_skills/*.md                         # 制作通用技法
```

### 技能工具

| 工具 | 功能 |
|------|------|
| `activate_skill` | 按名称激活技能，加载完整指令到上下文（防重复激活） |
| `read_skill_file` | 读取已激活技能的资源文件（含路径穿越防护） |

### 技能加载流程

1. `createArtSkills()` — 加载画风+风格专属技法，用于衍生资产和资产生成
2. `useProductionSkills()` — 加载画风+风格+制作通用技法，用于分镜表和分镜面板

技能文件使用 frontmatter 格式，必须包含 `name` 和 `description` 字段。

### 子Agent与技能的绑定关系

| 子Agent | 技能来源 |
|---------|---------|
| 导演规划 | `createArtSkills`（画风+风格） |
| 衍生资产分析 | `createArtSkills`（画风+风格） |
| 衍生资产生成 | `createArtSkills`（画风+风格） |
| 分镜表构建 | `useProductionSkills`（画风+风格+制作） |
| 分镜面板写入 | `useProductionSkills`（画风+风格+制作） |
| 分镜图生成 | `useProductionSkills`（画风+风格+制作） |

---

## 六、通信机制

### Socket.IO 通道

**命名空间**: `/api/socket/productionAgent`
**文件**: `src/socket/routes/productionAgent.ts`

与 ScriptAgent 的关键差异：

| 特性 | ScriptAgent | ProductionAgent |
|------|-------------|-----------------|
| 认证数据 | `projectId` | `projectId` + `scriptId` |
| 动态上下文 | 不支持 | `updateContext` 事件切换集数 |
| ResTool角色名 | `统筹` | `视频策划` |
| 子Agent角色名 | `编剧` / `编辑` | `执行导演` / `监制` |

事件列表：

| 事件 | 方向 | 用途 |
|------|------|------|
| `chat` | Client→Server | 用户消息，触发决策层 |
| `updateContext` | Client→Server | 动态切换 projectId/scriptId/isolationKey |
| `updateThinkConfig` | Client→Server | 调整思考模式 |
| `stop` | Client→Server | 中止当前任务 |
| `getFlowData` | Server→Client→Server | Agent请求前端工作区数据 |
| `addDeriveAsset` | Server→Client→Server | 新增/更新衍生资产通知 |
| `delDeriveAsset` | Server→Client→Server | 删除衍生资产通知 |
| `generateDeriveAsset` | Server→Client→Server | 发起衍生资产生成 |
| `generateStoryboard` | Server→Client→Server | 发起分镜图生成 |

### REST API

| 路由 | 方法 | 用途 |
|------|------|------|
| `/api/production/getFlowData` | POST | 读取完整工作区状态（含资产层级、分镜、剧本） |
| `/api/production/saveFlowData` | POST | 保存工作区数据（含分镜排序更新） |
| `/api/production/storyboard/batchGenerateImage` | POST | 批量生成分镜图片 |
| `/api/production/assets/batchGenerateAssetsImage` | POST | 批量生成资产图片 |
| `/api/production/workbench/generateVideo` | POST | 生成视频 |
| `/api/production/workbench/batchGenerateVideo` | POST | 批量生成视频 |

---

## 七、工具系统

**文件**: `src/agents/productionAgent/tools.ts`

| 工具 | 功能 | 数据源 |
|------|------|--------|
| `get_flowData` | 获取工作区指定数据 | Socket 事件 → 前端工作区 |
| `add_deriveAsset` | 新增/更新衍生资产 | SQLite `o_assets` + `o_scriptAssets` |
| `del_deriveAsset` | 删除衍生资产 | SQLite `o_assets` + `o_scriptAssets` |
| `generate_deriveAsset` | 生成衍生资产图片 | Socket 事件 → 异步图片生成 |
| `generate_storyboard` | 生成分镜图片 | Socket 事件 → 异步图片生成 |
| `deepRetrieve` | 深度检索记忆 | SQLite `memories` + 向量搜索 |
| `activate_skill` | 激活技能文件 | 文件系统 `data/skills/` |
| `read_skill_file` | 读取技能资源文件 | 文件系统 `data/skills/` |

### 工作区数据结构

```typescript
interface FlowData {
  script: string;           // 剧本内容
  scriptPlan: string;       // 导演拍摄计划
  assets: AssetItem[];      // 资产列表（含衍生）
  storyboardTable: string;  // 分镜表（Markdown）
  storyboard: Storyboard[]; // 分镜面板数据
}

interface AssetItem {
  id: number;
  name: string;
  type: "role" | "tool" | "scene" | "clip";
  prompt: string;
  desc: string;
  derive: DeriveAsset[];    // 衍生资产列表
}

interface Storyboard {
  id: number;
  duration: number;          // 秒
  prompt: string;            // 图片生成提示词
  associateAssetsIds: number[]; // 关联资产ID
  src: string | null;        // 图片路径
  videoDesc: string;         // 视频描述
  shouldGenerateImage: boolean;
}
```

---

## 八、数据模型

### 涉及的数据库表

| 表名 | 用途 |
|------|------|
| `o_project` | 项目信息（含 imageModel、videoModel、artStyle、directorManual、mode） |
| `o_script` | 剧本记录 |
| `o_assets` | 资产定义（父资产 `assetsId=null`，衍生资产 `assetsId` 指向父） |
| `o_scriptAssets` | 剧本-资产关联 |
| `o_storyboard` | 分镜数据（含 prompt、duration、videoDesc、shouldGenerateImage） |
| `o_assets2Storyboard` | 资产-分镜关联 |
| `o_image` | 图片生成记录（含 filePath、state、errorReason） |
| `o_agentWorkData` | Agent工作区数据（按 projectId + episodesId 存储 JSON） |
| `o_setting` | 系统配置 |
| `memories` | 记忆数据 |

### 资产层级关系

```
父资产 (o_assets, assetsId=null)
├── 衍生资产1 (o_assets, assetsId=父ID)
├── 衍生资产2 (o_assets, assetsId=父ID)
└── 衍生资产3 (o_assets, assetsId=父ID)
```

---

## 九、AI调用机制

### 模型配置

与 ScriptAgent 相同的 key 格式，但使用 `productionAgent` 前缀：

```
productionAgent:decisionAgent         → 决策层
productionAgent:directorPlanAgent     → 导演规划执行层
productionAgent:deriveAssetsAgent     → 衍生资产分析执行层
productionAgent:generateAssetsAgent   → 衍生资产生成执行层
productionAgent:storyboardTableAgent  → 分镜表构建执行层
productionAgent:storyboardPanelAgent  → 分镜面板写入执行层
productionAgent:storyboardGenAgent    → 分镜图生成执行层
productionAgent:supervisionAgent      → 监督层
```

### 模型信息注入

决策层在 `runDecisionAI()` 中解析项目配置，提取：
- 图像模型名称
- 视频模型名称
- 多参模式（`isRef`，根据 `project.mode` 判断是否为数组）

这些信息注入到 `assistant` 角色消息中，供决策层判断阶段5的写入模式。

---

## 十、执行流程总览

### 完整制作流程

```
1. 用户连接 Socket.IO → JWT认证 → 建立 ResTool（含 projectId + scriptId）
2. 用户发送消息 → runDecisionAI()
3. 加载决策层提示词 + 记忆 + 项目信息（含模型配置）
4. 阶段1: 派发导演规划任务 → 加载画风/风格技法 → 六维度规划 + 衍生预划
5. 自动监督审核 → 等待用户确认
6. 阶段2: 按预划清单逐条写入衍生资产 → 展示给用户确认
7. 阶段3(可选): 异步生成衍生资产图片 → 询问是否并行进入阶段4
8. 阶段4: 构建分镜表（13字段 + 视觉连续性校验）→ 自动监督审核 → 用户确认
9. 阶段5: 根据多参模式选择写入方式 → 逐行写入分镜面板 XML
10. 阶段6(可选): 异步生成分镜图片
11. 全程记忆存储 → 支持跨会话恢复进度
```

### 子Agent执行流程

与 ScriptAgent 相同的消息切换机制：
```
决策层消息 → 完成 → 创建子Agent消息(subMsg, "执行导演"/"监制")
  → 加载技能文件 + 风格技法 → AI流式调用 → consumeFullStream
  → 响应存入记忆 → 子消息完成 → 创建新决策层消息("视频策划")
```

---

## 十一、设计特点与约束

### 设计特点

1. **动态技能系统**: `activate_skill`/`read_skill_file` 实现运行时按需加载技能，支持画风/风格/制作三类技法
2. **硬约束传递**: 阶段1的衍生预划清单是阶段2的硬约束，不可超出不可缺漏
3. **多模式分镜写入**: 根据视频模型参数（多参/非多参）和用户选择，支持三种分镜面板写入模式
4. **6条绝对红线**: R1~R6 贯穿所有审核，违反即判严重，确保产出可执行性
5. **导演具象化原则**: 所有描述以"摄像机能拍到什么"为标准，彻底消除抽象表达
6. **光影分离设计**: Agent层不规划光影/色调，由视频模型从场景图自动推导，避免冲突
7. **动态上下文切换**: `updateContext` 支持在不重新连接的情况下切换集数
8. **异步图片生成**: 衍生资产和分镜图的生成均为异步，不阻塞流水线

### 关键约束

- 派发指令正文不超过100字
- 6阶段必须串行执行（阶段3可选跳过）
- 仅阶段1和阶段4需要审核
- 阶段2必须严格按阶段1预划执行
- 阶段4/5/6只能使用资产库中已存在的资产
- 分镜面板行数和时长必须与分镜表一致
- 视频生成请求由决策层拒绝，引导用户前往视频生成面板
