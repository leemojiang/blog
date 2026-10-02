---
title: "ISP 在游戏 Runtime 中的应用"
date: 2026-09-21
weight: 7
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

本篇回答如何在原生 C# 中实现 Interface Segregation Principle（接口隔离原则），并说明它与 Runtime View、Capability Interface 和 Component 数量之间的关系。

## Q：ISP 在游戏 Runtime 中如何具体实现？是否需要为每一种 Runtime View 创建一个新的 interface？

### 原始提问

> 非常棒 你刚才提到的 Interface Segregation Principle 在游戏 runtime 里的具体应用。 这个怎么具体实现? 我理解这个似乎是在不同runtime暴露不同interface 但是原生c#怎么支持这种语法? 我觉得不能对每一个可能的暴露新建一个interface吧?

## A：

对，你抓到了一个很实际的问题：**ISP（Interface Segregation Principle）不是让我们为每一种 Runtime View 的排列组合创建一个 interface。** 如果这么干：

```text
IWeaponRuntime
IWeaponWithPhysicsRuntime
IWeaponWithAmmoRuntime
IWeaponWithAmmoAndPhysicsRuntime
IWeaponWithAmmoPhysicsAndTeamRuntime
...
```

马上组合爆炸。

我们真正需要的是：

> **Capability 的接口数量有限，而 Runtime View 是这些 Capability 的动态/静态组合。**

C# 完全可以比较优雅地实现。

---

## 1. 先区分两个概念：Capability Interface 和 Runtime View

比如我们只有这些稳定的小接口：

```csharp
public interface IAmmoSource
{
    bool TryConsume(int amount);
}

public interface IPhysicsBody
{
    Vector3 Velocity { get; }
    void AddForce(Vector3 force);
}

public interface IProjectileSpawner
{
    void Spawn(ProjectileSpec spec);
}

public interface IClock
{
    float Time { get; }
}
```

这些接口不是为了 Weapon 创建的。

它们表达的是整个游戏中长期存在的**能力语义**：

```text
IAmmoSource
IPhysicsBody
IProjectileSpawner
IClock
IDamageReceiver
IAimTarget
IMovement
...
```

可能整个游戏最终只有几十种或者一百来种。

而 Weapon 需要：

```text
IAmmoSource
IProjectileSpawner
IClock
```

Movement 需要：

```text
IPhysicsBody
IClock
```

这时候**不需要**创建：

```csharp
IWeaponRuntime
IMovementRuntime
```

---

# 2. 最简单实现：构造时直接注入 Capability

其实最原始的 C# 就已经支持：

```csharp
public sealed class Weapon
{
    private readonly IAmmoSource ammo;
    private readonly IProjectileSpawner projectile;
    private readonly IClock clock;

    public Weapon(
        IAmmoSource ammo,
        IProjectileSpawner projectile,
        IClock clock)
    {
        this.ammo = ammo;
        this.projectile = projectile;
        this.clock = clock;
    }
}
```

Builder：

```csharp
var weapon = new Weapon(
    runtime.Require<IAmmoSource>(),
    runtime.Require<IProjectileSpawner>(),
    runtime.Require<IClock>()
);
```

于是：

```text
Full Runtime
│
├── Ammo
├── Physics
├── Projectile
├── Clock
├── Audio
├── FX
├── Team
└── Network

        ↓ Builder

Weapon
│
├── IAmmoSource
├── IProjectileSpawner
└── IClock
```

这实际上就是最严格的 Runtime View。

**Weapon 在语言层面甚至没有办法访问 Physics/Audio/Network。**

这非常强。

---

# 3. 但这样参数很多，所以可以有一个 Context

比如：

```csharp
public readonly struct WeaponContext
{
    public readonly IAmmoSource Ammo;
    public readonly IProjectileSpawner Projectile;
    public readonly IClock Clock;

    public WeaponContext(
        IAmmoSource ammo,
        IProjectileSpawner projectile,
        IClock clock)
    {
        Ammo = ammo;
        Projectile = projectile;
        Clock = clock;
    }
}
```

然后：

```csharp
class Weapon
{
    private WeaponContext ctx;

    public void Fire()
    {
        if (!ctx.Ammo.TryConsume(1))
            return;

        ctx.Projectile.Spawn(...);
    }
}
```

这就是我前面说的：

> Runtime View

但这里确实有你的问题：

**每个 Behavior 都定义 Context，会不会又很多？**

会。

所以小型模块我反而不会这么做。

---

# 4. 更 DSH 风格的方法：Typed Capability Resolver

Runtime 内部可以是：

```csharp
public interface IRuntime
{
    T Require<T>();
}
```

Weapon：

```csharp
class Weapon
{
    private readonly IAmmoSource ammo;
    private readonly IProjectileSpawner projectile;

    public Weapon(IRuntime runtime)
    {
        ammo = runtime.Require<IAmmoSource>();
        projectile = runtime.Require<IProjectileSpawner>();
    }
}
```

这样方便很多。

但是这里存在我们刚才讨论的危险：

