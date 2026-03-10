# 扭蛋机活动完整示例

本文档提供基于 Activity 框架的扭蛋机活动完整实现示例，参考 `Assets/Scripts/Capsule/` 目录下的实际代码。

## 1. CapsulePackData - 扭蛋活动数据

文件位置: `Assets/Scripts/Capsule/Pack/Manager/CapsulePackData.cs`

```csharp
using System.Collections.Generic;
using Activity;
using cfg;
using Newtonsoft.Json;

namespace Capsule.Pack.Manager
{
    public class CapsulePackData : AEventData
    {
        public int CurrentRoundCount;
        private List<int> _currentProgress;
        public List<int> CurrentProgress => _currentProgress ??= new List<int>();

        public override void RefreshCache()
        {
            base.RefreshCache();
            CurrentProgress?.Clear();
        }

        [JsonIgnore]
        public int CurrentProgressCount => CurrentProgress?.Count ?? 0;

        [JsonIgnore]
        public EventToyMachineMain MainConfig => Tables.TbEventToyMachineMain.GetOrDefault(FishingEvent?.RedirectID ?? 0);

        [JsonIgnore]
        public override bool IsActive => base.IsActive && CurrentRoundCount < MainConfig.PackManagerID.Count;
    }
}
```

## 2. CapsulePackManager - 扭蛋活动数据管理器

文件位置: `Assets/Scripts/Capsule/Pack/Manager/CapsulePackManager.cs`

```csharp
using System.Collections.Generic;
using asap.core;
using cfg;
using game;
using GameCore;
using OrderManager;
using UnityEngine;

namespace Capsule.Pack.Manager
{
    public class CapsulePackManager : Activity.AEventDataManager<CapsulePackData>, IOrderReplenishment
    {
        #region data

        public override CapsulePackData RefreshData(FishingEvent eventConfig)
        {
            var data = base.RefreshData(eventConfig);
            SetRedPoint(data);
            if (!data.IsActive) return data;
            UITypes.CapsulePackPanel.SetType(data.MainConfig.UIPanel);
            GContext.container.Resolve<IFaceUIService>().AddGiftFaceUI(eventConfig.ID, UITypes.CapsulePackPanel, 0);
            return data;
        }

        public EventToyMachineRound GetRoundConfig(CapsulePackData data)
        {
            var packConfig = data.GetPackConfig();
            var roundConfig = Tables.TbEventToyMachineRound.GetOrDefault(packConfig.DropID);
            if (roundConfig == null)
                Debug.LogError($"配置错误 约定 dropId配置 roundID packConfig.ID {packConfig.ID}");

            return roundConfig;
        }

        public void AddProgress(int index, CapsulePackData data, ref bool isRoundCompleted)
        {
            data.CurrentProgress.Add(index);
            TryAddRoundCount(data, ref isRoundCompleted);
            SaveData(data);
            SetRedPoint(data);
        }

        private void TryAddRoundCount(CapsulePackData data, ref bool isRoundCompleted)
        {
            var roundConfig = GetRoundConfig(data);
            if (data.CurrentProgress.Count != roundConfig.TotalDrawPerRound) return;
            data.CurrentProgress?.Clear();
            data.CurrentRoundCount += 1;
            isRoundCompleted = true;
            data.RefreshCache();
        }

        #endregion data

        #region Red

        private string GetRedPointKey()
        {
            return "eventCapsulePack.red";
        }

        private bool GetRedPointState(CapsulePackData data)
        {
            if (!data.IsActive) return false;
            var packConfig = data.GetPackConfig();
            var iapItem = Tables.TbIAPItemList?.GetOrDefault(packConfig.IAPID);
            return iapItem == null;
        }

        private void SetRedPoint(CapsulePackData data)
        {
            RedPointManager.Instance?.SetRedPointState(GetRedPointKey(), GetRedPointState(data));
        }

        #endregion

        public void OnReplenish(ShopBuyTypeData buyType, IAPItemList iAPItemList, List<ItemData> itemDataGift, RewardType rewardShowType, bool voucher = false)
        {
            if (buyType.extraPurcheData == null) return;
            //判断是不是我的补单 CapsulePackData
            if (!buyType.extraPurcheData.TryGetValue(nameof(CapsulePackData), out var value)) return;
            //判断是不是同一期活动
            if (!buyType.extraPurcheData.TryGetValue("EventID", out var eventIdjToken)) return;
            var data = LoadData();
            var eventID = (int)eventIdjToken;
            if (eventID != data.EventID)
            {
                //todo 保底机制
                return;
            }
            //判断是不是已经加过这个进度，加过的是为重复补单，走保底
            if (!buyType.extraPurcheData.TryGetValue("Progress", out var progressjToken)) return;
            var progress = (int)progressjToken;
            if (progress > data.CurrentProgressCount)
            {
                //todo 保底机制
                return;
            }
            if (!buyType.extraPurcheData.TryGetValue("DropID", out var dropIDjToken)) return;
            var dropID = (int)dropIDjToken;
            var itemDatas = GContext.container.Resolve<PlayerItemData>()
                .GetItemDataByDropIdLureInflation(data.InflationRate, dropID);
            if (itemDatas == null)
            {
                Debug.LogError($"itemDatas is null, dropID:{dropID}");
                return;
            }
            GContext.Publish(new ShowData(itemDatas, RewardType.NormalOne));
            GContext.container.Resolve<PlayerItemData>().AddItem(itemDatas, null);
            GContext.Publish(new ShowData());
            var isRoundCompleted = false;
            AddProgress((int)value, data, ref isRoundCompleted);
        }
    }
}
```

