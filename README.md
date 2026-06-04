# Decision Agent: Multi-Agent Workers Architecture

## Topic

Decision agent 是一种通过 multi-agent workers team 协同完成任务的决策架构，可作为独立组件部署。本课题追踪该架构的技术路径与业界最新进展，重点关注技能管理、协同进化与长期任务执行三个维度。

## Papers

| # | Title | Axis | Priority | Status |
|---|-------|------|----------|--------|
| [01](notes/01_co_evolving_llm_decision_and_skill.md) | Co-Evolving LLM Decision and Skill Bank Agents for Long-Horizon Tasks | Architecture / Co-evolution | high | ✅ deep-read |
| [02](notes/02_graph_of_agents_a_graph_based.md) | Graph-of-Agents: A Graph-based Framework for Multi-Agent LLM Collaboration | Architecture / Graph Routing | med | ✅ deep-read |
| [03](notes/03_why_reasoning_fails_to_plan_a.md) | Why Reasoning Fails to Plan: A Planning-Centric Analysis of Long-Horizon Decision Making in LLM Agents | Architecture / Planning Mechanism | med | ✅ deep-read |
| [04](notes/04_rethinking_the_value_of_multi_agent.md) | Rethinking the Value of Multi-Agent Workflow: A Strong Single Agent Baseline | Architecture / Single-Agent Baseline | high | ✅ deep-read |
| [05](notes/05_agent_as_a_graph_knowledge_graph.md) | Agent-as-a-Graph: Knowledge Graph-Based Tool and Agent Retrieval for LLM Multi-Agent Systems | Tool Management / Catalog Retrieval | high | ✅ deep-read |
| [06](notes/06_why_do_multi_agent_llm_systems.md) | Why Do Multi-Agent LLM Systems Fail? | Architecture / Failure Diagnostics | high | ✅ deep-read |
| [07](notes/07_mcp_zero_active_tool_discovery_for.md) | MCP-Zero: Active Tool Discovery for Autonomous LLM Agents | Tool Management / Active Discovery | high | ✅ deep-read |
| [08](notes/08_simulating_human_cognition_heartbeat_driven_autonomous.md) | Simulating Human Cognition: Heartbeat-Driven Autonomous Thinking Activity Scheduling for LLM-based AI systems | Runtime / Heartbeat Scheduling | high | ✅ deep-read |
| [09](notes/09_llm_based_multi_agent_blackboard_system.md) | LLM-based Multi-Agent Blackboard System for Information Discovery in Data Science | Architecture / Broadcast-and-Self-Select | high | ✅ deep-read |
| [10](notes/10_collaborative_memory_multi_user_memory_sharing.md) | Collaborative Memory: Multi-User Memory Sharing in LLM Agents with Dynamic Access Control | Memory / Cross-Session Layered Access Control | high | ✅ deep-read |
| [11](notes/11_retrieval_models_aren_t_tool_savvy.md) | Retrieval Models Aren't Tool-Savvy: Benchmarking Tool Retrieval for Large Language Models | Tool Management / Retrieval Benchmarking | high | ✅ deep-read |
| [12](notes/12_automated_composition_of_agents_a_knapsack.md) | Automated Composition of Agents: A Knapsack Approach for Agentic Component Selection | Composer / Sandbox-Validated Knapsack Selection | high | ✅ deep-read |
| [13](notes/13_toolomni_enabling_open_world_tool_use.md) | ToolOmni: Enabling Open-World Tool Use via Agentic Learning with Proactive Retrieval and Grounded Execution | Tool Management / RL-Trained Proactive Retrieval | high | ✅ deep-read |
| [16](notes/16_g_memory_tracing_hierarchical_memory_for.md) | G-Memory: Tracing Hierarchical Memory for Multi-Agent Systems | Memory / MAS Cross-Trial Self-Evolution | high | ✅ deep-read |
| [17](notes/17_fademem_biologically_inspired_forgetting_for_efficient.md) | FadeMem: Biologically-Inspired Forgetting for Efficient Agent Memory | Memory / Importance-Modulated Decay & Forgetting | high | ✅ deep-read |
| [18](notes/18_governed_memory_a_production_architecture_for.md) | Governed Memory: A Production Architecture for Multi-Agent Workflows | Memory / Org-Level Governed Memory & Schema Lifecycle | high | ✅ deep-read |
| [19](notes/19_think_before_you_act_a_neurocognitive.md) | Think Before You Act — A Neurocognitive Governance Model for Autonomous AI Agents | Governance / Pre-Action Action-Level Deliberation | high | ✅ deep-read |
| [20](notes/20_anticipate_and_learn_unleashing_idle_time.md) | Anticipate and Learn: Unleashing Idle-Time Compute in Proactive Agents | Runtime / Prediction-Guided Idle-Time Compute | high | ✅ deep-read |
| [21](notes/21_agentinit_initializing_llm_based_multi_agent.md) | AgentInit: Initializing LLM-based Multi-Agent Systems via Diversity and Expertise Orchestration | Composer / Init-Time Multi-Objective Team Selection | high | ✅ deep-read |

## Thesis

Decision Agent 是"治理优先、Harness-first 的企业级决策智能体底座"，5 个 0.8 design goals 须被设计为有治理边界的有限主动模式（goal 2 与 ISF 张力、goal 3/4 按"内部处理 vs 外部副作用"分轨），治理底座由 [19] PAGRL（动作层）+ [18] Governed Memory（记忆层）构成参考骨架但须以 AgentSpec 类外部强制兜底。BKN 语义解耦获 [5][7][11][13] 四条证据线支持，goal 5 Memory 蓝图由 [10]+[16]+[17]+[18] 四层堆叠；goal 2 主动 Runtime 由 [20] ProAct 提供机制骨架但须改造为 store-only + 显式调度来源 + push 经 PAGRL ESCALATE。goal 4 Composer 的团队选择获 [12] knapsack 与 [21] AgentInit Pareto 两条 init/装配期算法基线，但二者枚举/嵌入度量均须换为 BKN 类型化召回 + 预过滤回退方可上企业规模。可证伪：若 PAGRL ESCALATE 漏报率 > 10%、BKN + 通用 embedding 在 5k+ 规模 Recall@10 ≥ 50%、或真实业务对话 Anticipation Recall 接近 Undirected Idle 水平，则核心假设需修订。

See [`.researcher/thesis.md`](.researcher/thesis.md) for the full working thesis.

## Depth

- [`notes/00_research_landscape.md`](notes/00_research_landscape.md) — living synthesis of all read papers
- [`.researcher/thesis.md`](.researcher/thesis.md) — working thesis and research posture

*Last Updated: 2026-06-04 (v19)*
