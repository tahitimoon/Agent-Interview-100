# 全库速览

> 每篇文章一行，一句话概括即该篇「简短回答」段的核心结论——扫一眼这份索引，就能了解全库 100 题各自讲了什么、当前的知识状态是什么。
>
> **更新日期**：2026-08-12 ｜ **收录**：100 篇 / 11 个模块

## 01-agent-architecture（Agent 架构，9 篇）

- [001-what-is-llm-agent](../01-agent-architecture/001-what-is-llm-agent.md) — Agent 是有状态、目标驱动、能与外界交互的自主系统
- [002-agent-core-components](../01-agent-architecture/002-agent-core-components.md) — 感知、推理、行动、记忆四模块构成认知闭环
- [003-agent-architecture-patterns](../01-agent-architecture/003-agent-architecture-patterns.md) — ReAct/Plan-Execute/LATS 各有侧重，生产多用混合模式
- [005-layered-agent-architecture](../01-agent-architecture/005-layered-agent-architecture.md) — Orchestrator-Worker 分层实现可扩展的专业化分工
- [006-agent-loop-and-error-recovery](../01-agent-architecture/006-agent-loop-and-error-recovery.md) — 循环终止靠外部强制，容错用四层防御模型
- [007-workflow-vs-agent](../01-agent-architecture/007-workflow-vs-agent.md) — 工作流与 Agent 是连续光谱，生产常用混合架构
- [009-self-reflection-correction](../01-agent-architecture/009-self-reflection-correction.md) — Reflexion 把环境反馈化为语言化反思存入记忆
- [010-production-agent-system-design](../01-agent-architecture/010-production-agent-system-design.md) — 生产系统须解决五大挑战，架构要 right-size
- [108-interview-deep-dive-chain](../01-agent-architecture/108-interview-deep-dive-chain.md) — 追问链逐层深入，找候选人能力天花板

## 02-rag（RAG，10 篇）

- [011-what-is-rag](../02-rag/011-what-is-rag.md) — 检索注入外部知识，解决知识截止、幻觉、领域缺失
- [012-rag-pipeline-components](../02-rag/012-rag-pipeline-components.md) — 索引、检索、生成三条流水线各有独立优化空间
- [013-chunking-strategies](../02-rag/013-chunking-strategies.md) — 递归分块是通用默认，核心权衡上下文与精度
- [014-vector-database-comparison](../02-rag/014-vector-database-comparison.md) — 选型看 Recall、尾延迟、过滤与运维，非 Benchmark
- [015-embedding-model-selection](../02-rag/015-embedding-model-selection.md) — Voyage 领先，OpenAI 3-large 是均衡生产默认
- [016-hybrid-retrieval](../02-rag/016-hybrid-retrieval.md) — 向量+BM25 经 RRF 融合，再加重排序两阶段
- [017-reranking-strategies](../02-rag/017-reranking-strategies.md) — Bi-Encoder 粗检 Top-100，Cross-Encoder 精排 Top-5
- [018-agentic-rag](../02-rag/018-agentic-rag.md) — 在检索生成流水线上加 Agent 控制循环，多轮自主决策
- [019-advanced-rag-variants](../02-rag/019-advanced-rag-variants.md) — Self-RAG/CRAG/Adaptive 各解一题，可组合多层防御
- [020-rag-evaluation-metrics](../02-rag/020-rag-evaluation-metrics.md) — 检索与生成双维度度量，RAGAS 整合为自动流水线

## 03-tool-use（工具使用，10 篇）

