---
name: activity-development
description: This skill should be used when the user asks to "创建活动", "开发新活动", "添加活动", "快速开发活动", "Activity 框架使用", or needs guidance on Activity framework development workflow. Provides 4-step development process and code templates.
version: 1.0.0
trigger_phrases:
  - "创建活动"
  - "开发新活动"
  - "添加活动"
  - "快速开发活动"
  - "Activity 框架"
  - "Activity 开发"
---

# Activity 快速开发技能

## 概述

基于现有 Activity 框架开发新活动时使用此 skill。框架采用事件驱动架构，通过 Sentry 监听事件并执行开关逻辑。

## 框架核心组件

| 组件 | 文件路径 | 职责 |
|------|----------|------|
| **ActivityManager** | `Assets/Scripts/Activity/ActivityManager.cs` | 监听事件、判断条件、调用 Sentry 执行开关 |
| **ActivityScopeManager** | `Assets/Scripts/Activity/ActivityScopeManager.cs` | 管理每个活动类型的 DI 容器 Scope |
| **ActivityResolver** | `Assets/Scripts/Activity/ActivityResolver.cs` | 提供静态 API 调用活动数据管理器 |
| **IActivitySentry** | `Assets/Scripts/Activity/Sentry/IActivitySentry.cs` | 活动哨兵接口 |
| **AActivitySentry** | `Assets/Scripts/Activity/Sentry/AActivitySentry.cs` | 活动哨兵基类，处理开关逻辑 |
| **AEventDataManager** | `Assets/Scripts/Activity/Data/AEventDataManager.cs` | 活动数据管理器基类 |

## 4 步开发流程

### 步骤 1: 定义活动数据 (IEventData)

继承 `AEventData`，定义活动所需的业务数据字段。

参考模板: **`references/templates.md`** - EventData 模板

### 步骤 2: 定义数据管理器 (AEventDataManager)

继承 `AEventDataManager<TD>` (泛型版本)，实现业务逻辑方法。

参考模板: **`references/templates.md`** - EventDataManager 模板

### 步骤 3: 定义活动哨兵 (IActivitySentry)

继承 `AActivitySentry<TM, TD>`，使用 `[ActivitySentryAttribute(eventType, eventSubType)]` 标记，实现生命周期钩子。

参考模板: **`references/templates.md`** - ActivitySentry 模板

### 步骤 4: 定义主页入口按钮 (可选)

继承 `AbstractHomeEntranceBtn<TD>`，实现图标、点击逻辑等。

参考模板: **`references/templates.md`** - HomeEntranceBtn 模板

## 完整示例

参考 **`references/examples.md`** - 包含扭蛋机活动的完整实现代码：
- GachaEventData - 扭蛋活动数据
- GachaEventDataManager - 扭蛋活动数据管理器
- GachaActivitySentry - 扭蛋活动哨兵
- GachaHomeEntranceBtn - 扭蛋入口按钮

## 关键实现要点

### ActivitySentry 生命周期钩子

| 钩子 | 调用时机 |
|------|----------|
| `OnEventActive` | 活动开始/激活时 |
| `OnEventChanged` | 活动数据变更时 |
| `OnEventClosed` | 活动关闭时 |

### 使用 ActivityResolver 获取数据管理器

```csharp
var manager = ActivityResolver.Resolve<GachaEventDataManager>();
var data = manager?.Data;
```

### 注册活动 Scope

在 `ActivityScopeManager` 中确保活动类型的 Scope 已注册。

## Additional Resources

### Reference Files
- **`references/templates.md`** - 完整代码模板 (EventData, EventDataManager, ActivitySentry, HomeEntranceBtn)
- **`references/examples.md`** - 扭蛋机活动完整示例

### 相关框架文件
- `Assets/Scripts/Activity/ActivityManager.cs`
- `Assets/Scripts/Activity/ActivityResolver.cs`
- `Assets/Scripts/Activity/Sentry/AActivitySentry.cs`
- `Assets/Scripts/Activity/Data/AEventDataManager.cs`
