---
name: activity-architecture-refactoring
description: 当用户讨论"活动架构重构"、"迁移到新活动框架"、"重构 FishingEventData"、"旧活动代码重构"或需要将旧活动系统迁移到新 Activity 框架时使用此 skill。提供新旧架构对比和渐进式重构方法论。
version: 1.0.0
trigger_phrases:
  - "活动架构重构"
  - "迁移到新活动框架"
  - "重构 FishingEventData"
  - "旧活动代码重构"
  - "活动系统迁移"
  - "Activity 重构"
  - "新活动框架"
---

# 活动架构重构方法论

## 概述

本文档提供将旧活动系统 (`DataCenter.FishingEventData`) 渐进式迁移到新 Activity 框架 (`Activity.ActivityManager`) 的方法论和最佳实践。

## 背景：新旧架构对比

### 旧架构 (FishingEventData.cs)

| 特征 | 实现方式 |
|------|----------|
| **代码规模** | ~2700行巨型单例类 |
| **活动类型处理** | switch-case 硬编码 (Type 1-15, 各子类型分别处理) |
| **条件检查** | 散落在 `GetEventCondition`、`Condition` 等方法中 |
| **活动开启** | 每个类型对应 `SetEvent1`~`SetEvent15` 方法 |
| **依赖管理** | 直接 `GContext.container.Resolve<XXX>()` |
| **状态存储** | 手动维护 `limitedTimeEventDic`、`fixedTimeEventDic` |
| **事件触发** | 遍历整个配置表 `TbFishingEvent.DataList` |

**核心问题**:
- 高耦合: 所有活动类型写在一个类里
- 难以扩展: 新增活动类型需修改主类
- 难以测试: 逻辑混杂在一起
- 维护困难: 2700行代码难以理解

### 新架构 (ActivityManager.cs)

| 特征 | 实现方式 |
|------|----------|
| **架构模式** | 依赖注入 + Sentry + ConditionChecker |
| **活动类型处理** | 通过 `ActivitySentryManager` 动态获取 Sentry 类型 |
| **条件检查** | `ConditionChecker` 插件化，每个条件独立实现 |
| **活动开启** | `IActivitySentry.Open()/Close()` 抽象接口 |
| **依赖管理** | `ActivityScopeManager` 作用域管理 |
| **高效查询** | `_dataTypeIndex` 按数据类型建立索引 |
| **事件触发** | 数据变化时只触发相关活动，非全表遍历 |

**核心优势**:
- 低耦合: 每个活动独立的 Sentry 类
- 可扩展: 新增活动只需添加新 Sentry
- 可测试: 逻辑封装在独立类中
- 职责清晰: 条件检查、活动执行、数据管理分离

## 重构策略

### 渐进式迁移原则

```
┌─────────────────────────────────────────────────────────┐
│                    迁移阶段图                             │
├─────────────────────────────────────────────────────────┤
│                                                         │
│   Phase 1          Phase 2          Phase 3            │
│   ┌─────┐          ┌─────┐          ┌─────┐           │
│   │ 新  │   →      │ 混  │    →     │ 旧  │           │
│   │ Sentry│        │ 合  │          │ 移除│           │
│   └─────┘          └─────┘          └─────┘           │
│                                                         │
│   新活动用新框架    旧活动逐个迁移    全面切换          │
└─────────────────────────────────────────────────────────┘
```

### 核心原则

1. **不破坏现有功能**: 迁移过程中旧系统必须正常工作
2. **逐个迁移**: 每个活动单独迁移，降低风险
3. **双向调用**: 迁移过程中允许新旧代码相互调用
4. **数据兼容**: 确保数据格式兼容，平滑过渡

## 迁移步骤

### Phase 1: 准备阶段

#### 1.1 分析旧活动类型

```csharp
// 旧架构中的活动类型分布
switch (t.Type)
{
    case 1: SetEvent1(t); break;  // 系统解锁类
    case 2: SetEvent2(t); break;  // 每日任务类
    case 3: SetEvent3(t); break;  // 收集类
    case 4: SetEvent4(t); break;  // 寻宝/钓鱼/魔药/弹珠/宾果
    case 5: SetEvent5(t); break;  // 社交类
    case 6: SetEvent6(t); break;  // 大转盘
    case 7: SetEvent7(t); break;  // 竞技场
    case 8: SetEvent8(t); break;  // 排行榜
    // ... more types
}
```

