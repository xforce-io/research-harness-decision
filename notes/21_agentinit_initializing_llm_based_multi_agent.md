---
paper: "AgentInit: Initializing LLM-based Multi-Agent Systems via Diversity and Expertise Orchestration for Effective and Efficient Collaboration"
arxiv: "2509.19236"
authors: ["Chunhao Tian", "Yutong Wang", "Xuebo Liu", "Zhexuan Wang", "Liang Ding", "Miao Zhang", "Min Zhang"]
year: 2025
venue: "arXiv preprint v1（HIT Shenzhen + University of Sydney，2025-09-23）"
note_number: 21
---

## Claims

- MAS 初始化阶段引入"无关或冗余 agent"是 task derailment 与 step repetition 的根因之一；在初始化阶段优化团队结构（精简冗余 agent）可改善整体性能 [21: §1, §3.1]。
- AgentInit 在 Complete Graph 框架下整体性能超过 SOTA 初始化方法（AutoAgents / EvoAgent）1.2 / 0.9 点、超过 pre-defined 策略 1.3 / 1.4 点（分别在 Qwen2.5-72B / DeepSeek-V3 上）[21: §4.2, Table 1]。
- 团队选择（Balanced Team Selection）是同时优化两个目标的多目标问题：task relevance（团队与查询的相关性）与 agent diversity（团队成员多样性），最优解取 Pareto optimal set 中由 Selector Agent 最终裁定的一个团队 [21: §3.3, Eq.(8)(12)]。
- 性能不随 relevance 或 diversity 单一目标单调上升——Pareto frontier 上得分最高的团队集中在"两目标平衡"的中段区域，极端偏向单一目标的配置反而更差 [21: §5.4, Figure 3]。
- 团队选择把团队内最大成对相似度 Max(s_ij) 平均降低 3.5%，验证选择过程确实削减了 agent 间冗余 [21: §5.5, Table 7]。
- 单目标消融：仅 relevance 或仅 diversity 都劣于联合优化；替代多样性度量 Div_avg（最小化平均成对相似度）比 relevance-only 高 0.6 点但仍劣于完整 AgentInit [21: §5.1, Table 4]。
- 选择策略消融：完全不选择（None，用全部候选 agent）相比默认配置掉 1.1 点；随机选同样数量 agent 也劣于 AgentInit——说明增益不能仅归因于"缩小团队规模"[21: §5.1 Selection Strategy block, Table 4]。
- NL-to-Format 标准化（Formatter Gf 把自然语言 agent 转 JSON）移除后整体掉 0.4 点 [21: §5.1, Table 4]。
- 迭代轮数 K=3 时性能最佳，K=5 仅差 0.1%，K=1（无 Observer）显著更差；迭代精炼有收益但边际递减 [21: §5.1, Table 4, §5.4]。
- AgentInit 在 Chain / Star / Layered / AutoGen 四种框架上均取得最佳整体性能，超 SOTA 最多 1.1 点、超 pre-defined 最多 1.2 点 [21: §4.2, Table 3, Appendix A.7 Table 12]。
- 强迁移性：用单条代表性查询或小批查询初始化（AgentInit(1,1) / (1,10) / (10,10)）的性能接近 per-query 完整初始化，且全部显著优于 MAS_none [21: §5.2, Table 5]。
- 在 AIME2025 上 AgentInit 超 vanilla baseline 12.3 点、超前 SOTA 3.4 点；在 Trivia Creative Writing（N=5 +5.0%）与 ScienceWorld（整体 26.59 vs AutoAgents 25.73）上同样取得最佳 [21: §A.4, Table 8/9/10]。
- 选择过程的计算开销可忽略：Nmax=5 时枚举全部组合 + 打分平均 GPU 时间 ~0.013s/题；Nmax=10 时枚举耗 8.47s/题；更大规模可用 NSGA-II（O(GNp²)，0.98s/题，覆盖 80% 全局 Pareto 前沿，GD=0.02）[21: §5.3, Table 6, Table 11]。

