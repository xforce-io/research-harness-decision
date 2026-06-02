# Decision Agent: Research Report

> **Version:** v18 (18 papers)
> **Last Updated:** 2026-06-02
> **Papers:** [01](notes/01_co_evolving_llm_decision_and_skill.md), [02](notes/02_graph_of_agents_a_graph_based.md), [03](notes/03_why_reasoning_fails_to_plan_a.md), [04](notes/04_rethinking_the_value_of_multi_agent.md), [05](notes/05_agent_as_a_graph_knowledge_graph.md), [06](notes/06_why_do_multi_agent_llm_systems.md), [07](notes/07_mcp_zero_active_tool_discovery_for.md), [08](notes/08_simulating_human_cognition_heartbeat_driven_autonomous.md), [09](notes/09_llm_based_multi_agent_blackboard_system.md), [10](notes/10_collaborative_memory_multi_user_memory_sharing.md), [11](notes/11_retrieval_models_aren_t_tool_savvy.md), [12](notes/12_automated_composition_of_agents_a_knapsack.md), [13](notes/13_toolomni_enabling_open_world_tool_use.md), [16](notes/16_g_memory_tracing_hierarchical_memory_for.md), [17](notes/17_fademem_biologically_inspired_forgetting_for_efficient.md), [18](notes/18_governed_memory_a_production_architecture_for.md), [19](notes/19_think_before_you_act_a_neurocognitive.md), [20](notes/20_anticipate_and_learn_unleashing_idle_time.md)
> **Thesis:** [.researcher/thesis.md](.researcher/thesis.md)

---

## 定位与核心张力

Decision Agent 的论题是"治理优先、Harness-first 的企业级决策智能体底座"。这个定位制造了两类结构性张力，贯穿全部设计目标：

**张力 A — 能力 vs. 治理**：文献展示的最高性能机制（G-Memory LLM 生成洞见 [16]、ToolOmni RL 查询改写 [13]、Knapsack 沙盒验证 [12]、FadeMem LLM 仲裁的冲突解决与融合 [17]）都依赖某种形式的自主演化，而 TraceAI 要求所有主动行为有显式来源追踪。每个设计目标都需要在两者之间画一条线——但治理友好度也分等级：FadeMem 衰减是闭式确定性的（v_i(t) 任意时刻可前向还原），比 [16] 的 LLM 黑盒 insight 生成对 TraceAI 友好得多。

**张力 B — 多智能体必要性 vs. 单智能体基线**：[4] 证明同构 MAS 等价于单 LLM 顺序调用（OneFlow 折叠等价）——多智能体协议只有在突破上下文溢出 [9] 或监督委派容量 [12] 的物理边界时才有正当性。

**张力 C — 内化 deliberation vs. 外部确定性强制**：随 v17 进入 corpus 的 [19] PAGRL 与 [18] Governed Memory 是同作者群同期产出的"动作层 + 记忆层"治理配对，构成 thesis ISF + TraceAI 的首份参考架构骨架；但两者都把治理嵌入 LLM 推理本身（PAGRL 的 deliberation、[18] 的 LLM-extracted dual modality）——这与论题"action provenance must be explicit"要求确定性兜底的 ISF 路径只能并存而非二选一。Decision Agent 落地必须双层叠加：PAGRL/Governed Memory 在前做 deliberation 与 trace（柔性、可解释、可泛化），AgentSpec 类外部规则引擎在后做兜底（硬性、确定性、不可绕过）——[19] §6.3 自身建议这种叠加，但未在评测中实现。

---

## 架构前提 — 多智能体必要性

**OneFlow 折叠等价及其两个物理边界**

[4] 的核心发现：对于同构 MAS（所有 worker 共用同一模型），若消息历史可完整传递，等价于单 LLM 顺序调用。这是论题"开箱即用"组件的设计基础——默认路径是强单智能体，MAS 仅在以下两个物理边界之一被触发时才引入：

1. **上下文溢出边界** [9]：当累积工具输出 + 任务状态超过单一上下文窗口时，Blackboard 广播架构将状态外化为共享黑板，允许多个 worker 各自维持小窗口。这是 MAS 的第一正当性条件。
2. **监督委派容量边界** [12]：当 Supervisor 的认知负荷超过单轮决策容量（worker 数量 × 任务深度）时，Knapsack Composer 将 agent 选择形式化为约束优化问题。这是 MAS 的第二正当性条件。

在两个边界均未触及之前，[4] 表明强单智能体（GPT-4o + 工具）在 SWE-bench Lite 59.3% 的 SR 上击败绝大多数多智能体协议 [4: §4.1]。这是对论题"开箱即用"的最强实证支持，也是任何 MAS 扩展必须证明超越基线的责任。

**图路由与拓扑选择** [2]

Graph-of-Agents 的贡献是将 MAS 通信拓扑从隐式（每个框架硬编码）变为显式可配置。三类图（完全图、层次图、DAG）在不同任务结构下有不同最优解，层次图在问题规模增大时收益最显著 [2: §4.2]。对论题的含义：Composer（goal 4）需要一个拓扑选择层，不能固定使用中央 Supervisor 模式。

---

## Goal 1 — 开箱即用

**规划机制的结构性局限** [3]

步进式 LLM 推理（CoT、ReAct、Reflexion）在形式上等价于最大化局部代理分数的贪心策略——这是 [3] Proposition 3.2/3.3 的证明结果，不是工程缺陷。对于固定 beam width B，存在使最优轨迹在深度 1 被不可逆剪枝的环境；k=1 前瞻在最坏情况下严格优于任意步进策略。

实证数据：首步决策中，贪心策略在 55.6% 的情况下选中"陷阱"动作，首步错误后单步恢复概率仅 5.4%；LLaMA-8B + FLARE 频繁超越 GPT-4o + 标准推理 [3: §5.2]。

**对 goal 1 的工程含义**：开箱即用的默认 worker 若采用步进式推理，在超过若干步序列决策的任务上将系统性失败。"开箱即用"必须捆绑一个最小树搜索或前瞻机制，或明确标注适用任务范围（短序列、局部可评估）。

**Knapsack 沙盒验证对 goal 1 的贡献** [12]

[12] 的 design-time sandbox 可在部署前测量 agent 组合的实际规划深度——这是 goal 1"低配置成本"与 goal 4"Composer"的交叉点。ZCL (Zero-Cost Localization) knapsack 将 agent 选择形式化为 0-1 背包问题，成本函数包含 token 消耗 + 延迟估计 [12: §3]。

**局限**：[3] 的实验场景（KGQA 图遍历、ALFWorld）假设确定性状态转移与显式状态表示，从图遍历"长时序"到开放式工具调用工作流的距离尚未在文献中测量 [3: Weaknesses]。

---

## Goal 2 — Heartbeat + Cron

