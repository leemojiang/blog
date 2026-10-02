---
title: "Super Runtime 与 Runtime View"
date: 2026-09-21
weight: 6
draft: false
math: true
tags:
  - "战地 2"
  - "游戏架构"
  - "组件系统"
  - "插件架构"
  - "Unity"
categories:
  - "软件架构"
---

本篇继续讨论 Scoped Runtime 的主要风险：Runtime 可能逐步膨胀为所有使用者都能访问的 Super Runtime。核心解决方案是把完整 Runtime 与使用者获得的 Runtime View 分开。

## Q：如何避免 Weapon Runtime 演化成 Super Runtime？除了接口统一和依赖树剪裁，还有哪些办法？

### 原始提问

> 确实 我觉得runtime 架构需要担心的就是变成super runtime 比如weapon 的runtime里面 有了10几种comp (就像战地2里面实际的情况 weapon 下面可以挂n种comp. 然后产生super runtime 这是要避免的.
>
>
>
> 我理解你刚才提的两种解决方法:
>
> 1. 用接口把多种类似的comp 统一起来
> 2. 依赖树剪裁 比如 weapon依赖的comp comp也有自己依赖的comp 但是weapon部管理这些 或者有些comp有自己的依赖 但是weapon不依赖它们. 比如FX 依赖event  来自weapon 但是weapon 不需要依赖FX (或者也可以反向设计?
>
>
>
> 还有哪些注意事项和解决办法呢?

## A：

对，你总结的两点都很准确。不过我会再做一个小修正：**真正要避免的不是 Runtime 里面“有十几个东西”，而是任何 Runtime 使用者都能看到、理解并操作这十几个东西。**

一个 Tank runtime 里存在 30 个 capability 本身不一定有问题；`Weapon` 能随手访问其中 30 个，才开始变成 Super Runtime。

所以我觉得可以把我们的设计原则进一步收敛成一句话：

> **Runtime 可以大，但 Runtime View 必须小。**

---

## 1. 你说的两个办法，可以分别理解成“横向压缩”和“纵向剪裁”

第一种接口抽象属于**横向压缩**：

```text
MagazineAmmo ──────┐
VehicleAmmoRack ───┼──► IAmmoSource
InfiniteAmmo ──────┘
```

Weapon 永远只看：

```text
IAmmoSource
```

而不是：

```text
MagazineAmmo
VehicleAmmoRack
AmmoCounter
AmmoStorage
AmmoLoader
...
```

第二种则是**纵向剪裁**。例如：

```text
Weapon
 │
 ├── IAmmoSource
 │       │
 │       └── AmmoRack
 │              └── Inventory
 │
 └── IProjectileLauncher
         │
         └── BallisticLauncher
                └── Physics
```

Weapon 不应该因此得到：

```text
Weapon
 ├── Ammo
 ├── Inventory
 ├── Ballistics
 └── Physics
```

它只看到：

```text
WeaponRuntimeView
 ├── IAmmoSource
 └── IProjectileLauncher
```

**依赖的依赖不是我的依赖。**

这条规则其实特别重要。

---

# 2. 第三个办法：区分“Required”和“Observed”

你举的 FX 正是最典型例子。

错误方向是：

```text
Weapon
 ├── Ammo
 ├── Projectile
 ├── FX
 ├── Audio
 └── CameraShake
```

更合理的是：

```text
              Weapon
              /    \
             ↓      ↓
          Ammo    Projectile
             \
              publish
                 ↓
           WeaponFired
          /     |      \
         ↓      ↓       ↓
        FX    Audio     AI
```

这里存在两个完全不同的关系：

```text
Weapon REQUIRES Ammo
FX OBSERVES WeaponFired
```

这意味着：

> **Observer 不应该出现在被观察者的 Runtime View 里。**

Weapon 根本不应该知道有没有 FX。

于是：

```text
Weapon without FX     ✓
Weapon with FX        ✓
Weapon with 5 FX      ✓
Dedicated Server      ✓
```

尤其最后一个非常漂亮。

Dedicated server 可以根本不安装：

```text
MuzzleFX
GunSound
CameraShake
```

Weapon 一行代码都不用变。

---

# 3. 你说“也可以反向设计？”——可以，但要看谁拥有语义

这是个很关键的判断。

例如：

```text
WeaponFired → FX
```

通常合理。

反过来：

```text
FX → Weapon
```

如果意思是 FX 主动调用：

```text
weapon.Fire()
```

通常就很可疑。

但如果：

```text
PlayerInput
     ↓
FireRequest
     ↓
Weapon
```

则完全合理。

判断方式可以非常简单：

> **谁拥有这个动作的业务语义？**

“能否开火”属于 Weapon：

```text
Weapon:
 ammo > 0?
 cooldown finished?
 chamber ready?
```

所以最终决定必须进入 Weapon。

而：

> “开火以后冒什么火光？”

不属于 Weapon。

所以：

```text
Weapon → event → FX
```

而不是：

```text
Weapon → FX
```

---

# 4. 第四个办法：Runtime 不提供“枚举所有东西”的能力

这个限制我觉得非常值得加。

不要让业务代码：

```csharp
runtime.GetAllComponents();
runtime.FindComponent(...);
runtime.Components;
```

否则迟早出现：

```csharp
foreach (var c in runtime.Components)
{
    if (c is AmmoComponent) ...
    if (c is PhysicsComponent) ...
}
```

整个抽象瞬间被绕过去。

Runtime 最好只有：

```text
require<T>()
optional<T>()
many<T>()
events
```

甚至正常业务代码连这些都不要直接拿到，而是 Builder 注入一个裁剪好的 View。

---

# 5. Runtime View 最好成为真正的一等公民

例如 Weapon 明确声明：

```text
Weapon requires:

IAmmoSource
IProjectileLauncher
IWeaponClock
```

Builder 从完整 Runtime：

```text
Cannon Runtime
├── Ammo
├── Projectile
├── Physics
├── Damage
├── Heat
├── Animation
├── FX
├── Audio
├── Network
├── Transform
├── Team
└── ...
```

裁剪出：

```text
WeaponView
├── ammo
├── launcher
├── clock
└── events.publish<WeaponFired>
```

然后真正给 Weapon 的甚至不是：

```csharp
Runtime runtime;
```

而是概念上的：

```csharp
WeaponContext context;
```

这样 **Super Runtime 在物理上存在，但在语义上不存在**。

这一点很像操作系统：

> 一个进程运行在拥有海量资源的 OS 上，不意味着这个进程应该拥有所有资源的访问权限。

---

# 6. 第五个办法：读和写也应该剪裁

这个是非常容易被忽略的。

假设：

```text
IHealth
```

提供：

```csharp
int CurrentHealth;
void SetHealth(int x);
void Kill();
void Revive();
void SetInvincible();
```

那实际上这个 interface 已经太大。

AI 可能只需要：

```text
IHealthReader
    Health
    IsAlive
```

Damage system 才需要：

```text
IDamageReceiver
    ApplyDamage(...)
```

Respawn system：

```text
IRespawnable
    Respawn()
```

所以不要因为“都是 Health”就做：

```text
IHealthGodInterface
```

这其实就是 Interface Segregation Principle 在游戏 runtime 里的具体应用。

---

# 7. 第六个办法：State 和 Behavior 也不要随便混

例如 Ammo 很容易写成：

```text
AmmoComponent
├── CurrentAmmo
├── Reload()
├── Consume()
├── SpawnMagazine()
├── PlayReloadAnimation()
├── PlayReloadSound()
└── UpdateUI()
```

又开始膨胀。

更干净可能是：

```text
AmmoState
    current
    capacity

AmmoSource
    consume()

ReloadBehavior
    reload()

Reloaded event
       │
       ├── Animation
       ├── Audio
       └── UI
```

不是为了疯狂拆 Component，而是：

> **核心状态变化与外围 reaction 分开。**

这样 dependency graph 会小很多。

---

# 8. 第七个办法：特别警惕“方便型依赖”

这是 Super Runtime 最容易慢慢长出来的地方。

第一天：

```text
Weapon requires Ammo
```

合理。

第二天：

> 我要判断阵营。

于是：

```text
Weapon requires Team
```

第三天：

> 我要播放声音。

```text
Weapon requires Audio
```

第四天：

> 我要做 UI 提示。

```text
Weapon requires UI
```

半年以后：

```text
WeaponContext
├── Ammo
├── Physics
├── Team
├── Audio
├── UI
├── Network
├── Player
├── Camera
├── Animation
└── World
```

所以每增加 dependency 都应该问：

> **这是 Weapon 完成其核心 invariant 必须知道的吗？**

如果不是，很可能应该变成 Event、Command 或外围 System。

---

# 9. 可以给 Dependency 设置“预算”

这个听起来有点机械，但工程上其实很好用。

例如规定：

> 一个 Behavior 正常应该直接依赖 2–5 个 capability。

不是说超过 5 个就违法，而是：

```text
Weapon requires 3          → 正常
VehicleMovement requires 4 → 正常
AircraftController requires 7 → 值得检查
Weapon requires 14         → architecture smell
```

如果出现 14 个，通常说明至少一个问题：

```text
职责太大
Interface 太细碎
外围 reaction 被拉进核心
缺少中间 abstraction
```

这可以直接成为 lint/validation warning。

---

# 10. 还有一个非常强的办法：Facade Capability

有时候 dependency 多并不是 Component 职责太大，而是底层 capability 太碎。

例如 Engine：

```text
requires:
    IFuel
    IGearbox
    IClutch
    IWheelTorque
    IRPM
    ITemperature
```

上层 Vehicle 不应该全部知道。

可以产生：

```text
             Vehicle
                │
                ▼
           IDriveTrain
                │
       ┌────────┼────────┐
       ↓        ↓        ↓
     Engine   Gearbox   Wheels
       ↓
      Fuel
```

Vehicle 只看到：

```text
IDriveTrain
```

这就是 Facade Pattern。

所以前面说：

> dependency 的 dependency 不是我的 dependency

进一步可以变成：

> **一组经常共同出现的底层能力，应该考虑形成更高层的语义 capability。**

---

# 11. Runtime 最好形成“能力层级”，而不是一个平面大字典

不要最终变成：

```text
Runtime
├── IAmmo
├── IPhysics
├── ITransform
├── IAudio
├── IFX
├── ITeam
├── INetwork
├── IDamage
├── IInventory
├── IMovement
├── IInput
├── ...
```

更自然的是：

```text
World Runtime
│
├── Time
├── Spawn
├── Network
│
└── Vehicle Runtime
    │
    ├── Physics
    ├── Damage
    ├── DriveTrain
    │
    └── Weapon Runtime
        │
        ├── Ammo
        ├── FireControl
        └── Projectile
```

但再次强调：

**这是 scope / ownership hierarchy，不代表调用链。**

---

# 12. 生命周期也可以帮助防止 Runtime 失控

Extension 最好只能在特定阶段修改 Runtime：

```text
Build
  ↓
Install
  ↓
Resolve
  ↓
Validate
  ↓
Seal
  ↓
Run
```

关键是：

### `Seal`

游戏运行以后：

```text
runtime.Register<X>()
runtime.Remove<X>()
```

默认禁止。

Runtime structure 固定。

这样业务逻辑只操作 capability，不会一边跑一边偷偷改变依赖图。

需要动态行为的话，可以动态改变：

```text
state
enabled
ownership
subscription
```

而不是随意修改 architecture graph。

---

# 13. Plugin 和 Runtime Extension 也最好是“安装后消失”

这个思想我很喜欢。

例如：

```text
MagazineExtension.Install(runtime)
```

完成：

```text
register IAmmoSource
register state
register lifecycle
```

以后 gameplay 根本不需要：

```text
MagazineExtension
```

本身。

也就是说：

```text
Extension
   │
   │ build time
   ▼
Runtime capability
```

而不是：

```text
Runtime
    ↓
Extension
    ↓
Extension
    ↓
Extension
```

否则 Plugin 本身最后又变成 runtime object graph。

---

# 14. 最终我会把关系限制成很少的几种

这是防止复杂度爆炸最有效的方式之一。

允许：

```text
A requires Capability B
A publishes Event E
A consumes Event E
A requests World Service C
Object owns Child Object
Extension installs Capability
```

尽量不允许：

```text
Component A references Component B
Component A searches Object Tree
Component A gets Owner and manipulates it
Component A knows all siblings
Component A holds arbitrary Runtime
```

于是整个游戏虽然可能有几百种 extension，关系的**语法**却始终只有五六种。

这正是好的 framework 最有价值的地方。

---

## 我现在会把我们的架构压缩成这样

```text
                    Object Tree
                ownership / scope
                       │
                       ▼
                 Scoped Runtime
                       │
            ┌──────────┼──────────┐
            ▼          ▼          ▼
       Capability    Events    World Services
            ▲          ▲          ▲
            │          │          │
       ┌────┴──────────┴──────────┴────┐
       │                               │
 Runtime Extensions              Runtime Users
   install things                consume views
       │                               │
       └──────────── Builder ──────────┘
                       │
                  dependency
                  validation
                       │
                      Seal
                       │
                      Run
```

其中真正对抗 **Super Runtime** 的不是某一个技巧，而是五道防线：

**接口聚合**防止具体实现泄漏；**依赖传递剪裁**防止 dependency-of-dependency 泄漏；**Event 解耦**防止外围 reaction 进入核心；**Runtime View** 防止使用者看到完整 Runtime；**Facade Capability** 防止底层能力过度碎片化。

再加上一个很实用的判断标准：

> **如果删掉某个模块以后，核心对象仍然可以正确完成它的领域职责，那么核心对象通常不应该依赖那个模块。**

比如删掉 `MuzzleFX`，Weapon 仍然能正确开火 → Weapon 不依赖 FX。

删掉 `Audio`，仍然能开火 → 不依赖 Audio。

删掉 `AmmoSource`，已经无法判断和执行正常开火 → 这是 Required Capability。

这个规则对于决定“到底应该 dependency 还是 event”特别好用。

我觉得我们现在其实已经逐渐从“模仿 BF2 Component System”走到了更有意思的东西：**保留 BF2 极强的组合能力，但通过 Runtime View/Capability/Event 把 BF2 内部 C++ Core 里我们看不到、也可能并不够干净的依赖关系重新设计一遍。**

## 整理说明

1. 本篇保留原始回答关于 Required/Observed、Runtime View、读写能力剪裁、依赖预算、Facade Capability、Runtime Seal 和关系语法约束的完整结构。
2. “Runtime 可以大，但 Runtime View 必须小”是本篇的中心原则；它强调的是使用者可见的能力面，而不是 Runtime 内部实际存储的对象数量。
3. 删掉模块后核心领域职责是否仍然成立，是判断 Required Capability、Event 和外围系统的实用标准。
4. Dependency budget 更适合作为架构预警和 lint 规则，而不是绝对的硬限制；不同领域对象的合理依赖数量需要结合职责和接口粒度判断。
