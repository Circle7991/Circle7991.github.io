---
title: 用 PlayableGraph 拆解大型动画状态机：按需加载与距离分级更新
date: 2026-10-17 23:30:00
updated: 2026-10-17 23:30:00
categories:
  - Unity动画
tags:
  - Unity
  - PlayableGraph
  - Animator
  - 动画系统
excerpt: 从拆解大型状态机、按需装配 Controller 出发，整理运行时动画图、已实现的求值频率控制，以及按角色距离分配动画更新频率的扩展思路。
---

最初引入 PlayableGraph 的目的，是拆解不断膨胀的动画状态机：角色在当前玩法里只需要一部分动画，却可能因为一个总 Controller 引用了所有动作而带入过大的资源依赖。

希望达到的目标是：**把复杂状态机按动作域拆成较小的 Controller，运行时只加载当前需要的控制器，再通过动画图统一装配、切换和输出。** 手动更新和动画降频，是在这个运行时控制基础上进一步扩展的能力。

本文以 Unity **6.3** 的 Playables API 为背景，讨论状态机拆分、控制器按需装配、参数归属和距离分级更新。文中的代码片段表达关键算法与接口关系，资源管理、异步请求和生命周期需要按各自工程补齐。

## 1. 为什么要拆解大型状态机

当跑步、骑行、交互、展示、特殊动作等逻辑长期堆入一个 Animator Controller 时，状态、过渡和片段引用会持续增加。角色当前只用其中一小部分，控制器却仍与大量不相关内容建立资源引用。

状态没有激活，并不等于其资源依赖不会随加载流程被带入。具体加载范围还取决于资源系统、Bundle 依赖和引用关系；仅把状态移动到 Sub-State Machine，仍然属于同一个 Controller 资产，也没有建立独立的加载边界。

真正想调整的是装配单位：从“一个 Controller 包含所有动作”，转为“按需要选择一份较小的动作控制器”。以下是拆分思路示例，并不表示项目已实现所有模块：

```text
原有结构
角色 → 总 Controller → 跑步 / 骑行 / 交互 / 展示 / 特殊动作

目标结构
当前动作需求 → 加载对应的小 Controller
                    │
                    ▼
               运行时动画图 → Animator → 骨骼
```

拆分边界可以围绕玩法、运动类型或动作组确定。每个小 Controller 只负责本组的状态、过渡与必要片段；跨动作域的选择和切换由运行时代码安排。一次性特殊动作也可以直接加载 Clip，减少把所有例外动作塞回基础状态机的压力。

如果仍把完整的大 Controller 交给 AnimatorControllerPlayable，只是改变了它的运行载体，没有完成状态机资产拆分。

## 2. 按需加载 Controller，OverrideController 又负责什么

运行时装配的目标流程是：

```text
业务决定当前动作组
    ↓
查询动作配置，得到对应 Controller 的资源地址
    ↓
异步加载 Controller / OverrideController
    ↓
在 PlayableGraph 中创建或替换 AnimatorControllerPlayable
    ↓
恢复新控制器需要的参数，继续输出到同一个 Animator
    ↓
旧节点不再使用，按资源系统规则释放旧控制器的加载引用
```

需要区分两件事：

| 手段 | 作用 |
| --- | --- |
| 将总 Controller 拆成独立的小 Controller | 划分状态结构和资源加载边界 |
| AnimatorOverrideController | 复用基础状态机结构，并替换其中的动画片段 |

