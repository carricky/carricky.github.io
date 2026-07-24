---
title: 'From Score-and-Rank to System Evolution: The LLM-Agent Paradigm Shift in Recommender Systems'
layout: post
categories: machine-learning
description: Why going all-in and replacing the whole recommendation stack with an LLM agent is usually the wrong move, and what should actually change across memory, reasoning, compute, reward, and simulation
giscus_comments: true
---

<div class="lang-switch">
  <button id="lang-toggle" class="lang-toggle-btn" type="button">中文</button>
</div>

<div id="content-en" class="lang-block" markdown="1">

I've been talking to a few people in the field lately, and one thing keeps coming up: recommendation systems are standing at a genuine watershed moment. The debate over how far to push large-model agents into the recommendation stack is real enough that [WWW 2026 dedicated a workshop track to it](https://llmandagents4recsys.github.io/) — and the instinct I keep running into, in conversations and in what practitioners write, is to go all-in: rip out the entire pipeline and hand recall, coarse ranking, ranking, and re-ranking over to a single LLM agent.

My take: that bet mostly loses. Not because LLMs aren't capable enough, but because "using an LLM" and "replacing the whole system with an LLM" get treated as the same thing. One practitioner laid out the technical case well in a recent write-up: [semantic-ID tokenization compresses away the long tail, retrieval has to juggle several competing objectives at once, and a tens-of-milliseconds latency budget is a serving-system problem, not just a modeling one](https://medium.com/@xhp0407/why-generative-recommendation-still-cannot-replace-the-full-ranking-pipeline-4bad2f023135). None of that goes away just because the model gets bigger. The real question isn't "should we adopt an agent," it's "where in the system should the agent actually live, and which decisions should it take over."

This piece is an attempt to answer that, across five dimensions: memory, reasoning, compute architecture, reward design, and simulation environments. The goal is to work out how an LLM agent should be embedded into a recommender system — not how to bluntly swap the whole thing out.

## 1. Rebuilding Memory: From Black-Box Embeddings to Evolvable Cognitive Memory

In traditional recommender systems, the user profile is usually just an embedding table that evolves continuously, or a periodically refreshed set of offline feature tags. This kind of "implicit memory" has some inherent limits:

- **No clean separation by time granularity**: it's hard to cleanly separate a session's temporary interest from a preference that's been stable for five years.
- **Un-debuggable**: semantic drift along an embedding dimension is a total black box to engineers — there's no way to intervene manually or sanity-check the logic.

LLM agent architectures introduce explicit memory, pushing user-state maintenance toward a layered cognitive model. Meta's ["ARS: Agentic Recommender System with Hierarchical Belief-State Memory"](https://arxiv.org/abs/2605.14401) (Shen et al., 2026) is a good illustration of this — it decouples user memory into three tiers:

| Memory Layer | Data Form | Core Function & Mechanism |
|---|---|---|
| Event Memory | Raw signal stream | Real-time capture of interaction traces and explicit feedback (clicks, skips, dislike signals). |
| Preference Memory | Fine-grained knowledge-graph / attribute pairs with confidence weights | Maintains mutable preferences with "confidence strength" and "supporting evidence," handling recent interest drift. |
| Profile Memory | Coherent natural-language document | Uses an LLM to periodically distill low-level signals into a human-readable, editable, debuggable natural-language profile. |

ARS ties these three tiers together with six adaptive operations — extraction, reinforcement, weakening, consolidation, forgetting, and resynthesis — scheduled by an LLM-based planner rather than a fixed interval. The paper reports average gains of 26.4% in HR@1 and 10.3% in NDCG@10 over baselines from this scheduling alone.

**Architectural note**: in practice, I'd decouple the system's self-evolution into two independent pipelines — one that asynchronously updates "what the system knows about the user" (the memory layer), and another that updates "the system's own reasoning and decision-making ability" (the policy layer), synchronously or near-line.

Explicit memory doesn't just make the self-evolution process far more interpretable — it opens up real space on the business and compliance side too. Users get actual data sovereignty over their own preference profile: they can view it, export it in one click, or manually erase a single disliked preference.

## 2. The Evolution of Reasoning: RecSys Is Replaying o1/R1's Reinforcement-Learning Path

Most prior work that brought LLMs into recommendation really just used them as a post-hoc explainer or a beefed-up feature encoder. Genuine agentic behavior requires reasoning to exist as an intrinsic part of decision-making, not a layer bolted on after the fact.

