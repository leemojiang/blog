---
title: "战地 2 架构、组件与插件系统"
date: 2026-09-21
draft: false
math: true
tags:
  - 战地 2
  - 游戏架构
  - 组件系统
  - 插件架构
  - Unity
categories:
  - 软件架构
description: >-
  从 Battlefield 2 与 Refractor 2 的内容组织边界出发，逐步讨论 Unity 集成、组件语义、Runtime、Capability、Runtime View 和接口隔离。
---

本组笔记整理“分析战地 2 架构”这段讨论，保留原始问答结构。讨论从 Battlefield 2 / Refractor 2 的内容组织方式开始，逐步延伸到 Unity 集成、Component 语义、Hexagonal Architecture、DSH/Cordis 式 Runtime、Capability、Runtime View，以及 Interface Segregation Principle（ISP）。

每个问题单独成篇，尽量保留原始提问、回答中的代码和图示。为了避免把讨论过度概括，事实核验、概念修正和补充内容会放在对应文件的“整理说明”中。

## 战地 2 架构的核心边界


本篇整理关于 Battlefield 2 / Refractor 2 架构先进性的第一个问题。重点是两个观察：一是 Python 胶水层如何承载游戏规则；二是 `.con/.tweak` 等 DSL 如何负责对象定义与组装，并将资源、对象定义、运行时实例和高层游戏逻辑分开。

### Q：战地 2 为什么在游戏架构上依然显得先进？

#### 原始提问

> 我感觉战地2作为20多年前的游戏，架构设计依然非常先进，甚至现在的游戏引擎都没法优雅地处理里面的一些问题：
>
> 1. 比如胶水脚本，控制游戏 gamelogic；
> 2. DSL 脚本实例化对象，资源和对象分开，不用把游戏逻辑、配置数据和资源对象混在一起（Unity 和 UE 这两点好像都做不到）。

### A：BF2 的先进之处主要是内容组织边界，而不是某一项孤立技术

我基本同意，而且经过我们刚才对 BF2 实际资源和脚本的逆向之后，我觉得这个感觉已经不只是“怀旧滤镜”了。更准确地说：**BF2 / Refractor 2 在“游戏内容如何组织”这个问题上，做了一套非常工程化、非常 data-driven 的分层；Unity/UE 虽然能力远强于它，但默认工作流反而经常把这些边界重新揉在一起。**

你提到的两点其实还是同一个架构思想的两个侧面：

```text
              ┌──────────────────────────┐
              │       Native Engine      │
              │ physics/render/net/world │
              └────────────┬─────────────┘
                           │ host / engine API
                    ┌──────▼──────┐
                    │ Python      │
                    │ Game Logic  │
                    │ rules/events│
                    └─────────────┘

       Object definition / assembly
                    │
           ┌────────▼────────┐
           │ .con / .tweak   │
           │ DSL             │
           └────────┬────────┘
                    │ references
     ┌──────────────┼──────────────┐
     ▼              ▼              ▼
 Geometry       Collision       Material...
 .bundledmesh   .collisionmesh  textures/shaders
```

#### 1. Python 胶水层的位置特别漂亮

我们逆向出来的 Python 层其实**没有试图成为游戏引擎本身**。

```text
dice_py.dll → host → bf2 facade → gamemode/scoring
```

这条链很重要。底下的 object manager、player、physics、network、world simulation 仍然由 native engine 管理；Python 主要处理：

> **“游戏世界里发生了 X，那么按照这个游戏的规则，接下来应该发生什么？”**

比如 conquest 的占点、计分、玩家事件、game status transition。

所以 Python 是非常纯粹的 **orchestration / policy / glue layer**。

这跟现代很多项目形成了一个很有意思的反差：Gameplay code 很容易不断膨胀，最后“规则”“对象实现”“状态”“资源引用”“编辑器序列化数据”全部纠缠起来。

BF2 的 Python API 能力其实远不如今天 UE C++/Blueprint 或 Unity C#，但**正因为边界很明确，架构反而特别容易理解**。

#### 2. `.con/.tweak` 更有意思：它本质上不是普通配置文件

这可能是 BF2 最值得借鉴的地方。

我们扫描出来：

- 55 种 `ObjectTemplate` 类型；
- 10,645 个命名模板；
- 8,835 次 `addTemplate`；
- `.con` 大量负责 create / structure；
- `.tweak` 大量通过 `activeSafe` 找回模板，再追加 component/property/template。

所以它其实更接近一种：

**Declarative Object Assembly DSL**

而不是：

**Serialized Object File**

区别非常大。

例如，概念上可以写成：

```text
ObjectTemplate.create Vehicle AH1Z
ObjectTemplate.geometry AH1Z

ObjectTemplate.create Engine AH1Z_Engine
ObjectTemplate.maxSpeed 80

ObjectTemplate.activeSafe Vehicle AH1Z
ObjectTemplate.addTemplate AH1Z_Engine
```

它描述的是：

> **“怎样构造一个 AH1Z 对象定义。”**

而不是把某个 C++ `Vehicle` 实例整体序列化下来。

这使得 BF2 很自然地拥有三层：

```text
Resource
   ↓
Template / Definition
   ↓
Runtime Instance
```

也就是：

```text
AH1Z.bundledmesh
        ↑
        │ resource reference
        │
ObjectTemplate "AH1Z"
        ↑
        │ instantiate
        │
Runtime AH1Z #137
```

**资源不是对象，对象定义也不是对象实例。**

这个边界非常干净。

#### 3. 这恰恰是 Unity 默认工作流比较容易混起来的地方

Unity 当然“能做到”，但它的默认抽象更接近：

```text
Asset
Prefab
GameObject
MonoBehaviour
Serialized Fields
```

于是一个 Prefab 很容易同时承担：

```text
visual resource references
+ hierarchy
+ components
+ gameplay parameters
+ script references
+ serialized state
```

