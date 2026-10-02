---
title: "Unity 中的低成本实现路径"
date: 2026-09-21
weight: 2
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

本篇讨论如何把 BF2 / Refractor 2 中有价值的设计思想低成本地迁移到 Unity，重点包括 Python 胶水脚本、对象定义与资源解耦、DSL、模板与运行时实例，以及现成 Unity 框架的组合方式。

## Q：BF2 的这些设计哪些可以比较低成本地集成到 Unity？

### 原始提问

> 它的这些设计可以哪些可以比较低成本的集成到 Unity 里面? 比如：
>
> 兼容 Python 脚本? Unity 对 Python 的支持似乎非常不好；
>
> 将模型的数据和资源解耦，好像 Unity 也做不到。Unity 似乎使用的是自己的资源管线，自定义资源加载管线不是很现实。
>
> 我觉得似乎最容易的是实现一个 DSL 解析器，通过脚本语言动态地组装对象，但是这个好像不是特别容易（所有的复杂度集成到解析器里面）。
>
> Unity 有哪些已经有的项目或者插件可以完成这种设计模式？

## A：不要复刻 Refractor 2，而是在 Unity 上增加 definition、composition 和 scripting 层

对，而且我查了一下 Unity 现在的能力后，我觉得有一个重要修正：**BF2 这套东西不需要真的重写 Unity 的资源管线。大约 70% 的架构思想可以比较低成本地架在 Unity 上面。**

真正应该避免的是“复刻 Refractor 2”；应该做的是让 Unity 继续负责它擅长的底层资产导入、渲染和物理，而在上面加一层 BF2 风格的 **definition + composition + scripting**。

### 各部分的成本判断

| BF2 思想 | Unity 实现 | 成本 | 值得程度 |
|---|---|---:|---:|
| Python gameplay glue | Python runtime / 换 Lua | 中~高 | ★★★ |
| `.con/.tweak` DSL | 自己做很薄的 DSL | 中 | ★★★★★ |
| Resource / Definition 分离 | Addressables + string key | **低** | ★★★★★ |
| Template / Instance 分离 | Definition + Factory | **低** | ★★★★★ |
| `addTemplate` composition | Component registry | 低~中 | ★★★★★ |
| 动态修改参数 | JSON/YAML/DSL overlay | **低** | ★★★★★ |
| 完全绕过 Unity asset pipeline | 自己 importer/runtime loader | **高** | ★ |
| BF2 式 mod 系统 | Addressables + DSL + scripting | 中 | ★★★★★ |

这里最值得重新讨论的是你说的第二点。

## 1. Unity 可以做到“数据与资源解耦”

关键是：**不要让 Definition 保存 Unity Object reference。**

比如通常 Unity 会写：

```csharp
public class TankDefinition : ScriptableObject
{
    public GameObject prefab;
    public AudioClip engineSound;
}
```

这就又回到了你不喜欢的：

```text
Definition → Unity Asset
```

但 Addressables 本身已经提供了 BF2 所需要的那层 indirection：

```text
"vehicle/t90/mesh"
        ↓
 Addressables Catalog
        ↓
AssetBundle / Content Directory
        ↓
Unity Asset
```

因此完全可以让：

```text
T90.definition

type = Vehicle
geometry = "vehicles/t90/body"
collision = "vehicles/t90/collision"
engineSound = "audio/tank/diesel_01"

mass = 46000
maxSpeed = 60
```

这里**没有任何 Unity asset reference**。

运行时：

```text
DSL
 ↓
T90 Definition
 ↓
ResourceRegistry
 ↓
"vehicles/t90/body"
 ↓
Addressables
 ↓
Mesh / Prefab / Material
```

所以这一项应该从“Unity 做不到”修改成：

> **Unity 默认工作流不会自动维护这种边界，但 Addressables 已经提供了实现 BF2 Resource Registry 所需的底层机制。**

完全不需要自己写模型加载器。

## 2. Python 反而确实是最麻烦的

你的印象是对的。

```text
BF2:

Native Engine
     ↑
   host
     ↑
 Python 2.3
     ↑
gamemode.py
```

不能低成本地直接变成：

```text
Unity
 ↑
Python
```

当然可以嵌入 CPython、Python.NET，或者通过 IPC 使用独立 Python 进程，但我**不建议为了复刻 BF2 而复刻 Python**。

因为真正值得继承的是：

> **一个 runtime scripting glue layer**

而不是：

> **Python 这门语言。**

所以完全可以换成：

```text
             Gameplay Script API
                     │
         ┌───────────┴───────────┐
         ↓                       ↓
       Lua                    C# Module
```

Lua 在这里其实非常符合 BF2 Python 的角色：

```lua
function onPlayerKilled(player, killer)
    killer:addScore(2)
end

function onControlPointCaptured(cp, team)
    GameLogic:addTickets(team, 10)
end
```

真正重要的是 API 边界：

```text
Script
  │
  │ Game.Player(...)
  │ Game.Spawn(...)
  │ Game.Events(...)
  ▼
C# facade
  │
  ▼
Unity implementation
```

这才是 BF2 的精髓。

## 3. DSL 没有想象中那么难，但不要做“通用语言”

你说：

> 所有复杂度集成到解析器里面。

如果做：

```text
if
for
function
expression
inheritance
variables
scope
reflection
```

确实马上会变成“我为什么在 Unity 里面重新实现一门编程语言”。

但 BF2 给我们的启示反而是：

**DSL 可以非常蠢。**

例如只允许：

```text
create Vehicle T90
geometry vehicles/t90/body
mass 46000
maxSpeed 60

component Engine
    power 1000
    sound audio/tank/diesel

component Weapon
    template weapons/125mm
```

Parser 根本不需要知道 Vehicle 是什么。

只需要输出：

