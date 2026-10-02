---
title: "从 Plugin 到 Runtime 的组件简化"
date: 2026-09-21
weight: 5
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

本篇讨论能否借鉴 DSH/Cordis 的 Plugin 思路，把 Component 和 Object 简化为 Runtime 的组成部分，同时保留游戏对象树与语义依赖之间的区分。

## Q：能否参考 DSH 的设计模式，把 Component 和 Object 简化为 Runtime 的组成部分？

### 原始提问

> 真的很精妙 我有一种恍然大悟的感觉 这么说我感觉 deepseek harness 的 plugin 模式其实是更简化了一部分,也就是说plugin 只提供runtime. 而Component设计模式则要复杂一点 component 和object 有了自己的语义,甚至comp之间还有区别,而不是简单的runtime.
>
> 那我们能不能参考DSH 的设计模式 进一步简化comp 和 obj. 也就是说所有的comp 或者obj 都是runtime的一部分,相当于依赖这个runtime的去访问所有的runtime,这样只要解决好依赖,逻辑关系似乎变得更清晰了. 同时保证了所有依赖关系按照单层依赖 (runtime 逐步扩大) 并且将所有的复杂度集合在了runtime 的使用者上面 避免了交叉引用/环形引用
>
> 不过问题在于这种设计模式会不会不是很适合游戏这个环境(比如plugin 是平等的,而游戏obj 有更复杂的比如树状的结构,但这种结构表示的是游戏对象的关系 而不是它们语义和逻辑上的关系 
>
> 你觉得我们可以按照这个思路简化comp系统吗?

## A：

可以，而且我觉得你这个方向可能比我们上一轮讨论的“Capability + Port + Event + Command + Scope”更适合作为**第一版核心模型**。

上一版虽然干净，但已经开始有一点“为了治理复杂度而创造很多概念”的趋势。DSH/Cordis 那种 Runtime 模式给了一个更激进的方向：

> **不要让 Component 之间建立依赖关系。所有 Component/Object 都只依赖它所属的 Runtime。**

这样很多问题确实会突然消失。

不过我会做一个关键修正：**Object Tree 和 Runtime Dependency Graph 必须是两张完全不同的图。**

---

## 1. 可以把 DSH 的思想直接搬过来

先考虑最简单的模型：

```text
Runtime
│
├── Physics
├── Movement
├── Engine
├── Weapon
├── Ammo
├── FX
└── Audio
```

每一个模块都：

```text
Plugin/Component
      │
      └── depends on Runtime
```

而不是：

```text
Weapon ──→ Ammo
   │         ↑
   ↓         │
  FX ──→ Physics
```

于是 Weapon：

```csharp
class Weapon
{
    Runtime runtime;

    void Fire()
    {
        runtime.ammo.Consume();
        runtime.projectile.Spawn(...);
        runtime.emit(new WeaponFired(...));
    }
}
```

这看起来像 Service Locator，通常 Service Locator 会被批评为隐藏依赖。

**但 DSH 式 Runtime 和无约束的 Service Locator 有一个很重要的区别：Runtime 本身可以是一个经过构建、验证、有生命周期和扩展规则的 capability surface。**

所以可以继续约束：

```text
Weapon
   │
   ▼
WeaponRuntime
   │
   ├── Ammo
   ├── Projectile
   ├── Physics
   └── Events
```

Weapon 永远不直接持有：

```text
Weapon → AmmoComponent
Weapon → PhysicsComponent
```

---

# 2. 最漂亮的地方：依赖图真的可以坍缩

原来：

```text
          Weapon
         /   |   \
        ↓    ↓    ↓
      Ammo Physics FX
       ↓     ↑     ↓
      ... ← Movement
```

现在：

```text
         Runtime
       ↗ ↑ ↑ ↑ ↖
      /  | | |  \
 Weapon Ammo FX Physics Movement
```

或者从代码依赖方向看：

```text
Weapon ────────┐
Ammo ──────────┤
Physics ───────┼──→ Runtime Contract
Movement ──────┤
FX ────────────┘
```

这有个非常强的性质：

> **模块之间从结构上无法形成 compile-time circular dependency。**

因为规则就是：

```text
Component → Runtime

禁止：

Component → Component
```

这个约束非常有价值。

---

# 3. 但是游戏和 DSH 最大的区别正是你发现的：存在 Object Tree

DSH plugin 大体上可以理解成：

```text
Runtime
├── Plugin A
├── Plugin B
├── Plugin C
└── Plugin D
```

比较扁平。

游戏却是：

```text
Tank
├── Hull
│   ├── Physics
│   └── Engine
│
├── Turret
│   ├── Rotation
│   └── Cannon
│       ├── Weapon
│       └── Ammo
│
└── Camera
```

但这里有一个非常重要的认识：

**这棵树根本不需要表示软件依赖。**

它只表示：

```text
Ownership
Transform
Lifetime
Spatial relationship
Serialization
```

例如：

```text
Tank/Turret/Cannon
```

并不意味着：

```text
Cannon → Turret → Tank
```

的软件依赖。

这两个东西应该彻底解耦。

---

# 4. 所以我会把整个模型压缩成两个核心概念

甚至不是之前的六七种。

只有：

### Object

负责**结构**：

```text
Object
├── parent
├── children
├── transform
└── runtime
```

### Runtime

负责**语义**：

```text
Runtime
├── capabilities
├── state
├── events
├── commands
└── lifecycle
```

于是：

```text
             Object Tree
                 │
       ┌─────────┼─────────┐
       ▼         ▼         ▼
     Hull      Turret    Cannon
       │         │         │
       ▼         ▼         ▼
    Runtime   Runtime    Runtime
```

这里开始变得非常漂亮。

---

# 5. Component 甚至可以从“核心概念”里面消失

这可能是最有意思的一步。

以前：

```text
Object
   ↓
Components
   ↓
Capabilities
```

现在可以变成：

```text
Object
   ↓
Runtime
```

所谓 Component 只是：

> **向 Runtime 安装能力的模块。**

例如：

```text
Tank Runtime
```

最开始：

```text
Runtime {}
```

安装 Physics：

```text
Runtime {
    physics
}
```

安装 Movement：

```text
Runtime {
    physics
    movement
}
```

安装 Damage：

```text
Runtime {
    physics
    movement
    damage
}
```

所以 DSL：

```text
Tank
{
    Physics
    Movement
    Damage
}
```

实际上不是：

> Tank 拥有三个神圣的“Component 对象”。

而是：

> 三个 plugin/module 逐步构造 Tank Runtime。

这就非常 DSH 了。

---

# 6. `FireComp` / `AmmoComp` 的困惑也因此消失很多

我们不用再纠结：

> FireComp 是 Behavior 还是 Capability？

它就是一个 Runtime Extension：

```text
install Fire
```

执行：

```text
runtime.fire = FireRuntime(...)
```

Ammo：

```text
install Ammo
```

执行：

```text
runtime.ammo = AmmoRuntime(...)
```

FX：

```text
install WeaponFX
```

执行：

```text
runtime.on<WeaponFired>(...)
```

所以：

```text
Fire
Ammo
FX
Physics
```

**安装方式是统一的，但安装结果并不需要具有相同语义。**

这正好继承了 BF2 “Comp 都是 composition unit”的简单性，又不要求它们实际上是同一种东西。

---

# 7. 而且“逐步扩大的 Runtime”非常有意思

这里甚至可以借鉴 TypeScript plugin augmentation 的感觉。

初始：

```text
Runtime<>
```

Physics plugin：

```text
Runtime<
    Physics
>
```

Movement plugin 要求：

```text
requires Physics
```

安装之后：

```text
Runtime<
    Physics,
    Movement
>
```

Weapon：

```text
requires Physics
```

安装以后：

```text
Runtime<
    Physics,
    Movement,
    Weapon
>
```

因此 Build 本身就是：

```text
Empty Runtime
      │
      ▼
 Physics Plugin
      │
      ▼
Runtime<Physics>
      │
      ▼
 Movement Plugin
      │
      ▼
Runtime<Physics, Movement>
      │
      ▼
 Weapon Plugin
      │
      ▼
Runtime<Physics, Movement, Weapon>
```

你说的“**runtime 逐步扩大**”其实是一个非常好的抽象。

---

# 8. 这样 dependency 也变成 Plugin Installation Dependency

以前我们考虑：

```text
Weapon requires IPhysicsBody
Weapon requires IAmmoSource
```

现在可以进一步提升：

```text
WeaponPlugin:
    requires:
        Physics
        Ammo
```

Builder 做 topological sort：

```text
Physics
   ↓
Ammo
   ↓
Weapon
   ↓
WeaponFX
```

然后依次 install。

如果：

```text
A requires B
B requires A
```

直接：

```text
Build Error:
Circular runtime extension dependency.
```

所以环依赖在 object 创建的时候就死掉。

这非常漂亮。

---

# 9. 但是我不会完全删除 Capability / Port

这里是我觉得需要保留上一版设计的一小部分。

比如 Weapon 如果写：

```text
requires AmmoPlugin
```

其实又开始和具体实现绑定了。

更好：

```text
Weapon requires:
    IAmmoSource
```

Runtime 里面：

```text
runtime.ammo : IAmmoSource
```

至于谁提供：

```text
MagazinePlugin
InfiniteAmmoPlugin
VehicleAmmoRackPlugin
NetworkAmmoPlugin
```

Weapon 不关心。

所以最终可能是：

```text
Plugin
   │
   │ installs
   ▼
Runtime Capability
   │
   │ consumed by
   ▼
Runtime User
```

Capability 仍然存在，但它不再成为一套庞大的架构体系。

只是 Runtime 的 contract。

---

# 10. Event 也可以直接成为 Runtime 的基础设施

这样甚至不用 `FxComp → FireComp`。

```csharp
runtime.events.Subscribe<WeaponFired>(...);
```

Fire：

```csharp
runtime.events.Publish(new WeaponFired(...));
```

所以：

```text
Fire ──┐
       │
       ▼
    Runtime
       │
       ├── events
       │
       ├── ammo
       │
       ├── physics
       │
       └── projectile
       │
       ▼
      FX
```

还是：

```text
Component → Runtime ← Component
```

没有：

```text
Fire → FX
```

---

# 11. Parent dependency 也可以用 Runtime chaining，而不是 Component tree search

这是我觉得游戏环境下最关键的改造。

例如：

```text
Tank Runtime
├── physics
├── movement
└── health

      │ parent scope
      ▼

Turret Runtime
├── rotation
└── aim

      │ parent scope
      ▼

Cannon Runtime
├── ammo
└── fire
```

Cannon：

```text
CannonRuntime
      │
      ├── local ammo
      ├── local fire
      │
      └── parent
             ↓
        TurretRuntime
             │
             └── parent
                    ↓
               TankRuntime
```

这很像 lexical scope：

```text
{
    physics

    {
        aim

        {
            ammo
            fire
        }
    }
}
```

---

# 12. 不过这里必须小心：不要自动无限向 parent 搜索

否则：

```text
runtime.physics
```

到底来自：

```text
Cannon?
Turret?
Tank?
World?
```

又变成隐式魔法。

所以我会让 Runtime 有非常少而明确的访问方式：

```text
runtime.local<T>()
runtime.parent<T>()
runtime.root<T>()
runtime.world<T>()
```

或者更进一步：

```text
runtime.require<T>(Scope.Local)
runtime.require<T>(Scope.Root)
```

仍然保持显式。

---

# 13. 甚至 World 本身也可以是 Runtime

这样整个游戏形成一个很漂亮的 fractal：

```text
World Runtime
│
├── Time
├── PhysicsWorld
├── SpawnSystem
├── Network
├── Audio
│
└── Tank Object
      │
      └── Tank Runtime
           ├── PhysicsBody
           ├── Movement
           │
           └── Turret Object
                │
                └── Turret Runtime
                     ├── Aim
                     │
                     └── Cannon Object
                          │
                          └── Cannon Runtime
                               ├── Ammo
                               └── Fire
```

注意：

**树表示 ownership/scope。**

而：

```text
Runtime capabilities
```

表示语义。

两者终于完全分开。

---

# 14. 这可能比传统 Component System 更适合我们的目标

传统 Unity：

```text
GameObject
├── Component
├── Component
├── Component
└── Component
```

容易导致：

```text
Component ↔ Component
```

传统 ECS：

```text
Entity
├── Data
├── Data
└── Data

Systems operate globally
```

又可能不太适合 BF2 那种非常强的 hierarchical object composition。

我们现在这个东西反而处于中间：

```text
             Object
               │
          owns Runtime
               │
      ┌────────┼─────────┐
      ▼        ▼         ▼
 Capability  Events    State
      ▲        ▲         ▲
      └────────┼─────────┘
               │
        Runtime Extensions
```

我甚至不太想叫它 Component Architecture 了。

更接近：

> **Hierarchical Runtime Composition**

---

# 15. 这样 DSL 也会变得异常简单

例如：

```text
object Tank
{
    use Physics
    use Damage
    use TrackedMovement

    object Turret
    {
        use TurretRotation

        object Cannon
        {
            use Magazine
            use CannonFire
            use MuzzleFX
        }
    }
}
```

这里 `use` 的含义统一：

> **把一个 Runtime Extension 安装到这个 Object 的 Runtime。**

而不是：

```text
这是 Component
这是 Behavior
这是 Provider
这是 Adapter
这是 Handler
```

这些 distinction 仍然存在于 extension 内部，但**不污染 object model 和 DSL**。

这点我非常喜欢。

---

# 16. Plugin Registry 也正好可以统一进来

Core 只需要：

```text
ExtensionRegistry
```

例如：

```text
"Physics"        → PhysicsExtension
"Damage"         → DamageExtension
"Magazine"       → MagazineExtension
"CannonFire"     → CannonFireExtension
"MuzzleFX"       → MuzzleFXExtension
```

DSL：

```text
use CannonFire
```

Builder：

```text
Registry
   ↓
CannonFireExtension
   ↓
check requirements
   ↓
install(runtime)
   ↓
runtime.fire
```

这就和我们之前喜欢的 DSH/Cordis 模型真正统一了。

---

## 但我会加一条非常重要的限制

**Runtime 不能成为一个巨大的 God Object。**

这是这个设计最大的风险。

如果最后所有代码都是：

```csharp
runtime.physics...
runtime.audio...
runtime.weapon...
runtime.ammo...
runtime.player...
runtime.world...
runtime.network...
runtime.ui...
runtime.ai...
runtime.whatever...
```

我们只是把 spaghetti 从：

```text
Component ↔ Component
```

变成了：

```text
Everything ↔ Runtime ↔ Everything
```

所以 Runtime 应该更像：

> **受约束的 capability namespace / context**

而不是：

> **万能对象。**

Extension 应该声明：

```text
CannonFire

requires:
    Local<IAmmoSource>
    Root<IPhysicsBody>
    World<IProjectileService>

provides:
    Local<IFireControl>

publishes:
    WeaponFired
```

Builder 验证以后，再给 CannonFire 一个**裁剪后的 Runtime View**：

```text
CannonFireContext
├── ammo
├── physics
├── projectiles
└── events
```

而不是把整个 Runtime 裸着塞进去。

这样我们就同时拿到了两个世界的优点：

```text
       DSH-style simplicity
               │
      everything → Runtime
               │
               ▼
       Runtime Composition
               │
        ┌──────┴───────┐
        ▼              ▼
 Explicit contract   Object scope
        │              │
        └──────┬───────┘
               ▼
       Game architecture
```

所以如果现在让我重新设计我们前面那套方案，我会明显简化成：

**Object Tree + Scoped Runtime + Runtime Extension + Capability Contract + Event Bus + Registry。**

Command、Adapter、Provider、Behavior、Port 等概念仍然有价值，但它们可以退居为**Runtime Extension 内部的设计模式**，不需要全部成为 framework 一级公民。

我觉得这一步很关键：我们从 BF2 学到了 **Object/Comp composition**，从 Hexagonal Architecture 学到了**语义边界**，然后从 DSH 学到了另一件事——**框架本身不一定需要把所有语义边界都建模成不同类型；可以用一个受约束的 Runtime 把它们重新统一起来。**

这可能已经开始形成一套相当简洁、而且确实适合游戏的架构了。

## 整理说明

1. 本篇保留原始回答关于 Object Tree、Runtime、Runtime Extension、Capability、Event 和 Registry 的完整推导。
2. 最关键的修正是：Object Tree 表示 Ownership、Transform、Lifetime、Spatial relationship 和 Serialization；它不自动表示软件依赖。
3. “Component 是向 Runtime 安装能力的模块”是一种统一的安装模型，不代表 Fire、Ammo、FX、Physics 在语义上变成同一种东西。
4. 本篇已经指出 Super Runtime 风险，因此后续第 6 篇继续讨论 Runtime View、依赖剪裁和访问边界。
