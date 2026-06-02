---
paper: "Anticipate and Learn: Unleashing Idle-Time Compute in Proactive Agents"
arxiv: "2605.25971"
authors: ["Haoyi Hu", "Qirong Lyu", "Xianghan Kong", "Weiwen Liu", "Jianghao Lin", "Zixuan Guo", "Yan Xu", "Yasheng Wang", "Weinan Zhang", "Yong Yu"]
year: 2026
venue: "arXiv preprint v2（上海交大 + 腾讯，2026-05-26；NeurIPS 2026 投稿匿名补充包）"
note_number: 20
---

## Claims

- 现有部署的 AI agent 在 request-response 范式下是反应式的：仅在显式 prompt 后才计算响应，任务完成即回到 dormant 状态，交互之间的 idle time 被浪费 [20: §1]。
- 提出 ProAct 架构：用 idle-time compute 在交互间隙预测并满足"可能的未来用户需求"，由两个紧耦合模块驱动——Future-State Prediction（持续预测用户潜在未来需求）与 Idle-Time Acquisition（评估这些需求、对高价值候选分配后台计算、检索并验证证据、生成知识 artifact 写回 memory）[20: §1, §3.1]。
- 在 ProActEval（200 场景 / 40 域）上，prediction-guided 的 Directed Idle 相对 Reactive baseline 把 T100（达成 100% must-have 覆盖所需轮数）降 14.8%、User Effort 降 11.7%、Hallucination Rate 降 28.1%（0.132 → 0.095）[20: §1, Table 2]。
- 消融揭示增益来自"预测方向"而非"后台搜索本身"：Undirected Idle（有 idle acquisition 但无 future-state prediction）花费 69.8k active token/场景，却仅把 T100 降 0.9%、User Effort 降 1.1%、Anticipation Recall 保持 0；加入预测方向后 Directed Idle 相对 Undirected 把 T100 降 14.1%、User Effort 降 10.7%、Anticipation Recall 达 0.428 [20: §5.1, Table 2]。
- 与改造自公开 ProactiveAgent (Lu et al., 2024) 决策协议的 GPT-4o prompting baseline 对比：ProactiveAgent 在 1,572 个可预测需求中只命中 32 个（judge-labeled Anticipation Recall 0.020），而 ProAct 命中 703 个（0.447）——证明"仅仅主动还不够，必须覆盖可预测且 benchmark 相关的需求才能减少后续用户努力" [20: §5.1, Table 3]。
- ProAct 的 memory 层在 MemBench reflective participation 设置上达 SOTA：10k token 下 reflective accuracy 0.843（最佳先前 baseline 0.742），100k 下 0.863（0.833）[20: §1, §5.2, Table 14]。
- value gate 是候选级评分 S(z) = wr·rz + wg·gz + wv·vz + wτ·τz（user relevance / knowledge gap / incremental value / timeliness 四项加权和），仅当 S(z) ≥ θval 才消耗即时搜索预算；下游真实效用在 idle 时不可观测，故用该评分作 acquisition gating 代理 [20: §3.2, §3.4]。
- delivery policy 把每个 artifact 分到 {push, queue, store} 三态：push 仅当期望未来效用超过中断成本（PushScore = Value − Cost + 50，>40 触发通知，≥70 高优先级），useful-but-not-urgent 入 queue，potentially-useful 静默 store 进 memory [20: §3.4, Appendix C]。
- 搜索预算 k 是 operating point 而非应最大化的参数：k 从 4 增到 16 使 Anticipation Recall 从 0.253 升到 0.432，但 active-token 成本稳步上升而 T100 / User Effort 趋平或波动——更高 recall 不线性转化为更少轮数 [20: §5.3, Appendix O]。
- memory-gap augmentation：当 memory 维护识别出 stale / incomplete / weakly-supported / missing 知识时，这些 gap 被转为候选未来需求加入 Zt——让记忆维护主动塑造 acquisition 目标，而非仅作被动存储 [20: §3.3]。

## Assumptions

