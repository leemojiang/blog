---
title: "HUD 与动态 Runtime"
date: 2026-09-22
weight: 4
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

本篇讨论 HUD 为什么同时具有单例和多对象两种性质，以及 Player Controller/Session、HUD Renderer、当前控制对象和 Presentation Surface 之间的生命周期关系。

## Q：HUD 如何作为 Player Controller/Session 的 Runtime，动态绑定不同物品的 Surface？

### 原始提问

> 哪这么想我们觉得我们似乎也解决了 HUD 的设计方法: 它似乎和camera很类似 展示部分是一个单例 但是下面需要读取很多object的属性,并且时刻进行更新.
>
> HUD 整体应该挂在player controller/session 作为 它的runtime
>
> 我觉得难点是如何处理初始化 以及数据同步的逻辑
> 按照之前的介绍,是不是可以理解成我把玩家控制的物品 比如 playercontrolobject / weapon 实现 hud surface 这样切换物品的时候进行动态绑定? 
>
> 不过这么说似乎又有点问题 hud 每个物体可能都不一样 也就是说它其实既有单例的特性 又有多个对象的性质, 那是不是需要动态runtime 的一些思路了, hud render 是最核心的runtime 加载了某个物品 它的surface 给render 提供需要渲染的对象 
>
> 另一个问题是hud 如何进行数据绑定 hud 似乎和物品很接近 但是似乎谁都不是很适合作为谁的runtime (hud 是物品的观察者) 也许需要让对象有一个 event comp, hud 把这个event comp作为runtime 来订阅物品的信息? 或者有别的设计模式吗 我很好奇deepseek harness里面是如何处理UI HUD 这些部分的 它们怎么作为plugin 进行设计的?

## A：

对，我觉得你已经抓到 HUD 最关键的性质了：

> **HUD Renderer 是 Session 级长期存在的 Runtime；具体 HUD 内容却是跟随当前 Context 动态组合出来的。**

所以 HUD 看起来“既是 Singleton 又是多个对象”，其实是因为我们把两个生命周期不同的东西都叫 HUD 了。把它拆开以后会非常清楚。

---

## 1. HUD 应该拆成 `HUD Runtime` 和 `HUD Presentation`

我会这样放：

```text
LocalPlayerSession
│
├── PlayerController
├── CameraRuntime
├── InputRuntime
│
└── HUDRuntime                 ← 长生命周期，通常只有一个
      │
      ├── HUDRenderer
      ├── Layout / Layer
      ├── WidgetFactory
      └── CurrentPresentation  ← 动态
```

而当前 Presentation：

```text
HUDPresentation
├── HealthWidget
├── AmmoWidget
├── CrosshairWidget
├── VehicleWidget
└── LockOnWidget
```

所以：

**HUD Runtime 是单例性质的 infrastructure；Widget/Presentation 是动态的 view composition。**

这和 Camera 非常像：

```text
CameraRuntime                    HUDRuntime
     │                               │
     ▼                               ▼
CameraProfile                  HUDPresentation
     │                               │
Modifiers                         Widgets
     │                               │
     ▼                               ▼
CameraState                      Render State
```

---

# 2. Object 不应该“实现一个巨大的 HUDSurface”

这里我会稍微修改你的设想。

不要：

```csharp
class Tank : IHUDSurface
class Rifle : IHUDSurface
class Soldier : IHUDSurface
```

然后：

```text
HUD ← IHUDSurface
```

因为最终 `IHUDSurface` 很可能变成：

```csharp
interface IHUDSurface
{
    float Health { get; }
    int Ammo { get; }
    bool Reloading { get; }
    float Speed { get; }
    float Altitude { get; }
    bool LockedOn { get; }
    ...
}
```

Super Runtime 又以 HUD 形式回来了。

应该继续贯彻我们的组合思想：

```text
Presentation Surface
│
├── IHealthStatus
├── IAmmoStatus
├── IWeaponStatus
├── IAimStatus
├── IVehicleStatus
└── ILockOnStatus
```

