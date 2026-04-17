# Review 模式：方案审核工作流

## 概述

`/game.review` 是 game-design skill 的审核模式，用于对已有策划方案进行逻辑完整性检查。通过 6 个检查维度的并行子代理（Fan-out/Fan-in）发现问题，再通过决策问题交给制作人做直觉判断。

核心原则：**AI 做逻辑层（6 维度并行检查 + 聚合），人做直觉层（Gate/Flag/Sense 决策）。**

## 与 Optimize/Craft 的关系

```
review 负责："有什么问题" + "问题多严重"
optimize 负责："怎么修"
craft 负责："从已有想法设计 + 内置审核迭代"
```

## 三阶段工作流

```
输入：input/ 中的方案文件（用户确认读取范围后）
    ↓
阶段1：读取方案 + 确定范围 + 选择引擎
    ↓
阶段2：Fan-out 子代理并行检查
    ↓
阶段3：Fan-in 聚合 + 生成决策问题 → 交给制作人
    ↓
输出：output/review-YYYY-MM-DD.md
```

### 阶段1：读取与引擎选择

**执行方式**：主代理（opus）读取方案文件 + media-descriptions.md（如有）。

**引擎选择**：根据方案类型参考 [review-dimensions.md](review-dimensions.md) 的裁剪规则表确定启用哪些引擎。

**方案类型判断规则**：
- 包含核心循环 + 数值 + 分层 + 经济 → 完整策划案
- 只涉及单个系统的设计 → 单系统设计
- 主要是数值调整/框架 → 数值方案
- 主要是活动/运营内容 → 活动/运营方案
- 主要是 UI/交互设计 → UI/交互方案
- 不确定 → 默认为完整策划案（宁可多查不可少查）

### 阶段2：Fan-out 子代理并行检查

**执行架构**：

```
主代理 opus
    ├─→ Purpose 子代理 sonnet   ─┐
    ├─→ Rhythm 子代理 sonnet    ─┤
    ├─→ RedLine 子代理 sonnet   ─┤ 并行 Fan-out
    ├─→ Value 子代理 sonnet     ─┤
    ├─→ Segment 子代理 sonnet   ─┤
    └─→ Economy 子代理 sonnet   ─┘
```

**模型分工**：
- 引擎子代理：sonnet（逻辑检查是执行性工作）
- 主代理聚合+决策问题生成：opus（需要推理能力综合判断）

每个引擎子代理的 prompt 按 [review-dimensions.md](review-dimensions.md) 中对应维度的"子代理 Prompt 模板"生成，并填入：
1. [producer-methodology.md](producer-methodology.md) 的文件路径
2. 方案文档的文件路径
3. media-descriptions.md 的内容（如有媒体描述）

### 阶段3：Fan-in 聚合 + 决策问题生成

**主代理（opus）的聚合工作**：
1. 收集所有引擎子代理的输出
2. 去重：不同引擎发现的同一问题合并
3. 按严重度排序：P0 → P1 → P2
4. 生成决策问题（按 [decision-protocol.md](decision-protocol.md) 的格式和规则）
5. 按 [templates.md](templates.md) 的 review 输出模板生成审核报告

**决策问题规则**：
- 每轮≤3个，先 Gate → 后 Flag → 后 Sense
- 尽可能提供选项
- 详见 [decision-protocol.md](decision-protocol.md)

## 异常处理

| 情况 | 处理方式 |
|------|---------|
| input/ 为空 | 报错，提示用户放入方案文件 |
| 方案文件无法判断类型 | 默认为完整策划案，全部6引擎 |
| 某个引擎子代理失败 | 报告该引擎跳过，其余引擎结果正常输出 |
| 无 P0/P1 问题 | 正常输出，标注方案质量良好 |
