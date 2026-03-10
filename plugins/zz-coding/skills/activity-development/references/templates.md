# Activity 开发代码模板

本文档提供 Activity 框架开发的完整代码模板。

## 1. EventData 模板

文件位置: `Assets/Scripts/{活动目录}/Data/{活动名}Data.cs`

```csharp
using System;
using UnityEngine;

namespace game
{
    /// <summary>
    /// {活动名}活动数据
    /// </summary>
    [Serializable]
    public class {活动名}Data : AEventData
    {
        /// <summary>
        /// 活动ID
        /// </summary>
        public int EventId;

        /// <summary>
        /// 活动类型
        /// </summary>
        public int EventType;

        /// <summary>
        /// 活动子类型
        /// </summary>
        public int EventSubType;

        /// <summary>
        /// 活动开始时间戳
        /// </summary>
        public long StartTime;

        /// <summary>
        /// 活动结束时间戳
        /// </summary>
        public long EndTime;

        /// <summary>
        /// 业务数据字段1
        /// </summary>
        public int BusinessField1;

        /// <summary>
        /// 业务数据字段2
        /// </summary>
        public string BusinessField2;

        /// <summary>
        /// 业务数据字段3
        /// </summary>
        public Vector3 BusinessField3;

        /// <summary>
        /// 业务数据字段4
        /// </summary>
        public List<int> BusinessField4 = new();

        public override int GetEventId() => EventId;

        public override int GetEventType() => EventType;

        public override int GetEventSubType() => EventSubType;

        public override long GetStartTime() => StartTime;

        public override long GetEndTime() => EndTime;
    }
}
```

## 2. EventDataManager 模板

文件位置: `Assets/Scripts/{活动目录}/Data/{活动名}Manager.cs`

```csharp
using System;
using System.Collections.Generic;
using game.Activity;
using UnityEngine;

namespace game
{
    /// <summary>
    /// {活动名}活动数据管理器
    /// </summary>
    public class {活动名}Manager : AEventDataManager<{活动名}Data>
    {
        /// <summary>
        /// 业务方法：检查是否满足某个条件
        /// </summary>
        public bool CheckCondition(int conditionId)
        {
            if (Data == null) return false;
            return Data.BusinessField1 >= conditionId;
        }

        /// <summary>
        /// 业务方法：增加进度
        /// </summary>
        public void AddProgress(int amount)
        {
            if (Data == null) return;
            Data.BusinessField1 += amount;
            SaveData();
        }

        /// <summary>
        /// 业务方法：重置数据
        /// </summary>
        public void ResetData()
        {
            if (Data == null) return;
            Data.BusinessField1 = 0;
            Data.BusinessField2 = string.Empty;
            Data.BusinessField4.Clear();
            SaveData();
        }

        /// <summary>
        /// 业务方法：领取奖励
        /// </summary>
        public bool ClaimReward(int rewardId)
        {
            if (Data == null) return false;
            // 实现奖励领取逻辑
            return true;
        }

        protected override void OnDataLoaded({活动名}Data data)
        {
            base.OnDataLoaded(data);
            // 数据加载完成后的初始化逻辑
        }

        protected override void OnDataReset()
        {
            base.OnDataReset();
            // 数据重置后的逻辑
        }
    }
}
```

## 3. ActivitySentry 模板

文件位置: `Assets/Scripts/{活动目录}/Sentry/{活动名}ActivitySentry.cs`

