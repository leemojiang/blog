---
title: "LLM 真的在做强化学习吗？还是一种加权 MLE？"
date: 2026-06-16
draft: false
math: true
tags:
  - 强化学习
  - 大模型
  - RLHF
  - DPO
  - PPO
  - LLM
  - 机器学习
description: "从 weighted MLE、KL 正则和 exponential tilting 的分布视角，理解 RLHF、PPO、GRPO 与 DPO 到底在把语言模型推向什么分布。"
zhihu_titles:
  - "LLM 真的在做强化学习吗？还是一种加权 MLE？"
  - "换个角度理解 RLHF、PPO、GRPO 和 DPO：它们都在诱导分布"
  - "从加权 MLE 到指数 Tilting：大模型 RL 的另一种理解方式"
---
> 这篇文章是我正在整理的开源项目 [leemojiang/llm-notes-all-in-one](https://github.com/leemojiang/llm-notes-all-in-one) 中 RL for LLM 系列的一篇。项目会持续整理 LLM、强化学习、Agent 和从零实现大模型相关的学习笔记。如果这份笔记对你有帮助，也欢迎到 GitHub 点一个 Star 支持一下。

在前面几节里，我们已经从 RL 的基本定义一路讲到了 LLM 中的 Policy Gradient、PPO、GRPO、DPO 和 RLHF。回头看这些算法，会发现 LLM 中的强化学习和传统环境里的强化学习有一个很大的差异：很多时候，reward 不是每个 state-action 都即时给出，而是对整条回答、整条推理轨迹或最终结果给出一个标量分数。

这会带来一个很自然的问题：

> LLM 的 RL 真的还是在做传统意义上的强化学习吗？还是说它更像是在根据 reward / preference / verifier score，对模型分布做一种带权重的最大似然估计？

> 那么很自然的追问是,这种由权重/标量诱导出来的分布和权重的关系是什么样子的?我们什么方法可以分析它,又能得到什么性质?

这一节尝试换一个视角：暂时不从 state、action、Bellman equation 出发，而是从“分布如何被标量函数诱导出来”出发。这个视角可以把 RLHF、PPO、GRPO、DPO、RLVR 等方法放到同一个图景里：

```text
reference / sampling distribution
        +
scalar feedback
        ↓
reward-induced target distribution
        ↓
fit policy model toward that distribution
```

更短地说：

> 标量/reward既可以理解成RL中的给样本打分，也可以理解成在诱导一个新的目标分布。

## Q: LLM 中常见的 RL 目标函数长什么样？

先把几个常见目标放在一起看。设 prompt 来自数据分布：



\[x\sim \mathcal{D}\]



给定 prompt $x$，模型生成完整回答：



\[y=(y_1,y_2,\ldots,y_T)\]



语言模型策略为：



\[\pi_\theta(y\mid x) = \prod_{t=1}^{T} \pi_\theta(y_t\mid x,y_{<t})\]



### Policy Gradient：从期望奖励到 weighted log-likelihood gradient

最朴素的 LLM policy optimization 目标是最大化完整回答的期望奖励：



\[J(\theta) = \mathbb{E}_{x\sim\mathcal{D},\,y\sim\pi_\theta(\cdot\mid x)} \left[ R(x,y) \right]\]



这里 $R(x,y)$ 是完整回答级别的标量奖励。它可以来自 reward model、规则奖励、verifier 或人工偏好打分。注意，这个目标本身还不是 MLE；它是一个期望奖励目标。

运用 log-derivative trick，目标的梯度会变成我们常见的 Policy Gradient 形式：



\[\nabla_\theta J(\theta) = \mathbb{E}_{x,y} \left[ R(x,y) \nabla_\theta \log \pi_\theta(y\mid x) \right]\]



代入自回归分解：



\[\nabla_\theta J(\theta) = \mathbb{E}_{x,y} \left[ R(x,y) \sum_{t=1}^{T} \nabla_\theta \log \pi_\theta(y_t\mid x,y_{<t}) \right]\]



如果使用 advantage，则更常见的写法是：



\[\nabla_\theta J(\theta) = \mathbb{E}_{x,y} \left[ \sum_{t=1}^{T} \hat{A}_t \nabla_\theta \log \pi_\theta(y_t\mid x,y_{<t}) \right]\]



这里“像 weighted MLE”要说得更精确一点。MLE 是目标函数，而上面的式子是梯度。真正相似的是：Policy Gradient 的梯度形式，和 weighted MLE 的梯度形式一致。

假设有一个固定采样分布 $q(z)$，weighted MLE 目标是：



\[\mathcal{L}_{\mathrm{wMLE}}(\theta) = \mathbb{E}_{z\sim q} \left[ w(z) \log p_\theta(z) \right]\]



那么它的梯度是：



\[\nabla_\theta \mathcal{L}_{\mathrm{wMLE}}(\theta) = \mathbb{E}_{z\sim q} \left[ w(z) \nabla_\theta \log p_\theta(z) \right]\]



对比 LLM Policy Gradient：



\[\nabla_\theta J(\theta) = \mathbb{E}_{x,y} \left[ R(x,y) \nabla_\theta \log \pi_\theta(y\mid x) \right]\]



可以看到，$\log\pi_\theta(y\mid x)$ 扮演 log-likelihood 的角色，$R(x,y)$ 或 $\hat{A}_t$ 扮演样本权重的角色。因此更准确的说法是：

> Policy Gradient 的梯度可以看成 reward / advantage 加权的 log-likelihood 梯度。

如果进一步把 reward 暂时看成非负权重，那么它还可以诱导出一个重加权后的目标分布。但这里要注意：在没有 KL reference 的普通 weighted fitting 里，目标分布不是只由权重决定的，而是由“原始采样分布”和“权重函数”共同决定的。后文会把这一点写成统一形式：

\[\tilde{q}(z)\propto q(z)w(z)\]

但它还不能直接等同于普通 weighted MLE，因为 reward / advantage 可能为负，而且采样分布通常来自当前策略或旧策略，不是一个固定数据集。

### RLHF with KL：奖励不是单独起作用，而是和 reference model 一起起作用

RLHF 通常还会加 KL 约束。固定一个 prompt $x$，常见分布级目标可以写成：



\[\max_{\pi(\cdot\mid x)} \mathbb{E}_{y\sim\pi(\cdot\mid x)} \left[ r_\phi(x,y) \right] - \tau \operatorname{KL} \left( \pi(\cdot\mid x) \Vert \pi_{\mathrm{ref}}(\cdot\mid x) \right)\]



其中 $r_\phi(x,y)$ 是 reward model 给出的完整回答级别奖励，$\pi_{\mathrm{ref}}$ 是参考模型，$\tau>0$ 是 KL 惩罚强度。这里的 scalar signal 是 $r_\phi(x,y)$，但它不是单独决定目标分布；它会和 reference distribution $\pi_{\mathrm{ref}}(y\mid x)$ 共同决定分布移动方向。

### PPO：用 advantage 加权 token log-prob，并限制策略移动幅度

PPO 并不直接求这个分布级目标的闭式解，而是使用旧策略采样的数据、advantage 和 ratio clipping 来做稳定的局部更新：



\[\rho_t(\theta) = \frac{ \pi_\theta(y_t\mid x,y_{<t}) }{ \pi_{\theta_{\mathrm{old}}}(y_t\mid x,y_{<t}) }\]





\[L^{\mathrm{CLIP}}(\theta) = \mathbb{E}_t \left[ \min \left( \rho_t(\theta)\hat{A}_t, \operatorname{clip} \left( \rho_t(\theta), 1-\epsilon, 1+\epsilon \right) \hat{A}_t \right) \right]\]



这里的 scalar signal 是 $\hat{A}_t$。如果 $\hat{A}_t>0$，PPO 希望提高这个 token 的概率；如果 $\hat{A}_t<0$，PPO 希望降低这个 token 的概率。ratio clipping 则限制这种概率移动不要太激进。

### GRPO：目标和 PPO 很接近，主要差别在 advantage 的估计方式

GRPO 进一步把 advantage 的来源换成组内相对奖励。对同一个 prompt $x$，采样 $G$ 个回答：



\[y^{(1)},y^{(2)},\ldots,y^{(G)}\]



如果对应奖励为 $R^{(1)},\ldots,R^{(G)}$，则可以用组内标准化分数构造 advantage：



\[\hat{A}^{(i)} = \frac{ R^{(i)}-\bar{R} }{ \sigma_R+\varepsilon }\]



也就是说，GRPO 的 policy optimization 形式和 PPO 非常接近，仍然是用某种 advantage-like scalar 去加权 log-prob / ratio objective。核心区别在于：PPO 通常用 value model / GAE 来估计 $\hat{A}_t$，而 GRPO 用同一个 prompt 下多个回答的组内相对分数来估计 advantage，从而减少甚至避免单独训练 value model。

### DPO：用 preference pair 隐式表示 reward difference

DPO 的表面形式更像分类损失。给定偏好数据：



\[(x,y_w,y_l)\]



DPO loss 可以写成：



\[\mathcal{L}_{\mathrm{DPO}}(\theta) = - \mathbb{E}_{(x,y_w,y_l)} \left[ \log\sigma \left( \beta \left[ \log \frac{ \pi_\theta(y_w\mid x) }{ \pi_{\mathrm{ref}}(y_w\mid x) } - \log \frac{ \pi_\theta(y_l\mid x) }{ \pi_{\mathrm{ref}}(y_l\mid x) } \right] \right) \right]\]



它看起来不像 $\mathbb{E}[R]$，但它背后使用了 KL-regularized RL 的闭式解，把 reward difference 隐式改写成 policy 相对 reference model 的 log-ratio。后面会专门解释这一点。

所以从最粗粒度看，这些方法都围绕同一件事：

> 用一个标量函数 $R$、$r_\phi$、$\hat{A}$、preference score 或 verifier score，改变模型应该提高哪些样本的概率。

## Q: 为什么这像 weighted MLE？

先不谈 RL，只看普通最大似然估计。设数据样本为：



\[z_1,z_2,\ldots,z_N\]



普通 MLE 是：



\[\max_\theta \sum_{i=1}^{N} \log p_\theta(z_i)\]



如果每个样本都有非负权重 $w_i\ge 0$，weighted MLE 是：



\[\max_\theta \sum_{i=1}^{N} w_i \log p_\theta(z_i)\]



可以把普通 MLE 看成在经验分布上做拟合。定义经验分布：



\[\hat{q}(z_i) = \frac{1}{N}\]



如果给每个样本加上权重 $w_i$，那么诱导出来的重加权经验分布不是只有 $w_i$，而是：



\[\tilde{p}(z_i) = \frac{ \hat{q}(z_i)w_i }{ \sum_{j=1}^{N}\hat{q}(z_j)w_j }\]



由于这里 $\hat{q}(z_i)=1/N$，所以上式也可以简化成：



\[\tilde{p}(z_i) = \frac{ w_i }{ \sum_{j=1}^{N}w_j }\]



由于 $\sum_j w_j$ 和 $\theta$ 无关，上面的目标等价于：



\[\max_\theta \mathbb{E}_{z\sim\tilde{p}} \left[ \log p_\theta(z) \right]\]



也就是说：

> weighted MLE = 在原始经验分布 $\hat{q}$ 被权重 $w$ 重加权后得到的分布 $\tilde{p}$ 上做普通 MLE。

如果模型族足够强，并且优化达到全局最优，那么 $p_\theta$ 会拟合这个重加权分布；如果模型族有限，那么它会找到模型族里最接近 $\tilde{p}$ 的分布。因为：



\[\arg\max_\theta \mathbb{E}_{z\sim\tilde{p}} \left[ \log p_\theta(z) \right] = \arg\min_\theta \operatorname{KL} \left( \tilde{p} \Vert p_\theta \right)\]



连续或大规模离散情形也是一样。给定一个采样分布 $q(z)$ 和非负权重函数 $w(z)$，重加权后的目标分布是：



\[\tilde{q}(z) = \frac{ q(z)w(z) }{ \int q(z')w(z')dz' }\]



这就是没有显式 KL reference 时最重要的形式：

> 无 KL 的 weighted fitting 中，诱导分布依赖于采样分布 $q(z)$ 和权重函数 $w(z)$ 的乘积，而不是只依赖权重。

把 $z$ 换成 LLM 的回答 $(x,y)$ 或 trajectory $\tau$，weighted MLE 的直觉就变成：

> reward / advantage 越高的样本，在训练中应该被赋予更大的 log-likelihood 权重。

这就是 RL 像 weighted SFT / weighted MLE 的原因。

不过这里必须保留两个修正。

第一，普通 weighted MLE 要求权重非负，但 reward 或 advantage 可以为负。负 advantage 的含义不是“少学习一点”，而是明确降低某个 token 或回答的概率。

第二，RL 中的数据分布常常来自当前策略或旧策略，而不是一个固定数据集。比如 PPO 先用 $\pi_{\theta_{\mathrm{old}}}$ 采样，再用 ratio 修正；这比普通监督学习更复杂。

所以更严谨的说法是：

> Policy Gradient 给出了 weighted log-likelihood gradient 的直觉；要得到严格的目标分布，通常需要引入非负权重、归一化，或者 KL 正则。

### 权重分布如何采样?
这里有一个容易忽略的点：权重分布能不能采样，取决于我们有没有 base distribution，以及能不能从这个 base distribution 采样。权重本身不能凭空定义分布，它必须作用在某个已有的支持集或采样分布上。

在有限数据集里，base distribution 就是经验分布。可以把每个样本 $z_i$ 看成原本有相同质量 $1/N$，weighted MLE 做的是把这些质量按 $w_i$ 重新分配。如果要从这个诱导分布采样，就是按下面的概率采样样本 index：



\[P(i) = \frac{ w_i }{ \sum_j w_j }\]



如果所有 $w_i$ 都相同，那么诱导分布就退化回原始经验分布，没有任何偏好方向。

如果 $w$ 完全不依赖样本 $z$，也就是 $w(z)=c$，那么连续情形下：



\[\tilde{q}(z) = \frac{ q(z)c }{ \int q(z')c\,dz' } = q(z)\]



所以并不是“无法采样”，而是“采样分布没有被改变”：仍然从原来的 $q(z)$ 采样。真正的问题是，如果没有经验数据集、没有 $q(z)$、也没有 $p_0(z)$ 这样的 base distribution，单独给一个权重函数 $w(z)$ 并不能定义概率分布。必须先有“在哪些 $z$ 上分配概率”的基准。

## Q: KL 正则为什么会导出指数 tilting？

weighted MLE 告诉我们：权重会改变经验分布。但如果我们想问得更精确：

> 一个 reward 标量到底诱导出什么目标分布？

最干净的答案来自 KL-regularized optimization。

先不引入 LLM。设 $z$ 是任意随机对象，可以是样本、回答或轨迹。给定：

- $p_0(z)$：基准分布；
- $r(z)$：标量 reward；
- $p(z)$：我们要优化的新分布；
- $\tau>0$：KL 惩罚强度。

考虑分布级优化问题：



\[\max_p \mathbb{E}_{z\sim p} \left[ r(z) \right] - \tau \operatorname{KL} \left( p \Vert p_0 \right)\]



离散情形下展开为：



\[\max_p \sum_z p(z)r(z) - \tau \sum_z p(z) \log \frac{p(z)}{p_0(z)}\]



约束是：



\[\sum_z p(z)=1\]



构造拉格朗日函数：



\[\mathcal{L}(p,\alpha) = \sum_z p(z)r(z) - \tau \sum_z p(z) \log \frac{p(z)}{p_0(z)} + \alpha \left( \sum_z p(z)-1 \right)\]



对每个 $p(z)$ 求偏导。用到：



\[\frac{\partial}{\partial u} \left[ u\log\frac{u}{a} \right] = \log\frac{u}{a} + 1\]



所以：



\[\frac{\partial\mathcal{L}}{\partial p(z)} = r(z) - \tau \left[ \log \frac{p(z)}{p_0(z)} + 1 \right] + \alpha\]



令偏导为 0，整理得到：



\[\log \frac{p(z)}{p_0(z)} = \frac{1}{\tau}r(z) + \frac{\alpha-\tau}{\tau}\]



右边第二项不依赖 $z$，只负责归一化。因此：



\[p^*(z) \propto p_0(z) \exp \left( \frac{1}{\tau}r(z) \right)\]



写成归一化形式：



\[p^*(z) = \frac{ p_0(z) \exp \left( \frac{1}{\tau}r(z) \right) }{ Z }\]



其中：



\[Z = \sum_{z'} p_0(z') \exp \left( \frac{1}{\tau}r(z') \right)\]



如果定义 $\beta=1/\tau$，则：



\[p^*(z) = \frac{ p_0(z)\exp(\beta r(z)) }{ Z }\]



这就是 exponential tilting。

推广到 LLM，固定 prompt $x$，把 $z$ 换成回答 $y$，把 $p_0$ 换成参考模型：



\[p_0(z) \longrightarrow \pi_{\mathrm{ref}}(y\mid x)\]



于是：



\[\pi^*(y\mid x) = \frac{ \pi_{\mathrm{ref}}(y\mid x) \exp \left( \beta r(x,y) \right) }{ Z(x) }\]



其中：



\[Z(x) = \sum_{y'} \pi_{\mathrm{ref}}(y'\mid x) \exp \left( \beta r(x,y') \right)\]



这句话是本章最重要的结论：

> KL-regularized RLHF 的理想目标分布，就是在每个 prompt 下，对参考模型按 reward 做指数重加权。

也就是说，RL 不只是“把高 reward 样本概率调大”这么口语化。KL 正则给出了明确的目标分布：



\[\pi_{\mathrm{target}}(y\mid x) \propto \pi_{\mathrm{ref}}(y\mid x) \exp \left( \beta r(x,y) \right)\]



## Q: 指数 tilting 是什么？

一般地，给定基准分布 $p_0(z)$、标量函数 $f(z)$ 和参数 $\lambda$，定义：



\[p_\lambda(z) = \frac{ p_0(z)\exp(\lambda f(z)) }{ Z(\lambda) }\]



其中：



\[Z(\lambda) = \sum_z p_0(z)\exp(\lambda f(z))\]



连续情形下把求和换成积分。只要 $Z(\lambda)<\infty$，这就是一个合法概率分布。

这个构造有很多名字：

- 在统计物理里，它像 Gibbs / Boltzmann 分布。
- 在统计学里，它是以 $p_0$ 为 base measure 的指数族。
- 在大偏差理论里，它常叫 exponential tilting。
- 在金融和精算里，相关构造也叫 Esscher transform。
- 在能量模型里，如果 $E(z)=-f(z)$，则 $p_\lambda(z)\propto p_0(z)\exp(-\lambda E(z))$。

写成指数族形式：



\[p_\lambda(z) = \exp \left( \lambda f(z) - A(\lambda) \right) p_0(z)\]



其中：



\[A(\lambda) = \log Z(\lambda)\]



这里 $A(\lambda)$ 是 log-partition function。它不仅负责归一化，也描述了 tilting 之后分布的性质。在可以交换求导和求和的条件下：



\[\frac{d}{d\lambda} A(\lambda) = \mathbb{E}_{z\sim p_\lambda} \left[ f(z) \right]\]



二阶导数是：



\[\frac{d^2}{d\lambda^2} A(\lambda) = \operatorname{Var}_{z\sim p_\lambda} \left[ f(z) \right]\]



因此：

- $\lambda=0$ 时，$p_\lambda=p_0$。
- $\lambda$ 越大，分布越偏向 $f(z)$ 高的区域。
- $A'(\lambda)$ 是新分布下的平均 score。
- $A''(\lambda)$ 是新分布下 score 的方差。
- $A(\lambda)$ 是凸函数。

所以“标量函数诱导分布”这句话，数学上就是：

> $f(z)$ 通过 $\exp(\lambda f(z))$ 改变 $p_0(z)$ 的概率质量，形成一条指数族路径 $p_\lambda$。

## Q: 如何用这个视角统一 RLHF、PPO、GRPO、DPO？

现在可以把几类方法放进同一个框架。

第一类是没有显式 KL 的 reward-weighted fitting。假设样本来自某个采样分布 $q(y\mid x)$，并且有非负 reward $r(x,y)$。如果做：



\[\max_\theta \mathbb{E}_{y\sim q(\cdot\mid x)} \left[ r(x,y) \log \pi_\theta(y\mid x) \right]\]



那么它等价于在下面这个重加权分布上做 MLE：



\[\tilde{q}(y\mid x) = \frac{ q(y\mid x)r(x,y) }{ \sum_{y'}q(y'\mid x)r(x,y') }\]



这就是 weighted MLE / weighted SFT 视角。

第二类是有 KL reference 的 RLHF。它的理想目标分布是：



\[\pi^*(y\mid x) \propto \pi_{\mathrm{ref}}(y\mid x) \exp \left( \beta r_\phi(x,y) \right)\]



这里最容易让人困惑的地方是：无 KL 的式子里有采样分布 $q(y\mid x)$，但 KL 正则的闭式解里看起来只剩下 $\pi_{\mathrm{ref}}(y\mid x)$。这不是因为采样分布真的不重要了，而是因为这两个式子回答的问题不同。

- 无 KL 的 weighted fitting 回答的是：给定来自 $q$ 的样本，按 $r$ 重加权以后，经验目标分布是什么？
- 有 KL 的分布级优化回答的是：如果可以直接在所有策略分布里优化，并以 $\pi_{\mathrm{ref}}$ 作为 reference，那么理论最优分布是什么？

所以在数学闭式解里，$\pi_{\mathrm{ref}}$ 取代了 $q$ 成为定义 target distribution 的 base distribution；但在实际训练里，我们仍然需要某个 sampling distribution 来产生样本、估计 reward 和更新梯度。

> 实际训练的时候,样本是如何来的?

这里的 $\pi_{\mathrm{ref}}$ 是 reference / prior distribution，它参与定义理论上的 tilted target distribution。但这不等于实际训练时的样本一定从 $\pi_{\mathrm{ref}}$ 采。

在 RLHF / PPO 中，实际 rollout 通常来自当前策略或旧策略：



\[y \sim \pi_{\theta_{\mathrm{old}}}(\cdot\mid x)\]



然后用 reward model、KL penalty、value model / advantage 和 PPO ratio 来更新 $\pi_\theta$。因此要区分两件事：

- 诱导分布：理论上由 $\pi_{\mathrm{ref}}(y\mid x)\exp(\beta r(x,y))$ 定义的目标分布。
- 采样分布：实际 rollout 或训练数据来自哪里，例如 $\pi_{\theta_{\mathrm{old}}}$、当前策略、replay buffer，或者离线偏好数据集。

PPO 可以理解为用采样、advantage、ratio clipping 和 KL penalty，局部近似地把策略往这个高 reward、受 reference 约束的方向推。

GRPO / RLVR 则把标量函数换成组内相对奖励或 verifier reward。更准确地说，GRPO 的优化目标和 PPO-style policy optimization 非常接近，仍然是在用 advantage-like scalar 加权策略更新；它的关键变化不是“换了一个完全不同的目标”，而是不用 value model / GAE 来估计 advantage，改用同一个 prompt 下多个回答的组间相对分数来估计 advantage。

DPO 则利用 KL 正则目标的闭式解。由：



\[\pi^*(y\mid x) = \frac{ \pi_{\mathrm{ref}}(y\mid x) \exp(\beta r(x,y)) }{ Z(x) }\]



两边取 log：



\[\log \pi^*(y\mid x) = \log \pi_{\mathrm{ref}}(y\mid x) + \beta r(x,y) - \log Z(x)\]



整理得：



\[r(x,y) = \frac{1}{\beta} \left[ \log \frac{ \pi^*(y\mid x) }{ \pi_{\mathrm{ref}}(y\mid x) } + \log Z(x) \right]\]



对于同一个 prompt 下的两个回答 $y_w$ 和 $y_l$，$\log Z(x)$ 会抵消：



\[r(x,y_w)-r(x,y_l) = \frac{1}{\beta} \left[ \log \frac{ \pi^*(y_w\mid x) }{ \pi_{\mathrm{ref}}(y_w\mid x) } - \log \frac{ \pi^*(y_l\mid x) }{ \pi_{\mathrm{ref}}(y_l\mid x) } \right]\]



DPO 用 $\pi_\theta$ 近似 $\pi^*$，于是可以直接用 policy log-ratio 写 preference loss，而不显式训练 reward model，也不需要计算 $Z(x)$。

可以粗略总结成：

| 方法 | 标量信号 | 是否显式有 reference / KL | 分布视角 |
|---|---|---|---|
| Weighted SFT / weighted MLE | 样本权重 | 不一定 | 拟合重加权经验分布 |
| Policy Gradient | reward / advantage | 不一定 | 用标量权重调整 log-prob 梯度 |
| RLHF with PPO | reward model + advantage | 通常有 | 近似优化 KL-regularized tilted target |
| GRPO / RLVR | 组内相对奖励 / verifier | 通常有 | PPO-style 目标基本保留，主要改变 advantage 估计方式 |
| DPO | preference pair | 有 reference | 用 policy log-ratio 隐式表示 reward difference |

所以，不从 RL 术语出发，也可以这样理解：

> 这些算法都在用 scalar feedback 改变模型分布。区别在于：这个 scalar 是 reward、advantage、group-relative score 还是 preference；有没有 reference distribution；以及算法是显式估计目标分布，还是用采样梯度、ratio clipping 或 pairwise loss 近似它。

## Q: 采样分布会如何影响我们学到诱导分布？

上面的 KL 形式非常漂亮：



\[p^*(z) \propto p_{\mathrm{ref}}(z) \exp \left( \frac{r(z)}{\tau} \right)\]



它看起来只和 reference distribution $p_{\mathrm{ref}}$、reward $r(z)$ 以及温度系数 $\tau$ 有关，采样分布似乎消失了。但这其实有点 too good to be true。

更准确地说，KL 闭式解刻画的是：

> 理论诱导分布：我们想学到什么。

而实际训练还必须面对另一个问题：

> 实际采样分布：我们看到了什么。

假设真实训练时样本来自某个采样分布：



\[z\sim \mu(z)\]



如果我们想用这些样本估计目标分布 $p^* $ 下的期望，那么可以写成 importance sampling 的形式：



\[\mathbb{E}_{z\sim p^*} \left[ f(z) \right] = \mathbb{E}_{z\sim \mu} \left[ \frac{p^*(z)}{\mu(z)} f(z) \right]\]



其中 importance weight 为：



\[\frac{p^*(z)}{\mu(z)} \propto \frac{ p_{\mathrm{ref}}(z) \exp(r(z)/\tau) }{ \mu(z) }\]



这个比值就是采样效率的核心。如果 $ \mu(z)$ 和 $ p^* (z) $ 很接近，那么权重比较平滑，估计方差小，训练更高效。如果 $ \mu(z) $ 和 $ p^* (z) $ 差得很远，比如高 reward 区域几乎采不到，那么 importance weight 会非常稀疏，估计方差巨大。理论 target 再漂亮，实际算法也很难学到它。

所以可以这样理解：

> KL 闭式解决定 target distribution；sampling distribution 决定我们估计这个 target 的方差、coverage 和可学习性。

采样分布好不好，可以用一些量来刻画。比如：



\[\operatorname{KL} \left( p^* \Vert \mu \right)\]



如果这个 KL 很大，说明目标分布认为重要的区域，采样分布给的概率太低。

也可以看 $\chi^2$ divergence：



\[\chi^2 \left( p^* \Vert \mu \right) = \mathbb{E}_{z\sim \mu} \left[ \left( \frac{p^*(z)}{\mu(z)} - 1 \right)^2 \right]\]



这个量和 importance sampling 的方差直接相关。越大，说明权重越稀疏，估计越不稳定。

在有限样本里，也可以看 effective sample size：



\[\mathrm{ESS} = \frac{ \left( \sum_i w_i \right)^2 }{ \sum_i w_i^2 }\]



权重越集中，ESS 越小。也就是说，看似有很多样本，真正对估计有贡献的样本却很少。

因此，更高效采样的本质是让 $\mu$ 更接近 $p^*$，但又不能太激进。RLHF / PPO 里用 old policy rollout，其实就是一种折中：



\[\mu = \pi_{\theta_{\mathrm{old}}}\]



然后通过概率比值：



\[\frac{ \pi_\theta(z) }{ \pi_{\theta_{\mathrm{old}}}(z) }\]



以及 clipping / KL penalty 控制新策略不要离采样分布太远。否则 off-policy 误差和方差都会迅速变大。

这也解释了很多 LLM post-training 方法为什么要采用类似的设计：

- on-policy / near-policy sampling：让采样分布接近当前要优化的策略，降低分布错配。
- KL to reference：避免目标分布离原模型太远，减少 reward 诱导出的极端分布。
- rejection sampling / best-of-N：从当前模型采多个样本，用 reward 选择更靠近 target 的样本。
- DPO：绕开显式从 $p^*$ 采样，直接用 preference pair 学 log-ratio。
- GRPO：对同一 prompt 采多个回答，用组内比较构造更稳定的相对 advantage。

最关键的一句话是：

> 理论上，KL 正则给出 $p^* (z)\propto p_{\mathrm{ref}}(z)\exp(r(z)/\tau)$；实践中，采样分布 $\mu$ 决定我们能不能有效估计并逼近这个 $p^*$。

所以采样分布并没有真的消失。它只是从“定义理论最优解”转移到了“决定统计效率和可学习性”的位置。

## Q: 这个视角有什么问题？

这个视角很有用，但不能过度神化。它更像一个理解工具，而不是直接告诉我们该怎么训练模型的完整算法。

第一，普通 Policy Gradient 只能说“像 weighted MLE 的梯度”，不能直接等同于 weighted MLE。因为 reward / advantage 可以为负，而且样本分布会随策略变化。

第二，权重或 reward 不能单独定义分布。无论是 weighted MLE 还是 exponential tilting，都需要一个 base distribution：有限数据集里的经验分布、PPO rollout 的旧策略分布、DPO 的偏好数据分布，或者 KL 正则里的 $\pi_{\mathrm{ref}}$。如果没有这个 base distribution，只说“按 reward 加权”是不完整的。

第三，指数 tilting 需要配分函数：



\[Z(x) = \sum_y \pi_{\mathrm{ref}}(y\mid x) \exp \left( \beta r(x,y) \right)\]



但 LLM 的回答空间巨大，$Z(x)$ 几乎不可直接计算。DPO 通过同 prompt 的 reward difference 消掉 $Z(x)$，PPO/GRPO 则通过采样和局部优化绕开它。

第四，闭式解 $\pi^*$ 是在“可以任意选择分布 $\pi$”的理想条件下得到的。真实 LLM 只能在参数化模型族 $\{\pi_\theta\}$ 里近似它。

第五，理论目标分布和实际采样分布不一定相同。RLHF 的 tilted target 使用 $\pi_{\mathrm{ref}}$ 作为 reference，但 PPO 的样本通常来自当前策略或旧策略。DPO 的样本来自离线 preference dataset。GRPO 的样本通常来自当前策略对同一 prompt 的多次 rollout。把这些分布混在一起，会造成很多概念误解。

第六，这个视角不能自动解决 exploration。目标分布：



\[\pi^*(y\mid x) \propto \pi_{\mathrm{ref}}(y\mid x) \exp(\beta r(x,y))\]



仍然受到 $\pi_{\mathrm{ref}}$ 的 coverage 限制。如果 reference model 几乎不会生成某类高 reward 回答，那么仅靠这个分布级视角并不能保证模型发现它。

第七，它很容易变成事后解释。几乎任何训练都可以说成“改变分布”。如果这个视角不能给出可计算的诊断量或可检验的预测，它就只是换了一种语言描述已有算法。

因此更合理的定位是：

> 它不是一个新的训练算法，而是一种分布层面的解释框架。它帮助我们理解 reward、preference、verifier、KL、reference model 到底如何共同改变目标分布。

## Q: 相关文献中是否已经有类似观点？

这个视角并不是凭空出现的。更准确地说，本章把几条已经存在的思想放到了一起：weighted likelihood、KL-regularized RL、control as inference、DPO 的 implicit reward、energy model / exponential tilting。不同文献使用的语言不同，但都在接近同一个核心问题：

> 标量反馈如何改变策略分布？

第一类最接近 weighted MLE 视角。Reward-weighted regression 和 Advantage-weighted regression 直接把 RL 写成加权回归或加权最大似然的形式。AWR 明确使用 value regression 和 advantage-weighted policy regression 两个监督学习子问题来做 RL。这类工作已经很清楚地表达了“高 advantage 样本应该获得更大 likelihood 权重”的思想。参考：[Advantage-Weighted Regression: Simple and Scalable Off-Policy Reinforcement Learning](https://arxiv.org/abs/1910.00177)。

第二类最接近 exponential tilting 视角。KL-regularized RL、relative entropy policy search、MPO 和 control-as-inference 都反复出现下面这种结构：



\[p^*(z) \propto p_0(z) \exp \left( \beta r(z) \right)\]



也就是说，这些工作并不只是说“reward 越高越好”，而是在分布层面说明：reference / prior distribution 会被 reward 的指数权重重新倾斜。MPO 和 control-as-inference 尤其接近本章的指数 tilting 表述。参考：[Maximum a Posteriori Policy Optimisation](https://arxiv.org/abs/1806.06920) 和 Sergey Levine 的综述 [Reinforcement Learning and Control as Probabilistic Inference](https://arxiv.org/abs/1805.00909)。

第三类是 RLHF 工程范式。InstructGPT 使用 reward model 和 PPO 来优化语言模型，同时使用 KL 约束避免模型偏离参考策略太远。它没有把“指数 tilting”作为核心术语来展开，但它使用的 reward model + reference policy + KL penalty，正是会诱导 tilted target distribution 的那套结构。参考：[Training language models to follow instructions with human feedback](https://arxiv.org/abs/2203.02155)。

PPO 本身是 policy optimization 的稳定化近似，通过概率比值和 clipping 限制新旧策略差异。它不是“分布闭式解”，但可以看成用采样化 surrogate objective 近似推动策略向高 advantage 区域移动。参考：[Proximal Policy Optimization Algorithms](https://arxiv.org/abs/1707.06347)。

第四类是 DPO 及其后续 preference optimization。DPO 最直接地使用了 KL-regularized RL 的闭式结构。它从：



\[\pi^*(y\mid x) \propto \pi_{\mathrm{ref}}(y\mid x) \exp \left( \beta r(x,y) \right)\]



出发，把 reward difference 改写成 policy 相对 reference model 的 log-ratio。也就是说，DPO 不是只“像”这个视角，而是直接依赖这个 tilted distribution 的闭式形式，只是它通过 pairwise difference 规避了 $Z(x)$。参考：[Direct Preference Optimization: Your Language Model is Secretly a Reward Model](https://arxiv.org/abs/2305.18290)。

后续 preference optimization 工作进一步分析了 DPO、IPO、RLHF 等方法背后的偏好学习假设。比如 Azar et al. 把 RLHF、DPO、IPO 放进统一理论框架，指出这些方法本质上是在用不同方式把 pairwise preference 转换成 policy objective。这和本章“标量或偏好信号诱导目标分布”的观点是同一条线。参考：[A General Theoretical Paradigm to Understand Learning from Human Preferences](https://arxiv.org/abs/2310.12036)。

第五类是对这个视角风险的分析。当 $\beta$ 较大时，tilted distribution 会更集中到高 reward 区域；如果 reward model 有偏差，就可能出现 reward overoptimization、mode collapse 或 diversity loss。这些工作虽然不一定使用“exponential tilting”这个名字，但它们分析的正是“reward-induced distribution 太集中或方向错误”带来的问题。相关讨论可以参考 [Scaling Laws for Reward Model Overoptimization in Direct Alignment Algorithms](https://arxiv.org/abs/2406.02900)、[Understanding Likelihood Over-optimisation in Direct Alignment Algorithms](https://arxiv.org/abs/2410.11677)，以及关于 DPO implicit reward 泛化的 [On the Limited Generalization Capability of the Implicit Reward Model Induced by Direct Preference Optimization](https://arxiv.org/abs/2409.03650)。

所以，本章的观点不是“我们发现了一个没人知道的新等价关系”。更准确地说，它是在把几条已有线索放到一个更直观的图景里：

```text
weighted likelihood
KL-regularized RL
control as inference
DPO implicit reward
energy / exponential tilting
```

这些表述都在从不同角度说明同一件事：

> LLM post-training 中的许多 RL / preference optimization 方法，都可以理解为由 scalar feedback 诱导目标分布，再让模型向这个目标分布移动。

## 小结

这一章可以总结为三句话。

第一，Policy Gradient 的形式让 RL 看起来像 weighted log-likelihood：reward 或 advantage 越高，样本对 log probability 更新的影响越大。没有显式 KL 时，这种重加权诱导出的分布依赖于采样分布和权重的乘积：



\[\tilde{q}(z) = \frac{ q(z)w(z) }{ \int q(z')w(z')dz' }\]



第二，如果加入 KL reference，reward 不只是线性权重，而是通过 exponential tilting 诱导出明确的目标分布：



\[\pi_{\mathrm{target}}(y\mid x) \propto \pi_{\mathrm{ref}}(y\mid x) \exp \left( \beta r(x,y) \right)\]



第三，PPO、GRPO、DPO、RLHF 可以看成对这个分布移动过程的不同近似：PPO 用采样和 clipping，GRPO 基本沿用 PPO-style 目标但用组间样本估计 advantage，DPO 用 preference pair 和 policy log-ratio 隐式消掉配分函数。

最后要特别记住：诱导分布和实际采样分布不是同一个概念。无 KL 的 weighted fitting 中，采样分布 $q$ 会直接进入诱导分布；有 KL reference 的理论闭式解中，$\pi_{\mathrm{ref}}$ 是定义 target distribution 的 reference / prior。但实际算法仍然要从某个分布采样：PPO 通常来自当前策略或旧策略；DPO 的样本来自偏好数据集；GRPO 的样本来自同一 prompt 下的多次 rollout。如果采样分布覆盖不到高 reward 区域，或者权重极端稀疏，那么理论 target 再漂亮，实际训练也很难学到它。分布视角有用，正是因为它迫使我们把“想靠近的分布”和“手里采到的样本”分开看。

因此，LLM 中的 RL 不只是“像传统 RL 那样在环境中学习策略”，也可以被理解成：

> 用 reward / preference / verifier 这样的标量信号，诱导一个新的回答分布，然后用各种可计算的近似方法把语言模型推向这个分布。


