---
title: "Surface 的组装与动态生命周期"
date: 2026-09-22
weight: 10
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

本篇讨论 Surface 如何组装，以及动态 UI Surface 的 OnChange、OnActive、OnDeactivate 等生命周期逻辑应该由谁负责。

## Q：Surface 应该由工厂组装，还是由类型自己负责动态生命周期？

### 原始提问

> 不过这还有个问题 surface 如何组装 我们之间 plugin 模式讨论了这部分 初始化对象的时候可以用工厂组装 surface  但是我们刚才提到的 UI surface 有点变成动态的了 是让这个类型自己负责 比如 on change on active on deactive 之类的 还是需要外部对这部分进行组装?

## A：

我会选择一个比较明确的边界：

> **Surface 不负责自己的组装，也不应该有复杂的 `OnActive / OnDeactivate / OnChange`。**
>
> **Surface 是一个被外部 Composer 生成的轻量 projection。**

也就是说，你前面 Object 初始化时的 Factory 思路没有错；只是 UI/Session 的 Surface 生命周期不同，所以需要一个 **Session-level Composer**，而不是 Object Factory。

---

## 1. 我会区分两类 Surface

### Object Surface：基本静态

Object 初始化：

```text
ObjectFactory
    ↓
Runtime Extensions install
    ↓
Runtime resolve
    ↓
Control / Capability registration
    ↓
Seal
```

例如 Tank：

```text
Tank Runtime
├── IMovementControl
├── IVehicleStatus
└── ...

Turret Runtime
├── IAimControl
└── ...

Cannon Runtime
├── ITriggerControl
├── IAmmoStatus
└── ...
```

这些东西跟着 Object 生命周期，通常不用频繁变化。

---

### Session Surface：动态 projection

玩家当前状态：

```text
ControlledObject = Tank
Seat             = Gunner
Weapon           = Cannon
Optics           = Scope
```

Composer 根据这些关系产生：

```text
ControlSurface
├── IAimControl       → Turret
├── ITriggerControl   → Cannon
└── IZoomControl      → Scope

PresentationSurface
├── IVehicleStatus    → Tank
├── IAmmoStatus       → Cannon
├── IAimStatus        → Turret
└── IZoomStatus       → Scope
```

这里 Surface 的动态性来自：

> **Source Context 在变化。**

而不是 Surface 自己在运行复杂逻辑。

---

# 2. 所以我会增加一个很薄的 `SessionContext`

例如：

```csharp
class PlayerSessionContext
{
    public Entity ControlledObject;
    public Seat CurrentSeat;
    public Entity CurrentWeapon;
    public Entity CurrentOptics;
}
```

它描述：

> 玩家当前到底处于什么上下文。

然后：

```text
PlayerSessionContext
        │
        ▼
  SurfaceComposer
        │
   ┌────┴────┐
   ▼         ▼
Control    Presentation
Surface      Surface
```

SurfaceComposer 才负责组装。

---

# 3. 谁触发 Composer？

这里我觉得没必要搞复杂的全自动 dependency tracking。

直接让 **Session Controller** 管理结构变化就够了。

例如：

```text
EquipWeapon(RPG)
    ↓
SessionContext.CurrentWeapon = RPG
    ↓
Recompose()
```

进入 Tank：

```text
EnterSeat(tank.GunnerSeat)
    ↓
SessionContext
    ControlledObject = tank
    Seat = gunner
    Weapon = cannon
    ↓
Recompose()
```

换瞄准镜：

```text
EquipOptics(scope)
    ↓
Context.Optics = scope
    ↓
Recompose()
```

也就是说：

```text
                  structural change
                        │
                        ▼
                Session Controller
                        │
                        ▼
                  update Context
                        │
                        ▼
                Surface Composer
                        │
             ┌──────────┴──────────┐
             ▼                     ▼
       ControlSurface      PresentationSurface
             │                     │
             ▼                     ▼
        Controller             HUD / Camera
```