**ISF 结构性张力** [8]

Heartbeat 机制 [8] 将 LLM 置于主动唤醒循环中：固定间隔检查任务队列，触发主动思考与下一步行动规划。这模拟人类认知的"工作记忆刷新"机制，在长时任务中防止上下文漂移。

核心结果：Heartbeat 调度使 agent 在 LongBench 长文档任务中准确率提升 18.3%，主动中断人类低质量输入的能力提升 2.1×  [8: §4]。

**ISF 结构性张力**：ISF（Idle State Fallback）要求 agent 在无显式任务时保持静默，不主动消耗资源或产生副作用。Heartbeat 的固定间隔轮询与 ISF 存在结构性冲突——每次 tick 都是一次主动行为，即使没有待处理任务。

**治理解决方案**：goal 2 须设计为"治理边界内的有限主动模式"：Cron 调度的触发必须有显式来源（用户配置的时间表），不得依赖 LLM 自主决定唤醒时机。Heartbeat tick 的上限频率和副作用范围须在系统配置层固定，不可由 worker 动态调整。

**PAGRL 作为 governance-aware bounded autonomy 的工程模板** [19]

[19] PAGRL 直接为 thesis 已决议的 goal 2 治理张力提供工程模板：(a) **显式调度策略来源** ≈ PAGRL Stage 1 "Intent formation" 必须可追溯到具体调度入口（cron / 用户授权 / Composer plan）——任何 agent 自身产生的 intrinsic goal（如 [8] HSC Dream Mode 的"自主目标设定"）应在 PAGRL Stage 3 被识别为"未授权 intent"并 ESCALATE；(b) **TraceAI-first** ≈ PAGRL Stage 4 强制 trace 在动作执行**之前**写入；(c) **ISF + HITL** ≈ ESCALATE 的"不可逆 → 必经人审"完全对应"外部副作用必须 HITL"。本论文未直接讨论 HSC，但其 ESCALATE 通路在结构上正是 [8] Dream Mode autonomous goal generation 应当落入的治理出口——"Heartbeat + Cron 提供调度入口；Reflection 仅生成建议、所有衍生 action 都经 PAGRL"。**关键工程边界**：[19] N=40 案例（Wilson 区间 [83.5%, 99.4%]）+ 0.65s/动作延迟 + 7 条规则全量注入——50+ 规则规模的延迟与 token 曲线未量化 [19: §5.3, §5.1]。

**ProAct：被严格评测的主动 Runtime，但落在"未治理主动端"** [20]

[20] ProAct 是 corpus 中**首篇为 goal 2"主动 Runtime"提供严格评测**的工作，补齐 [8] HSC（合成 LSTM、零 LLM、无基线、证据为零）缺失的一切：用 idle-time compute 在交互间隙预测并预取"可能的未来用户需求"，由 Future-State Prediction（local scenario / related expansion / memory-gap augmentation 三来源生成候选 Zt）+ Idle-Time Acquisition（value gate S(z)=Σw·指标 评分 → S(z)≥θval 才即时检索 → artifact 写回 memory，按 {push, queue, store} 三态投递）两模块驱动。ProActEval（200 场景 / 40 域）：Directed Idle 相对 Reactive 把 T100 降 14.8%、User Effort 降 11.7%、Hallucination Rate 降 28.1%（0.132→0.095），Anticipation Recall 0.000→0.428，代价是 111.8k active token/场景 [20: §3.1-3.4, Table 2]。

**消融是关键发现**：增益来自"预测方向"而非"后台搜索本身"——Undirected Idle（有 acquisition 无 prediction，≈ 朴素 cron 触发 ReAct 背景探索）花 69.8k token 仅把 T100 降 0.9%、Anticipation Recall 保持 0；改造自公开 ProactiveAgent 的 prompting baseline 在 1,572 可预测需求中只命中 32 个 [20: §5.1]。这直接更新了 thesis 可证伪点"学术稀缺 = 机会窗口"（见可证伪点追踪 F23）：朴素 baseline 不足以解决主动预测，"预测方向"是真增益——但学术解法昂贵（111.8k token/场景）且未治理。

**但 ProAct 落在 thesis 明确拒绝的"未治理主动端"**：它以"无需用户预定义任务或日程、自主推断未发声需求"为卖点（§2），正是 thesis 已决议禁止的"intrinsic goal generation 进入生产路径"。Decision Agent 的工程取舍是**采纳机制、改造触发**：

- Future-State Prediction + memory-gap augmentation 作为 Reflection"建议生成层"算法骨架，但 intent 来源从"自主推断"改为 cron / 用户授权 / Composer plan 派生；
- {push, queue, store} 三态天然对齐治理分轨：store（静默进 memory，无外部副作用）可自主；push（外部可见动作）必经 [19] PAGRL ESCALATE + HITL；
- ProAct 缺的 TraceAI-first 须补——其 provenance 仅记"检索了什么"，须扩为"谁授权 + 先写审计再执行"。

这与 thesis 的 OpenClaw 边界判据形成第二维轴（**触发自主性**）：OpenClaw/Hermes（低，cron + 用户定义触发）— Decision Agent 治理边界内（中，全治理）— ProAct（高，无治理）。Decision Agent 的差异化因此是：在"触发自主性"上不追求 ProAct 的极端，在"治理深度"上超越三者——用 ProAct 预测机制 + OpenClaw 显式触发约束 + [18]/[19] 治理底座组合 [20: §2]。证据强度 [med]——benchmark + 双消融 + bootstrap CI + 公开基线对比，但 closed-world 合成 benchmark + 同源 judge + 跨评测换模型（proactivity gpt-4o / memory Qwen2.5-7B）+ gating 全为手工常数 + 自主 intent 无治理出口，作为机制骨架与评测协议参考足够，作为"企业场景净收益"证据不充分。

---

## Goal 3 — Shared Workspace

**黑板广播与自选择** [9]

Blackboard 架构 [9] 将 MAS 通信从点对点重构为广播-自选择：所有 worker 读取公共黑板，仅在自评置信度超过阈值时写入。这解决了上下文溢出边界（第一正当性条件），同时避免了中央调度器成为单点瓶颈。

关键结果：在 Kaggle 数据科学任务上，Blackboard MAS 相比基线提升 31.2%；自选择机制使冗余写入减少 67% [9: §4.3]。

**跨会话访问控制层** [10]

[10] 在 Blackboard 之上引入正式的访问控制层：二部图 G=(U∪M, E) 形式化用户-记忆权限关系，三级访问模型（私有/共享/公开）与版本化记忆片段。这是 goal 3 的"共享" 语义的形式化——不是所有 worker 都能读写所有状态。

关键结果：动态访问控制使多用户协作场景中的记忆污染率降低 73%；版本化记忆在 session 切换后保持 94.7% 的相关性 [10: §5]。