Tank 当前的 Presentation Surface 可能：

```text
Tank
├── IHealthStatus
├── IVehicleStatus
├── ISpeedStatus
└── IFuelStatus
```

Cannon：

```text
Cannon
├── IAmmoStatus
├── IWeaponStatus
└── IAimStatus
```

然后当前 Gunner Seat 的 Presentation Context 可以把两边**组合起来**：

```text
Tank ───────────┐
                │
Turret ─────────┼──► PresentationContext
                │
Cannon ─────────┘
```

得到：

```text
Gunner PresentationContext
├── IHealthStatus
├── IVehicleStatus
├── IAmmoStatus
├── IWeaponStatus
└── IAimStatus
```

HUD 消费这个 Context。

---

# 3. 这和前面的 Control Surface 正好形成镜像

我觉得这是目前整个架构里非常漂亮的一点：

```text
                    Object Graph
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
        Control Surface      Presentation Surface
              │                     │
              ▼                     ▼
          Controller               HUD
              │                   Camera
              │                    Audio?
              ▼
        writes / commands       reads / observes
```

甚至可以简单理解成：

```text
Control Surface
    外界 → Object

Presentation Surface
    Object → 外界
```

所以 HUD 不属于 Weapon，Weapon 也不属于 HUD。

它们通过 Presentation Surface 相遇。

---

# 4. 切换武器时，不需要重建 HUD Runtime

例如当前：

```text
Player
└── AK47
```

PresentationContext：

```text
IHealthStatus  → Player
IAmmoStatus    → AK47
IWeaponStatus  → AK47
```

HUD Runtime：

```text
HealthWidget → IHealthStatus
AmmoWidget   → IAmmoStatus
Crosshair    → IWeaponStatus
```

换 RPG：

```text
Player
└── RPG
```

只发生：

```text
PresentationContext changed

IAmmoStatus:
    AK47 → RPG

IWeaponStatus:
    AK47 → RPG
```

HUD Runtime 不动。

甚至 Widget 如果两种武器都提供相同 capability：

```text
IAmmoStatus
```

`AmmoWidget` 都不用重新创建，只需要 **rebind**。

---

# 5. 进入载具才可能改变 HUD Composition

比如步兵：

```text
PresentationContext
├── IHealthStatus
├── IAmmoStatus
└── IWeaponStatus
```

HUD：

```text
Health
Ammo
Crosshair
```

进入坦克：

```text
PresentationContext
├── IVehicleHealthStatus
├── ISpeedStatus
├── IAmmoStatus
├── IAimStatus
└── IReloadStatus
```

HUD Composer 可以发现：

```text
ISpeedStatus
    ↓
install SpeedWidget

IReloadStatus
    ↓
install ReloadWidget
```

因此：

```text
Presentation Context
        │
        ▼
    HUD Composer
        │
   capability matching
        │
        ▼
HUD Presentation
├── SpeedWidget
├── AmmoWidget
├── ReloadWidget
└── CrosshairWidget
```

这就是你说的“动态 Runtime”的地方。

但我会叫它：

> **Dynamic Presentation Composition**

而不是 Dynamic Runtime。

因为 HUD 的核心 Runtime 并没有改变。

---

# 6. 数据同步：不要让 HUD 每帧 GetComponent

这是第二个核心问题。

你提出：

> HUD 订阅 object 的 event comp？

方向是对的，但我不会做：

```text
Object
└── EventComponent
       ↑
       │
      HUD
```

因为这样 HUD 又依赖一个具体 Component。

应该是：

```text
IAmmoStatus
├── CurrentAmmo
└── Changed event
```

或者更一般一点：

```csharp
public interface IReadonlyValue<T>
{
    T Value { get; }
    event Action<T> Changed;
}
```

于是：

```csharp
public interface IAmmoStatus
{
    IReadonlyValue<int> Ammo { get; }
    IReadonlyValue<int> Capacity { get; }
}
```

HUD：

```text
AmmoWidget
     │
     ▼
IAmmoStatus.Ammo
     │
     ├── initial Value
     │
     └── Changed event
```

