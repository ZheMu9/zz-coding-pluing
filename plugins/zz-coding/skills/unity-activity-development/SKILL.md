---
name: unity-activity-development
description: 用于 Unity 游戏活动功能开发，尤其适合用户提到活动开发、活动面板、活动数据管理、配置驱动活动、EventDataManager、活动 UI、活动入口按钮、活动表现联动、TV 动画或活动代码重构时。
version: 0.1.0
trigger_phrases:
  - "Unity 活动开发"
  - "活动功能开发"
  - "活动面板"
  - "活动数据管理"
  - "EventDataManager"
  - "活动入口按钮"
  - "活动配置驱动"
  - "活动表现联动"
---

# Unity Activity Development

## 概述

该 skill 面向 Unity 项目中的活动功能开发，目标是帮助 Agent 在已有活动框架中快速定位并补齐以下内容：

- 配置表和活动类型
- 数据结构与数据管理器
- 面板 / 入口 / 红点 / 倒计时
- 表现层与真实数据的边界
- 验证路径与回归点

如果需求包含 TV 动画、延迟表现、结算回放、先改数据后播表现等时序问题，应联动使用 `data-presentation-separation`。

如果需求是“基于既有 Activity 框架新增一个标准活动”，也可以参考 `activity-development`。

## 使用原则

本 skill 不强推脱离项目现实的“理想架构”，优先遵循以下原则：

1. 先贴合当前项目主流结构
2. 优先做渐进式优化，而不是一次性大重构
3. 不改大框架的前提下，先把职责边界理顺
4. 对异步表现问题，优先用 `PresentationData` 解决
5. 对面板复杂展示问题，优先用 `ViewData` 解决

## 先读哪个参考文件

按需求选择：

- 需要快速套模板时，读 `references/templates.md`
- 需要判断“当前项目里最适合怎么拆”时，读 `references/examples.md`
- 需要判断“该先抽 ViewData、先补 PresentationData，还是先收口刷新”时，读 `references/refactor-checklist.md`
- 需要识别常见坏味道并给出最小修法时，读 `references/anti-patterns.md`
- 需要输出稳定的 review / 重构建议格式时，读 `references/review-output-template.md`
- 需求同时涉及主面板、TV 动画、数据快照时，两个都读

## 开发前先确认

在开始编码前，先确认下面 6 件事：

1. 活动是新增类型，还是已有类型下的新子玩法
2. 配置表中有哪些字段，客户端真正需要哪些字段
3. 活动状态是服务端驱动、客户端推导，还是两者混合
4. UI 是单面板还是多弹窗 / 多分页
5. 是否存在延迟动画、领奖回放、异步刷新
6. 成功标准是什么：只要能跑通，还是要求后续可复用

## 推荐开发顺序

### 1. 明确活动模型

先梳理：

- 活动唯一标识：`eventType` / `eventSubType`
- 核心状态：进行中、未开启、已结算、已结束
- 核心资源：次数、积分、进度、奖励、倒计时
- 触发点：登录、推配置、战斗结算、定时刷新、领奖

如果这些还不清楚，不要急着写 UI，先把状态流转画清楚。

### 2. 数据层优先

优先明确数据对象和管理入口，典型职责如下：

| 层级 | 典型职责 |
| ---- | -------- |
| Config | 静态规则、奖励、阈值、文案映射 |
| Runtime Data | 当前分数、次数、阶段、奖励状态 |
| Manager | 读写运行态、处理推包、派发刷新 |
| Presenter/ViewModel | 给 UI 的可展示数据 |
| Panel/View | 纯展示、交互转发 |

尽量避免 UI 直接拼业务规则。

### 3. UI 层只读展示数据

面板层建议只消费“可展示结果”，不要直接读取原始运行态后自行推导复杂规则。

推荐模式：

```text
原始活动数据 -> 业务整理 / Presenter -> 面板刷新
```

如果有异步动画、延迟播放、结算回放，展示层应优先使用快照数据，而不是直接读会变化的实时状态。

### 4. 刷新入口收口

统一列出所有刷新来源：

- 打开面板时初始化
- 服务端推包后刷新
- 倒计时 tick 刷新
- 用户点击领奖后刷新
- 玩法结算后刷新