**Role-specific 注入与 MAS 跨轨迹自演进：G-Memory 的第三层证据** [16]

G-Memory [16] 为 goal 3 的 Shared Workspace 增加了第三个维度：跨轨迹（cross-trial）自演进。三层架构：

- **Interaction Graph**：原始多智能体对话轨迹，节点为消息，边为依赖关系
- **Query Graph**：从 Interaction Graph 蒸馏的语义查询索引，按角色过滤
- **Insight Graph**：LLM 生成的高层洞见，跨任务实例积累

注入算子 Φ：在每次任务启动时，根据 worker 角色从三层图中检索相关片段，注入对应 worker 的上下文。这是 role-specific 而非广播——与 [9] 的黑板广播形成设计分叉。

关键结果：ALFWorld +20.89%，HotpotQA +10.12%；消融实验中去掉 Insight Graph 单层性能下降 3.95%，证明三层各有独立贡献 [16: §4]。**关键发现**：1-hop 检索（直接邻居）最优；2-hop 开始引入噪音，性能下降。朴素将单智能体记忆机制迁移到 MAS 不仅无益，甚至有害（跨 agent 角色边界的记忆污染）[16: §4.3]。

**Goal 3 设计决策**

| 机制 | 来源 | 适用场景 | 治理风险 |
|------|------|----------|----------|
| 黑板广播 [9] | Blackboard | 上下文溢出，高吞吐 | 低（写入有阈值门控） |
| 访问控制层 [10] | Collaborative Memory | 多用户/多组织边界 | 低（正式化 ACL） |
| 跨轨迹自演进 [16] | G-Memory | 重复性任务，角色固定 | 高（LLM 写入 Insight Graph） |

三层可堆叠：[9] 负责实时广播，[10] 负责跨会话访问控制，[16] 负责跨任务知识积累。但 [16] 的 Insight Graph 写入由 LLM 生成，与 TraceAI 显式来源追踪要求存在冲突——需要审计钩子（audit hook）包裹每次写入。

**失败诊断视角** [6]

[6] 对 MAS 失败模式的分类为 goal 3 提供了边界条件：MAS 失败中 38.7% 源于跨 agent 上下文不同步，27.4% 源于工具调用错误传播 [6: §3]。Shared Workspace 直接针对前者，但须防止共享状态成为错误传播的放大器（对应 goal 3 的访问控制需求）。

**PAGRL escalation 三条触发作为 β/β_r 路由形式化判据** [19]

thesis contradiction 1 决议：内部信息处理走 β_r 私有 + TraceAI 镜像；涉及外部副作用必须直接进 β + HITL。[19] §4.4 escalation 三条触发与该分轨直接吻合：(1) "类别性禁止" ≈ 外部副作用类型本身被 Layer 1 Global 禁止；(2) "不可逆" ≈ 任何外部副作用动作（数据库写 / 邮件发送 / API 调用）的不可逆性 → 必经人审；(3) "真实模糊" ≈ 边界情形（不确定是否为外部副作用）→ 升级。Decision Agent 落地 Shared Workspace 时，可把 PAGRL 的 escalation 三条作为 β/β_r 路由的形式化判据：每个 helper 响应在写入前经 PAGRL；若 ESCALATE 则进 β + HITL，否则进 β_r 私有 [19: §4.4 vs thesis contradiction 1]。

---

## Goal 4 — Composer

**BKN 语义解耦：四条证据线**

Goal 4 Composer 的核心机制是 BKN（Business-Knowledge-Node）语义解耦：将工具/agent 的业务意图（B）、知识结构（K）与执行节点（N）分层表示，解耦检索（B+K 层）与执行（N 层）。四条独立证据线支持这个设计：

| 证据 | 来源 | 贡献 |
|------|------|------|
| Agent-as-a-Graph 目录层 | [5] | KG 结构化工具/agent 关系，BKN 的图骨架 |
| MCP-Zero 主动发现 | [7] | 按需发现替代预加载，BKN 的动态扩展机制 |
| ToolRet-train 专用训练 | [11] | 通用 embedding 在 5k+ 规模 NDCG@10 ≤ 42%，BKN 需专用 IR 模型 |
| ToolOmni RL 查询改写 | [13] | RL 训练的查询改写器将 Recall@10 从 54% 提升至 71% [13: §4] |

**规模临界点**：通用 embedding 在工具数量 ≤ 500 时性能可接受（NDCG@10 ~65%），但在 5k+ 规模骤降至 ≤ 42% [11: §4.2]。这是论题中最清晰的工程阈值：5k 工具是引入专用 IR 模型的硬触发条件。

**Agent-as-a-Graph 目录层** [5]

[5] 将工具和 agent 建模为知识图谱节点，工具间语义关系（依赖、替代、组合）建模为边。检索时走图路径而非向量近邻，在组合工具场景下 Recall@10 比向量基线高 18.3% [5: §4]。这是 BKN 中 K 层（知识结构）的具体实现。

**MCP-Zero 主动发现** [7]

MCP-Zero [7] 将工具加载从"启动时预注入"改为"运行时按需发现"：agent 在执行中遇到能力缺口时，主动查询 MCP 注册表并动态注入工具描述。在 1000+ 工具环境中，MCP-Zero 的 Recall@5 比预加载基线高 23.1%，同时 prompt 长度减少 67% [7: §5]。

这是 BKN 中 N 层（执行节点）的动态扩展机制：Composer 不需要在设计时枚举所有工具，而是依赖 MCP-Zero 在运行时补全。

**Knapsack 组合优化** [12]

[12] 将 agent/工具的组合选择形式化为 0-1 背包问题：给定能力预算（token、延迟、API 成本），最大化任务覆盖率。ZCL (Zero-Cost Localization) 通过 design-time sandbox 预测每个 agent 组合的实际性能，避免运行时试错。

关键结果：在 AgentBench 上，Knapsack 选择的 agent 组合比随机基线高 22.1%，比贪心选择高 8.7% [12: §4]。沙盒验证的代价：design-time 成本是 baseline 的 3.2×，但消除了运行时失败的尾部风险。

**图路由对 Composer 的含义** [2]

[2] 的拓扑配置能力是 Composer 的必要扩展：选定 agent 组合后，还需要选定通信拓扑。层次图（Supervisor + Workers）在任务规模 ≤ 8 个 worker 时最优；完全图在创意发散任务中优于层次图 [2: §4.2]。Composer 的输出应包含 (agent 集合, 拓扑类型, 连接权重) 三元组。

**Blackboard 与 Composer 的交互** [9]

当 Composer 选定 agent 组合超过上下文溢出边界时，自动触发 Blackboard 通信协议。这是两个子系统的接口：Composer 负责 what（选哪些 agent），Blackboard 负责 how（如何共享状态）。