```csharp
runtime.Require<IAudio>();
runtime.Require<INetwork>();
runtime.Require<IUI>();
runtime.Require<IWhatever>();
```

Weapon 实际拿到了 Super Runtime 的钥匙。

所以我不太喜欢把完整 `IRuntime` 长期保存在：

```csharp
this.runtime = runtime;
```

更好的办法是：

```csharp
Weapon(IRuntime runtime)
{
    ammo = runtime.Require<IAmmoSource>();
    projectile = runtime.Require<IProjectileSpawner>();

    // constructor 结束后不保留 runtime
}
```

也就是：

> **Runtime 只在 wiring 阶段存在，运行阶段只保留 Capability。**

这个模式非常简单，我其实挺推荐第一版这样做。

---

# 5. 再高级一点：用泛型表达 Runtime Requirement

如果我们想让 Builder 知道 Weapon 的依赖，可以：

```csharp
public interface IRequires<T>
{
}
```

然后：

```csharp
class Weapon :
    IRequires<IAmmoSource>,
    IRequires<IProjectileSpawner>,
    IRequires<IClock>
{
}
```

于是类型本身已经表达：

```text
Weapon
requires
├── IAmmoSource
├── IProjectileSpawner
└── IClock
```

Builder 可以扫描这些 requirement。

但我不会太依赖 reflection magic。

更实用可能还是显式 metadata：

```csharp
[Requires(typeof(IAmmoSource))]
[Requires(typeof(IProjectileSpawner))]
class Weapon
{
}
```

或者 extension registration 时声明：

```csharp
registry.Register<Weapon>()
    .Requires<IAmmoSource>()
    .Requires<IProjectileSpawner>()
    .Requires<IClock>();
```

我个人其实最喜欢最后一种。

因为所有架构 metadata 都集中在 composition root：

```csharp
registry.Register<Weapon>()
    .Requires<IAmmoSource>(Scope.Local)
    .Requires<IProjectileSpawner>(Scope.World)
    .Publishes<WeaponFired>();
```

非常容易做 graph validation。

---

# 6. 但 ISP 更重要的地方其实不是 Runtime View

这里我觉得有必要纠正一个容易产生的理解。

ISP 原始思想不是：

> 给不同用户创建不同 Runtime interface。

而是：

> **Client 不应该依赖它根本不需要的方法。**

例如我们有：

```csharp
interface IHealth
{
    int Health { get; }

    void Damage(int amount);
    void Heal(int amount);

    void Kill();
    void Revive();

    void SetMaxHealth(int hp);
    void SetInvincible(bool value);
}
```

这个 interface 太胖。

---

## AI 其实只需要：

```csharp
public interface IHealthStatus
{
    int Health { get; }
    bool IsAlive { get; }
}
```

Weapon 只需要：

```csharp
public interface IDamageReceiver
{
    void ApplyDamage(DamageInfo damage);
}
```

Medic：

```csharp
public interface IHealable
{
    void Heal(float amount);
}
```

Respawn system：

```csharp
public interface IRespawnable
{
    void Respawn();
}
```

但是！

**并不意味着我们需要四个 Component。**

同一个：

```csharp
HealthComponent
```

完全可以：

```csharp
class HealthComponent :
    IHealthStatus,
    IDamageReceiver,
    IHealable,
    IRespawnable
{
    ...
}
```

于是：

```text
                     HealthComponent
                  /       |       |       \
                 /        |       |        \
                ▼         ▼       ▼         ▼
       IHealthStatus IDamageReceiver IHealable IRespawnable
             ▲            ▲          ▲          ▲
             │            │          │          │
             AI         Weapon      Medic     Respawn
```

**这是 ISP 真正在我们 Runtime 架构里面最漂亮的应用。**

---

# 7. 所以“Component 数量”和“Interface 数量”完全不是一回事

比如：

```text
AmmoComponent
```

可以提供：

```text
IAmmoStatus
IAmmoSource
IReloadable
IAmmoStorage
```

不同消费者看到不同面：

```text
                    AmmoComponent
                         │
          ┌──────────────┼─────────────┐
          ▼              ▼             ▼
     IAmmoSource     IAmmoStatus    IReloadable
          │              │             │
          ▼              ▼             ▼
       Weapon            UI         PlayerInput
```

UI 不应该能够：

```csharp
ammo.Consume();
```

因为它得到的是：

```csharp
IAmmoStatus
```

语言层面就没有 `Consume()`。

这个比：

```csharp
if (caller == UI)
    don't consume ammo
```

漂亮太多。

---

# 8. 这其实就是“同一个 Runtime Object 有很多面”

我觉得用“面”理解特别直观。

例如一个 Tank：

```text
                    Tank Runtime
                         │
       ┌─────────────────┼──────────────────┐
       │                 │                  │
       ▼                 ▼                  ▼
   IDamageable       IMovable          ITargetable
       ▲                 ▲                  ▲
       │                 │                  │
     Weapon              AI               Radar
```

Tank 并不是：

```text
TankRuntime : ITankEverything
```

而是从不同使用者看来：

