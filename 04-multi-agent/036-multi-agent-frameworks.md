# 比较主流多 Agent 框架：CrewAI、AutoGen、LangGraph

> 难度：中级
> 分类：Multi-Agent

## 简短回答

三大多 Agent 框架各有侧重：**CrewAI** 以角色为核心，用 Crew（团队，自治协作）+ Flow（流程，deterministic 生产级编排）两层架构覆盖原型到生产；**LangGraph** 以图为核心，用节点-边-状态机实现精确的流程控制，可追踪可调试，适合复杂生产系统；**AutoGen** 系列（Microsoft 体系）以对话/事件驱动为核心，但现状是三个不同项目并存：**AG2**（ag2ai/ag2，社区 fork，AutoGen 0.2 延续）、**AutoGen 0.4+**（微软 2025-01 重写，actor 模型）已进入维护模式、微软新主推 **Microsoft Agent Framework (MAF)**（已于 2026.4 发布 1.0 GA，正式取代 AutoGen）。2026 年云厂商级新框架进一步涌入：**Google ADK**（Agent Development Kit，开源、原生 A2A 协议、可部署到 Vertex AI Agent Engine）、**OpenAI Agents SDK**（轻量级 Handoff 编排）。选择原则：需要快速上手选 CrewAI，需要精确控制选 LangGraph，需要群体对话/事件驱动选 AG2/AutoGen 0.4，需要微软最新生态选 MAF，需要 Google Cloud 一体化部署选 Google ADK。

## 详细解析

### 设计哲学对比

```
CrewAI:  角色驱动 —— "谁做什么"
         Crew（团队）= Agent + Task + Process（自治协作）
         Flow（流程）= deterministic 生产级编排（事件驱动 + state 持久化）

LangGraph: 图驱动 —— "怎么流转"
           Graph = Node + Edge + State
           支持条件分支、循环、并行

AutoGen 体系（三个不同项目，常被混淆）:
  AG2 (ag2ai/ag2)        : 2024-11 社区 fork，AutoGen 0.2 延续
  AutoGen 0.4+ (microsoft): 2025-01 微软重写，event-driven actor 架构，已进入维护
  MAF (Microsoft Agent Framework): 微软新主推，1.0 GA (2026.4)，正式取代 AutoGen

2026 厂商级新框架:
  Google ADK     : 开源，内置 Sequential/Parallel/Loop 编排，原生 A2A 协议
                   可一键部署到 Vertex AI Agent Engine
  OpenAI Agents SDK : 轻量级 Handoff 机制，代码优先编排，绑定 Responses API
```

### 架构详解

#### CrewAI

```python
from crewai import Agent, Task, Crew, Process

# 定义角色
researcher = Agent(
    role="高级研究分析师",
    goal="发现关于 {topic} 的最新趋势",
    backstory="你是一位经验丰富的研究者...",
    tools=[search_tool],
)

writer = Agent(
    role="技术写作专家",
    goal="将研究成果写成引人入胜的文章",
    backstory="你擅长将复杂技术概念简化...",
)

# 定义任务
research_task = Task(
    description="研究 {topic} 的最新进展",
    agent=researcher,
    expected_output="详细的研究报告"
)

write_task = Task(
    description="基于研究报告撰写博客文章",
    agent=writer,
    expected_output="1500 字的技术博客"
)

# 组装团队
crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, write_task],
    process=Process.sequential  # 注：Process.hierarchical 已不主推，复杂层级编排建议改用 Flow
)

result = crew.kickoff(inputs={"topic": "AI Agent"})
```

#### LangGraph

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict, Annotated

class State(TypedDict):
    messages: list
    research_data: str
    draft: str

# 定义节点（每个节点可以是 Agent）
async def research_node(state: State) -> dict:
    data = await researcher.invoke(state["messages"])
    return {"research_data": data}

async def write_node(state: State) -> dict:
    draft = await writer.invoke(state["research_data"])
    return {"draft": draft}

def should_revise(state: State) -> str:
    if quality_check(state["draft"]):
        return "end"
    return "revise"  # 条件路由

# 构建图
graph = StateGraph(State)
graph.add_node("research", research_node)
graph.add_node("write", write_node)
graph.add_node("revise", revise_node)

graph.add_edge(START, "research")
graph.add_edge("research", "write")
graph.add_conditional_edges("write", should_revise, {
    "end": END,
    "revise": "revise"
})
graph.add_edge("revise", "write")  # 循环

