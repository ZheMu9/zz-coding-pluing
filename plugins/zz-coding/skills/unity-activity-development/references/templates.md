# Unity Activity Templates

本文档提供更贴近现有 Unity 活动项目的参考模板。目标不是强推最理想的架构，而是给出一套能在现有 `AEventDataManager + Panel` 体系内渐进优化的写法。

## 推荐分层

优先采用下面的轻量分层：

| 层 | 职责 | 是否必须 |
| ---- | ---- | -------- |
| `Data` | 活动运行态、配置访问缓存 | 必须 |
| `Manager` | 业务入口、推包处理、状态修改、UI 入口协调 | 必须 |
| `PresentationData` | 异步表现快照，隔离真实数据变化 | 按需 |
| `ViewData` | 面板展示所需的只读数据 | 推荐 |
| `Panel` | 纯展示、按钮响应、动画编排 | 必须 |

## 什么时候不要过度设计

- 如果活动只是普通领奖、倒计时、红点刷新，不必强行加 `Presenter`
- 如果 UI 推导逻辑不多，可以先只加 `ViewData`
- 如果存在 TV、结算回放、异步动画，优先加 `PresentationData`
- 如果 `Manager` 已承担较多协调职责，先做“方法收口”再谈拆类

## 1. Data 模板

文件位置建议：`Assets/Scripts/{Activity}/Data/{Activity}Data.cs`

```csharp
using System;
using Activity;
using cfg;
using Newtonsoft.Json;

namespace DemoActivity.Data
{
    [Serializable]
    public class DemoActivityData : AEventData
    {
        public int Score;
        public int RemainCount;
        public int CurrentStageId;
        public long EndTimestamp;

        [JsonIgnore]
        public DemoMainConfig MainConfig => Tables.TbDemoMain.GetOrDefault(FishingEvent?.RedirectID ?? 0);

        [JsonIgnore]
        public bool HasRemainCount => RemainCount > 0;

        [JsonIgnore]
        public bool CanOpenPanel => IsActive && IsOnGoing;

        public override void RefreshCache()
        {
            base.RefreshCache();
            // 清理需要按次构建的缓存
        }
    }
}
```

### Data 建议

- 只放运行态和轻量级派生属性
- 配置访问可以放在 `JsonIgnore` 属性里
- 不要把大量流程控制写进 `Data`
- 不要让 `Panel` 直接修改 `Data`

## 2. Manager 模板

文件位置建议：`Assets/Scripts/{Activity}/Data/{Activity}Manager.cs`

```csharp
using Activity;
using cfg;
using GameCore;
using UniRx;

namespace DemoActivity.Data
{
    public class DemoActivityManager : AEventDataManager<DemoActivityData>
    {
        private readonly DemoActivityPresentationData _presentationData = new();

        public DemoActivityPresentationData PresentationData => _presentationData;

        public override DemoActivityData RefreshData(FishingEvent eventConfig)
        {
            var data = base.RefreshData(eventConfig);
            RefreshRedPoint(data);
            return data;
        }

        protected override void OnDataInitialized(IEventData data, FishingEvent eventConfig)
        {
            base.OnDataInitialized(data, eventConfig);

            var demoData = (DemoActivityData)data;
            InitOnActive(demoData);
            SaveData(demoData);
        }

        public void InitOnActive(DemoActivityData data)
        {
            data.RemainCount = 3;
            data.CurrentStageId = data.MainConfig.InitialStageId;
        }

        public bool CanPlay(DemoActivityData data)
        {
            return data != null && data.CanOpenPanel && data.HasRemainCount;
        }

        public void ExecutePlay(DemoActivityData data)
        {
            if (!CanPlay(data)) return;

            _presentationData.Clear();
            _presentationData.SyncFromRealData(data);
            _presentationData.IsPlaying = true;

            data.RemainCount -= 1;
            data.Score += 10;

            SaveData(data);
            ShowTvPanel();
        }

        public DemoActivityViewData BuildViewData(DemoActivityData data)
        {
            return new DemoActivityViewData
            {
                ScoreText = data?.Score.ToString() ?? "0",
                RemainCountText = data?.RemainCount.ToString() ?? "0",
                ShowRedPoint = data != null && data.HasRemainCount,
                CanClickPlay = CanPlay(data),
                CountdownText = FormatCountdown(data?.EndTimestamp ?? 0)
            };
        }

        private void RefreshRedPoint(DemoActivityData data)
        {
            // 统一处理红点刷新
        }

        private string FormatCountdown(long endTimestamp)
        {
            // 统一处理倒计时文案
            return string.Empty;
        }

        private void ShowTvPanel()
        {
            // 统一处理表现层入口
        }
    }
}
```

