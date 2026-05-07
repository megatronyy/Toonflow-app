# ScriptAgent 智能体分析

## 一、概述

ScriptAgent 是 Toonflow 项目中的**剧本改编智能体**，负责将小说转换为短剧剧本。它采用**三层智能体架构**（决策层/执行层/监督层），通过 Socket.IO 与前端实时通信，具备持久化的跨会话记忆系统。

### 核心能力

- 将小说内容分析、提炼为短剧故事骨架
- 制定改编策略（删减、节奏、情绪规划）
- 逐集生成符合短剧规范的完整剧本
- 自动质量审核与评分

---

## 二、文件结构

```
src/agents/scriptAgent/
├── index.ts              # 主入口：决策层 + 子Agent调度 + 流式响应处理
└── tools.ts              # 工具定义：数据库查询 + 工作区数据读写

src/socket/routes/
└── scriptAgent.ts        # Socket.IO 命名空间处理：认证、消息路由、生命周期

src/routes/scriptAgent/
├── setPlanData.ts        # POST API: 保存工作区数据（骨架+策略+剧本）
├── updateData.ts         # POST API: 更新已有工作区数据
└── getPlanData.ts        # POST API: 读取工作区数据

src/utils/agent/
├── memory.ts             # 记忆系统：短期/长期/RAG/深度检索
├── embedding.ts          # 文本向量化（ONNX + @huggingface/transformers）
└── skillsTools.ts        # 技能文件加载工具

data/skills/
├── script_agent_decision.md       # 决策层提示词
├── script_execution_skeleton.md   # 执行层-故事骨架提示词
├── script_execution_adaptation.md # 执行层-改编策略提示词
├── script_execution_script.md     # 执行层-剧本编写提示词
└── script_agent_supervision.md    # 监督层提示词
```

---

## 三、三层架构详解

### 3.1 决策层（Decision Layer）

**文件**: `src/agents/scriptAgent/index.ts` → `runDecisionAI()`
**提示词**: `data/skills/script_agent_decision.md`

决策层是唯一与用户直接交互的层，承担四个核心职责：

#### (1) 需求分析与项目初始化

用户发起改编请求时，决策层必须先确认项目参数（集数、单集时长、原著范围、平台规格、风格定位、付费策略），校验章节范围有效性后才能进入流水线。

```
用户请求 → 询问项目参数 → 校验章节范围(get_novel_events) → 确认并保存配置
```

若用户需要推荐，决策层会先分析小说事件，给出类型推荐和配置建议。

#### (2) 流水线调度

改编流水线包含三个严格串行的阶段：

```
项目初始化 → 阶段1: 故事骨架 → 阶段2: 改编策略 → 阶段3: 剧本编写
```

- 阶段1-2 必须串行执行，后一阶段依赖前一阶段的输出
- 每个阶段完成后自动触发监督层审核
- 阶段3（剧本编写）不需要监督层审核，由决策层循环调度

#### (3) 子Agent调度

决策层通过 4 个工具函数调度子Agent：

| 工具名 | 目标层 | 用途 |
|--------|--------|------|
| `run_sub_agent_storySkeleton` | 执行层 | 构建故事骨架 |
| `run_sub_agent_adaptationStrategy` | 执行层 | 制定改编策略 |
| `run_sub_agent_script` | 执行层 | 编写单集剧本 |
| `run_supervision_agent` | 监督层 | 审核产出物 |

调度时的关键约束：
- 派发指令正文不超过100字
- 项目配置必须附带在每条指令头部
- 决策层**禁止**自行接管执行——子Agent失败时必须汇报用户并终止

#### (4) 审核结果处理

监督层返回 A/B/C/D 评分后，决策层根据评分引导用户：
- A → 建议进入下一阶段
- B → 询问修复还是继续
- C → 建议修复
- D → 建议重做

展示审核报告后**必须等待用户回复**，收到明确指示前不得派发新任务。

### 3.2 执行层（Execution Layer）