```csharp
using game.Activity;
using game.Activity.Sentry;

namespace game
{
    /// <summary>
    /// {活动名}活动哨兵
    /// </summary>
    [ActivitySentryAttribute(EventType.{事件类型}, EventSubType.{事件子类型})]
    public class {活动名}ActivitySentry : AActivitySentry<{活动名}Manager, {活动名}Data>
    {
        /// <summary>
        /// 活动激活时调用
        /// </summary>
        protected override void OnEventActive({活动名}Data data)
        {
            base.OnEventActive(data);
            // 活动开始时的逻辑
            // 例如：显示入口按钮、弹出公告、初始化资源等
        }

        /// <summary>
        /// 活动数据变更时调用
        /// </summary>
        protected override void OnEventChanged({活动名}Data oldData, {活动名}Data newData)
        {
            base.OnEventChanged(oldData, newData);
            // 活动数据变更时的逻辑
            // 例如：更新UI、刷新奖励状态等
        }

        /// <summary>
        /// 活动关闭时调用
        /// </summary>
        protected override void OnEventClosed({活动名}Data data)
        {
            // 活动结束时的逻辑
            // 例如：隐藏入口按钮、弹出结束提示、清理临时资源等
            base.OnEventClosed(data);
        }

        /// <summary>
        /// 活动是否正在进行
        /// </summary>
        public override bool IsEventOngoing({活动名}Data data)
        {
            if (data == null) return false;
            var now = TimeUtils.GetCurrentTimestamp();
            return now >= data.StartTime && now < data.EndTime;
        }
    }
}
```

### ActivitySentryAttribute 参数说明

| 参数 | 说明 |
|------|------|
| `eventType` | 活动主类型，如 EventType.Gacha |
| `eventSubType` | 活动子类型，用于区分同一类型的多个活动 |

## 4. HomeEntranceBtn 模板

文件位置: `Assets/Scripts/{活动目录}/HomeEntranceBtn/{活动名}HomeEntranceBtn.cs`

```csharp
using UnityEngine;
using UnityEngine.UI;
using game.Activity;

namespace game
{
    /// <summary>
    /// {活动名}主页入口按钮
    /// </summary>
    public class {活动名}HomeEntranceBtn : AbstractHomeEntranceBtn<{活动名}Data>
    {
        [SerializeField]
        private Image _iconImage;

        [SerializeField]
        private Text _countText;

        [SerializeField]
        private GameObject _redDot;

        /// <summary>
        /// 入口按钮点击时调用
        /// </summary>
        protected override void OnBtnClick()
        {
            base.OnBtnClick();
            // 点击入口按钮的逻辑
            // 例如：打开活动面板
        }

        /// <summary>
        /// 刷新入口按钮显示
        /// </summary>
        protected override void RefreshEntrance({活动名}Data data)
        {
            base.RefreshEntrance(data);
            if (data == null)
            {
                gameObject.SetActive(false);
                return;
            }

            // 检查活动是否正在进行
            var now = TimeUtils.GetCurrentTimestamp();
            var isOngoing = now >= data.StartTime && now < data.EndTime;
            gameObject.SetActive(isOngoing);

            if (isOngoing)
            {
                // 更新图标
                // _iconImage.sprite = LoadIcon(data.IconPath);

                // 更新红点
                if (_redDot != null)
                {
                    _redDot.SetActive(HasUnclaimedReward(data));
                }
            }
        }

        /// <summary>
        /// 检查是否有未领取的奖励
        /// </summary>
        private bool HasUnclaimedReward({活动名}Data data)
        {
            // 实现奖励检查逻辑
            return false;
        }

        /// <summary>
        /// 加载图标资源
        /// </summary>
        private Sprite LoadIcon(string iconPath)
        {
            // 实现图标加载逻辑
            return null;
        }
    }
}
```

## 5. 使用 ActivityResolver 获取数据

```csharp
using game.Activity;

// 获取活动数据管理器
var manager = ActivityResolver.Resolve<{活动名}Manager>();

if (manager != null)
{
    var data = manager.Data;
    if (data != null)
    {
        // 活动正在进行中
        var now = TimeUtils.GetCurrentTimestamp();
        var isOngoing = now >= data.StartTime && now < data.EndTime;

        if (isOngoing)
        {
            Debug.Log($"活动进行中: {data.EventId}");
            // 执行业务逻辑
        }
    }
}
```

## 6. 注意事项

1. **EventType 和 EventSubType**: 需要与后端约定或从配置表读取
2. **时间戳**: 使用 `TimeUtils.GetCurrentTimestamp()` 获取当前时间戳
3. **数据保存**: 修改数据后调用 `SaveData()` 保存
4. **DI 容器**: Activity 框架使用 DI 容器管理依赖
