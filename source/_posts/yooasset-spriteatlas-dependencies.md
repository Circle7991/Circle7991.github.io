---
title: YooAsset 图集白块排查：理解 TrackSpriteAtlasDependencies
date: 2026-10-15 18:00:00
updated: 2026-10-15 18:00:00
categories:
  - Unity打包
tags:
  - Unity
  - YooAsset
  - SpriteAtlas
  - AssetBundle
excerpt: 从 Sprite 引用与图集纹理的关系出发，理解 TrackSpriteAtlasDependencies 的作用、生效条件与常见排查方法。
---

使用 YooAsset 加载 UI 预制体时，可能遇到文字正常、图片变成白块或透明块的情况。即使预制体引用了 Sprite、SpriteAtlas 也已打包，图片仍可能因为图集依赖缺失而无法显示。

本文以 YooAsset 2.3.18 的 Scriptable Build Pipeline（SBP）为例，说明这一问题的原因、构建配置和排查方法。

## 1. 为什么引用了 Sprite，仍然需要加载图集

UI Image 的 `Source Image` 通常引用源图片中的 Sprite 对象。图片加入 SpriteAtlas 后，构建过程会将 Sprite 的渲染数据关联到图集纹理。

```text
UI 预制体
  └─ Sprite 对象
       └─ 渲染所需的图集纹理
            └─ 图集所在的 AssetBundle
```

**Sprite 引用完整，不代表运行时已经加载了图集纹理。**

YooAsset 根据资源清单加载依赖。如果清单没有记录预制体所需的图集 Bundle，加载预制体时就可能只拿到了 Sprite，而缺少其渲染所需的纹理。

因此，需要分别确认三个环节：

1. 图集是否参与构建，并生成了 Bundle。
2. 资源清单是否记录了相应的图集依赖。
3. 运行时是否成功加载并保持了这些资源的有效引用。

白块也可能来自资源加载失败、材质设置或过早卸载等问题，不能仅凭外观认定是图集依赖缺失。

## 2. TrackSpriteAtlasDependencies 的作用

`TrackSpriteAtlasDependencies` 是 SBP 构建参数，用于在生成 YooAsset 清单时补充图集依赖。

该参数在 YooAsset **2.3.15** 中加入；本文讨论的 **2.3.18 默认值为 false**。

开启后，清单生成流程会根据 SBP 的 Bundle 引用关系，将图集 Bundle 补充到相关主资源的依赖列表中，使运行时能够沿着依赖链加载图集。

```text
加载预制体
→ 查询资源清单
→ 加载预制体及其依赖 Bundle
→ 获取 Sprite 和图集纹理
→ 正常渲染
```

**这是构建阶段的设置。修改参数后必须重新构建资源，并让设备使用新清单和新资源包。**

它不会让所有图集常驻内存，也不会代替资源引用与释放管理。

## 3. 在哪里开启

在实际使用的构建入口中，创建并填写 `ScriptableBuildParameters` 后、调用构建管线的 `Run()` 前，设置：

```csharp
// 为相关资源补充图集 Bundle 依赖。
buildParameters.TrackSpriteAtlasDependencies = true;
```

其中 `buildParameters` 是传给构建管线的参数对象，变量名可按自己的构建脚本调整。

如果同时通过编辑器构建窗口和 CI 脚本打包，应分别检查两个入口。它们可能各自创建参数对象，一个入口开启后，另一个入口不会自动继承。

对于预制体静态引用 Sprite、希望加载预制体时自动带入图集的方案，可以开启该参数；已有显式图集管理或后期绑定方案时，应结合其加载策略判断。

## 4. 生效条件与打包粒度

在本文讨论的 2.3.18 实现中，需要满足：

- 使用 `ScriptableBuildPipeline`。
- Sprite Packer 模式不为 `Disabled`。
- SpriteAtlas 被当前 Package 收集并参与构建。

