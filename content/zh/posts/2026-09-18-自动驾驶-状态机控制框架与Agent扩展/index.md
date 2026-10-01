---
title: "自动驾驶：基于状态机的控制框架与 Agent 扩展"
date: 2026-09-18
draft: false
math: true
tags:
  - 自动驾驶
  - 状态机
  - Agent
  - LLM
categories:
  - 工程实践
description: >-
  从一个简单、可验证的状态机 Runtime 出发，讨论游戏自动驾驶控制框架，以及 Agent 和 LLM 应该处于怎样的决策层级。
---

本笔记讨论一种面向游戏自动驾驶和未来航模实验的控制框架。主线是先建立一个简单、可验证的状态机 Runtime；在此基础上，再讨论如何把 Agent/LLM 放到更高层的决策位置。Agent 是后续扩展方向，不是第一版控制框架的核心。

## Q: 如何设计一个基于状态机的自动驾驶控制框架？

> 挺好 回到一个我具体的问题上 我想设计一个自动驾驶程序 (比如在战争雷霆里面自动操作载具 或者理解成一个自动驾驶模块 未来去自动驾驶航模 ) 我在设想它的设计模式 我感觉使用状态机模式似乎是比较合适的 (我还没想好怎么后续结合agent 可以先设计一版简单的状态机模式) 完成每个类的类型 接口 语义设计 第一版尽量简单一点
>
> 帮我完善一下类型接口的设计 我的初步设计如下:
> 有一个总loop/runtime类型
> 里面有一些控制接口 比如 get current state , set parameters for state
> state 的注册接口 register("xxxx") state state的切换 switch(xxxx)
> runtime 功能 比如 runtime.video_stream
>
> 似乎整体的循环迭代 和状态切换也在这个类型里面实现
> 比如 while true current_state.on_update(runtime)
>
> state 类型 类似一个逻辑包装器
> on_enter
> on_exit
> on_update(runtime) (在loop 里面迭代调用的
> on_event_call(event)
>
> 感觉需要两种触发模式/或者也可以混合在一种里面? 但总之自动驾驶这种可能急需要有一个时刻推送的执行系统 又需要一个紧急事件的处理系统 (虽然似乎也能在loop里面处理事件 if runtime.event 检查 但是我建议做成两种入口系统 一个是主动推送 loop 高频 另一个则是 回调式 event来了 再call 因为这是两种不同的调用模式 (stream 推送 和 event call back
>
> 可能还需要一个功能 类似controller 检查一下当前的状态 比如现在是在机场准备起飞? 还是已经在天上了 或者在游戏菜单里
> 我建议把这个做成一个特殊的state/类型 而不是放在loop 或者 普通的state里面
> 比如它暴露 check_current_state(runtime, assume_state)
>
> state切换始终由runtime/loop 完成
> 但是这个类型需要注意 如何让它不要变成万能controller 完成所有的状态检查和切换
> 也许更合理的设计是 每个state 加入 on check检查函数 再on enter 的时候进行检查 如果检查失败 就进入 这个switch特殊类型 它判断当前类型 然后再选择进入的类型 (相当于check逻辑放到每个类型里面执行 确保时刻状态正确

## A: 第一版不要追求万能 Controller，而要让 Runtime 只维护机械规则

我觉得你的方向是对的，而且这里很适合把前面讨论的原则直接用上：**第一版不要追求“万能自动驾驶 Controller”，而是让 Runtime 只维护机械性的状态机规则，每个 State 自己拥有局部语义。**

我会对你的设计做一个关键调整：**不要把“检查当前究竟属于什么 State”做成一个特殊 SwitchState。** 否则它非常容易重新长成你担心的万能 Controller。更干净的办法是把“状态是否仍然成立”作为 State contract 的一部分，同时单独保留一个很小的 `StateResolver`，只在“当前状态已经无法解释世界”时负责重新定位。

这样第一版其实只需要 6 个核心概念：

```python
class Runtime: ...
class State: ...
class Transition: ...
class StateRegistry: ...
class StateResolver: ...
class Event: ...
```

### 1. Runtime：调度、生命周期和切换的唯一入口

```python
class Runtime:
    """
    状态机的执行环境。
    负责生命周期、调度、状态切换、事件分发。
    不负责具体驾驶逻辑。
    """

    current_state: "State"
    states: "StateRegistry"

    video_stream: "VideoStream"
    controller: "VehicleController"
    events: "EventBus"

    def run(self): ...
    def switch(self, state_id: str, **params): ...
    def dispatch(self, event: "Event"): ...
```

这里我甚至建议 `Runtime` 不要出现：

```python
check_current_state()
```

因为 Runtime 应该尽可能 dumb：

```text
Runtime 知道：
“现在运行哪个 State”

但不知道：
“飞机现在究竟是在跑道还是天空”
```

Runtime 负责的是：

```text
生命周期
调度
状态切换
事件分发
服务访问
```

而“什么叫起飞状态”“什么叫巡航状态”，应该由各个 State 自己解释。

### 2. State：每个状态拥有自己的局部语义

你的接口已经基本正确，我会稍微调整成：

```python
class State:
    id: str

    def on_enter(self, runtime: Runtime, **params):
        pass

    def on_update(self, runtime: Runtime, dt: float):
        pass

    def on_event(self, runtime: Runtime, event: Event):
        pass

    def on_exit(self, runtime: Runtime):
        pass

    def check(self, runtime: Runtime) -> bool:
        """
        当前现实世界是否仍符合这个 State 的假设。
        """
        return True
```

这里 `check()` 很重要。

例如：

```python
class TakeoffState(State):

    def check(self, runtime):
        return (
            runtime.telemetry.on_runway
            and not runtime.telemetry.airborne
        )
```

而：

```python
class CruiseState(State):

    def check(self, runtime):
        return runtime.telemetry.airborne
```

这样你刚才担心的：

> 谁负责判断“我现在到底在哪里？”

就没有集中进 Controller。

而是：

```text
TakeoffState
    ↓
只知道“什么叫 Takeoff”

CruiseState
    ↓
只知道“什么叫 Cruise”

TaxiState
    ↓
只知道“什么叫 Taxi”
```

这是一种非常好的 **local knowledge**。

### 3. 高频更新与事件入口：语义上分开，执行上串行

你说的：

```text
高频 stream/update
+
event callback
```

非常适合控制系统。

逻辑上：

```text
                    Runtime
                       │
            ┌──────────┴──────────┐
            │                     │
       periodic update          events
            │                     │
            ▼                     ▼
       on_update(dt)          on_event(event)
            │                     │
            └──────────┬──────────┘
                       ▼
                 Current State
```

例如：

```python
while runtime.running:

    runtime.update_sensors()

    runtime.dispatch_pending_events()

    state = runtime.current_state

    if not state.check(runtime):
        runtime.resolve_state()
        continue

    state.on_update(runtime, dt)
```

与此同时：

```text
CollisionWarning
EngineFailure
TargetLost
ConnectionLost
UserAbort
```

这些属于 Event。

所以：

```python
state.on_event(runtime, event)
```

会非常自然。

不过第一版我建议**不要真的让 Event callback 在另一个线程里直接调用 State**。

也就是说，外部可以异步：

```text
Sensor Thread
     ↓
Event arrives
```

但先进入：

```text
Event Queue
```

然后由 Runtime loop 统一处理：

```text
Sensor → Event Queue → Runtime → State.on_event()
```

这样状态机本身仍然是**单线程、顺序执行**的。

否则非常容易出现：

```text
Thread A:
TakeoffState.on_update()

         同时

Thread B:
TakeoffState.on_event()
    ↓
runtime.switch(EmergencyState)

         同时

Thread A:
继续操作已经退出的 TakeoffState
```

第一版完全没必要碰这种 concurrency complexity。

所以你的“双入口”在**语义上保留**：

<div class="math-display">\[
\text{Update} + \text{Event}
\]</div>

这里 `Update` 表示周期性更新入口，`Event` 表示外部事件入口；但在执行层最好统一串行化。

### 4. State 可以提出切换请求，但只有 Runtime 能 commit

State 可以**请求**切换：

```python
runtime.switch("cruise")
```

但我甚至建议第一版再严格一点：

```python
return Transition("cruise")
```

例如：

```python
class TakeoffState(State):

    def on_update(self, runtime, dt):

        if runtime.telemetry.airborne:
            return Transition("climb")

        runtime.controller.throttle(1.0)
```

Runtime 收到：

```python
Transition("climb")
```

才执行：

```text
Takeoff.on_exit()
        ↓
current_state = Climb
        ↓
Climb.on_enter()
```

也就是：

```python
result = current_state.on_update(self, dt)

if isinstance(result, Transition):
    self.switch(result.target, **result.params)
```

我更喜欢这个设计，因为它把：

> **State 提出决策**

和：

> **Runtime 修改状态机**

严格分开了。

这和前面讨论的：

```text
LLM decides what
Runtime guarantees how
```

其实是同一种思想。不过在当前第一版中，`State` 仍然是确定性的，暂时还不需要 LLM。

### 5. StateResolver：重新定位现实状态，而不是成为万能 Controller

假设：

```text
current_state = Takeoff
```

但是程序刚刚启动，画面其实是：

```text
Game Menu
```

那么：

```python
TakeoffState.check(runtime)
```

返回：

```python
False
```

这时候我不建议：

```text
Takeoff → SwitchState → Menu
```

因为 **SwitchState 并不是现实世界里的一个状态**。

它其实是一个 service：

```python
class StateResolver:

    def resolve(
        self,
        runtime: Runtime,
        assume: State | None = None
    ) -> str | None:
        ...
```

但最重要的是：

### Resolver 不应该自己知道所有状态的判断逻辑

错误方向：

```python
class StateResolver:

    def resolve(runtime):

        if runway_detected and speed < 10:
            return "taxi"

        if runway_detected and speed > 100:
            return "takeoff"

        if altitude > 100:
            return "cruise"

        if menu_detected:
            return "menu"

        ...
```

这就是你担心的 **万能 Controller**。

正确方向是让 Registry 提供候选 State：

```python
for state in runtime.states:

    confidence = state.match(runtime)

    if confidence > best:
        best = confidence
        candidate = state
```

这里我会进一步把刚才的：

```python
check() -> bool
```

升级一点：

```python
match(runtime) -> float
```

`check()` 更适合回答“当前状态是否仍然成立”；`match()` 更适合在多个候选状态之间进行重新定位。

例如：

```python
class CruiseState(State):

    def match(self, runtime) -> float:

        if not runtime.telemetry.airborne:
            return 0.0

        if runtime.telemetry.altitude > 100:
            return 0.95

        return 0.6
```

于是：

```text
StateResolver
      │
      │ ask everyone
      ▼
┌───────────────────────────────┐
│ MenuState        → 0.00       │
│ TaxiState        → 0.05       │
│ TakeoffState     → 0.20       │
│ ClimbState       → 0.72       │
│ CruiseState      → 0.94       │
└───────────────────────────────┘
              │
              ▼
           Cruise
```

这样新增：

```python
LandingState
```

只需要：

```python
registry.register(LandingState())
```

并实现：

```python
LandingState.match()
```

**Resolver 完全不需要修改。**

这正好就是前面讨论的“注册机制消除 Node dependency”。

### 6. 第一版的完整结构

```text
                         Runtime
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
     StateRegistry      EventQueue        Services
          │                                  │
          │                         ┌────────┼────────┐
          │                         ▼        ▼        ▼
          │                       Video   Sensors  Control
          │
          ▼
   ┌──────────────┐
   │ CurrentState │◄──────────────┐
   └──────┬───────┘               │
          │                       switch
     ┌────┴─────┐                  │
     ▼          ▼                  │
 on_update   on_event              │
     │          │                  │
     └────┬─────┘                  │
          ▼                        │
      Transition ──────────────────┘


如果 CurrentState.match() 失败：

        StateRegistry
              │
              ▼
        StateResolver
              │
       ask every State
              │
              ▼
      best match State
              │
              ▼
       Runtime.switch()
```

核心类型第一版甚至就这些：

```python
class Runtime: ...
class State: ...
class Transition: ...
class StateRegistry: ...
class StateResolver: ...
class Event: ...
```

然后把真正硬件/游戏环境相关的东西全部放进 Runtime services：

```python
runtime.video
runtime.telemetry
runtime.control
runtime.clock
runtime.events
```

### 7. 第一版应先验证的闭环

第一版先不要加入 Agent、Behavior Tree、复杂的父子状态、并行状态、插件系统甚至 LLM。现在最值得先验证的是一个非常干净的核心：

<div class="math-display">\[
\boxed{
\text{Sense}
\rightarrow
\text{State}
\rightarrow
\text{Decide}
\rightarrow
\text{Control}
\rightarrow
\text{Sense}
}
\]</div>

其中：

- `Sense`：读取视频、遥测和环境输入；
- `State`：解释当前处于哪个局部状态；
- `Decide`：根据状态规则提出控制或转换请求；
- `Control`：将请求变成车辆/飞机控制量。

外加：

<div class="math-display">\[
\text{Event}\rightarrow\text{State}
\]</div>

表示事件进入当前状态；以及：

<div class="math-display">\[
\text{State mismatch}
\rightarrow
\text{Resolver}
\rightarrow
\text{State}
\]</div>

表示当前状态假设失效后，Runtime 重新定位现实状态。

如果这个骨架设计干净，以后 Agent 非常容易插进去——甚至很可能只是把某些 State 内部的决策器，或者 `StateResolver` 的一部分，替换成更高层的 Agent，而 Runtime、Event、State lifecycle 和硬实时控制部分完全不用推倒重来。

## Q: 这个状态机自动驾驶框架如何与 Agent/LLM 结合？Agent 是否应该直接决定上下左右飞行？

> 我觉得这个架构设计的非常好 不过我感觉这个架构非常符合人工设计这种 目标明确的任务 (自动驾驶) 不过我感觉它和我们之前讨论的Agent 似乎整体思维又不太一样 (似乎整体思维上就不太一样 感觉这确实是很明确的状态 以及要做什么/操作流程 而不是给一堆能力让agent来选择 ) 你觉得我们还能做什么 把这个架构改成agent/llm 更适配的. 不过我觉得也许这种任务就不是很适合llm 因为大量的任务是专用的操作执行流程 比如起飞降落 这似乎更适合专用模型/专用流程 (不过反正是游戏里 我们是否能更大胆一点给llm 暴露一些接口 让llm自己做决定 比如上下左右飞行 而不是把做什么目标限制住

## A: 不要为了 Agent 化而拆掉状态机，应按时间尺度和抽象层级组合

对，我觉得这里不应该为了“Agent 化”而强行把状态机拆掉。这个任务恰好能帮助我们区分两类系统：

> **自动驾驶更像 control system；通用 Agent 更像 decision system。**

它们真正适合结合的地方，不是让 LLM 替代整个状态机，而是**重新划分决策的时间尺度和抽象层级**。

你现在的设计：

```text
Sense
  ↓
State
  ↓
专用控制逻辑
  ↓
Actuator
```

对于：

```text
起飞
降落
保持高度
姿态稳定
航向保持
失速恢复
```

这些任务非常合理。因为这里很多东西本质是连续控制：

<div class="math-display">\[
u_t=f(x_t,x^*)
\]</div>

其中：

- `t` 是当前时刻；
- `x_t` 是当前系统状态，例如高度、速度、姿态和航向；
- `x^*` 是目标状态或期望状态；
- `u_t` 是时刻 `t` 的控制输入，例如副翼、升降舵、方向舵和油门；
- `f` 是控制器，根据当前状态与目标状态计算控制输入。

控制器需要几十 Hz 甚至更高频率地根据误差调整控制量。让 LLM 每帧决定：

> 副翼左 12%、升降舵上 7%……

其实是在让语言模型干一个控制器/专用策略模型更擅长的工作。

但这不意味着 LLM 没有很大的空间。

### 1. 把 State 的一部分重新解释成 Capability

现在可能是：

```text
TakeoffState
    on_enter()
    on_update()
    on_exit()

CruiseState
    ...

LandingState
    ...
```

Runtime 决定：

```text
Taxi → Takeoff → Climb → Cruise → Landing
```

这还是经典 FSM。

可以把其中一部分重新解释成：

```text
Capabilities

takeoff()
land()
fly_to(position)
follow(target)
hold_altitude(h)
turn_to(heading)
evade(threat)
return_to_base()
```

每个 capability 内部仍然完全可以是：

```text
State Machine
+
PID / MPC
+
CV
+
专用模型
```

LLM 根本不需要知道这些内部细节。

于是架构变成：

```text
                     LLM Agent
                         │
                    Goal / World
                         │
                         ▼
                 choose capability
                         │
       ┌─────────────────┼─────────────────┐
       ▼                 ▼                 ▼
    Takeoff            Navigate           Land
       │                 │                 │
      FSM               FSM               FSM
       │                 │                 │
  controller        controller        controller
       └─────────────────┼─────────────────┘
                         ▼
                      Aircraft
```

这时候就和前面讨论的 Harness 思想重新统一了：

> **上层开放世界决策，下面封装确定性能力。**

### 2. 可以给 LLM 暴露不同抽象层级的 Action Space

既然是在《战争雷霆》这样的游戏里做实验，能不能更大胆地直接把上下左右飞行暴露给 LLM？

**可以，而且作为实验非常值得。**

但我不会只设计一套接口，而会故意设计成不同抽象层级的 capability。

#### Level 0：直接执行器控制

```text
set_pitch(x)
set_roll(x)
set_yaw(x)
set_throttle(x)
```

#### Level 1：离散操纵动作

```text
pitch_up()
roll_left()
turn_right()
increase_throttle()
```

#### Level 2：中层控制目标

```text
hold_heading(270)
hold_altitude(1200)
climb_to(2000)
turn_to(target)
```

#### Level 3：高层驾驶能力

```text
takeoff()
land()
navigate_to(point)
follow_aircraft(target)
return_to_base()
```

这样 LLM 的 action space 不是只有：

```text
takeoff / land
```

也不是只有：

```text
W A S D
```

而是：

<div class="math-display">\[
\mathcal A
=
A_{\mathrm{low}}
\cup
A_{\mathrm{mid}}
\cup
A_{\mathrm{high}}
\]</div>

其中：

- `𝒜` 是 LLM 可以选择的总动作空间；
- `A_low` 是直接操纵层动作；
- `A_mid` 是中层控制目标；
- `A_high` 是高层驾驶能力。

这会变得非常有意思。

### 3. 为什么这种设计可能比纯 FSM 更强？

假设游戏里发生：

```text
正在返航
↓
发现前方山体
↓
左侧出现飞机
↓
高度不足
↓
机场方向在右后方
```

FSM 很容易开始增长：

```text
RETURNING
→ OBSTACLE_AVOIDANCE
→ TRAFFIC_AVOIDANCE
→ CLIMBING
→ REALIGN
→ RETURNING
```

然后你又开始写越来越多的 transition。

Agent 可以看到 world state：

```text
altitude: 430m
heading: 270°
terrain_ahead: 380m
aircraft_left: 200m
airport: bearing 110°, distance 8km
fuel: 31%
```

同时看到：

```text
Capabilities:
  climb_to()
  turn_to()
  hold_altitude()
  navigate_to()
  land()
  set_roll()
  set_pitch()
  ...
```

然后自己决定：

```text
先爬升
→ 向右转
→ 重新导航机场
```

这里前面关于动态 Edge 的思想又重新出现了。令 `Edge_t` 表示时刻 `t` 实际选择的能力关系，`Context_t` 表示当前上下文，`Capabilities_t` 表示当前可用能力集合，则可以写成：

<div class="math-display">\[
\text{Edge}_t
=
\pi_{\mathrm{LLM}}(\text{Context}_t,\text{Capabilities}_t)
\]</div>

**你不再需要显式定义所有异常情况下的状态转移。**

但这里的“减少显式转移”不等于“让 LLM 直接绕过 Runtime 控制执行器”。LLM 仍然应该通过 Capability Layer 或安全 Runtime 提出目标，不能直接突破硬约束。

### 4. 不要让 LLM 成为真正的飞控：三个时间尺度

这里最好区分三个时间尺度。

#### 最底层：快速、数值化、安全相关

```text
100Hz+
```

例如姿态稳定、执行器、安全限制。这一层应该完全 deterministic / 专用 controller。

#### 中间层：轨迹和控制目标

```text
5~50Hz
```

例如：

```text
保持航向
跟踪轨迹
控制爬升率
转向目标
```

这里可以是 PID、MPC、专用神经网络或其他 controller。

#### 最上层：慢速、语义化、目标决策

```text
0.2~2Hz
```

甚至更慢：

```text
我现在应该干什么？
我要去哪？
现在应该返航还是继续？
当前计划失败了，换什么办法？
```

这里才特别适合 Agent。

因此，更准确的结构是：

```text
             Slow / Semantic

                 LLM
                  │
              Goal/Plan
                  │
                  ▼
        Capability Controller
                  │
            Target / Setpoint
                  │
                  ▼
             Flight Control
                  │
          actuator commands
                  │
                  ▼

             Fast / Numeric
```

这个结构其实比“FSM vs Agent”更准确。

它们不是互斥的。

### 5. State 仍然可以保留，但不一定继续负责下一步选择

甚至可以做一个很有意思的变化：

**State 不再决定下一 State。**

例如：

```python
class LandingState(State):

    def match(self, runtime):
        ...

    def on_update(self, runtime, dt):
        ...
```

它主要告诉系统：

> “当前世界处于 Landing。”

但是下一步：

```text
Landing → ?
```

未必由：

```python
LandingState.on_update()
```

决定。

可以由：

```text
State
   +
WorldModel
   +
Goal
   +
Capabilities
       ↓
     Agent
       ↓
Action / Capability
```

决定。

于是 State 从：

> **Controller**

退化成：

> **World semantic representation**

不过这只适合那些不需要硬保证的语义状态。失联、人工接管、执行器限制、紧急刹车、地理围栏和 failsafe 等仍应由 Runtime 或 Safety Layer 强制维护。

### 6. 状态机、专用模型和 LLM 的最终分层

```text
┌─────────────────────────────────────┐
│             Agent / LLM             │
│                                     │
│  Goal + World Model + Capabilities  │
│             ↓                       │
│       “下一步做什么？”              │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│          Capability Layer           │
│                                     │
│ takeoff / land / navigate / evade   │
│ follow / climb / turn / ...         │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│          Control / Skill            │
│                                     │
│ PID / MPC / FSM / Neural Policy     │
│ trajectory / perception             │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│           Safety Runtime            │
│                                     │
│ limits / watchdog / override        │
│ event / lifecycle / failsafe        │
└────────────────┬────────────────────┘
                 ▼
               Game
```

我觉得这比直接：

```text
FSM → 换成 LLM
```

漂亮得多。

因为 **FSM、专用模型和 LLM 被放到了各自真正擅长的位置。**

而且现在是在游戏里做实验，这反而给了一个现实自动驾驶项目不敢有的自由：完全可以故意同时暴露 `turn_to(heading)` 和更底层的 `set_roll()/set_pitch()`，观察 LLM 在什么情况下愿意调用高级 capability、什么时候自己组合低级动作。

这甚至可以成为一个很有价值的实验问题：

<div class="math-display">\[
\boxed{\text{Agent 应该看到多低层级的 Action Space？}}
\]</div>

太高层，Agent 几乎只是 FSM transition selector；太低层，它又会被连续控制问题淹没。**找到中间那个抽象层，可能恰恰是整个系统最有意思的设计部分。**

## 总结：这不是“FSM 或 Agent”，而是分层控制系统

这份讨论最终形成的不是一个“用 LLM 替换状态机”的方案，而是一个分层结构：

```text
感知 / 遥测
    ↓
状态识别与 Runtime
    ↓
状态机 / Capability Controller
    ↓
PID / MPC / 专用策略 / 执行器控制
    ↓
Safety Runtime
    ↓
载具或游戏
```

Agent 位于更高层：

```text
目标 + 世界模型 + 可用能力
            ↓
          Agent
            ↓
      Capability / Setpoint
            ↓
      确定性控制系统
```

第一版最应该验证的是：

1. Runtime 能否稳定驱动 State 生命周期；
2. `on_update` 与事件队列能否在同一执行上下文中顺序处理；
3. State 能否只维护自己的局部语义；
4. Transition 是否只能由 Runtime commit；
5. StateResolver 是否能通过注册机制工作，而不变成万能 Controller；
6. 控制接口是否能让上层 State 与底层游戏输入/未来航模执行器解耦。

Agent/LLM 不是第一版必须加入的部分。它更适合在这个确定性骨架稳定后，作为慢速的目标选择器、计划器或异常处置器逐步接入。

## 整理说明：适用范围与安全边界

1. 本笔记首先面向《战争雷霆》这类游戏或仿真环境中的架构实验。这里的接口和频率是概念设计，不是经过飞行器认证的控制参数。

2. “高频更新 + 事件入口”是两种调用语义，不等于必须使用两个线程。第一版推荐让传感器线程把事件放入队列，再由 Runtime 单线程顺序处理。

3. `StateResolver` 的职责是重新定位当前现实状态，不是集中编写所有状态判断逻辑。候选 State 自己提供 `check()` 或 `match()`，才能保持局部知识。

4. 如果未来迁移到真实航模，`control` 应保持为抽象接口，并在状态机之下加入独立的确定性 Safety Runtime，用于失联、人工接管、限幅、地理围栏、看门狗和 failsafe。普通 State 或 Agent 不应绕过这层直接操作执行器。

5. 本文中的 Agent 结合方案是架构探索：LLM 可以选择目标、能力或 setpoint，但不应直接承担高频姿态稳定和未经约束的执行器控制。