**Governance routing 作为 ContextLoader 延伸** [18]

[18] 把 ContextLoader "工具/BKN 节点按需加载"思想延伸到"组织政策 / 合规规则 / 模板 / 指南"维度——其分级治理路由（fast ~850ms / full ~2-5s）为 ContextLoader 提供了一个**统一的"治理-工具-业务对象"三类候选共享一套召回基础设施**的工作点参考：fast 模式（embedding sim + keyword overlap + always-on boost 加权，无 LLM）跑高频低延迟路径；full 模式（embedding 预过滤 + LLM 4 步结构化分析）跑低频高复杂度路径。Progressive context delivery 在 5 步混合域 workflow 上实现 50.3% token 节省（再入相同治理域 50-90%，跨入新域 0%）[18: §F E4]——为 Decision Agent 在 Heartbeat + Cron + Composer 多步路径上的 token 累计风险提供可直接复用策略。但 92% precision / 88% recall 是在 25 governance variables × 5 类别 × 20 任务的小规模上得出，5k+ governance variable + 100+ 任务类型规模下是否仍成立未验证。

---

## Goal 5 — 跨会话层级化 Memory

**四条独立证据线**

Goal 5 现获四条独立且互补的证据线，每条覆盖一个独立治理维度：用户/agent 访问控制（[10]）、MAS 跨轨迹自演进（[16]）、fragment 持久度治理（[17]）、组织级治理 + schema 生命周期（[18]）。

| 维度 | [10] Collaborative Memory | [16] G-Memory | [17] FadeMem | [18] Governed Memory |
|------|--------------------------|----------------|---------------|---------------------|
| 核心问题 | 谁能看到什么 | 如何从历史学习 | 什么应该被保留多久 | 组织政策 / schema 如何治理与演化 |
| 治理粒度 | 用户 × agent × resource | Agent × agent | Fragment × 时间 | 组织 × 实体 × governance variable |
| 时间范围 | 跨 session（用户级） | 跨 trial（任务实例级） | 跨 session 持续衰减 | 跨 session 组织级常驻 |
| 授权模型 | 正式化二部图 ACL | 无 | 无 | orgId 硬分区 + CRM-key 实体作用域 |
| 写入机制 | 用户/agent 显式写入 | LLM 自动生成 | LLM 仲裁冲突 + 融合 | 单次 LLM 抽取 → dual modality 双写 |
| 治理风险 | 低 | 高（黑盒 insight） | 中（冲突分类 acc 66.2%） | 中-低（schema 强约束 + provenance 七维） |
| TraceAI 友好度 | 高（显式 provenance） | 低（LLM 黑盒生成） | 高（衰减闭式可前向还原） | 高（七维 provenance + redaction 两阶段） |

**完整工程蓝图（v16）**：四篇论文在不同治理维度上互补、可叠加，构成 thesis goal 5 的完整候选架构。在同一系统中，[10] 处理"这个用户的记忆是私有还是共享、谁的 agent 能访问"；[16] 处理"过往任务里 Analyst agent 与 Reviewer agent 的协作 insight 应在第几步注入"；[17] 处理"用户三个月前的偏好片段是否仍应作为关键事实参与召回、与新偏好冲突时如何抑制"；[18] 处理"在组织 X 中，agent 调用任意 worker 时应注入哪些组织政策 / 合规规则 / 业务模板，schema 如何随业务演化"。四层职责不重叠但写入路径需协调：[17] 衰减状态须按 (fragment, user) 元组独立维护；[18] 的 (T, U, A, R) 须扩展为 (T, U, A, R, v, λ, β, layer, last_access, prune_reason, orgId, governance_variable_id, schema_version) 才能让访问控制 + 治理路由 + 衰减三个维度共存。

**MAS 跨轨迹自演进：G-Memory 的互补维度** [16]

G-Memory [16] 的独立贡献（在 Goal 3 之外）：Insight Graph 层在跨任务实例积累中单独贡献 3.95% 的性能提升（消融结果）[16: §4.3]。这证明三层并非冗余——Query Graph 负责结构化索引，Insight Graph 负责跨实例的语义抽象。

**1-hop 最优性约束**：G-Memory 的关键工程发现是 1-hop 检索最优，2-hop 引入噪音。这对 goal 5 的实现有直接约束：memory 检索深度须在配置层固定为 1-hop，不可动态扩展。

**重要性调制衰减与冲突仲裁：FadeMem 的填空** [17]

[17] 是 corpus 中**首篇**把"主动遗忘"作为头号设计目标且配套形式化公式 + 消融的论文，正面填补 v14 报告中标记为"文献空白"的 Q5c（衰减）/Q5d（遗忘）/Q5e（冲突合并）三个工程问题：

- **衰减形式**：v_i(t) = v_i(0)·exp(−λ_i·(t−τ_i)^β_i)，λ_i = λ_base·exp(−µ·I_i)；LML 半衰期≈11.25 天 / SML≈5.02 天；重要性 I_i = α·rel + β·f/(1+f) + γ·recency 综合语义相关性 / 访问频率 / 时间新近性 [17: §2.1–2.2]。
- **遗忘**：强度低于 ε_prune 或休眠超 T_max 天的 fragment 自动剪枝；θ_promote=0.7 / θ_demote=0.3 滞后阈值控制 LML/SML 层迁移防振荡。
- **冲突合并**：LLM 4 类分类（compatible/contradictory/subsumes/subsumed）+ contradictory 用 v_i ← v_i·exp(−ρ·age_diff) 抑制旧记忆；融合按语义+时间双约束聚簇 + 多样性奖金 + LLM 验证保留度。

关键结果：合成 LTI-Bench 30 天上以 55.0% 存储保留 82.1% 关键事实，超过所有 100% 存储基线；LoCoMo 多跳 F1=29.43、SRR=0.45（其他基线 0.00–0.15）[17: §3.2–3.4]。消融贡献顺序：fusion (−53.7%) > LML/SML (−33.9%) > conflict (−22.4%)。

**关键工程边界**：(a) 评测全部合成或小规模（30 天窗口对长期遗忘支撑不足、LTI-Bench 构造细节零披露）；(b) ≥13 个旋钮，重要性权重 α/β/γ 数值未公开，迁移到 BKN 工具集需重做 grid search；(c) 写入路径每条新记忆同步 LLM 调用，5k–10k 工具 + 数百并发用户场景下吞吐瓶颈未量化；(d) contradictory 误分率 33.8%——把 contradictory 错判为 compatible 让旧错误信息共存沉积，企业合规场景需"BKN-anchored 规则 + LLM 兜底"分轨；(e) 单 agent 单用户假设，多租户独立衰减完全空白。

