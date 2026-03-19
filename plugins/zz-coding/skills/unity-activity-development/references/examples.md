# Unity Activity Examples

本文档给出一个更贴近当前项目风格的示例，不追求“最理想”的分层，而是强调在现有活动代码中如何低风险演进。

## 示例背景

适用活动类型：

- 有主面板
- 有 TV / Popup 演示
- 数据会先变化，表现后播放
- 面板需要展示倒计时、红点、按钮状态

这和你当前项目中的 `Event1v1BattleSlapManager + Event1v1BattleSlapTvPopupPanel + PresentationData` 非常接近。

## 推荐落地结构

```text
{Activity}/
├── Data/
│   ├── DemoBattleData.cs
│   ├── DemoBattleManager.cs
│   ├── DemoBattlePresentationData.cs
│   └── DemoBattleViewData.cs
└── Panel/
    ├── DemoBattleMainPanel.cs
    └── TVPanel/
        └── DemoBattleTvPopupPanel.cs
```

## 为什么不强推更复杂的 Presenter

在当前项目风格里，`Manager` 已经天然承担：

- 数据入口
- 事件订阅
- 推包与状态修改
- 面板打开协调
- 红点和部分展示逻辑

这时如果立刻再加一个 `Presenter`，收益未必大，反而会：

- 增加类数量
- 增加理解成本
- 让现有活动之间风格不统一

因此更推荐的顺序是：

1. 保留 `Manager` 作为业务入口
2. 在 `Manager` 内增加 `BuildViewData()`
3. 对异步表现增加 `PresentationData`
4. 只有当展示拼装非常复杂时，再抽独立 `Presenter`

## 示例 1：一次“攻击并播放 TV”的推荐流程

### 不推荐的写法

```text
按钮点击
-> Panel 自己算伤害
-> Panel 改 Data
-> Panel 直接播 TV
-> TV 面板再读最新 Data
```

问题：

- UI 直接承载业务
- 真实数据已经变化，TV 可能拿不到旧状态
- 主面板和 TV 面板容易读到不同步的数据

### 推荐写法

```text
按钮点击
-> Manager.ExecuteAttack()
-> 清理并同步 PresentationData
-> 修改真实 Data
-> SaveData()
-> 打开 TV 面板
-> TV 面板优先读 PresentationData
```

## 示例 2：Manager 内部组织方式

推荐按“职责分区”而不是按“调用顺序”堆方法：

```csharp
public class DemoBattleManager : AEventDataManager<DemoBattleData>
{
    private readonly DemoBattlePresentationData _presentationData = new();
    public DemoBattlePresentationData PresentationData => _presentationData;

    #region Init

    public void InitOnActive(DemoBattleData data) { }

    #endregion

    #region Query

    public bool CanAttack(DemoBattleData data) => data != null && data.IsActive && data.IsOnGoing;

    public DemoBattleViewData BuildViewData(DemoBattleData data)
    {
        return new DemoBattleViewData();
    }

    #endregion

    #region Action

    public void ExecuteAttack(DemoBattleData data)
    {
        if (!CanAttack(data)) return;

        _presentationData.Clear();
        _presentationData.SyncFromRealData(data);
        _presentationData.IsPlayingAttack = true;

        data.EnemyHp -= 10;
        SaveData(data);
        ShowBattleTv();
    }

    #endregion

    #region UI

    private void ShowBattleTv() { }
    private void RefreshRedPoint(DemoBattleData data) { }

    #endregion
}
```

### 优点

- 业务入口仍集中在 `Manager`
- 查询类方法和执行类方法容易区分
- UI 协调逻辑没有散落到 `Panel`

## 示例 3：主面板推荐写法

主面板应尽量只做三件事：

1. 绑定事件
2. 请求展示数据
3. 按展示数据刷新 UI

```csharp
private void RefreshAll()
{
    var viewData = _manager.BuildViewData(Data);
    RefreshTop(viewData);
    RefreshReward(viewData);
    RefreshButtons(viewData);
}
```

### 主面板常见反模式

- 在 `RefreshAll()` 里直接写大量业务判断
- 在按钮回调里边改数据边刷新多个控件
- 每个局部刷新自己重新算一遍业务状态

### 更好的方式

- 业务判断统一放 `Manager`
- 主面板尽量只消费 `ViewData`
- 每次刷新走一个统一入口

## 示例 4：TV 面板推荐写法

TV / Popup 面板的核心原则是：

**优先消费表现快照，不直接依赖已变化的真实运行态。**

```csharp
private int GetEnemyHpForDisplay()
{
    if (PresentationData?.IsPresenting == true)
        return PresentationData.EnemyHp;

    return Data.EnemyHp;
}
```

### 为什么这很重要

例如：

- 玩家攻击时，真实 `Data.EnemyHp` 可能已经扣完
- 但 TV 动画还要先显示“旧血量 -> 新血量”的过渡
- 如果 TV 面板直接读真实数据，就会失去动画起点

## 示例 5：什么时候该加 ViewData

当你发现下面任一情况时，建议加 `ViewData`：

- 面板里出现 3 个以上“是否显示 / 是否可点 / 文案拼接”判断
- 同一状态在多个刷新函数中重复计算
- 主面板和奖励面板都依赖同一组展示判断

### 示例

```csharp
public struct DemoBattleViewData
{
    public string ScoreText;
    public string CountdownText;
    public bool ShowRedPoint;
    public bool CanClickAttack;
    public bool ShowFinalStageTag;
}
```

## 示例 6：什么时候不要急着抽 Presenter

下面这些情况下，先别急着抽：

- 活动只维护一个主面板
- 展示逻辑集中但还没复杂到跨多个 UI
- `Manager.BuildViewData()` 已经够用
- 团队当前代码风格基本都是 `Manager + Panel`

这时更推荐：

- 先把 `Manager` 里方法分区
- 先收口刷新和状态判断
- 先增加 `ViewData`

## 最适合当前项目的优化策略

结合现有代码，推荐 skill 里统一强调这条路线：

1. **不改大框架**
2. **让 `Manager` 继续做业务入口**
3. **用 `ViewData` 解决主面板展示复杂度**
4. **用 `PresentationData` 解决异步表现与真实数据错位**
5. **用统一刷新入口降低 Panel 混乱度**

这比直接要求所有活动都改成完全独立的 Presenter/MVVM，更容易落地。

## 产出时建议 Agent 优先回答的内容

当 Agent 使用该 skill 回答“怎么开发/怎么重构活动”时，优先输出：

1. 当前活动应保留哪些既有结构
2. 哪些逻辑留在 `Manager`
3. 哪些 UI 判断应下沉成 `ViewData`
4. 是否需要 `PresentationData`
5. 最小改动顺序是什么
