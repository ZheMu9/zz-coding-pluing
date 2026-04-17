---
name: ft-unity-build
description: >-
  Documents the FT Client Unity build pipeline (BuildTool CI/batchmode, Editor menus,
  Addressables/state.bin, cfgPack, RuStore flags) and post-processors (tysdk, iOS alternate icons).
  Use when the user asks about 打包、CI 构建、BuildTool、APK/AAB/IPA、热更资源、Addressables 构建、
  Rustore/RuStore、buildApk、发版构建、或修改 iOS/Android 构建后处理。
---

# FT Client Unity 构建（BuildTool）

## 必读源码位置

- **主逻辑**：`Assets/Editor/BuildTool.cs`（`CIBuildAndroid`、`CIBuildIOS`、`DoBuild`、`BuildHotUpdate`、菜单项）
- **发版清单**：仓库根 `docs/发版同步清单.md`（RuStore、`cfgPack`、`state.bin`、AppLovin）
- **Android CI 封装**：仓库根 `buildscripts/buildApk.py`（`-executeMethod BuildTool.CIBuildAndroid`）
- **iOS PostProcess**：`LocalPackages/tysdk/Editor/IOSBuildProcessor.cs`（order 1000）；`Assets/Editor/IOSBuildProcessor.cs`（order 1001，备选图标）

## CI 入口与参数

- **Android**：`BuildTool.CIBuildAndroid` — 解析 `--out`、`--define-symbol`、`--bundle-type`（`None|FirstPack|ALL`）、`--build-type`（`New|Hotupdate`）、`--split`、`--aab`、`--rustore`（切换 Addressables profile `arksgameru` 并执行 `DoRuSpecial`）
- **iOS**：`BuildTool.CIBuildIOS` — 同上除 RuStore 外大部分参数；`Hotupdate` 分支仍检查 `aab`（与 Android 对称）
- **仅更 Addressables**：`BuildTool.CIUpdateAddressable`（`RemoveAssertbundles` + `UpdateAddressable`）

`UpdateAddressable` **依赖** Content Update 的 `state.bin`；缺失则抛异常。

## `DoBuild` 要点（不打散用户问题时默认知晓）

临时宏 → SyncGroups → UpdateAddressable → 按 `bundleType` 拷贝 Addressables 到 StreamingAssets → `WriteAssetsFile` + `CopyCfgData` → `BuildPipeline.BuildPlayer` → 清理宏与 StreamingAssets Addressables 拷贝、删除构建版本文件。

## 易混文件

- `Assets/Scripts/Stages/BuildAct.cs`：**游戏内**营地建造玩法，**不是**打包脚本。
- `Assets/Editor/BuildHandler.cs`：**已整体注释**，勿作当前入口。

## 详细说明

完整参数表、菜单列表、脚本链与排查项见个人库文档：

`untiy_coding_library/llm-wiki/wiki/projects/ft-client-build-pipeline.md`

修改 `DoBuild` / CI 参数时，同步检查 `buildscripts/buildApk.py` 与实际 Jenkins/流水线是否一致。