## Assumptions

- **agent description 的句向量余弦相似度是"团队协作有效性"的有效代理**——relevance 用 `cos(E(agent_desc), E(query))`、diversity 用 agent 描述向量构成相似度矩阵的 Vendi Score。两者都只度量**文本描述**，不度量 agent 的实际求解能力或运行时协作结果 [21: §3.3, Eq.(9)(10)(11)]。
- 候选 agent 池足够小，使得团队选择可**枚举所有规模在 [Nmin=1, Nmax=5] 内的组合**；这是 Pareto 精确求解可行的前提 [21: §3.3 Eq.(7), §5.3]。
- LLM 存在 self-preference bias、倾向不批评自身输出，因此需要一个**非 LLM 的统计选择层**（Pareto + 嵌入）来剔除冗余 agent——但最终仍由 Selector Agent Gs（LLM）从 Pareto 集中挑选 [21: §1, §3.3 Eq.(12)]。
- 任务主要是单轮 QA / 数学 / 代码（MMLU/GSM8K/HumanEval 等），ScienceWorld 是唯一交互式环境——隐含"团队结构在短链任务上即可体现初始化质量差异"[21: §4.1, §A.4]。
- 自动初始化流程的有效性"严重依赖高性能语言模型的能力"，并带来显著 token 开销——论文在 Limitations 显式承认这一前提 [21: Limitations]。
- 把 agent 生成与团队选择解耦后，"先生成标准化 agent 再选团队"不需要额外训练数据即可优于需训练数据的 AFlow/EvoFlow [21: §6.2]。

## Method

**核心论点：把 MAS 初始化拆成"标准化 agent 生成"+"多目标团队选择"两段，用 Pareto(relevance, diversity) 在 LLM 生成的候选 agent 上做子集筛选，避免 LLM self-preference bias 直接决定团队。** 形式化为 `f_init^AgentInit(q) ⊆ f_init^LLM(q)`——AgentInit 是在 LLM 生成结果上叠加的一个选择模块 [21: §2.2]。

**❶ Standardized Agent Generation（§3.2，红色模块）。** 三个 agent 协作迭代 K 轮（默认 K=3）：
- **Planner Gp = Gp2 ∘ Gp1**：Gp1 把查询 q（带上一轮反馈 φ_{t-1} 与上一轮子任务/agent）分解为子任务集 {τ_i^t}；Gp2 为每个子任务生成一个自然语言 agent 角色 A_i^t [21: Eq.(1)(2)(3)]。
- **Formatter Gf**：用 NL-to-Format（Tam et al. 2024）把自然语言 agent 转成标准化 JSON 表示 Â_i^t（name/description/suggestions/prompt 四字段）。**生成时不施加格式约束**（避免限制 Planner 表达力损害 agent 质量），仅在生成**后**标准化，以便后续公平比较 [21: Eq.(4)]。
- **Observer Go**：每轮末评审子任务分解与 agent 分配的合理性，产出反馈 φ_t；**最后一轮排除 Observer** [21: Eq.(5)(6)]。
- K 轮迭代后得到候选 agent 集 A_candidate。

**❷ Balanced Team Selection（§3.3，蓝色模块）。**
- **候选团队构造**：枚举 A_candidate 所有规模 r ∈ [Nmin=1, Nmax=5] 的子集，得团队集合 T [21: Eq.(7)]。
- **两个目标**：用预训练文本编码器 E（Sentence-BERT 或 all-MiniLM-L6-v2）编码 agent 描述与 query。
  - Relevance：团队内每个 agent 描述与 query 的平均余弦相似度 [21: Eq.(9)]。
  - Diversity：对团队成员描述向量构造相似度矩阵 S，取其特征值 λ_i 的 **Vendi Score** `exp(−Σ λ_i log λ_i)`（机器学习多样性度量）[21: Eq.(10)(11)]。