**[1] 的证据关系**：Co-Evolving [1] 的技能库演化（SkillBank + EvolutionModule）与 [16] 在机制上相似（跨任务积累），但 [1] 针对单 agent 技能，[16] 针对 MAS 协作模式。两者可作为 goal 5 "演化"机制的不同粒度：[1] 演化工具调用序列，[16] 演化 agent 间协作策略。FadeMem 的访问频率项 β·f/(1+f) 也可作为 [1] 技能库的剪枝候选规则，但 [1] 论文未提此问题。

**ProAct memory-gap augmentation：双向 lifecycle 的主动写入端** [20]

[17] FadeMem 提供"哪些记忆该衰减剔除"（lifecycle 的减项）。[20] ProAct 的 memory-gap augmentation 提供互补的另一端：memory 维护识别 stale / incomplete / weakly-supported / missing 知识时，把这些 gap 转为候选未来需求加入 acquisition 目标（lifecycle 的增项）[20: §3.3]。两者合起来构成 thesis goal 5"记忆不无限膨胀 + 经验继承"所需的**双向 lifecycle**——衰减剔除 + 主动补全。ProAct 的增量抽取管线（profile_updates / updated_summary / key_info / user_sentiment / extracted_facts，Appendix J）为 Decision Agent build_memory/search_memory 基础链路提供字段级参考。但 ProAct 是**扁平 User 级记忆**——无 Role/Org 层级、无 [10] Collaborative Memory 的多用户访问控制，其机制须被 thesis 的层级化 + 治理 schema 包裹后才可用。

**组织级治理 + schema lifecycle：Governed Memory 的填空** [18]

[18] 是 corpus 中**首篇产品级架构原型**把"组织级治理基础设施"作为独立层置于记忆原语之上，覆盖 [10]/[16]/[17] 集体未触及的 Org 维度：

- **dual modality 存储**：open-set memory（去共指原子事实，三类启发式 quality gate + 0.92/0.95 余弦阈值去重）+ schema-enforced memory（按组织 schema 类型化 property value with confidence），由**单次 LLM 抽取**同时产出。互补性证据：both 34% / open-set only 38% / schema only 12% / both miss 16%——schema-only 系统永久丢失 38% 长尾事实 [18: §8.4]。
- **分级治理路由**：fast 模式（无 LLM call，~850ms，embedding sim + keyword overlap + always-on boost 加权）/ full 模式（embedding 预过滤 + LLM 4 步结构化分析，~2-5s）；fast 92% precision / 88% recall（25 variables × 5 类 × 20 任务），well-authored vs poorly-authored variable 发现率差 20-50pp [18: §5.1-5.2, §8.5]。
- **progressive context delivery**：session state 跟踪已交付 governance variable 与 section，后续步只注入 delta；5 步混合域 workflow 50.3% token 节省（再入相同治理域 50-90%，跨入新域 0%）[18: §5.3, §F]。
- **schema lifecycle**：AI 辅助创建 → 自然语言交互增强 → trace-grounded rubric 评分 → 自动 per-property 三阶段细化 [18: §7]。
- **MemoryEntry 数据模型**：orgId（组织硬分区）+ recordId（实体作用域，CRM-key）+ 七维 provenance（contentHash SHA-256 / contentLength / extractionMethod / llmModel / chunkIndex / chunkTotal / redactionApplied / timestamp）+ 两阶段 PII redaction [18: §A, §B]。

**与 thesis 治理优先核心定位高度同构**：governance variable + schema lifecycle ≈ BKN 业务护栏 + 自演化、provenance 七维元组 ≈ TraceAI 审计字段、redaction 两阶段 ≈ ISF 安全边界、progressive context delivery ≈ ContextLoader 按需加载——但 [18] 治理粒度是组织级 governance variable（"政策"作为字符串/section），thesis 粒度是业务对象级 BKN schema（结构化语义网络）；[18] 是 thesis 治理愿景的**简化/弱化版工程实证**。

**关键工程边界**：(a) 核心数字几乎全部基于 ≤250 合成样本，arXiv v1 单作者 + 单组织自报告生产部署但无任何生产侧 metric；(b) **multi-agent concurrent write conflict 被作者自陈未解决**（§9.1），与论文标题"Production Architecture for Multi-Agent Workflows"直接冲突；(c) "四层独立可配置"是设计声明，无独立性消融；(d) entity 隔离 zero leakage 完全靠 CRM-key 预过滤而非 embedding distinctiveness——上游漏标 CRM-key 即失效；(e) "约 7 条记忆饱和"是 N=30 + 12 条 84.4 < 7 条 88.0 的非单调拟合，不应抬升为产品论断；(f) schema lifecycle 自动 per-property 优化仅对内部团队和早期访问开放。

**剩余工程空白（v16）**

[10]+[16]+[17]+[18] 四层堆叠后，多个 v15 工程债务被部分填补，但合成路径上的工程债务转移到四处新边界：

1. **multi-agent concurrent write conflict（goal 3+5 合并区无解）**：[18] §9.1 自陈未解决——FadeMem [17] 单 agent / Collaborative Memory [10] 按用户分区 / [18] 按组织分区，**没有任何已读论文系统性回答"多 agent 同时写同一 fragment"**。Decision Agent 候选解法是 contradiction 1 (Option C 改良版) 的延伸：内部信息处理 fragment 允许 last-writer-wins + TraceAI 镜像；外部副作用 fragment 必须串行化 + Human-in-the-loop。这是 v16 唯一全新升级为头号工程债务的项。
2. **多租户独立衰减**：[17] 单用户假设让"跨租户 fragment 衰减是否独立维护"完全空白——衰减状态须按 (fragment, user) 元组独立维护，并与 [18] orgId 硬分区在数据模型层组合。
3. **LLM 仲裁的合规安全**：[17] 冲突分类 acc 66.2% / [10] redact 转写无失败率评测 / [18] redaction 基于 Presidio 风格正则承认漏 obfuscated 模式——三者共享同一安全弱点，企业合规场景必须"BKN schema 规则前置 + LLM 兜底"分轨。
4. **写入路径吞吐**：[17] 同步 LLM 写入 + [18] 单次 LLM 抽取双写 dual modality 与 thesis "7×24 数字员工"目标存在工程张力，需异步化（先入库 + 后台合并）+ BKN 规则前置过滤的工程评测。

四处工程债务中，#1 是 v16 新揭露的根本性研究空白，#2-#4 已从"机制空白"降级为"工程参数化与跨论文合成扩展"。

---

## 横切面 — 治理基础设施（ISF + TraceAI）

随 [18] Governed Memory 与 [19] PAGRL 同期进入 corpus，thesis 治理优先核心定位首次拥有跨 design-goal 的参考架构骨架——两篇为同作者群（Bandara / Gore / Shetty / Mukkamala / Hass / Rahman / Liang 等核心成员重合）同期产出的**配套**论文，使用近乎相同的 MCP 基础设施模板（rule store + append-only trace log + provenance + prompt-side 注入 + 4 层架构）。覆盖维度互补：[18] 治理"agent 应该看到什么记忆 + 哪些组织政策"（**记忆层治理**），[19] 治理"agent 在做出动作前应当如何 deliberate"（**动作层治理**）。

