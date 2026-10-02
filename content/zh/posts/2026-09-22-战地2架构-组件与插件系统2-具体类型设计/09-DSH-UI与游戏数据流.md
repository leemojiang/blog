---
title: "DSH UI 与游戏数据流"
date: 2026-09-22
weight: 9
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

本篇继续整理 DSH UI 设计对游戏 HUD 的启发，重点讨论对象事件、连续状态、中央 HUD Surface 和输入/控制数据流之间的区别。

## Q：游戏 UI 是否应该区分对象事件、连续状态和中央 HUD Surface？DSH UI 的数据流哪些可以借鉴？

### 原始提问

> 是的 我感觉 DSH 的UI 设计非常有启发性 UI plugin 只解决能力依赖这个问题 数据走一条新的路线 订阅 或者 stream 并且也是向物体本身进行订阅
>
>
>
> 我觉得有两个需要注意的点,不知道我的理解是否准确:
>
> 1. DSH 它的ui/控制流相对简单一点 不需要动态的切换加载大量物体 但游戏里面对应的gameobject 可能需要频繁变动 以及数据连续变动 用pull 可能比直接event push 更好一点 (感觉游戏里面会有很多事件系统 而不是一个全局事件系统 我感觉这里需要特别注意 比如每个weapon 都有fire event 即使不是玩家选中的 这些event也可以服务fx plugin 但ui 最好和他们分开 变成中央的数据组织surface 否则感觉很容易出问题 (比如绑定了错误的信号
> 2. 同样DSH 的ui 其实还复杂一点 需要发布信息 (比如用户输入) 我看它走的是全局的host event 这点游戏的ui 不需要 但是可以在输入系统里面借鉴

## A：

对，我认为你的两个判断都基本准确，而且第 1 点尤其重要。它实际上揭示了 **DSH UI 和游戏 HUD 最不应该直接照搬的地方**：DSH 更偏“事件/消息驱动应用”，游戏本身则已经有一个稳定的 frame/tick loop，因此很多连续状态没必要强行消息化。

我甚至会把我们现在的架构进一步收敛成：

```text
Object 内部交互     → Local Event / Capability
玩家控制            → Control Surface
玩家观察            → Presentation Surface
HUD 连续数据        → Pull
瞬时 HUD 反馈       → Session Event
```

这样不同问题走不同通道。

---

## 1. 我非常赞成：不要搞一个“大一统 Global Event Bus”

假设场景里有：

```text
Player
├── Rifle A
├── Rifle B
└── Pistol

Enemy 1
└── Rifle C

Enemy 2
└── Rifle D
```

每个 Weapon 都会：

```text
WeaponFired
Reloaded
AmmoConsumed
Overheated
```

如果全部扔进：

```text
GlobalEventBus<WeaponFired>
```

HUD 就不得不：

```csharp
OnWeaponFired(e)
{
    if (e.weapon == currentWeapon)
        ...
}
```

然后 FX：

```csharp
if (e.weapon == myWeapon)
```

Audio：

```csharp
if (...)
```

这其实已经开始出现明显 architecture smell：

> **订阅者收到了一堆本来就不属于自己的消息，然后靠 ID/filter 恢复上下文。**

你说的“容易绑定错误的信号”正是这个问题。

---

## 2. Weapon Event 最自然应该属于 Weapon Runtime / Scope

例如：

```text
Weapon A Runtime
├── ITriggerControl
├── IAmmoStatus
│
└── Events
    ├── WeaponFired
    ├── ReloadStarted
    └── ReloadCompleted
```

Weapon B：

```text
Weapon B Runtime
└── Events
    ├── WeaponFired
    └── ...
```

它们是完全不同的 event source。

于是 Weapon A 自己的 FX：

```text
Weapon A
│
├── Fire
│     └── WeaponFired
│
├── MuzzleFX
│      ▲
│      │ subscribe
│
└── Audio
       ▲
       │ subscribe
```

非常自然。

这里 `WeaponFired` 根本不需要：

```text
weaponId = xxxx
```

因为 **event source 本身已经表达 identity**。

这其实很漂亮：

> Event scope 本身就是上下文。

---

# 3. 这和 Presentation Surface 的价值正好形成对比

HUD 不应该到处订阅：

```text
Weapon A Events
Weapon B Events
Tank Events
Player Events
...
```

HUD 只需要：

```text
Current Presentation Surface
```

比如：

```text
PresentationSurface
├── IHealthStatus  → Player
├── IAmmoStatus    → CurrentWeapon
├── IWeaponStatus  → CurrentWeapon
├── IAimStatus     → CurrentAim
└── IZoomStatus    → CurrentOptics
```

切武器：

```text
                 AK47
                  │
                  ▼
HUD ← PresentationSurface

        weapon switch

                 RPG
                  │
                  ▼
HUD ← PresentationSurface
```

所以 HUD 根本不需要问：

> “这个 AmmoChanged 是不是我当前 Weapon 的？”

它手上的：

```csharp
IAmmoStatus ammo;
```

**天然就是当前 weapon 的。**

这就是 Surface 除了 interface segregation 之外另一个非常大的价值：

> **Surface 不只是隐藏能力，它同时建立了 Context。**

---

# 4. 所以我现在甚至更赞成 HUD 连续数据直接 Pull

游戏里：

```csharp
void UpdateHUD()
{
    ammoText.text = presentation.Ammo.Current.ToString();
    healthBar.value = presentation.Health.Current;
    speedText.text = presentation.Speed.Value.ToString();
}
```

这里并没有多少问题。

因为：

```text
PresentationSurface
        ↓
cached interface
        ↓
direct property read
```

并不是：

```text
每帧 FindObject
↓
GetComponent
↓
Reflection
↓
Runtime Resolve
```

所以性能和复杂度完全不是一个量级。

而且很多数据本来就是连续的：

```text
speed
altitude
aim offset
reload progress
heat
RPM
zoom interpolation
health
ammo
```

如果全变成：

```text
SpeedChanged
AltitudeChanged
AimChanged
ReloadProgressChanged
HeatChanged
...
```

反而很荒谬。

---

# 5. Event 仍然非常重要，但它更应该是 scoped event

我现在会把事件分成至少三个 scope：

```text
Object / Runtime Events
───────────────────────
WeaponFired
Reloaded
EngineStarted

Session Events
───────────────────────
ControlledObjectChanged
SelectedWeaponChanged
SeatChanged
PresentationChanged

World / Game Events
───────────────────────
PlayerKilled
ObjectiveCaptured
RoundEnded
```

它们生命周期和受众完全不同。

例如：

### WeaponFired

```text
Weapon Runtime Event
```

主要给：

```text
MuzzleFX
Audio
Recoil
ShellEjection
```

### SelectedWeaponChanged

```text
Player Session Event
```

给：

```text
HUD
Camera
Input binding
```

### ObjectiveCaptured

```text
Game/World Event
```

给：

```text
HUD
Announcer
Scoreboard
GameMode
```

这比：

```text
GlobalEventBus
```

全部塞进去健康很多。

---

# 6. Presentation Surface 本身甚至可以成为一个“中央 Context Projection”

你说的：

> 中央的数据组织 surface

我觉得这个描述非常准确。

例如 PlayerSession 当前：

```text
Controlled:
    Soldier

Equipped:
    AK47

Seat:
    none

Aim:
    SoldierAim

Optics:
    RedDot
```

Session Builder/Composer 根据这些关系生成：

```text
PresentationSurface
│
├── IHealthStatus ────── Soldier
├── IAmmoStatus ──────── AK47.Magazine
├── IWeaponStatus ────── AK47
├── IAimStatus ───────── SoldierAim
└── IZoomStatus ──────── RedDot
```

HUD 不理解 Object Tree。

Camera 也不理解 Object Tree。

它们只看到：

```text
Current Presentation Context
```

这个边界非常有价值。

---

# 7. 这样可以把“结构变化”和“数据变化”彻底分开

这是我觉得现在最值得固定成原则的一点：

### 数据变化

```text
Ammo 30 → 29
Health 100 → 83
Speed 20 → 25
Aim yaw 10° → 11°
```

**不重建 Surface。**

直接：

```text
HUD → cached capability → Pull
```

### 结构变化

```text
AK47 → RPG
Soldier → Tank
DriverSeat → GunnerSeat
IronSight → Scope
```

才：

```text
Session Context Changed
          ↓
Presentation Composer
          ↓
Rebind PresentationSurface
          ↓
HUD / Camera receive new context
```

因此：

```text
High Frequency
──────────────
State → Pull

Low Frequency
──────────────
Structure → Rebind
```

这特别符合游戏。

---

# 8. 你的第二点也对：DSH 的 Host Event 更像我们的 Input/Command 路径

DSH UI 不只是 presentation，它还有用户操作：

```text
UI
 ↓
click
 ↓
host event
 ↓
plugin/service
```

但游戏 HUD 大部分时候其实不承担角色控制。

我们已经有：

```text
Unity Input
     ↓
Input Runtime
     ↓
Player Controller
     ↓
Control Surface
     ↓
Object
```

所以没必要让：

```text
HUD → Object
```

再建第二条控制通道。

---

# 9. 但游戏 UI 里确实有交互型 UI

例如：

```text
Inventory
Map
Weapon Selection
Vehicle Seat Menu
Respawn Screen
Settings
```

这时候 DSH 的模式就又很值得借。

例如 Inventory UI：

```text
Inventory UI
      │
      │ user clicked "Equip"
      ▼
UI Intent
      │
      ▼
Player Session / Controller
      │
      ▼
IEquipmentControl
      │
      ▼
Inventory Runtime
```

而不是：

```text
InventoryButton
      ↓
inventoryComponent.Equip(...)
```

可以定义：

```text
UI
 ↓
Intent / Command
 ↓
Session Controller
 ↓
Control Surface
```

这和 DSH：

```text
UI
 ↓
Host Event
 ↓
Plugin
```

概念上很接近。

---

# 10. Input System 甚至可以把 DSH Host Event 思想吸收得更多一点

我们现在 Player Input 可能是：

```text
Keyboard / Mouse
Gamepad
Touch
UI Button
Network?
AI?
```

都产生某种 **Intent**：

```text
FirePressed
FireReleased
AimDelta
MoveAxis
Reload
SelectWeapon
```

然后：

```text
              Keyboard
                 │
              Gamepad
                 │
              UI Button
                 │
                 ▼
            Input Runtime
                 │
              Intent
                 │
                 ▼
         Player Controller
                 │
                 ▼
          Control Surface
```

这和 DSH Host Event 最大的共同点不是“用了 event bus”，而是：

> **输入源不知道最终执行者是谁，只表达用户 Intent。**

这一点非常值得借。

---

## 我觉得现在我们已经自然形成了四条完全不同的数据/控制通道

这比试图用 Runtime/Event/Interface 一招解决所有问题清晰很多：

```text
                  GAME OBJECT
                /      |       \
               /       |        \
              ▼        ▼         ▼

        Capability    State    Local Event
             │          │          │
             │          │          ├── FX
             │          │          ├── Audio
             │          │          └── local reactions
             │          │
      ┌──────┘          │
      ▼                 ▼
Control Surface   Presentation Surface
      ▲                 │
      │                 │ Pull
      │                 ▼
 Controller          HUD / Camera
      ▲
      │
   Intent
      ▲
      │
Input Runtime
```

再在外围：

```text
Session Events
    └── WeaponChanged / SeatChanged / ControlledObjectChanged
          ↓
    rebuild/rebind surfaces

World Events
    └── Kill / Objective / Round
          ↓
    global game systems / HUD
```

我觉得这比我们最开始那个“大 Runtime + EventHub”模型成熟很多。

尤其有三个原则现在已经很明确：

**Surface 解决的是“我现在和谁交互”。**

**Pull State 解决的是“它现在是什么状态”。**

**Scoped Event 解决的是“这个上下文里刚刚发生了什么”。**

而 DSH 的 UI plugin 最值得我们拿过来的，不是它具体用 event 还是 observable，而是那个更高层的思想：**UI Plugin 只是 glue——声明自己需要哪些 UI capability 和 domain capability，由 host/runtime 负责把它们组合起来；UI 本身不拥有业务对象。**

## 整理说明

1. 本篇保留关于 DSH UI 更偏事件/消息驱动、游戏连续状态更适合 Pull、局部对象事件与中央 UI 数据组织需要隔离的完整讨论。
2. 每个 Weapon 可以有自己的 fire event，这类事件可以服务 FX；但 HUD 不应盲目订阅所有对象事件，而应通过 Player Session/HUD Surface 读取当前上下文。
3. 游戏 UI 可以借鉴 DSH 的输入和 Host Event 思路，但不应照搬全局事件总线；输入仍应经过 Input/Controller 层转换为明确的控制命令。
