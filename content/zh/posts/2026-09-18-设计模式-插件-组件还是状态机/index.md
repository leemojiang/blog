---
title: "设计模式：插件、组件还是状态机？"
date: 2026-09-18
draft: false
math: true
tags:
  - 插件架构
  - 组件
  - 状态机
  - Agent
categories:
  - 软件架构
description: >-
  从 DeepSeek Harness 的插件架构出发，比较插件、组件、状态机和 Agent 控制流各自解决的问题，以及它们如何组合。
---

本笔记保留“分析插件与状态机”这段讨论的原始问答结构。每个问题先保留原始提问，再尽量保留原回答中的解释、代码和图示；必要的事实核验与补充放在“整理说明”中，不替换原始讨论。

## Q: DeepSeek Harness 的插件设计模式和状态机设计模式有什么关系与区别？为什么 DeepSeek Harness 不采用状态机作为主要设计模式？

> 我有点好奇 deepseek harness这种插件设计模式 和 状态机设计模式的关系和区别是什么? 我感觉两者似乎很接近 但是又有点区别. 比如为什么 deepseek harness不采用状态机的设计模式? 我个人的理解是这样的,插件设计模式主要体现在功能的重新组合上,比如每个位置可能需要依赖组合的功能,但是它的控制流程基本是固定而简单的 (agent 一个loop 中间似乎不会在控制流程上面来回切换跳跃 有点像是简单的状态机 只有一个环线 而没有复杂的树状结构) 但是状态机模式就是适合功能固定,我不需要给他增加新的能力,但是控制状态时网状的 可能会越来越复杂.

## A: 插件架构与状态机不是同一个维度的设计模式

对，你这个理解已经非常接近本质了。更准确地说：**插件架构和状态机不是同一个维度的设计模式。**

> **插件架构解决“系统由哪些能力组成、这些能力如何替换/组合”；状态机解决“系统现在处于什么状态、下一步允许转移到哪里”。**

所以它们并不互斥。事实上，DeepSeek Harness 自己内部就有状态机：Cordis 的每个插件 Fiber 都经历类似 `PENDING → LOADING → ACTIVE → UNLOADING → DISPOSED` 的生命周期状态机。但 **Agent 的主控制流没有被设计成一个大型显式 FSM**。这一点正好对应你观察到的现象。

### 1. 可以把两者放在两个坐标轴上理解

假设我们有一个 Agent：

```text
                 “系统有什么能力？”
                        │
                        │  插件 / 组合
                        ▼
        ┌──────── Agent Runtime ────────┐
        │                               │
        │  LLM   Tools   FS   Shell    │
        │   ↑      ↑     ↑      ↑      │
        │ plugin plugin plugin plugin   │
        │                               │
        └───────────────────────────────┘
                        │
                        │
              “程序下一步去哪？”
                        │
                        ▼
                   控制流 / 状态机
```

插件系统更偏**横向扩展能力**，FSM 更偏**纵向组织流程**。

所以你说：

> 插件主要体现在功能重新组合；状态机适合功能比较固定、但控制状态复杂。

这个判断作为工程直觉是非常好的。只是我会稍微修正成：

**插件架构适合“能力集合变化频繁”；状态机适合“合法状态转移本身就是业务核心”。**

### 2. 为什么 DSH 的 Agent Loop 不需要复杂状态机？

DSH 的核心 loop 可以抽象成一个很简单的循环：

1. 构造请求；
2. 调用 LLM；
3. 如果模型产生工具调用，就执行工具；
4. 把工具结果写回 Session；
5. 继续下一轮，直到得到最终结果。

抽象之后差不多就是：

```text
用户输入
   ↓
┌───────────────┐
│ Assemble      │
│ Context       │
└───────┬───────┘
        ↓
     Call LLM
        ↓
   ┌────┴─────┐
   │          │
final?     tool calls?
   │          │
   ↓          ↓
  END     Execute Tools
              │
              └──────────→ Call LLM
```

甚至可以粗略写成：

```ts
while (!finished) {
    request = buildRequest(session)

    response = await llm(request)

    if (response.toolCalls) {
        results = await tools.execute(response.toolCalls)
        session.append(results)
    } else {
        break
    }
}
```

所以你说它像：

> “一个简单状态机，只有一个环线。”

我觉得这个描述很准确。

它当然存在 `turn/start`、`step/start`、LLM streaming、tool execution、`turn/end` 等生命周期，但**这些状态之间的拓扑结构没有复杂到值得把整个 Agent 建模成一个巨大 FSM**。

DSH 的 `agent-loop` 可以被看作具体的循环实现，而其他复杂行为——compaction、retry、sandbox、permission、subagent、UI 等——尽量放到 loop 外面的插件和事件扩展点里。

这实际上是一个非常重要的架构选择：

```text
方案 A：把复杂度放进状态机

                  ┌→ RETRY ─────┐
                  │             │
INPUT → THINK → TOOL → VERIFY → THINK
           │       │       │
           │       ↓       └→ COMPACT
           │     APPROVAL
           │       │
           └→ SUBAGENT
                  │
                WAIT
                  ↓
                RESUME
```

久而久之可能变成：

```text
State × State × State × State
```

而 DSH 更倾向于：

```text
             Agent Loop
          ┌──────────────┐
          │ LLM → Tool   │
          │  ↑      ↓    │
          └──┴──────┘
                 │
      ┌──────────┼──────────┐
      ↓          ↓          ↓
 permission    retry     compaction
   plugin      plugin       plugin

      ↓          ↓          ↓
 sandbox     telemetry    subagent
 plugin       plugin       plugin
```

**让主循环保持“笨”和稳定，把复杂度向插件边界扩散。**

### 3. 一个例子：写代码 Agent

假设你要做一个“写代码 Agent”。

如果采用**大型状态机思路**，可能设计成：

```text
IDLE
 ↓
PLANNING
 ↓
CODING
 ↓
TESTING
 ├── success → REVIEWING
 │               ↓
 │              DONE
 │
 └── failed → DEBUGGING
                ↓
              CODING
```

然后业务逻辑大量变成：

```ts
switch (state) {
  case "PLANNING":
  case "CODING":
  case "TESTING":
  case "DEBUGGING":
  ...
}
```

如果后来增加：

```text
用户审批
Git 操作
浏览器测试
子 Agent
部署
代码 review
```

状态图可能开始爆炸：

```text
CODING_WAITING_APPROVAL
TESTING_WAITING_APPROVAL
DEBUGGING_WAITING_SUBAGENT
DEPLOYING_RETRYING
...
```

这就是典型的 **state explosion**。

DSH 的思想则更接近：

```text
Agent Loop

LLM
 ↓
Tool
 ↓
LLM
 ↓
Tool
 ↓
...
```

至于：

```text
写代码？
→ filesystem/tool plugin

测试？
→ shell plugin

Git？
→ git capability/plugin

审批？
→ permission plugin

失败重试？
→ retry plugin

context 太长？
→ compaction plugin

需要子 Agent？
→ subagent plugin
```

于是：

> **LLM 自己承担了过去一部分“状态转移决策”的工作。**

传统程序里：

```text
程序员：
TEST_FAILED → DEBUG
```

Agent 系统里：

```text
tool_result:
Tests failed: xxx

       ↓

LLM 看到这个事实

       ↓

LLM 自己决定：
“我要检查文件”
       ↓
read_file

       ↓

“发现 bug”
       ↓
edit_file

       ↓
run_test
```

换句话说，**很多原本需要程序员显式编码成 FSM transition 的控制决策，被模型吸收了。**

所以 DSH 才可以维持一个如此简单的外层循环。