#### 1.2 创建对应的新 Sentry

根据每个 `SetEventN` 方法，创建对应的 `IActivitySentry` 实现。

### Phase 2: 逐个迁移

#### 2.1 迁移模式

```
旧代码中的 SetEventN(t)
         ↓
提取条件检查逻辑到 ConditionChecker
         ↓
提取活动开启逻辑到 OnEventActive 钩子
         ↓
提取活动关闭逻辑到 OnEventClosed 钩子
         ↓
在旧代码中调用新 ActivityManager
```

#### 2.2 迁移检查清单

| 序号 | 迁移项 | 旧代码位置 | 新代码实现 |
|------|--------|-----------|-----------|
| 1 | 条件检查器 | `GetEventCondition()` | `IConditionChecker` |
| 2 | 活动开启逻辑 | `SetEventN()` | `OnEventActive()` |
| 3 | 活动关闭逻辑 | (implicit) | `OnEventClosed()` |
| 4 | 数据管理器 | 各类型特有 | `AEventDataManager<T>` |
| 5 | 数据存储 | `limitedTimeEventDic` | `EventDataRepository` |

### Phase 3: 清理阶段

#### 3.1 移除旧代码标志

当一个活动完全迁移后：
- 删除对应的 `SetEventN` 方法
- 删除对应的 `Condition` 分支
- 确保 `ActivityManager` 正常工作

#### 3.2 最终清理

- 移除 `FishingEventData` 中的活动相关方法
- 保留数据兼容相关如 `方法 (InitData`)
- 最终删除整个 `FishingEventData` 类

## 关键迁移点映射

### 条件检查迁移

```csharp
// 旧代码
bool GetEventCondition(EventCondition eventCondition)
{
    switch (eventCondition.Condition)
    {
        case ConditionType.AccountLevel:
            return GContext.container.Resolve<PlayerData>().lv >= int.Parse(eventCondition.Param[0]);
        case ConditionType.GetFishCount:
            return GContext.container.Resolve<PlayerFishData>().GetAnglingCount() >= int.Parse(eventCondition.Param[0]);
        // ... more conditions
    }
}

// 新代码 - 创建独立的 ConditionChecker
public class AccountLevelConditionChecker : IConditionChecker
{
    public bool CanOpen(FishingEvent evt)
    {
        var param = evt.ConditionList[0].Param[0];
        return GContext.container.Resolve<PlayerData>().lv >= int.Parse(param);
    }
}
```

### 活动开启逻辑迁移

```csharp
// 旧代码 - SetEvent1 示例
void SetEvent1(FishingEvent t)
{
    switch (t.SubType)
    {
        case 3:
            GContext.container.Resolve<PlayerShopData>().vipTipforUnlocked = t.ID;
            break;
        case 4:
            GContext.container.Resolve<PlayerShopData>().shopTipforUnlocked = t.ID;
            break;
        // ...
    }
}

// 新代码 - 独立的 Sentry 类
[ActivitySentry(1, 3)]
public class VipUnlockActivitySentry : AActivitySentry<ShopDataManager, VipUnlockData>
{
    protected override void OnEventActive(FishingEvent evt)
    {
        DataManager.VipTipforUnlocked = evt.ID;
    }
}
```

### 数据存储迁移

```csharp
// 旧代码
Dictionary<int, LimitedTimeEvent> limitedTimeEventDic = new Dictionary<int, LimitedTimeEvent>();
Dictionary<int, FixedTimeEvent> fixedTimeEventDic = new Dictionary<int, FixedTimeEvent>();

// 新代码 - 使用 EventDataRepository
public class EventDataRepository : IEventDataRepository
{
    private Dictionary<int, LimitedTimeEvent> _limitedTimeEvents = new();
    private Dictionary<int, FixedTimeEvent> _fixedTimeEvents = new();

    // 自动持久化
    public void Save() { /* ... */ }
}
```