```text
Weapon 眼中的 Tank = IDamageable

AI 眼中的 Tank = IMovable + ITargetable

UI 眼中的 Tank = IHealthStatus
```

这和 Hexagonal Architecture 的 Port 思想其实完全接上了。

---

# 9. 这样我们甚至可以避免大量“权限判断”

比如以前：

```csharp
runtime.Health.SetHealth(999);
```

任何拿到 runtime 的人都能改。

现在：

```text
UI
 ↓
IHealthStatus

Weapon
 ↓
IDamageReceiver

Admin/Debug
 ↓
IHealthMutable
```

C# 编译器自己帮我们限制能力。

这其实有点像 **Capability-Based Security**：

> 拿到什么接口，就拥有什么能力。

---

# 10. 所以不是“每一种组合一个 interface”

这是最关键的区别。

不要：

```text
IWeaponRuntime =
    IAmmo + IProjectile + IClock

IAIRuntime =
    IMovement + ITarget + IHealth

...
```

这种组合 interface 很容易爆炸。

而是保留**原子 capability**：

```text
IAmmoSource
IAmmoStatus

IDamageReceiver
IHealthStatus

IPhysicsBody
IMovement

ITargetable
IAimProvider
```

然后消费者组合：

```text
Weapon
    IAmmoSource
    IProjectileSpawner
    IClock

AI
    IMovement
    ITargetable
    IHealthStatus
```

组合发生在**消费者的依赖声明**上，而不是 interface inheritance hierarchy 上。

---

# 11. C# 甚至可以用 `ref struct` / struct Context 做非常严格的 View

如果以后特别追求 Runtime View，可以自动生成：

```csharp
public readonly struct WeaponContext
{
    public IAmmoSource Ammo { get; }
    public IProjectileSpawner Projectile { get; }
    public IClock Clock { get; }
}
```

然后 Builder：

```text
Full Runtime
      ↓
Dependency Resolver
      ↓
generated WeaponContext
      ↓
Weapon
```

甚至可以用 Source Generator，根据：

```csharp
[Requires<IAmmoSource>]
[Requires<IProjectileSpawner>]
```

自动生成 `WeaponContext`。

这样开发者根本不用手写。

不过我不会第一版就做 Source Generator——容易为了 framework elegance 把工程复杂化。

---

# 12. 我觉得第一版最适合我们的实现其实非常简单

Capability：

```csharp
interface IAmmoSource { ... }
interface IProjectileSpawner { ... }
interface IPhysicsBody { ... }
```

Provider：

```csharp
class Magazine :
    IAmmoSource,
    IAmmoStatus
{
}
```

Runtime：

```csharp
class Runtime
{
    T Require<T>();
    T? Optional<T>();
    IReadOnlyList<T> Many<T>();
}
```

Extension：

```csharp
class WeaponExtension
{
    public Weapon Install(Runtime runtime)
    {
        var ammo =
            runtime.Require<IAmmoSource>();

        var projectile =
            runtime.Require<IProjectileSpawner>();

        return new Weapon(
            ammo,
            projectile
        );
    }
}
```

**注意最后这一点：**

```csharp
return new Weapon(ammo, projectile);
```

不是：

```csharp
return new Weapon(runtime);
```

所以完整 Runtime 在这里就停止传播了：

```text
               Full Runtime
                    │
                    ▼
            WeaponExtension
               /          \
              ▼            ▼
        IAmmoSource   IProjectileSpawner
              \            /
               \          /
                  Weapon
```

这可能是目前我们讨论的所有设计里，我最喜欢的一条边界。

**Runtime 是 composition-time infrastructure，而 Capability 是 runtime dependency。**

这样 DSH 式 Runtime 给我们提供非常舒服的插件组装体验，却不会让 DSH 的“大家都拿 Runtime”一路传播进 gameplay code，最终形成 Super Runtime。

而 ISP 则负责第二道防线：

> 即使 `HealthComponent`、`AmmoComponent` 本身很复杂，也只把消费者真正需要的那一“面”作为 capability 暴露出去。

所以我们实际上不需要大量 Runtime Interface。我们需要的是一套**规模受控、语义稳定、尽量正交的 Capability Interfaces**。真正值得仔细设计的是这些接口的粒度，而不是为每种 Object/Weapon 创建一套接口。

## 整理说明

1. 本篇保留原始回答关于原子 Capability、构造注入、Typed Capability Resolver、Context、Requirement Metadata、读写权限分离和 Source Generator 的讨论。
2. ISP 不是为每一种 Capability 组合创建一个新接口，而是让不同消费者只依赖自己真正需要的原子能力。
3. 同一个 HealthComponent 或 AmmoComponent 可以实现多个小接口；Component 数量与 Interface 数量是两个不同维度。
4. 第一版建议采用显式 Install/Builder：Runtime 只在 composition/wiring 阶段使用，运行阶段把 Capability 注入 Weapon 等 gameplay object，不长期传播完整 IRuntime。
5. 这里的 Capability-Based Security 是架构类比：C# 类型系统可以限制调用者可见的方法，但真正的授权、生命周期和运行时策略仍需要由 Runtime 和业务规则共同保证。