最后可能变成：

```text
Tank.prefab
 ├── MeshRenderer
 │    └── tank.fbx
 ├── Collider
 ├── Rigidbody
 ├── TankController.cs
 ├── WeaponController.cs
 ├── HP = 1000
 ├── MaxSpeed = 60
 └── Gun
      └── ...
```

这当然非常方便——拖进去就能运行。

但 BF2 的思路更像：

```text
tank.mesh                  ← resource

Tank.con                   ← structure
Tank.tweak                 ← parameters

Tank Python/game rules     ← high-level policy

C++ engine                 ← implementation
```

所以如果想做一个 Mod：

**改模型 ≠ 改对象定义 ≠ 改参数 ≠ 改游戏规则。**

这是一个非常强的 property。

#### 4. UE 其实也有类似能力，但默认 authoring model 还是偏“对象中心”

UE 有 DataAsset、DataTable、Config、Blueprint、C++、Primary Asset、Gameplay Tags 等等，所以理论上完全可以搭出类似 BF2 的体系。

问题在于默认工作流非常容易变成：

```text
BP_Tank
 ├── StaticMesh
 ├── components
 ├── variables
 ├── gameplay logic
 ├── resource references
 ├── event graph
 └── child blueprints...
```

然后 `BP_Tank` 本身逐渐成为一个巨大中心。

BF2 的 ObjectTemplate DSL 则天然迫使你考虑：

```text
Tank 是什么？

不是 mesh
不是 runtime entity
不是 Python class

Tank 是一个 definition。
```

我觉得这是两种架构哲学最本质的区别。

#### 5. BF2 与 ECS/data-oriented 思想有相似之处，但不能直接等同

BF2 当然**不是现代意义上的 ECS**，不能把两者直接等同。

但它已经明显弱化了传统的：

```text
class Tank : public Vehicle
```

这种“所有东西首先是一个 OO class”的内容建模方式。

从我们实际统计到的 `addTemplate` 和各种 component/template 组合关系来看，它更强调：

```text
Object Definition
      │
      ├── Geometry
      ├── Collision
      ├── Engine
      ├── Weapon
      ├── Camera
      ├── Effects
      └── ...
```

也就是：

> **Composition over inheritance**

而且这个 composition 不是埋在 C++ 里面，而是**暴露给内容层 DSL**。

这对于 2005 年的游戏来说确实非常漂亮。

#### 6. BF2 最先进的可能不是某个具体技术，而是它的“边界意识”

把我们目前逆向出来的东西放在一起看，会发现 Refractor 2 非常喜欢建立明确边界：

```text
Native engine
────────────────────────
Python gameplay rules
────────────────────────
ObjectTemplate definition
────────────────────────
World instance placement
────────────────────────
Geometry
────────────────────────
Material / Shader
────────────────────────
Texture / raw resources
```

甚至 shader 那边也是类似的。我们之前发现，shader/technique/texture binding 很大一部分实际上存在 mesh 中，而 ObjectTemplate 主要关心：

> “我用哪个 GeometryTemplate。”

也就是说，它没有要求 `ObjectTemplate` 理解 renderer 内部所有细节。

这其实就是很典型的 **separation of concerns**。

所以我会稍微修改你的原话：

> **不是 Unity / UE 做不到 BF2 做的事情，而是 Unity / UE 的默认内容模型没有像 Refractor 2 那样强制维护这些边界。**

现代引擎能力强太多了，因此我们完全可以自己在 UE/Unity 上构建：

```text
Native/System Layer
        ↓
Capability API
        ↓
Gameplay Logic
        ↓
Object Definition DSL
        ↓
Template Registry
        ↓
Runtime Entity
        ↓
Resource Registry
        ↓
Raw Assets
```

而且这跟我们前面讨论的 **Plugin + FSM + Agent** 架构其实突然接上了：**BF2 可以作为一个相当好的母版，继续把它现代化成“DSL 定义世界 + Plugin 提供能力 + FSM 管确定性流程 + Agent 管高层决策”的游戏架构。**

我觉得这个方向很值得继续拆。尤其是可以专门研究一个问题：**如果今天重新设计一个“Refractor 3”，哪些 BF2 的架构应该原封不动保留，哪些应该用 ECS、事件总线、插件系统和 Agent/LLM 重做。**这可能比单纯继续逆向 BF2 更有价值。

### 整理说明

1. 本篇保留原始讨论的核心内容，只对标点、断句、标题层级和少量表达进行了整理。
2. “55 种 `ObjectTemplate` 类型、10,645 个命名模板、8,835 次 `addTemplate`”是此前对 BF2 资源的逆向统计，表示本次讨论中的分析依据；后续如果把这篇笔记作为正式研究材料，还需要补充统计脚本、样本范围和可复现实验记录。
3. “Unity/UE 做不到”在整理时收敛为“默认工作流不会自动维护这些边界”。Unity 和 UE 都可以通过 DataAsset、Addressables、Primary Asset、配置系统、自定义工厂等机制搭建类似分层，但这不是它们最自然的默认 authoring model。
4. “BF2 类似 ECS”只是一种结构上的类比。这里真正要强调的是 **composition over inheritance** 和“对象定义、资源、运行时实例分层”，不应据此把 Refractor 2 直接归类为 ECS。

## Unity 中的低成本实现路径


本篇讨论如何把 BF2 / Refractor 2 中有价值的设计思想低成本地迁移到 Unity，重点包括 Python 胶水脚本、对象定义与资源解耦、DSL、模板与运行时实例，以及现成 Unity 框架的组合方式。

### Q：BF2 的这些设计哪些可以比较低成本地集成到 Unity？

#### 原始提问