非常直白。

---

# 4. Surface 最好是 dumb object

甚至可以近似：

```csharp
public sealed class Surface
{
    private readonly Dictionary<Type, object> capabilities;

    public T? Get<T>() where T : class;
}
```

或者更加 immutable：

```csharp
var surface = Surface.Create(
    aimControl,
    triggerControl,
    zoomControl
);
```

它没有：

```text
Update()
OnActive()
OnDeactivate()
OnWeaponChanged()
OnSeatChanged()
```

这些生命周期。

最多：

```text
Create
Dispose
```

甚至 `Dispose` 都可能不需要。

因为它只是：

> **一组已经 resolve 好的 references。**

---

# 5. 动态切换最好是“换 Surface”，而不是“修改 Surface”

例如当前：

```text
PresentationSurface #17

IHealthStatus → Soldier
IAmmoStatus   → AK47
IZoomStatus   → RedDot
```

换 RPG 后，不要：

```text
surface.Remove(IAmmoStatus)
surface.Add(RPGAmmo)
surface.Remove(IZoomStatus)
...
```

直接：

```text
PresentationSurface #18

IHealthStatus → Soldier
IAmmoStatus   → RPG
IZoomStatus   → RPGOptics
```

然后：

```csharp
hud.Bind(newSurface);
camera.Bind(newSurface);
```

旧 Surface 直接丢掉。

这种 immutable snapshot/projection 思路会简单很多。

而且切武器、换座位本身就是低频操作，创建几个 interface reference 的小对象根本不是性能问题。

---

# 6. 但有一个问题：Composer 怎么知道“去哪找 IAimControl”？

这里才是整个问题真正有意思的地方。

我认为有两种方式，而且可以同时存在。

### A. Convention / Scope resolution

例如：

```text
GunnerSeat
requires:
    IAimControl     from Turret
    ITriggerControl from Weapon
```

Composer：

```text
Seat
 ↓
ControlBinding Definition
 ↓
resolve capability sources
```

比如 Definition：

```yaml
control:
  aim: turret
  fire: weapon
  zoom: optics
```

最终：

```text
IAimControl     ← CurrentTurret
ITriggerControl ← CurrentWeapon
IZoomControl    ← CurrentOptics
```

---

### B. Seat/Object 自己声明 Binding Recipe

我其实更喜欢这个。

例如 GunnerSeat 并不自己成为 Surface，但它知道：

> “坐在我这里的人应该控制什么。”

```text
GunnerSeat
│
└── ControlBinding
    ├── Aim  → ../Turret
    ├── Fire → ../Turret/Cannon
    └── Zoom → ../Optics
```

Presentation：

```text
GunnerSeat
└── PresentationBinding
    ├── Vehicle → $root
    ├── Weapon  → Cannon
    ├── Aim     → Turret
    └── Optics  → Optics
```

然后：

```text
Binding Recipe
      +
Current Object Graph
      ↓
SurfaceComposer
      ↓
Surface
```

这样就不会让 Composer 本身出现：

```csharp
if (tank)
   ...
else if (soldier)
   ...
else if (aircraft)
   ...
```

---

# 7. 这里 Plugin 思想又能回来

不同对象类型可以注册自己的 Surface Provider/Recipe：

```text
SoldierControlPlugin
TankGunnerControlPlugin
AircraftPilotControlPlugin
```

但它们不是 Surface 本身。

而是：

```text
Plugin
   ↓ registers
Binding Recipe / Surface Provider
   ↓ used by
SurfaceComposer
   ↓ produces
Surface
```

比如：

```text
Tank Plugin

register:
    GunnerSeatBinding
        requires IAimControl
        requires ITriggerControl
        requires IZoomControl
```

这和我们之前 Object Runtime Extension 很像：

```text
ObjectFactory + Plugins
        ↓
Object Runtime
```

对应：