- [021-function-calling-basics](../03-tool-use/021-function-calling-basics.md) — LLM 只生成调用 JSON，实际执行由应用代码完成
- [022-tool-schema-design](../03-tool-use/022-tool-schema-design.md) — Schema 三要素中描述最关键，直接影响选择准确率
- [023-common-tool-patterns](../03-tool-use/023-common-tool-patterns.md) — 数据访问、代码执行、写操作三类，安全级别不同
- [024-tool-gateway-permissions](../03-tool-use/024-tool-gateway-permissions.md) — Gateway 中间层做鉴权限流审计，Agent 视为不可信
- [025-tool-selection-strategy](../03-tool-use/025-tool-selection-strategy.md) — 意图与描述语义匹配，描述质量是第一影响因素
- [026-tool-failure-handling](../03-tool-use/026-tool-failure-handling.md) — 超时+退避重试+断路器+降级分层防御，禁无限重试
- [027-model-context-protocol](../03-tool-use/027-model-context-protocol.md) — MCP 是 AI 界 USB-C，统一协议解决 N×M 集成问题
- [028-parallel-vs-sequential-tools](../03-tool-use/028-parallel-vs-sequential-tools.md) — 无依赖并行降延迟，有依赖顺序，生产用混合模式
- [029-dynamic-tool-discovery](../03-tool-use/029-dynamic-tool-discovery.md) — 运行时发现工具，解决上下文浪费与选择准确率下降
- [030-tool-use-security](../03-tool-use/030-tool-use-security.md) — 工具是最危险攻击面，纵深防御、不信任 LLM 输出

## 04-multi-agent（多 Agent，10 篇）

- [031-what-is-multi-agent](../04-multi-agent/031-what-is-multi-agent.md) — 多 Agent 专精分工并行协作，代价是通信与复杂度
- [032-communication-patterns](../04-multi-agent/032-communication-patterns.md) — 消息传递、共享状态、黑板三模式，黑板效率更优
- [033-orchestration-patterns](../04-multi-agent/033-orchestration-patterns.md) — Pipeline/Hub-Spoke/层级三种编排，按依赖与并行度选
- [034-task-allocation-coordination](../04-multi-agent/034-task-allocation-coordination.md) — 任务拆分分配协调，明确角色边界与通信模式
- [035-conflict-resolution](../04-multi-agent/035-conflict-resolution.md) — 投票、共识、仲裁解冲突，警惕 Agent 趋同效应
- [036-multi-agent-frameworks](../04-multi-agent/036-multi-agent-frameworks.md) — CrewAI 快、LangGraph 控制精、AutoGen 对话灵活
- [037-agent-handoff](../04-multi-agent/037-agent-handoff.md) — transfer_to 工具化交接，难点在上下文可靠传递
- [038-emergent-behavior](../04-multi-agent/038-emergent-behavior.md) — 涌现既是优势也是风险，需拓扑与监控做可控设计
- [039-debugging-monitoring-multi-agent](../04-multi-agent/039-debugging-monitoring-multi-agent.md) — 分布式追踪是核心方法，行业收敛到 OTel 标准
- [101-a2a-protocol](../04-multi-agent/101-a2a-protocol.md) — A2A 管 Agent 间横向协作，与 MCP 纵向扩展互补

## 05-memory-and-state（记忆与状态，8 篇）

- [040-memory-types](../05-memory-and-state/040-memory-types.md) — 短期/长期/工作记忆三类，LLM 无状态全靠外部工程
- [041-context-window-management](../05-memory-and-state/041-context-window-management.md) — 截断到分层摘要多种策略，简单策略常不逊于复杂
- [043-persistent-memory](../05-memory-and-state/043-persistent-memory.md) — 双层存储跨会话保留，关键挑战是选择性存储与遗忘
- [044-state-management-patterns](../05-memory-and-state/044-state-management-patterns.md) — LangGraph State+Reducer+Checkpoint 已成业界主流
- [045-cross-session-preferences](../05-memory-and-state/045-cross-session-preferences.md) — 持久记忆+偏好提取+动态适配让 Agent 了解用户
- [046-vector-vs-structured-memory](../05-memory-and-state/046-vector-vs-structured-memory.md) — 向量擅语义检索，结构化擅关系推理，推荐混合
- [047-knowledge-graph-memory](../05-memory-and-state/047-knowledge-graph-memory.md) — KG 记忆支持多跳与时间推理，Graphiti 双时间线
- [048-memory-forgetting-updating](../05-memory-and-state/048-memory-forgetting-updating.md) — 时间衰减、频率淘汰、冲突更新，效用删除更优

## 06-planning-and-reasoning（规划与推理，9 篇）