执行层由三个专职子Agent组成，各自加载独立的技能提示词：

#### 故事骨架 Agent

**提示词**: `data/skills/script_execution_skeleton.md`
**AI Key**: `scriptAgent:storySkeletonAgent`
**输出格式**: `<storySkeleton>...</storySkeleton>`

核心任务：
1. 读取事件表 (`get_novel_events`)
2. 读取已有工作区数据 (`get_planData`)
3. 构建包含以下要素的骨架：
   - 故事核（一句话核心吸引力）
   - 隐线（主角人物弧光）
   - 三幕结构（含幕末转折）
   - 分集决策（≤20集逐集展开，>20集总览表+关键集展开）
   - 全局删减决策表
   - 付费卡点设计（按 10%/30%/50%/70%/90% 比例分布）

关键技能：
- 大三角嵌套小三角结构（3核心角色 + 次要矛盾逐个解决）
- 前10集黄金结构
- 5大付费点标准（关键瞬间/根本性改变/好奇心/高燃场景/爱情拉扯）
- 信息差设计（先知型/焦急型/上帝型）
- 集末钩子多样化

#### 改编策略 Agent

**提示词**: `data/skills/script_execution_adaptation.md`
**AI Key**: `scriptAgent:adaptationStrategyAgent`
**输出格式**: `<adaptationStrategy>...</adaptationStrategy>`

核心任务：
1. 读取事件表和故事骨架
2. 输出改编策略，包含：
   - 3-5条核心改编原则（含正面指导和负面边界）
   - 主要删除决策表（被删内容/原因/替代方案）
   - 世界观呈现策略（出场节奏/解释度/锚点角色）

7大核心要点：强画面感、台词精简、节奏极致快、只沿主线、降低理解成本、情绪大于一切、开篇给足期待感。

#### 剧本编写 Agent

**提示词**: `data/skills/script_execution_script.md`
**AI Key**: `scriptAgent:scriptAgent`
**输出格式**: `<scriptItem name="剧本名称">...</scriptItem>`

核心任务：
1. 读取骨架、策略、前集剧本、原文、事件表
2. 仅提取当前任务集信息编写单集剧本
3. 输出标准剧本格式：场号 + △场景描述 + 人物台词 + OS/VO

关键规范：
- 单句台词≤20字（竖屏适配）
- 每集必含至少1个情绪要点（爆点/虐点/爽点）
- 场景描述需具体到可直接用于AI视频生成
- 5种转场标注（硬切/淡入/闪白/闪黑/叠化）
- 单集时长控制在目标值±10秒

### 3.3 监督层（Supervision Layer）

**文件**: `src/agents/scriptAgent/index.ts` → `run_supervision_agent` 工具
**提示词**: `data/skills/script_agent_supervision.md`
**AI Key**: `scriptAgent:supervisionAgent`

监督层是纯审核角色，**只提出问题和建议，不做任何修改决策**。

审核流程：
1. 识别审核对象（骨架/策略）
2. 通过工具读取实际数据（不凭记忆审核）
3. 对照红线清单逐项检查
4. 生成结构化审核报告

审核维度：

| 审核对象 | 维度 |
|----------|------|
| 故事骨架 | 结构完整性、分集与时长、章节全覆盖、付费点分布、前10%黄金结构、情绪布局、信息差标注、集末钩子、节奏框架 |
| 改编策略 | 用户意图一致、与骨架一致、7大要点覆盖、原则质量、情绪基调一致、人物弧光保留、删减合理性、世界观呈现、语言适配 |

通用红线（违反即标记严重）：
- 连续3集以上无情绪爆点
- 多线并行叙事
- 第1集无强冲突
- 使用现实官职称谓
- 大段旁白解说世界观

评分标准：A（可直接使用）、B（小修后可用）、C（需较大修改）、D（建议重做）。

---

## 四、通信机制

### 4.1 Socket.IO 通道

**命名空间**: `/api/socket/scriptAgent`
**文件**: `src/socket/routes/scriptAgent.ts`