> 它的这些设计可以哪些可以比较低成本的集成到 Unity 里面? 比如：
>
> 兼容 Python 脚本? Unity 对 Python 的支持似乎非常不好；
>
> 将模型的数据和资源解耦，好像 Unity 也做不到。Unity 似乎使用的是自己的资源管线，自定义资源加载管线不是很现实。
>
> 我觉得似乎最容易的是实现一个 DSL 解析器，通过脚本语言动态地组装对象，但是这个好像不是特别容易（所有的复杂度集成到解析器里面）。
>
> Unity 有哪些已经有的项目或者插件可以完成这种设计模式？

### A：不要复刻 Refractor 2，而是在 Unity 上增加 definition、composition 和 scripting 层

对，而且我查了一下 Unity 现在的能力后，我觉得有一个重要修正：**BF2 这套东西不需要真的重写 Unity 的资源管线。大约 70% 的架构思想可以比较低成本地架在 Unity 上面。**

真正应该避免的是“复刻 Refractor 2”；应该做的是让 Unity 继续负责它擅长的底层资产导入、渲染和物理，而在上面加一层 BF2 风格的 **definition + composition + scripting**。

#### 各部分的成本判断

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

### 1. Unity 可以做到“数据与资源解耦”

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

### 2. Python 反而确实是最麻烦的

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

### 3. DSL 没有想象中那么难，但不要做“通用语言”

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

### 4. 第一版甚至不一定要解析 DSL

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

### 5. Unity 现成项目更适合“组合使用”，而不是寻找一个插件全包

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

### 6. 可以自己做一个很小的 Refractor-style Unity Framework

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

### 整理说明