---

# 7. 初始化 + 增量更新是最好用的组合

绑定的时候：

```text
Bind
 ↓
读取当前值
 ↓
Render initial state
 ↓
Subscribe Changed
```

之后：

```text
Ammo changed
 ↓
Changed(29)
 ↓
AmmoWidget
 ↓
"29 / 30"
```

解绑：

```text
Unbind
 ↓
unsubscribe
```

这解决一个经典 Event-only 架构的问题。

如果 HUD 只订阅：

```text
AmmoChanged
```

那么刚打开 HUD 的时候：

> 当前 ammo 到底是多少？

不知道。

所以正确模型应该是：

> **State 用于初始化，Event 用于增量同步。**

这个原则非常通用。

---

# 8. 我甚至建议把这种模式做成 Framework Primitive

因为 Camera/UI/Debug Panel/AI perception 很可能都会用。

例如：

```csharp
public interface IReadOnlyReactiveValue<T>
{
    T Value { get; }
    IDisposable Subscribe(Action<T> listener);
}
```

那么：

```text
IAmmoStatus
    Ammo : IReadOnlyReactiveValue<int>

IHealthStatus
    Health : IReadOnlyReactiveValue<float>

ILockStatus
    Target : IReadOnlyReactiveValue<Entity?>
```

Widget：

```csharp
subscription =
    ammo.Ammo.Subscribe(UpdateAmmo);
```

Subscribe 时甚至可以约定：

> 立即发送 current value。

那么初始化 + 同步直接统一：

```text
Bind
 ↓
Subscribe
 ↓
29        ← immediately
 ↓
30
 ↓
29
 ↓
28
...
```

这就是 Reactive UI 的基本思想。

---

# 9. 但也不要把所有属性都 Reactive 化

这是另一个容易过度设计的地方。

例如：

```text
DisplayName
Icon
WeaponType
ReticleStyle
```

通常是 definition/static descriptor：

```csharp
interface IWeaponDescriptor
{
    string Name { get; }
    Sprite Icon { get; }
}
```

没必要：

```text
ReactiveValue<string> Name
```

而：

```text
Ammo
Health
ReloadProgress
LockTarget
```

是动态 state：

```text
ReactiveValue<T>
```

所以可以区分：

```text
Descriptor
──────────
Name
Icon
Type

State
──────────
Ammo
Health
Reload

Event
──────────
WeaponFired
HitConfirmed
Kill
```

三种数据不要全部强行 Event 化。

---

# 10. HUD Widget Plugin 可以声明自己需要什么

这就真正进入我们之前的 Plugin 思路了。

例如：

```text
AmmoWidgetPlugin

requires:
    IAmmoStatus
```

```text
VehicleSpeedWidgetPlugin

requires:
    ISpeedStatus
```

```text
LockOnWidgetPlugin

requires:
    ILockOnStatus
    IAimStatus
```

HUD Composer：

```text
Current Presentation Surface
              │
              ▼
       Widget Registry
              │
        match requirements
              │
      ┌───────┼────────┐
      ▼       ▼        ▼
    Ammo    Speed    LockOn
```

所以 Widget 本身其实很像 Runtime Extension：

```text
Plugin
   ↓
requires capabilities
   ↓
install Widget
```

只不过它安装到：

```text
HUD Presentation Runtime
```

而不是 Game Object Runtime。

这就是你感觉 HUD “既单例又多对象”的真正原因：

```text
HUD Core Runtime       1 个

HUD Presentation       当前 context 一个

Widget Instances       N 个
```

三种 lifecycle 不一样。

---

# 11. 更进一步：一个 Widget 可以绑定多个 Object

这也解释了为什么不要说：

> HUD 是某个 Object 的 Runtime。

例如 Minimap：

```text
Minimap
  ↓
World
  ↓
100 objects
```

Kill Feed：

```text
KillFeed
  ↓
World Events
```

Objective UI：

```text
ObjectiveWidget
  ↓
GameMode Runtime
```

Ammo：