连接流程：
```
前端连接 → JWT token验证 → isolationKey校验 → 创建ResTool实例 → 监听事件
```

事件列表：

| 事件 | 方向 | 用途 |
|------|------|------|
| `chat` | Client→Server | 用户发送消息，触发决策层 |
| `updateThinkConfig` | Client→Server | 动态调整AI思考模式（think开关+级别0-3） |
| `stop` | Client→Server | 中止当前任务（AbortController） |
| `getPlanData` | Server→Client→Server | Agent请求前端工作区数据 |
| 消息创建/更新/完成 | Server→Client | 实时消息流推送 |

### 4.2 REST API

| 路由 | 方法 | 用途 |
|------|------|------|
| `/api/scriptAgent/setPlanData` | POST | 保存完整工作区数据（骨架+策略+剧本入库） |
| `/api/scriptAgent/updateData` | POST | 按 ID 更新已有工作区数据 |
| `/api/scriptAgent/getPlanData` | POST | 按 projectId 读取工作区数据 |

### 4.3 消息流（ResTool）

`src/socket/resTool.ts` 实现实时消息流，支持多种内容类型：

- **Text**: 文本流式输出（逐字追加）
- **Thinking**: 思考过程展示（含计时）
- **Markdown**: Markdown 渲染内容
- **Search**: 搜索结果
- **ToolCall**: 工具调用展示
- **Image**: 图片展示

每条消息有独立生命周期：`创建 → 流式输出 → 完成/错误`。

子Agent运行时的消息切换逻辑：
```
决策层消息(msg) → 完成当前消息 → 创建子Agent消息(subMsg) → 子Agent流式输出 → 完成 → 创建新的决策层消息继续交互
```

---

## 五、记忆系统

**文件**: `src/utils/agent/memory.ts`

### 三层记忆结构

| 层次 | 存储 | 用途 | 默认限制 |
|------|------|------|----------|
| 短期记忆 (shortTerm) | `memories` 表, `type=message, summarized=0` | 最近的未总结消息 | 5条 |
| 长期摘要 (summaries) | `memories` 表, `type=summary` | AI 生成的消息摘要 | 10条 |
| RAG 检索 (rag) | 向量相似度搜索 | 按相关性检索历史消息 | 3条 |

### 记忆生命周期

```
消息写入 → 生成向量嵌入 → 存入 memories 表
    ↓
累积到阈值(默认3条) → AI生成摘要 → 存为 summary 类型 → 原消息标记 summarized=1
```

### 深度检索 (deepRetrieve)

当用户明确要求回顾历史时触发：
```
关键词 → 向量搜索 summaries → AI判断相关性 → 展开查询原始messages
```

### 记忆注入方式

`buildMemPrompt()` 将三层记忆格式化为：
```
## Memory
以下是你对用户的记忆，可作为参考但不要主动提及：
[相关记忆] ...      ← RAG结果
[历史摘要] ...      ← summaries
[近期对话] ...      ← shortTerm
```

注入到 `messages` 的 `assistant` 角色消息中（与项目信息一起）。

---

## 六、工具系统

**文件**: `src/agents/scriptAgent/tools.ts`

决策层可用的工具集合：

| 工具 | 功能 | 数据源 |
|------|------|--------|
| `get_novel_events` | 获取章节事件 | SQLite `o_novel` 表 |
| `get_planData` | 获取工作区数据 | Socket 事件 → 前端工作区 |
| `get_novel_text` | 获取章节原文 | SQLite `o_novel` 表 |
| `get_script_content` | 获取已有剧本 | SQLite `o_script` 表 |
| `deepRetrieve` | 深度检索记忆 | SQLite `memories` 表 + 向量搜索 |
| `run_sub_agent_storySkeleton` | 调度骨架Agent | 加载 skeleton 技能文件 |
| `run_sub_agent_adaptationStrategy` | 调度策略Agent | 加载 adaptation 技能文件 |
| `run_sub_agent_script` | 调度剧本Agent | 加载 script 技能文件 |
| `run_supervision_agent` | 调度审核Agent | 加载 supervision 技能文件 |