```text
SurfaceComposer + Binding Providers
        ↓
Session Surface
```

其实是一模一样的 composition pattern，只是 lifecycle 不一样。

---

# 8. HUD Plugin 又在 Surface 之后

所以不要把两个 Plugin composition 混起来。

第一层：

```text
Game Object
     ↓
SurfaceComposer
     ↓
PresentationSurface
```

解决：

> 当前玩家正在观察什么？

第二层：

```text
PresentationSurface
       ↓
HUD Composer
       ↓
HUD Plugins
```

解决：

> 当前这些能力应该显示成什么 UI？

完整就是：

```text
Tank ──────────┐
Turret ────────┤
Cannon ────────┼── SurfaceComposer
Optics ────────┘
                       │
                       ▼
             PresentationSurface
             ├── IVehicleStatus
             ├── IAmmoStatus
             ├── IAimStatus
             └── IZoomStatus
                       │
                       ▼
                  HUD Composer
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
      AmmoPlugin   VehiclePlugin   ZoomPlugin
          │            │            │
          └────────────┼────────────┘
                       ▼
                  HUD Runtime
```

这两个 Composer 职责完全不同。

---

# 9. `OnActive / OnDeactivate` 应该放在哪里？

不是 Surface。

应该在 **Surface Consumer / Plugin**。

比如：

```csharp
AmmoWidgetPlugin.Bind(IAmmoStatus ammo)
{
    this.ammo = ammo;
}

AmmoWidgetPlugin.Unbind()
{
    this.ammo = null;
}
```

或者更简单：

```text
HUD detects surface changed
        ↓
destroy affected widget scope
        ↓
create new widget scope
```

如果 Widget 需要订阅 `HitConfirmed`：

```text
WidgetPlugin.OnMount
    ↓
subscribe

WidgetPlugin.OnUnmount
    ↓
unsubscribe
```

这就是 DSH/Cordis 生命周期管理真正适合借过来的地方。

因此生命周期属于：

```text
Plugin / Consumer
```

而不是：

```text
Surface
```

---

## 我现在会把整个初始化流程定成两种 Composition

这可能是目前最重要的收敛：

```text
        STATIC COMPOSITION
        ==================

Definition
    ↓
ObjectFactory
    ↓
Runtime Extensions
    ↓
Object Runtime
    ↓
Seal


        DYNAMIC COMPOSITION
        ===================

SessionContext
    ↓
Binding Recipe
    ↓
SurfaceComposer
    ↓
Surface Snapshot
    ↓
Consumer Rebind
    ├── Controller
    ├── HUD Composer
    └── Camera
```

前者发生在 **Object creation**。

后者发生在 **Player context structural change**。

普通游戏帧里两个都不发生：

```text
Frame N
────────────────────────

Controller
   ↓ cached ControlSurface
   ↓ call

Game Logic

HUD
   ↓ cached PresentationSurface
   ↓ pull

Camera
   ↓ cached PresentationSurface
   ↓ pull
```

所以运行时最热路径反而极其简单。

我尤其不建议让 Surface 自己实现 `OnChange / OnActive / OnDeactivate`，因为那会让 Surface 从一个很干净的 **projection/value object** 慢慢长成另一个 Runtime。**动态性应该属于 SessionContext；组装逻辑属于 Composer；生命周期属于 Consumer/Plugin；Surface 本身保持“傻”。**

这四个职责一分开，我觉得这部分架构就基本不会乱了。

## 整理说明

1. 本篇保留原始回答关于 Factory、Composer、Surface 自身职责和动态生命周期边界的完整讨论。
2. 整理时特别保留了“Surface 不负责自己的组装”这一核心判断，因为它直接关系到动态 UI 和 Runtime 初始化的复杂度。
3. 动态绑定不等于每帧重新解析依赖；结构变化时重新 compose，普通数据变化则应通过已建立的能力接口读取或订阅。