### 4. 这不意味着 Agent Harness 不需要状态机

假设你做的不是通用 coding agent，而是一个严格业务流程：

```text
贷款申请
   ↓
身份验证
   ↓
信用审核
   ↓
风险审核
   ↓
人工审批
   ↓
签约
```

这里你绝对不希望：

```text
LLM:
“我觉得今天可以跳过身份验证，
直接签合同。”
```

这种情况下，FSM / workflow engine 才应该拥有最高控制权：

```text
                State Machine
                     │
          ┌──────────┼───────────┐
          ↓          ↓           ↓
       VERIFY      REVIEW      APPROVE
          │          │           │
          ↓          ↓           ↓
        Agent      Agent       Agent
          │          │           │
          ↓          ↓           ↓
       plugins     plugins     plugins
```

这里：

> **FSM 决定 Agent 什么时候能做什么；Harness 决定 Agent 拥有什么能力。**

这两个模式组合起来反而非常漂亮。

### 5. 一个二维模型

可以把“能力是否容易组合”和“控制流程是否复杂”放在一起看：

| | 控制流程简单 | 控制流程复杂 |
|---|---|---|
| **能力固定** | 普通程序 | **状态机 / Workflow** |
| **能力高度可组合** | **DSH / Plugin Harness** | **Plugin Harness + FSM / Workflow** |

DSH 正好落在左下到右下之间：**能力组合非常复杂，但核心 Agent 控制拓扑刻意保持简单。**

还有一个特别值得注意的点：**DSH 并不是“拒绝状态机”，而是在把状态机压缩到真正需要状态机的局部。** Cordis 插件生命周期本身就是 FSM；tool execution 有自己的阶段；Agent 有 lifecycle；Session 有事件演化。

所以整体架构不是：

```text
插件 OR 状态机
```

而更像：

```text
                 DSH
                  │
        ┌─────────┴─────────┐
        │                   │
   Composition          Orchestration
        │                   │
     Plugins           simple loop
        │                   │
        │             local state machines
        │                   │
        └─────────┬─────────┘
                  ↓
               Runtime
```

**它避免的是“用一个中央巨型状态机描述整个 Agent 系统”，而不是避免状态机本身。**

这其实也是理解 Cordis/DSH 时很容易差一点的地方：Cordis 看起来像一个调度框架，但它真正擅长的是**动态能力组合和生命周期管理**；Agent Loop 负责 orchestration，而 Session/Event 又负责记录状态事实。把这三个轴分开之后，DSH 的设计会清楚很多。

## Q1 整理说明：资料核验与补充