- [049-cot-and-tot](../06-planning-and-reasoning/049-cot-and-tot.md) — CoT 线性推理，ToT 探索回溯，按任务路径明确度选
- [050-task-decomposition](../06-planning-and-reasoning/050-task-decomposition.md) — LLM/程序化/HTN/ADaPT 分解，核心权衡拆分粒度
- [052-plan-and-solve-replanning](../06-planning-and-reasoning/052-plan-and-solve-replanning.md) — 先规划后执行，配合动态重规划形成完整闭环
- [053-llm-planning-limitations](../06-planning-and-reasoning/053-llm-planning-limitations.md) — LLM 不能真正规划，LLM-Modulo 外部验证器补救
- [055-reasoning-models](../06-planning-and-reasoning/055-reasoning-models.md) — 测试时计算扩展先想后答，强于推理但高延迟成本
- [056-mcts-in-agent-planning](../06-planning-and-reasoning/056-mcts-in-agent-planning.md) — MCTS 四步循环补 LLM 无法回溯之短，LATS 为代表
- [057-reasoning-quality-evaluation](../06-planning-and-reasoning/057-reasoning-quality-evaluation.md) — 超越答案对错评推理过程，过程评估更可靠
- [058-causal-reasoning](../06-planning-and-reasoning/058-causal-reasoning.md) — 因果阶梯三层，LLM 擅关联弱反事实，需外部因果模型
- [103-agentic-rl-grpo](../06-planning-and-reasoning/103-agentic-rl-grpo.md) — 任务完成度做奖励训 Agent，GRPO 免 Critic 降成本

## 07-prompt-engineering（Prompt 工程，11 篇）

- [059-system-prompt-principles](../07-prompt-engineering/059-system-prompt-principles.md) — System Prompt 是宪法：角色、约束、示例、分层结构
- [060-few-shot-vs-zero-shot](../07-prompt-engineering/060-few-shot-vs-zero-shot.md) — 简单任务 Zero-shot，特定格式 Few-shot，边界在扩大
- [061-structured-output](../07-prompt-engineering/061-structured-output.md) — 四种实现，生产推荐 Function Calling+Pydantic 验证
- [062-agentic-prompting](../07-prompt-engineering/062-agentic-prompting.md) — 优化多步决策链而非单次输出，含工具描述与护栏
- [063-prompt-chaining](../07-prompt-engineering/063-prompt-chaining.md) — 任务拆为顺序 LLM 调用链，可控可靠可观测
- [064-prompt-injection-defense](../07-prompt-engineering/064-prompt-injection-defense.md) — 头号安全威胁，无银弹，必须多层纵深防御
- [065-dspy-programmatic-prompting](../07-prompt-engineering/065-dspy-programmatic-prompting.md) — 编程而非提示，框架自动优化 Prompt 与示例
- [066-prompt-versioning-ab-testing](../07-prompt-engineering/066-prompt-versioning-ab-testing.md) — Prompt 即代码：版本控制、A/B 测试、渐进发布
- [067-meta-prompting](../07-prompt-engineering/067-meta-prompting.md) — 用 LLM 优化 Prompt，APE/OPRO 等可超人类手写
- [068-cross-model-prompt-portability](../07-prompt-engineering/068-cross-model-prompt-portability.md) — Prompt 高度模型特异，核心层+模型适配层解决
- [102-context-engineering](../07-prompt-engineering/102-context-engineering.md) — 从写好 Prompt 转向上下文的选择、组装与管理

## 08-evaluation（评估，8 篇）

- [069-evaluation-methodology](../08-evaluation/069-evaluation-methodology.md) — 自动指标、人工、LLM-as-Judge 三类混合使用
- [071-llm-as-judge](../08-evaluation/071-llm-as-judge.md) — 与人工一致性 80%+，需缓解位置、冗长等偏差
- [072-agent-benchmarks](../08-evaluation/072-agent-benchmarks.md) — SWE-bench/WebArena/GAIA 评完整任务执行过程
- [073-regression-testing](../08-evaluation/073-regression-testing.md) — Golden Dataset+Judge 评分+CI 集成防性能退化
- [074-traces-and-spans](../08-evaluation/074-traces-and-spans.md) — Trace/Span 树形结构展决策链，OTel 成行业标准
- [075-evaluation-tools-comparison](../08-evaluation/075-evaluation-tools-comparison.md) — Ragas 专 RAG，LangSmith 全栈，Langfuse 开源首选
- [076-static-benchmark-trap](../08-evaluation/076-static-benchmark-trap.md) — 高分不等于高能力，必须用自己的数据测试
- [077-continuous-evaluation-pipeline](../08-evaluation/077-continuous-evaluation-pipeline.md) — 评估贯穿开发、上线、运行期的持续闭环

