---
name: data-presentation-separation
description: 当用户讨论活动TV动画展示、需要延迟展示数据、表现层与数据层分离、设计模式、活动表现数据、TV面板数据同步等话题时使用此skill。提供数据与表现分离的解决方案和代码模板。
version: 1.0.0
trigger_phrases:
  - "数据与表现分离"
  - "TV动画展示"
  - "活动表现数据"
  - "活动设计模式"
  - "TV面板数据延迟"
  - "表现层与数据层分离"
---

# 数据与表现分离设计模式

## 适用场景

当活动有TV动画/表现展示需求，且存在以下问题时使用：

1. **时序问题**：表现展示时数据已被更新，导致状态不一致
2. **数据污染**：表现层直接修改真实数据
3. **展示延迟**：需要先更新数据再做展示

## 常见问题

```
当前逻辑：触发攻击 → 扣NPC血 → NPC死亡 → 立即更新Data → TV动画检查IsOnGoing时已是false
```

## 解决方案

**核心思路**：创建单独的表现数据类，界面根据状态选择用真实数据或表现数据，表现完成后切换回真实数据。

```
┌─────────────┐    攻击触发     ┌─────────────┐
│   Manager   │ ──────────────> │ 表现数据类   │
│  (真实数据)  │  记录攻击信息   │(PresentationData)│
└─────────────┘                 └─────────────┘
       │                              │
       │ 更新Data                      │ TV展示用
       ▼                              ▼
┌─────────────┐                 ┌─────────────┐
│   真实Data   │<────────────────│  TV面板     │
│  (已更新)    │   表现完成       │ (展示表现)   │
└─────────────┘   清理           └─────────────┘
```

## 实现步骤

### 步骤 1: 新建表现数据类

**文件位置**: `Assets/Scripts/{活动目录}/Data/{活动名}PresentationData.cs`

```csharp
using System;

namespace game
{
    /// <summary>
    /// 活动表现数据
    /// 用于TV动画展示，与真实数据分离
    /// </summary>
    [Serializable]
    public class {活动名}PresentationData
    {
        /// <summary>
        /// 当前关卡ID（表现用）
        /// </summary>
        public int CurrentStageID;

        /// <summary>
        /// NPC名称（表现用）
        /// </summary>
        public string NpcName;

        /// <summary>
        /// NPC头像（表现用）
        /// </summary>
        public string NpcIcon;

        /// <summary>
        /// NPC当前HP（表现用）
        /// </summary>
        public int NpcHp;

        /// <summary>
        /// NPC最大HP（表现用）
        /// </summary>
        public int NpcMaxHp;

        /// <summary>
        /// 玩家HP（表现用）
        /// </summary>
        public int PlayerHp;

        /// <summary>
        /// 攻击来源（0:轰炸 1:射鱼 2:钓鱼）
        /// </summary>
        public int AttackSource;

        /// <summary>
        /// 本次攻击伤害
        /// </summary>
        public int AttackDamage;

        /// <summary>
        /// 是否击杀了NPC
        /// </summary>
        public bool KilledNpc;

        /// <summary>
        /// 是否触发了进入下一关
        /// </summary>
        public bool StageCompleted;

        /// <summary>
        /// 下一关ID（StageCompleted为true时有效）
        /// </summary>
        public int NextStageId;

        /// <summary>
        /// 是否正在进行表现
        /// </summary>
        public bool IsPresenting = false;

        /// <summary>
        /// 清理数据
        /// </summary>
        public void Clear()
        {
            AttackSource = -1;
            AttackDamage = 0;
            KilledNpc = false;
            StageCompleted = false;
            NextStageId = 0;
            IsPresenting = false;
        }

        /// <summary>
        /// 从真实数据同步基本信息
        /// </summary>
        public void SyncFromRealData({活动名}Data realData)
        {
            CurrentStageID = realData.CurrentStageID;
            NpcName = realData.NpcName;
            NpcIcon = realData.NpcIcon;
            NpcHp = realData.NpcHp;
            NpcMaxHp = realData.NpcMaxHp;
            PlayerHp = realData.PlayerHp;
        }
    }
}
```

### 步骤 2: 在 Manager 中添加表现数据管理

**文件**: `Assets/Scripts/{活动目录}/Data/{活动名}Manager.cs`