1. 本篇保留原始回答的主要结构、代码和图示，只去除了对话系统内部的引用标记，并统一了标点与英文术语格式。
2. 这里的“70%”和“60–70%”是架构判断，不是严格测量结果。更准确的说法是：Definition、Registry、Factory、资源 key 和脚本 API 这几层可以先低成本实现；完整的 mod 热加载、内容构建和工具链仍然需要单独评估。
3. Unity 的 Python Scripting 包定位于 Editor 工作流，并不适用于运行时 Player build；因此笔记中的建议是保留“胶水脚本层”的位置，而不强行保留 Python 语言本身。
4. Addressables 的 Catalog、地址 key 和 Resource Provider 机制可以支持“逻辑资源名 → 实际 Unity 资源”的间接层。参考：[Addressables 内容 Catalog 文档](https://docs.unity3d.com/Packages/com.unity.addressables@1.21/manual/build-content-catalogs.html)、[IResourceProvider API](https://docs.unity3d.com/Packages/com.unity.addressables@1.21/api/UnityEngine.ResourceManagement.ResourceProviders.IResourceProvider.html)。
5. Unity 的 Python 运行时限制可参考：[Python Scripting 官方文档](https://docs.unity3d.com/Packages/com.unity.scripting.python@7.0/manual/index.html)。因此第一版采用 YAML/JSON + Factory + Addressables，再考虑 Lua 或其他运行时脚本，是成本更可控的路径。

## 组件组合与依赖治理


本篇讨论 Component 系统真正困难的部分：异构对象的组件组合、父子对象之间的能力依赖、Unity 原生能力的边界，以及如何避免组件之间形成混乱的循环引用。

### Q：复杂 GameObject 的 Component 系统如何优雅设计？

#### 原始提问

> 我非常同意你的建议 不过我们讨论另一个层面的问题 这种component的设计模式 该如何设计的优雅? 尤其是解决实际一个Gameobject可能会由多个component组合 而这种组合似乎不是免费的
>  实际中有些不同的gameobject可能的component还不一样 每一个去判断 if null 是不现实的. 
> 还有另一点 感觉bf2这个设计其实还不完全是单纯的类型设计 它实际组合的是游戏对象 也就是说实际是一颗对象树 有些子节点的component的能力 可能要依赖parent 的能力 比如child的engine 可能就需要整个root的physic/movement组件 有些则可能是需要本节点的组件. 
> 这会引出一个新的问题有一些组件 比如physic 组件 在unity中接近引擎原生 如何划分也是问题
> 以及还有一个问题,就是游戏逻辑如何分配,最理想的情况就是所有逻辑都出现在组件里面.但是实际情况中可能组件之间会产生混乱的依赖,包括obj依赖组件.比如开火的时候武器可能需要依赖ammo组件 整个开火逻辑又可能导致组件互相引用 这样很容易发生对象需要知道组件 这种情况
>
> 请帮我分析一下有哪些设计模式和方法能尽量减缓上面这些以及其他可能潜在的问题. (比如我们之前看到的plugin注册逻辑似乎就不错 事情开始变得复杂 我觉得单纯依赖注入 之类的办法 还需要结合更多的设计方法

### A：

对，我觉得你现在碰到的已经是这个架构真正困难的部分了：**DSL、Addressables、Lua/Python 都只是外壳；真正决定这个系统能不能长期维护的是 runtime object model。**

而且你指出了 BF2 一个很关键的事实：它并不是简单的 Unity 式“一个 GameObject 挂一堆 Component”，而更接近：

```text
Object
├── Component
├── Component
├── Child Object
│   ├── Component
│   └── Child Object
└── Child Object
    └── Component
```

也就是 **Composite Object Tree + Components**。这会让单纯的 DI、`GetComponent<T>()`、event bus 或 ECS 任何一种单独方案都不够。

我会考虑把它设计成 **Object Tree + Capability + Service Scope + Message/Command + Plugin Registry + 显式 Wiring** 几种模式的组合。

---

## 1. 第一原则：不要让 Component “寻找”依赖

最危险的模式其实就是：

```csharp
void Fire()
{
    var ammo = GetComponent<AmmoComponent>();
    var physics = GetComponentInParent<PhysicsComponent>();
    var audio = GetComponent<AudioComponent>();

    if (ammo != null)
        ...
}
```

它的问题远不只是性能。

真正的问题是：

> **依赖关系被隐藏在代码执行路径里面。**

你根本不知道一个 Weapon 到底需要什么，直到运行到 `Fire()`。

最后就会出现你说的：

```text
if (x != null)
if (y != null)
GetComponent
GetComponentInParent
FindObjectOfType
```

所以我会建立一个非常强的约束：

> **Component 不能主动遍历 Object Tree 查找依赖。**

它可以声明需求，但不能自己寻找。

---

## 2. 把“组件”进一步拆成 Component 和 Capability

这是我觉得最重要的一步。

例如：

```text
RigidbodyComponent
TrackedMovementComponent
WheelMovementComponent
AircraftMovementComponent
```

它们是具体实现。

但其他组件不应该依赖这些具体类型。

应该依赖：

```csharp
IMovement
IPhysicsBody
IDamageable
IAmmoSource
IFireControl
IAimProvider
IPowerSource
```

也就是 **Capability**。

于是：

```text
Weapon
   │
   ├── requires IAmmoSource
   ├── optional IAimProvider
   └── uses IPhysicsBody
```

而不是：

```text
Weapon
   ↓
AmmoComponent
   ↓
VehicleComponent
   ↓
RigidbodyComponent
```

这样就已经把很多依赖关系切断了。

---

## 3. Plugin Registry 在这里确实非常有价值

这就是你提到之前 Cordis/plugin 模型让我觉得特别适合这个架构的原因。

例如：

```csharp
registry.Register<IAmmoSource>(ammoComponent);
registry.Register<IPhysicsBody>(physicsComponent);
registry.Register<IDamageable>(healthComponent);
```

之后 Weapon 不需要知道 GameObject 上到底挂了什么：

```csharp
class Weapon
{
    IAmmoSource ammo;
    IPhysicsBody physics;
}
```

Object build 阶段：

```text
DSL
 ↓
Object Tree
 ↓
instantiate components
 ↓
register capabilities
 ↓
resolve dependencies
 ↓
validate
 ↓
activate
```

这里有一个很重要的区别：

**Dependency Resolution 应该发生在对象组装阶段，而不是 gameplay hot path。**

所以不是每次：

```csharp
Fire()
{
    if (ammo != null)
```

而是：

```text
Build Weapon
    ↓
requires IAmmoSource
    ↓
resolver 找不到
    ↓
ObjectDefinition invalid
```

直接启动失败。

这其实很像编译。

---

## 4. 我甚至会把整个 DSL Build 当成“编译过程”

这是解决很多问题非常漂亮的办法。

比如：

```text
tank.definition
```

不是：

> 加载文件 → 立刻开始跑。

而是：

```text
Parse
 ↓
Object Graph IR
 ↓
Resolve Types
 ↓
Instantiate
 ↓
Register Capabilities
 ↓
Resolve Dependencies
 ↓
Validate Graph
 ↓
Activate
```

甚至：

```text
Tank
├── Physics
├── Movement
├── Engine
└── Turret
    └── Cannon
        ├── Weapon
        └── Ammo
```

可以在进入游戏前检查：

```text
Engine
requires:
    IPowerSink?        ✓
    IMovement          ✓ parent

Weapon
requires:
    IAmmoSource        ✓ local
    IPhysicsBody       ✓ root

Movement
requires:
    IPhysicsBody       ✓ root
```

然后直接产生错误：

```text
ERROR:

Tank/Turret/Cannon/Weapon
requires IAmmoSource(local)

No provider found.
```

这比 runtime `NullReferenceException` 强太多。

---

## 5. 你提出的 Parent / Local 问题，可以引入 Scope

这里 DI 的思想确实有用，但不能只是传统 DI container。

我会明确设计 **Capability Scope**：

```text
Root
│
├── Physics       exports IPhysicsBody
├── Movement      exports IMovement
│
└── Turret
    │
    ├── Rotation  exports IAimProvider
    │
    └── Cannon
        │
        ├── Weapon
        └── Ammo  exports IAmmoSource
```

然后 dependency 可以声明：

```text
Weapon:

IAmmoSource
scope = Local

IAimProvider
scope = Ancestor

IPhysicsBody
scope = Root
```

甚至进一步有：

```text
Local
Parent
Ancestor
Root
Children
Object
World
```

但我不会一开始把 scope 做这么复杂。

最开始三个基本够：

```text
Local
Ancestor
Root
```

---

## 6. 这实际上形成一种“层级 DI”

例如 Cannon：

```text
Tank
│
├── Physics
│
└── Turret
    │
    └── Cannon
        ├── Weapon
        └── Ammo
```

Weapon resolution：

```text
resolve<IAmmoSource>(Local)
            ↓
         Cannon
            ↓
          Ammo
```

而：

```text
resolve<IPhysicsBody>(Root)
            ↓
          Tank
            ↓
         Physics
```

这非常适合 BF2 这种对象树。

---

## 7. 但是不要让 Scope 自动搜索得太聪明

这里存在一个巨大的坑。

如果设计成：

```csharp
Resolve<IPhysicsBody>()
```

然后框架自动：

```text
local 没有
↓
parent 没有
↓
grandparent 有
↓
拿过来
```

开始会觉得特别方便。

半年以后就会变成灾难。

因为：

```text
Tank
└── Trailer
    └── Weapon
```

Weapon 到底应该使用：

```text
Tank.Physics
```

还是：

```text
Trailer.Physics
```

？

所以最好：

```csharp
[RequireCapability(
    Scope = CapabilityScope.Root)]
IPhysicsBody body;
```

而不是魔法搜索。

**Explicit dependency > clever dependency resolution。**

---

## 8. 另一类依赖根本不应该用 DI：行为协作

这是非常重要的区别。

比如：

```text
Weapon → Ammo
```

可以是 dependency。

但：

```text
Weapon fired
↓
play sound
↓
spawn muzzle flash
↓
camera shake
↓
AI hears gunshot
↓
statistics
```

如果全部直接 dependency：

```text
Weapon
 ├─ Ammo
 ├─ Audio
 ├─ Particle
 ├─ Camera
 ├─ AI
 └─ Statistics
```

Weapon 就会变成 God Component。

这时候应该使用 **Event / Message**：

```text
Weapon
   │
   └── emits WeaponFired
               │
       ┌───────┼────────┬──────────┐
       ↓       ↓        ↓          ↓
     Audio    FX       AI       Statistics
```

Weapon 根本不应该知道这些系统存在。

---

## 9. 所以应该明确区分三种关系

这是我会直接写进架构规范的一条。

#### A. Dependency

> “没有它，我无法完成自己的职责。”

例如：

```text
Weapon → IAmmoSource
Movement → IPhysicsBody
Engine → IMovement
```

使用：

**Capability / DI**

---

#### B. Notification

> “我发生了一件事情，谁关心谁处理。”

例如：

```text
WeaponFired
VehicleDestroyed
EngineStarted
DamageReceived
```

使用：

**Event / Message**

---

#### C. Request

> “我要别人替我完成一个动作，但不关心具体实现者。”

例如：

```text
ApplyImpulse
SpawnProjectile
PlaySound
CreateExplosion
```

这更适合：

**Command / Service**

于是：

```text
Dependency → Capability
Notification → Event
Request → Command/Service
```

这个分类可以避免大量 component spaghetti。

---

## 10. 开火就是一个非常好的例子

不要：

```csharp
Weapon.Fire()
{
    ammo.TakeAmmo();

    physics.AddForce(...);

    audio.Play(...);

    particle.Spawn(...);

    owner.NotifyWeaponFired();

    aiSystem.NotifySound(...);
}
```

可以变成：

```text
FireCommand
     ↓
Weapon
     │
     ├── IAmmoSource.consume()
     │
     ├── IProjectileSpawner.spawn()
     │
     └── emit WeaponFired
                   │
          ┌────────┼─────────┐
          ↓        ↓         ↓
        Audio      FX        AI
```

Weapon 的核心职责就非常清楚：

> **判断能不能开火，并执行武器自身必要的状态变化。**

声音不是 Weapon。

粒子不是 Weapon。

AI 听见声音也不是 Weapon。

---

## 11. Unity 原生 Physics 应该包起来，而不是直接传播

你提到 Physics 特别关键。

我不会允许整个 framework 到处：

```csharp
Rigidbody rb;
```

而会做：

```text
Unity Rigidbody
       ↓
UnityPhysicsBodyAdapter
       ↓
IPhysicsBody
```

例如：

```csharp
interface IPhysicsBody
{
    Vector3 Velocity { get; }
    void AddForce(Vector3 force);
}
```

Unity-specific：

```csharp
class RigidbodyPhysicsBody : IPhysicsBody
{
    Rigidbody rb;
}
```

那么：

```text
Movement
Engine
Weapon
Damage
```

都只知道：

```text
IPhysicsBody
```

而不是：

```text
UnityEngine.Rigidbody
```

这其实是 **Adapter / Anti-Corruption Layer**。

以后即使某个东西不是 Rigidbody：

```text
CharacterControllerPhysicsBody
KinematicPhysicsBody
CustomVehiclePhysicsBody
```

上层都不用改。

---

## 12. 甚至 GameObject 本身都不应该成为核心架构对象

这是一个更激进但我认为很重要的原则。

不要：

```text
GameObject
= Game Entity
```

最好：

```text
Game Entity
    │
    ├── Definition
    ├── Capability Registry
    ├── Components
    ├── Children
    │
    └── Unity Binding
          ↓
       GameObject
```

也就是说 Unity `GameObject` 是 **runtime backend representation**。

不是 domain object。

这跟：

```text
IPhysicsBody
      ↓
Rigidbody
```

是同一个哲学。

这样未来 DSL 构造的是：

```text
GameEntity
```

而不是：

```text
Unity GameObject
```

---

## 13. Object Tree 本身最好使用 Composite Pattern

这样 BF2 那种：

```text
Vehicle
├── Engine
├── Wheel
├── Turret
│   └── Cannon
└── Camera
```

可以成为真正的：

```text
EntityNode
 ├── Components
 └── Children<EntityNode>
```

每个 Node：

```text
EntityNode
├── Local Capability Registry
├── Components
├── Parent
└── Children
```

于是：

```text
Tank
[Physics]
[Movement]
│
└── Turret
    [Aim]
    │
    └── Cannon
        [Weapon]
        [Ammo]
```

结构本身就有语义。

---

## 14. 生命周期也必须从一开始设计，否则依赖解析还是会乱

我会明确规定：

```text
Construct
   ↓
Attach
   ↓
Register
   ↓
Resolve
   ↓
Validate
   ↓
Start
   ↓
Running
   ↓
Stop
   ↓
Dispose
```

尤其：

#### Register phase

所有 component：

```text
I provide:
    IAmmoSource
```

先注册。

#### Resolve phase

然后：

```text
Weapon requires:
    IAmmoSource
```

统一解析。

绝对不要：

```text
Component A Awake()
    ↓
Get Component B

但是 B 还没 Awake
```

Unity 的 Awake/Start 顺序问题就基本被你的 framework 隔离掉了。

---

## 15. Optional dependency 应该非常少

这正好解决你最开始的：

> 每一个去判断 `if null` 不现实

我甚至会把 dependency 分成：

```text
Require<T>
Optional<T>
Many<T>
```

比如：

```text
Weapon

Require<IAmmoSource>
Require<IProjectileSpawner>

Optional<IAimProvider>

Many<IWeaponModifier>
```

Require 找不到：

```text
Build Failed
```

而不是 null。

Optional 才允许不存在。

Many 则天然支持：

```text
Weapon
├── SilencerModifier
├── HeatModifier
├── DamageModifier
└── AccuracyModifier
```

返回空集合，而不是 null。

于是 gameplay code 基本不会出现：

```csharp
if (xxx != null)
```

---

## 16. 再进一步，可以引入“Port”思想

如果我们真的想把组件系统做漂亮，我甚至不一定把它叫 dependency。

可以借鉴 Hexagonal Architecture：

```text
Weapon
│
├── AmmoPort
├── ProjectilePort
└── AimPort
```

DSL 负责 wiring：

```text
Cannon:
    Weapon:
        ammo -> ./Ammo
        physics -> $root.Physics
        aim -> ../TurretAim
```

这就非常有意思了。

因为 dependency graph **不再完全由类型自动推导**。

而是：

```text
Object Graph
+
Component Graph
+
Wiring Graph
```

这其实可能比自动 DI 更适合复杂游戏对象。

---

## 17. 最终我会把整个系统理解成“三张图”

这是我觉得我们现在讨论到最关键的抽象。

#### ① Object Graph

谁是谁的子对象：

```text
Tank
└── Turret
    └── Cannon
```

#### ② Capability Graph

谁提供什么能力：

```text
Physics ──provides──> IPhysicsBody

Ammo ──provides─────> IAmmoSource

Turret ──provides───> IAimProvider
```

#### ③ Interaction Graph

运行时谁和谁交流：

```text
Weapon
  │
  ├─command──> ProjectileSpawner
  │
  └─event────> WeaponFired
                    │
              ┌─────┼─────┐
              ↓     ↓     ↓
             FX   Audio   AI
```

**不要试图用一张 Component dependency graph 同时表达这三件事情。**

很多 component architecture 最后变乱，我觉得根源就在这里。

---

## 18. Plugin Registry 在整个架构里的位置也因此变清楚了

它不是用来解决所有依赖。

它解决：

> **“系统有哪些类型/能力/handler 可以被构造？”**

比如：

```text
Plugin: Vehicle

register component:
    Engine

register capability:
    IMovement

register DSL:
    vehicle.*
    engine.*

register command:
    ApplyTorque

register event handler:
    VehicleDestroyed
```

另一个：

```text
Plugin: Weapons

register component:
    Weapon
    Ammo

register capability:
    IAmmoSource
    IFireControl

register DSL:
    weapon.*
```

于是核心 framework 根本不知道：

```text
Tank
Weapon
Aircraft
Character
```

是什么。

---

## 19. 最后会形成一个我觉得非常漂亮的架构

```text
                    DSL / YAML
                        │
                        ▼
                 Definition IR
                        │
                        ▼
                ┌──────────────┐
                │ Object Builder│
                └──────┬───────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
      Object Tree   Components    Wiring
          │            │            │
          └────────────┼────────────┘
                       ▼
               Capability Registry
                       │
                 Resolve / Validate
                       │
                       ▼
                    Runtime
                       │
          ┌────────────┼─────────────┐
          ▼            ▼             ▼
      Capability     Commands       Events
          │            │             │
          └────────────┼─────────────┘
                       ▼
                Engine Adapters
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
     Physics        Rendering        Audio
        │
        ▼
                Unity Runtime
```

而 Plugin 系统包在整个体系外围：

```text
Plugins
  │
  ├── register Component Types
  ├── register Capabilities
  ├── register Commands
  ├── register Event Handlers
  └── register DSL extensions
```

---

### 我认为有几条规则尤其值得坚持

如果以后真的实现，我会把它们直接写进 architecture constraints：

1. **Component 不允许 `GetComponent*` 寻找业务依赖。**
2. **Required dependency 在 build/validation 阶段解决，runtime 不做 null checking。**
3. **依赖具体能力，不依赖具体 Component class。**
4. **“必须协作”用 Capability；“发生了什么”用 Event；“请做某事”用 Command/Service。**
5. **跨对象树依赖必须明确 scope 或显式 wiring，不允许隐式 ancestor magic。**
6. **Unity Rigidbody/GameObject/AudioSource 等通过 Adapter 留在 framework 边界。**
7. **Component 不应该知道整个 Owner 对象的内部结构。**
8. **DSL 只描述 Object Graph + Component + Wiring，不承担 gameplay programming。**
9. **Plugin 负责扩展类型系统，而不是让 Core 不断增加 `if(type == Tank)`。**
10. **所有 runtime object 在 Activate 前完成 graph validation。**

这样我们其实不是在设计“Unity Component 系统的另一种写法”。

而是在 Unity 上面建立一个小型的 **game object runtime / composition kernel**。

我甚至觉得这里可以从 BF2 再往前走一步：**BF2 的 `ObjectTemplate.addTemplate` 已经非常接近这种思想，但如果加入现代的 Capability/Port、scoped dependency、graph validation、event/command separation 和 plugin registry，会比 BF2 原始架构更加干净。**

而这套东西和我们之前讨论的 **FSM / Agent** 也不会冲突：FSM、Behavior Tree、Agent 都可以只是某些 Component 的内部实现；它们通过 Capability/Command/Event 与外部世界交流，而不是反过来污染整个对象模型。

### 整理说明

1. 本篇基本保留原始回答的 19 个小节、代码示例和三张图的区分，只对问题表述做了断句和错别字整理。
2. “像编译”是架构类比：Definition 构建阶段可以执行类型解析、能力注册、依赖解析和图验证，但不意味着必须真的生成机器码。
3. Local、Ancestor、Root 是建议的最小 Scope 集合，不是 Unity 或 BF2 的既有标准；实际实现时还需要明确多 provider、优先级、覆盖和生命周期规则。
4. “三张图”是本篇最重要的抽象：Object Graph 表示结构与所有权，Capability Graph 表示提供/消费关系，Interaction Graph 表示事件与命令。

## Hexagonal Architecture 与组件语义


本篇把 Hexagonal Architecture（六边形架构，也称 Ports and Adapters）引入 BF2 的 Component 讨论，用来区分 Behavior、Capability Provider、Engine Adapter 和 Event Consumer。

### Q：Hexagonal Architecture 是什么？BF2 中不同语义的 Component 应该如何区分？

#### 原始提问

> Hexagonal Architecture  是什么? 我觉得你说的特别有启发 BF2 的comp/object 系统 似乎没有在命名上强调 能力 和通知之类的设计 (毕竟只有comp 的名称 但是如何实现的是在core部分 脚本部分是看不出来的 所以我感觉遇到了很多 又像是comp 又像是能力的comp 比如 firecomp 和 ammocomp 它们似乎和 挂载的武器类太接近了 但是有些fxcomp又太远了 如果把这些都当作comp来处理 确实会变得很混乱

### A：

对，你现在这个观察其实正好切到了 **Hexagonal Architecture（六边形架构）** 对我们这个问题最有价值的地方。

它并不是一种“怎么写 Component”的模式，而是在提醒我们：

> **不要把“一个东西是什么”和“它如何与外界交互”混成同一种抽象。**

这恰好可以解释为什么 BF2 里 `FireComp / AmmoComp / FxComp / PhysicsComp` 全都叫 Comp，会让我们逆向时感觉它们明明不是同一种东西。

---

### Hexagonal Architecture 到底是什么？

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

## 这正好可以解决 BF2 的“所有东西都叫 Comp”的问题

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

## 关键是：Component 应该只是“物理容器”，不是架构语义

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

## 举一个完整的武器例子

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

## Ports 思维完全不一样

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

## 那么 FX 和声音呢？

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

## 所以我们甚至可以建立一个“距离”的概念

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

## 这也解释了 Physics 为什么让我们之前觉得别扭

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

## 甚至 Input 都可以这样处理

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

## 这样我们可以重新定义“Component”

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

## 但不要把这些分类做成死板的 class hierarchy

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

## 最后 DSL 甚至可以把这些关系暴露出来

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

### 整理说明

1. 本篇保留原始回答的核心推导、武器示例、Port/Adapter 图示和 DSL 示例。
2. 六边形架构在这里不是要求每个 Component 都继承某个固定基类，而是要求 Domain 通过稳定的 Port 与外部系统交互，具体实现由 Adapter 提供。
3. FireComp、AmmoComp、PhysicsComp、FxComp 都可以仍然是底层承载单位，但它们承担的架构角色不同；Component 名称本身不足以表达依赖方向。
4. 本篇的“中心/外围”是依赖方向的解释工具，不等于所有外围模块都必须在物理目录或 Unity 层级上位于外侧。

## 从 Plugin 到 Runtime 的组件简化


本篇讨论能否借鉴 DSH/Cordis 的 Plugin 思路，把 Component 和 Object 简化为 Runtime 的组成部分，同时保留游戏对象树与语义依赖之间的区分。

### Q：能否参考 DSH 的设计模式，把 Component 和 Object 简化为 Runtime 的组成部分？

#### 原始提问

> 真的很精妙 我有一种恍然大悟的感觉 这么说我感觉 deepseek harness 的 plugin 模式其实是更简化了一部分,也就是说plugin 只提供runtime. 而Component设计模式则要复杂一点 component 和object 有了自己的语义,甚至comp之间还有区别,而不是简单的runtime.
>
> 那我们能不能参考DSH 的设计模式 进一步简化comp 和 obj. 也就是说所有的comp 或者obj 都是runtime的一部分,相当于依赖这个runtime的去访问所有的runtime,这样只要解决好依赖,逻辑关系似乎变得更清晰了. 同时保证了所有依赖关系按照单层依赖 (runtime 逐步扩大) 并且将所有的复杂度集合在了runtime 的使用者上面 避免了交叉引用/环形引用
>
> 不过问题在于这种设计模式会不会不是很适合游戏这个环境(比如plugin 是平等的,而游戏obj 有更复杂的比如树状的结构,但这种结构表示的是游戏对象的关系 而不是它们语义和逻辑上的关系 
>
> 你觉得我们可以按照这个思路简化comp系统吗?

### A：

可以，而且我觉得你这个方向可能比我们上一轮讨论的“Capability + Port + Event + Command + Scope”更适合作为**第一版核心模型**。

上一版虽然干净，但已经开始有一点“为了治理复杂度而创造很多概念”的趋势。DSH/Cordis 那种 Runtime 模式给了一个更激进的方向：

> **不要让 Component 之间建立依赖关系。所有 Component/Object 都只依赖它所属的 Runtime。**

这样很多问题确实会突然消失。

不过我会做一个关键修正：**Object Tree 和 Runtime Dependency Graph 必须是两张完全不同的图。**

---

### 1. 可以把 DSH 的思想直接搬过来

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

## 2. 最漂亮的地方：依赖图真的可以坍缩

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

## 3. 但是游戏和 DSH 最大的区别正是你发现的：存在 Object Tree

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

## 4. 所以我会把整个模型压缩成两个核心概念

甚至不是之前的六七种。

只有：

#### Object

负责**结构**：

```text
Object
├── parent
├── children
├── transform
└── runtime
```

#### Runtime

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

## 5. Component 甚至可以从“核心概念”里面消失

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

## 6. `FireComp` / `AmmoComp` 的困惑也因此消失很多

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

## 7. 而且“逐步扩大的 Runtime”非常有意思

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

## 8. 这样 dependency 也变成 Plugin Installation Dependency

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

## 9. 但是我不会完全删除 Capability / Port

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

## 10. Event 也可以直接成为 Runtime 的基础设施

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

## 11. Parent dependency 也可以用 Runtime chaining，而不是 Component tree search

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

## 12. 不过这里必须小心：不要自动无限向 parent 搜索

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

## 13. 甚至 World 本身也可以是 Runtime

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

## 14. 这可能比传统 Component System 更适合我们的目标

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

## 15. 这样 DSL 也会变得异常简单

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

## 16. Plugin Registry 也正好可以统一进来

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

### 但我会加一条非常重要的限制

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

### 整理说明

1. 本篇保留原始回答关于 Object Tree、Runtime、Runtime Extension、Capability、Event 和 Registry 的完整推导。
2. 最关键的修正是：Object Tree 表示 Ownership、Transform、Lifetime、Spatial relationship 和 Serialization；它不自动表示软件依赖。
3. “Component 是向 Runtime 安装能力的模块”是一种统一的安装模型，不代表 Fire、Ammo、FX、Physics 在语义上变成同一种东西。
4. 本篇已经指出 Super Runtime 风险，因此后续第 6 篇继续讨论 Runtime View、依赖剪裁和访问边界。

## Super Runtime 与 Runtime View


本篇继续讨论 Scoped Runtime 的主要风险：Runtime 可能逐步膨胀为所有使用者都能访问的 Super Runtime。核心解决方案是把完整 Runtime 与使用者获得的 Runtime View 分开。

### Q：如何避免 Weapon Runtime 演化成 Super Runtime？除了接口统一和依赖树剪裁，还有哪些办法？

#### 原始提问

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

### A：

对，你总结的两点都很准确。不过我会再做一个小修正：**真正要避免的不是 Runtime 里面“有十几个东西”，而是任何 Runtime 使用者都能看到、理解并操作这十几个东西。**

一个 Tank runtime 里存在 30 个 capability 本身不一定有问题；`Weapon` 能随手访问其中 30 个，才开始变成 Super Runtime。

所以我觉得可以把我们的设计原则进一步收敛成一句话：

> **Runtime 可以大，但 Runtime View 必须小。**

---

### 1. 你说的两个办法，可以分别理解成“横向压缩”和“纵向剪裁”

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

## 2. 第三个办法：区分“Required”和“Observed”

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

## 3. 你说“也可以反向设计？”——可以，但要看谁拥有语义

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

## 4. 第四个办法：Runtime 不提供“枚举所有东西”的能力

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

## 5. Runtime View 最好成为真正的一等公民

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

## 6. 第五个办法：读和写也应该剪裁

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

## 7. 第六个办法：State 和 Behavior 也不要随便混

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

## 8. 第七个办法：特别警惕“方便型依赖”

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

## 9. 可以给 Dependency 设置“预算”

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

## 10. 还有一个非常强的办法：Facade Capability

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

## 11. Runtime 最好形成“能力层级”，而不是一个平面大字典

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

## 12. 生命周期也可以帮助防止 Runtime 失控

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

#### `Seal`

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

## 13. Plugin 和 Runtime Extension 也最好是“安装后消失”

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

## 14. 最终我会把关系限制成很少的几种

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

### 我现在会把我们的架构压缩成这样

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

### 整理说明

1. 本篇保留原始回答关于 Required/Observed、Runtime View、读写能力剪裁、依赖预算、Facade Capability、Runtime Seal 和关系语法约束的完整结构。
2. “Runtime 可以大，但 Runtime View 必须小”是本篇的中心原则；它强调的是使用者可见的能力面，而不是 Runtime 内部实际存储的对象数量。
3. 删掉模块后核心领域职责是否仍然成立，是判断 Required Capability、Event 和外围系统的实用标准。
4. Dependency budget 更适合作为架构预警和 lint 规则，而不是绝对的硬限制；不同领域对象的合理依赖数量需要结合职责和接口粒度判断。

## ISP 在游戏 Runtime 中的应用


本篇回答如何在原生 C# 中实现 Interface Segregation Principle（接口隔离原则），并说明它与 Runtime View、Capability Interface 和 Component 数量之间的关系。

### Q：ISP 在游戏 Runtime 中如何具体实现？是否需要为每一种 Runtime View 创建一个新的 interface？

#### 原始提问

> 非常棒 你刚才提到的 Interface Segregation Principle 在游戏 runtime 里的具体应用。 这个怎么具体实现? 我理解这个似乎是在不同runtime暴露不同interface 但是原生c#怎么支持这种语法? 我觉得不能对每一个可能的暴露新建一个interface吧?

### A：

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

### 1. 先区分两个概念：Capability Interface 和 Runtime View

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

## 2. 最简单实现：构造时直接注入 Capability

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

## 3. 但这样参数很多，所以可以有一个 Context

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

## 4. 更 DSH 风格的方法：Typed Capability Resolver

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

## 5. 再高级一点：用泛型表达 Runtime Requirement

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

## 6. 但 ISP 更重要的地方其实不是 Runtime View

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

### AI 其实只需要：

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

## 7. 所以“Component 数量”和“Interface 数量”完全不是一回事

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

## 8. 这其实就是“同一个 Runtime Object 有很多面”

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

## 9. 这样我们甚至可以避免大量“权限判断”

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

## 10. 所以不是“每一种组合一个 interface”

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

## 11. C# 甚至可以用 `ref struct` / struct Context 做非常严格的 View

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

## 12. 我觉得第一版最适合我们的实现其实非常简单

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

### 整理说明

1. 本篇保留原始回答关于原子 Capability、构造注入、Typed Capability Resolver、Context、Requirement Metadata、读写权限分离和 Source Generator 的讨论。
2. ISP 不是为每一种 Capability 组合创建一个新接口，而是让不同消费者只依赖自己真正需要的原子能力。
3. 同一个 HealthComponent 或 AmmoComponent 可以实现多个小接口；Component 数量与 Interface 数量是两个不同维度。
4. 第一版建议采用显式 Install/Builder：Runtime 只在 composition/wiring 阶段使用，运行阶段把 Capability 注入 Weapon 等 gameplay object，不长期传播完整 IRuntime。
5. 这里的 Capability-Based Security 是架构类比：C# 类型系统可以限制调用者可见的方法，但真正的授权、生命周期和运行时策略仍需要由 Runtime 和业务规则共同保证。