采用默认图集绑定方案时，还应检查 SpriteAtlas 的 `Include In Build`。如果主动关闭它，应配套设计 `SpriteAtlasManager.atlasRequested` 后期绑定流程。

依赖补充基于 Bundle 引用关系，因此打包粒度会影响加载范围：

- 多个预制体合入一个 Bundle，可能连带获得较多图集依赖。
- 多个图集合入一个 Bundle，加载其中一个图集时，需要加载其所在 Bundle。
- 需要更细粒度控制时，可调整预制体和图集的打包规则，再检查构建报告与实际内存。

单独拆分预制体，不代表图集 Bundle 也已经拆分。

## 5. 手动预加载图集能否解决

如果原因确实是图集未及时加载，可以在加载预制体之前，先加载并持有对应 SpriteAtlas：

```text
加载图集并等待完成
→ 加载预制体
→ 实例化并显示
→ 按资源生命周期释放相应引用
```

在预制体实例化后才加载图集，与上述顺序并不等价；图集绑定时机和 UI 初始化流程都可能影响结果。

如果提前加载后恢复显示，说明图集的可用时机值得重点排查。接着仍应检查清单依赖、加载结果及释放时机。

其他 UI 在未开启该参数时仍能正常显示，可能是启动资源直接引用了图集，或已有预加载、显式图集加载等流程。不能仅凭另一个 UI 正常显示，就推断当前预制体的依赖完整。

## 6. 自动加载后再手动加载，会重复占用内存吗

同一个 Package 中的同一份 Bundle，通常会复用已有的 Bundle 加载器。自动依赖加载后，再手动加载同一个图集，不等于再复制一整份图集纹理。

但仍需注意：

- 手动加载可能增加 Handle、Provider 和引用管理等开销，获取的引用应配对释放。
- 相同资源如果重复打入不同 Bundle 或不同 Package，可能产生重复资源内存。
- 图集纹理按纹理页加载，不是只加载当前图标对应的像素区域。

## 7. 图集白块与纹理零冗余

Sprite 对象与 Texture2D 像素数据应分开理解。源 Sprite 所在 Bundle 仍包含 Sprite 对象，并不能直接说明存在纹理冗余。

SBP 包含剔除未使用源 Sprite 纹理数据的流程。源纹理被剔除后，如果图集纹理没有被正确加载，Sprite 就可能表现为白块或透明块。

排查重点是补齐加载依赖。若需要验证零冗余，应进一步检查构建产物中的 Texture2D 对象、尺寸与像素数据；仅看到 Texture2D 类型不足以判断，因为图集纹理本身也是 Texture2D。

## 8. 推荐排查顺序

1. 确认目标平台的图集已经构建，Bundle 已进入实际发布内容。
2. 确认真正执行的构建入口开启了 `TrackSpriteAtlasDependencies`，并满足生效条件。
3. 在新清单中按路径定位预制体与图集，检查图集 Bundle 是否进入预制体资源的依赖列表。仅在 Bundle 层面看到关联还不够。
4. 确认设备使用的是新版本清单和资源包，必要时清理旧构建缓存后完整重构建。
5. 使用真实 Bundle 验证首次打开、关闭重开及卸载重载，编辑器模拟模式不能替代设备验证。
6. 验证自动依赖时，避免额外的提前预加载掩盖结果，同时保持正在使用的资源引用有效。
7. 依赖完整但仍白块时，继续检查加载错误、图集绑定、`Image.sprite.texture`、材质与过早释放问题。

## 参考资料

- [YooAsset 2.3.15 发布记录](https://github.com/tuyoogame/YooAsset/releases/tag/2.3.15)
- [YooAsset：图集白块问题](https://www.yooasset.com/docs/solution/SpriteAtlasWhiteBlock)
- [YooAsset：图集打包零冗余](https://www.yooasset.com/docs/solution/SpriteAtlasZeroRedundancy)

本文参数说明以 YooAsset 2.3.18 为范围。在线文档可能随版本更新，使用其他版本时应核对相应源码和构建结果。