添加字段：

```csharp
// 表现数据
private {活动名}PresentationData _presentationData = new();

/// <summary>
/// 获取表现数据
/// </summary>
public {活动名}PresentationData PresentationData => _presentationData;

/// <summary>
/// 是否有正在进行的表現
/// </summary>
public bool IsPresenting => _presentationData.IsPresenting;
```

修改执行攻击的方法（如 `ExecutePlayerAttack`）：

```csharp
public int ExecutePlayerAttack(int multiplier, int sourceIndex)
{
    var damage = CalculateDamageBySource(multiplier, sourceIndex);
    // buff 效果
    damage *= (int)(1 + GetBuffBenefit());

    // 记录表现数据（在更新真实Data之前）
    _presentationData.Clear();
    _presentationData.IsPresenting = true;
    _presentationData.AttackSource = sourceIndex;
    _presentationData.AttackDamage = damage;
    _presentationData.SyncFromRealData(Data);

    Data.NpcHp -= damage;

    if (Data.NpcHp <= 0)
    {
        _presentationData.KilledNpc = true;
        _presentationData.NextStageId = Data.IsFinalStage ? Data.InitConfig.InitialStage : Data.StageConfig.NextStage;
        OnStageHPDepleted();
    }

    SaveData();
    ShowTvFace();
    return damage;
}
```

添加表现完成方法：

```csharp
/// <summary>
/// 表现完成，清理表现数据
/// </summary>
public void FinishPresentation()
{
    _presentationData.Clear();
}
```

### 步骤 3: 修改界面中的检查和展示逻辑

**文件**: `HomePanel.cs` 或对应的界面文件

修改触发方法：

```csharp
private async void SlapBattleAdd(int id, int addCount, Vector3 startPos)
{
    try
    {
        var manager = GContext.container.Resolve<{活动名}Manager>();

        // 检查是否有正在进行的表現，如果有就显示TV面板
        if (manager.IsPresenting)
            await manager.ShowTvPanelOnFishing();

        await GetRewardFly(id, startPos, addCount);
        btn_mask.gameObject.SetActive(false);
    }
    catch (Exception e)
    {
        Debug.LogError(e);
    }
}
```

### 步骤 4: TV面板使用表现数据

**文件**: `{活动名}TvPopupPanel.cs`

修改显示逻辑，优先使用表现数据：

```csharp
var manager = ...;
var presentationData = manager.PresentationData;

if (presentationData.IsPresenting)
{
    // 使用表现数据展示动画
    ShowNpcWithAnimation(presentationData.NpcIcon, presentationData.NpcHp, presentationData.NpcMaxHp);
    ShowDamage(presentationData.AttackDamage);
    if (presentationData.KilledNpc)
        PlayKillAnimation();
}
else
{
    // 使用真实数据展示
    ShowNpc(manager.Data.NpcIcon, manager.Data.NpcHp, manager.Data.NpcMaxHp);
}

// 表现完成后清理
manager.FinishPresentation();
```

## 核心流程

| 步骤 | 操作 | 说明 |
|------|------|------|
| 1 | 触发攻击 | 钓鱼/轰炸/射鱼 |
| 2 | 计算伤害 | 先算伤害值 |
| 3 | 填充表现数据 | 记录攻击来源、伤害、击杀状态等 |
| 4 | 更新真实Data | HP减少，判断死亡，进入下一关，SaveData |
| 5 | 界面检查表现状态 | `IsPresenting = true` 则显示TV动画 |
| 6 | TV动画使用表现数据 | 展示击杀、NPC等信息 |
| 7 | 表现完成 | 调用 `FinishPresentation()` 清理表现数据 |

## 界面使用数据的选择

| 场景 | 使用的数据源 |
|------|-------------|
| 正常游戏界面 | `manager.Data` (真实数据) |
| TV动画展示 | `manager.PresentationData` (表现数据) |
| 表现完成后的TV关闭动画 | `manager.Data` (已更新的真实数据) |

## 优势

1. **更清晰** - 表现数据独立，职责分明
2. **更安全** - TV展示用独立数据，不会意外污染真实Data
3. **更好维护** - 后续加字段只需在表现数据类添加
4. **符合游戏开发常见模式** - 表现层与数据层分离是常规做法
