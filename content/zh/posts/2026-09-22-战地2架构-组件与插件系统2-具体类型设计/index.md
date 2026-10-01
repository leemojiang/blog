---
title: "战地 2 架构、组件与插件系统 2：具体类型设计"
date: 2026-09-22
draft: false
math: true
tags:
  - 战地 2
  - 游戏架构
  - 组件系统
  - 插件架构
  - UI
categories:
  - 软件架构
description: >-
  按讨论顺序整理具体类型设计：从输入、Control Surface、Camera 与 HUD，延伸到能力接口、事件流、UI 数据流和 Surface 生命周期。
---

本组笔记严格按照原始讨论的时间顺序，逐个整理后续 11 个问题。前半部分讨论 Input、Control Surface、Camera、HUD、Ammo 和 Event；最后一组连续整理 DSH UI、游戏 UI 数据流以及 Surface 的组装、数据和 Present。

## 输入、武器控制与其他特殊类型


本篇是新增讨论中最早的问题，先讨论 Input、玩家控制、AI 控制、Weapon、Camera 和 UI 这些容易跨越多个架构边界的特殊类型。

### Q：输入、武器控制、Camera 和 UI 这些特殊类型应该如何按照 Runtime/Surface/Plugin 思路设计？

#### 原始提问

> 在这个思路下 游戏内几种比较特殊的类型如何设计?
>
> 1. 首先是输入 比如一个weapon需要可以配置输入 传统的方法是做成input comp 这样比较符合面向对象的思路 把input comp作为weapon的runtime 但这样的问题是输入逻辑和对象内部逻辑耦合的有点严重, 里面可能会有大量的 if (firecomp.firebuttondown) 并且最麻烦的是如果涉及多人游戏 以及 AI 控制 就会更复杂. 似乎可以根据不同的控制对象动态修改comp 比如ai 控制的时候使用它自己的firecomp 但是游戏里对象很多 comp也很多 这么做似乎不是很容易
> 2. 与之相对的做法似乎可以把 weapon 当作 input 的 runtime, 即weapon提供能力,谁持有weapon调用weapon的接口. 这样的设计思路似乎对 AI 部分很有帮助.  AI 和玩家都能比较顺利的控制武器. 但是又会有几个问题: 
>
> 1\. 从面向对象的角度有点怪,开火是武器内部逻辑,但是现在变成了外部控制,这样一些内部属性,比如这个武器开火按键的配置,怎么传递? 
>
> 2.这样会引入额外抽象,比如原本的fire就是一个判断语句,现在需要设计一套接口,会引入不小的复杂度,尤其是实际控制可能有比较细腻的控制方法 比如 button down button hold button up 这样接口可能变得非常多,还需要内部状态的维护. 
>
> 3.虽然都是武器控制,但是ai和玩家需要的接口可能不一样,比如玩家可能可以直接修改武器的yaw pitch 但是ai 则需要 aim at. 这就需要武器考虑自己挂载的平台. 比如武器挂载在炮塔上 和 挂载在人物身上 可能提供的comp 有不一样 也就是说 可能会有多种runtime的组合? 但这样总之一个武器身上会引入额外的复杂度 (这点很好奇战地2怎么做的 它就是genericfirearm 似乎手持武器和固定武器都是这个类型 然后既可以挂在solider身上 也可以挂在 rotation bundle 控制的炮塔上面 rotation bundle 之类的也可以接受输入 进行旋转控制
>
> 3. 另一个特殊的类型是摄像机 既需要有一些配置 又是一个单例 只有玩家视角才有摄像机
> 4. 还有游戏里常用的UI 系统 可能需要根据不同武器 配置进行切换 他该如何用plugin 思路设计 (DSH 里面ui 部分似乎是plugin 它们是怎么做的呢?

### A：

对，而且我觉得你列出的 **Input / AI / Camera / UI** 恰好暴露出一个非常重要的边界：

> 前面我们讨论的 Runtime/Component 模型很适合描述“对象是什么、对象能做什么”，但不应该继续承担“谁正在控制它、谁正在观察它、当前玩家正在使用什么”的职责。

我会在 Object Runtime 之外再引入一个很薄的 **Controller / Session 层**。这样很多看起来特殊的问题会一起变简单。

---

## 1. Input 最好不要成为 Weapon Component

我会明确避免：

```text
Weapon Runtime
├── Fire
├── Ammo
├── Recoil
├── Physics
└── Input     ← 不太好
```

因为 Input 不是 Weapon 的固有属性。

一把枪放在地上：

```text
Weapon
├── Fire
├── Ammo
└── Aim
```

仍然是一把完整的枪。

而：

```text
Player
AI
Network
Replay
```

只是不同的**控制来源**。

所以更合理的是：

```text
             Player Input
                  │
                  ▼
             Controller
                  │
                  ▼
Weapon ────── IFireControl
       └───── IAimControl
```

AI：

```text
AI Controller
      │
      ▼
 IFireControl
      ▲
      │
    Weapon
```

Network：

```text
Network Controller
        │
        ▼
   IFireControl
        ▲
        │
      Weapon
```

这样 Weapon 完全不需要切换 InputComp / AIComp。

---

## 2. 但“外部控制”不等于“开火逻辑在外部”

这是你第一个疑虑里最关键的区别。

我们不是：

```text
PlayerController:
    if mouseDown:
        ammo--
        spawnProjectile()
        recoil()
```

而是：

```text
PlayerController
       │
       │ "Trigger Pressed"
       ▼
  IFireControl
       │
       ▼
     Weapon
       │
       ├── check cooldown
       ├── check ammo
       ├── chamber
       ├── fire mode
       ├── spawn projectile
       ├── recoil
       └── WeaponFired event
```

所以：

> **控制意图来自外部，行为规则仍然属于 Weapon。**

这是非常标准而且非常漂亮的 OO。

类比汽车：

```text
Driver → accelerator pedal → Engine
```

司机告诉发动机：

> 我要加速。

但发动机自己决定：

```text
fuel injection
RPM
gear
torque
temperature
```

而不是 Driver 做这些事情。

---

## 3. `button down / hold / up` 不应该污染 Weapon

这里我甚至不会让 Weapon 知道 Mouse Button。

因为：

```text
MouseDown
MouseHold
MouseUp
```

是 **输入设备语义**。

Weapon 真正关心的是：

```text
TriggerPressed
TriggerHeld
TriggerReleased
```

所以有一个非常薄的映射：

```text
Unity Input
    │
    │ Mouse0 Down
    ▼
PlayerController
    │
    │ Trigger.Press
    ▼
Weapon
```

接口甚至可以很简单：

```csharp
public interface ITriggerControl
{
    void Press();
    void Release();
}
```

你甚至不一定需要：

```csharp
Hold()
```

因为 Weapon 自己维护状态：

```text
Press
 ↓
triggerHeld = true

Update
 ↓
if triggerHeld && canFire
    fire()

Release
 ↓
triggerHeld = false
```

于是：

```text
Semi Auto:

Press → Fire once


Full Auto:

Press
  ↓
held = true
  ↓
Fire → cooldown → Fire → cooldown → Fire

Release
  ↓
held = false
```

这实际上比：

```csharp
if (Input.GetButtonDown(...))
if (Input.GetButton(...))
if (Input.GetButtonUp(...))
```

更干净。

---

## 4. 武器按键配置也不属于 Weapon

例如：

```text
AK47.fireButton = Mouse0
```

我认为这其实是错误的数据归属。

AK47 并不知道鼠标是什么。

真正应该是：

```text
Player Control Profile

FirePrimary → Mouse0
Aim         → Mouse1
Reload      → R
NextWeapon  → MouseWheel
```

Weapon 只暴露：

```text
ITriggerControl
IReloadControl
```

所以：

```text
Mouse0
   ↓
"FirePrimary"
   ↓
Controller
   ↓
ITriggerControl.Press()
```

玩家换键：

```text
Space
 ↓
FirePrimary
 ↓
Weapon
```

Weapon definition 完全不用变。

如果某种特殊武器：

```text
LaserDesignator
```

需要：

```text
Primary
Secondary
Zoom
```

Controller 根据 capability 判断当前对象有哪些 action。

---

## 5. AI 和 Player 接口不同，其实是正常的

你这里提出了一个非常好的例子：

Player：

```text
yaw += mouseDeltaX
pitch += mouseDeltaY
```

AI：

```text
AimAt(targetPosition)
```

我反而**不会强迫 AI 和 Player 使用完全相同的接口**。

因为它们工作在不同 abstraction level。

可以是：

```text
               Aim System
             /            \
            /              \
           ▼                ▼
    IAimDirectControl   IAimTargetControl

    SetYawPitch()          AimAt()
           ▲                ▲
           │                │
        Player              AI
```

内部：

```text
IAimTargetControl.AimAt(position)
             │
             ▼
        AimController
             │
             ▼
     desired yaw/pitch
             │
             ▼
       RotationMotor
```

所以 AI：

```text
AimAt(enemy)
```

而玩家：

```text
AddYaw(mouseX)
AddPitch(mouseY)
```

最终都落到：

```text
RotationMotor
```

---

## 6. 这正好解决 BF2 的 GenericFireArm + RotationBundle 问题

你提到这个特别有意思。

我没有重新核查 BF2 Core 的具体内部调用，所以不想把下面说成 BF2 确切实现；但从我们现在的架构来看，它可以非常漂亮地解释为什么 `GenericFireArm` 可以同时出现在 Soldier 和 turret 体系里：

```text
Soldier
│
└── Weapon
    └── GenericFireArm
```

和：

```text
Tank
│
└── Turret
    ├── RotationBundle
    │
    └── Cannon
        └── GenericFireArm
```

关键在于：

> **Fire 和 Aim 根本不需要是同一个 capability。**

例如：

```text
GenericFireArm

provides:
    ITriggerControl

requires:
    IAmmoSource
    IProjectileSpawner
```

而 Turret：

```text
RotationBundle

provides:
    IAimDirectControl
    IAimTargetControl
```

于是坦克 Controller 看到：

```text
Tank Control Surface
├── IMovementControl
├── IAimControl
└── ITriggerControl
```

注意这个非常漂亮的变化：

**Controller 根本不关心这些能力来自对象树哪个节点。**

```text
Tank
├── Movement ────────────── IMovementControl
│
└── Turret
    ├── RotationBundle ──── IAimControl
    │
    └── Cannon
        └── Fire ────────── ITriggerControl
```

Controller 得到一个经过聚合的：

```text
VehicleControlView
```

---

## 7. 这里出现了一个新的重要概念：Control Surface

我觉得它值得成为我们 framework 中少数几个一级概念之一。

以前：

```text
Runtime = 这个 Object 有什么
```

现在：

```text
Control Surface = 外部允许怎样控制这个 Object
```

例如 Tank 内部可能有 30 个 capability：

```text
Tank Runtime
├── Physics
├── Damage
├── Fuel
├── Engine
├── Transmission
├── Suspension
├── Ammo
├── Turret
├── Weapon
├── Smoke
├── ...
```

但对玩家：

```text
PlayerControlSurface
├── IMovementControl
├── IAimDirectControl
├── ITriggerControl
├── IReloadControl
└── ISeatControl
```

AI：

```text
AIControlSurface
├── INavigationControl
├── IAimTargetControl
├── ITriggerControl
└── IWeaponSelection
```

这就又进行了一次非常漂亮的**剪裁**。

---

## 8. 而且 Control Surface 可以跨 Object Tree 聚合

这个特别适合游戏。

例如：

```text
Tank
├── DriveTrain
│   └── IMovementControl
│
├── Turret
│   └── IAimControl
│
└── Cannon
    └── ITriggerControl
```

Seat：

```text
GunnerSeat
     │
     ▼
ControlSurface
├── Aim   → Tank/Turret
├── Fire  → Tank/Turret/Cannon
└── Zoom  → Tank/Turret/Optics
```

DriverSeat：

```text
DriverSeat
     │
     ▼
ControlSurface
├── Move   → Tank/DriveTrain
├── Brake  → Tank/DriveTrain
└── Camera → Tank/DriverCameraMount
```

这比：

```text
Player controls Tank GameObject
```

准确得多。

实际上：

> **Player 控制的是 Seat 暴露出来的一组 capability。**

这个模型特别适合 BF2。

---

## 9. Multiplayer 也因此自然解决

客户端：

```text
Unity Input
     ↓
PlayerController
     ↓
Control Intent
     ↓
Network
```

Server：

```text
Network
   ↓
RemoteController
   ↓
ControlSurface
   ↓
Weapon / Movement / Aim
```

AI：

```text
AI
 ↓
AIController
 ↓
ControlSurface
```

Replay：

```text
Replay
 ↓
ReplayController
 ↓
ControlSurface
```

所以：

```text
                  ControlSurface
                       ▲
           ┌───────────┼────────────┐
           │           │            │
        Player        AI          Network
```

对象完全不需要：

```csharp
if (isAI)
if (isLocalPlayer)
if (isRemotePlayer)
```

这点价值非常大。

---

## 10. Camera 更应该放到 Session Runtime，而不是 Object Runtime

Camera 是你举的另一个非常典型的“看起来像 Component，但其实不属于 Object”的东西。

我会区分：

```text
World Runtime
Object Runtime
Player Session Runtime
```

Camera 属于：

```text
PlayerSession
```

例如：

```text
Player Session Runtime
├── Input
├── Camera
├── HUD
├── Audio Listener
└── Controlled Object
```

而 Tank 只提供：

```text
Tank
├── DriverCameraMount
├── GunnerCameraMount
└── CommanderCameraMount
```

注意：

Tank **没有 Camera**。

Tank 只有：

> Camera 可以挂在哪里、有哪些视角配置。

真正 Camera：

```text
PlayerSession.Camera
```

根据：

```text
current seat
current mode
current camera mount
```

决定位置。

---

## 11. 这样多人游戏的 Camera 问题直接消失

例如 server：

```text
Tank
├── CameraMount
└── ...
```

但：

```text
Dedicated Server

PlayerSession Camera = none
```

本地玩家：

```text
Local Player Session
└── Camera
```

Spectator：

```text
Spectator Session
└── Camera
```

Split screen 甚至可以：

```text
Session A → Camera A
Session B → Camera B
```

所以严格来说 Camera 甚至不应该是“全局 Singleton”。

它应该是：

> **per local presentation session singleton**

单玩家情况下碰巧只有一个而已。

这个 distinction 以后很有价值。

---

## 12. UI 和 Camera 是同一类东西

HUD 也不是 Tank/Weapon 的一部分。

不要：

```text
Weapon
└── WeaponUIComponent
```

而是：

```text
PlayerSession
└── HUD
```

HUD **观察当前 Control/Presentation Context**。

例如：

```text
PlayerSession
     │
     ├── controlled = Tank
     │
     ├── selectedWeapon = Cannon
     │
     └── seat = Gunner
             │
             ▼
           HUD
```

Cannon 提供：

```text
IAmmoStatus
IWeaponStatus
IAimStatus
```

HUD 消费：

```text
IAmmoStatus
    CurrentAmmo
    Capacity

IWeaponStatus
    Reloading
    Cooldown

IAimStatus
    Crosshair
    Target
```

注意这里又回到了我们上一轮说的 ISP：

Weapon 的 UI 只能拿到：

```text
IAmmoStatus
```

而不是：

```text
IAmmoSource
```

所以 UI 能读：

```text
Ammo = 12
```

却不能：

```text
ConsumeAmmo(10)
```

---

## 13. UI Plugin 则可以根据 Capability 自动激活

这就非常 DSH/plugin 了。

比如：

```text
AmmoHUDPlugin

requires:
    IAmmoStatus
```

如果当前 Presentation Context 有：

```text
IAmmoStatus
```

就安装：

```text
AmmoWidget
```

如果当前控制对象还有：

```text
IVehicleHealthStatus
```

安装：

```text
VehicleHealthWidget
```

如果有：

```text
ILockOnStatus
```

安装：

```text
LockOnWidget
```

于是切换：

```text
Soldier
   ↓
HUD
├── Health
├── Ammo
└── Crosshair
```

进入坦克：

```text
Tank Gunner
   ↓
HUD
├── VehicleHealth
├── CannonAmmo
├── Crosshair
└── ReloadIndicator
```

上飞机：

```text
Aircraft
   ↓
HUD
├── Airspeed
├── Altitude
├── Weapon
├── Radar
└── LockOn
```

不是：

```csharp
if (vehicle == Tank)
    ShowTankHUD();

if (vehicle == Plane)
    ShowPlaneHUD();
```

而是：

```text
Capability
    ↓
UI Plugin Matching
    ↓
Compose HUD
```

这个设计和我们整个 Runtime Plugin 模型非常一致。

---

## 14. 但这里还要区分 Control Surface 和 Presentation Surface

这是我觉得特别值得引入的一对概念：

```text
Object Runtime
      │
      ├── Control Surface
      │       外界能对我做什么
      │
      └── Presentation Surface
              外界能从我观察什么
```

例如 Weapon：

```text
Control Surface
├── ITriggerControl
├── IReloadControl
└── IAimControl

Presentation Surface
├── IAmmoStatus
├── IWeaponStatus
├── IAimStatus
└── IWeaponIcon
```

于是：

```text
                 Weapon
               /        \
              /          \
             ▼            ▼
      Control Surface   Presentation Surface
             ▲            ▲
             │            │
      ┌──────┼─────┐      │
      │      │     │      │
    Player   AI  Network   UI
```

这比所有人都直接拿 Weapon Runtime 干净很多。

---

## 15. 我现在甚至会把 Runtime 分成四个层级

我们之前只有：

```text
World
 ↓
Object Runtime
```

现在我觉得更完整的是：

```text
World Runtime
│
│  global services
│
├───────────────┐
│               │
▼               ▼
Object Runtime       Session Runtime
│                    │
│ object semantics   │ local user context
│                    │
│                    ├── Input
│                    ├── Camera
│                    ├── HUD
│                    └── Controller
│
├── Internal Capabilities
│
├── Control Surface ◄──────── Controller
│
└── Presentation Surface ───► HUD / Camera
```

这一下你列出来的四个“特殊情况”其实全都找到了位置。

---

### 回头看 Weapon，就非常清楚了

```text
                        World
                          │
                    ProjectileService
                          │
                          ▼
                    Weapon Runtime
                   ┌──────┼──────┐
                   │      │      │
                 Ammo    Fire   Recoil
                   │
          ┌────────┴──────────┐
          ▼                   ▼
   Control Surface      Presentation Surface
          │                   │
    ITriggerControl       IAmmoStatus
    IReloadControl        IWeaponStatus
          ▲                   │
          │                   ▼
      Controller              HUD
       /   |   \
      /    |    \
 Player   AI   Network
```

而 Aim 甚至可能根本不属于 Weapon：

```text
Tank
└── Turret
    ├── RotationBundle ─── IAimControl
    │
    └── Cannon
        └── Weapon ─────── ITriggerControl
```

Seat/Controller 将两者组合：

```text
GunnerControlSurface
├── Aim  → RotationBundle
└── Fire → Cannon
```

**这点尤其重要：不要为了让 Weapon“接口完整”，硬把 Aim 塞进 Weapon。能力属于真正拥有它的 Object，Control Surface 可以跨 Object Tree 聚合它们。**

这样反而非常接近你观察到 BF2 的 `GenericFireArm + RotationBundle + ObjectTemplate tree` 为什么能表现出很强的通用性。

所以我觉得我们现在又找到了一个很关键的抽象边界：

> **Object Runtime 描述“我是什么”；Control Surface 描述“你可以怎么操作我”；Presentation Surface 描述“你可以从我这里观察什么”；Session Runtime 决定“当前是谁在操作和观察”。**

这四个东西分开以后，Input、AI、Multiplayer、Camera、HUD 原本互相缠绕的问题，会一下子变成四个相对独立的问题。

### 整理说明

1. 本篇保留原始问题中关于 Input Component、Weapon 作为 Input Runtime、button down/hold/up、玩家与 AI 接口差异、GenericFirearm、Rotation Bundle、Camera 单例和 UI Plugin 的完整回答。
2. 核心边界是：Object Runtime 描述对象是什么、对象能做什么；Controller/Session 描述谁正在控制它；Presentation/HUD Runtime 描述谁正在观察它。
3. 玩家输入和 AI 决策都应转换为统一的 Control Command 或 Control Surface 调用，Weapon 不应直接读取 Unity Input，也不应知道调用者是玩家还是 AI。
4. 玩家和 AI 可以获得不同的 Surface View：玩家可能需要低层 yaw/pitch 控制，AI 可能需要 aim-at 目标接口；差异应由 Controller/Adapter 组合解决，而不是让 Weapon 内部堆积大量 if。

## Control Surface 与 Aim 组合


本篇继续讨论控制层如何组合，重点是把 Aim 从 Weapon 的原生能力中拆出，并让 Soldier、Turret、Vehicle 等不同控制来源共享统一接口。

### Q：是否需要多种 Aim Runtime 或组合 Runtime？Control Surface 应该如何初始化和切换？

#### 原始提问

> 确实 我觉得Control Surface 也要贯彻组合的思想,功能复杂了就拆开 Aim不属于Weapon的原生能力 干脆拆出来. 不过这么说 我们有没有必要引入 多种runtime组合 比如 solider的aim 控制操作 (旋转人的身体 视角俯仰 和 载具上面的aim 炮塔俯仰 由最近的两层rotation bundle决定 显然需要不同的runtime 初始化. 我们是不是应该实现不同的aim comp 然后使用统一接口? 
>
> 或者有没有必要引入一下组合runtime 比如一个comp 可以依赖runtime a 也可以依赖runtime b (减少类型的爆炸和复杂
>
> 还有一个问题这种control surface 怎么初始化 感觉它是一个动态的 那么是预先缓存(类似runtime  的处理方式 还是游戏进行中控制改换的时候 getcomponent 这种动态获取比较好?

### A：

对，我会选择 **“不同 Aim 实现 + 统一的 capability contract”**，但不会进一步演化成很多 `SoldierRuntime / TankRuntime / TurretRuntime / WeaponRuntime` 类型。你最后提到的“组合 Runtime”其实可以解决这个类型爆炸，不过最好把它理解成 **多个 scope/source 的组合视图**，而不是新的 Runtime class。

核心可以收敛成：

```text
Object Runtime        = 能力实际存在在哪里
Runtime View          = 某个模块允许看到哪些能力
Control Surface       = Controller 可以操作哪些能力
```

这三个东西组合起来就够了。

---

### 1. Soldier Aim 和 Turret Aim：不同实现，统一接口

例如先定义非常小的：

```csharp
public interface IAimControl
{
    void AddLookInput(Vector2 delta);
}
```

Soldier：

```text
SoldierAim
   │
   ├── BodyYaw
   └── ViewPitch
         │
         ▼
     IAimControl
```

Tank：

```text
TurretAim
   │
   ├── TurretYaw RotationBundle
   └── GunPitch RotationBundle
         │
         ▼
     IAimControl
```

外面的 Player Controller 完全不知道区别：

```csharp
aim.AddLookInput(mouseDelta);
```

所以具体差异被关在 provider 里面。

AI 又可能不适合这个接口，因此另外提供：

```csharp
public interface IAimTargetControl
{
    void AimAt(Vector3 worldPosition);
}
```

同一个 `TurretAim` 完全可以同时实现：

```text
TurretAim
├── IAimControl          ← Player
└── IAimTargetControl    ← AI
```

这正好又是 ISP：不是制造两个 Turret，而是一个实现暴露两个不同的“面”。

---

## 2. SoldierAim / TurretAim 内部确实需要不同 Runtime View

比如：

```text
SoldierAim requires
├── BodyRotation
├── ViewRotation
└── Pose
```

而：

```text
TurretAim requires
├── YawRotation
├── PitchRotation
└── Stabilizer(optional)
```

这没有问题。

关键是 **Aim 的使用者不应该知道这些区别**：

```text
                       IAimControl
                         ▲      ▲
                         │      │
               SoldierAim      TurretAim
                  │                │
          SoldierRuntime      Tank Object Tree
```

复杂性停在 implementation boundary。

---

## 3. 你提出的“Comp 可以依赖 Runtime A + Runtime B”非常值得做

但我不会真的写：

```csharp
class AimComponent
{
    RuntimeA a;
    RuntimeB b;
}
```

因为这样过几年 Runtime 类型还是会爆炸。

更好的抽象是：

> **一个 Runtime View 可以从多个 Capability Source 组合出来。**

例如 TurretAim：

```text
Turret Object Runtime
    └── IYawRotation
              \
               \
                → AimContext → TurretAim
               /
Gun Object Runtime
    └── IPitchRotation
```

甚至再加：

```text
Vehicle Root Runtime
    └── IPhysicsBody
```

最终：

```text
AimContext
├── IYawRotation     ← Turret local
├── IPitchRotation   ← Gun local
└── IPhysicsBody     ← Vehicle root
```

Aim 不需要知道：

```text
Runtime A
Runtime B
Runtime C
```

它只知道自己的 Context。

---

## 4. 这样 Object Tree 和 Dependency Graph 又成功分开了

比如：

```text
Tank
│
├── Physics
│
└── Turret
    │
    ├── YawRotation
    │
    └── Gun
        ├── PitchRotation
        └── Fire
```

这是 Object Tree。

而：

```text
             TurretAim
             /    |    \
            ↓     ↓     ↓
          Yaw   Pitch  Physics
```

才是 dependency graph。

Builder 可以根据 scope/wiring：

```text
Yaw     ← ../YawRotation
Pitch   ← ./PitchRotation
Physics ← $root/Physics
```

构造：

```csharp
new TurretAim(
    yaw,
    pitch,
    physics
);
```

**运行以后 TurretAim 完全不需要知道 Object Tree。**

这一点很重要。

---

## 5. 所以我倾向于“允许多 Source，禁止多 Runtime Dependency”

听起来像文字游戏，其实区别很大。

不要：

```text
Aim
├── runtimeA
├── runtimeB
└── runtimeC
```

而是：

```text
Runtime A ──┐
Runtime B ──┼── Builder ──► AimContext ──► Aim
Runtime C ──┘
```

Runtime 是 **composition infrastructure**。

Capability 才是 **gameplay dependency**。

这样我们前面建立的原则仍然成立：

> Runtime 不传播进入 gameplay。

---

## 6. Control Surface 怎么初始化：我强烈倾向“构建时缓存，切换时绑定”

这里我不建议 gameplay 中不断：

```csharp
GetComponent<IAimControl>();
GetComponent<ITriggerControl>();
GetComponent<IWhatever>();
```

因为我们前面花那么多力气做 Runtime/Capability/Validation，最后如果 Control 又退回 `GetComponent`，等于绕回 Unity 默认模型。

我会把过程分成两个不同阶段：

```text
Object Build                         Gameplay
────────────                         ────────

构造 Object
   ↓
安装 Runtime Extensions
   ↓
Resolve Capability
   ↓
生成 Control Surface
   ↓
Validate
   ↓
Cache
                                     Player enters seat
                                            ↓
                                     Bind(surface)
                                            ↓
                                         Control
                                            ↓
                                     Player exits
                                            ↓
                                        Unbind
```

也就是说：

> **Surface 本身预先构建；Controller 与哪个 Surface 的连接是动态的。**

---

## 7. 例如 Tank 创建时就已经生成不同 Surface

Tank：

```text
Tank
├── DriverSeat
├── GunnerSeat
└── CommanderSeat
```

Build 阶段：

```text
DriverSurface
├── IMovementControl
└── IBrakeControl

GunnerSurface
├── IAimControl
├── ITriggerControl
└── IWeaponSelection

CommanderSurface
├── ITargetDesignation
└── IObservationControl
```

全部缓存下来。

玩家进入 Gunner：

```text
PlayerController
      │
      ▼
bind(GunnerSurface)
```

换 Driver：

```text
unbind(GunnerSurface)

bind(DriverSurface)
```

完全不需要重新：

```text
GetComponent
GetComponentInChildren
GetComponentInParent
```

---

## 8. Surface 本身最好也是 Capability Set，而不是固定 class

这里又要避免：

```text
TankGunnerControlSurface
SoldierControlSurface
AircraftPilotControlSurface
HelicopterGunnerControlSurface
...
```

类型爆炸。

可以就是：

```csharp
ControlSurface
{
    T Get<T>();
    T? Optional<T>();
}
```

内部：

```text
Gunner Surface

IAimControl       → TurretAim
ITriggerControl   → CannonFire
IZoomControl      → Optics
```

Controller：

```csharp
aim = surface.Optional<IAimControl>();
fire = surface.Optional<ITriggerControl>();
```

但注意——**这里的 Optional 查询发生在 Bind 时一次，而不是每帧。**

然后缓存：

```csharp
class PlayerController
{
    IAimControl? aim;
    ITriggerControl? fire;
    IReloadControl? reload;
}
```

Update：

```csharp
if (aim != null)
    aim.AddLookInput(...);
```

甚至这里的 `null` 我也不觉得是问题。

因为这里表达的是真的：

> 当前控制对象可能没有 Aim 能力。

这和 Weapon 内部 `AmmoComponent` 不知道存不存在完全不同。

前者是合法的 capability variability，后者是隐藏的 required dependency。

---

## 9. 可以进一步让 Input Binding 也跟着 Surface 动态变化

例如：

```text
PlayerController
      │
      ▼
GunnerSurface
├── Aim
├── Fire
└── Zoom
```

Input system 发现这些能力以后：

```text
Aim  → Mouse XY
Fire → Mouse 0
Zoom → Mouse 1
```

进入 Driver：

```text
DriverSurface
├── Move
├── Brake
└── Handbrake
```

重新 binding：

```text
Move      → WASD
Brake     → S
Handbrake → Space
```

所以不是：

```text
Tank 决定 Input
```

而是：

```text
Control Surface
       ↓
Controller Binding
       ↓
Input Profile
```

这对于以后 gamepad / keyboard / AI / network 都很好。

---

## 10. 这里其实可以区分“静态 Capability”和“动态 Binding”

这是整个问题最清楚的划分。

#### 静态/低频变化

对象能做什么：

```text
Cannon:
    ITriggerControl

Turret:
    IAimControl

Tank:
    IMovementControl
```

Build 时 Resolve + Cache。

#### 动态

谁正在使用：

```text
Player A → GunnerSurface
AI       → DriverSurface
Player B → CommanderSurface
```

只修改 Binding。

所以动态变化的是：

```text
Controller → Surface
```

而不是：

```text
Component Graph
```

这能让 runtime architecture 稳定很多。

---

## 11. 真正需要动态改变 Capability 时怎么办？

当然游戏里还是会有。

比如：

```text
Tank turret destroyed
```

现在：

```text
IAimControl
```

没了。

我也不会删除 Runtime component。

更倾向：

```text
IAimControl
    IsAvailable = false
```

或者发布：

```text
CapabilityStateChanged<IAimControl>
```

Controller 仍然持有同一个引用：

```text
TurretAim
```

只是：

```text
aim.Enabled = false
```

这样 dependency graph 不变。

同理：

```text
weapon disabled
engine destroyed
jammed
out of ammo
```

优先改变 **state**，而不是改变 Runtime structure。

---

## 12. 真正结构变化才重新 Build Surface

比如：

```text
安装新的炮塔
切换机甲手臂
拾取新武器
进入载具
装备不同 equipment
```

这种低频事件可以：

```text
Object Graph changed
        ↓
Runtime changed
        ↓
rebuild affected Surface
        ↓
SurfaceChanged
        ↓
Controller rebind
```

这种成本完全可以接受。

所以不是绝对“不动态”。

而是：

> **状态变化不要伪装成结构变化；真正的结构变化才重新 resolve。**

这条原则我觉得游戏架构里特别重要。

---

### 最终我觉得可以形成一个非常干净的模型

```text
                     OBJECT TREE
                         │
             ownership / hierarchy
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
     Soldier           Tank             Turret
        │                │                │
        ▼                ▼                ▼
     Runtime          Runtime           Runtime
        │                │                │
        └───────┬────────┴───────┬────────┘
                │                │
                ▼                ▼
        Capability Sources   Capability Sources
                │
                └──────┬─────────┘
                       ▼
                 Runtime View
                       │
               implementation
                       │
              ┌────────┴─────────┐
              ▼                  ▼
         SoldierAim          TurretAim
              │                  │
              └────────┬─────────┘
                       ▼
                  IAimControl
                       │
                       ▼
                 Control Surface
                       ▲
                       │ bind/unbind
                ┌──────┼──────┐
                ▼      ▼      ▼
             Player    AI   Network
```

所以对于你问的两个核心选择，我的答案会比较明确：

**Aim 应该允许不同实现，但尽量共享小而稳定的接口。** `SoldierAim` 和 `TurretAim` 内部依赖完全可以不同，不要为了复用强行统一实现。

**允许一个模块从多个 Runtime/Scope 获取依赖，但 Runtime 只参与构建。** Builder 把多个 source 裁剪成一个小的 Runtime View/Context，真正 gameplay module 不持有 Runtime A/B。

而 Control Surface 则采用一个很适合游戏的折中：**Build 时 resolve + cache，控制权变化时 bind/unbind；每帧直接调用缓存引用。** `GetComponent` 可以留给 Unity adapter/editor/debug 等边界代码，但不应该成为核心 gameplay architecture 的依赖发现机制。

这样其实把“动态游戏世界”和“静态可验证依赖”这两个看起来矛盾的目标同时保住了。

### 整理说明

1. 本篇保留了不同 Aim 实现、统一 capability contract、组合 Runtime 视图，以及预缓存与动态 GetComponent 之间取舍的讨论。
2. 核心建议是使用不同实现的 Aim Runtime/Component，但对外提供统一的 Aim Surface；不要为 Soldier、Tank、Turret 分别制造大量顶层 Runtime 类型。
3. 所谓组合 Runtime 更适合理解为多个 scope/source 的组合 View，而不是把多个 Runtime 继承成一个更大的 Runtime class。
4. 动态切换时应缓存结构化的 Control Context，在控制对象或姿态层级发生变化时重新 compose；不建议在 gameplay hot path 中反复 GetComponent。

## Camera 与 Zoom 的单例 Runtime


本篇讨论 Camera、Zoom、Camera Config 和 Player Session 之间的关系，重点是单例系统如何同时暴露行为、只读状态和配置数据。

### Q：Camera 和 Zoom 应该如何设计？Camera 是单例 Runtime 吗？Zoom 应该通过属性还是接口暴露？

#### 原始提问

> 非常不错 我感觉清晰很多了 分开runtime (需要的具体类型)  surface (提供能力的接口,可以不同的类型组成 runtime view但是能力的接口是 统一的 也就是说surface 接口即是能力 也方便思考交互关系.  那么你觉得 Camera 和zoom 要怎么设计? 
>
> 我觉得camera zoom 有了一点小变化 之前的object 有很多 所以谁来使用它变得有些复杂 但是camera 似乎是单例 而camera config类似运行时override 的config . 我感觉camera 应该的层级是类似player controller 这个位置 (似乎是一个单例插件? 那么camera comp 就相当于是他的runtime? 
>
> 类似的zoomcomp 怎么处理呢 它似乎也不是进行控制了 而是一个数据runtime 提供一些必要的数据封装? runtime.zoom.data这样? (但这个接口似乎和行为不太一样了 是使用属性暴露 还是依然构造接口暴露?

### A：

对，我觉得这里正好能把我们现在的模型再推进一步：**Surface 不一定只表示“可执行行为”，它更准确地表示“对外 contract”。Contract 可以是 Command，也可以是 Read-only State，甚至可以是 Config/Descriptor。**

Camera / Zoom 正好能帮助我们把这个概念想清楚。

---

### 1. Camera 我会放在 Player Session Runtime，而不是 Object Runtime

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

## 2. Object 不提供 Camera，而是提供 Camera Surface

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

## 3. 但这里不要让 `CameraConfig` 成为一个巨大 mutable config

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

## 4. Camera 可以分成 Profile 和 Override

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

## 5. 这时候 Zoom 的身份就非常有意思

你说：

> ZoomComp 好像不再是“控制”，而是 data runtime。

我认为**对一半**。

Zoom 通常同时存在两种语义：

#### Control

外部告诉它：

```text
StartZoom
StopZoom
SetZoomLevel
NextZoomLevel
```

#### State

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

## 6. 这其实回答了“Surface 到底是不是行为接口”

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

## 7. 但我不建议出现 `runtime.zoom.data`

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

## 8. Camera 甚至不应该直接依赖 ZoomComponent

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

## 9. 可以把 Camera Override 抽象成统一 Modifier

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

## 10. 不过 Zoom 本身不一定等于 Camera Modifier

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

## 11. Config、State、Control 最好明确区分

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

## 12. Camera Config 也一样

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

## 13. Camera“单例 Plugin”可以这么理解

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

### 所以我们现在其实可以把 Surface 再定义得更准确一点

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

### 整理说明

1. 本篇保留了 Surface 作为统一 Contract 的扩展：Contract 不仅可以表示 Command，也可以表示 Read-only State、Config 或 Descriptor。
2. Camera 更接近 Player Session/Controller 层的长期存在 Runtime；Camera Config 是可组合、可覆盖的数据，而不是必须拥有完整行为的 Component。
3. Zoom 可以作为 Camera Runtime 的一个独立 capability 或 config/state provider；是否提供属性取决于它是稳定只读状态还是需要行为约束的操作接口。
4. 单例只描述生命周期和作用域，不意味着 Camera 可以访问整个游戏 Runtime；它仍然应该通过裁剪后的 Surface/Capability View 读取需要的数据。

## HUD 与动态 Runtime


本篇讨论 HUD 为什么同时具有单例和多对象两种性质，以及 Player Controller/Session、HUD Renderer、当前控制对象和 Presentation Surface 之间的生命周期关系。

### Q：HUD 如何作为 Player Controller/Session 的 Runtime，动态绑定不同物品的 Surface？

#### 原始提问

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

### A：

对，我觉得你已经抓到 HUD 最关键的性质了：

> **HUD Renderer 是 Session 级长期存在的 Runtime；具体 HUD 内容却是跟随当前 Context 动态组合出来的。**

所以 HUD 看起来“既是 Singleton 又是多个对象”，其实是因为我们把两个生命周期不同的东西都叫 HUD 了。把它拆开以后会非常清楚。

---

### 1. HUD 应该拆成 `HUD Runtime` 和 `HUD Presentation`

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

## 2. Object 不应该“实现一个巨大的 HUDSurface”

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

## 3. 这和前面的 Control Surface 正好形成镜像

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

## 4. 切换武器时，不需要重建 HUD Runtime

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

## 5. 进入载具才可能改变 HUD Composition

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

## 6. 数据同步：不要让 HUD 每帧 GetComponent

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

## 7. 初始化 + 增量更新是最好用的组合

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

## 8. 我甚至建议把这种模式做成 Framework Primitive

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

## 9. 但也不要把所有属性都 Reactive 化

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

## 10. HUD Widget Plugin 可以声明自己需要什么

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

## 11. 更进一步：一个 Widget 可以绑定多个 Object

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

## 12. 关于 DSH：它给我们的启发更多是 Plugin 生命周期，而不是照搬 UI 模型

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

### 我觉得现在可以把 Camera 和 HUD 统一成一个更大的模式

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

### 整理说明

1. 本篇保留了 HUD Renderer、HUD Runtime、HUD Slot、HUD Composer 和 Presentation Surface 的完整推导。
2. 关键拆分是：HUD Renderer 是 Session 级长期存在的 Runtime；当前武器、载具或控制对象提供的 Presentation Surface 是动态 Context；HUD Plugin 则把二者连接起来。
3. HUD 不应直接成为物品的子对象，也不应复制物品状态；它应该观察由物品 Runtime 所拥有的状态或 Surface。
4. 对象切换时重新生成 Presentation Context，普通数值更新则从已缓存的 Capability 或 Observable State 读取。

## IAmmoStatus 与能力接口


本篇澄清 Surface/Capability 接口和 Event 接口的区别，以 IAmmoStatus 为例说明状态读取能力应该由谁实现。

### Q：IAmmoStatus 应该由谁实现？它是 Surface 还是 Event 接口？

#### 原始提问

> IammoStatus 这个接口应该谁实现 我有点没搞懂 它是我们之前广义上的surface 还是一个event 接口 (我理解这是两种不同的接口

### A：

对，这里确实需要把概念再切清楚一点。**`IAmmoStatus` 是 Surface/Capability 接口，不是 Event 接口。** 我上一条把 reactive/event 放进它里面，容易让两层概念看起来混在一起。

我会这样定义：

```text id="f4uyqa"
Runtime Object
    ↓ implements
Capability Interface
    ↓ exposed through
Presentation Surface
    ↓ consumed by
HUD
```

而 Event 是另一条独立机制。

---

### 1. `IAmmoStatus` 应该由谁实现？

最自然就是由真正拥有 Ammo 状态的 runtime object 实现，例如：

```csharp id="eh8d7v"
public interface IAmmoStatus
{
    int Current { get; }
    int Capacity { get; }
}
```

然后：

```csharp id="8w0c9d"
public sealed class Magazine :
    IAmmoSource,
    IAmmoStatus
{
    private int ammo = 30;

    // 给 Weapon 使用
    public bool TryConsume(int amount)
    {
        if (ammo < amount)
            return false;

        ammo -= amount;
        return true;
    }

    // 给外界观察
    public int Current => ammo;
    public int Capacity => 30;
}
```

所以：

```text id="7x46m6"
                 Magazine Runtime
                 /              \
                ▼                ▼
          IAmmoSource        IAmmoStatus
              ▲                  ▲
              │                  │
           Weapon               HUD
```

这就是我们之前说的：

> **同一个 Runtime Object 暴露不同的“面”。**

`Magazine` 是 implementation/runtime object。

`IAmmoSource` 和 `IAmmoStatus` 都是 capability/surface contracts。

---

## 2. Event 是另外一个东西

例如：

```csharp id="mp50hw"
public readonly record struct AmmoChanged(
    int Current,
    int Capacity
);
```

Magazine 修改 ammo 后：

```csharp id="g13b3x"
ammo -= amount;

events.Publish(
    new AmmoChanged(ammo, capacity)
);
```

所以关系其实是：

```text id="8cjwq7"
                  Magazine
                 /        \
                /          \
               ▼            ▼
       IAmmoStatus      AmmoChanged Event
               │            │
               ▼            ▼
              HUD ◄──────── HUD
```

两条路径承担不同职责：

```text id="j4ok6m"
IAmmoStatus
    ↓
“现在 ammo 是多少？”

AmmoChanged
    ↓
“ammo 刚刚发生变化。”
```

这是 **State / Event** 的区别。

---

## 3. 为什么两个都需要？

假设 HUD 在 `t=100` 时才绑定 Weapon。

之前：

```text id="dqzyyn"
t=10  ammo 30 → 29
t=20  ammo 29 → 28
t=30  ammo 28 → 27
...
t=100 HUD bind
```

如果 HUD 只有：

```text id="29nkra"
AmmoChanged Event
```

它不知道当前到底是：

```text id="idn7jf"
27?
12?
0?
```

因为以前的 Event 已经过去了。

但有：

```csharp id="0p0uox"
IAmmoStatus.Current
```

HUD bind 时：

```text id="5lxnyq"
HUD
 ↓
IAmmoStatus.Current
 ↓
27
```

然后以后再听：

```text id="0s24ob"
AmmoChanged
```

所以：

> **Surface/State 回答“现在是什么”；Event 回答“刚发生了什么”。**

---

## 4. Presentation Surface 也不是 `IAmmoStatus` 本身

这里还可以再精确一级。

`IAmmoStatus` 是一个 **Capability Interface**。

Presentation Surface 是：

> **一组允许 Presentation 层看到的 Capability。**

例如：

```text id="ywut30"
Weapon Runtime
├── IAmmoSource
├── IAmmoStatus
├── IReloadControl
├── IReloadStatus
├── ITriggerControl
└── IWeaponStatus
```

但它暴露给 HUD 的：

```text id="8kn6tv"
Presentation Surface
├── IAmmoStatus
├── IReloadStatus
└── IWeaponStatus
```

暴露给 Controller 的：

```text id="n7z4he"
Control Surface
├── ITriggerControl
└── IReloadControl
```

所以严格术语是：

```text id="2a01f6"
Capability
    ↓
被不同 Surface 选择性暴露
```

可以理解成：

```text id="2jxdu2"
                   Weapon Runtime
                        │
        ┌───────────────┼──────────────┐
        │                              │
        ▼                              ▼
 Control Surface              Presentation Surface
        │                              │
 ITriggerControl                  IAmmoStatus
 IReloadControl                   IReloadStatus
        │                              │
        ▼                              ▼
   Controller                         HUD
```

---

## 5. 那么谁负责 Event？

这里有两种设计，我现在更倾向于**不要让每个 Capability 自己发明 event API**。

比如不要大量写：

```csharp id="0s31po"
interface IAmmoStatus
{
    int Ammo { get; }

    event Action AmmoChanged;
}

interface IHealthStatus
{
    float Health { get; }

    event Action HealthChanged;
}

interface ISpeedStatus
{
    float Speed { get; }

    event Action SpeedChanged;
}
```

虽然能用，但久了以后每个接口都有自己的一套 subscription 方式。

更统一的方式是 Runtime 有：

```text id="yp0a4c"
EventHub
```

Magazine：

```text id="1cfdme"
Magazine
 ├── provides IAmmoStatus
 └── publishes AmmoChanged
```

HUD：

```text id="p01n4e"
AmmoWidget
 ├── reads IAmmoStatus
 └── subscribes AmmoChanged
```

这和我们之前的：

```text id="ewm4xg"
Provides
Requires
Publishes
Consumes
```

又完全对应起来了。

---

## 6. 不过 UI 数据绑定可以做一层便利封装

这是我上一条说 `ReactiveValue<T>` 的原因。

底层概念仍然是：

```text id="pm8c1r"
State + Event
```

但 UI 每次都写：

```csharp id="pfeyqd"
ammoText.text = ammo.Current.ToString();

events.Subscribe<AmmoChanged>(e =>
{
    ammoText.text = e.Current.ToString();
});
```

很烦。

所以 Presentation framework 可以提供：

```csharp id="r9ugdf"
Bind(
    initial: () => ammo.Current,
    changed: AmmoChanged,
    render: value => ammoText.text = value.ToString()
);
```

甚至包装成：

```text id="c86t2a"
ObservableProperty
```

但这是 **UI binding convenience layer**，不是我们基础 architecture 必须把：

```text id="0bv44q"
Capability = Event
```

混成一种东西。

---

### 所以我建议把现在的术语正式固定下来

这个 distinction 以后非常重要：

| 概念 | 例子 | 意义 |
|---|---|---|
| **Runtime Object** | `Magazine` | 真正持有状态和逻辑 |
| **Capability Interface** | `IAmmoSource`, `IAmmoStatus` | Runtime 对外提供的某一种能力 |
| **Control Surface** | `ITriggerControl + IReloadControl` | 允许 Controller 使用的 Capability 集合 |
| **Presentation Surface** | `IAmmoStatus + IWeaponStatus` | 允许 HUD/Camera 观察的 Capability 集合 |
| **Event** | `AmmoChanged`, `WeaponFired` | 描述已经发生的变化/事实 |

因此：

```text id="5hh3sk"
Magazine
│
├── state
│   ammo = 27
│
├── implements
│   ├── IAmmoSource
│   └── IAmmoStatus
│
└── publishes
    └── AmmoChanged
```

然后：

```text id="plm7am"
Weapon
   ↓
IAmmoSource
   ↓
Magazine

HUD
   ↓
IAmmoStatus
   ↓
Magazine

HUD
   ↓ subscribe
AmmoChanged
   ↑ publish
Magazine
```

我觉得这样是最干净的。

尤其可以记住一句很简单的判断：

> **Interface/Capability 表示“你现在允许我做什么/知道什么”；Event 表示“刚刚发生了什么”。**

所以 `IAmmoStatus` 明确属于前者。

### 整理说明

1. 本篇保留了对 IAmmoStatus、Surface/Capability、Event 和 Reactive/Observable 数据流的概念切分。
2. IAmmoStatus 应由真正拥有弹药状态的 Ammo Runtime 或其 Adapter 实现；HUD 只消费这个接口，不应该把事件方法塞进同一个接口。
3. 能力接口描述当前可读取或调用的 contract，事件接口描述已经发生的离散事实；两者可以协作，但不应混成一个类型。

## 事件推送、Pull 与能力接口


本篇讨论 HUD 数据同步中 Event Push、每帧 Pull、订阅绑定和 Capability Surface 的边界。

### Q：Event Push 和自动 Pull 同时存在是否过于复杂？订阅绑定与 Runtime 初始化如何处理？

#### 原始提问

> 这是不是有点太复杂了 event push变更 但是还得自动pull 数值 然后event的订阅绑定似乎和runtime 的初始化里面也是一个挺需要考虑的事情

### A：

是的。我觉得这里我们刚才确实有一点**架构过度设计**了。

如果 HUD 为每个数据都变成：

```text id="h1a3ny"
找到 IAmmoStatus
    ↓
读取 Current
    ↓
找到 EventHub
    ↓
订阅 AmmoChanged
    ↓
保存 subscription
    ↓
切武器 unsubscribe
    ↓
重新 resolve
    ↓
读取新值
    ↓
重新 subscribe
```

那一个简单的“显示子弹数”已经承担了太多 infrastructure complexity。

我反而建议把它收回来。

### 对 HUD，Pull 很可能就是最好的默认方案

游戏本来就有 frame update：

```text id="2vwlht"
Update()
  ↓
HUD.Render()
  ↓
读取当前 Presentation Surface
  ↓
显示
```

例如：

```csharp id="ht01sf"
public interface IAmmoStatus
{
    int Current { get; }
    int Capacity { get; }
}
```

HUD bind 时只做一次：

```csharp id="pskvwz"
ammo = surface.Optional<IAmmoStatus>();
```

之后：

```csharp id="wql5zf"
void Update()
{
    if (ammo != null)
        ammoText.text = $"{ammo.Current}/{ammo.Capacity}";
}
```

就结束了。

这里每帧并没有 `GetComponent`、reflection、runtime lookup。

只是：

```text id="v1qu8u"
cached interface reference
       ↓
property read
```

成本和复杂度都很低。

---

### Event 应该留给真正的“事件”

我们前面有点把两个概念混在一起了：

```text id="30d6w7"
AmmoChanged
HealthChanged
SpeedChanged
PositionChanged
AimChanged
...
```

如果所有 state mutation 都生成 event，很快会得到一个巨大的 reactive framework。

其实游戏里很多东西天然就是连续状态：

```text id="rmam12"
Position
Velocity
Aim
RPM
Speed
Health
Ammo
ReloadProgress
Heat
```

HUD 每帧读取完全合理。

而 Event 更适合：

```text id="8zdibj"
WeaponFired
WeaponReloaded
VehicleDestroyed
TargetLocked
PlayerEnteredVehicle
WeaponSwitched
```

也就是：

> **State → Pull**
>
> **Occurrence → Event**

这个规则简单很多。

---

## 1. 甚至 `AmmoChanged` 都不一定需要存在

Magazine：

```csharp id="hs8v8x"
class Magazine :
    IAmmoSource,
    IAmmoStatus
{
    int ammo;

    public int Current => ammo;

    public bool TryConsume(int amount)
    {
        if (ammo < amount)
            return false;

        ammo -= amount;
        return true;
    }
}
```

HUD：

```text id="b38uz6"
AmmoWidget
     │
     │ cached
     ▼
IAmmoStatus
     │
     ▼
Magazine
```

每帧：

```csharp id="t5f0f1"
text.text = ammo.Current.ToString();
```

非常朴素。

**朴素在这里反而是优点。**

---

## 2. Event 只负责“不能通过当前状态完整表达”的事情

例如：

```text id="p4usqk"
WeaponFired
```

你不能通过：

```text id="hqtmrn"
weapon.IsFiring
```

可靠知道：

> 刚刚发生了一次开火。

因为可能一帧开一次：

```text id="8j86cs"
false → true → false
```

甚至你根本不需要 `IsFiring`。

所以：

```text id="jld5fz"
WeaponFired
```

是 Event。

同理：

```text id="j5w41n"
HitConfirmed
Kill
Explosion
EnteredVehicle
ExitedVehicle
WeaponChanged
```

天然适合 Event。

但：

```text id="rj0b3f"
CurrentAmmo
Health
Speed
FOV
ReloadProgress
```

天然适合 State。

---

## 3. 这样 Runtime 初始化也突然简单很多

我们原来的复杂模型：

```text id="cb76wm"
Resolve
 ↓
Bind state
 ↓
Subscribe events
 ↓
Track subscriptions
 ↓
Unsubscribe
 ↓
Rebind
```

现在 Presentation Surface：

```text id="qj2vcl"
Weapon switched
      ↓
Build/Resolve Presentation Surface
      ↓
HUD.Bind(surface)
      ↓
cache interfaces
```

例如：

```csharp id="0hs3zx"
void Bind(PresentationSurface surface)
{
    ammo   = surface.Optional<IAmmoStatus>();
    weapon = surface.Optional<IWeaponStatus>();
    aim    = surface.Optional<IAimStatus>();
}
```

以后 Update：

```csharp id="hxrz9f"
void Update()
{
    if (ammo != null)
        DrawAmmo(ammo);

    if (weapon != null)
        DrawWeapon(weapon);

    if (aim != null)
        DrawCrosshair(aim);
}
```

切武器：

```text id="n0jq3k"
AK47 Surface
      ↓
Bind
      ↓
HUD

换武器

RPG Surface
      ↓
Bind
      ↓
HUD
```

没有 subscription 生命周期。

---

## 4. 甚至 Widget Composition 也可以在 Bind 时一次解决

比如：

```csharp id="fh4ulv"
void Bind(PresentationSurface surface)
{
    ammoWidget.SetVisible(
        surface.Has<IAmmoStatus>()
    );

    vehicleWidget.SetVisible(
        surface.Has<IVehicleStatus>()
    );

    lockWidget.SetVisible(
        surface.Has<ILockStatus>()
    );
}
```

然后缓存：

```text id="kk5jtp"
AmmoWidget
   └── IAmmoStatus reference

VehicleWidget
   └── IVehicleStatus reference
```

运行阶段完全没有 dependency resolution。

---

## 5. EventHub 仍然存在，但它变得非常小

现在 EventHub 不再负责：

```text id="r2u1jd"
所有数据同步
```

只负责真正的：

```text id="u1kxhz"
domain occurrences
```

例如：

```text id="w4w8wu"
WeaponFired
VehicleDestroyed
HitConfirmed
SeatChanged
WeaponEquipped
```

FX：

```text id="ttfkr4"
WeaponFired
   ↓
MuzzleFlash
```

Audio：

```text id="c1yby8"
WeaponFired
   ↓
GunSound
```

HUD 有时候也可以听：

```text id="hdxd2d"
HitConfirmed
   ↓
HitMarker
```

因为 HitMarker 本身就是：

> 发生一次 → 播一个短暂 UI 动画。

这就是 Event 非常合适的地方。

但 HUD 上：

```text id="rt24cv"
Ammo: 27
Health: 76
Speed: 42 km/h
```

直接 Pull。

---

## 6. Camera 其实也一样

我们刚才如果走得太 reactive：

```text id="vnjwxx"
ZoomChanged
AimChanged
FovChanged
CameraMountChanged
...
```

也会很复杂。

Camera 本来就是每帧更新：

```csharp id="jwb5at"
void LateUpdate()
{
    var mount = profile.Mount;

    var targetFov = profile.BaseFov;

    if (zoom != null)
        targetFov *= zoom.FovMultiplier;

    ...
}
```

所以：

```text id="dj7aqy"
ICameraProfile
IZoomState
IAimState
```

直接读取。

只有：

```text id="eg60a5"
CameraModeChanged
EnteredVehicle
ExitedVehicle
```

这种低频结构变化才触发 rebind。

---

## 7. 我甚至会给 Framework 定一个非常简单的规则

#### 高频 / 连续 / 当前值

**Pull**

```text id="ohdf2j"
Position
Rotation
Velocity
Health
Ammo
Heat
RPM
Aim
Zoom
ReloadProgress
```

#### 离散 / 瞬时 / “发生了一件事”

**Event**

```text id="hw00pm"
Fired
Hit
Killed
ReloadCompleted
EnteredVehicle
WeaponSwitched
Explosion
```

#### 结构发生改变

**Rebind / Rebuild Surface**

```text id="zghmm3"
Equip weapon
Unequip weapon
Enter seat
Exit seat
Attach turret
Change controlled object
```

这三个机制基本覆盖绝大多数情况：

```text id="k0kw5f"
       Game State
          │
   ┌──────┼─────────┐
   ▼      ▼         ▼
 State   Event   Structure
   │      │         │
 Pull    Push     Rebind
```

我觉得这个模型比“全部 reactive”健康得多。

---

## 8. 这也回答了一个更深的问题：Surface 应该尽量“傻”

Presentation Surface 不需要：

```text id="vwm1cs"
subscription management
event forwarding
reactive graph
data synchronization
```

它就只是：

> **一组已经 resolve 好的接口引用。**

例如：

```text id="3l4xqz"
PresentationSurface
├── IHealthStatus → Soldier.Health
├── IAmmoStatus   → Rifle.Magazine
├── IWeaponStatus → Rifle.Fire
└── IAimStatus    → Soldier.Aim
```

HUD：

```text id="7f39yh"
Bind once
   ↓
cache references
   ↓
read every frame
```

Control Surface：

```text id="i7z4mh"
ControlSurface
├── IMovementControl
├── IAimControl
├── ITriggerControl
└── IReloadControl
```

Controller：

```text id="8e95fl"
Bind once
   ↓
cache references
   ↓
call every frame
```

你会发现两边现在惊人地对称：

```text id="hj8p3e"
              Object Runtime
              /            \
             /              \
            ▼                ▼
    Control Surface   Presentation Surface
            │                │
        cached ref       cached ref
            │                │
            ▼                ▼
       Controller           HUD
            │                │
            ▼                ▼
           Call             Read
```

这比我们刚才引入 `ReactiveValue + EventHub + Subscription` 那套明显更符合我们一直追求的“**把复杂度留在 build/bind 阶段，让 runtime 本身非常简单**”。

所以你这个质疑我觉得非常关键：如果一个架构为了显示 `Ammo = 27` 都需要一套状态 + event + subscription + lifecycle，那么说明 abstraction 已经开始反过来服务自己了。这里完全可以大胆选择游戏引擎最擅长的模式：**Surface 在结构变化时 resolve 一次，连续状态每帧直接 pull，真正的瞬时事件才 push。**

### 整理说明

1. 本篇保留了对连续状态、离散事件、缓存 Surface、订阅生命周期和 Runtime 初始化的完整讨论。
2. 连续数值适合 Pull 或 Observable Store，离散变化适合 Event；两者可以共存，但不应让同一条数据同时产生多套无约束的真相来源。
3. 订阅关系应由 Composer/Runtime 在初始化和销毁阶段统一建立、撤销，而不是让 UI 在运行时到处寻找对象或重复绑定。
4. 这一篇承接 IAmmoStatus，说明为什么状态 Surface 和事件通知需要保持不同的接口边界。

## DSH UI 调研问题


本篇保留关于搜索 DSH HUD/UI Plugin 设计的原始提问。该轮没有独立回答，下一篇记录随后得到的调研结果。

### Q：能否搜索 DSH 中 HUD/UI 如何按照 Plugin 方式设计？

#### 原始提问

> 你可以让codex 搜索看一下dsh 里面hud/ui 是如何按照plugin 方式设计的吗 这确实变得有意思了 我感觉ui 既涉及底层对象 又和用户控制 单例相关
> 然后把结果在这里总结告诉我

本轮没有独立回答；下一篇保留了随后对 DSH UI 源码与文档的调研结果。

### 整理说明

1. 本篇刻意保留没有独立回答的原始问题，避免把问题和后续调研结果伪装成同一轮回答。
2. 该问题是后续 DSH UI 讨论链的入口，因此按照原始顺序放在最后一组，而不是提前合并到 HUD 文件中。

## DSH UI Plugin 的调研结果


本篇整理针对 DSH UI/HUD Plugin 的源码和文档调研结果，重点是 UI Host、Slot/Seat、Plugin-owned State、Global UI 和 Session UI。

### Q：DSH 的 UI Plugin 实际是如何设计的？它与我们的 HUD 模型有什么关系？

#### 原始提问

> 你之前的分析有结果了吗

### A：

有结果了。我刚又对照了 DSH 当前公开源码和文档。结论很有意思：**DSH 的 UI 设计和我们刚才推导的 HUD 模型非常接近，但它比我们设想的“Object → HUD Surface”更偏向 `UI Host + Slot + Plugin-owned State`。** 

#### DSH 实际怎么做 UI Plugin

DSH 不是让业务对象“拥有 UI”。它有一个长期存在的 Web UI Host，然后各个 plugin 往 Host 暴露的 **slot/seat** 里注册 UI：

```text
Web UI / Layout Host
│
├── sidebar.footer.action
│       ↑
│   Cordis UI Plugin
│
├── tool.call.toolview
│       ↑
│   Cordis UI Plugin
│
└── tool.view.cordis
        ↑
    Dynamic Plugin UI
```

例如 `ui-cordis` 自己的 host-side `apply()` 甚至是空的；真正 UI 在 browser/client half，通过 package 声明被发现。它注入 `slots / locale / inputTriggers / remote / runner` 等能力，然后把自己的 panel/card 注册到现有 UI slot，而不是创建自己的 UI root。

这和我们现在的游戏设计可以非常漂亮地对应：

```text
DSH                         Game
──────────────────────────────────────
Web UI Host                HUD Runtime
Slot / Seat                HUD Slot
UI Plugin                  HUD Plugin
Cordis Service             Runtime Capability
Session                    Player Session
```

---

#### 更关键的是：DSH 没有把所有 UI 状态塞进 React Component

`ui-cordis` README 特别强调了一件事：**运行状态并不保存在 UI component 自己的 state 里，而是由真正负责这些状态生命周期的对象拥有 observable/state。**

例如 browser runner 拥有 open requests、run outcome 等状态；UI 只是观察这些状态。

所以它实际更接近：

```text
Runtime / Service
      │
      │ owns state
      ▼
Observable State
      │
      ▼
UI Plugin
      │
      ▼
UI Slot
```

而不是：

```text
Business Object
      ↓
HUD Component
      ↓
自己复制一份状态
```

这一点对我们的设计很有启发。

---

#### DSH 还有一个非常重要的区分：Global UI 和 Session UI

这正好回答你之前说的：

> UI 既有单例性质，又有多个对象性质。

DSH 明确两种 scope。

`ui-cordis` 有一个 frame-wide panel：

```text
整个 Web App
   │
   └── Cordis Panel
```

它故意是 global 的，因为它必须能看到**所有 session** 的动态 plugin 状态；甚至当前没打开那个 session，也必须能批准那个 session 发出的请求。

同时又有：

```text
tool.view.cordis
    kind: keyed
    scope: session
```

也就是 session-scoped UI。

所以 DSH 实际上已经面对了和我们非常类似的问题：

```text
Global UI
   +
Session UI
   +
Dynamic Plugin UI
```

并没有试图用一种 ownership 关系解决全部。

---

### 这让我想修改一下我们 HUD 的设计

之前我们说：

```text
Object
    ↓
Presentation Surface
    ↓
HUD
```

我现在觉得这个方向没错，但可以吸收 DSH 的思路变成：

```text
                    Player Session
                         │
                    HUD Runtime
                         │
                 ┌───────┴───────┐
                 ▼               ▼
             HUD Slots      HUD Composer
                 ▲               │
                 │               │
           HUD Plugins           │
                                 │
                          Presentation Context
                                 ▲
                  ┌──────────────┼──────────────┐
                  │              │              │
               Player          Weapon         Vehicle
```

这里出现两个完全不同的东西：

**HUD Slot 解决“UI 放在哪里”。**

**Presentation Surface 解决“UI 从哪里拿数据”。**

这是个非常好的分离。

---

#### 例如 Ammo HUD 不再属于 Weapon

可以写成：

```text
AmmoHUDPlugin

requires:
    IAmmoStatus

renders:
    HUD.BottomRight
```

进入步兵状态：

```text
Presentation Context
├── IHealthStatus
├── IAmmoStatus
└── IWeaponStatus

        ↓ match

HUD
├── HealthWidget
├── AmmoWidget
└── CrosshairWidget
```

进入坦克：

```text
Presentation Context
├── IVehicleStatus
├── IAmmoStatus
├── ISpeedStatus
└── IAimStatus

        ↓ match

HUD
├── VehicleHealthWidget
├── AmmoWidget
├── SpeedWidget
└── VehicleCrosshairWidget
```

注意 `AmmoWidget` 完全可能继续存在，只是：

```text
IAmmoStatus
 RifleMagazine
       ↓
 TankCannonMagazine
```

发生 rebind。

---

### 而且 DSH 给了我们一个很好的 Plugin 生命周期方案

Cordis 本身负责 plugin mounting/unmounting，并且通过 `ctx` 注册的 event listener、timer 等会随着 plugin unload 自动清理；需要显式资源清理时则使用 effect/disposer。插件还可以通过 `inject` 声明依赖，只有依赖 ready 后才加载。

这特别值得我们借。

我们之前担心：

```text
HUD bind
 ↓
subscribe
 ↓
保存 subscription
 ↓
unbind
 ↓
unsubscribe
```

可以引入一个非常轻量的：

```text
HUDPluginScope
```

例如：

```csharp
scope.Bind<IAmmoStatus>(...);
scope.On<HitConfirmed>(...);
scope.AddWidget(...);
```

当 scope dispose：

```text
HUDPluginScope.Dispose()
        ↓
remove widget
unsubscribe events
release bindings
```

这其实就是 Cordis `ctx` lifecycle management 的游戏版。

不过我仍然维持上一轮结论：**不要因为 Cordis 能自动清 event，就让 Ammo/Health/Speed 全部 event-driven。**

---

### DSH 自己其实也是 Hybrid：Snapshot/State + Event/Stream

DSH 的 UI cookbook 描述的是一种混合模型：durable session/event records 用于已经确定的状态/历史，实时 token 展示则消费 transient stream。

这其实进一步支持我们刚才收敛出来的：

```text
Current State     → Pull / observable state
Occurrence        → Event
Continuous stream → Stream
Structure change  → Rebind
```

而不是：

```text
Everything → Event
```

---

### 映射到游戏，我现在会这样定

```text
                       Game Runtime
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
         World Runtime             Player Session
                                        │
                          ┌─────────────┼─────────────┐
                          ▼             ▼             ▼
                    Controller       Camera         HUD
                          │             │             │
                          │             │       ┌─────┴─────┐
                          │             │       │ HUD Slots │
                          │             │       └─────▲─────┘
                          │             │             │
                          │             │        HUD Plugins
                          │             │             │
                          ▼             ▼             ▼
                     Control       Presentation   Presentation
                     Surface         Surface        Surface
                          ▲             ▲             ▲
                          └─────────────┼─────────────┘
                                        │
                                   Object Tree
```

其中 HUD Plugin 类似 DSH UI plugin：

```text
AmmoHUDPlugin
────────────────
inject:
    HUD.BottomRight
    IAmmoStatus

apply:
    create AmmoWidget

dispose:
    remove AmmoWidget
```

而：

```text
HUD.BottomRight
```

是 **UI infrastructure capability**；

```text
IAmmoStatus
```

是 **gameplay presentation capability**。

这两个 dependency 来源完全不同，却可以在同一个 HUD Plugin 中组合起来。

这其实正是 Cordis 思路最有价值的地方：

> **Plugin 不属于任何一个 dependency。Plugin 是把多个 capability 接起来的胶水。**

AmmoHUD 不属于 Weapon，也不属于 HUD Runtime：

```text
Weapon
   │
   └── IAmmoStatus ─────┐
                        │
                        ▼
                   AmmoHUDPlugin
                        ▲
                        │
HUD Runtime ─ HUD Slot ─┘
```

我觉得这比我们之前说“HUD 是物品的观察者”还要更准确。

---

#### 还有一个 DSH 很值得借、但游戏里不要完全照搬的点

DSH 的 UI 可以高度动态：plugin load/unload、slot registration、HMR 等都是核心能力；Cordis 本身就是为这种动态 plugin composition 设计的。官方也明确把 UI 和 sessions、tools、models 等都作为可组合 plugin。

游戏 HUD 不需要这么动态。

我们完全可以：

```text
Game Start
    ↓
注册所有 HUD Plugin factories
    ↓
Seal Registry

Weapon/Seat changed
    ↓
重新生成 Presentation Context
    ↓
匹配可用 HUD Plugins
    ↓
mount/unmount affected widgets
```

而不是每帧动态 resolve plugin。

这样保留 Cordis 的：

**Plugin composition + dependency declaration + scoped lifecycle + slot UI**

但去掉它对我们不必要的：

**任意 runtime plugin load/unload + HMR + 全动态 service graph。**

我觉得这次看 DSH 后，我们的 HUD 架构反而可以收敛得更简单：**HUD Runtime 是 Session 级 UI Host；HUD Slot 是布局能力；Object 提供 Presentation Capability；HUD Plugin 是二者之间的 adapter/glue；结构切换时重新 compose，普通数值每帧直接从缓存的 capability pull。**

这套模型已经相当统一，而且 Camera 其实也可以用几乎同样的 Plugin/Slot/Context 思维来解释。

### 整理说明

1. 本篇保留调研回答中的 Slot/Seat、UI Host、Plugin-owned State、Global UI 与 Session UI 等关键内容，并清除了对话系统内部引用标记。
2. DSH 的 Web UI Host 类似 HUD Runtime，Slot/Seat 类似 HUD Slot，UI Plugin 负责把 Runtime Capability 接入 UI，而不是让业务对象直接拥有整个 UI。
3. 相关调研可参考 [DSH 扩展包说明](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/extensions/README.md)、[Web Client 架构](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/web-client.md) 和 [客户端 Slot 规范](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/AGENTS.md)。

## DSH UI 与游戏数据流


本篇继续整理 DSH UI 设计对游戏 HUD 的启发，重点讨论对象事件、连续状态、中央 HUD Surface 和输入/控制数据流之间的区别。

### Q：游戏 UI 是否应该区分对象事件、连续状态和中央 HUD Surface？DSH UI 的数据流哪些可以借鉴？

#### 原始提问

> 是的 我感觉 DSH 的UI 设计非常有启发性 UI plugin 只解决能力依赖这个问题 数据走一条新的路线 订阅 或者 stream 并且也是向物体本身进行订阅
>
>
>
> 我觉得有两个需要注意的点,不知道我的理解是否准确:
>
> 1. DSH 它的ui/控制流相对简单一点 不需要动态的切换加载大量物体 但游戏里面对应的gameobject 可能需要频繁变动 以及数据连续变动 用pull 可能比直接event push 更好一点 (感觉游戏里面会有很多事件系统 而不是一个全局事件系统 我感觉这里需要特别注意 比如每个weapon 都有fire event 即使不是玩家选中的 这些event也可以服务fx plugin 但ui 最好和他们分开 变成中央的数据组织surface 否则感觉很容易出问题 (比如绑定了错误的信号
> 2. 同样DSH 的ui 其实还复杂一点 需要发布信息 (比如用户输入) 我看它走的是全局的host event 这点游戏的ui 不需要 但是可以在输入系统里面借鉴

### A：

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

### 1. 我非常赞成：不要搞一个“大一统 Global Event Bus”

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

### 2. Weapon Event 最自然应该属于 Weapon Runtime / Scope

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

## 3. 这和 Presentation Surface 的价值正好形成对比

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

## 4. 所以我现在甚至更赞成 HUD 连续数据直接 Pull

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

## 5. Event 仍然非常重要，但它更应该是 scoped event

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

#### WeaponFired

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

#### SelectedWeaponChanged

```text
Player Session Event
```

给：

```text
HUD
Camera
Input binding
```

#### ObjectiveCaptured

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

## 6. Presentation Surface 本身甚至可以成为一个“中央 Context Projection”

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

## 7. 这样可以把“结构变化”和“数据变化”彻底分开

这是我觉得现在最值得固定成原则的一点：

#### 数据变化

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

#### 结构变化

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

## 8. 你的第二点也对：DSH 的 Host Event 更像我们的 Input/Command 路径

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

## 9. 但游戏 UI 里确实有交互型 UI

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

## 10. Input System 甚至可以把 DSH Host Event 思想吸收得更多一点

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

### 我觉得现在我们已经自然形成了四条完全不同的数据/控制通道

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

### 整理说明

1. 本篇保留关于 DSH UI 更偏事件/消息驱动、游戏连续状态更适合 Pull、局部对象事件与中央 UI 数据组织需要隔离的完整讨论。
2. 每个 Weapon 可以有自己的 fire event，这类事件可以服务 FX；但 HUD 不应盲目订阅所有对象事件，而应通过 Player Session/HUD Surface 读取当前上下文。
3. 游戏 UI 可以借鉴 DSH 的输入和 Host Event 思路，但不应照搬全局事件总线；输入仍应经过 Input/Controller 层转换为明确的控制命令。

## Surface 的组装与动态生命周期


本篇讨论 Surface 如何组装，以及动态 UI Surface 的 OnChange、OnActive、OnDeactivate 等生命周期逻辑应该由谁负责。

### Q：Surface 应该由工厂组装，还是由类型自己负责动态生命周期？

#### 原始提问

> 不过这还有个问题 surface 如何组装 我们之间 plugin 模式讨论了这部分 初始化对象的时候可以用工厂组装 surface  但是我们刚才提到的 UI surface 有点变成动态的了 是让这个类型自己负责 比如 on change on active on deactive 之类的 还是需要外部对这部分进行组装?

### A：

我会选择一个比较明确的边界：

> **Surface 不负责自己的组装，也不应该有复杂的 `OnActive / OnDeactivate / OnChange`。**
>
> **Surface 是一个被外部 Composer 生成的轻量 projection。**

也就是说，你前面 Object 初始化时的 Factory 思路没有错；只是 UI/Session 的 Surface 生命周期不同，所以需要一个 **Session-level Composer**，而不是 Object Factory。

---

### 1. 我会区分两类 Surface

#### Object Surface：基本静态

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

#### Session Surface：动态 projection

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

## 2. 所以我会增加一个很薄的 `SessionContext`

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

## 3. 谁触发 Composer？

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

## 4. Surface 最好是 dumb object

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

## 5. 动态切换最好是“换 Surface”，而不是“修改 Surface”

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

## 6. 但有一个问题：Composer 怎么知道“去哪找 IAimControl”？

这里才是整个问题真正有意思的地方。

我认为有两种方式，而且可以同时存在。

#### A. Convention / Scope resolution

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

#### B. Seat/Object 自己声明 Binding Recipe

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

## 7. 这里 Plugin 思想又能回来

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

## 8. HUD Plugin 又在 Surface 之后

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

## 9. `OnActive / OnDeactivate` 应该放在哪里？

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

### 我现在会把整个初始化流程定成两种 Composition

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

### 整理说明

1. 本篇保留原始回答关于 Factory、Composer、Surface 自身职责和动态生命周期边界的完整讨论。
2. 整理时特别保留了“Surface 不负责自己的组装”这一核心判断，因为它直接关系到动态 UI 和 Runtime 初始化的复杂度。
3. 动态绑定不等于每帧重新解析依赖；结构变化时重新 compose，普通数据变化则应通过已建立的能力接口读取或订阅。

## Surface 的数据、逻辑与 Present


本篇是这组讨论的最后一篇，整理 Surface 只保存索引、Controller 负责维护索引，以及 Plugin 的 present 能力为什么需要谨慎限制。

### Q：Surface 是否应该把数据和逻辑分开？Plugin 是否应该通过 present 提供其他 Runtime 的接口？

#### 原始提问

> 我觉得你说特别好 很有启发
>
> 1. surface 最好数据和逻辑分开 surface 只保存索引(数据) 但是维护索引的事情交给controller 否则surface会失控
> 2. Plugin 可以提供一个 present 表示它可以给出别人的接口 (比如 seat 给出 weapon 和 tank) 不过我对这个设计比较谨慎 我觉得需要非常小心 我会倾向一个plugin 只有他自己的runtime 而有些别的接口 可能是他周围的runtime 提供的 比如seat 如果要提供 weapon 和 vehicle 感觉语义就开始复杂了 不过确实需要考虑如何让他们一个比较有序的方式初始化 但这点我确实没想好 比如vehicle 需要weapon 但是 两者似乎很难说是runtime 依赖

### A：

对，我也会对 `Seat provides Weapon/Vehicle` 这种设计保持警惕。因为一旦允许 Plugin 随意“代替别人提供 capability”，我们前面好不容易建立起来的：

> **Capability 属于真正拥有语义和状态的 Runtime**

就会被破坏。

我觉得这里需要区分一个非常重要的概念：

> **Dependency 和 Relationship 不是一回事。**

你举的 `Vehicle ↔ Weapon` 正好是最典型的例子。

---

### 1. `Vehicle has Weapon` 很可能根本不是 Runtime Dependency

例如：

```text
Tank
├── VehicleRuntime
├── Turret
│   └── Cannon
│       └── WeaponRuntime
└── GunnerSeat
```

这里：

```text
WeaponRuntime → VehicleRuntime
```

真的有必要吗？

很多情况下没有。

Weapon 自己只需要：

```text
WeaponRuntime
├── Fire
├── Ammo
├── Heat
└── Projectile
```

Vehicle 自己：

```text
VehicleRuntime
├── Movement
├── Health
└── Physics
```

它们只是因为：

```text
Cannon belongs to Tank
```

而在 Object Graph 上产生关系。

所以：

```text
Tank owns Cannon
```

并不意味着：

```text
TankRuntime depends on WeaponRuntime
```

更不意味着：

```text
WeaponRuntime depends on TankRuntime
```

这和我们之前“Object Tree ≠ Dependency Graph”的原则完全一致。

---

## 2. 我现在甚至会再增加一个词：Relation

这样我们有三种完全不同的连接：

```text
Ownership
─────────
Tank owns Turret
Turret owns Cannon

Dependency
──────────
Weapon requires IAmmoSource
Movement requires IPhysicsBody

Relation
────────
Seat controls Turret
Seat uses Cannon
Cannon mounted on Vehicle
Player equipped Weapon
```

第三种特别重要。

Relation 表示：

> **A 和 B 在当前游戏结构中有关联，但 A 的内部逻辑不因此依赖 B。**

这就是 Vehicle/Weapon/Seat 很多关系真正所属的位置。

---

## 3. Seat 不应该 `provide IWeapon`

比如我不太喜欢：

```text
GunnerSeat
├── provides IAimControl
├── provides IWeapon
└── provides IVehicle
```

因为你马上就会问：

> 为什么 Seat 提供 Weapon？

Seat 又不是 Weapon。

最后会演化成：

```text
Seat
├── Vehicle
├── Weapon
├── Aim
├── Camera
├── HUD
├── Player
├── Input
└── ...
```

Seat 变成一个 context god object。

---

## 4. Seat 更适合只保存 Relation

例如：

```text
GunnerSeat Runtime
──────────────────

occupant → Player
aimSource → Turret
weaponSource → Cannon
cameraMount → Optics
```

注意这里甚至未必是 capability。

它可以只是：

```text
ObjectRef<T>
```

或者：

```text
EntityRef
```

也就是**结构数据**。

例如：

```csharp
class SeatRuntime
{
    EntityRef aimSource;
    EntityRef weaponSource;
    EntityRef cameraSource;
}
```

Seat 没有假装自己实现：

```text
IAimControl
ITriggerControl
IAmmoStatus
```

它只是告诉外部 Composer：

> “和这个 seat 有关系的几个对象在这里。”

---

## 5. 然后 Surface Composer 才 dereference + resolve

例如：

```text
GunnerSeat
│
├── aimSource ───────→ Turret
├── weaponSource ────→ Cannon
└── vehicle ─────────→ Tank
```

Composer：

```text
Seat Relation
      ↓
resolve related runtimes
      ↓
collect requested capabilities
      ↓
ControlSurface
PresentationSurface
```

最终：

```text
ControlSurface
├── IAimControl     → TurretRuntime
└── ITriggerControl → CannonRuntime

PresentationSurface
├── IVehicleStatus  → TankRuntime
├── IAmmoStatus     → CannonRuntime
└── IAimStatus      → TurretRuntime
```

这样 capability ownership 完全没有被污染。

---

## 6. 甚至 Seat 不一定需要知道“我要提供哪些接口”

这是我现在更倾向的版本。

Seat 只描述：

```text
aimSource
weaponSource
vehicle
cameraSource
```

而不是：

```text
provides:
    IAimControl
    ITriggerControl
    IAmmoStatus
```

后者其实已经开始包含 Presentation policy。

前者只是游戏世界事实：

```text
这个座位控制这个炮塔
这个座位使用这门炮
这个座位属于这辆车
```

然后 Session 的 Surface Policy 决定：

```text
CurrentWeapon
    → find ITriggerControl
    → find IAmmoStatus

CurrentAimSource
    → find IAimControl
    → find IAimStatus

CurrentVehicle
    → find IVehicleStatus
```

职责会清楚很多。

---

## 7. 这样我们其实得到了非常漂亮的四种 Graph

之前我们有三种，现在我觉得 Relation Graph 值得明确加进来：

```text
① Ownership Graph
─────────────────
Tank
└── Turret
    └── Cannon


② Runtime Dependency Graph
──────────────────────────
CannonFire → IAmmoSource
CannonFire → IProjectileSpawner


③ Relation Graph
────────────────
GunnerSeat ─weapon→ Cannon
GunnerSeat ─aim───→ Turret
GunnerSeat ─vehicle→ Tank


④ Event Graph
─────────────
CannonFire
   ↓ WeaponFired
MuzzleFX
Audio
Recoil
```

然后 Surface 是：

> **根据当前 Relation Graph，对 Runtime Capability Graph 做的一次 projection。**

这句话我觉得已经相当接近我们这个 framework 的数学定义了。

---

## 8. 这也解决你说的“Vehicle 需要 Weapon，但是又不像 Runtime Dependency”

因为我们首先应该问：

#### Vehicle 的逻辑真的需要 Weapon 吗？

如果只是：

> Tank 上安装了一门 Cannon。

这是：

```text
Ownership / Relation
```

不是 dependency。

如果 Tank HUD 想显示 Cannon ammo：

```text
Tank → Weapon
```

也不是 dependency。

是：

```text
Session Presentation Composer
    ↓
CurrentVehicle relation
    ↓
CurrentWeapon
    ↓
IAmmoStatus
```

Tank 本身完全不知道。

---

## 9. 只有真的存在语义依赖时，才成为 Runtime Dependency

例如有一个 Vehicle Fire Control Computer：

```text
FireControlComputer
```

它确实需要：

```text
IAimControl
IWeaponBallistics
IVehicleMotion
```

那么：

```text
FireControlComputer
      │
      ├── requires IAimControl
      ├── requires IWeaponBallistics
      └── requires IVehicleMotion
```

这就是合法 dependency。

因为没有这些 capability，它的逻辑确实无法运行。

区别非常简单：

> **“没有 B，A 的逻辑还能成立吗？”**

如果不能：

```text
Dependency
```

如果能，只是世界结构上有关：

```text
Relation
```

这个判断标准非常实用。

---

## 10. 初始化也因此可以分阶段

你说“怎么有序初始化”其实可以通过分阶段解决，而不是通过 dependency 强行解决。

我会考虑：

```text
Phase 1 — Construct
───────────────────
创建所有 Object / Runtime

Tank
Turret
Cannon
Seat


Phase 2 — Install
─────────────────
Plugin 安装自己的 capability

Tank   → IVehicleStatus
Turret → IAimControl
Cannon → ITriggerControl
Seat   → ISeatControl


Phase 3 — Resolve Dependencies
──────────────────────────────
真正 runtime dependency

CannonFire → IAmmoSource
TurretAim  → IRotationMotor


Phase 4 — Resolve Relations
───────────────────────────
建立 object references

Seat.weapon → Cannon
Seat.aim    → Turret
Seat.vehicle→ Tank


Phase 5 — Validate
──────────────────
dependency valid?
relation valid?


Phase 6 — Seal / Activate
─────────────────────────
开始运行
```

关键是：

> **Relation resolution 可以晚于 Runtime construction。**

这样就不需要因为：

```text
Seat references Weapon
Weapon belongs to Vehicle
Vehicle contains Seat
```

产生所谓 circular dependency。

因为这根本不是 dependency cycle。

只是 object graph 有互相关系。

---

## 11. 这也是为什么我现在不太想让 Runtime 直接保存大量其他 Runtime

例如：

```csharp
class VehicleRuntime
{
    WeaponRuntime weapon;
    SeatRuntime seat;
    TurretRuntime turret;
}
```

这很容易重新回到传统 OO object graph。

更倾向：

```text
Runtime
    ↓
只保存自己的 state + required capabilities

Relation
    ↓
由 Object/Definition/Binding 层描述
```

例如：

```yaml
Tank:
  children:
    turret:
      type: Turret

    cannon:
      type: Cannon

    gunnerSeat:
      type: Seat
      relations:
        weapon: ../cannon
        aim: ../turret
```

这已经很像 BF2 ObjectTemplate 的组合能力，但我们把**语义 dependency 和结构 relation 分开了**。

---

### 我觉得你第一点也可以稍微修改成一个很漂亮的原则

你说：

> Surface 只保存索引，维护索引交给 Controller。

我会稍微改成：

> **Surface 只保存 resolved references；Controller 维护 Context；Composer 根据 Context + Relation 生成 Surface。**

也就是：

```text
        Context
           │
           │ maintained by
           ▼
      Controller
           │
           ▼
    SurfaceComposer
       ▲        ▲
       │        │
   Relations  Runtime Capabilities
       │        │
       └───┬────┘
           ▼
        Surface
           │
     resolved refs only
           │
           ▼
      HUD / Camera
```

这样 Surface 连“索引维护”都不知道。

---

我觉得 **Relation** 这个概念可能正好补上了我们现在框架缺失的一块，而且它不会让框架明显复杂化。相反，它是在告诉我们：**不要把所有连接都塞进 Runtime Dependency。**

最终 Plugin 的纪律可以非常严格：

```text
Plugin
├── owns its Runtime
├── provides its own Capabilities
├── requires genuine Dependencies
├── may declare Relations to other Objects
└── may publish/consume scoped Events
```

尤其是这条可以作为硬规则：

> **Plugin 不应该 re-export 别人的 capability；需要跨对象组合能力时，由 Relation + Composer 在更高层做 projection。**

我觉得这会有效阻止 Seat、Vehicle、Weapon 逐渐演化成“什么都能提供”的超级 Runtime。

### 整理说明

1. 本篇保留了关于 Surface 只保存索引、Controller 维护索引，以及 present 设计需要谨慎限制的完整讨论。
2. 核心判断是：Surface 更接近对外 contract 或数据投影，不应因为方便访问而成为隐藏的状态管理器。
3. Plugin 通过 present 代替其他 Runtime 提供能力时，必须明确语义拥有者、生命周期和依赖方向；否则会把 Seat、Weapon、Vehicle 之间的关系重新混成隐式引用。
4. 这篇作为 DSH UI → Surface 讨论链的最后一篇，收束前面关于 UI Host、HUD Slot、Presentation Surface 和 Runtime View 的讨论。