- **用户未来需求是"可预测的"**——ProAct 的整套价值建立在"dialogue history + persistent memory 使某些未来信息需求可预测"之上；ProActEval 通过 `predictable_after` 链人为构造了可预测的 need chain，但真实开放世界对话的可预测性是否达到 benchmark 水平从未论证 [20: §3.1, §4.1]。
- **idle time 与 idle compute 是"免费/已浪费"资源**——论文反复用"idle time is wasted"作为动机，隐含假设后台计算不产生边际成本；但 Directed Idle 实测每场景额外消耗 111.8k active token，这是真实 API 开销，无论用户最终是否提出该需求都已支出 [20: §1 vs Table 2]。
- **value score 的权重与阈值可硬编码**——实现默认 wr = wg = wv = wτ = 0.25、θconf = 0.6、θval = 60、push 阈值 40 / 高优先级 70，全部为手工常数；论文未对权重做敏感性分析，也未学习这些权重 [20: Appendix C]。
- **LLM judge 与 user simulator 可代表真实人类**——ProActEval 的 user simulator 是 gpt-4o、coverage judge 是 gpt-4o-mini，被评测系统也用 gpt-4o；"正确覆盖 / 失真 / 幻觉"全由 LLM judge 标注，与被测同源模型族 [20: §4.3, Appendix F]。
- **"自主预测未来需求"在伦理/授权上是可接受的**——ProAct 明确把"无需用户预定义任务或日程"作为相对 OpenClaw/Hermes 的卖点（§2），即 agent 自主决定在 idle 时去研究哪些主题；论文仅在 Broader Impacts 把隐私/同意列为"真实部署需补充"的未来工作，未在架构中内建授权约束 [20: §2, Appendix P]。
- **memory backbone 的能力可跨模型迁移**——proactivity 实验（ProActEval）用 gpt-4o 系列，memory 实验（MemBench）用本地 Qwen2.5-7B-Instruct；"memory backbone 能支撑 proactive 行为"的证据是在与 proactivity 不同的模型上取得的 [20: §5.2, Appendix F/H]。

## Method

**核心论点：把"何时为未来做准备"形式化为一个闭环决策问题——foreground 交互后更新 memory、预测未来需求、对高价值候选分配 idle 计算、决定 artifact 的投递方式——并用统一 policy π 而非无约束后台搜索来约束 idle-time compute。**

**❶ 形式化（§3.2）。** 设 Ht = 截至第 t 轮的对话历史，Mt = idle 计算前的 persistent memory 状态，∆t = idle 窗口、Bt = 计算/检索预算。预测器 Zt = fpred(Ht, Mt) 产生候选集，每个候选 z = (qz, ez, cz, ρz)：qz = 预期需求，ez = 来自 Ht/Mt 的 grounding rationale，cz = 预测置信度，ρz = 被选中时的 retrieval plan。policy π 在 (Ht, Mt, ∆t, Bt) 下选候选、分预算、生成 artifact、给每个 artifact 赋投递决策 dz，目标为
max E[Ufuture − λi·Cinterrupt − λb·Cbudget − λh·Rhallucination]。
因下游效用 idle 时不可观测，用候选级 S(z) = wr·rz + wg·gz + wv·vz + wτ·τz 作 acquisition gating 代理。

**❷ Future-State Prediction（§3.3）= fpred 实例化。** 构造紧凑候选集 Zt，成员须可追溯到当前对话 / persistent memory / 已识别 memory gap：
- **local scenario prediction**：从最近若干轮与当前任务外推近期 follow-up 需求；
- **related expansion**：基于 Mt（用户画像 / 对话摘要 / 存储 artifact / 未解决目标）提议邻接主题；
- **memory-gap augmentation**：memory 维护发现的 stale/incomplete/weak/missing 知识转为候选；
- **过滤与优先级**：confidence < θconf 的候选剔除，按主题相似度去重与分组（实现：最多返回 3 个 predictor 候选 + 独立的 memory-gap 候选）。