1. 原回答中关于 DSH 总体定位的表述与官方资料一致：DeepSeek Harness 将模型、工具、技能、会话、沙箱、存储、循环、调度和 UI 等能力作为插件组合，并由 Cordis 内核负责插件加载、卸载和依赖管理。参见 [DeepSeek Harness 官方介绍](https://www.deepseek.com/harness/)。

2. 原回答中关于 Cordis 插件生命周期状态机的表述已核验。官方教程列出了 `PENDING → LOADING → ACTIVE → UNLOADING → DISPOSED`，并说明依赖未就绪时插件会处于 `PENDING`，依赖消失时会卸载，依赖恢复后可以重新加载。参见 [Cordis 生命周期教程](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-tutorial/02-lifecycle-and-effects.md) 与 [中文文档](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/framework/index.zh.md)。

3. 原回答中“`agent-loop` 是具体循环实现，而其他行为尽量通过插件和扩展点提供”的表述也有官方代码依据：`agent-loop` 文档将其描述为具体的 Agent driver/loop，而其他部分通过抽象服务或插件扩展。参见 [dsh-agent-loop README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent-loop/README.md)。

4. “LLM 吸收一部分状态转移决策”是本次讨论中的架构解释，不应理解为 DSH 的官方定义。更严谨地说：DSH 通过 Harness、工具、Session 和插件提供能力与约束；LLM 在这些能力和上下文之上提出下一步行动，而 Runtime 仍然负责生命周期、权限、执行和其他必须保证的协议状态。

## Q: 有了 LLM 之后，某些状态似乎变成了能力；这算不算一种特殊的状态机？

> 我觉得这是一个特别微妙的设计选择 似乎某些状态变成了能力. 比如刚才你写的coding agent这个例子: compaction、retry、sandbox、permission、subagent 在状态机的视角下 似乎用状态建模是很正常的 但是有了llm 似乎变成了能力的视角 llm 就像一个超级选择器 这算不算也是一种特殊状态机的设计 我不考虑状态之间的复杂转移 只有一个超级选择器的循环 然后它每一步都可以自动选择下一步干什么 于是状态退化成了能力?

## A: 状态没有消失，而是从显式状态变成了隐式语义状态

对，而且我觉得你已经碰到了 **LLM Agent 架构和传统状态机之间最有意思的那个转变**。

我会稍微修正你最后一句：

> 与其说“状态退化成了能力”，不如说是：**原来由程序显式编码的状态转移函数，被部分替换成了 LLM 这个通用、上下文驱动的策略函数；很多原本需要成为控制状态的东西，因此可以降格成“上下文 + 可调用能力”。**

传统有限状态机可以抽象成：

```text
显式状态 s_t
      │
      ▼
确定性转移规则 δ(s_t, x_t)
      │
      ▼
下一个显式状态 s_{t+1}
```

这里：

- `s_t` 表示时刻 `t` 的显式状态，例如 `TESTING`；
- `x_t` 表示时刻 `t` 观察到的输入，例如测试失败；
- `δ` 是程序员预先写好的状态转移函数；
- `s_{t+1}` 是转移后的下一个状态。

Coding Agent 则越来越像：

```text
历史 H_t
   +
当前能力集合 𝒜_t
       │
       ▼
   LLM policy π
       │
       ▼
    action a_t
       │
       ▼
环境 observation o_t
       │
       ▼
新的历史 H_{t+1}
```

其中：

- `H_t` 是截至时刻 `t` 的 Session/history；
- `𝒜_t` 是当前可用的能力集合；
- `π` 是根据上下文选择行动的策略；
- `a_t` 是 LLM 选择的行动；
- `o_t` 是执行行动后得到的观察结果。

用强化学习中常见的记号，可以写成：

<div class="math-display">\[
 a_t \sim \pi_{\mathrm{LLM}}(H_t,\mathcal A_t)
\]</div>

这里的意思是：LLM 根据当前历史 `H_t` 和可用能力集合 `𝒜_t`，选择下一步行动 `a_t`。

状态转移则不再主要表现为：

```text
CODING → TESTING → DEBUGGING → CODING
```

而是表现为：

```text
             ┌─────────────┐
             │     LLM     │
             │ 超级选择器   │
             └──────┬──────┘
                    │
       ┌────────────┼─────────────┐
       ↓            ↓             ↓
    edit_file     run_test     read_file
       ↑            │             │
       └────────────┴─────────────┘
                    │
                 result
                    │
                    └────→ LLM
```

此时 `CODING / TESTING / DEBUGGING` **甚至不必作为 Runtime 中真实存在的 enum state**。

LLM 从历史中自己推断：

> “刚才测试失败 → 我现在实际上处于 debugging 阶段 → 应该 read_file。”

也就是说，**状态没有消失，而是从显式符号状态变成了隐式语义状态。**

### 这可以被看成一种“特殊状态机”，但更准确地说是 policy loop

如果把“状态机”定义得足够宽，当然可以这样理解。

传统 FSM：

```text
State = TESTING

if test_failed:
    State = DEBUGGING
```

Agent：

```text
Session:
  assistant: 我修改了 foo.ts
  tool: pytest
  result: test_x failed...
```

这里没有：

```text
state = DEBUGGING
```

但 LLM 阅读 Session 后形成了一个隐式判断：

```text
“现在需要 debug”
```

因此，严格一点说，这已经更像 **policy / agent-environment loop**，而不是经典有限状态机：

```text
传统 FSM

显式状态 S
   ↓
确定性/规则转移 δ
   ↓
显式状态 S'
```

变成：

```text
LLM Agent

完整历史 H
   ↓
LLM policy π
   ↓
选择 action
   ↓
环境 observation
   ↓
新的完整历史 H'
```

所以可以得到一个很有用的三分法：

> **Harness 定义 action space；LLM 定义 policy；Session/history 承载 state。**

例如，当前能力集合可以抽象成：

<div class="math-display">\[
\mathcal A_t=
\{\text{read},\text{write},\text{shell},\text{browser},\text{subagent},\ldots\}
\]</div>

其中 `𝒜_t` 表示时刻 `t` 可用的动作集合，集合中的每个元素都代表一种可调用能力。

### “状态开始变成能力”是什么意思？

例如传统 workflow 可能写成：

```text
NORMAL
  ↓ context too long
COMPACTING
  ↓
NORMAL
```

但 Agent Harness 可以把它表达成：

```text
LLM Loop
   │
   ├── normal context
   │
   └── compaction capability/interceptor
           ↓
       压缩 Session
           ↓
       回到同一个 loop
```

于是 `COMPACTING` 不再需要成为整个系统的一级状态。

同样：

```text
WAITING_PERMISSION
RETRYING
DELEGATING
SANDBOXED
```

很多东西都可以从：

> **“我现在是什么状态？”**

重新表达成：

> **“当前发生了什么事实？现在有哪些能力？什么约束生效？”**

然后交给通用循环继续处理。

### 但不是所有状态都能退化成能力

这是一个非常重要的边界。

**语义状态**很适合交给 LLM 隐式维护：

```text
正在 coding
正在 debugging
正在研究
现在应该测试
可能需要搜索
需要查看更多文件
```

因为这些状态本来就模糊，而且转移条件难以穷举。

但**协议/安全状态**通常仍应该显式维护：

```text
RUNNING / CANCELLED
AUTHORIZED / UNAUTHORIZED
TRANSACTION_OPEN / COMMITTED
LOCKED / UNLOCKED
SUBAGENT_RUNNING / FINISHED
```

不能让 LLM 看历史然后说：

> “根据上下文，我感觉用户应该已经授权了。”

这些状态必须由 Runtime 保证。

因此，成熟的 Agent 架构往往会形成这样的分层：

```text
                 Session / World State
                         │
                         ▼
                  ┌───────────┐
                  │    LLM    │
                  │   Policy  │
                  └─────┬─────┘
                        │ choose
          ┌─────────────┼──────────────┐
          ▼             ▼              ▼
        Tool         Subagent       Compaction
          │             │              │
          └─────────────┼──────────────┘
                        ▼
                    Observation
                        │
                        └────→ Loop

────────────────────────────────────────────
下面仍然存在硬状态机：

Permission FSM
Session lifecycle FSM
Tool execution FSM
Process lifecycle FSM
Transaction FSM
Plugin lifecycle FSM
```

所以 DSH 的巧妙之处不是“消灭状态机”，而是：

> **把过去那个中央巨型 FSM 拆掉。**
>
> **语义控制交给 LLM policy；能力交给插件；历史状态交给 Session；必须保证正确性的协议状态则保留成局部小型 FSM。**

如果没有 LLM，那个“超级选择器”最终还是得由程序员写：

```ts
if (...) ...
else if (...) ...
else if (...) ...
```

写着写着，它就重新变回状态机了。

而现在 LLM 本身提供了一个开放世界的、自然语言条件驱动的通用 transition policy。于是原本：

<div class="math-display">\[
\text{State Machine}
=
\text{State}+\text{Transition Rules}
\]</div>

可以被重新分解成：

<div class="math-display">\[
\boxed{
\text{Agent}
=
\text{State/History}
+
\text{LLM Policy}
+
\text{Action Space/Plugins}
+
\text{Hard Constraints}
}
\]</div>

这里的 `State/History` 表示事实与历史，`LLM Policy` 表示语义决策，`Action Space/Plugins` 表示可用能力，`Hard Constraints` 表示不能交给模型猜测的约束。

“只有一个超级选择器的循环”这个描述其实非常准确。复杂度并没有消失，而是从显式控制图转移到了模型 policy、上下文和 capability composition 里面。

## Q: 如何进一步判断哪些东西应该是状态，哪些东西应该是能力？

> 我觉得你后半段讲的特别好 这个语义分层就是我之前纠结的点. 你一下把这个给梳理清楚了 有些适合退化成能力 有些则还是应该保持状态的边界. 其核心边界可够理解为,llm 其实是一个超级隐式的节点选择器 没有llm 之前这个选择器如果手动编码 最后一定复杂度爆炸 不过有了llm之后 这部分反而适合退化成能力了 这是个非常微妙的边界

## A: 关键不在于它“看起来像不像状态”，而在于是否需要确定性保证

对，我觉得你现在已经把这个边界抓得非常准了。而且可以再提炼一步：**真正决定“状态还是能力”的，不是这个东西在业务语义上看起来像不像状态，而是它的转移是否需要由系统提供确定性保证。**

比如 `DEBUGGING` 看起来当然是一个状态：

```text
CODING → TESTING → DEBUGGING → CODING
```

但实际上系统并不在乎你是否真的进入一个叫 `DEBUGGING` 的状态。系统只在乎当前有这些事实：

```text
刚刚修改了代码
测试执行失败
错误信息是 xxx
当前可以 read / edit / run_test
```

LLM 根据这些信息完全可以隐式得出：

```text
“下一步应该 debug”
```

所以 `DEBUGGING` 这个显式节点其实是**可删除的**。

删除以后，原来挂在这个节点下面的东西：

```text
DEBUGGING
 ├─ read source
 ├─ inspect logs
 ├─ search symbols
 ├─ edit code
 └─ rerun tests
```

就变成：

```text
LLM
 │
 ├─ read_source
 ├─ inspect_logs
 ├─ search_symbols
 ├─ edit_code
 └─ run_tests
```

这就是你说的：

> **状态退化成能力。**

而 `LLM + context` 接管了原本状态节点承担的**选择功能**。

但反过来看 `AUTHORIZED / UNAUTHORIZED`：

```text
UNAUTHORIZED
     │
 user approves
     ↓
AUTHORIZED
```

这个节点不能删除。

因为如果删除它，变成：

```text
LLM
 ├─ execute_command
 ├─ delete_file
 └─ ...
```

然后要求 LLM 根据历史判断：

```text
“用户刚才是不是已经授权？”
```

整个系统的安全性质就改变了。

所以这里必须存在一个显式的权限状态。令 `s_permission` 表示权限状态，则：

<div class="math-display">\[
s_{\mathrm{permission}}
\in
\{\text{unauthorized},\text{authorized}\}
\]</div>

其中 `s_permission` 只能取“未授权”或“已授权”两种值。Runtime 而不是 LLM 决定状态转移：

<div class="math-display">\[
s'_{\mathrm{permission}}
=
\delta(s_{\mathrm{permission}}, event)
\]</div>

这里 `event` 是用户批准、撤销授权等事件，`δ` 是由 Runtime 实现的确定性转移规则，`s'` 是事件处理后的新权限状态。

LLM 最多只能提出：

> 我要执行 X。

Runtime 再判断：

> 当前状态允许不允许这么做？

所以我现在反而会把我们前面那个结论进一步压缩成一句很有用的设计原则：

> **需要“判断”的状态，可以考虑交给 LLM；需要“保证”的状态，应该留给状态机。**

例如：

| 问题 | 更适合 |
|---|---|
| 接下来应该 debug 还是继续实现？ | LLM |
| 要不要再搜索一个文件？ | LLM |
| 当前信息够不够，需要 subagent 吗？ | LLM |
| 失败以后是否值得换一种方法？ | LLM |
| 用户是否已经授权？ | FSM / Runtime |
| Tool 是否正在执行？ | FSM / Runtime |
| transaction 是否 commit？ | FSM / Runtime |
| Session 是否 cancelled？ | FSM / Runtime |
| 子进程是否已经退出？ | FSM / Runtime |

前一组本质是：

<div class="math-display">\[
\text{decision under semantics}
\]</div>

也就是需要结合上下文进行语义判断的问题。

后一组本质是：

<div class="math-display">\[
\text{invariant enforcement}
\]</div>

也就是系统必须持续保证的不变量。

这两个东西以前很容易被一起塞进一个 Workflow/FSM。LLM 出现以后，这两个问题终于可以分开：

```text
              LLM
       “我应该干什么？”
              │
              ↓
          choose action
              │
              ▼
       Runtime / FSM
       “你能不能这么干？”
              │
         ┌────┴────┐
         ↓         ↓
       allow      reject
         │
         ▼
      capability
         │
         ▼
     observation
         │
         └──────────→ LLM
```

因此，LLM 不只是让状态机变简单，也改变了“状态是否值得显式建模”的标准：

> **显式状态只保留那些 Runtime 必须保证的事实和协议。**

剩余大量语义状态可以留在 `History + LLM latent interpretation` 中。

这可能是传统 workflow architecture 走向 agent architecture 时最本质的架构变化之一：不是简单地“把 AI 塞进状态机”，而是因为出现了一个强大的隐式节点选择器，我们第一次可以把大量语义控制状态从程序结构里删除，只留下能力、事实和必须被系统强制保证的硬状态。

## Q: 插件模式、状态机模式和 Unity Component 如何维护？能否通过注册机制实现真正自由的组合？

> 这时候引入了一个新的问题 这种插件模式 和状态机模式都是怎么维护的? 我感觉插件模式的解耦似乎也只能做到: 插件系统负责插件的依赖 但是我加入了新的依赖 我还是需要修改现在的代码 比如调用另一个能力 只是如果这个能力不存在 被外部handle 了. 状态机似乎也是类似 我加入新的状态 可能需要在选择节点做修改 也可能需要对和这个状态相关的节点做修改 (这样能不能引入一个注册机制 比如 如果访问了不存在或者未实现的状态 有一个fallback?
>
> 与之对比的是 我之前做unity开发的时候 有一个组件机制 (它和插件机制的区别是什么?) 我感觉它确实和插件有点想 但是就像是刚才说的 就比如我加入了新的组件/能力 但是我还是要修改调用这个组件的代码 并且同时还要判断处理这个组件是否真的存在 比如动态添加组件 组件似乎并不能真的做到彻底自由的组合 这点让我很头疼

## A: 真正难解耦的不是节点，而是节点之间的协调关系

对，你现在碰到的是比“插件 vs 状态机”更底层的问题：

> **模块化可以降低耦合，但不能消灭“新功能必须被某个东西知道”这件事。**

你在 Unity 里觉得“明明用了 Component，为什么加一个能力还是得到处 `GetComponent`、判空、修改调用代码”，本质就是这个问题。

真正值得区分的不是只有 Plugin / FSM / Component，而是：

```text
谁负责发现新东西？
谁负责选择新东西？
谁负责调用新东西？
```

一旦把这三个问题拆开，Plugin、Component、FSM 和 LLM Harness 的区别会清楚很多。

### 1. 最朴素的组件系统，确实存在你说的问题

比如 Unity：

```csharp
class Enemy : MonoBehaviour
{
    void Update()
    {
        var weapon = GetComponent<Weapon>();

        if (weapon != null)
            weapon.Fire();
    }
}
```

后来加入：

```text
Shield
Magic
Jetpack
Stealth
```

最后：

```csharp
var weapon = GetComponent<Weapon>();
var shield = GetComponent<Shield>();
var magic = GetComponent<Magic>();
var jetpack = GetComponent<Jetpack>();

if (weapon != null) ...
if (shield != null) ...
if (magic != null) ...
```

虽然每个能力都 Component 化了：

```text
Enemy
 ├─ Health
 ├─ Weapon
 ├─ Shield
 ├─ Magic
 └─ Jetpack
```

但真正的控制器：

```text
EnemyController
```

仍然知道所有东西。

所以你会产生一种非常正确的不适感：

> **数据/实现解耦了，但控制耦合还在。**

这就是关键。

### 2. Plugin 和 Component 解决的是两个略有不同的问题

Component 更强调：

> **一个对象由什么行为/数据组成。**

例如：

```text
GameObject
 ├── Transform
 ├── Rigidbody
 ├── Collider
 ├── Health
 └── Weapon
```

所以它主要解决：

<div class="math-display">\[
\text{Object}
=
\text{Component}_1
+
\text{Component}_2
+
\cdots
\]</div>

这里的 `Object` 表示一个对象，`Component_i` 表示附着在它上面的第 `i` 个组件。这个问题本质上是 **composition**。

Plugin 更强调：

> **整个 Runtime 当前安装了什么能力。**

例如 DSH：

```text
Runtime
 ├── filesystem provider
 ├── shell provider
 ├── LLM provider
 ├── sandbox
 ├── MCP
 ├── subagent
 └── compaction
```

它主要解决：

<div class="math-display">\[
\text{Runtime}
=
\text{Plugin}_1
+
\text{Plugin}_2
+
\cdots
\]</div>

这里的 `Runtime` 表示系统运行环境，`Plugin_i` 表示当前装载的第 `i` 个插件。

两者结构确实很像，甚至工程实现都可能是：

```text
注册
依赖注入
生命周期
事件
Service lookup
```

主要区别是**作用域和语义**：

```text
Component
    └── 附着于某个 Entity / Object

Plugin
    └── 附着于 Application / Runtime / Context
```

所以 Cordis 的：

```ts
ctx.xxx
```

和 Unity 的：

```csharp
GetComponent<T>()
```

在思想上确实有亲缘关系，但它们服务的对象层级不同。

### 3. 调用方仍然知道被调用方

假设：

```text
A → B
```

把 B 做成 Plugin：

```text
A → IService
      ↑
    Plugin B
```

确实已经改善很多。

A 不知道：

```text
BPlugin
LocalB
RemoteB
MockB
```

但 A **仍然知道 `IService` 存在**。

也就是说：

<div class="math-display">\[
A \rightarrow \text{Interface}
\]</div>

这个依赖无法凭空消失。

所以：

> **Dependency Injection 解决的是“依赖谁的具体实现”，不是“是否存在依赖”。**

如果 A 的业务逻辑本身就是：

> 我要调用文件系统。

那它就必须知道某种：

```text
FileSystem capability
```

否则“调用文件系统”这件事情本身都无法表达。

### 4. 更进一步的解耦：注册 + 分发

不要：

```csharp
class Player
{
    void OnDamage()
    {
        var shield = GetComponent<Shield>();

        if (shield != null)
            shield.AbsorbDamage();
    }
}
```

而变成：

```text
Player
   │
   │ emit
   ▼
DamageEvent
   │
   ├──── ShieldComponent
   ├──── ArmorComponent
   ├──── AchievementComponent
   └──── SoundComponent
```

Player 只说：

```csharp
Emit(new DamageEvent(50));
```

至于谁响应：

```text
Shield:
    OnDamage(...)

Armor:
    OnDamage(...)

Sound:
    OnDamage(...)
```

由组件自己注册。

这时候加入：

```text
MagicBarrierComponent
```

Player **完全不用修改**。

这才是真正意义上的：

> **开放组合。**

### 5. 模块系统存在一个“解耦梯度”

可以从低到高看：

```text
① 直接调用

A → ConcreteB
```

最耦合。

然后：

```text
② Interface / DI

A → IB
     ↑
     B1
     B2
```

A 不知道具体实现。

然后：

```text
③ Optional capability

A → capability("B")?
```

不存在也没关系。

然后：

```text
④ Event / Hook

A → emit(Event)
       │
       ├→ B
       ├→ C
       └→ D
```

A 连消费者是谁都不知道。

再进一步：

```text
⑤ Registry / Dispatcher

A → “我要完成 X”
          │
          ▼
       Registry
      ┌───┼────┐
      B   C    D
```

A 只描述意图，由 Registry 选择实现。

再到 Agent：

```text
⑥ LLM Capability Selection

          available capabilities
                 ↓
Context ───────→ LLM
                 │
             semantic select
                 │
       ┌─────────┼─────────┐
       ↓         ↓         ↓
       B         C         D
```

这时候连：

```text
if situation X:
    call B
```

都不一定需要程序员写了。

这就是前面说的 **超级选择器**。

### 6. 状态机也有同样的问题：真正难维护的是 Transition

传统 FSM：

```text
A → B → C
```

加入 D：

```text
A → B → D → C
```

当然得改：

```text
B 的 transition
D 的 transition
可能还有 C
```

所以普通 FSM 的扩展性确实不高。

但你完全可以把它改成**注册式状态机**：

```ts
registry.register("idle", IdleState)
registry.register("running", RunningState)
registry.register("error", ErrorState)
```

然后：

```ts
transition("error")
```

Runtime：

```ts
const state = registry.get("error");

if (!state)
    fallback();
```

这就实现了你说的：

> 如果访问不存在/未实现状态，就 fallback。

完全可以。

甚至可以：

```text
StateRegistry
 ├── IDLE
 ├── RUNNING
 ├── DEBUGGING
 ├── RETRYING
 └── ...

unknown
   ↓
FallbackState
```

但这里有一个很重要的问题：

**你只是解决了“状态实现如何发现”，没有解决“什么时候应该进入这个状态”。**

也就是：

```text
register(DEBUGGING)
```

很容易。

真正麻烦的是：

```text
TEST_FAILED → DEBUGGING
```

谁决定？

如果还是：

```ts
if (testFailed)
    transition("debugging");
```

那么 coupling 仍然存在。

### 7. Node 容易解耦，Edge 很难解耦

假设有 `N` 个状态。状态实现本身可能非常漂亮：

```text
StateA
StateB
StateC
StateD
...
```

但是关系可能是：

<div class="math-display">\[
A\rightarrow B,
\quad
A\rightarrow C,
\quad
B\rightarrow D,
\quad
C\rightarrow A,
\quad
D\rightarrow B
\]</div>

复杂度其实藏在 **Edge**，不是 Node。

所以：

> Component/Plugin 的扩展问题通常是“谁调用谁”。
>
> FSM 的扩展问题通常是“谁能转移到谁”。

它们其实是同一个更抽象的问题：

<div class="math-display">\[
\boxed{\text{Relationship / Coordination}}
\]</div>

**节点很好解耦，边很难解耦。**

### 8. Unity Component 为什么没有给出真正的“自由组合”？

因为：

```text
Componentization ≠ Automatic Coordination
```

Component 只告诉你：

> Weapon 可以独立存在。

却没有告诉系统：

> Player 在什么情况下应该使用 Weapon。

所以：

```text
Player
 ├─ Weapon
 ├─ Shield
 └─ Jetpack
```

仍然需要一个：

```text
???
```

决定：

```text
什么时候 Fire
什么时候 Shield
什么时候 Fly
```

传统游戏里这个 `???` 通常就是：

```text
PlayerController
AIController
StateMachine
BehaviorTree
GameplaySystem
```

于是复杂度重新集中到 Controller。

这不是 Unity Component 设计失败，而是：

> **组合（Composition）和协调（Coordination）本来就是两个不同问题。**

### 9. LLM 提供了一个通用 Coordination Layer

传统架构：

```text
        Components
      A   B   C   D
       \\  |  /   /
        \\ | /   /
         \\|/   /
       Controller
          │
     巨量 if/else
```

状态机：

```text
       A ───→ B
       ↑     ↓
       D ←── C

程序员维护所有 transition
```

Event system：

```text
        Event Bus
       /   |   \\
      A    B    C
```

Registry：

```text
      Dispatcher
       / | | \\
      A  B C  D
```

而 Agent：

```text
          Context
             │
             ▼
        ┌─────────┐
        │   LLM   │
        │ semantic│
        │ selector│
        └────┬────┘
             │
       capability registry
       ┌────┼────┬────┐
       ↓    ↓    ↓    ↓
       A    B    C    D
```

以前最难解耦的：

<div class="math-display">\[
\boxed{\text{Edges}}
\]</div>

现在有一部分可以动态产生：

<div class="math-display">\[
\text{Edge}_t
=
\pi_{\mathrm{LLM}}(\text{Context}_t,\text{Capabilities}_t)
\]</div>

这里 `Edge_t` 表示时刻 `t` 由系统实际采用的协调关系，`Context_t` 表示当前上下文，`Capabilities_t` 表示当前可用能力集合，`π_LLM` 表示 LLM 的语义选择策略。

**节点仍然由程序员实现，但节点之间的一部分连接关系不再由程序员预先穷举。**

这可能才是 Harness 架构真正革命性的地方。

### 10. 但不要追求“彻底自由组合”

这一点反而非常重要。

如果两个模块真的完全互不知道：

```text
A    B
```

那么凭什么 A 加进系统以后就知道什么时候应该和 B 交互？

一定需要第三样东西：

```text
       Coordinator
       /         \\
      A           B
```

区别只是 Coordinator 是什么：

```text
硬编码 Controller
FSM
Behavior Tree
Event Bus
Rule Engine
Dependency Injection
Registry
Scheduler
LLM
```

**协调复杂度不可能消失，只能转移。**

LLM 很特殊，因为它允许我们把一部分 coordination complexity 从：

```text
source code
```

转移到：

```text
semantic reasoning
```

但安全、事务、一致性、生命周期这些东西仍然不能这么干。

所以整个问题可以归结成：

<div class="math-display">\[
\boxed{
\text{System}
=
\text{Components}
+
\text{Composition}
+
\text{Coordination}
+
\text{Constraints}
}
\]</div>

其中：

```text
Components
    ↓
“有哪些东西？”

Plugin / DI / Registry
    ↓
“这些东西如何装进系统？”

FSM / Controller / Event / LLM
    ↓
“什么时候谁和谁交互？”

Permission / Transaction / Lifecycle FSM
    ↓
“哪些事情绝对不能出错？”
```

你之前 Unity Component 那种“明明组件化了，却还是觉得耦合没有真正消失”的感觉，其实非常准确：**你解决的是 Composition，但真正让你痛苦的一直是 Coordination。**

而前面讨论 DSH 时所谓的“状态退化成能力”，本质上就是：**LLM 让一部分 Coordination 第一次可以不再以显式代码 Edge 的形式存在。**

## Q: LLM 是否带来了真正意义上的解耦和能力自由？

> 确实 我感觉一下清晰了 就是controller 控制的逻辑始终是无法解耦的,传统的办法最多把它做成通用的接口 使用注册和事件分发的机制来处理这些信息 但是比如什么时候分发这个消息? 这部分的复杂度始终是无法省略的 (起码需要知道有这个接口) 但是llm 真的有点像个万能系统 他把这部分复杂的逻辑完全吸收了.真的有点llm as system的意味 这意味着真正意义的解耦 和能力自由

## A: LLM 没有消灭 Coordination，而是把它变成运行时生成的语义关系

对，而且我觉得你这里说的 **“LLM as System”** 比单纯的 “LLM as Controller” 更值得琢磨。不过我会给“真正意义的解耦和能力自由”加一个很重要的限定：

> **LLM 没有消灭 coordination，而是把 coordination 从“静态代码中的显式依赖关系”变成了“运行时根据语义动态生成的关系”。**

这可能才是质变。

传统架构无论怎么抽象，最后总有某处必须写出关系。例如最直接的是：

```text
controller.fire()
```

接口化后变成：

```text
IWeapon.Fire()
```

事件化后变成：

```text
emit("attack")
```

注册化后变成：

```text
registry.dispatch("attack")
```

你确实可以让 Controller 不知道具体是谁响应，但它至少必须知道：

```text
“现在应该发出 attack 这个消息”
```

因此真正无法彻底解耦的是这一条**边**：

```text
当前情况 ─────────→ 应该触发 Attack
```

程序员必须事先知道这条边存在。

而 LLM Agent 出现以后，可以变成：

```text
                Runtime Context
                     │
                     ▼
             ┌──────────────┐
             │     LLM      │
             │ semantic     │
             │ coordinator  │
             └──────┬───────┘
                    │
        runtime-generated edge
                    │
                    ▼
              Capability X
```

甚至 Capability X 是后来才安装进去的，只要它把自己的语义描述暴露出来：

```text
name: inspect_database

description:
  Inspect the application's database
  when diagnosing persistent-state problems.
```

LLM 就有可能第一次看到它时直接理解：

> “哦，这个能力适合解决我现在的问题。”

这里没有程序员事先写：

```ts
if (persistentStateProblem)
    inspectDatabase();
```

也没有：

```text
PERSISTENCE_ERROR → DATABASE_INSPECTION
```

甚至没有预定义：

```text
PersistenceErrorEvent → DatabaseInspector
```

**这才是真正不同的地方。**

### 用图论描述这种变化

如果用图论来描述，传统模块化其实一直主要在解决 **Node 的可替换性**。

令：

<div class="math-display">\[
G=(V,E)
\]</div>

其中 `G` 表示系统的模块关系图，`V` 表示模块节点集合，`E` 表示模块之间的连接或协调关系。Plugin / Component / DI 可以让节点 `V_i` 很容易替换、注册和卸载，但边 `E` 仍然大量存在于源代码里。

于是你之前在 Unity 里的感觉就是：

> “节点明明已经完全 Component 化了，为什么系统还是不自由？”

因为真正束缚系统的是 **Edge**。

LLM 带来的变化是：

<div class="math-display">\[
\boxed{
E_t
=
\pi_{\mathrm{LLM}}(Context_t,\ Capabilities_t)
}
\]</div>

这里 `E_t` 表示时刻 `t` 实际采用的协调关系，`Context_t` 表示当前运行时上下文，`Capabilities_t` 表示当前可用能力集合，`π_LLM` 表示 LLM 根据语义选择关系的策略。

**Edge 第一次可以在运行时根据语义产生。**

这比“插件化”本身更激进。

### 能力自由意味着能力不再声明“什么时候调用我”

理想情况下，一个 Harness 可以只有：

```text
Capability Registry

filesystem
shell
browser
git
database
search
python
subagent
deployment
CAD
simulation
...
```

每个 Capability 只需要比较完整地声明：

```text
我是谁
我能做什么
输入是什么
输出是什么
有什么约束
```

而不需要知道：

```text
谁会调用我
我前面是什么状态
后面应该是什么状态
我是哪个 workflow 的第几步
```

LLM 则面对整个 capability space：

<div class="math-display">\[
\mathcal A=\{a_1,a_2,\dots,a_n\}
\]</div>

其中 `𝒜` 表示能力空间，`a_i` 表示其中第 `i` 个可调用能力。

根据目标和当前世界状态不断做：

<div class="math-display">\[
a_t
=
\pi_{\mathrm{LLM}}(goal,history,observations,\mathcal A)
\]</div>

其中 `goal` 是目标，`history` 是历史，`observations` 是观察结果，`𝒜` 是当前能力空间，`a_t` 是时刻 `t` 的行动。

于是就出现了一种过去很难真正实现的软件形态：

> **能力负责声明“我能做什么”，而不是声明“什么时候调用我”。**

“什么时候调用我”从 Capability 自己、Controller、FSM 中被抽离出来，交给一个通用语义协调层。

这就是为什么你说 **LLM as System** 是有道理的。

它已经不只是：

```text
Application
    ↓
call LLM()
```

而越来越像：

```text
               Goal
                ↓
        ┌──────────────┐
        │     LLM      │
        │ Coordination │
        │    Layer     │
        └──────┬───────┘
               │
       Capability Space
   ┌────┬────┬────┬────┬────┐
   FS  Git  Shell Web  DB  Agent
   │    │     │    │    │    │
   └────┴─────┴────┴────┴────┘
               │
               ▼
             World
```

过去的 Application Code 很大一部分其实就是那个中间的：

```text
Coordination Layer
```

现在这一层第一次有可能被模型吸收。

### 必须保留的边界

不过最后一个边界一定要保留，否则很容易从一个很漂亮的洞察走到“所有软件都可以扔给 LLM”。

真正成熟的结构可能不是：

<div class="math-display">\[
\text{System}=\text{LLM}+\text{Tools}
\]</div>

而是：

<div class="math-display">\[
\boxed{
\text{System}
=
\text{LLM Semantic Coordination}
+
\text{Deterministic Runtime}
+
\text{Capabilities}
+
\text{Hard Invariants}
}
\]</div>

这里 `LLM Semantic Coordination` 表示开放式语义决策，`Deterministic Runtime` 表示确定性的运行时控制，`Capabilities` 表示具体能力，`Hard Invariants` 表示必须被系统强制维护的不变量。

也就是：

```text
LLM 决定：
“应该做什么”

Runtime 决定：
“能不能做”

Capability 决定：
“具体怎么做”

Hard State 决定：
“哪些事实绝不能靠猜”
```

这样看 DSH 就非常有意思了：**它不像是在传统软件里增加一个 AI 功能，而更像是在探索一种新的软件组织原则——把 application-specific controller 的一部分，从代码变成模型。**

而这也正好解释了为什么你之前一路从 Cordis → Plugin → Component → FSM 都觉得“好像很接近，但总差一点”。真正差的那一点，很可能一直不是模块化机制本身，而是 **谁来维护模块之间那些不断增长的 Edge**。LLM 恰好第一次给了这个问题一个完全不同的答案。

## Q: DSH 中不是给 LLM 调用的插件如何设计？它们又是如何被调用的？

> 不过DSH 似乎插件也有不是给llm 调用的部分 它们的插件的接口是怎么设计的呢 即每一个插件都需要实现哪些部分 然后不是llm部分的插件 它们的调用是怎么处理的呢?

## A: Cordis Plugin 是更底层的 Runtime Composition Unit，LLM Tool 只是其中一种贡献

这里要把“Plugin = 给 LLM 增加 capability”纠正一下：**在 DSH 里，Cordis Plugin 是更底层、更一般的模块化单位；LLM Tool 只是 Plugin 可以贡献的一种东西。**

官方教程把 tool、LLM adapter、file access、agent loop 等都作为插件挂到共享 Context 上。

### 1. DSH 的 Plugin 本身其实非常“薄”

Cordis 并没有要求“每个插件必须实现 `execute()` / `onMessage()` / `tool()`”这样统一的业务接口。

插件入口可以是函数、class，或者带 `apply` 的对象；加载时 `ctx.plugin(...)` 创建一个 Fiber。插件声明 `inject` 表示自己依赖哪些 service，Cordis 等这些 service 可用后再运行插件。依赖的 service 如果发生变化，`ctx.inject()` 对应的插件逻辑可以被卸载并重新运行。

概念上差不多是：

```ts
const MyPlugin = {
    inject: ["foo", "bar"],

    apply(ctx, config) {
        // 初始化
        // 注册 service
        // 注册 event listener
        // 注册 tool
        // 注册 hook
        // ...
    }
}
```

所以 Plugin 真正统一的约定其实只有：

```text
       Cordis Plugin
            │
     ┌──────┴───────┐
     │              │
 dependencies     apply()
   inject           │
                    ▼
             “往 Context 里贡献东西”
```

至于“贡献什么”，完全取决于这个插件。

### 2. DSH 插件可以分成不同类型

一种确实是我们之前讨论的：

```text
Plugin
  ↓
注册 Tool
  ↓
ctx.tools
  ↓
暴露 schema 给 LLM
  ↓
LLM 选择调用
```

这条链路中，**LLM 是 coordinator**。

但还有大量插件根本不是这样。

例如官方 Core 的结构就是：

```text
ctx.sessions
ctx.systemPrompt
ctx.tools
ctx.agents
ctx.agentLoop
```

这些都是普通 Runtime service。`session`、`system-prompt`、`tools`、`agent`、`agent-loop` 分别由不同核心包提供，并通过 `ctx.xxx` 暴露服务。

因此：

```text
Cordis Plugin
     │
     ├── 提供 Service
     ├── 监听 Event
     ├── 注册 Hook
     ├── 提供 Provider
     ├── 注册 Tool ─────→ LLM 可见
     ├── 修改 Prompt
     └── 启动 Runtime 逻辑
```

**只有其中某些分支会最终进入 LLM。**

### 3. 那些“不经过 LLM”的插件，谁调用？

答案恰好和传统软件架构一致：

> **还是普通程序调用。**

比如 Agent Loop 自己就会显式依赖这些服务。一轮流程基本是：

```text
agent-loop
    │
    ├── ctx.sessions
    │      ↓
    │   开启 turn / 写 session
    │
    ├── ctx.systemPrompt
    │      ↓
    │   构造 prompt
    │
    ├── LLM adapter
    │      ↓
    │   请求模型
    │
    ├── ctx.tools
    │      ↓
    │   执行模型请求的 tool
    │
    └── ctx.sessions
           ↓
       写回结果
```

这里：

```ts
ctx.sessions.xxx()
ctx.systemPrompt.xxx()
ctx.tools.xxx()
```

本质仍然是传统程序调用。

LLM 并没有决定：

> “我觉得现在应该调用 SessionService。”

Agent Loop 的源码已经决定必须这么做。

### 4. DSH 的双层结构

```text
                    DSH Runtime
                        │
              ┌─────────┴─────────┐
              │                   │
       Deterministic Layer    Semantic Layer
              │                   │
        普通代码协调              LLM协调
              │                   │
      ┌───────┼───────┐      ┌────┼────┐
      ↓       ↓       ↓      ↓    ↓    ↓
   Session  Prompt   LLM     FS  Shell Search
   Service  Service Adapter  Tool Tool  Tool
```

左边：

```text
谁调用谁？
```

仍然由源码确定。

右边：

```text
下一步用哪个能力？
```

可以交给 LLM。

这正好接上前面讨论的边界。

### 5. Cordis 解决的是实现耦合，而不是语义依赖

比如 Agent Loop 不应该写：

```ts
const session = new JsonlSession(...)
```

而应该面向：

```ts
ctx.sessions
```

于是底下可以是：

```text
             ctx.sessions
                  │
             interface
          ┌───────┼────────┐
          ↓       ↓        ↓
       Memory   JSONL    Future DB
```

Agent Loop 仍然知道：

> “我需要 Session。”

这个**逻辑依赖没有消失**。

但它不知道：

> “Session 到底怎么实现。”

这正是：

<div class="math-display">\[
\text{Plugin/DI}
\Rightarrow
\text{消除 implementation coupling}
\]</div>

但不消除：

<div class="math-display">\[
\text{semantic dependency}
\]</div>

扩展插件可以依赖公开的 `agent` contract，而不是直接依赖具体的 `agent-loop` 实现，因此 loop 可以替换。

### 6. `inject` 解决“能力不存在怎么办”

例如：

```ts
ctx.inject(["foo"], (ctx) => {
    // 只有 foo 存在的时候这里才运行
})
```

它不是让业务代码到处：

```ts
if (ctx.foo) {
   ...
}
```

而是把：

```text
“foo 存不存在？”
```

提升到了 Plugin lifecycle / composition 层。

于是：

```text
没有 Git Plugin
     ↓
ctx.git 不存在
     ↓
依赖 Git 的 Plugin 不启动

安装 Git Plugin
     ↓
ctx.git 出现
     ↓
依赖它的 Plugin 激活

Git Plugin 卸载
     ↓
ctx.git 消失
     ↓
依赖它的 Fiber 一并清理
```

所以它比 Unity 里经常写的：

```csharp
var x = GetComponent<X>();

if (x != null)
    ...
```

更进一步。

**“能力是否存在”的处理被 composition framework 吸收了。**

### 7. 为什么 DSH 的 Plugin 看起来特别“万能”？

因为它不是：

```text
Plugin = LLM Tool
```

而是：

<div class="math-display">\[
\boxed{
\text{Plugin}
=
\text{Runtime Composition Unit}
}
\]</div>

Tool 只是：

<div class="math-display">\[
\text{Plugin}
\rightarrow
\text{ToolRegistry}
\]</div>

的一种情况。

可以把整个体系理解成：

```text
                     Cordis
                       │
                Plugin / Fiber
                       │
          ┌────────────┼────────────┐
          ↓            ↓            ↓
       Service       Events       Registry
          │            │            │
          │            │       ┌────┴─────┐
          │            │       ↓          ↓
          │            │     Tool      Provider
          │            │       │
          │            │       ▼
          │            │      LLM
          │            │
          └──────┬─────┘
                 ↓
           普通 Runtime Code
```

所以“LLM 让能力能够自由组合”只是 **DSH Plugin architecture 的上半层效果**。

它下面其实还有一个非常传统、而且非常严谨的软件架构：

> **Cordis 负责模块的生命周期、依赖、注册和发现；普通 TypeScript 负责确定性的 Runtime coordination；LLM 只接管那些适合语义决策的 coordination。**

这也是 DSH 设计得漂亮的地方：**它没有因为有 LLM 就把所有东西都做成 Tool。** Session、Agent lifecycle、Prompt assembly、Provider、Registry 这些确定性基础设施仍然保持传统软件结构；到了真正“下一步应该干什么”的边界，才把控制权交给 LLM。

## Q: 这种 Runtime 框架是否也解决了依赖问题？Python 或 Unity 中有没有类似框架？

> 我懂了 这似乎是一个维护的非常好的 runtime 框架 (我感觉这个架构非常的先进) 我们之前讨论的只是它特别适合llm 的一面 另一部分 他也特别好的解决了依赖的问题 调用逻辑需要依赖 但是这个依赖被框架解决了 (也就是说代码里面不需要显式判断这个依赖是否存在了 这变成了runtime的一部分 感觉确实是为了动态runtime创建的框架 并且很好的解决了这些问题 类似的 python 或者 unity 有这样的框架吗?

## A: Cordis/DSH 不只是 DI，而更接近 Dynamic Runtime Composition Framework

对，你现在对 Cordis/DSH 的理解已经可以进一步概括成：**它不只是 DI，而更接近“动态 Runtime Composition Framework”。** DI 只是其中最重要的机制之一。

你说的这句话尤其关键：

> 调用逻辑需要依赖，但是“这个依赖当前是否存在、什么时候存在、什么时候失效”不应该由业务代码反复处理，而应该成为 Runtime 的责任。

例如业务代码理想上只声明：

```text
Plugin A requires:
    FileSystem
    Git
```

Runtime 负责：

```text
FileSystem 有了吗？
Git 有了吗？
→ 都有：activate A

Git 被卸载了？
→ deactivate A

Git 又被装进来了？
→ reactivate A
```

这比单纯的 constructor DI 又向前走了一步，因为它把**依赖生命周期**也纳入 Runtime。

### 1. Unity：VContainer / Zenject 这一脉非常接近

Unity 里比较接近这种思路的是 VContainer。它提供 constructor/method/property injection、生命周期 Scope、嵌套 LifetimeScope、EntryPoint 等机制。也就是说你可以声明“这个对象需要哪些服务”，Container 负责把整个 object graph 拼起来，而不是到处 `GetComponent<T>()` + 判空。

例如思想上：

```csharp
class EnemyAI
{
    public EnemyAI(
        IPathFinder pathFinder,
        IWeaponSystem weaponSystem)
    {
        ...
    }
}
```

Composition Root：

```csharp
builder.Register<PathFinder>(Lifetime.Singleton)
       .As<IPathFinder>();

builder.Register<WeaponSystem>(Lifetime.Singleton)
       .As<IWeaponSystem>();

builder.Register<EnemyAI>(Lifetime.Transient);
```

于是：

```text
EnemyAI
   │ requires
   ├── IPathFinder
   └── IWeaponSystem

          ↓

      VContainer

      自动 resolve

          ↓

EnemyAI
   ├── PathFinder
   └── WeaponSystem
```

你之前 Unity 里头疼的：

```csharp
var x = GetComponent<X>();

if (x != null)
   ...
```

相当大一部分可以消掉。

更经典的是 Zenject / Extenject。这一套甚至明确支持 optional dependency、runtime factory、nested container、signals、scene-level composition 等，所以和我们前面讨论的“动态组合”更接近。

但需要注意：VContainer、Zenject/Extenject 主要解决的是对象图构造、依赖注入和生命周期管理；它们并不会自动替代业务层的协调逻辑。`EnemyAI` 什么时候使用 `IWeaponSystem`，仍然需要由 AI、状态机、行为树或其他协调层决定。

### 2. Python：有 DI，但生态哲学不太一样

Python 里一个比较完整的是 Dependency Injector。它有：

```text
Provider
Container
Factory
Singleton
Resource
Dependency
Selector
Wiring
Override
```

甚至支持动态 Container、异步 dependency、resource lifecycle 和 provider override。

例如：

```python
class Service:
    def __init__(self, database: Database):
        self.database = database
```

然后：

```python
class Container(containers.DeclarativeContainer):
    database = providers.Singleton(Database)

    service = providers.Factory(
        Service,
        database=database,
    )
```

业务代码不需要：

```python
if database_exists:
    ...
```

Container 负责构造 object graph。

如果只需要一个非常轻量的 IoC Container，Punq 更简单：显式创建 Container，注册 service → implementation，然后 resolve。它刻意避免 global state 和复杂 decorator。

### 3. Cordis 和普通 DI 框架的区别

普通 DI 更多是：

```text
        Application startup

               ↓

       Build dependency graph

               ↓

      ┌────────Container────────┐
      │                         │
      A → B → C                 │
      D → C                     │
      E → B                     │
      └─────────────────────────┘

               ↓

          Run application
```

它隐含的世界观通常是：

> **先把 Application 组装好，然后运行。**

而 Cordis/DSH 给你的感觉明显更像：

```text
                     Runtime

 Plugin A ──install──→ │
                       │
 Plugin B ──install──→ │
                       │
 Plugin C ←─remove──── │
                       │
 Service X appears ──→ │ dependency satisfied
                       │        ↓
                       │ activate dependent fiber
                       │
 Service X disappears →│
                       │        ↓
                       │ dispose dependent fiber
```

也就是说：

<div class="math-display">\[
\text{Composition}
\]</div>

本身就是 Runtime 的持续过程，而不仅仅是 application boot 时的一次性操作。

这就是为什么你会觉得它特别适合 DSH。

### 4. 从 Unity Component 到 Semantic Runtime Composition

其实可以把现在讨论出来的几个概念放在一条连续线上：

```text
Unity Component
      │
      │ “对象由什么组成？”
      ▼
Component Composition

      ↓

Dependency Injection
      │
      │ “依赖的具体实现是谁？”
      ▼
Dependency Resolution

      ↓

IoC Container
      │
      │ “谁负责创建和连接这些对象？”
      ▼
Runtime Assembly

      ↓

Cordis-like Plugin Runtime
      │
      │ “能力可以动态出现/消失，
      │  Runtime 持续维护依赖和生命周期”
      ▼
Dynamic Runtime Composition

      ↓

LLM Harness
      │
      │ “不仅能力可以动态组合，
      │  连能力之间部分 coordination
      │  都可以运行时决定”
      ▼
Semantic Runtime Composition
```

所以你说它“先进”，具体先进的地方不只是用了插件，而是它把几个以前经常散落在业务代码里的 concern：

<div class="math-display">\[
\boxed{
\text{Discovery}
+
\text{Dependency}
+
\text{Lifecycle}
+
\text{Composition}
+
\text{Disposal}
}
\]</div>

统一提升到了 Runtime 层。

然后 DSH 又在这个基础上加了：

<div class="math-display">\[
\boxed{
\text{LLM Semantic Coordination}
}
\]</div>

这两件事结合起来才特别有意思：

> **Cordis 解决“这些能力怎样安全地存在于系统里”；LLM 解决“这些能力什么时候应该被使用”。**

这也解释了为什么 Cordis 和 LLM Harness 有一种异常好的契合感：它并不一定是“专门为了 LLM 才成立”的架构，但 LLM 恰好把这种动态 Runtime 原本最后还缺失的那块——**动态 coordination**——补上了。

### 5. 资料与适用范围说明

本问中的 DSH/Cordis 部分以 [DeepSeek Harness 官方介绍](https://www.deepseek.com/harness/)、[Cordis 教程](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-tutorial/index.md) 和 [Cordis 生命周期文档](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/framework/index.zh.md) 为依据。

Unity 方向可进一步参考 [VContainer](https://github.com/hadashiA/VContainer) 和 [Zenject/Extenject](https://github.com/Mathijs-Bakker/Extenject)；Python 方向可进一步参考 [Dependency Injector](https://python-dependency-injector.ets-labs.org/) 和 [Punq](https://punq.readthedocs.io/en/latest/)。这些框架主要覆盖 DI、对象图构造和生命周期，并不自动消除业务协调问题。