- **Pareto 最优集 T\***：取相对 relevance 与 diversity 双目标的非支配团队集（A1 支配 A2 当两目标均不劣且至少一个严格更优）[21: Eq.(8), Appendix A.1]。
- **最终选择**：Selector Agent Gs（LLM）从 T\* 中选一个最合适团队 A\* [21: Eq.(12)]。

**❸ 部署（§3.3，绿色模块）。** A\* 被注入不同 MAS 推理框架——Formally Modeled（Agentprune 的 chain/star/layered/complete graph）与 Loosely Modeled（AutoGen）——做协作推理。

**❹ 实验配置（§4.1）。** Backbone：Qwen2.5-72B-Instruct、DeepSeek-V3-671B。temperature=1（follow AgentDropout）。K=3，Nmin=1/Nmax=5，编码器 all-MiniLM-L6-v2，图框架推理 1 轮。

**❺ 关键消融（§5）。** 迭代轮数 K∈{1,3,5}；标准化 on/off；选择目标（Only Rel / Only Div / Rel+Div_avg / 默认 Rel+Div）；选择策略（None / Global Worst / Pareto Worst / Random / 默认 Pareto Best）；Nmax∈{5,10} 的效率与 token；单查询/小批迁移性。

## Eval

**Backbone：** Qwen2.5-72B-Instruct、DeepSeek-V3-671B-Instruct。

**Benchmark：** 主表用 MMLU（通用推理）、GSM8K/AQuA/MultiArith/SVAMP（数学）、HumanEval（代码）；复杂任务用 MATH/AIME2025（高阶数学）、Trivia Creative Writing（知识接地写作）、ScienceWorld（交互式科学推理）。

**框架：** Complete Graph（主表）+ Chain/Star/Layered（Agentprune 图结构）+ AutoGen。

**Baseline：** Vanilla、CoT、AgentPrune（完整 MAS baseline）、MAS_none（无角色分配）、Pre-defined（Agentprune 预置 agent）、EvoAgent、AutoAgents。

**Metric：** 任务准确率/pass@1；token 成本（Ptok 提示 token / Ctok 补全 token）；GPU 时间、迭代轮数、团队规模。

**主结果（Complete Graph 平均，Table 1）：**
| Backbone | MAS_none | Pre-defined | EvoAgent | AutoAgents | **AgentInit** |
|---|---|---|---|---|---|
| Qwen2.5-72B | 89.2 | 90.0 | 89.8 | 90.1 | **91.3** |
| DeepSeek-V3 | 91.5 | 92.3 | 92.5 | 92.8 | **93.7** |

**Token（Qwen, Complete Graph, Avg, Table 2）：** AgentInit Ptok 964K / Ctok 311K vs AutoAgents 1.0M / 323K vs Pre-defined 2.0M / 421K vs EvoAgent 1.9M / 687K——相对 EvoAgent/Pre-defined 显著省，相对 AutoAgents 基本持平。

**效率随 Nmax 放大（Table 11，MMLU）：** Nmax=5 时 AgentInit ≈ AutoAgents（706K/267K vs 704K/261K）；Nmax=10 时 AgentInit 959K/339K vs AutoAgents 1089K/367K，省约 10%。

**复杂任务：** AIME2025 45.6（vanilla 33.3，前 SOTA AutoAgents 42.2）；MATH 84.8 最佳；Trivia Creative Writing N=5 79.2(+5.0%)、N=10 81.8(+2.4%)；ScienceWorld 整体 26.59 vs AutoAgents 25.73 / EvoAgent 25.72 / Vanilla 25.29 [21: Table 8/9/10]。

**指标缺失：**
- **主表无方差 / 显著性检验**——多数 cell 差距 ≤1 点，GSM8K/MultiArith 等已近饱和（MultiArith 100.0、GSM8K ~94），1.2 点整体增益是否超噪声未知。
- token "显著降低"主要在 Nmax=10 才成立，但主实验默认 Nmax=5（此时与 AutoAgents 持平）。
- ScienceWorld 整体增益仅 +0.86，长链交互场景优势远不如短链 QA 明显。

## Weaknesses

