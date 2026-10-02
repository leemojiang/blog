---
title: "Hexagonal Architecture 与组件语义"
date: 2026-09-21
weight: 4
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

本篇把 Hexagonal Architecture（六边形架构，也称 Ports and Adapters）引入 BF2 的 Component 讨论，用来区分 Behavior、Capability Provider、Engine Adapter 和 Event Consumer。

## Q：Hexagonal Architecture 是什么？BF2 中不同语义的 Component 应该如何区分？

### 原始提问

> Hexagonal Architecture  是什么? 我觉得你说的特别有启发 BF2 的comp/object 系统 似乎没有在命名上强调 能力 和通知之类的设计 (毕竟只有comp 的名称 但是如何实现的是在core部分 脚本部分是看不出来的 所以我感觉遇到了很多 又像是comp 又像是能力的comp 比如 firecomp 和 ammocomp 它们似乎和 挂载的武器类太接近了 但是有些fxcomp又太远了 如果把这些都当作comp来处理 确实会变得很混乱

## A：

对，你现在这个观察其实正好切到了 **Hexagonal Architecture（六边形架构）** 对我们这个问题最有价值的地方。

它并不是一种“怎么写 Component”的模式，而是在提醒我们：

> **不要把“一个东西是什么”和“它如何与外界交互”混成同一种抽象。**

这恰好可以解释为什么 BF2 里 `FireComp / AmmoComp / FxComp / PhysicsComp` 全都叫 Comp，会让我们逆向时感觉它们明明不是同一种东西。

---

## Hexagonal Architecture 到底是什么？

Hexagonal Architecture 也叫 **Ports and Adapters Architecture**，核心其实非常简单。

假设我们有一个武器：

```text
                 Ammo System
                     │
                     ▼
                  [ Port ]
                     │
                     ▼
            ┌────────────────┐
            │                │
 Input ───► │     Weapon     │ ───► Effects
            │                │
            └────────────────┘
                     ▲
                     │
                  [ Port ]
                     │
                     ▼
                  Physics
```

Weapon 本身不应该知道：

```text
Unity Rigidbody
Unity AudioSource
AmmoComponent
ParticleSystem
GameObject
```

它只知道自己需要几个**接口（Port）**：

```text
IAmmoSource
IProjectileSpawner
IAimProvider
```

真正把这些接口接到 Unity/BF2/网络/脚本上的东西叫 **Adapter**：

```text
                    IPhysicsBody
                         │
                       Port
                         │
               ┌─────────┴─────────┐
               │                   │
        RigidbodyAdapter     CustomPhysicsAdapter
               │                   │
        Unity Rigidbody      自定义物理系统
```

所以：

**Port = 我需要/提供什么能力**

**Adapter = 具体是谁帮我实现这个能力**

这就是最核心的思想。

---

# 这正好可以解决 BF2 的“所有东西都叫 Comp”的问题

假设我们逆向看到：

```text
Weapon
├── FireComp
├── AmmoComp
├── RecoilComp
├── PhysicsComp
├── FxComp
└── SoundComp
```

表面看：

> 六个都是 Component。

但从架构语义上看，它们其实可能属于完全不同的类别。

我会把它们拆成至少四种东西：

| BF2 看起来像 | 我们的架构语义 | 作用 |
|---|---|---|
| `FireComp` | Behavior | 实现武器行为 |
| `AmmoComp` | Capability Provider | 提供弹药能力 |
| `PhysicsComp` | Engine Adapter | 连接物理系统 |
| `FxComp` | Event Consumer | 响应事件 |
| `SoundComp` | Event Consumer | 响应事件 |
| `Weapon` | Entity/Object | 组合这些东西 |

这样一下就清楚很多。

---

# 关键是：Component 应该只是“物理容器”，不是架构语义

这可能是我们刚才讨论里值得进一步修正的一点。

我们之前一直说：

```text
Component
Component
Component
```

其实容易掉进 Unity/BF2 同样的坑。

更好的理解应该是：

```text
             Object
               │
      ┌────────┼───────────┐
      ▼        ▼           ▼
   Behavior  Provider    Adapter
      │        │           │
      ▼        ▼           ▼
     Port     Port       Engine
      │
      ▼
    Events
```

而 Unity `MonoBehaviour` 或 BF2 `Comp` 只是：

> **这些架构角色在 runtime 中的一种承载形式。**

也就是说：

```text
Component != Capability
Component != Behavior
Component != Service
Component != Event Handler
```

一个 Component **可以扮演其中一个或多个角色**。

这个区别很重要。

---

# 举一个完整的武器例子

假设 DSL 写：

```text
Object Cannon
{
    FireBehavior

    AmmoMagazine
    {
        capacity 30
    }

    ProjectileWeapon
    {
        projectile "120mm_AP"
    }

    MuzzleFlash
    GunSound
}
```

如果只是传统 Component 思维：

```text
Cannon
├── FireComponent
├── AmmoComponent
├── ProjectileComponent
├── MuzzleFlashComponent
└── SoundComponent
```

然后就开始互相：

```text
FireComponent → AmmoComponent
FireComponent → ProjectileComponent
FireComponent → MuzzleFlashComponent
FireComponent → SoundComponent
```

很快变成：

```text
              Fire
            ↙  ↓  ↘
         Ammo  FX  Sound
          ↑  ↘ ↓ ↙
       Projectile
```

Component spaghetti 出现了。

---

# Ports 思维完全不一样

FireBehavior 声明：

```text
requires:

IAmmoSource
IProjectileSpawner
```

所以：

```text
                 FireBehavior
                  /        \
                 /          \
                ▼            ▼
         IAmmoSource   IProjectileSpawner
             ▲                ▲
             │                │
      AmmoMagazine     ProjectileWeapon
```

FireBehavior 根本不知道：

```text
AmmoMagazine
ProjectileWeapon
```

存在。

它只知道：

> 我需要弹药。

> 我需要一个能发射 projectile 的东西。

---

# 那么 FX 和声音呢？

这里甚至**不应该存在 dependency**。

FireBehavior：

```text
Fire()
   │
   ├── Ammo.consume()
   │
   ├── Projectile.spawn()
   │
   └── publish WeaponFired
```

然后：

```text
                       WeaponFired
                            │
              ┌─────────────┼──────────────┐
              ▼             ▼              ▼
        MuzzleFlash     GunSound       CameraShake
```

于是 FireBehavior 完全不知道 FX。

这正好解释了你说的：

> `FxComp` 感觉离 Weapon 太远。

对。

因为从 architecture semantics 来说，**它本来就不是 Weapon 的核心依赖。**

它只是：

```text
WeaponFired
     ↓
Event Consumer
```

---

# 所以我们甚至可以建立一个“距离”的概念

我觉得这对于游戏架构特别有用。

一个对象内部的关系不是平等的。

可以想成：

```text
                     Weapon
                        │
               ┌────────┴────────┐
               │   Core Domain   │
               │                 │
               │ FireBehavior    │
               │ Ammo            │
               │ Projectile      │
               └────────┬────────┘
                        │
                     Events
                        │
        ┌───────────────┼──────────────┐
        ▼               ▼              ▼
       FX             Audio          Camera
                        │
                  Infrastructure
```

越靠中心：

```text
Fire
Ammo
Projectile
```

越属于：

**这个对象“是什么”。**

越往外：

```text
FX
Sound
UI
Telemetry
Camera shake
```

越属于：

**这个对象发生事情以后，世界如何表现/响应。**

这其实就是 Hexagonal Architecture 那个“六边形”的真正含义。

六边形本身没什么意义。

重要的是：

```text
          Infrastructure

       ┌─────────────────┐
       │                 │
       │     DOMAIN      │
       │                 │
       └─────────────────┘

          Infrastructure
```

**Domain 在里面，外部系统在外面。**

---

# 这也解释了 Physics 为什么让我们之前觉得别扭

你前面说：

> Physics Component 很接近 Unity 引擎原生，到底怎么划分？

用这个思路马上就清楚了。

Physics 不应该侵入：

```text
TankMovement
Weapon
Engine
```

而应该：

```text
              Domain
                 │
                 ▼
            IPhysicsBody
                 │
─────────────────┼────────────────
                 │
          Unity Adapter
                 │
                 ▼
             Rigidbody
```

因此：

```text
TankMovement
     │
     ▼
IPhysicsBody       ← Port
     ▲
     │
RigidbodyBody      ← Adapter
     │
     ▼
Unity Rigidbody
```

如果以后我们自己做车辆物理：

```text
IPhysicsBody
     ▲
     │
CustomVehicleBody
```

Domain 根本不变。

---

# 甚至 Input 都可以这样处理

传统 Unity：

```csharp
void Update()
{
    if (Input.GetKey(KeyCode.Space))
        weapon.Fire();
}
```

这实际上已经让 Weapon 的控制逻辑和 Unity Input 耦合。

更干净的是：

```text
Unity Input System
       │
       ▼
PlayerInputAdapter
       │
       ▼
   FireCommand
       │
       ▼
   FireBehavior
```

AI：

```text
AI Controller
      │
      ▼
 FireCommand
      │
      ▼
 FireBehavior
```

网络：

```text
Network
   │
   ▼
FireCommand
   │
   ▼
FireBehavior
```

于是非常漂亮：

```text
Player ───┐
AI ───────┼──► FireCommand ──► Weapon
Network ──┘
```

Weapon 根本不关心谁在操作它。

这对我们之前讨论的 **FSM / Agent** 尤其重要。

因为：

```text
Player
FSM
Behavior Tree
LLM Agent
Network Replay
```

全部只是不同 Adapter。

---

# 这样我们可以重新定义“Component”

我会考虑以后不要在 architecture document 里面笼统地说 Component，而是定义几个明确概念：

```text
Entity / Object
    │
    ├── Behavior
    │      └── 这个对象会做什么
    │
    ├── Capability Provider
    │      └── 这个对象能提供什么
    │
    ├── Adapter
    │      └── 怎么连接 Unity/外部系统
    │
    └── Event Handler
           └── 对其他事情如何响应
```

它们底层**完全可以全部实现成 MonoBehaviour**。

但架构意义不同。

例如：

```text
Cannon
│
├── FireBehavior        [Behavior]
│
├── Magazine            [Provider: IAmmoSource]
│
├── ProjectileLauncher  [Provider: IProjectileSpawner]
│
├── RigidbodyAdapter    [Adapter: IPhysicsBody]
│
├── MuzzleFlash         [Event Handler: WeaponFired]
│
└── GunSound            [Event Handler: WeaponFired]
```

这一张图就比：

```text
Cannon
├── FireComp
├── AmmoComp
├── ProjectileComp
├── PhysicsComp
├── FXComp
└── SoundComp
```

包含的信息多得多。

---

# 但不要把这些分类做成死板的 class hierarchy

这里又有一个潜在坑。

不要真的搞：

```csharp
class BehaviorComponent : Component
class ProviderComponent : Component
class AdapterComponent : Component
class EventComponent : Component
```

然后所有东西必须四选一。

因为现实中一个模块可能同时：

```text
Engine
├── Behavior
├── provides IPowerSource
├── requires IFuelSource
└── publishes EngineStarted
```

所以我更倾向：

```text
EngineComponent
```

声明自己的 contract：

```text
Provides:
    IPowerSource

Requires:
    IFuelSource
    IMovementDrive

Publishes:
    EngineStarted
    EngineStopped

Consumes:
    DamageReceived
```

于是 Component 是实现单位。

而真正的 architecture interface 是：

```text
Provides
Requires
Publishes
Consumes
```

这个设计我觉得非常强。

---

# 最后 DSL 甚至可以把这些关系暴露出来

未来我们的 definition 可以接近：

```text
Engine:
    type: CombustionEngine

    provides:
        power: IPowerSource

    requires:
        fuel: ./FuelTank
        drive: $root/Movement

Weapon:
    type: Cannon

    requires:
        ammo: ./Magazine
        physics: $root/Physics

    publishes:
        - WeaponFired
```

FX：

```text
MuzzleFlash:
    type: ParticleEffect

    consumes:
        - ../Weapon.WeaponFired
```

那么 DSL 描述的已经不只是：

> **有哪些 Component。**

而是在描述：

> **这个游戏对象内部有哪些模块，以及这些模块之间允许通过什么 contract 发生关系。**

这比单纯 `addComponent` 高一个抽象层。

---

所以我现在反而觉得，我们可以从 BF2 得出一个很有价值的“现代化改造”：

**BF2 的 Object/Comp 解决了“组合而不是继承”；Ports & Adapters 再进一步解决“组合以后彼此如何保持边界”。**

最终就变成：

```text
Object Tree
     +
Component Composition
     +
Capability Ports
     +
Explicit Wiring
     +
Commands
     +
Events
     +
Engine Adapters
     +
Plugin Registry
```

这几层不是重复造轮子，而是在分别解决**结构、能力、依赖、行为通信、引擎耦合和扩展性**六个不同问题。

这可能就是为什么你现在会感觉单纯“DI + Component”还差一点东西——**DI 主要解决“我怎么拿到依赖”，但它并不回答“这两者为什么应该依赖”“什么关系应该根本不是依赖”“哪些东西应该属于 Domain、哪些只是外围响应”。Hexagonal Architecture 恰恰是在回答后面这些问题。**

## 整理说明

1. 本篇保留原始回答的核心推导、武器示例、Port/Adapter 图示和 DSL 示例。
2. 六边形架构在这里不是要求每个 Component 都继承某个固定基类，而是要求 Domain 通过稳定的 Port 与外部系统交互，具体实现由 Adapter 提供。
3. FireComp、AmmoComp、PhysicsComp、FxComp 都可以仍然是底层承载单位，但它们承担的架构角色不同；Component 名称本身不足以表达依赖方向。
4. 本篇的“中心/外围”是依赖方向的解释工具，不等于所有外围模块都必须在物理目录或 Unity 层级上位于外侧。