### Manager 建议

- `Manager` 依旧是当前项目最自然的业务入口
- 但内部要按职责分组：初始化、业务执行、展示数据构建、UI 协调
- 把“是否可操作”这类判断下沉到 `Manager`
- 面板不要直接拼装复杂业务规则

## 3. PresentationData 模板

适用于以下场景：

- 先改真实数据，再播动画
- 关闭面板后还有异步表现
- HP、奖励、阶段切换需要按旧状态播放过渡

```csharp
using System;

namespace DemoActivity.Data
{
    [Serializable]
    public class DemoActivityPresentationData
    {
        public int Score;
        public int RemainCount;
        public int CurrentStageId;

        public bool IsPlaying;
        public bool IsVictory;
        public bool ShouldAdvanceStage;
        public int NextStageId;

        public bool IsPresenting => IsPlaying;

        public void SyncFromRealData(DemoActivityData realData)
        {
            Score = realData.Score;
            RemainCount = realData.RemainCount;
            CurrentStageId = realData.CurrentStageId;
        }

        public void Clear()
        {
            Score = 0;
            RemainCount = 0;
            CurrentStageId = 0;
            IsPlaying = false;
            IsVictory = false;
            ShouldAdvanceStage = false;
            NextStageId = 0;
        }
    }
}
```

### PresentationData 建议

- 只保存表现需要的字段
- 不要把真实业务方法写到这里
- `SyncFromRealData()` 负责快照
- `Clear()` 保证面板关闭后状态不会串场

## 4. ViewData 模板

如果面板里出现多段 if/else 拼文案、红点、按钮可点状态，就应该引入 `ViewData`。

```csharp
namespace DemoActivity.Data
{
    public struct DemoActivityViewData
    {
        public string ScoreText;
        public string RemainCountText;
        public string CountdownText;
        public bool ShowRedPoint;
        public bool CanClickPlay;
    }
}
```

### ViewData 建议

- 面板消费 `ViewData`，不要直接拼业务结果
- 尽量在 `Manager.BuildViewData()` 中一次构造完成
- `ViewData` 适合 UI 文案、显隐、按钮可点等展示结果

## 5. Panel 模板

文件位置建议：`Assets/Scripts/{Activity}/Panel/{Activity}Panel.cs`

```csharp
using Activity;
using UnityEngine;
using UnityEngine.UI;

namespace DemoActivity.Panel
{
    public class DemoActivityPanel : MonoBehaviour
    {
        [SerializeField] private Button playButton;
        [SerializeField] private GameObject redPoint;

        private DemoActivityManager _manager;
        private DemoActivityData Data => _manager?.LoadData();

        private void Start()
        {
            _manager = ActivityResolver.Resolve<DemoActivityManager>();
            BindEvents();
            RefreshAll();
        }

        private void BindEvents()
        {
            playButton.onClick.AddListener(OnClickPlay);
        }

        private void OnClickPlay()
        {
            _manager?.ExecutePlay(Data);
            RefreshAll();
        }

        private void RefreshAll()
        {
            var viewData = _manager.BuildViewData(Data);
            RefreshButtons(viewData);
            RefreshTopInfo(viewData);
        }

        private void RefreshButtons(DemoActivityViewData viewData)
        {
            playButton.interactable = viewData.CanClickPlay;
            redPoint.SetActive(viewData.ShowRedPoint);
        }

        private void RefreshTopInfo(DemoActivityViewData viewData)
        {
            // 刷新文本和其他静态展示
        }
    }
}
```

### Panel 建议

- `Panel` 的核心职责是绑定、刷新、播放 UI 动画
- `RefreshAll()` 作为统一入口
- 把刷新拆成 `RefreshTopInfo()`、`RefreshRewardList()`、`RefreshButtons()` 这种可维护的小方法
- 如果是 TV/Popup 面板，优先读 `PresentationData`

## 6. TV / Popup 面板建议模板

```csharp
private DemoActivityPresentationData PresentationData => _manager?.PresentationData;

private void RefreshHpOrProgress()
{
    var usePresentation = PresentationData?.IsPresenting == true;
    var score = usePresentation ? PresentationData.Score : Data.Score;
    var remainCount = usePresentation ? PresentationData.RemainCount : Data.RemainCount;
}
```

### TV 面板建议

- 优先读 `PresentationData`
- 真实数据只作为兜底
- 面板关闭时记得清理表现状态

## 7. 渐进式优化顺序

如果现有活动代码已经比较重，建议按下面顺序改：

1. 先收口刷新方法
2. 再抽 `BuildViewData()`
3. 再引入 `PresentationData`
4. 最后再考虑是否拆 `Presenter`

这能避免“一上来重构太大，改动风险高”。