- **核心增益微弱且置于近饱和 benchmark 上，无显著性检验。** 主表整体增益 1.2/0.9 点，落在 MMLU/GSM8K/HumanEval 这类单轮 QA 上，GSM8K(~94)/MultiArith(100) 已近天花板。论文从不报告 trial 间方差或显著性，"consistently outperforms" 的"1.2 点"难与随机种子波动区分 [21: §4.2, Table 1]。
- **"显著降低 token"对实际采用配置（Nmax=5）不成立。** Table 2 显示 Nmax=5 时 AgentInit 与 AutoAgents 的 Ptok/Ctok 基本持平（964K vs 1.0M）；只有 Nmax=10（超出生成 agent 数）才出现 ~10% 节省。但主实验默认 Nmax=5——abstract 的 "significantly reducing token consumption" 头条对论文实际跑的配置是夸大的 [21: §A.5, Table 11]。
- **relevance 度量奖励"复述查询"而非"贡献求解能力"。** Rel = cos(agent_desc, query) 是纯文本相似度：一个把查询关键词照搬进自身 description 的 agent 会拿高 relevance 分，却未必能贡献求解。论文未验证嵌入相似度与 agent 实际任务贡献的相关性，relevance 作为"专业度"代理缺乏构造效度 [21: §3.3 Eq.(9)]。
- **diversity 度量的是"描述文本多样性"而非"功能/行为多样性"。** Vendi Score 在 agent 描述向量上计算——两个措辞不同但功能等价的角色会被判为高多样性，两个措辞雷同但分工不同的角色会被判为低多样性。Max(s_ij) 降低 3.5% 只证明削减了文本冗余，不证明削减了功能冗余 [21: §3.3 Eq.(10)(11), §5.5]。
- **Selector Gs 把论文批判的 LLM self-preference bias 重新引回。** §1 的动机是"LLM 有 self-preference bias，需非 LLM 选择层"，但 Eq.(12) 最终仍由 LLM Selector 从 Pareto 集中挑一个团队。统计 Pareto 层只是把 bias 收窄到 Pareto 前沿上，并未消除——论文未消融"Pareto 集随机选 vs Selector 选"以隔离 Selector 的净贡献 [21: §1 vs §3.3 Eq.(12)]。
- **枚举式精确求解仅在玩具规模可行。** 候选团队枚举 Σ_{r=1}^{Nmax} C(|A_candidate|, r) 随候选 agent 数组合爆炸；Nmax≤5、候选池极小（平均团队规模 2.2–2.7）才使精确 Pareto 可行。NSGA-II 仅在 population=100 验证，对企业级 Agent/Skill 组合空间（goal 4 Composer 目标 5k–10k 实体）既未给精确解也未给近似解的可行性证据 [21: §5.3, §3.3]。
- **生成阶段 token 开销重，"高效"只指推理侧。** Limitations 自承"自动初始化带来 significant token overhead 且严重依赖高性能 LLM"。多轮 Planner/Observer/Formatter 生成成本未计入头条效率叙事，节省仅落在下游推理；在 init-once / inference-many 假设不成立的场景（每查询重做初始化）净成本可能为负 [21: Limitations vs §4.2]。
- **几乎无长链/多会话/有状态任务。** 10 个 benchmark 中 9 个是单轮 QA，ScienceWorld 是唯一交互式环境且增益最小（+0.86）。企业级跨会话、审批门、human-in-the-loop 的长链 AutoFlow 完全未测——团队初始化质量在长链上的影响是论文无法回答的 [21: §A.4, Table 10]。

## Relations