每个工具执行时会通过 ResTool 的 `thinking` 组件向前端展示实时进度。

---

## 七、数据模型

### 涉及的数据库表

| 表名 | 用途 |
|------|------|
| `o_project` | 项目信息（名称、类型、简介、画风、画幅） |
| `o_novel` | 小说章节（章节号、标题、事件、原文） |
| `o_script` | 剧本记录（按 projectId 和 name 存储） |
| `o_agentWorkData` | Agent工作区数据（JSON格式存储骨架和策略） |
| `o_setting` | 系统配置（含 tokenKey、记忆参数） |
| `memories` | 记忆数据（消息+摘要+向量嵌入） |

### 工作区数据结构

```typescript
{
  storySkeleton: string,      // 故事骨架（XML格式）
  adaptationStrategy: string,  // 改编策略（XML格式）
  script: Array<{             // 剧本列表（存储在 o_script 表）
    id?: number,
    name: string,
    content: string
  }>
}
```

---

## 八、AI调用机制

### 模型配置解析

通过 `u.Ai.Text(key, think, thinkLevel)` 获取AI实例：

```
key格式: "scriptAgent:{agentName}"
  - scriptAgent:decisionAgent     → 决策层
  - scriptAgent:storySkeletonAgent → 骨架执行层
  - scriptAgent:adaptationStrategyAgent → 策略执行层
  - scriptAgent:scriptAgent        → 剧本执行层
  - scriptAgent:supervisionAgent   → 监督层
```

key 用于从数据库中查找对应的模型配置和 vendor 代码。

### 调用方式

- **流式调用**: `.stream()` — 用于决策层和执行层，支持实时流式输出
- **同步调用**: `.invoke()` — 用于记忆系统内部的摘要生成和相关性判断

### 思考模式

前端可通过 `updateThinkConfig` 事件动态调整：
- `think: boolean` — 是否启用深度思考
- `thinlLevel: 0|1|2|3` — 思考深度级别

---

## 九、执行流程总览

### 完整改编流程

```
1. 用户连接 Socket.IO → JWT认证
2. 用户发送消息 → 触发 runDecisionAI()
3. 决策层加载提示词 + 记忆 + 项目信息 → AI调用
4. 项目初始化 → 确认参数 → 校验章节
5. 阶段1: 派发骨架任务 → 执行层生成 → 监督层审核 → 用户确认
6. 阶段2: 派发策略任务 → 执行层生成 → 监督层审核 → 用户确认
7. 阶段3: 循环逐集派发剧本任务（每次1集，单次上限5集）
8. 全程记忆存储 → 后续对话可回溯
```

### 子Agent执行流程

```
决策层调用工具 → 当前消息完成 → 创建子Agent消息
  → 加载技能文件 → AI流式调用 → consumeFullStream处理响应
  → 响应存入记忆 → 子消息完成 → 创建新决策层消息继续交互
```

---

## 十、设计特点与约束

### 设计特点

1. **提示词外部化**: 所有技能提示词存储在 `data/skills/` 目录，运行时可编辑，无需重新构建
2. **严格分层**: 决策层不读取工作区、不接管执行；监督层不修改内容；各层职责清晰
3. **流式体验**: 全链路流式输出，用户可实时看到AI思考和生成过程
4. **可中断**: 每次请求配备 AbortController，支持用户随时中止
5. **跨会话记忆**: 三层记忆结构确保长项目多会话的上下文连续性
6. **自动审核**: 执行完成后自动触发质量审核，形成闭环质量控制

### 关键约束

- 派发指令正文不超过100字（子Agent已有完整技能指令）
- 阶段1-2必须串行，审核与执行串行
- 剧本编写单次循环上限5集（防止上下文超载）
- 决策层严禁自行接管执行层任务
- 子Agent失败时严禁触发审核流程