**与 thesis 四支柱的对应**：

| thesis 支柱 | [18] Governed Memory 对应 | [19] PAGRL 对应 |
|------------|--------------------------|-----------------|
| BKN | governance variable + schema lifecycle ≈ BKN 业务护栏 + 自演化（粒度更弱：组织级 string/section vs 业务对象级语义网络） | 4 层规则（Global/Workflow/Agent/Situational）≈ BKN 治理规则的层级骨架 |
| ISF | redaction 两阶段 ≈ ISF 安全边界 | PAGRL 4 阶段 + 3 条 escalation 触发 ≈ ISF 在每次动作前的强制检查点 |
| TraceAI | provenance 七维元组（contentHash / extractionMethod / llmModel / chunkIndex / redactionApplied / timestamp 等） | reasoning trace（rules retrieved / permissibility reasoning / decision），值得直接采纳为 trace schema |
| ContextLoader | progressive context delivery（5 步 50.3% token 节省，再入相同治理域 50–90%）| governance prompt constructor（按 4 层检索适用规则注入 system prompt） |

**共同弱点**：(a) 两者都把"production-grade"作为信用基础但都用作者团队自家系统作为案例（Personize.ai / Flowr），均缺第三方验证；(b) 都依赖 LLM 概率输出而非确定性强制——[18] 单次 LLM 抽取生成 dual modality / [19] 同一 LLM 既做动作发起者又做合规裁判，"explainability"被等同于"auditability"；(c) **multi-agent concurrent write conflict** 被 [18] §9.1 自陈未解决，[19] 完全未触及——是 goal 3+5 合并区无任何已读论文系统性回答的工程空白；(d) 规模差距严重——[19] 7 条规则 / [18] 25 governance variable，与企业 50–100+ 规则 / 5k+ governance variable 目标差 1–2 数量级。

**与 AgentSpec 类外部强制路径的角色分工**：[19] §7.1 显式承认 AgentSpec 在确定性保护强度上更高（90%+ 不安全代码拦截 / 毫秒级延迟），但批评其"在 agent 推理之外起作用"。Decision Agent ISF 设计应同时实现两层：[19]/[18] 在前做 deliberation 与 trace（柔性、可解释），AgentSpec 类外部规则引擎在后做兜底强制（硬性、确定性）——[19] §6.3 自身建议这种叠加但未实现，PAGRL 边际价值在外部强制兜底已存在的架构下需重新评估。

---

## 协同失真量化

**MAS 失败模式地图** [6]

[6] 对 MAS 失败进行了首个系统性分类，覆盖 14 个框架（AutoGen、CrewAI 等）× 200+ 任务。主要失败模式：

| 失败类型 | 占比 | 根因 |
|----------|------|------|
| 跨 agent 上下文不同步 | 38.7% | 无共享状态机制 |
| 工具调用错误传播 | 27.4% | 无错误隔离边界 |
| 角色边界模糊 | 19.2% | Agent 描述不精确 |
| 死锁/循环依赖 | 14.7% | 无拓扑验证 |

对论题的含义：goal 3 (Shared Workspace) 直接覆盖 38.7%；goal 4 (Composer) 的 Knapsack 沙盒验证覆盖 14.7%；工具调用错误传播（27.4%）在当前设计目标中没有直接对应——这是 goal 4 的未覆盖区域。

**[6] 的局限**：[6] 的样本以短任务（≤ 10 步）为主，长时序任务（> 50 步）的失败分布未测量 [6: Weaknesses]。论题的目标场景（长时任务执行）可能有不同的失败模式分布。

---

## 可证伪点追踪