- **builds-on 06_why_do_multi_agent_llm_systems (MAST) [high]**：AgentInit §1 / §3.1 **显式引用 Pan et al. 2025（即 MAST 同源 workshop 论文）**，把 task derailment 与 step repetition 作为问题动机——这两者正是 MAST 的 FM-2.3 Task Derailment 与 FM-1.3 Step Repetition（15.7%，MAST 最高频失败模式）。AgentInit 主张"在初始化阶段剔除冗余/无关 agent 可缓解这些失败模式"，是 MAST 诊断框架的一个**初始化阶段的工程干预**。但 AgentInit 从未用 MAST 量化"剔除冗余 agent 后 FM-1.3/FM-2.3 实际下降多少"——只测了 Max(s_ij) 文本冗余降 3.5% 与端到端 +1.2 点，failure-mode 级因果链缺失。关系 [high]：引用关系显式，问题域定义性重叠（init 冗余 → MAST 高频失败模式）。
- **competes-with 04_rethinking_the_value_of_multi_agent (OneFlow) [med]**：两者都论证"更少 agent ≥ 更多 agent"，但路径相反且边界互补。OneFlow 把同质 MAS **折叠为单 agent**（agent 数→1）；AgentInit 保留多 agent 但**用 Pareto 选子集**（平均团队 2.2–2.7）。AgentInit 的 Random-selection 消融（随机选同样数量 agent 劣于 AgentInit）正面回应了 OneFlow 隐含的反问"增益是否只来自缩小规模"——AgentInit 主张增益来自**选择策略**而非规模本身。但二者都未跳出 OneFlow 的边界条件（任务 fits 单/小 LLM 上下文）：AgentInit 平均团队 2.7、Nmax≤5，恰在 OneFlow 折叠等价适用的小规模区间，故"AgentInit > 折叠"在上下文溢出场景（见 [9] DS-GRU 反证）未被验证 [med，AgentInit 未引用 OneFlow，关系为我推断]。
- **competes-with 12_automated_composition_of_agents (knapsack) [med]**：两者都解"从候选 agent 池组合出一个团队"的 MAS 组合问题，但目标函数不同：knapsack 类方法把组合建模为价值/成本约束下的背包优化，AgentInit 建模为 relevance/diversity 双目标 Pareto。二者对 goal 4 Composer 提供**两条可对照的选择算法基线**——值得在 Decision Agent Composer 设计中并列评估"背包式硬约束 vs Pareto 软权衡"在 BKN-anchored capability 上的召回质量 [med，需复核 note 12 具体建模]。
- **orthogonal to 09_llm_based_multi_agent_blackboard_system (Blackboard) [med]**：两者都服务 goal 3/4 但作用在 MAS 生命周期不同阶段——AgentInit 是**初始化期（init-time）静态团队选择**，Blackboard 是**运行期（runtime）动态请求路由 + helper 自荐**。互补而非竞争：AgentInit 先选出"哪些 agent 进队"，Blackboard 决定"运行时谁响应哪条广播"。一个值得注意的张力：Blackboard 主张"移除中心需精确知道每个 agent 能力的强假设"（运行时自荐），AgentInit 则**前置**用嵌入相关性 + Selector 精确评估每个 agent——两者对"中心是否需建模 agent 能力"给出相反工程立场，可在 Composer 设计中作为"静态精选 vs 动态自荐"两端权衡 [med]。
- **builds-on 05_agent_as_a_graph_knowledge_graph [low]**：Agent-as-a-Graph 用 KG 类型化做 typed retrieval（候选 agent/tool 的结构化召回，Recall@5 +14.9%）；AgentInit 用句向量余弦做 relevance 召回 + Vendi 多样性。两者都在"候选实体集上做结构化选择改善 flat 选择"上同向，但 AgentInit 用的是**无类型的嵌入相似度**而非显式 schema 类型——BKN 同构性弱于 [05]。AgentInit 的 relevance 度量恰是 [05] 所超越的"flat embedding retrieval"，可作为 BKN 类型化召回的对照下界 [low，我的综合]。
- **orthogonal to 03_why_reasoning_fails_to_plan (FLARE) [low]**：FLARE 论证步进推理在长链规划上的结构性局限；AgentInit 优化的是团队**组成**而非规划深度，且评测以单轮 QA 为主。两者层级不同，AgentInit 的"团队选好了"不解决"主 agent 单步贪心推理"的规划退化问题——对 goal 1/4 而言是正交补充而非替代 [low]。