## 09-safety-and-alignment（安全与对齐，8 篇）

- [078-agent-safety-risks](../09-safety-and-alignment/078-agent-safety-risks.md) — 四类风险，Prompt Injection 居首，警惕致命三角
- [079-guardrails-basics](../09-safety-and-alignment/079-guardrails-basics.md) — 输入输出双侧护栏，规则型与模型型组合多层防御
- [080-human-in-the-loop](../09-safety-and-alignment/080-human-in-the-loop.md) — 审批、置信度路由等模式，平衡自动化与人类监督
- [081-least-privilege-sandboxing](../09-safety-and-alignment/081-least-privilege-sandboxing.md) — 最小权限+沙箱隔离，Agent 需动态运行时权限
- [082-hallucination-detection](../09-safety-and-alignment/082-hallucination-detection.md) — 不确定性估计、知识验证、一致性检查三类检测
- [083-content-filtering-toxicity](../09-safety-and-alignment/083-content-filtering-toxicity.md) — 规则、分类器、LLM 多层级联做实时内容拦截
- [084-agent-alignment](../09-safety-and-alignment/084-agent-alignment.md) — 对齐失败是做错事，防规格游戏与欺骗性规划
- [085-red-teaming-agents](../09-safety-and-alignment/085-red-teaming-agents.md) — 攻击者视角主动测漏洞，已从可选变为合规必需

## 10-production-and-deployment（生产与部署，12 篇）

- [086-llmops-basics](../10-production-and-deployment/086-llmops-basics.md) — LLMOps 以使用模型为核心，成本重心在运行时推理
- [087-deployment-architecture](../10-production-and-deployment/087-deployment-architecture.md) — 接入、编排、模型网关、数据工具、观测五层架构
- [088-cost-optimization](../10-production-and-deployment/088-cost-optimization.md) — 缓存、路由、批处理等六策略组合可省 80-90%
- [089-model-routing](../10-production-and-deployment/089-model-routing.md) — 按请求复杂度动态选模型，RouteLLM 省 85% 成本
- [090-latency-optimization](../10-production-and-deployment/090-latency-optimization.md) — 流式、多层缓存、批处理三大手段降感知延迟
- [091-prompt-drift-management](../10-production-and-deployment/091-prompt-drift-management.md) — Prompt 未改输出也会漂移，需版本控制+持续监控
- [092-logging-monitoring-alerting](../10-production-and-deployment/092-logging-monitoring-alerting.md) — 结构化日志、分布式追踪、多维指标三层可观测
- [093-canary-ab-testing](../10-production-and-deployment/093-canary-ab-testing.md) — 灰度小流量+自动回滚，A/B 测试数据驱动决策
- [094-scaling-strategies](../10-production-and-deployment/094-scaling-strategies.md) — 无状态设计、队列解耦、按队列深度智能扩缩容
- [095-disaster-recovery-ha](../10-production-and-deployment/095-disaster-recovery-ha.md) — 多提供商冗余、检查点恢复、优雅降级保高可用
- [104-agent-production-troubleshooting](../10-production-and-deployment/104-agent-production-troubleshooting.md) — Trace 驱动排查配 OODA 循环，用数据不猜测
- [107-agent-code-review](../10-production-and-deployment/107-agent-code-review.md) — 循环保护、超时、错误处理等六维度检查清单

## 11-frameworks（框架，5 篇）

- [096-framework-overview](../11-frameworks/096-framework-overview.md) — LangChain 全能、LlamaIndex 专数据、Haystack 重生产
- [097-langgraph-concepts](../11-frameworks/097-langgraph-concepts.md) — State/Node/Edge 建模有状态工作流，含检查点
- [098-framework-vs-custom](../11-frameworks/098-framework-vs-custom.md) — 框架价值=省的时间减绕限制的时间，渐进式演进
- [099-assistants-api-vs-claude-sdk](../11-frameworks/099-assistants-api-vs-claude-sdk.md) — OpenAI 最小抽象云重，Claude SDK 绑 MCP 开放生态
- [100-testable-extensible-framework](../11-frameworks/100-testable-extensible-framework.md) — 端口-适配器+依赖注入+中间件实现可测可扩展
