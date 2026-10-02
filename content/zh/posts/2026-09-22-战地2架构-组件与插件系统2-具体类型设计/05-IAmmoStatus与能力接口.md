---
title: "IAmmoStatus 与能力接口"
date: 2026-09-22
weight: 5
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

本篇澄清 Surface/Capability 接口和 Event 接口的区别，以 IAmmoStatus 为例说明状态读取能力应该由谁实现。

## Q：IAmmoStatus 应该由谁实现？它是 Surface 还是 Event 接口？

### 原始提问

> IammoStatus 这个接口应该谁实现 我有点没搞懂 它是我们之前广义上的surface 还是一个event 接口 (我理解这是两种不同的接口

## A：

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

## 1. `IAmmoStatus` 应该由谁实现？

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

# 2. Event 是另外一个东西

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

# 3. 为什么两个都需要？

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

# 4. Presentation Surface 也不是 `IAmmoStatus` 本身

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

# 5. 那么谁负责 Event？

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

# 6. 不过 UI 数据绑定可以做一层便利封装

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

## 所以我建议把现在的术语正式固定下来

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

## 整理说明

1. 本篇保留了对 IAmmoStatus、Surface/Capability、Event 和 Reactive/Observable 数据流的概念切分。
2. IAmmoStatus 应由真正拥有弹药状态的 Ammo Runtime 或其 Adapter 实现；HUD 只消费这个接口，不应该把事件方法塞进同一个接口。
3. 能力接口描述当前可读取或调用的 contract，事件接口描述已经发生的离散事实；两者可以协作，但不应混成一个类型。
