---
name: game-design
description: |
  基于制作人方法论的策划方案审核、优化、设计工具。
  采用双层协作模式：AI做逻辑完整性检查（6大检查维度），人做直觉判断（Gate/Flag/Sense决策问题）。
  品类无关，可处理任何游戏的策划方案。

  **自动加载场景**：
  - 执行 /game.review、/game.optimize、/game.craft 命令时
  - 用户要求审核、评审、优化策划方案时
  - 用户提交设计文档要求检查时
  - 用户讨论活动设计、数值平衡、玩家分层、付费公平性、经济系统、情绪节奏时
  - 用户提到设计红线、设计误区、价值排序、核心循环时

  **关键词触发**：审核、评审、review、检查设计、优化方案、方案评估、
  策划审查、设计评分、红线检查、节奏分析、价值冲突、game design、
  活动设计、数值平衡、玩家分层、付费设计、经济系统、核心循环、
  设计方案、策划案、craft、optimize
---

# 策划方案设计工具（Game Design）

## 双层协作架构

```
┌─────────────────────────────────┐
│         人（直觉层）             │
│  目标感判断 / 有趣判断 / sense  │
└──────────┬──────────────────────┘
           ↕ 决策问题 + 回答
┌──────────┴──────────────────────┐
│         AI（逻辑层）             │
│  6大检查维度（子代理并行）       │
│  聚合 + 决策问题生成            │
└─────────────────────────────────┘
```

## 命令路由

| 命令 | 用途 | Workflow 文件 |
|------|------|--------------|
| `/game.review` | 审核已有方案 | [review-workflow.md](references/review-workflow.md) |
| `/game.optimize` | 优化已有方案 | [optimize-workflow.md](references/optimize-workflow.md) |
| `/game.craft` | 从已有想法设计 | [craft-workflow.md](references/craft-workflow.md) |
| `/game.uxdesign` | 生成 UI 线框图 | **预留，暂不实现** |

收到命令后，读取对应 workflow 文件执行。

## I/O 规范

### 输入

- 所有输入文件放在**当前工作目录**的 `input/` 下
- 来源：用户手动放入 或 通过 Outline 子代理拉取
- **运行前必须**：扫描 input/，列出文件清单（标注类型），让用户确认读取范围
- 支持类型：md/txt/doc(x)（文档）、png/jpg/gif（图片）、mp4/mov/webm（视频）

### 输出

- 所有产出写入**当前工作目录**的 `output/` 下
- 命名规则：`output/{命令}-{YYYY-MM-DD}.md`
- optimize 额外输出：`output/optimize-diff-{YYYY-MM-DD}.md`
- 去向：留在本地 或 通过 Outline 子代理上传

## 处理链路

```
input/（用户放入 或 Outline 下载子代理拉取）
    ↓
[确认读取范围] — 列出文件清单，用户确认
    ↓
[媒体预处理子代理 sonnet]（仅 input/ 含图片/视频时）
  → 生成 input/media-descriptions.md
    ↓
[主代理 opus] review / optimize / craft
  - 读取文档 + media-descriptions.md（纯文本）
  - 输出保留所有媒体引用路径
    ↓
output/（完整方案/审核报告/diff）
    ↓（可选）
[Outline 上传子代理 sonnet] → 发布到 Outline
```

## 媒体预处理子代理

**触发条件**：input/ 中存在图片或视频文件时自动触发。
**职责**：图片→文字描述，视频→关键帧抽取→文字描述，输出 `input/media-descriptions.md`。
**Prompt 详情**：见 [subagent-prompts.md](references/subagent-prompts.md)

## Outline 子代理（可选）

**触发条件**：用户指定从 Outline（wiki.bamboogames.top）读取或要求上传时。
**职责**：下载文档+附件到 input/，或将 output/ 上传到 Outline。
**Prompt 详情**：见 [subagent-prompts.md](references/subagent-prompts.md)

## 引擎裁剪速查

| 方案类型 | 必要维度 | 可选维度 |
|----------|---------|---------|
| 完整策划案 | 全部6个 | — |
| 单系统设计 | Purpose, Rhythm, RedLine | Value, Segment, Economy |
| 数值方案 | Purpose, RedLine, Economy | Rhythm, Value, Segment |
| 活动/运营方案 | Purpose, RedLine, Value | Rhythm, Segment, Economy |
| UI/交互方案 | Purpose, Rhythm, RedLine | — |

## 子代理总览

| 角色 | 模型 | 触发时机 |
|------|------|---------|
| 媒体预处理 | sonnet | input/ 含图片/视频 |
| 审核维度引擎（×6） | sonnet | /game.review 或 /game.craft 阶段3 |
| Outline 下载 | sonnet | 用户指定从 Outline 读取 |
| Outline 上传 | sonnet | 用户要求发布到 Outline |

## 媒体引用保留规则

1. 媒体引用是不可变锚点——路径和文件名不得删除或修改
2. 位置可调整——跟随所在段落移动
3. 不内联媒体内容——输出保持相对路径引用
4. 新增媒体——以文字占位符标注（如 `[建议插入：XX 流程图]`）

## Reference 文件索引

| 文件 | 内容 |
|------|------|
| [review-workflow.md](references/review-workflow.md) | review 三阶段工作流 |
| [optimize-workflow.md](references/optimize-workflow.md) | optimize 五阶段工作流 |
| [craft-workflow.md](references/craft-workflow.md) | craft 四阶段工作流 |
| [craft-info-gathering.md](references/craft-info-gathering.md) | craft 信息收集协议 |
| [review-dimensions.md](references/review-dimensions.md) | 6 检查维度（检查项 + Prompt + 输出格式） |
| [decision-protocol.md](references/decision-protocol.md) | Gate/Flag/Sense 决策协议 |
| [producer-methodology.md](references/producer-methodology.md) | 制作人方法论（所有检查的知识源头） |
| [templates.md](references/templates.md) | 输出模板（审核报告 + 优化报告 + 策划文档） |
| [project-context.md](references/project-context.md) | 项目专属数据（设计模式 + ARPU + 框架，可整体替换） |
| [subagent-prompts.md](references/subagent-prompts.md) | 子代理 Prompt 规范（媒体预处理 + Outline 上下载） |
