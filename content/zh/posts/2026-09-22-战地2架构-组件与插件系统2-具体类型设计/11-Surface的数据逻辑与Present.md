---
title: "Surface 的数据、逻辑与 Present"
date: 2026-09-22
weight: 11
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

本篇是这组讨论的最后一篇，整理 Surface 只保存索引、Controller 负责维护索引，以及 Plugin 的 present 能力为什么需要谨慎限制。

## Q：Surface 是否应该把数据和逻辑分开？Plugin 是否应该通过 present 提供其他 Runtime 的接口？

### 原始提问

> 我觉得你说特别好 很有启发
>
> 1. surface 最好数据和逻辑分开 surface 只保存索引(数据) 但是维护索引的事情交给controller 否则surface会失控
> 2. Plugin 可以提供一个 present 表示它可以给出别人的接口 (比如 seat 给出 weapon 和 tank) 不过我对这个设计比较谨慎 我觉得需要非常小心 我会倾向一个plugin 只有他自己的runtime 而有些别的接口 可能是他周围的runtime 提供的 比如seat 如果要提供 weapon 和 vehicle 感觉语义就开始复杂了 不过确实需要考虑如何让他们一个比较有序的方式初始化 但这点我确实没想好 比如vehicle 需要weapon 但是 两者似乎很难说是runtime 依赖

## A：

对，我也会对 `Seat provides Weapon/Vehicle` 这种设计保持警惕。因为一旦允许 Plugin 随意“代替别人提供 capability”，我们前面好不容易建立起来的：

> **Capability 属于真正拥有语义和状态的 Runtime**

就会被破坏。

我觉得这里需要区分一个非常重要的概念：

> **Dependency 和 Relationship 不是一回事。**

你举的 `Vehicle ↔ Weapon` 正好是最典型的例子。

---

## 1. `Vehicle has Weapon` 很可能根本不是 Runtime Dependency

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

# 2. 我现在甚至会再增加一个词：Relation

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

# 3. Seat 不应该 `provide IWeapon`

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

# 4. Seat 更适合只保存 Relation

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

# 5. 然后 Surface Composer 才 dereference + resolve

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

# 6. 甚至 Seat 不一定需要知道“我要提供哪些接口”

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

# 7. 这样我们其实得到了非常漂亮的四种 Graph

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

# 8. 这也解决你说的“Vehicle 需要 Weapon，但是又不像 Runtime Dependency”

因为我们首先应该问：

### Vehicle 的逻辑真的需要 Weapon 吗？

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

# 9. 只有真的存在语义依赖时，才成为 Runtime Dependency

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

# 10. 初始化也因此可以分阶段

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

# 11. 这也是为什么我现在不太想让 Runtime 直接保存大量其他 Runtime

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

## 我觉得你第一点也可以稍微修改成一个很漂亮的原则

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

## 整理说明

1. 本篇保留了关于 Surface 只保存索引、Controller 维护索引，以及 present 设计需要谨慎限制的完整讨论。
2. 核心判断是：Surface 更接近对外 contract 或数据投影，不应因为方便访问而成为隐藏的状态管理器。
3. Plugin 通过 present 代替其他 Runtime 提供能力时，必须明确语义拥有者、生命周期和依赖方向；否则会把 Seat、Weapon、Vehicle 之间的关系重新混成隐式引用。
4. 这篇作为 DSH UI → Surface 讨论链的最后一篇，收束前面关于 UI Host、HUD Slot、Presentation Surface 和 Runtime View 的讨论。