建议把刷新收口成几个固定方法，例如：

```csharp
RefreshAll();
RefreshStaticView();
RefreshRuntimeView();
RefreshRewardState();
RefreshCountdown();
```

不要在多个点击回调里散落重复 UI 刷新逻辑。

### 5. 表现与数据分离

遇到以下场景时，优先考虑表现数据快照：

- 先扣次数，再播动画
- 先领奖，再播奖励飞行动画
- 先更新 Boss/NPC 血量，再播击杀表现
- 面板关闭后异步表现还在继续

此时建议保存一份“表现所需数据”，而不是依赖后续已变化的真实运行态。

## 常见代码组织建议

### 数据管理器

- 提供清晰的只读访问接口
- 推包更新和本地派生逻辑分开
- 对外暴露语义化方法，不暴露可随意改写的数据结构

示例：

```csharp
public class DemoEventDataManager
{
    public DemoEventData Data { get; private set; }

    public bool CanClaimReward(int rewardId) { /* ... */ return false; }
    public int GetRemainCount() { /* ... */ return 0; }
    public long GetEndTimestamp() { /* ... */ return 0; }
}
```

### 面板层

- 只负责绑定按钮、刷新视图、处理开关动画
- 复杂业务判断尽量下沉到 Manager 或 Presenter

示例：

```csharp
private void RefreshAll()
{
    var viewData = BuildViewData();
    RefreshTopInfo(viewData);
    RefreshRewardList(viewData);
    RefreshButtons(viewData);
}
```

### 展示数据

当 UI 逻辑越来越多时，建议增加展示模型：

```csharp
public struct DemoEventViewData
{
    public int Score;
    public int RemainCount;
    public bool CanPlayAnim;
    public bool ShowRedDot;
    public string CountdownText;
}
```

更完整模板见 `references/templates.md`。

## 实现检查清单

- [ ] 活动类型和配置入口已确认
- [ ] 运行态数据结构已明确
- [ ] Manager 的职责边界清晰
- [ ] Panel 不直接承载过多业务规则
- [ ] 异步表现是否需要快照已判断
- [ ] 刷新入口已统一收口
- [ ] 倒计时 / 红点 / 奖励状态已覆盖
- [ ] 关闭面板后的异步行为已考虑
- [ ] 至少有 1 条主流程和 1 条边界流程验证

## 推荐落地顺序

在当前项目里，优先使用下面的演进顺序：

1. 先保留既有 `Manager + Panel` 主结构
2. 把分散的 UI 刷新收口成统一入口
3. 把展示判断下沉为 `BuildViewData()`
4. 如果有延迟表现，再补 `PresentationData`
5. 只有在展示逻辑非常复杂时，才考虑单独抽 `Presenter`

更完整说明见 `references/examples.md`。

如果需求重点是“已有活动代码很乱，先怎么改最稳”，优先读 `references/refactor-checklist.md`。

如果需求重点是“这段活动代码有哪些坏味道、怎么低成本优化”，优先读 `references/anti-patterns.md`。

如果需求重点是“帮我 review 这段活动代码”或“给我一版稳定的重构建议输出”，优先读 `references/review-output-template.md`。

## 典型提示词

当用户需求类似下面内容时，优先套用本 skill：

- “帮我新增一个 Unity 活动面板”
- “这个活动的数据管理怎么拆”
- “活动领奖后 UI 状态不一致”
- “活动表现播放时数据已经被改掉了”
- “想把活动代码整理成可复用模式”

## 输出偏好

当使用此 skill 产出方案或代码时，优先给出：

1. 受影响的核心类
2. 数据流和刷新流
3. 是否需要展示快照
4. 最小可落地改法
5. 验证点和回归风险

## Additional Resources

- `references/templates.md`: 贴近当前 Unity 活动项目的模板集合
- `references/examples.md`: 更贴近现有项目风格的分层与演进示例
- `references/refactor-checklist.md`: 判断该先收口刷新、先抽 `ViewData` 还是先补 `PresentationData`
- `references/anti-patterns.md`: 常见坏味道与对应的最小修法
- `references/review-output-template.md`: 活动代码 review / 重构建议的推荐输出格式