## 3. CapsulePackSentry - 扭蛋活动哨兵

文件位置: `Assets/Scripts/Capsule/Pack/Manager/CapsulePackSentry.cs`

```csharp
using Activity;
using Capsule.Pack.Manager;
using cfg;
using UnityEngine;

namespace Capsule.Pack
{
    [ActivitySentry(9, 10)]
    public class CapsulePackSentry : AActivitySentry<CapsulePackManager, CapsulePackData>
    {
    }
}
```

## 4. IceCapsuleEntranceButton - 扭蛋入口按钮

文件位置: `Assets/Scripts/Capsule/Pack/IceCapsuleEntranceButton.cs`

```csharp
using System;
using System.Collections.Generic;
using Activity.HomeEntranceBtn;
using Capsule.Pack.Manager;
using Capsule.Pack.UIPanel;
using GameCore;
using UnityEngine;

namespace Capsule.Pack
{
    public class IceCapsuleEntranceButton : AbstractHomeEntranceBtn<CapsulePackData>
    {
        protected override Type AssociatedSentryType => typeof(CapsulePackSentry);

        protected override async void OnClickBtn()
        {
            try
            {
                UITypes.CapsulePackPanel.SetType(Data?.MainConfig.UIPanel);
                var obj = await UIManager.Instance.ShowUILoad(UITypes.CapsulePackPanel);
                obj.GetComponent<CapsulePackPanel>()?.SetData(_manager as CapsulePackManager);
            }
            catch (Exception e)
            {
                Debug.LogError(e);
            }
        }

        protected override string GetIconName()
        {
            return Data?.MainConfig?.Icon;
        }

        protected override List<string> GetResourceNames()
        {
            var mainConfig = Data?.MainConfig;
            if (mainConfig == null) return null;
            return new List<string> { mainConfig.UIPanel, mainConfig.Icon, "OfferCapsuleRewardsProbabilityPopupPanel" };
        }
    }
}
```

## 5. 使用示例

```csharp
using Capsule.Pack.Manager;
using Activity;

// 获取扭蛋活动管理器
var manager = ActivityResolver.Resolve<CapsulePackManager>();

if (manager != null)
{
    var data = manager.Data;

    if (data != null && data.IsActive)
    {
        Debug.Log($"扭蛋活动进行中: 当前回合 {data.CurrentRoundCount}，进度 {data.CurrentProgressCount}");

        // 获取回合配置
        var roundConfig = manager.GetRoundConfig(data);
        if (roundConfig != null)
        {
            Debug.Log($"回合奖励: {roundConfig.TotalDrawPerRound}");
        }

        // 增加进度
        bool isRoundCompleted = false;
        manager.AddProgress(index, data, ref isRoundCompleted);

        if (isRoundCompleted)
        {
            Debug.Log("回合完成！");
        }
    }
}
```