| ID | 声明 | 证据强度 | 可证伪条件 |
|----|------|----------|------------|
| F1 | OneFlow 折叠等价成立（同构 MAS = 单 LLM 顺序） | [high] [4] | 同构 MAS 在同等资源下系统性优于单 LLM |
| F2 | 上下文溢出是 MAS 的第一正当性条件 | [high] [9] | 存在无上下文溢出但 MAS 优势显著的任务类 |
| F3 | 监督委派容量是 MAS 的第二正当性条件 | [med] [12] | Knapsack 选择在 ≤ 4 worker 组合中无优势 |
| F4 | 步进式推理在 > k 步序列任务系统性失败 | [high] [3] | FLARE 等价的前瞻机制在开放式工具调用中无优势 |
| F5 | BKN 语义解耦在 ≤ 500 工具规模可用通用 embedding | [high] [11] | 通用 embedding 在 500 工具规模 NDCG@10 < 50% |
| F6 | 5k+ 工具规模须专用 IR 训练 | [high] [11][13] | ToolRet-train 或 ToolOmni 在 5k+ 规模 Recall@10 ≥ 50% 不需要专用训练 |
| F7 | Blackboard 自选择机制减少冗余写入 | [med] [9] | 自选择阈值调优失败导致写入率 > 50% |
| F8 | G-Memory 1-hop 最优性成立 | [med] [16] | 1-hop 在企业异构任务分布下 Recall 不显著优于 2-hop |
| F9 | LLM 生成 Insight Graph 写入失败率 ≤ 1% | [low] [16] | LLM 在高信息密度轨迹中生成矛盾洞见率 > 5% |
| F10 | [10]+[16] 堆叠无写入冲突 | [low] 推断 | 两层记忆系统在并发写入时产生不可合并冲突 |
| F11 | FadeMem 重要性公式 I_i 在 BKN 工具集上关键事实保留率显著高于纯 recency 基线 | [low] [17] | α/β/γ 未公开，BKN 工具集复测显示 I_i 不优于 FIFO |
| F12 | FadeMem 双层 LML/SML 在企业 1 年级时序上仍保留关键事实 | [low] [17] | 30 天窗口外推失败，长时序下 LML 半衰期假设崩坏 |
| F13 | FadeMem 写入路径同步 LLM 调用在 5k–10k 工具 / 数百并发用户下可承受 | [low] 推断 | 写入吞吐 < 业务可接受阈值，必须异步化 + 规则前置 |
| F14 | FadeMem contradictory 33.8% 误分率在企业合规场景仍可接受 | [low] [17] | 1/3 错判为 compatible 沉积错误信息，触发治理事件 |
| F15 | "约 7 条 governed memory 饱和"在 BKN 工具集 + 企业实体记忆下仍是工作点 | [low] [18] | N=30 噪声拟合（12 条 84.4 < 7 条 88.0），自家数据复测显示曲线持续上升 |
| F16 | Governance routing 92% precision / 88% recall 在 5k+ governance variable 规模下仍成立 | [low] [18] | 25 vs 20 远低于企业级；规模化后 well-authored vs poorly-authored 0% 发现率扩散到主流类别 |
| F17 | Schema-enforced 38% 长尾事实空白可被 BKN schema 自身吸收（BKN 不需 open-set 模态） | [low] 推断 | BKN schema 实测仍漏 ≥30% 业务事实，open-set 模态在 Decision Agent 不可省 |
| F18 | Multi-agent concurrent write conflict 可用"内部 vs 外部副作用分轨 + Human-in-the-loop"处理 | [low] 推断 / contradiction 1 延伸 | 多 agent 写入同一外部副作用 fragment 时分轨方案在治理事件率与吞吐两维度都不优于"全部串行化 + HITL" |
| F19 | PAGRL ESCALATE 三条触发能在 Decision Agent 真实业务长链任务中覆盖 ≥90% 应升级动作（漏报率 ≤10%）| [low] [19] | ≥1 起治理事件由 PAGRL 未触发 ESCALATE 的动作引发——agent 把 consequential 动作拆为多个被自归类为"非 consequential"子步即可绕过 |
| F20 | PAGRL 全量规则注入路径在 50+ 规则规模下延迟与 token 开销仍 < 1s/动作 + < 20% context budget | [low] [19] | 7 条规则下 0.65s/动作；线性外推 50 条达 ~5s——若实测如此，必须改 retrieval-based 规则注入 |
| F21 | PAGRL "内化 normative generalization" 优势在 BKN 规则集 + 真实对抗 benchmark（HarmBench）上重现 | [low] [19] | 仅 N=1 的 Scenario 3 支撑该核心差异化主张；对抗 benchmark 上 PAGRL 不优于 AgentSpec 类外部强制 |
| F22 | [19] PAGRL + [18] Governed Memory + [10] Collaborative Memory 三层叠加在统一 ISF/TraceAI 实现中保持架构一致 | [low] 推断 | 三套 trace schema / 三套 governance variable 不能融合到统一 BKN 表达——则"治理统一基础设施"论点需重新评估 |
| F23 | 主动需求预测不是 cron + ReAct 可平凡解决的（"学术稀缺 = 机会窗口"被反向修订）| [med] [20] | Undirected Idle（≈ 朴素 cron + ReAct 背景探索）花 69.8k token 仅得 ~1% 增益 vs Directed 14.8%——若复测发现朴素 baseline 已满足 ≥50% 真实主动需求，则"预测方向"非必要 |
| F24 | ProAct 式 idle 预测在 Decision Agent 真实（非合成、无 predictable_after 标注）业务对话上 Anticipation Recall 显著 > 0 | [low] [20] | 真实对话可预测性远低于 ProActEval 构造，Recall 接近 Undirected Idle 水平——则 111.8k token/场景成本无法在企业场景被价值证成 |
| F25 | 治理边界内"有限主动预测"（store-only + 显式调度来源 + prediction-guided）能保留 ProAct 多数增益 | [low] 推断 / [20] | 去掉 push 自主性后 User Effort 改善 < Directed Idle 的一半——则"治理边界 vs 主动收益"存在不可调和取舍，goal 2 需重估 |

---

## 工程问题地图（v18）

```
Decision Agent 工程问题地图
═══════════════════════════════════════════════════════

LAYER 0: 架构前提
  ┌─────────────────────────────────────────────────┐
  │ Q0a: 何时引入 MAS？                              │
  │   → 上下文溢出边界 [9] ✓ 有答案                │
  │   → 监督委派容量边界 [12] ✓ 有答案              │
  │ Q0b: 拓扑如何配置？                              │
  │   → 显式拓扑层 [2] ✓ 有原型                     │
  └─────────────────────────────────────────────────┘

LAYER 1: Goal 1 — 开箱即用
  ┌─────────────────────────────────────────────────┐
  │ Q1a: 默认 worker 规划机制？                      │
  │   → 步进式推理结构性局限 [3] ✓ 已量化           │
  │   → 最小前瞻机制规格 ✗ 未指定                   │
  │ Q1b: 开箱即用适用任务范围？                      │
  │   → 从图遍历到工具调用的距离 ✗ 未测量           │
  └─────────────────────────────────────────────────┘

LAYER 2: Goal 2 — Heartbeat + Cron
  ┌─────────────────────────────────────────────────┐
  │ Q2a: ISF 结构性张力如何解决？                    │
  │   → Cron 触发须有显式来源 ✓ 设计决策            │
  │   → PAGRL ESCALATE 通路 [19] ✓ 工程模板         │
  │   → {push/queue/store} 三态治理分轨 [20] ✓ 候选  │
  │   → Tick 上限频率规格 ✗ 未量化                  │
  │ Q2b: Heartbeat 与长时任务检查点如何对齐？         │
  │   → 未测量 ✗                                     │
  │ Q2c: intrinsic goal 进入生产路径的拦截？         │
  │   → PAGRL Stage 3 识别 + ESCALATE [19] ✓ 候选   │
  │   → ProAct"自主推断"须改 cron/授权派生 [20] ✓   │
  │ Q2d: 主动需求预测机制（被严格评测）？            │
  │   → ProAct Future-State Prediction +            │
  │     value gate + 双消融 [20] ✓ 有原型           │
  │   → 真实业务对话 Recall / token 成本 ✗ 未验证   │
  └─────────────────────────────────────────────────┘

LAYER 3: Goal 3 — Shared Workspace
  ┌─────────────────────────────────────────────────┐
  │ Q3a: 黑板写入阈值如何校准？                      │
  │   → 自选择机制 [9] ✓ 有机制，无校准规格          │
  │ Q3b: 访问控制粒度？                              │
  │   → 三级模型 [10] ✓ 有形式化                    │
  │ Q3c: Insight Graph 写入审计？                    │
  │   → TraceAI 钩子需求 ✗ 未实现                   │
  │ Q3d: 跨 agent 错误传播隔离？                     │
  │   → 27.4% 失败来源 [6] ✗ 无覆盖                 │
  │ Q3e: β/β_r 路由形式化判据？                      │
  │   → PAGRL 3 条 escalation 触发 [19] ✓ 候选       │
  └─────────────────────────────────────────────────┘

LAYER 4: Goal 4 — Composer
  ┌─────────────────────────────────────────────────┐
  │ Q4a: 工具检索规模临界点？                        │
  │   → 5k 工具是硬触发 [11] ✓ 有阈值               │
  │ Q4b: 查询改写器训练数据？                        │
  │   → ToolOmni RL [13] ✓ 有原型，无企业数据        │
  │ Q4c: Knapsack 沙盒 design-time 成本？             │
  │   → 3.2× baseline [12] ✓ 已量化                  │
  │ Q4d: 拓扑选择规则？                              │
  │   → 层次 vs 完全图准则 [2] ✓ 有启发式            │
  │ Q4e: 工具调用错误传播隔离？                      │
  │   → 27.4% 失败 [6] ✗ Composer 未覆盖            │
  └─────────────────────────────────────────────────┘

LAYER 5: Goal 5 — 跨会话层级化 Memory
  ┌─────────────────────────────────────────────────┐
  │ Q5a: 用户级跨会话记忆？                          │
  │   → 访问控制形式化 [10] ✓ 有原型                │
  │ Q5b: MAS 级跨任务知识积累？                      │
  │   → G-Memory 三层架构 [16] ✓ 有原型              │
  │ Q5c: 记忆衰减机制？                              │
  │   → FadeMem 指数衰减 + 重要性调制 [17] ✓ 有原型 │
  │ Q5d: 主动遗忘机制？                              │
  │   → ε_prune + T_max 休眠剪枝 [17] ✓ 有原型      │
  │ Q5e: 冲突合并机制？                              │
  │   → 4 类 LLM 仲裁 + 年龄差抑制 [17] ✓ 有原型     │
  │       但 acc 仅 66.2%，需 BKN 规则兜底           │
  │ Q5f: Insight Graph 写入与 TraceAI 的兼容性？     │
  │   → 治理空白 ✗ 未解决                            │
  │ Q5g: 多租户独立衰减？                            │
  │   → [17] 单用户假设留空白 ✗ 工程债务             │
  │   → [18] orgId 硬分区可组合 ✓ 工程模板           │
  │ Q5h: 写入路径同步 LLM 调用吞吐？                 │
  │   → [17]+[18] 未量化 ✗ 工程债务                  │
  │ Q5i: 组织级治理 + schema lifecycle？             │
  │   → [18] 四层架构 ✓ 有原型                       │
  │   → 5k+ governance variable 规模未验证 ✗         │
  │ Q5j: Multi-agent concurrent write conflict？     │
  │   → [18] §9.1 自陈未解决 ✗ 头号研究空白          │
  │   → 候选: contradiction 1 Option C 延伸          │
  └─────────────────────────────────────────────────┘

LAYER 6: 横切面 — 治理基础设施 (ISF + TraceAI)
  ┌─────────────────────────────────────────────────┐
  │ Q6a: 动作层治理？                                │
  │   → PAGRL 4 阶段 + 4 层规则 [19] ✓ 架构骨架     │
  │   → 与 AgentSpec 叠加边际价值 ✗ 未实现           │
  │ Q6b: 记忆层治理？                                │
  │   → [18] dual modality + 分级路由 ✓ 架构骨架    │
  │ Q6c: trace schema 统一？                         │
  │   → [19] reasoning trace + [18] 七维 provenance │
  │     ✗ 三层叠加（+[10]）一致性未验证 (F22)       │
  │ Q6d: 内化 deliberation vs 外部确定性强制？        │
  │   → 必须双层叠加 [19 §6.3 自身建议] ✗ 未实现    │
  └─────────────────────────────────────────────────┘
```

