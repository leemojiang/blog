---
title: "Camera 与 Zoom 的单例 Runtime"
date: 2026-09-22
weight: 3
draft: false
math: true
tags:
  - "战地 2"
  - "游戏架构"
  - "组件系统"
  - "插件架构"
  - "UI"
categories:
  - "软件架构"
---

本篇讨论 Camera、Zoom、Camera Config 和 Player Session 之间的关系，重点是单例系统如何同时暴露行为、只读状态和配置数据。

## Q：Camera 和 Zoom 应该如何设计？Camera 是单例 Runtime 吗？Zoom 应该通过属性还是接口暴露？

### 原始提问

> 非常不错 我感觉清晰很多了 分开runtime (需要的具体类型)  surface (提供能力的接口,可以不同的类型组成 runtime view但是能力的接口是 统一的 也就是说surface 接口即是能力 也方便思考交互关系.  那么你觉得 Camera 和zoom 要怎么设计? 
>
> 我觉得camera zoom 有了一点小变化 之前的object 有很多 所以谁来使用它变得有些复杂 但是camera 似乎是单例 而camera config类似运行时override 的config . 我感觉camera 应该的层级是类似player controller 这个位置 (似乎是一个单例插件? 那么camera comp 就相当于是他的runtime? 
>
> 类似的zoomcomp 怎么处理呢 它似乎也不是进行控制了 而是一个数据runtime 提供一些必要的数据封装? runtime.zoom.data这样? (但这个接口似乎和行为不太一样了 是使用属性暴露 还是依然构造接口暴露?

## A：

对，我觉得这里正好能把我们现在的模型再推进一步：**Surface 不一定只表示“可执行行为”，它更准确地表示“对外 contract”。Contract 可以是 Command，也可以是 Read-only State，甚至可以是 Config/Descriptor。**

Camera / Zoom 正好能帮助我们把这个概念想清楚。

---

## 1. Camera 我会放在 Player Session Runtime，而不是 Object Runtime

你说 Camera 和 Player Controller 是同一层级，我基本同意。

更准确一点：

```text
Game
│
├── World Runtime
│
└── Local Player Session
    │
    ├── PlayerController
    ├── CameraController
    ├── Input
    ├── HUD
    └── AudioListener
```

所以 Camera 不是：

```text
Soldier
└── CameraComponent
```

也不是：

```text
Tank
└── CameraComponent
```

而是：

```text
LocalPlayerSession
└── CameraRuntime
```

如果游戏永远只有一个本地玩家，它**表现得像 Singleton**；但架构上最好定义成：

> **Per-Local-Player Session Service**

这样以后 spectator、split screen、editor preview 都不会被 singleton 卡死。

---

# 2. Object 不提供 Camera，而是提供 Camera Surface

例如 Soldier：

```text
Soldier
│
├── Head
├── Aim
└── CameraSurface
    ├── CameraMount
    ├── FOV = 75
    └── ...
```

Tank Gunner：

```text
Tank
└── GunnerSeat
    └── CameraSurface
        ├── Mount → Optics
        ├── FOV = 40
        └── ...
```

Sniper Scope：

```text
Weapon
└── Scope
    └── CameraSurface
        ├── ZoomFOV
        └── ScopeOverlay
```

真正的 Camera 永远只有 Session 那一个：

```text
              Soldier CameraSurface
                     \
                      \
Tank CameraSurface ───→ CameraController → Unity Camera
                      /
                     /
              Scope CameraSurface
```

CameraController 消费这些数据，然后控制 Unity Camera。

---

# 3. 但这里不要让 `CameraConfig` 成为一个巨大 mutable config

比如这样我不太喜欢：

```csharp
camera.config.fov = 30;
camera.config.position = xxx;
camera.config.sensitivity = 0.5f;
camera.config.nearClip = ...
camera.config.postProcess = ...
```

因为所有模块开始一起修改一个全局 Camera 状态。

最后变成：

```text
Weapon ───────┐
Vehicle ──────┤
Sprint ───────┼──→ CameraConfig
Damage ───────┤
Aim ──────────┤
UI ───────────┘
```

这就是另一种 Super Runtime。

更好的模型是：

> **Object 提供 Camera Requirement / Profile，CameraController 负责合成最终 Camera State。**

---

# 4. Camera 可以分成 Profile 和 Override

例如基础视角：

```csharp
public interface ICameraProfile
{
    Transform Mount { get; }
    float BaseFov { get; }
}
```

Soldier：

```text
SoldierCameraProfile

Mount   = Head
BaseFov = 75
```

Tank Gunner：

```text
GunnerCameraProfile

Mount   = OpticsMount
BaseFov = 55
```

然后 Zoom 并不修改 Camera Profile。

Zoom 提供：

```text
Camera Override
```

例如：

```text
Base Camera
    FOV 75
       │
       ▼
Aim Override
    FOV 60
       │
       ▼
Scope Zoom Override
    FOV 25
       │
       ▼
CameraController
       │
       ▼
Unity Camera
```

所以最终状态来自组合，而不是大家一起写 Camera。

---

# 5. 这时候 Zoom 的身份就非常有意思

你说：

> ZoomComp 好像不再是“控制”，而是 data runtime。

我认为**对一半**。

Zoom 通常同时存在两种语义：

### Control

外部告诉它：

```text
StartZoom
StopZoom
SetZoomLevel
NextZoomLevel
```

### State

外部观察：

```text
IsZoomed
ZoomLevel
CurrentFovMultiplier
```

所以一个 Zoom Runtime 可以同时提供两个 Surface：

```text
ZoomRuntime
│
├── IZoomControl
│
└── IZoomState
```

例如：

```csharp
public interface IZoomControl
{
    void SetZoom(bool enabled);
}

public interface IZoomState
{
    bool IsZoomed { get; }
    float FovMultiplier { get; }
}
```

同一个：

```csharp
class ZoomComponent :
    IZoomControl,
    IZoomState
```

完全没问题。

---

# 6. 这其实回答了“Surface 到底是不是行为接口”

不是。

我们现在最好把 Surface 理解成：

> **Capability Contract**

而 Capability 可以分成两大类：

```text
Capability
│
├── Control / Command
│
│   ITriggerControl
│   IAimControl
│   IZoomControl
│
└── State / Query
    │
    IAmmoStatus
    IWeaponStatus
    IZoomState
    IHealthStatus
```

甚至以后还有：

```text
Descriptor / Config
```

例如：

```text
IWeaponDescriptor
├── Icon
├── DisplayName
└── Category
```

因此：

```text
Surface ≠ Method Interface

Surface = 对外可见的 contract
```

属性完全合理。

---

# 7. 但我不建议出现 `runtime.zoom.data`

这会开始产生这种 API：

```csharp
runtime.weapon.fire.data...
runtime.weapon.zoom.data...
runtime.vehicle.engine.data...
```

最后其实是在重新发明一棵对象树。

我更喜欢直接 capability：

```csharp
IZoomControl zoomControl;
IZoomState zoomState;
```

使用者根据需求拿不同的面。

PlayerController：

```text
IZoomControl
```

HUD：

```text
IZoomState
```

Camera：

```text
IZoomState
```

于是：

```text
                 ZoomComponent
                 /           \
                /             \
               ▼               ▼
        IZoomControl       IZoomState
             ▲             ▲       ▲
             │             │       │
       PlayerController   HUD   CameraController
```

非常清楚。

---

# 8. Camera 甚至不应该直接依赖 ZoomComponent

这一点非常重要。

不要：

```text
CameraController → ZoomComponent
```

而是：

```text
CameraController → ICameraModifier
```

因为 Zoom 只是 Camera modifier 的一种。

以后可能还有：

```text
SprintFOV
DamageShake
Recoil
VehicleVibration
Underwater
ADS
ScopeZoom
Cinematic
```

如果 CameraController 逐个知道：

```csharp
if (zoom != null) ...
if (sprint != null) ...
if (recoil != null) ...
if (damage != null) ...
```

Camera 又变 Super Runtime。

---

# 9. 可以把 Camera Override 抽象成统一 Modifier

例如：

```csharp
public interface ICameraModifier
{
    void Apply(ref CameraState state);
}
```

基础：

```text
CameraProfile
    ↓
CameraState
```

然后：

```text
CameraState
    ↓
ADS Modifier
    ↓
Zoom Modifier
    ↓
Recoil Modifier
    ↓
Shake Modifier
    ↓
Final CameraState
```

最后：

```text
Final CameraState
        ↓
UnityCameraAdapter
```

于是 CameraController 只知道：

```text
ICameraProfile
Many<ICameraModifier>
```

而不知道：

```text
Zoom
Sprint
Recoil
Vehicle
Explosion
```

这就又是我们前面说的 **Many<T> + Interface Segregation**。

---

# 10. 不过 Zoom 本身不一定等于 Camera Modifier

这个 distinction 很重要。

例如狙击镜 Zoom：

```text
ZoomComponent
├── zoom level
├── current state
└── transition
```

它可能影响：

```text
Camera FOV
Mouse sensitivity
HUD scope overlay
Weapon sway
Aim precision
```

所以不要让：

```text
Zoom = CameraEffect
```

更合理：

```text
                     ZoomRuntime
                   /      |       \
                  /       |        \
                 ▼        ▼         ▼
          IZoomControl IZoomState ZoomChanged
              ▲           ▲          │
              │           │          │
           Player       Camera       ├── HUD
                                     └── Aim
```

Camera 可以有一个 adapter：

```text
ZoomCameraModifier
      │
      ├── consumes IZoomState
      │
      └── provides ICameraModifier
```

于是：

```text
Zoom
 ↓
IZoomState
 ↓
ZoomCameraModifier
 ↓
ICameraModifier
 ↓
Camera
```

这比 Camera 直接知道 Zoom 更干净。

---

# 11. Config、State、Control 最好明确区分

我觉得这可能是你这个问题里最值得加入我们 framework 的 distinction：

```text
ZoomDefinition / Config
────────────────────────
maxZoom = 4x
levels = [1x, 2x, 4x]
transitionTime = 0.15

          ↓ creates

ZoomRuntime State
────────────────────────
currentLevel = 2
isZoomed = true
transition = 0.73

          ↓ exposes

Surface
────────────────────────
IZoomControl
IZoomState
```

也就是：

```text
Definition
    ↓
Runtime
    ↓
Surface
```

分别回答：

```text
Definition → 应该是什么？
Runtime    → 现在是什么？
Surface    → 别人允许看到/操作什么？
```

这和我们最早从 BF2 学到的 Definition / Runtime separation 又接回来了。

---

# 12. Camera Config 也一样

比如 DSL：

```text
CameraProfile SoldierCamera
{
    fov = 75
    nearClip = 0.05
    smoothing = 0.1
}
```

这是：

```text
Definition
```

进入 Tank：

```text
CameraController.Bind(
    GunnerCameraProfile
)
```

运行中：

```text
CameraState
{
    position
    rotation
    currentFov
}
```

ADS：

```text
AimCameraModifier
```

Explosion：

```text
ShakeCameraModifier
```

最后：

```text
                 Definition
                     │
                     ▼
               CameraProfile
                     │
                     ▼
                 Base State
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
       ADS          Zoom        Recoil
     Modifier      Modifier     Modifier
        │            │            │
        └────────────┼────────────┘
                     ▼
              Final CameraState
                     │
                     ▼
              Unity Camera Adapter
```

这个模型非常适合游戏 Camera，因为 Camera 本来就是一个**大量 runtime override 叠加**的问题。

---

# 13. Camera“单例 Plugin”可以这么理解

如果沿用我们的 plugin/runtime 语言：

```text
LocalPlayerSession
│
├── InputPlugin
│      └── InputRuntime
│
├── ControllerPlugin
│      └── ControllerRuntime
│
├── CameraPlugin
│      └── CameraRuntime
│
└── HUDPlugin
       └── HUDRuntime
```

`CameraPlugin` 负责安装：

```text
CameraController
UnityCameraAdapter
CameraModifierRegistry
```

然后 Session runtime 对外可能提供：

```text
ICameraService
```

但 Object 不依赖这个 service。

Object 只提供：

```text
ICameraProfile
ICameraMount
IZoomState
...
```

Camera Session Plugin 主动消费当前 Presentation Surface。

这个依赖方向非常漂亮：

```text
Object
  │
  │ provides
  ▼
Presentation Surface
  ▲
  │ consumes
Camera Plugin
```

而不是：

```text
Weapon → GlobalCamera.Instance.SetFOV()
```

后者应该尽量禁止。

---

## 所以我们现在其实可以把 Surface 再定义得更准确一点

我会放弃“Surface = 提供行为”的定义，改成：

> **Surface 是 Runtime 向某类外部使用者暴露的、经过裁剪的 Capability 集合。**

Capability 又可以有三种常见形态：

```text
Control Capability
    ITriggerControl
    IAimControl
    IZoomControl

State Capability
    IAmmoStatus
    IZoomState
    IHealthStatus

Presentation / Descriptor Capability
    ICameraProfile
    IWeaponDescriptor
    IReticleProvider
```

这样 Zoom 一点都不特殊：

```text
Zoom Definition
      ↓
Zoom Runtime
      ├── IZoomControl ─── Player/AI
      └── IZoomState ───── Camera/HUD
```

Camera 才是真正比较特殊的那个：**它不是普通 Object Component，而是 Local Player Session Plugin；它消费当前对象提供的 Presentation Surface，把 Profile + 多个 Modifier 合成为最终 CameraState。**

这样也能避免一个非常常见的 FPS 架构灾难：

```text
Weapon → Camera
Sprint → Camera
Player → Camera
Vehicle → Camera
Damage → Camera
Explosion → Camera
UI → Camera
```

全部改成：

```text
            Presentation / State
                    ↓
              Camera Runtime
                    ↓
         profile + modifiers
                    ↓
              Camera State
                    ↓
               Unity Camera
```

我觉得到这里，我们的 `Definition → Runtime → Surface → Session Consumer` 四层模型已经开始相当完整了。

## 整理说明

1. 本篇保留了 Surface 作为统一 Contract 的扩展：Contract 不仅可以表示 Command，也可以表示 Read-only State、Config 或 Descriptor。
2. Camera 更接近 Player Session/Controller 层的长期存在 Runtime；Camera Config 是可组合、可覆盖的数据，而不是必须拥有完整行为的 Component。
3. Zoom 可以作为 Camera Runtime 的一个独立 capability 或 config/state provider；是否提供属性取决于它是稳定只读状态还是需要行为约束的操作接口。
4. 单例只描述生命周期和作用域，不意味着 Camera 可以访问整个游戏 Runtime；它仍然应该通过裁剪后的 Surface/Capability View 读取需要的数据。