app = graph.compile()
```

#### AutoGen / AG2 / MAF

> **重要**：这是三个不同项目，import 路径与 API 都不同：
> - **AG2**（社区 fork，AutoGen 0.2.x 延续）：`from autogen import ConversableAgent`
> - **AutoGen 0.4+**（微软重写，event-driven actor 模型）：`from autogen_agentchat.agents import AssistantAgent`
> - **MAF**（Microsoft Agent Framework，微软新主推，1.0 GA 2026.4）：`from agent_framework import ChatAgent`（取代 AutoGen）
>
> 以下代码示例对应 **AG2 / AutoGen 0.2.x** API（最易上手），实际新项目建议优先 AutoGen 0.4+ 或 MAF。

```python
from autogen import ConversableAgent

# 定义对话 Agent
researcher = ConversableAgent(
    name="Researcher",
    system_message="你是研究分析师...",
    llm_config={"model": "gpt-4"},
)

writer = ConversableAgent(
    name="Writer",
    system_message="你是技术写作专家...",
    llm_config={"model": "gpt-4"},
)

critic = ConversableAgent(
    name="Critic",
    system_message="你是内容审核专家...",
    llm_config={"model": "gpt-4"},
)

# 群聊模式
from autogen import GroupChat, GroupChatManager

groupchat = GroupChat(
    agents=[researcher, writer, critic],
    messages=[],
    max_round=10
)
manager = GroupChatManager(groupchat=groupchat)
researcher.initiate_chat(manager, message="研究 AI Agent 的最新趋势")
```

### 核心维度对比

| 维度 | CrewAI | LangGraph | AG2 / AutoGen 0.4 / MAF |
|------|--------|-----------|---------|
| **核心抽象** | 角色/团队 | 图/状态机 | 对话/消息（AG2）、actor（0.4）、ChatAgent（MAF） |
| **学习曲线** | 低 | 高 | 中（AG2）、中高（0.4） |
| **流程控制** | Crew 顺序/Flow 事件驱动 | 任意图（条件、循环、并行） | 对话/事件驱动 |
| **状态管理** | 内置 state + Flow 持久化 | 精细状态 + Reducer | 对话历史/actor 状态 |
| **调试性** | 中 | 高（图可视化） | 低（非确定性对话） |
| **项目状态** | 活跃迭代 | 活跃迭代（1.0 GA，2025.10） | AG2 社区活跃；AutoGen 0.4 已进入维护；MAF 1.0 GA（2026.4） |
| **工具生态** | 中 | 300+ 集成 + LangSmith | Azure AI / Semantic Kernel 集成 |
| **Human-in-Loop** | 支持 | 原生支持 | 原生支持 |
| **配置方式** | 代码 + YAML 双轨 | 代码驱动 | 代码/Studio GUI |

### 各框架最佳适用场景

```python
framework_guide = {
    "CrewAI": [
        "业务流程自动化（角色清晰的团队协作）",
        "快速原型和 MVP",
        "非技术团队参与的项目（YAML 配置）",
    ],
    "LangGraph": [
        "复杂生产系统（需要精确流程控制）",
        "需要条件分支和循环的工作流",
        "对可观测性要求高的企业应用",
    ],
    "AG2 / AutoGen 0.4": [
        "群体决策和多 Agent 辩论（AG2 ConversableAgent / GroupChat）",
        "高并发事件驱动场景（AutoGen 0.4 actor 模型）",
        "研究探索和创意生成",
    ],
    "MAF (Microsoft Agent Framework)": [
        "微软主推方向，1.0 GA（2026.4），正式取代 AutoGen",
        "Azure AI Foundry / Semantic Kernel 深度集成",
        "企业级 Agent 平台（Microsoft 生态新项目优先）",
    ],
    "Google ADK (Agent Development Kit)": [
        "Google Cloud / Vertex AI 生态项目优先",
        "需要原生 A2A 协议的多 Agent 通信（见 #101）",
        "云上一体化构建-调试-部署（Agent Engine 托管运行时）",
    ],
}
```

### 2026 厂商级新框架

Agent 框架赛道在 2026 年持续升温，除三大开源框架外，云厂商相继推出自有框架，进一步碎片化但也验证了赛道价值：

- **Microsoft Agent Framework (MAF) 1.0 GA（2026.4）**：微软继 AutoGen 0.4 后的新主推方向，1.0 GA 标志其进入生产可用阶段。深度集成 Azure AI Foundry 与 Semantic Kernel，定位企业级 Agent 平台。微软体系内 AutoGen 0.4 已进入维护模式，新项目官方推荐迁移到 MAF。
- **Google ADK（Agent Development Kit）**：Google 开源的 Agent 开发套件，内置 SequentialAgent、ParallelAgent、LoopAgent 三种编排原语，**原生集成 A2A（Agent-to-Agent）协议**，可一键部署到 Vertex AI Agent Engine 托管运行。ADK 是 Google「Agent 平台化」战略的核心开发工具，与 A2A 协议深度绑定（多 Agent 通信协议详见 [#101 A2A 协议](./101-a2a-protocol.md)）。
- **OpenAI Agents SDK**：轻量级 Handoff 机制，代码优先编排，与 OpenAI Responses API 紧密集成。
- **AWS Agent Squad**：Agent-as-Tools 架构，Lead Agent 协调团队。

> **趋势判断**：LangGraph（1.0 GA，2025.10）、OpenAI Agents SDK、Google ADK、Microsoft Agent Framework 已形成「四强 + 开源社区（CrewAI / AG2）」格局。厂商级框架的优势是云生态绑定（Azure / Vertex AI / OpenAI）与开箱即用的托管运行时，代价是厂商锁定；开源框架（CrewAI、LangGraph）胜在模型与部署中立、社区生态更广。选型时先确认是否已有强云生态绑定，再决定走厂商路线还是开源路线。

## 常见误区 / 面试追问

1. **误区："选最流行的框架就对了"** — 框架选择应基于具体需求：需要可控性选 LangGraph，需要快速迭代选 CrewAI，需要灵活对话选 AutoGen。没有万能框架。

2. **误区："框架 = 生产就绪"** — 框架提供基础抽象，但生产环境还需要自行解决可观测性、错误处理、安全性等横切关注点。LangGraph 在这方面最成熟（配合 LangSmith），其他框架可能需要更多自定义。

3. **追问："能否混合使用多个框架？"** — 可以。例如用 LangGraph 做整体编排，单个节点内部用 CrewAI 的 Crew 完成子任务。但需要注意状态同步和调试复杂度的增加。

4. **追问："自研 vs 使用框架，如何取舍？"** — 如果需求与框架的抽象契合，使用框架；如果框架的限制导致大量 hack，考虑自研。OpenAI 的建议是：先用代码编排（最可控），只在需要灵活性时才引入 LLM 编排。

## 参考资料

- [CrewAI vs LangGraph vs AutoGen (DataCamp)](https://www.datacamp.com/tutorial/crewai-vs-langgraph-vs-autogen)
- [AI Agent Framework Comparison (Latenode)](https://latenode.com/blog/platform-comparisons-alternatives/automation-platform-comparisons/langgraph-vs-autogen-vs-crewai-complete-ai-agent-framework-comparison-architecture-analysis-2025)
- [Top AI Agent Frameworks (Codecademy)](https://www.codecademy.com/article/top-ai-agent-frameworks-in-2025)
- [Detailed Comparison of Top 6 AI Agent Frameworks (Turing)](https://www.turing.com/resources/ai-agent-frameworks)
- [Best AI Agent Frameworks 2025 (Maxim AI)](https://www.getmaxim.ai/articles/top-5-ai-agent-frameworks-in-2025-a-practical-guide-for-ai-builders/)
- [LangGraph vs CrewAI vs OpenAI Agents SDK 2026（Particula Tech，含 MAF 1.0 GA 对比）](https://particula.tech/blog/langgraph-vs-crewai-vs-openai-agents-sdk-2026)
- [Best AI Agent Frameworks 2026（AliceLabs 框架评测）](https://alicelabs.ai/en/insights/best-ai-agent-frameworks-2026)
- [LangChain + LangGraph 1.0 官方公告](https://www.langchain.com/blog/langchain-langgraph-1dot0)
- [AI Agent Frameworks 资源汇总（LangChain，含 Microsoft Agent Framework）](https://www.langchain.com/resources/ai-agent-frameworks)
- [Google ADK 官方文档](https://docs.cloud.google.com/gemini-enterprise-agent-platform/build/adk)
- [Agent Development Kit：轻松构建多 Agent 应用（Google 开发者博客）](https://developers.googleblog.com/en/agent-development-kit-easy-to-build-multi-agent-applications/)