OverrideController 保留基础 Controller 的结构、参数和状态机逻辑。它适合给一个较小的基础模板提供不同角色或动作风格的片段变体，**不会自动把一个庞大的基础状态机裁剪成小状态机**。[Unity：Animator Override Controller](https://docs.unity3d.com/6000.3/Documentation/Manual/AnimatorOverrideController.html)。

按需加载要落到资源层面，还需要让加载依赖与使用范围保持一致：

- 角色预制体、全局配置或常驻对象不应意外强引用全部动作控制器。
- 配置中的资源地址与直接持有资源对象引用，需要区分管理。
- 不同控制器如果被合入同一个大 Bundle，依赖加载范围仍可能大于预期。
- 销毁 Playable 节点只是解除图中的使用，不能代替资源系统 Handle 的配对释放。
- 缓存中的 Controller、Clip 和加载引用，也会影响旧资源能否释放。

因此，这套结构提供了缩小按需加载范围的条件；实际内存是否减少，还取决于控制器拆分、构建依赖和资源生命周期，不能仅由“改成 PlayableGraph”得出结论。

## 3. 把“加载”和“播放装配”分开

资源层负责“拿到哪份控制器”，动画图负责“如何使用控制器输出姿态”。两者通过 RuntimeAnimatorController 资源引用衔接：

```text
业务动作需求
    ↓
控制器资源地址 / 资源标识
    ↓
资源层异步加载，返回 Controller 及其持有凭据
    ↓
动画图替换 Controller 节点
    ↓
业务恢复参数，旧资源不再使用后配对释放
```

当一个小型控制器加载完成，可以断开旧节点、销毁旧节点，再创建并连接新节点。Mixer 和输出端继续使用：

```csharp
/// <summary>替换图中的动作控制器，保留 Mixer 与输出。</summary>
internal void SetController(RuntimeAnimatorController controller)
{
    if (controller == null || !_graph.IsValid()) return;

    if (_controllerPlayable.IsValid())
    {
        _graph.Disconnect(_mixer, 0);
        _graph.DestroyPlayable(_controllerPlayable);
    }

    _controllerPlayable = AnimatorControllerPlayable.Create(_graph, controller);
    _graph.Connect(_controllerPlayable, 0, _mixer, 0);
}
```

上面的示意是直接替换，并不提供两个 Controller 之间的交叉淡化。如果需要保留旧姿态并平滑过渡，可以临时为新旧控制器分配两个输入，在过渡完成后释放旧节点和资源；这会增加过渡期内的资源持有和求值工作。

异步加载还需要一个明确的“谁有权应用结果”的约定。例如动作请求 A 尚未完成时又发出 B，A 的较晚返回不应覆盖更新的 B。常见处理方式是请求序号校验或取消旧请求，销毁角色时也应取消未完成请求，并释放已获得但不再使用的资源持有凭据。

这使控制器拆分不仅是资产组织问题，也成为一条可管理的生命周期：**请求 → 加载 → 应用 → 使用 → 替换 → 释放**。

## 4. 先明确各层负责什么

角色动画可以划分为三层职责：

| 层次 | 职责 | 本次实现中的对应内容 |
| --- | --- | --- |
| 业务层 | 决定角色的运动意图 | 运动速度、停止、动作请求 |
| 动画控制层 | 设置参数、安排求值、控制混合 | 角色动画组件与动画图封装 |
| 输出层 | 将动画图的结果应用到角色 | Animator 与骨骼 |

业务层提供速度，动画控制层把它转换为状态机参数；角色世界坐标的移动由移动组件负责。这样，动画姿态更新频率与角色整体移动频率可以分别控制。

```text
运动数据 Speed
    │
    ▼
业务层平滑得到 AnimationSpeed
    │
    ▼
图内状态机的 Speed 参数
    │
    ▼
PlayableGraph 求值 → Animator → 骨骼姿态

角色根节点位移：由移动组件单独更新
```

因此，降低动画求值频率后，角色整体依然可以连续前进。判断动画是否降频，应观察手臂、腿部等骨骼姿态，而不是角色是否仍向前移动。

## 5. 图结构：基础运动与特殊动作两路混合

一种实用的组合方式是保留当前动作域的基础状态机，并额外提供一路特殊动作输入：

```text
AnimatorControllerPlayable ── 输入 0 ──┐
基础运动：Speed 驱动 Blend Tree         │
                                      ├─ AnimationMixerPlayable
AnimationClipPlayable ─────── 输入 1 ──┘            │
特殊动作：代码请求播放                              ▼
                                      AnimationPlayableOutput
                                                  │
                                                  ▼
                                               Animator
                                                  │
                                                  ▼
                                                骨骼
```

Mixer 始终保留两个输入端口。基础运动默认权重为 1，特殊动作默认权重为 0；特殊动作请求到来时才创建对应的 Clip 节点。

这里的“图结构稳定”指两路输入和输出的骨架稳定。切换特殊片段时仍会销毁、重建 Clip 节点，替换 Controller 时也会替换 Controller 节点，不能理解为所有节点终身不变。

建图的关键部分如下：

```csharp
// 创建由代码手动求值的动画图。
_graph = PlayableGraph.Create("CharacterAnimation");
_graph.SetTimeUpdateMode(DirectorUpdateMode.Manual);

// 用独立的运行时节点承载基础运动状态机。
_controllerPlayable = AnimatorControllerPlayable.Create(_graph, controller);

// 固定两个输入端口，第二路留给特殊动作。
_mixer = AnimationMixerPlayable.Create(_graph, 2);
_graph.Connect(_controllerPlayable, 0, _mixer, 0);
_mixer.SetInputWeight(0, 1f);
_mixer.SetInputWeight(1, 0f);

// Animator 负责接收动画图的输出。
_output = AnimationPlayableOutput.Create(_graph, "Animation", animator);
_output.SetSourcePlayable(_mixer);
_graph.Play();
```

Manual 模式下，需要代码显式调用 `PlayableGraph.Evaluate(deltaTime)`。`Play()` 将图置于播放状态，并不把 Manual 模式改回 GameTime。[Unity：Manual 更新模式](https://docs.unity3d.com/6000.3/Documentation/ScriptReference/Playables.DirectorUpdateMode.Manual.html)。

## 6. 基础运动还保留在状态机里

接入 PlayableGraph，不等于已经把状态机完全拆成代码。

例如，一个小型移动 Controller 可以保留待机、行走、跑步和速度 Blend Tree。当前控制器只需要描述移动动作域，交互、展示等其他动作域由独立资产承载。

同一动作域需要不同外观或动作风格时，可以用 OverrideController 复用较小的控制结构，再替换其中的片段。速度阈值和播放倍率属于动画内容配置，不需要变成所有业务共享的硬编码。

这一步拆分后，代码接管了更新调度和特殊动作混合，基础运动的片段选择、权重计算及播放倍率仍保留在 Controller 的 Blend Tree 中。进一步拆解时，才需要把这些计算移到运行时代码，并用多个 AnimationClipPlayable 直接构建运动混合图。

## 7. 特殊动作由运行时代码控制

特殊动作通过独立 Clip 节点与基础运动进行全身交叉混合。设特殊动作权重为 `w`：

```text
基础运动权重 = 1 - w
特殊动作权重 = w
```

播放流程为：

1. 若存在旧特殊片段，释放其节点。
2. 创建新的 AnimationClipPlayable，并接入 Mixer 的输入 1。
3. 将权重目标设为 1，按淡入时间逐步增加权重。
4. 停止时将权重目标设为 0，淡出完成后释放 Clip 节点。
5. 非循环片段到达末尾后请求淡出；循环片段在达到长度后回绕时间。

这是一种权重交叉混合，尚未引入 AvatarMask 或分层覆盖。连续替换特殊片段时，这里讨论的实现保留混合权重，但直接替换片段节点，并没有同时保留两个特殊片段做独立的片段间淡化。

如果特殊动作计时和权重推进也随 Graph 求值执行，降低求值频率会同时降低淡入、淡出和结束判断的时间分辨率。基础运动是否继续推进应成为明确策略，不能把特殊片段权重达到 1 等同于基础状态机已经暂停。

特殊动作和控制器替换是两种能力：前者混合一个独立 Clip，后者切换当前动作域的状态机。它们可以共用输出结构，但各自的切换和释放规则应独立设计。

## 8. 手动求值：降频，同时保留正确的播放时间

动画更新频率约定为每个游戏秒内的目标求值次数：

| 配置值 | 含义 |
| --- | --- |
| 0 | 每个渲染帧求值一次 |
| 30 | 目标每游戏秒求值 30 次 |
| 15 | 目标每游戏秒求值 15 次 |
| 1 | 目标每游戏秒求值 1 次，适合观察逐格效果 |

每帧最多求值一次，实际次数不超过当前渲染帧率。例如实际只有 20 FPS，配置 30 次/秒，也不会得到 30 个可见姿态。

这里需要两份计时：

- **动画经过时间**：累计尚未传给 Evaluate 的游戏时间。
- **调度计时**：判断是否到达下一次更新时机，并保留周期余量。

```csharp
// 以下字段由组件持有：目标频率、未求值时间和调度余量。
private float _animationUpdateRate;
private float _animationElapsedTime;
private double _animationUpdateTimer;

/// <summary>按目标频率推进动画图，合并跳过帧的时间。</summary>
private void Update()
{
    if (_animGraph == null) return;

    // 使用游戏时间，跟随 timeScale。
    float deltaTime = Time.deltaTime;
    _animationElapsedTime += deltaTime;

    if (_animationUpdateRate > 0f)
    {
        // 每个渲染帧最多求值一次。
        double interval = 1d / _animationUpdateRate;
        _animationUpdateTimer += deltaTime;
        if (_animationUpdateTimer < interval) return;

        // 保留周期余量，避免非整除频率长期偏低。
        _animationUpdateTimer %= interval;
    }
    else
    {
        _animationUpdateTimer = 0d;
    }

    // 一次推进完整的累计时间，保持整体播放节奏。
    _animGraph.Evaluate(_animationElapsedTime);
    _animationElapsedTime = 0f;
}
```

以 60 FPS、动画目标 15 次/秒为例，理想情况下每隔四帧求值一次，每次向动画传入约 `4 / 60` 秒。这样改变的是可见姿态的更新间隔，跑步循环的整体节奏保持一致。

如果跳过三帧后只传入当前帧的 `Time.deltaTime`，动画时间就会落后，表现成慢动作。

调度余量也不能每次简单清零。例如 60 FPS 下目标 40 次/秒，保留余量才能让更新间隔在不同渲染帧数之间分配，长期接近目标次数。

`Evaluate` 会求值图中的输出并更新连接的 Playable，并非仅修改一个计时变量。[Unity：PlayableGraph.Evaluate](https://docs.unity3d.com/6000.3/Documentation/ScriptReference/Playables.PlayableGraph.Evaluate.html)。

## 9. 基于距离的动画更新分级

当每个角色已经拥有可独立设置的 `AnimationUpdateRate`，上层就具备了按重要程度分配动画更新频率的基础。最直接的扩展是：其他角色距离自己越远，降低其动画姿态的更新频率。

实践中已经实现了每角色可配置的频率控制，并通过接管测试观察到了降频效果。下面的距离分级是建立在这一能力上的上层设计思路，尚未作为自动策略实现。

分级策略可以从下面的关系开始，具体数值应结合镜头、角色屏幕占比和动作观感调整：

| 角色情况 | 更新策略示例 | 意图 |
| --- | --- | --- |
| 主角、当前重点观察角色 | 0，每帧求值 | 保持主要角色动作细节 |
| 距离自己较近的其他角色 | 30 次/游戏秒 | 保留较清晰的动作变化 |
| 中远距离角色 | 15 次/游戏秒 | 降低不易辨认的姿态更新密度 |
| 更远、仍需显示的角色 | 5～10 次/游戏秒 | 用更少的求值呈现远处运动 |

这些频率只是策略示例，没有在本次测试中被验证为通用最优值，也不意味着场景本身的渲染帧率随角色距离改变。

```text
距离 / 角色重要性
        ↓
上层选择目标动画频率
        ↓
角色动画组件.AnimationUpdateRate
        ↓
累计未求值时间，到期后 Evaluate
        ↓
远处角色保持运动节奏，以较低频率更新骨骼姿态
```

这个策略调整的是动画姿态更新。业务运动数据和角色根节点位移仍按其原有流程处理，所以远处角色可以连续移动，同时减少手脚姿态求值次数。

落地时有几个边界需要明确：

- **距离参照**：可以使用其他角色到主角的距离；如果镜头能够自由移动，屏幕占比和到相机的距离也值得纳入策略。
- **分级抖动**：角色在距离阈值附近往返时，可使用不同的进入与退出阈值，避免频率来回切换。
- **策略更新频率**：距离等级可以定期刷新，没有必要为了调低动画频率而每帧重复修改配置。
- **时间连续性**：切换等级时保留尚未求值的动画经过时间，避免改变跑步节奏；相同频率重复赋值也不应打断当前周期。频率设置接口应遵守这两个约定。
- **角色批次**：大量角色同步切换到相同频率，可能集中在同一帧求值，可进一步设计不同调度相位。
- **完全不可见角色**：是否停止动画、回收模型或移出图，应属于独立的可见性与生命周期策略。当前频率值 0 表示每帧更新，不能用来表达停更。

距离分级的价值在于把已实现的底层控制能力转成上层策略：动画图负责“如何按频率正确推进”，业务负责“当前角色值得以多高的频率更新”。

## 10. 为什么减少 Evaluate 后，动画看起来没有变化

第一次把更新频率降到约 15 次/秒时，观察到动画表现没有明显差异。这个现象本身存在两种解释：

- 15 次/秒在当时的角色尺寸和观察条件下，差异不容易辨认。
- 当前看到的姿态还有其他驱动来源，手动图的降频没有完全反映到最终输出。

排查时应优先检查 Controller 的归属和参数入口是否统一。

### 10.1 同一个 Controller 资产，可以对应不同运行时实例

一个容易混淆的接法，是保留 Animator 上的 Controller，同时使用同一资源创建 AnimatorControllerPlayable：

```text
同一 Controller 资产
    ├─ Animator 自带的状态机实例：自动更新路径
    └─ AnimatorControllerPlayable 实例：自建 Graph 的手动更新路径
```

`AnimatorControllerPlayable.Create(graph, controller)` 是在图中创建节点，不是把 Animator 自带状态机的唯一实例搬进图中。

因此，只对自建 Graph 设置 Manual，不能直接推断 Animator 自身的自动状态机也已经被关闭。最终骨骼姿态由哪条路径贡献、是否发生覆盖，需要结合具体输出连接和运行时行为判断。

Unity 工程师在手动求值的相关讨论中建议：保持 Animator 启用，让它不挂状态机，使用自建图中的 AnimatorControllerPlayable 承载状态机。[官方人员的说明与示例](https://discussions.unity.com/t/manual-evaluation-doesnt-work/886418/5)。该讨论发表于 2022 年，本文将它作为设计参考，当前版本的表现仍以实际对照为准。

### 10.2 参数也应写入被手动驱动的实例

原来的速度入口调用：

```csharp
// 原路径：写入 Animator 的浮点参数。
_animator.SetFloat(speedId, speed);
```

接管后，明确把参数写入自建图的 Controller 节点：

```csharp
// 手动图路径：写入图内的状态机实例。
_controllerPlayable.SetFloat(speedId, speed);
```

资源相同不能代替参数同步。在设计上明确参数属于哪个运行时实例，才能让“速度输入—状态计算—骨骼输出”形成可追踪的链路。[Unity：AnimatorControllerPlayable.SetFloat](https://docs.unity3d.com/6000.3/Documentation/ScriptReference/Animations.AnimatorControllerPlayable.SetFloat.html)。

## 11. 接管顺序：先保留资源并建图，再清空 Controller

初始化时应先保留资源并建立手动图，再移除 Animator 自带 Controller。可以用一个临时测试开关对照两条路径：

```text
角色动画组件 Awake
    读取并缓存 Animator 原始 Controller
    使用该 Controller 建立 Manual Graph

角色包装组件 Start
    测试开启 → Animator.runtimeAnimatorController = null
    后续参数写入 → 图内 AnimatorControllerPlayable

角色动画组件 Update
    按目标频率调用 Graph.Evaluate
```

初始化的关键片段为：

```csharp
// 保留 Controller 资源，供建图和后续重建使用。
_initialController = animator.runtimeAnimatorController;

// 使用显式资源引用创建图内的状态机。
_animGraph = new AnimGraph(animator, _initialController);

// Animator 保持启用，仅移除其自带 Controller。
animator.runtimeAnimatorController = null;
```

不能先清空再依赖 `animator.runtimeAnimatorController` 建图，否则取到的是空引用。同样，重建 Graph 时也不能继续从已经清空的 Animator 上读取 Controller：

```csharp
// 先释放旧图，再使用缓存的初始 Controller 重建。
_animGraph.Release();
_animGraph = new AnimGraph(animator, _initialController);
```

这里沿用的是“重置到初始 Controller”的语义。如果产品要求恢复最后一次动态切换的动作 Controller，就应单独维护当前 Controller 引用，而不是直接复用初始缓存。

**清空 Controller 与禁用 Animator 组件不是同一件事。** Animator 仍然是 AnimationPlayableOutput 的目标，测试不通过禁用 Animator 来切断整条骨骼输出。

## 12. 对照测试实际说明了什么

测试修改同时完成了两件事：

1. 清空 Animator 自带 Controller。
2. 将速度参数直接写入图内状态机。

修改后重新测试，能够观察到降低求值频率带来的明显姿态变化。

这说明修改后的手动求值链路已经能够控制最终可见姿态，也支持继续排查原自动状态机及参数归属的方向。但由于两项调整同时发生，不能把它描述为已经独立证明“此前两份状态机每帧都完整运行”，也不能仅凭视觉变化推导性能下降比例。

一个容易复现的观察方法是：

1. 运行后选择实际显示的角色实例，确认 Animator 的 Controller 为 None。
2. 保持业务运动速度稳定，将动画更新频率设为 1～2 次/秒。
3. 近距离观察手脚是否逐格变化，排除根节点移动和镜头跟随的影响。
4. 恢复 15、30 和 0，比较姿态变化的流畅度与动作节奏。
5. 对照旧路径时，退出运行模式，关闭临时测试开关后重新运行。开关只在启动时读取，运行中取消勾选不会自动还原 Controller。

测试修改发生在运行时，未修改 Controller 资源。对普通 Play Mode 测试而言，退出后重新实例化角色会重新读取预制体配置；采用特殊的免重载工作流时，需要单独验证状态恢复。

## 13. 生命周期需要一起考虑

手动图接管后，组件生命周期需要明确管理：

- **Awake**：缓存建图资源，建立动画图。
- **Update**：写入对应实例的参数，按调度决定是否求值；参数写入与求值的先后要明确。
- **OnDisable**：这里讨论的实现清空累计动画时间和调度余量，图保留用于后续恢复。
- **Reset**：释放并重建图，使用保存的初始 Controller，清理频率计时。
- **OnDestroy**：释放 Graph，取消相关加载任务。

对于 Manual 图，跳过 Evaluate 的帧会保持上次输出的姿态；这里讨论的实现没有额外增加骨骼姿态插值。整体位移仍可每帧更新，因此低频动画配合连续移动时，近看会更容易察觉逐格效果。

如果运行中替换图内 Controller，原节点的状态和参数不会因为复用同一图而自动保留。需要哪些参数、当前动作或播放进度，都应由业务明确恢复。换装后的骨骼绑定和特殊动作切换也应单独做生命周期验证。

## 14. 这套实现的边界

运行时动画图可以统一管理创建与释放、控制器节点替换、特殊片段混合、参数入口及求值时机。每角色更新频率是已经落地并观察到效果的基础能力，距离分级则是可继续接入的上层策略。

目标是让角色按需装配小型动作控制器，而不是要求所有状态机都改写成代码。基础运动 Blend Tree 可以继续保留在较小的 Controller 内。各角色独立驱动或使用统一调度器，是另一个可以单独选择的调度层设计，不影响资产拆分原则。

要解决状态机过大的问题，核心是先划分 Controller 资产与资源依赖的边界，再由运行时代码按需装配。落到动画输出时，还要把驱动链路收敛到明确的位置：**Controller 资源由代码保留，状态机实例位于手动 Graph，参数写入该实例，Animator 接收输出。** 只有明确了这些归属，减少 Evaluate 次数才有清楚、可验证的含义。

## 参考资料

- [Unity 6.3：Animator Override Controller](https://docs.unity3d.com/6000.3/Documentation/Manual/AnimatorOverrideController.html)
- [Unity 6.3：DirectorUpdateMode.Manual](https://docs.unity3d.com/6000.3/Documentation/ScriptReference/Playables.DirectorUpdateMode.Manual.html)
- [Unity 6.3：PlayableGraph.Evaluate](https://docs.unity3d.com/6000.3/Documentation/ScriptReference/Playables.PlayableGraph.Evaluate.html)
- [Unity 6.3：AnimatorControllerPlayable.Create](https://docs.unity3d.com/6000.3/Documentation/ScriptReference/Animations.AnimatorControllerPlayable.Create.html)
- [Unity 6.3：AnimatorControllerPlayable.SetFloat](https://docs.unity3d.com/6000.3/Documentation/ScriptReference/Animations.AnimatorControllerPlayable.SetFloat.html)
- [Unity Discussions：Manual Evaluation doesn't work，Unity 工程师的回复](https://discussions.unity.com/t/manual-evaluation-doesnt-work/886418/5)