**❸ Idle-Time Acquisition & Delivery（§3.4）= π 的 acquisition + delivery 组件。**
- **Value evaluation**：算 S(z)，仅 S(z) ≥ θval 才即时 acquire；低于阈值入 queue/store。
- **Memory-aware acquisition**：先查 Mt 覆盖——高覆盖复用存储证据、部分覆盖只搜缺失子主题、低覆盖则分解为子问题做迭代检索+证据抽取+覆盖检查（增量式而非每个需求全量重启）。
- **Artifact generation**：检索/复用证据合成紧凑 artifact Az（含支撑的候选需求、preparation note、链接到记忆/检索证据的 provenance）。
- **Utility-aware delivery**：选 dz ∈ {push, queue, store}（实现为 PushScore = Value − Cost + 50，clip 到 [0,100]）。
- **Memory update**：artifact + provenance 写回 memory。闭环为 (Ht, Mt) → Zt → S(z) → Az → dz → Mt+1。

**❹ memory 层。** 统一 vector / relational / document 存储 + active knowledge lifecycle；增量抽取管线在每轮后产出 profile_updates / updated_summary / key_info / user_sentiment（七类情感 + [−1,1] intensity）/ extracted_facts（Appendix J）。

**❺ ProActEval 构造（§4）。** 每场景围绕自包含 fact sheet（带稳定 ID 的原子事实，全部虚构实体）+ 有序 user needs（带 importance / grounding fact IDs / turn order / predictable_after / reveal group）构成隐藏的 user-needs graph；assistant 运行时看不到该 graph，仅 simulator + judge 用它判定何时主动覆盖应减少未来努力。五个 cognitive archetype 作为构造控制（非任务标签）。三阶段评测循环：user simulator 按序发已覆盖则跳过的需求 → 被测系统仅用运行时可见信息响应 → LLM coverage judge 标注 facts_conveyed / distorted / hallucinated / needs_addressed（reactive vs proactive 模式）。

## Eval

**两套评测：**

1. **ProActEval（§5.1, Table 2, 200 场景 / 40 域，每域 5 场景）。** 三条件：Reactive（双模块关闭）/ Undirected Idle（有 acquisition 无 prediction）/ Directed Idle（全 ProAct）。配置：seed 42，simulator = gpt-4o，judge = gpt-4o-mini，per-search query budget = 1，per-idle intent ≤ 3，idle trigger = 5s。
   - 效率：T80 6.615 → 5.530（−16.4%）、T100 8.110 → 6.910（−14.8%）、User Effort 9.140 → 8.075（−11.7%）（Directed vs Reactive）。
   - 覆盖与预测：Total Coverage 0.892 → 0.956、Must-Have Coverage 0.938 → 0.977、Anticipation Recall 0.000 → 0.428。
   - 事实完整性与成本：Fact Accuracy 0.972 → 0.985、Hallucination Rate 0.132 → 0.095（−28.1% 相对）、Active Tokens 0 → 111.8k。
   - 配对 bootstrap 95% CI（Appendix G，10k resample，seed 2026）：所有主要 delta 区间不跨 0（如 T100 vs Reactive −1.20 [−1.45, −0.96]）。
   - 预算扫描（§5.3，50 场景子集，k ∈ {4,8,12,16}）：Directed Idle 在每个匹配预算下 T100 / User Effort 均低于 Undirected；Anticipation Recall 随 k 升（0.253 → 0.432），但 active-token 单调升而 T100/UE 趋平。

2. **MemBench reflective participation（§5.2, Table 14, Qwen2.5-7B-Instruct，本地 Apple M5 / 16GB）。** reflective accuracy：ProAct 10k = 0.843 / 100k = 0.863，对比 7 个 baseline（FullMemory / RecentMemory / RetrievalMemory / GenerativeAgent / MemoryBank / MemGPT / SCMemory），均为最高；按场景细分 food 0.94 / book 0.87 / movie 0.80 / emotion 0.76（10k）。memory 操作延迟 read ~0.04s / write ~0.06s（Table 16）。

**基线覆盖：** 比 note 08 充分得多——含 Reactive / Undirected Idle 双消融（隔离"预测方向"贡献）、ProactiveAgent 外部协议适配、MemBench 7 个已发表 memory baseline。

**失败模式分析（Appendix O，论文自述）：** (a) reactive 兼容性回退——6/200 场景 Directed Idle 的 must-have 覆盖反低于 Reactive（museum 保护队列案例覆盖 1.000 → 0.500）；(b) precision-recall 解耦——192/200 场景有非零 anticipation recall，但其中 82 个 User Effort 未下降；(c) low-value push 压力 + search direction failure。