## 迁移示例：Type 1 系统解锁活动

### Step 1: 分析旧逻辑

```csharp
// FishingEventData.SetEvent1
void SetEvent1(FishingEvent t)
{
    switch (t.SubType)
    {
        case 3: // VIP 解锁提示
            GContext.container.Resolve<PlayerShopData>().vipTipforUnlocked = t.ID;
            break;
        case 4: // 商店解锁提示
            GContext.container.Resolve<PlayerShopData>().shopTipforUnlocked = t.ID;
            break;
        case 5: // 俱乐部解锁提示
            GContext.container.Resolve<ClubService>().socialTipforUnlocked = t.ID;
            break;
        // ...
    }
}
```

### Step 2: 创建新 Sentry

```csharp
[ActivitySentry(1, 3)]  // Type=1, SubType=3
public class VipUnlockSentry : AActivitySentry<PlayerShopData, VipUnlockData>
{
    protected override void OnEventActive(FishingEvent evt)
    {
        DataManager.VipTipforUnlocked = evt.ID;
    }

    protected override void OnEventClosed(FishingEvent evt)
    {
        DataManager.VipTipforUnlocked = -1;
    }
}
```

### Step 3: 迁移条件检查

```csharp
// 创建对应的 ConditionChecker
public class AccountLevelConditionChecker : IConditionChecker
{
    public bool CanOpen(FishingEvent evt)
    {
        var level = int.Parse(evt.ConditionList[0].Param[0]);
        return GContext.container.Resolve<PlayerData>().lv >= level;
    }
}
```

### Step 4: 双轨运行

```csharp
// 在旧代码中保留调用，同时调用新框架
void SetEvent1(FishingEvent t)
{
    // 旧逻辑保留 (临时)
    switch (t.SubType) { /* ... */ }

    // 新逻辑调用
    var sentry = ActivityManager.Instance.GetSentry((t.Type, t.SubType));
    sentry?.Open(t);
}
```

### Step 5: 验证后移除旧代码

确认迁移成功后，删除 `SetEvent1` 方法中的旧逻辑。

## 注意事项

### 向后兼容

1. **数据格式**: 确保旧代码序列化的数据能被新代码读取
2. **事件触发**: 新旧系统必须同时响应关键事件
3. **回滚机制**: 迁移失败时能快速回退到旧代码

### 性能考虑

1. **避免重复**: 迁移完成后移除旧代码的重复逻辑
2. **索引优化**: 新架构的 `_dataTypeIndex` 可大幅提升性能
3. **按需加载**: 新架构支持按活动类型按需加载

### 测试策略

1. **单元测试**: 为每个新 Sentry 编写单元测试
2. **集成测试**: 确保新旧系统数据同步
3. **回归测试**: 确保原有功能不受影响

## 常见问题

### Q: 旧活动的数据如何迁移?

A: 优先保持数据格式兼容。如果必须迁移，创建数据迁移脚本在登录时执行。

### Q: 如何处理复杂的条件组合?

A: 新架构的 `ConditionChecker` 支持组合模式:
```csharp
// AND 组合
checkers.All(c => c.CanOpen(evt))

// OR 组合
checkers.Any(c => c.CanOpen(evt))
```

### Q: 迁移过程中如何调试?

A:
1. 在新旧系统中添加相同的日志
2. 比对两者的行为一致性
3. 使用 Feature Flag 控制新旧代码的执行

## 参考资料

### 核心文件
- `Assets/Scripts/Activity/ActivityManager.cs` - 新活动管理器
- `Assets/Scripts/DataCenter/DataCenter.FishingEventData.cs` - 旧活动数据类
- `Assets/Scripts/Activity/Sentry/IActivitySentry.cs` - Sentry 接口
- `Assets/Scripts/Activity/Sentry/AActivitySentry.cs` - Sentry 基类
- `Assets/Scripts/Activity/Condition/IConditionChecker.cs` - 条件检查器接口

### 相关 Skill
- `activity-development` - 新活动开发流程
- `data-presentation-separation` - 活动数据与表现分离