### Relation to thesis

直接命中 **goal 4 (Composer)**，并对 thesis"更少/精选 agent"线与"协同失真治理"线提供 init-time 工程证据。

**1. AgentInit = Composer "自然语言任务 → 可审查 Agent Team Plan" 的一个候选选择算法 [high 关联度]。** thesis goal 4（Composer 基于已有 Agent/Template/Skill 组合，自然语言描述 → 可审查 Team Plan → 调度执行，明确不做端到端 spawn / DAG / A2A）的核心子问题是"**从候选能力池选哪几个组成 team**"。AgentInit 给出一个可直接移植的两段式：(a) 候选生成（对应 Composer 从已有 Agent/Template 库取候选），(b) Pareto(relevance, diversity) + Selector 选 team（对应 Composer 产出 reviewable Team Plan）。**关键工程启示：Composer 的团队选择不应只看 relevance（与任务相关），还要显式约束 diversity（避免选入功能冗余的 Agent）**——AgentInit 的 None/Random 消融（无选择 −1.1 点、随机选 −同规模仍劣）量化了"只去重不够、需双目标平衡"。

**2. 但 AgentInit 的两个度量在企业治理 + 5k–10k 规模下都需替换 [med，证伪向]。** (a) **规模**：AgentInit 枚举式 Pareto 仅在 Nmax≤5、候选池极小可行；Composer 目标实体规模高出两个数量级，NSGA-II 近似仅在 population=100 验证——与 [9] Blackboard 的"broadcast 成本线性放大"是同一类规模断层。Composer 必须保留 BKN 预过滤回退路径，不能直接套枚举 Pareto。(b) **治理**：AgentInit 的 relevance = 句向量余弦，是无类型的文本相似度——这正是 thesis Taste 中"纯 prompt/嵌入 vs BKN 语义结构化解耦"的对立面。Decision Agent Composer 应把 relevance 替换为 **BKN-anchored capability 类型化匹配**（对照 [05] +14.9% Recall@5），把 AgentInit 的嵌入 relevance 当作**对照下界**而非采用方案。

**3. 对"更少 agent"可证伪点的更新 [med]。** thesis 可证伪点"Composer 的'已有能力优先 + 半自动'路径"当前状态"未验证"——AgentInit 提供**间接支持**：在 4 框架 × 多 benchmark 上"精选子集 > 全量 agent"一致成立，且 Random 消融证明增益来自选择策略而非单纯缩规模。可把状态更新为"init-time 小规模有间接证据，缺企业级规模 + BKN 类型化选择的验证"。同时与 [4] OneFlow / [6] MAST(FM-1.3) 形成三角：OneFlow 说"同质场景折叠到 1"、MAST 说"冗余 agent → step repetition"、AgentInit 说"init 期精选可缓解"——三者共同把"Decision Agent 默认应倾向更小、经显式选择的 team"这一工程取向的证据强度从单点抬到 [med]。

**4. 对 goal 3 (Shared Workspace) 与 goal 2 (Heartbeat) 证据中性 [low]。** AgentInit 是一次性 init-time 静态选择，不涉及运行时共享存储（goal 3 由 [9] 承担）、也不涉及主动调度/心跳（goal 2 由 [8] 承担）。对这两个 goal 既不支持也不证伪。

**证据强度：** 论文整体 **[med-low]**——方法清晰、消融完整（None/Random/Pareto Worst 隔离了选择策略净贡献），但核心增益微弱（≤1.2 点、近饱和 benchmark、无显著性检验）、效率头条对默认 Nmax=5 不成立、两个选择度量是无类型文本相似度且仅在玩具规模验证。**应作为 Decision Agent Composer 团队选择算法的设计参考（双目标 relevance+diversity 框架值得借鉴）**，但其嵌入式 relevance 与枚举式 Pareto 都不能直接作为企业 5k–10k 规模 + BKN 治理场景下的采用方案——relevance 须换成 BKN 类型化召回、选择须保留预过滤回退。