## Weaknesses

- **"idle-time compute is wasted/free"的动机叙事与 111.8k token 的实际开销直接冲突，且未对该成本定价。** 论文通篇把 idle 计算当作回收"被浪费的时间"，但 Directed Idle 每场景多花 111.8k active token 换约 1.2 轮 T100 缩减。这不是回收闲置资源，而是把响应期计算搬到 off-peak 并放大——若 57% 可预测需求都未被命中（Anticipation Recall 仅 0.428）、且 82/200 场景 recall 升而 User Effort 不降，则大部分 token 是投机性消耗。论文把它归为"operating-point trade-off"，但从未给出 token 成本 vs 用户价值的货币化或单位化曲线 [20: §1 vs Table 2, §5.3]。
- **"−28.1% hallucination"是 benchmark 设计的产物而非可迁移机制。** ProActEval 的所有事实都封闭在可审计 fact sheet 里，幻觉定义为"无法追溯到 fact sheet 的内容"。Directed Idle 通过提前检索把证据预置进 memory，幻觉率从 0.132 降到 0.095（绝对仅 3.7 个百分点，Fact Accuracy 本已 0.972）。在封闭世界提前抓取受控事实自然减少无依据陈述——这不能外推为开放世界的幻觉抑制机制；"28.1%"是相对包装 [20: §4.1, Table 2]。
- **value gate / delivery gate 全是手工常数，无学习无敏感性分析。** S(z) 四权重一律 0.25、θconf=0.6、θval=60、push 阈值 40/70 都是 implementation default（Appendix C）。论文标题含"and Learn"，但被"学习"的只是 memory 内容增长，gating policy π 本身没有任何参数被优化（与 note 08 形式化的 π_θ 学习相反，ProAct 的 π 是规则化的）。改权重是否改变结论无从判断 [20: Appendix C vs §1 标题]。
- **跨两套评测换模型，削弱"memory backbone 支撑 proactivity"的链路。** proactivity（ProActEval）用 gpt-4o 系列、memory（MemBench）用 Qwen2.5-7B；"memory backbone 能可靠支撑 proactive 行为"（§5 研究问题 2）的证据是在与 proactivity 不同的模型上取得，二者无法直接拼成同一系统的端到端证据 [20: §5.2, Appendix F]。
- **同源 LLM judge 闭环。** simulator(gpt-4o) + 被测(gpt-4o) + judge(gpt-4o-mini) 同属一个模型族，"coverage / hallucination"由同源裁判标注，无第三方标注、无 inter-rater agreement、无人审 rubric（与 note 19 同型的可复现性缺口，但 note 19 至少是人审 trace；此处连人审都缺）[20: §4.3]。
- **"自主预测未来需求"在架构层无授权/治理出口。** 论文以"无需用户预定义任务或日程"作为相对 OpenClaw/Hermes 的核心差异（§2），即 agent 自行决定 idle 时去检索哪些主题；但 delivery gate 的 Cost 项度量的是"中断成本"（是否打扰用户），而非"该 agent 是否被授权研究此主题"。一个自主决定去检索用户未提及主题的 agent，其 provenance 字段记录的是"我检索了什么"，不是"谁授权我检索"——授权链缺失，论文仅把隐私/同意推给 Broader Impacts 的部署清单 [20: §2, §3.4, Appendix P]。
- **预测器对"benchmark 内可预测性"过拟合的风险未隔离。** ProActEval 的 need 由 `predictable_after` 链人为定义为可预测，Future-State Prediction 的两类来源（local scenario / related expansion）恰好对齐这种链式结构。论文未提供"真实人类对话可预测性分布"的任何外部锚点，也未在非合成对话日志上复测 Anticipation Recall——0.447 这个数字可能是对构造规律的拟合 [20: §4.1 vs §5.1]。
- **regression 案例（museum 0.500 覆盖）暴露"主动准备挤占 reactive 预算"的结构性风险，但仅当作 3% 噪声带过。** Appendix O 把它归为 reactive compatibility regression，未做机制级根因（是 context budget 竞争？还是 push 改变了后续对话轨迹？）的隔离实验，也未给出"哪些场景特征会触发回退"的可操作判据——这对企业生产部署是关键的安全边界问题，被压缩为一个失败模式段落 [20: Appendix O, Table 13]。