```text
AmmoWidget
  ↓
Current Weapon
```

Health：

```text
HealthWidget
  ↓
Controlled Character
```

所以 HUD 实际上是：

```text
                 HUD Runtime
                      │
       ┌──────────────┼───────────────┐
       ▼              ▼               ▼
Player Context    World Runtime   GameMode Runtime
       │              │               │
       ▼              ▼               ▼
Ammo / Health      Minimap        Objective UI
```

这也再次说明：

> HUD 不应该和某一个 Object 建立 ownership 关系。

它是 Session Presentation System。

---

# 12. 关于 DSH：它给我们的启发更多是 Plugin 生命周期，而不是照搬 UI 模型

这里我会谨慎一点：我们前面讨论 DeepSeek Harness 时主要提炼的是它的 Runtime/Plugin 组织思想；如果要精确回答“DSH 当前 UI 插件具体如何注册、拿什么 runtime、如何更新状态”，我需要重新看你那份 DSH 开发指南/源码，不能凭之前的架构讨论把我们的模型说成它的真实实现。

但从我们已经提炼出的 DSH 思路来看，最值得借的是：

```text
Core Runtime
     │
Plugin installs capability
     │
Other plugin consumes capability
```

对应 HUD 就是：

```text
HUD Core
   │
   ├── Widget Registry
   ├── Layout
   └── Render Host
          ▲
          │
    UI Plugins register
          │
    ┌─────┼─────┐
    ▼     ▼     ▼
  Ammo  Radar  Vehicle
```

也就是说 HUD Core 不应该写死：

```csharp
new AmmoHUD();
new TankHUD();
new PlaneHUD();
```

而是 UI Plugin 注册自己的：

```text
requirements
factory
layout metadata
priority
lifecycle
```

然后由 Composer 决定当前安装哪些。

---

## 我觉得现在可以把 Camera 和 HUD 统一成一个更大的模式

我们之前觉得它们只是“很像”，现在其实可以更明确：

```text
                     Local Player Session
                              │
               ┌──────────────┴──────────────┐
               ▼                             ▼
          Controller                    Presentation
               │                             │
               ▼                    ┌────────┴────────┐
       Control Surface              ▼                 ▼
                                Camera Runtime     HUD Runtime
                                     │                 │
                                     ▼                 ▼
                              CameraProfile      PresentationContext
                              + Modifiers        + Widget Plugins
                                     │                 │
                                     ▼                 ▼
                               CameraState        HUD RenderState
```

而 Object 位于中间：

```text
                        OBJECT
                       /      \
                      /        \
                     ▼          ▼
           Control Surface    Presentation Surface
                  ▲                 │
                  │                 ▼
             Controller        Camera / HUD
```

**左边是 command path，右边是 observation path。**

数据同步则采用一个很简单、也很成熟的规则：

> **绑定时 snapshot，运行时 event/reactive update，解绑时 unsubscribe。**

这样 HUD 不需要每帧遍历 Object，也不需要 Object 知道 HUD 存在；切武器/换座位时只需要重新构造或 rebind `PresentationContext`。

我觉得这一步之后，我们其实已经可以开始把这些概念压缩成真正的 framework API 了，因为现在 `Runtime / RuntimeView / ControlSurface / PresentationSurface / Session / Plugin / ReactiveState` 各自解决的问题已经比较明确，再继续只做概念讨论反而容易产生同义概念。

## 整理说明

1. 本篇保留了 HUD Renderer、HUD Runtime、HUD Slot、HUD Composer 和 Presentation Surface 的完整推导。
2. 关键拆分是：HUD Renderer 是 Session 级长期存在的 Runtime；当前武器、载具或控制对象提供的 Presentation Surface 是动态 Context；HUD Plugin 则把二者连接起来。
3. HUD 不应直接成为物品的子对象，也不应复制物品状态；它应该观察由物品 Runtime 所拥有的状态或 Surface。
4. 对象切换时重新生成 Presentation Context，普通数值更新则从已缓存的 Capability 或 Observable State 读取。