```text
Command("create", ["Vehicle", "T90"])
Command("geometry", ["vehicles/t90/body"])
Command("mass", ["46000"])
...
```

然后：

```text
DSL Parser
    ↓
Command
    ↓
Command Registry
    ↓
C# Handler
```

例如概念上：

```csharp
registry.Register("mass",
    (obj, args) => obj.Mass = float.Parse(args[0]));

registry.Register("geometry",
    (obj, args) => obj.GeometryKey = args[0]);
```

那么复杂度不是：

```text
Parser
████████████████████
```

而是：

```text
Parser          ██

VehicleModule   ███
WeaponModule    ███
PhysicsModule   ███
AudioModule     ███
ResourceModule  ███
```

这反而和我们之前讨论的 **Cordis/plugin architecture** 非常像。

DSL parser 只是 dispatch。

## 4. 第一版甚至不一定要解析 DSL

第一版我可能直接用 YAML：

```yaml
name: T90
type: Vehicle

geometry: vehicles/t90/body

components:
  - type: Engine
    power: 1000
    sound: audio/tank/diesel

  - type: Weapon
    template: weapons/125mm

physics:
  mass: 46000
  maxSpeed: 60
```

然后：

```text
YAML
 ↓
Definition DTO
 ↓
ObjectFactory
 ↓
ComponentFactory
 ↓
GameObject
```

等整个 object model 稳定以后，再给它套一层漂亮的 BF2-style DSL。

这样成本会低很多。

## 5. Unity 现成项目更适合“组合使用”，而不是寻找一个插件全包

比较有意思的是 **Zenject** 这类 DI/composition framework。它解决的是：

```text
Definition
    ↓
Factory
    ↓
composition
    ↓
runtime object
```

然后 Unity Addressables 解决：

```text
Resource Key
    ↓
Catalog
    ↓
Asset
```

Lua runtime 解决：

```text
game event
    ↓
script
    ↓
gameplay rule
```

最后由我们自己的薄层解决：

```text
DSL/YAML
   ↓
Object Definition
```

因此并不是寻找：

> “Unity Battlefield 2 architecture plugin”

而是把几个成熟 subsystem 拼起来。

## 6. 可以自己做一个很小的 Refractor-style Unity Framework

第一版甚至可以控制在很小的规模：

```text
RefractorUnity/
│
├── Runtime/
│   ├── Definition/
│   │   ├── ObjectDefinition.cs
│   │   └── ComponentDefinition.cs
│   │
│   ├── Registry/
│   │   ├── ObjectTypeRegistry.cs
│   │   ├── ComponentRegistry.cs
│   │   └── ResourceRegistry.cs
│   │
│   ├── Factory/
│   │   └── ObjectFactory.cs
│   │
│   ├── Resources/
│   │   └── AddressableResourceProvider.cs
│   │
│   └── Scripting/
│       └── GameAPI.cs
│
└── Content/
    ├── vehicles/
    │   └── t90.yaml
    └── weapons/
        └── 125mm.yaml
```

然后 runtime：

```text
                   t90.yaml
                       │
                       ▼
               ObjectDefinition
                       │
                ObjectFactory
                       │
          ┌────────────┼─────────────┐
          ▼            ▼             ▼
       Engine        Weapon        Physics
      Component     Component      Component
          │            │
          └──────┬─────┘
                 ▼
              GameObject
                 │
       ┌─────────┴──────────┐
       ▼                    ▼
 ResourceRegistry       Gameplay API
       │                    │
 Addressables              Lua
       │
 Mesh / Audio / Material
```

这个架构最漂亮的地方在于：**Unity 被降级成 Runtime，而不再是你的 Game Architecture。**

Unity 负责：

```text
render
physics
audio
input
platform
asset import/build
```

我们自己控制：

```text
object model
definition
composition
game rules
resource naming
modding
```

这实际上已经相当接近 BF2 的哲学了。

而且我认为第一版甚至**不用 Python、不用 DSL、不用自定义 Asset Pipeline**。先做：

**YAML/JSON Definition + Registry + Factory + Addressables**

就已经拿到了 BF2 架构大概最有价值的 60–70%。等这套跑通，再加入 Lua gameplay scripting；最后如果 YAML 开始难以维护，再做 BF2 风格 DSL。这样每一步都很便宜，也不会一开始掉进“自己造游戏引擎”的坑里。

## 整理说明

1. 本篇保留原始回答的主要结构、代码和图示，只去除了对话系统内部的引用标记，并统一了标点与英文术语格式。
2. 这里的“70%”和“60–70%”是架构判断，不是严格测量结果。更准确的说法是：Definition、Registry、Factory、资源 key 和脚本 API 这几层可以先低成本实现；完整的 mod 热加载、内容构建和工具链仍然需要单独评估。
3. Unity 的 Python Scripting 包定位于 Editor 工作流，并不适用于运行时 Player build；因此笔记中的建议是保留“胶水脚本层”的位置，而不强行保留 Python 语言本身。
4. Addressables 的 Catalog、地址 key 和 Resource Provider 机制可以支持“逻辑资源名 → 实际 Unity 资源”的间接层。参考：[Addressables 内容 Catalog 文档](https://docs.unity3d.com/Packages/com.unity.addressables@1.21/manual/build-content-catalogs.html)、[IResourceProvider API](https://docs.unity3d.com/Packages/com.unity.addressables@1.21/api/UnityEngine.ResourceManagement.ResourceProviders.IResourceProvider.html)。
5. Unity 的 Python 运行时限制可参考：[Python Scripting 官方文档](https://docs.unity3d.com/Packages/com.unity.scripting.python@7.0/manual/index.html)。因此第一版采用 YAML/JSON + Factory + Addressables，再考虑 Lua 或其他运行时脚本，是成本更可控的路径。