## Relations

- **builds-on / competes-with 08_simulating_human_cognition_heartbeat_driven_autonomous [high]**：两篇是 thesis goal 2（Heartbeat + Cron + Reflection 主动 Runtime）的**同一问题、两种证据强度**。HSC[8] 提出心跳触发 Dream Mode 做"空闲态自主认知活动"（记忆压缩 / 反事实回放 / 自主目标设定），但实验是合成 LSTM next-action 预测、零 LLM、零 Dream 实例化、无基线——证据为零；ProAct 几乎覆盖 HSC 描述的同一空闲态用途（memory 维护 / 证据预取 / 未来需求预测），但给出了 HSC 缺失的全部东西：200 场景 benchmark、Reactive/Undirected/ProactiveAgent 三类基线、bootstrap CI、MemBench SOTA。两者**机制重叠**（idle-time compute、memory-driven anticipation、空闲态主动准备）但 ProAct 用 prediction-guided gating 取代 HSC 的 intrinsic goal generation。这是 builds-on（继承"idle = 主动准备机会"的范式立场）兼 competes-with（在"如何把空闲变成有用工作"上给出可证伪的工程实现，直接补齐 HSC 的空白）。ProAct 的存在使 thesis 中"goal 2 学术稀缺"的判断需要更新：稀缺的是"被严格评测的"主动 Runtime，而非"主动 Runtime 的想法"。

- **contradicts 19_think_before_you_act_a_neurocognitive [high]**：ProAct 与 PAGRL[19] 在"主动行为是否需要 action provenance 治理出口"上**正面冲突**，并直接复用 thesis goal 2 的治理张力决议。
  - **ProAct 路径**：明确以"无需用户预定义任务或日程，自主推断未发声的未来需求"为卖点（§2）——这正是 thesis（由 [8] 触发、[19] 强化的决议）所禁止的"intrinsic goal generation 进入生产路径"。ProAct 的 delivery gate 用"中断成本"门控 push，但不要求每个 idle acquisition 有显式调度策略来源。
  - **PAGRL 路径**：每个 consequential action 前必经 4 阶段 deliberation，agent 自身产生的、无显式授权来源的 intent 应被 ESCALATE。
  - **结构性映射**：ProAct 的 {push, queue, store} 三态恰好可被 PAGRL 治理化——store（静默进 memory，无外部副作用）≈ 低治理风险、可自主；push（主动通知用户 / 触发外部动作）≈ 必经 ESCALATE/HITL。即 thesis 的"governance-aware bounded autonomy"可表述为：**ProAct 的 idle research 与 store 模式可保留为"建议生成层"，但任何 push（尤其触发外部副作用的）必须经 [19] PAGRL ESCALATE + ISF + HITL，且 idle acquisition 的 intent 必须可追溯到 cron / 用户授权 / Composer plan，而非 ProAct 现状的"自主推断"**。ProAct 为 thesis 提供了"未治理的主动端"长什么样的具体参照，[19] 提供其治理改造模板 [20: §2, §3.4 vs 19: §4.1, §4.4]。

- **builds-on 17_fademem_biologically_inspired_forgetting_for_efficient [med]**：ProAct 的 memory-gap augmentation（memory 维护识别 stale / incomplete / weakly-supported / missing 知识 → 转为主动 acquisition 目标，§3.3）与 FadeMem[17] 的生物启发遗忘/衰减是**同一记忆生命周期问题的正反两面**——FadeMem 决定"哪些记忆该衰减剔除"，ProAct 决定"哪些记忆 gap 该主动补全"。两者共同构成 thesis goal 5"记忆库不会无限膨胀 + 经验继承"所需的双向 lifecycle（写入补全 + 衰减剔除）。ProAct 把 gap 检测从被动存储升级为主动 acquisition 信号，是对 FadeMem 类衰减机制的补充而非竞争 [20: §3.3 vs 17]。

