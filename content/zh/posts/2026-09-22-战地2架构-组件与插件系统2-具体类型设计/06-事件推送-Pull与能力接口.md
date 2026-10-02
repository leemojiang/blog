---
title: "事件推送、Pull 与能力接口"
date: 2026-09-22
weight: 6
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

本篇讨论 HUD 数据同步中 Event Push、每帧 Pull、订阅绑定和 Capability Surface 的边界。

## Q：Event Push 和自动 Pull 同时存在是否过于复杂？订阅绑定与 Runtime 初始化如何处理？

### 原始提问

> 这是不是有点太复杂了 event push变更 但是还得自动pull 数值 然后event的订阅绑定似乎和runtime 的初始化里面也是一个挺需要考虑的事情

## A：

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

## 对 HUD，Pull 很可能就是最好的默认方案

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

## Event 应该留给真正的“事件”

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

# 1. 甚至 `AmmoChanged` 都不一定需要存在

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

# 2. Event 只负责“不能通过当前状态完整表达”的事情

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

# 3. 这样 Runtime 初始化也突然简单很多

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

# 4. 甚至 Widget Composition 也可以在 Bind 时一次解决

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

# 5. EventHub 仍然存在，但它变得非常小

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

# 6. Camera 其实也一样

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

# 7. 我甚至会给 Framework 定一个非常简单的规则

### 高频 / 连续 / 当前值

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

### 离散 / 瞬时 / “发生了一件事”

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

### 结构发生改变

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

# 8. 这也回答了一个更深的问题：Surface 应该尽量“傻”

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

## 整理说明

1. 本篇保留了对连续状态、离散事件、缓存 Surface、订阅生命周期和 Runtime 初始化的完整讨论。
2. 连续数值适合 Pull 或 Observable Store，离散变化适合 Event；两者可以共存，但不应让同一条数据同时产生多套无约束的真相来源。
3. 订阅关系应由 Composer/Runtime 在初始化和销毁阶段统一建立、撤销，而不是让 UI 在运行时到处寻找对象或重复绑定。
4. 这一篇承接 IAmmoStatus，说明为什么状态 Surface 和事件通知需要保持不同的接口边界。
