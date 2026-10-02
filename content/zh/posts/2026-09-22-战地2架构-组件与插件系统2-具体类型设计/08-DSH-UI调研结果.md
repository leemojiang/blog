---
title: "DSH UI Plugin 的调研结果"
date: 2026-09-22
weight: 8
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

本篇整理针对 DSH UI/HUD Plugin 的源码和文档调研结果，重点是 UI Host、Slot/Seat、Plugin-owned State、Global UI 和 Session UI。

## Q：DSH 的 UI Plugin 实际是如何设计的？它与我们的 HUD 模型有什么关系？

### 原始提问

> 你之前的分析有结果了吗

## A：

有结果了。我刚又对照了 DSH 当前公开源码和文档。结论很有意思：**DSH 的 UI 设计和我们刚才推导的 HUD 模型非常接近，但它比我们设想的“Object → HUD Surface”更偏向 `UI Host + Slot + Plugin-owned State`。** 

### DSH 实际怎么做 UI Plugin

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

### 更关键的是：DSH 没有把所有 UI 状态塞进 React Component

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

### DSH 还有一个非常重要的区分：Global UI 和 Session UI

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

## 这让我想修改一下我们 HUD 的设计

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

### 例如 Ammo HUD 不再属于 Weapon

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

## 而且 DSH 给了我们一个很好的 Plugin 生命周期方案

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

## DSH 自己其实也是 Hybrid：Snapshot/State + Event/Stream

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

## 映射到游戏，我现在会这样定

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

### 还有一个 DSH 很值得借、但游戏里不要完全照搬的点

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

## 整理说明

1. 本篇保留调研回答中的 Slot/Seat、UI Host、Plugin-owned State、Global UI 与 Session UI 等关键内容，并清除了对话系统内部引用标记。
2. DSH 的 Web UI Host 类似 HUD Runtime，Slot/Seat 类似 HUD Slot，UI Plugin 负责把 Runtime Capability 接入 UI，而不是让业务对象直接拥有整个 UI。
3. 相关调研可参考 [DSH 扩展包说明](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/extensions/README.md)、[Web Client 架构](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/web-client.md) 和 [客户端 Slot 规范](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/AGENTS.md)。