This is essentially the o1/R1 paradigm replaying itself in the recommendation domain: SFT can only teach a model the "output format" and "surface logic" of recommendation — its capability ceiling is capped by the quality of the labeled dataset. Real preference alignment and deep reasoning require reinforcement learning.

### Explicit Intrinsic Reasoning

In the more recent [OneRec-Think](https://arxiv.org/abs/2510.11639) architecture (Liu et al., 2025), the system no longer outputs a recommendation list directly. Instead, it treats a user's historical behavior log as an entirely new continuous/offline modality, token-aligned with the LLM at the hidden-layer level. Before producing the final action, the model first generates an explicit reasoning trace — a chain of thought (CoT).

> Raw behavior log → modality-alignment encoder → reasoning-token space → multi-objective action output

Because preference in a recommendation context has "multi-validity" — there's no single ground truth the way there is in math or code — OneRec-Think designs a multi-objective, distributional reward function specifically for the recommendation setting (blending clicks, dwell time, and a diversity penalty). Deployed on Kuaishou, the paper reports a 0.159% lift in App Stay Time — a modest-looking number, but a real, shipped gain at that scale.

To keep the model from letting reasoning become performative in multi-turn interaction — i.e., generating a plausible-looking chain of thought that's actually disconnected from the final decision — the common industry pattern is a two-stage training recipe:

1. **SFT stage**: quickly gets the backbone model to grasp the domain's basic task format and constraint boundaries.
2. **RL stage**: introduces fine-grained user-preference rewards, letting the model close the loop with the interactive environment and gradually converge toward the user's real, long-horizon value (LTV).

## 3. The Compute and Architecture Ledger: Where to Spend, Where to Save

This is really the same point I opened with: going all-in and replacing the entire pipeline is usually bad math. "Agent architectures are too expensive — they can't meet a recommender system's millisecond-level SLA." That's the most common objection I hear from architects, and to get past it you have to drop the "compute has to blow up everywhere" framing and see the "architectural-consolidation dividend" instead.

### Where should compute go? (Precision targeting)

In an LLM-powered recommendation setting, expensive reasoning compute should never blanket 100% of traffic. It should be aimed precisely at high-value decision points:

- Proactive clarification for ambiguous or long-tail intent (interactive exploration)
- Re-engaging users at high risk of churn
- Guiding long-chain decisions ahead of high-ticket purchases

### Where can compute be saved? (A System 1 / System 2 rearchitecture)

The standard answer here borrows the human System 1 (fast, intuitive) / System 2 (slow, deliberate) division of labor.

Massive online traffic splits in two: System 1 (lightweight retrieval / coarse ranking) handles real-time response at low cost; System 2 (a large model doing a single "reason-once" pass, asynchronously) handles deep reasoning and updates the explicit Profile Memory, at higher compute cost.

- **Reason once, serve many**: the expensive part of LLM inference sits offline or near-line. The system asynchronously triggers the LLM to deeply reason about the user's real intent and update Explicit Memory (the profile document), while the online front end only calls cheap retrieval and lightweight re-ranking.
- **End-to-end generation as a blow to the traditional pipeline**: a concrete piece of industry evidence comes from Kuaishou's [OneRec](https://arxiv.org/abs/2506.13695) — when the traditional multi-stage cascade ("recall → coarse rank → fine rank → re-rank") was torn out entirely and replaced with a unified, end-to-end generative recommendation architecture, **overall operating cost dropped to just 10.6% of the traditional pipeline's**, per the team's own technical report. OneRec has since been fully deployed across the Kuaishou app and its Lite version, serving roughly a quarter of production QPS.

Traditional cascaded architectures waste most of their compute on cross-service transfer (I/O overhead), redundant feature engineering repeated across multiple models, and heavy serialization/deserialization. Once everything runs on a unified large-model backbone, OneRec reports training- and inference-time model FLOPs utilization (MFU) of **23.7% and 28.8%** respectively — approaching the maturity level of general LLM infrastructure. The overhead saved by architectural simplification is enough to absorb the compute cost of running a large model at the single decision point.

**To be clear**: OneRec is an end-to-end generative architecture, not an agent in the multi-turn-interaction sense. It's cited here purely to show how much system-level compute a "consolidated architecture" can free up. The real link is that the compute budget saved by a generative backbone is exactly what can get reinvested into an agent's reasoning overhead — the two are complementary, not the same thing.

## 4. Reward and LTV: Using Explicit Memory to Crack the Credit-Assignment Problem

Traditional recommendation optimizes for CTR or single-impression value, which comes with three built-in flaws:

1. **Noisy**: position bias and clickbait-style content mean the click signal is full of noise.
2. **Not verifiable**: unlike math or code, which have an objective right answer (RLVR-style tasks), a single click is just one individual's random event — there's no physical-world ground truth.
3. **Short-sighted local optima**: this setup easily pushes the system toward filter bubbles and locally optimal taste.

Industry has spent years chasing LTV and long-term retention as an optimization target. But the sparsity of long-horizon metrics runs into the classic credit-assignment problem: if a user churns — or stays — seven days later, which of the dozens of impressions from those seven days actually caused it?

### Explicit Memory as a State Proxy

The agent paradigm offers a genuinely new angle on credit assignment: turn a long-horizon, sparse-reward problem into a local, continuous problem about memory evolving over time.

$$ R_{total} = \alpha \cdot R_{immediate} + \beta \cdot \Delta(Memory\_State) $$

(illustrative formula)

In this kind of hybrid design — combining explicit memory with generative recommendation — the system no longer tries to directly predict retention seven days out. Instead, it evaluates whether the current interaction successfully moved the user's explicit memory document (Profile Memory) toward a healthier "target state." By mapping immediate click reward and long-term engagement reward into a single blended score (a P-Score), the explicit memory layer acts as a smoother for delayed reward, substantially easing RL training divergence in a high-noise, high-drift environment. To be upfront about it: this formula is my own synthesis, not a quote from a single paper — it's what ARS's confidence-weighted preference chunks and OneRec-style reward blending would look like combined. I haven't found a published P-Score weighting; the point is the shape of the idea, not a specific coefficient.

## 5. The Safe Test Track: Where User Simulators Have to Stay Honest

For an RL policy to go live safely, industry needs a high-fidelity user simulation environment. Traditional logging data is observational — it carries heavy selection bias — and directly experimenting on production UI to explore is both costly and risky in terms of user complaints.

User simulators in recommendation are currently going through two technology generations:

### First generation: model-driven

Represented by classic systems like [Virtual Taobao](https://arxiv.org/abs/1805.10000) (Shi et al., Nanjing University & Alibaba, AAAI 2019) and other GAN-based environments. Virtual Taobao fits real customer-behavior dynamics with a GAN-based simulator (GAN-SD) and uses multi-agent adversarial imitation learning (MAIL) — GAIL-style training that implicitly recovers a reward signal through the discriminator, rather than hand-specifying one. This lets the downstream recommendation RL policy operate within a more principled framework, instead of relying entirely on hand-coded rules.

### Second generation: LLM-driven, behavior-aligned

The newer generation of simulators — represented by work like ["Learning User Simulators with Turing Rewards"](https://arxiv.org/abs/2606.19336) (MIT/Stanford, 2026), which trains a "Turing-RL" simulator using an LLM judge that scores how indistinguishable a generated response is from a real user's — no longer just output a flat click probability. They produce natural-language feedback, complete with emotion and critique.

To keep the simulator itself from collapsing into adversarial reward hacking, the core idea is: don't force the simulator to rigidly reproduce "what one specific user said in the past." Instead, use RL to push the simulator's generated interaction sequences to the point where even an LLM judge can't tell them apart from the real thing.

**A finding on closing the sim-to-real gap, worth being precise about**: the intuitive story here — that a "weaker," more cooperative simulated user closes the sim-to-real gap because real users tend to be more forgiving than a synthetic one — doesn't hold up well against the evidence. A 2026 paper, ["Beyond Cooperative Simulators"](https://arxiv.org/abs/2605.12894), reports close to the opposite: agents trained against more realistic, behaviorally diverse simulated personas were about 17% more robust on out-of-distribution behavior than agents trained against homogeneous, cooperative simulators. So the actual fix for the sim-to-real gap isn't a weaker, easier simulator — it's one that's a more honest, less cooperative stand-in for how real users actually behave.

Even so, fully closing off exploitation of simulator quirks (reward hacking against the simulator's blind spots) remains genuinely hard. The most realistic, industry-proven path right now is still a hybrid loop: the simulator serves as a sandbox for large-scale, tens-of-thousands-of-steps pretraining, calibrated against a small slice of real online A/B traffic.

## 6. Looking Ahead: A Qualitative Shift in the Action Space

Talking about how LLM agents reconstruct recommender systems isn't really about a point upgrade to one algorithm. It's a systemic reconstruction spanning capability, objective, and infrastructure all at once.

The most interesting part of this shift is the unlocking of the action space.

A traditional recommender system's action space is scarce and passive by design — it can only decide which 20 items to surface out of a fixed candidate pool of, say, ten thousand, and mechanically order them.

An agentic recommender system, in a multi-turn feedback loop with the user, can proactively ask a clarifying question, call an external tool, generate entirely new multimodal content on the fly, actively reconstruct the candidate pool itself, or even guide the user to jointly explore the edges of preferences neither of them knew about yet.

Which brings me back to the question I opened with: should you go all-in and replace your whole recommender system with an agent? My answer is still no. What's actually worth investing in is getting the agent to live in the right place in the system, opening up the action space step by step — not tearing down a pipeline that took years to tune.

From "a machine that scores and ranks" to "a digital assistant that co-evolves with the user" — the new paradigm for RecSys is just getting started.

</div>

<div id="content-zh" class="lang-block" style="display:none;" markdown="1">

最近跟几个同行聊天，一个越来越明显的感觉是：推荐系统正走到一个分水岭上。"大模型 Agent 到底该在推荐系统里插多深"这个争论是真实存在的——[WWW 2026 专门开了一个 workshop track 讨论这件事](https://llmandagents4recsys.github.io/)。而我在跟人聊、看业内文章时反复碰到的一种本能反应，是想"一步到位"——直接拿一个大模型 Agent，把召回、粗排、精排、重排整条链路推倒重来。

我的判断是，这条路大概率要吃亏。不是因为 LLM 不够强，而是因为大家把"用 LLM"和"把系统整体换成 LLM"划了等号。有位从业者最近写了篇文章，把技术上的坑讲得很清楚——[语义 ID 压缩会丢掉长尾、检索本身要在多个互相竞争的目标之间博弈、几十毫秒的延迟预算是一个服务架构问题而不只是建模问题](https://medium.com/@xhp0407/why-generative-recommendation-still-cannot-replace-the-full-ranking-pipeline-4bad2f023135)——这些都不会因为模型变大就自动消失。真正该问的问题不是"要不要上 Agent"，而是"Agent 应该长在系统的哪个位置、替代哪一段决策"。

这篇文章想把这件事聊清楚：从记忆、推理、算力架构、奖励设计、仿真环境五个维度，拆一拆大模型 Agent 到底该怎么"嵌进"推荐系统，而不是简单粗暴地把它整个取代掉。

## 1. 记忆的重构：从黑箱 Embedding 到可进化的认知记忆（Cognitive Memory）

在传统推荐系统中，用户画像（User Profile）通常等价于连续演进的 Embedding Table 或定期的离线特征标签。这种"隐式记忆（Implicit Memory）"存在天生的局限性：

- **无法区分时间粒度的偏好特征**：难以优雅地剥离当前 Session 的临时兴趣与五年不变的底座偏好。
- **不可调试性（Un-debuggable）**：向量维度的语义漂移对工程师而言完全是黑箱，无法进行人工干预或逻辑校准。

大模型 Agent 架构引入了显式记忆（Explicit Memory），将用户状态维护的标准框架推向了分层认知模型。Meta 的论文 [《ARS: Agentic Recommender System with Hierarchical Belief-State Memory》](https://arxiv.org/abs/2605.14401)（Shen et al., 2026）是个很好的例子——它把用户记忆解耦为三层架构：

| 记忆层级 | 数据形态 | 核心功能与机制 |
|---|---|---|
| 事件记忆（Event Memory） | 原始信号流 | 实时捕获交互轨迹、显式反馈（点击、滑过、反感信号）。 |
| 偏好记忆（Preference Memory） | 带强度的细粒度知识图谱/属性对 | 维护具备"置信度强度"和"事实证据"的可变偏好，处理近期兴趣漂移。 |
| 画像记忆（Profile Memory） | 连贯的自然语言文档 | 利用大模型将低层信号定期蒸馏为人类可读、可编辑、可 debug 的自然语言画像。 |

ARS 用六种自适应操作——提取、强化、弱化、合并、遗忘、重组——把这三层串联起来，由一个 LLM 规划器动态调度（而非固定周期触发）。论文报告仅这一调度机制就带来了平均 26.4% 的 HR@1 和 10.3% 的 NDCG@10 提升。

**架构建议**：在工程落地时，建议将系统的自进化（Self-Evolution）解耦为两条独立的流水线：一条异步更新"对用户的认知"（Memory 层面），另一条同步或近线更新"自身的推理决策能力"（Policy 层面）。

这种显式记忆不仅能大幅提升 self-evolution 过程的可解释性，更在商业和合规层面带来了巨大想象空间——它让用户真正拥有了对自己偏好档案的"数据主权"（可查看、可一键导出、可手动抹除单条反感偏好）。

## 2. 推理的演进：RecSys 在重演 o1/R1 的强化学习之路

此前绝大多数将大模型引入推荐的工作，本质上都只是将其作为"Post-hoc（事后）解释器"或增强特征的 Encoder。真正的智能体化，需要推理能力（Reasoning）以内生决策（Intrinsic Decision）的形式存在。

这正是 o1/R1 范式在推荐垂域的重演：SFT（监督微调）只能让大模型学会推荐的"输出格式"与"表面逻辑"，其能力天花板被局限在标注数据集的质量内；而真正的偏好对齐与深度推理，必须依赖强化学习（RL）。

### 显式内生推理（Explicit Intrinsic Reasoning）

在最新演进的 [OneRec-Think](https://arxiv.org/abs/2510.11639) 架构中（Liu et al., 2025），系统不再直接输出推荐列表，而是将用户的历史行为日志作为一种全新的连续/离线模态（Modality），在隐藏层与大模型进行 Token 级的对齐。模型在输出最终 Action 之前，会先生成一段显式的推理轨迹（Chain of Thought, CoT）。

> 原始行为日志 → 模态对齐编码器 → 思索空间（Reasoning Tokens）→ 多目标动作输出

由于推荐场景的偏好具备 Multi-validity（多重有效性），不存在数学或代码物理世界中唯一的 Ground Truth，因此 OneRec-Think 专门针对推荐环境设计了多目标的分布式 Reward 函数（混合了点击、时长、多样性惩罚）。在快手实际生产环境上线后，论文报告 App Stay Time（用户留存时长）提升了 0.159%——数字看着不大，但在这个体量下是真实上线的收益。

为了防范大模型在多轮交互中流于形式（即避免生成表面化、与最终决策脱节的推理），业界普遍采用两步走的训练范式：

1. **SFT 阶段**：快速让 Backbone 模型掌握场景的基础任务范式与约束边界。
2. **RL 阶段**：引入细粒度的用户偏好 Reward，让模型在与交互环境的闭环互动中，逐步逼近用户真实的深层长周期价值（LTV）。

## 3. 算力与架构的账本：往哪投，怎么省？

这其实也是我开头想说的那件事："一步到位"把整条 pipeline 换掉，很多时候是笔算错的账。"Agent 架构太贵，无法满足推荐系统毫秒级的耗时（SLA）要求。"这是工业界架构师最普遍的质疑。要解开这个死结，需要打破"算力单点膨胀"的思维误区，看清"架构重组红利"。

### 算力往哪投？（场景精准投放）

在大模型推荐场景下，昂贵的 Reasoning 算力绝不应该无脑覆盖全量流量，而应精准定向投喂给高价值决策点：

- 模糊/长尾意图的主动追问（Interactive Exploration）
- 处于高流失风险边缘的用户唤醒
- 大额高客单价消费前的长链路决策引导

### 算力怎么省？（System 1 / System 2 架构重构）

在架构设计上，标准的解法是借鉴人类大脑的 System 1（直觉快速）与 System 2（理性慢速）分工架构。

线上海量流量进入后一分为二：System 1（轻量检索/粗排）负责实时响应、低成本；System 2（大模型 Reason Once，异步）负责深度推理后更新显式记忆 Profile Memory、高算力。

- **Reason once, serve many**：极其昂贵的大模型推理被放在离线或近线（Near-line）层。系统异步触发大模型去深度推理用户的真实意图并更新 Explicit Memory（画像文档），而线上前台只需调用便宜的检索与轻量级重排。
- **端到端生成对传统 Pipeline 的降维打击**：一个具体的工业界佐证来自快手 [OneRec](https://arxiv.org/abs/2506.13695)——当他们彻底干掉传统"召回-粗排-精排-重排"的多阶段级联管线，换成统一的端到端生成式推荐架构后，据其官方技术报告，**整体运营成本降到了传统 Pipeline 的 10.6%**。OneRec 目前已在快手 App 及极速版全量上线，承接约四分之一的线上 QPS。

传统的级联架构因为跨服务传输（IO 开销）、多模型间特征工程冗余计算、大量序列化与反序列化，浪费了系统绝大多数算力。统一大模型底座后，OneRec 报告训练和推理阶段的模型 FLOPs 利用率（MFU）分别达到 **23.7% 和 28.8%**，接近 LLM 基础设施的成熟水平。架构精简省下来的开销，足以弥补大模型单点推理的算力膨胀。

**需要澄清**：OneRec 属于端到端生成式架构，本身并非多轮交互意义上的 Agent；此处引用它，是为了说明"架构重组"能省出多少系统算力。真正的逻辑闭环在于：生成式底座省下来的算力预算，恰好可以再投入到 Agent 的推理开销上，二者互补而非同一件事。

## 4. Reward 与 LTV：利用显式记忆破解信度分配难题

传统推荐优化 CTR（点击率）或单次曝光价值，本质上带有"三层原罪"：

1. **高噪声（Noisy）**：位置偏置（Position Bias）与标题党行为让点击信号充满噪音。
2. **无法客观验证（Not Verifiable）**：与数学、代码等具备客观对错的 RLVR 任务不同，一次点击只是个体的随机事件，没有物理世界的 Ground Truth。
3. **局部最优的短视性（Short-sighted）**：极易将系统推向信息茧房与用户审美的局部最优解。

工业界多年来一直在摸索 LTV（生命周期价值）与长期留存的优化。然而，长期指标的稀疏性导致了经典的信度分配（Credit Assignment）难题——用户 7 天后的留存或流失，究竟归因于 7 天前的哪一次具体曝光？

### 显式记忆作为状态中介（State Proxy）

Agent 范式为破解 Credit Assignment 带来了全新的思路：将长周期稀疏奖励问题，转化为记忆演进的局部连续问题。

$$ R_{total} = \alpha \cdot R_{immediate} + \beta \cdot \Delta(Memory\_State) $$

（示意公式）

在一类融合显式记忆与生成式推荐的混合设计中，系统不再直接预测 7 天后的留存，而是评估当前的交互 Action 是否成功推动了用户显式记忆文档（Profile Memory）向更加健康的"目标状态"演进。通过把即时点击奖励与长期用户活跃度奖励映射为一个混合分数（P-Score），显式记忆层充当了延迟 Reward 的平滑器，极大缓解了高噪声、高漂移环境下的 RL 训练发散问题。说清楚一点：这个公式是我自己的综合构想，不是引用自某一篇论文——它是把 ARS 的置信度加权偏好块和 OneRec 式的 reward 融合放在一起会长什么样。我没有找到已发表的 P-Score 具体权重方案；这里想说明的是这个思路的形状，不是某个具体系数。

## 5. 安全试车场：User Simulator 的诚实边界

为了让 RL 策略能够安全上线，工业界必须拥有高保真的用户仿真环境（User Simulator）。传统的 Logging Data 属于观测数据（Observational Data），带有严重的选择性偏置（Selection Bias）；而直接在生产环境改动 UI 做 Exploration，探索成本与客诉风险又极高。

目前，推荐领域的 User Simulator 正在经历两代的技术跨越：

### 第一代：模型驱动（Model-driven）

以经典的[虚拟淘宝（Virtual Taobao）](https://arxiv.org/abs/1805.10000)（Shi et al.，南京大学与阿里巴巴，AAAI 2019）及其他基于 GAN（生成对抗网络）的环境为代表。Virtual Taobao 用 GAN-SD 拟合真实用户行为动态，并采用多智能体对抗模仿学习（MAIL）——一种 GAIL 类方法，通过判别器隐式学出奖励信号，而不是靠人工指定奖励函数。这种方案可以让后续的推荐 RL 策略在更有原则的框架下运行，而非完全依赖人工硬编码规则。

### 第二代：LLM 驱动与行为对齐（LLM-driven）

新一代仿真器——以 [Learning User Simulators with Turing Rewards](https://arxiv.org/abs/2606.19336)（MIT/Stanford，2026）一类工作为代表，用 LLM Judge 给"生成的回复像不像真实用户"打分来训练 Turing-RL simulator——不再仅仅输出一个死板的点击概率，而是能够输出自然语言形式的、带情绪与批评偏好的反馈。

为了防止 Simulator 本身陷入对抗性坍塌（Reward Hacking），其核心理念是：不强求 Simulator 去生硬拟合"某个具体用户历史上说过什么"，而是通过强化学习让 Simulator 生成的交互序列，达到连大模型 Judge（裁判模型）都无法分辨真伪的境界。

**关于 Sim-to-Real Gap 的一个发现，值得说准确一点**：一个很直觉的说法是——"弱一点、更配合"的模拟用户能缓解 Sim-to-Real Gap，因为真人比合成的模拟器更宽容、更好糊弄。但我查到的证据其实指向相反的方向。一篇 2026 年的论文 [《Beyond Cooperative Simulators》](https://arxiv.org/abs/2605.12894) 报告的结论几乎是反过来的：用更真实、行为更多样化的模拟人设训练出来的 agent，在分布外（out-of-distribution）行为上的鲁棒性，比用同质化、"很配合"的模拟器训练出来的 agent 高出约 17%。也就是说，解决 Sim-to-Real Gap 真正靠谱的办法，不是找一个更弱、更好对付的模拟器，而是找一个更诚实、更不"配合"、更接近真人实际行为的模拟器。

尽管如此，完全杜绝对仿真器漏洞的利用（Reward Hacking against Simulator Quirks）依然充满挑战。目前最现实、在工业界最稳妥的落地路径依然是："Simulator 充当沙盒进行万级步数的规模化预训练 + 极小流量的真实线上 A/B 测试校准"的混合闭环方案。

## 6. 终局展望：动作空间的质变

当我们在讨论大模型 Agent 对推荐系统的重构时，我们探讨的绝不仅仅是算法模型的单点升级，而是一场涉及能力、目标以及基础设施的**体系性重构**。

这场变革最迷人的地方在于动作空间（Action Space）的解封：

传统推荐系统的 Action Space 是极度匮乏且被动的——它只能在一万个确定好的商品候选集里决定吐出哪 20 个，并机械地排好顺序。

而在智能体化的推荐系统里，Agent 在与用户的多轮反馈循环中，可以主动反问、可以调用外部工具、可以动态生成全新的多模态内容、可以主动重构候选集本身，甚至可以引导用户共同探索未知的偏好边界。

这也回到我开头提的问题——要不要"一步到位"用 Agent 把整个推荐系统换掉？我的答案还是：不该。真正值得投入的，是让 Agent 长在系统里正确的位置上，把动作空间一点点打开，而不是把一套打磨了多年的系统推倒重来。

从"打分排序的机器"到"共同进化的数字助理"，RecSys 的新范式大幕才刚刚拉开。

</div>

<style>
.lang-switch {
  display: flex;
  justify-content: flex-end;
  margin-bottom: 1rem;
}
.lang-toggle-btn {
  font-size: 0.85rem;
  padding: 0.3rem 0.9rem;
  border-radius: 999px;
  border: 1px solid var(--global-divider-color);
  background: transparent;
  color: var(--global-text-color);
  cursor: pointer;
  transition: background 0.15s ease, color 0.15s ease;
}
.lang-toggle-btn:hover {
  background: var(--global-theme-color);
  color: var(--global-hover-text-color);
  border-color: var(--global-theme-color);
}
</style>

<script>
document.addEventListener('DOMContentLoaded', function () {
  var btn = document.getElementById('lang-toggle');
  var en = document.getElementById('content-en');
  var zh = document.getElementById('content-zh');
  if (!btn || !en || !zh) return;
  btn.addEventListener('click', function () {
    var showingEn = en.style.display !== 'none';
    en.style.display = showingEn ? 'none' : '';
    zh.style.display = showingEn ? '' : 'none';
    btn.textContent = showingEn ? 'English' : '中文';
  });
});
</script>

<script src="https://hypothes.is/embed.js" async></script>
