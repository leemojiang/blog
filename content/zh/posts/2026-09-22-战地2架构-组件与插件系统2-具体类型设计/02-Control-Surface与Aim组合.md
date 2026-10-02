---
title: "Control Surface 与 Aim 组合"
date: 2026-09-22
weight: 2
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

本篇继续讨论控制层如何组合，重点是把 Aim 从 Weapon 的原生能力中拆出，并让 Soldier、Turret、Vehicle 等不同控制来源共享统一接口。

## Q：是否需要多种 Aim Runtime 或组合 Runtime？Control Surface 应该如何初始化和切换？

### 原始提问

> 确实 我觉得Control Surface 也要贯彻组合的思想,功能复杂了就拆开 Aim不属于Weapon的原生能力 干脆拆出来. 不过这么说 我们有没有必要引入 多种runtime组合 比如 solider的aim 控制操作 (旋转人的身体 视角俯仰 和 载具上面的aim 炮塔俯仰 由最近的两层rotation bundle决定 显然需要不同的runtime 初始化. 我们是不是应该实现不同的aim comp 然后使用统一接口? 
>
> 或者有没有必要引入一下组合runtime 比如一个comp 可以依赖runtime a 也可以依赖runtime b (减少类型的爆炸和复杂
>
> 还有一个问题这种control surface 怎么初始化 感觉它是一个动态的 那么是预先缓存(类似runtime  的处理方式 还是游戏进行中控制改换的时候 getcomponent 这种动态获取比较好?

## A：

对，我会选择 **“不同 Aim 实现 + 统一的 capability contract”**，但不会进一步演化成很多 `SoldierRuntime / TankRuntime / TurretRuntime / WeaponRuntime` 类型。你最后提到的“组合 Runtime”其实可以解决这个类型爆炸，不过最好把它理解成 **多个 scope/source 的组合视图**，而不是新的 Runtime class。

核心可以收敛成：

```text
Object Runtime        = 能力实际存在在哪里
Runtime View          = 某个模块允许看到哪些能力
Control Surface       = Controller 可以操作哪些能力
```

这三个东西组合起来就够了。

---

## 1. Soldier Aim 和 Turret Aim：不同实现，统一接口

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

# 2. SoldierAim / TurretAim 内部确实需要不同 Runtime View

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

# 3. 你提出的“Comp 可以依赖 Runtime A + Runtime B”非常值得做

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

# 4. 这样 Object Tree 和 Dependency Graph 又成功分开了

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

# 5. 所以我倾向于“允许多 Source，禁止多 Runtime Dependency”

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

# 6. Control Surface 怎么初始化：我强烈倾向“构建时缓存，切换时绑定”

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

# 7. 例如 Tank 创建时就已经生成不同 Surface

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

# 8. Surface 本身最好也是 Capability Set，而不是固定 class

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

# 9. 可以进一步让 Input Binding 也跟着 Surface 动态变化

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

# 10. 这里其实可以区分“静态 Capability”和“动态 Binding”

这是整个问题最清楚的划分。

### 静态/低频变化

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

### 动态

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

# 11. 真正需要动态改变 Capability 时怎么办？

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

# 12. 真正结构变化才重新 Build Surface

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

## 最终我觉得可以形成一个非常干净的模型

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

## 整理说明

1. 本篇保留了不同 Aim 实现、统一 capability contract、组合 Runtime 视图，以及预缓存与动态 GetComponent 之间取舍的讨论。
2. 核心建议是使用不同实现的 Aim Runtime/Component，但对外提供统一的 Aim Surface；不要为 Soldier、Tank、Turret 分别制造大量顶层 Runtime 类型。
3. 所谓组合 Runtime 更适合理解为多个 scope/source 的组合 View，而不是把多个 Runtime 继承成一个更大的 Runtime class。
4. 动态切换时应缓存结构化的 Control Context，在控制对象或姿态层级发生变化时重新 compose；不建议在 gameplay hot path 中反复 GetComponent。