- **competes-with 18_governed_memory_a_production_architecture_for [med]**：两篇都主张"memory + provenance"，但治理深度截然不同。Governed Memory[18] 把治理（governance variable / schema / append-only audit / 人机共享 MCP 接口）作为记忆层的一等公民；ProAct 的 provenance 只是 artifact 上的"证据来源"元数据，用于复用时维持 factual grounding，**不承载授权 / 访问控制 / 审计语义**。对 Decision Agent（thesis Harness-first 治理优先）而言，ProAct 提供了"主动记忆补全"的机制候选，但其 provenance 必须被 [18] 式治理 schema 包裹后才能进生产——ProAct 的 memory write-back 路径缺 [18]/[19] 的 pre-action 治理检查 [20: §3.4 vs 18: §1, §3]。

- **competes-with 06_why_do_multi_agent_llm_systems [low]**：ProAct 是单 agent 时序范式，不直接面对 MAST 的多 agent 失败分类；但其"−28.1% hallucination"声明若被 thesis 用作"主动准备降低幻觉"的证据，须与 MAST 的 FM-2.6 Reasoning-Action Misalignment / FM-3.3 Incorrect Verification 对照——ProAct 的幻觉下降来自封闭 fact sheet 的提前检索，无法证明能减少多 agent 长链任务中的任一类 MAST 失败占比 [20: Table 2 vs 06: §4]。

- **orthogonal to 07_mcp_zero_active_tool_discovery_for [low]**：MCP-Zero 的"主动工具检索"驱动对象是工具召回，ProAct 的"主动需求预测"驱动对象是未来信息需求；两者都是"主动 vs 被动"思路但层级不同——ProAct 决定"idle 时去研究什么主题"，MCP-Zero 决定"该研究阶段如何写工具请求"，可叠加但论文互不触及 [20: §3.3 vs 07]。

### Relation to thesis

直接命中 thesis **goal 2（Heartbeat + Cron + Reflection 主动 Runtime）**与 **goal 5（跨会话层级化 Memory）**两条主线，并对 thesis "学术稀缺 = 机会窗口"与"Decision Agent 与 OpenClaw 边界判据"两条可证伪点产生影响。

**1. goal 2：ProAct 是 HSC[8] 缺席的"被严格评测的主动 Runtime"，但落在 thesis 明确拒绝的"未治理主动端" [high 关联度]。**
thesis 已决议 goal 2 必须是 governance-aware bounded autonomy——主动行为须有显式调度策略来源、TraceAI-first、外部副作用必经 ISF + HITL。ProAct 恰恰相反：以"无需用户预定义任务或日程的自主需求推断"为卖点（§2）。对 Decision Agent 的可操作启示是**采纳其机制、改造其触发**：
- ProAct 的 Future-State Prediction + memory-gap augmentation 可作为 Reflection"建议生成层"的算法骨架（预测候选 + value gate 评分），但 intent 来源必须从"自主推断"改为 cron / 用户授权 / Composer plan 派生（满足决议 a）；
- ProAct 的 {push, queue, store} 三态天然对齐 thesis 治理分轨：store = 静默进 memory（低风险，可自主）；push = 外部可见动作（必经 [19] PAGRL ESCALATE + HITL，满足决议 c）；
- ProAct 缺的 TraceAI-first（决议 b）须补——其 provenance 仅记"检索了什么"，须扩为"谁授权 + 先写审计再执行"。

**2. 修订可证伪点"学术稀缺 = 机会窗口"[high 关联度]。**
该点此前被 [8] 部分证伪为"问题可由 cron + ReAct + 治理围栏简单解决"。ProAct 提供**反向证据**：Undirected Idle（≈ 无方向的后台 search，最接近"朴素 cron 触发 ReAct 背景探索"）花 69.8k token 仅得 ~1% 增益，ProactiveAgent prompting baseline 在 1,572 需求中只命中 32 个——说明"主动准备"若无 prediction-guided 方向，几乎无用。即**朴素 baseline 不足以解决主动预测问题**，"预测方向"是真增益来源。但 Directed Idle 的代价是 111.8k token/场景，且 57% 可预测需求仍未命中。因此该可证伪点应更新为：**主动需求预测不是 cron+ReAct 可平凡解决的（[8] 推断在此被部分反驳），但学术解法昂贵且未治理；Decision Agent 的工程取舍是"在治理边界内做有限预测"——保留 prediction-guided 方向（避免 Undirected 的浪费），但用 store-only + 显式调度来源约束成本与授权**。建议在 thesis 可证伪点表"学术稀缺=机会窗口"行补注：[20] 反驳"朴素 baseline 足够"，但成本侧（111.8k token/场景）与授权侧（自主 intent）两个缺口仍支持"治理边界内有限主动"的取舍 [20: §5.1, Table 2/3]。