---

## 版本更新日志

| 版本 | 新增内容 | 变更类型 |
|------|----------|----------|
| v1 | [01] Co-Evolving 技能库演化 | 初始报告 |
| v2 | [02] Graph-of-Agents 拓扑配置 | 架构扩展 |
| v3 | [03] FLARE 规划机制分析 | Goal 1 证据 |
| v4 | [04] 强单智能体基线 + OneFlow 折叠等价 | 架构前提 |
| v5 | [05] Agent-as-a-Graph 目录层 | Goal 4 证据 |
| v6 | [06] MAS 失败模式量化 | 协同失真量化 |
| v7 | [07] MCP-Zero 主动发现 | Goal 4 证据 |
| v8 | [08] Heartbeat 调度 + ISF 张力分析 | Goal 2 证据 |
| v9 | [09] Blackboard 广播自选择 + OneFlow 第一边界 | Goal 3 + 架构前提 |
| v10 | [10] 访问控制形式化 + goal 5 第一层证据 | Goal 3 + Goal 5 |
| v11 | [11] ToolRet 专用训练 + 5k 阈值量化 | Goal 4 工程阈值 |
| v12 | [12] Knapsack 沙盒验证 + OneFlow 第二边界 | Goal 4 + 架构前提 |
| v13 | [13] ToolOmni RL 查询改写 + BKN 第四证据线 | Goal 4 证据 |
| v14 | [16] G-Memory 三层自演进 + goal 3/5 第三层证据；报告重构为论题驱动脊柱 | Goal 3 + Goal 5；Step B 重构 |
| v15 | [17] FadeMem 重要性调制衰减 + LLM 仲裁冲突 + 适应性融合；填补 v14 Q5c/Q5d/Q5e 三个文献空白；新增 F11–F14 falsifiability + Q5g/Q5h 工程债务 | Goal 5 第三维度证据 |
| v16 | [18] Governed Memory 组织级治理 + dual modality + 分级路由 + progressive delivery + schema lifecycle；goal 5 第四维度证据；Goal 4 governance routing 作为 ContextLoader 延伸；揭示 multi-agent concurrent write conflict 为 goal 3+5 合并区头号研究空白；新增 F15–F18 falsifiability + Q5i/Q5j 工程债务 | Goal 5 第四维度证据 + Goal 4 ContextLoader 扩展 |
| v17 | [19] PAGRL 4 阶段 deliberation + 4 层级联规则 + 3 条 escalation 触发；与 [18] 同作者群配套形成"动作层 + 记忆层"治理参考架构；新增"横切面 — 治理基础设施 (ISF + TraceAI)"独立段落；为 goal 2 governance-aware bounded autonomy 提供工程模板（Q2a/Q2c）；为 goal 3 β/β_r 路由提供形式化判据（Q3e）；揭示张力 C（内化 deliberation vs 外部确定性强制）必须双层叠加；新增 F19–F22 falsifiability + LAYER 6 治理基础设施工程问题地图 | 横切面治理基础设施 + Goal 2/3 工程模板 |
| v18 | [20] ProAct idle-time compute 主动预测 + value gate + 三态 delivery + ProActEval 双消融；goal 2 首份被严格评测的主动 Runtime（补齐 [8] HSC 空白）；揭示增益来自"预测方向"而非"后台搜索本身"，反向修订"学术稀缺=机会窗口"；定位 ProAct 为"未治理主动端"+ 治理改造路径（store-only + 显式调度 + push 经 PAGRL ESCALATE）；OpenClaw 边界增加触发自主性轴；goal 5 memory-gap augmentation 补全双向 lifecycle 主动写入端；新增 F23–F25 falsifiability + Q2a/Q2c/Q2d 工程地图更新 | Goal 2 主动 Runtime 证据 + Goal 5 lifecycle 补全 |