**3. Decision Agent 与 OpenClaw 边界判据：ProAct 显式点名 OpenClaw，给出"治理深度差异"之外的第二维边界 [med 关联度]。**
thesis 把 Decision Agent vs OpenClaw 定为"治理深度差异 + 双方都具备 7×24 能力"的并列形态。ProAct §2 明确把 OpenClaw 与 Hermes 归为"always-on personal assistants，但主动行为仍由用户指定的 schedule / routine / 显式 automation 指令触发"，并把 ProAct 定位为"无需预定义即自主推断未发声需求"。这给 thesis 的边界判据增加一条**触发自主性轴**：OpenClaw/Hermes（cron + 用户定义触发，低自主）— Decision Agent 治理边界内（cron + 授权 Reflection + Composer plan 派生，中自主且全治理）— ProAct（完全自主推断，高自主但无治理）。Decision Agent 的差异化因此可表述为：**在"触发自主性"上不追求 ProAct 的极端，在"治理深度"上超越三者**——即用 ProAct 的预测机制 + OpenClaw 的显式触发约束 + [18]/[19] 的治理底座组合，而非选择 ProAct 的未治理自主 [20: §2 vs thesis OpenClaw 段]。

**4. goal 5：memory-gap augmentation 为"双向记忆 lifecycle"补全主动写入端 [med 关联度]。**
thesis goal 5 关注 User/Role/Org 三层记忆的合并/冲突/衰减 + 不无限膨胀。ProAct 的 memory-gap augmentation（gap 检测 → 主动 acquisition）与 [17] FadeMem 的衰减/遗忘构成 lifecycle 两端（补全 + 剔除）；其增量抽取管线（profile_updates / updated_summary / key_info / extracted_facts，Appendix J）为 Decision Agent build_memory/search_memory 基础链路提供字段级参考。但 ProAct 是扁平 User 级记忆，**无 Role/Org 层级**，也无 [10] Collaborative Memory 的多用户访问控制——其机制须被 thesis 的层级化 + 治理 schema 包裹后才可用 [20: §3.3, Appendix J vs 15/16/17/10]。

**新增可证伪点（建议加入 thesis 表）：**
- "Decision Agent 在治理边界内的'有限主动预测'（store-only + 显式调度来源 + prediction-guided）能否保留 ProAct 多数增益——若实测发现去掉 push 自主性后 User Effort 改善 < Directed Idle 的一半，则'治理边界 vs 主动收益'存在不可调和取舍，goal 2 需重估。"
- "ProAct 式 idle-time 预测在 Decision Agent 真实（非合成、无 predictable_after 标注）业务对话上的 Anticipation Recall 是否显著 > 0——若接近 Undirected Idle 水平（即真实对话可预测性远低于 ProActEval 构造），则主动预测的 111.8k token/场景成本无法在企业场景被价值证成。"

**证据强度：** 整体 [med]——相比 note 08（[low]，零相关实验）证据强得多：200 场景 benchmark + 双消融 + bootstrap CI + MemBench SOTA + 公开 ProactiveAgent 对比 + 匿名可复现包。但限制明确：closed-world 合成 benchmark（论文自陈）、同源 LLM judge、两套评测换模型、gating 全为手工常数、自主 intent 无治理出口、111.8k token 成本未对价值定价。**作为"主动 Runtime 机制骨架与评测协议参考"足够**（Future-State Prediction / value gate / 三态 delivery / ProActEval need-graph 构造法可直接借鉴），**作为"主动预测在企业场景净收益"的证据则不充分**——Decision Agent 落地前须在真实业务对话上复测 Anticipation Recall 与 token 成本，并必须以 [19] PAGRL ESCALATE + store-only 默认 + 显式调度来源把 ProAct 的"自主推断"改造为"治理边界内有限主动"。
