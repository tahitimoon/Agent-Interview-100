# LLMOps 与 Agent 部署架构

> 难度：中级
> 分类：Production & Deployment

## 简短回答

**LLMOps** 是专门为 LLM 驱动应用设计的运维实践体系，是 MLOps 在大语言模型时代的演进。与 MLOps 的核心区别：MLOps 以**训练模型**为核心（数据→特征→训练→部署），LLMOps 以**使用模型**为核心（Prompt→RAG→评估→部署），主要成本从训练 GPU 转移到**运行时推理 API**，迭代周期从周/月级缩短到小时/天级，核心产物从模型文件变为 **Prompt、向量索引、工具配置、护栏规则**，评估方式从精确指标（AUC/F1）转为 **LLM-as-Judge + 人工评估**。

**生产级 Agent 部署架构**包含五个核心层：(1) **接入层**——API Gateway 负责认证、限流、路由，支持 WebSocket/SSE 流式响应；(2) **Agent 编排层**——Agent 运行时（LangGraph、自研框架）管理推理循环、工具调用、状态管理；(3) **模型网关层**——LLM Gateway（LiteLLM/Portkey）统一多模型 API、故障转移、Prompt 缓存；(4) **数据与工具层**——向量数据库（RAG）、工具服务（MCP）、持久化存储；(5) **可观测性层**——Trace 追踪（Langfuse）、指标监控（Prometheus）、告警。2025 关键趋势：**Context Engineering 成为核心学科**、**Plan-then-Execute** 优于 ReAct、**MCP 协议**成工具集成行业标准。部署策略推荐：先用 Serverless 快速上线，按需迁移到 K8s。

LLMOps 不替代 MLOps，而是在其基础上扩展，企业通常需要两者协同工作。

## 详细解析

### Part A：LLMOps vs MLOps

#### 核心差异对比

| 维度 | MLOps | LLMOps |
| --- | --- | --- |
| 核心模型 | 自训练模型 | 预训练 LLM（API） |
| 数据类型 | 结构化数据 | 非结构化文本 |
| 主要产物 | 模型文件、特征 | Prompt、向量索引 |
| 迭代周期 | 周/月级 | 小时/天级 |
| 主要成本 | 训练（GPU 时间） | 推理（API 调用） |
| 评估方式 | AUC/F1/MSE | LLM Judge/人工 |
| 版本管理 | 模型+数据版本 | Prompt+配置版本 |
| 可观测性 | 模型漂移 | Prompt 漂移+Trace |
| 部署模式 | 模型服务器 | API Gateway |
| 安全关注 | 数据隐私 | Prompt Injection |

#### LLMOps 的核心组件

```python
llmops_components = {
    "Prompt 管理": {
        "内容": "Prompt 版本控制、A/B 测试、模板管理",
        "工具": "LangSmith, Humanloop, PromptLayer",
        "类比": "MLOps 中的特征工程",
    },
    "模型网关": {
        "内容": "API 路由、负载均衡、故障转移、成本控制",
        "工具": "LiteLLM, Portkey, Kong AI Gateway",
        "类比": "MLOps 中的模型服务器",
    },
    "RAG 基础设施": {
        "内容": "向量数据库、文档处理、索引更新",
        "工具": "Pinecone, Weaviate, Chroma",
        "类比": "MLOps 中的特征存储",
    },
    "评估管道": {
        "内容": "自动化评估、回归测试、LLM Judge",
        "工具": "DeepEval, Ragas, Braintrust",
        "类比": "MLOps 中的模型验证",
    },
    "可观测性": {
        "内容": "Trace/Span 追踪、成本监控、质量监控",
        "工具": "Langfuse, LangSmith, Arize Phoenix",
        "类比": "MLOps 中的模型监控",
    },
    "安全护栏": {
        "内容": "输入/输出过滤、PII 检测、内容安全",
        "工具": "Guardrails AI, NeMo Guardrails",
        "类比": "MLOps 中的数据验证（但范围更广）",
    },
}
```

#### LLMOps 工作流

```python
class LLMOpsWorkflow:
    """LLMOps 的典型工作流"""

    def development_cycle(self):
        """开发迭代循环"""
        return [
            "1. Prompt 设计与迭代",
            "   - 编写/修改 System Prompt",
            "   - 在 Playground 中测试",
            "   - 版本化保存",

            "2. RAG 配置（如需要）",
            "   - 文档处理和分块",
            "   - 向量索引构建",
            "   - 检索策略调优",

            "3. 评估",
            "   - 在 Golden Dataset 上运行评估",
            "   - LLM Judge 自动评分",
            "   - 对比基线版本",

            "4. 部署",
            "   - Prompt 和配置推送到生产",
            "   - 灰度发布（5% → 25% → 100%）",
            "   - 实时监控",

            "5. 监控与优化",
            "   - 追踪质量指标和成本",
            "   - 收集用户反馈",
            "   - 识别优化机会",
        ]

    def key_metrics(self):
        """LLMOps 核心监控指标"""
        return {
            "质量": ["回答准确率", "用户满意度", "幻觉率"],
            "性能": ["延迟 P50/P95", "TTFT", "吞吐量"],
            "成本": ["每请求成本", "每用户日均成本", "Token 使用量"],
            "安全": ["护栏触发率", "注入检测率", "PII 泄露率"],
        }
```

#### 企业协同：MLOps + LLMOps

```
实际企业 AI 系统中 MLOps 和 LLMOps 的协同：

保险行业示例：
├── MLOps 管理：
│   ├── 定价模型（结构化数据 → 风险评分）
│   ├── 欺诈检测模型（交易数据 → 欺诈概率）
│   └── 客户分群模型（行为数据 → 用户画像）
│
└── LLMOps 管理：
    ├── 智能客服（用户问题 → 自然语言回答）
    ├── 保单解释助手（保单文档 → RAG 回答）
    └── 理赔报告生成（结构化数据 → 文本报告）

两者共享：CI/CD 管道、监控基础设施、数据治理框架
```

### Part B：Agent 系统部署架构

#### 五层架构全景

```mermaid
flowchart TD
    subgraph L1["接入层"]
        A["API Gateway / Load Balancer<br/>认证 · 限流 · 路由 · 流式响应"]
    end
    subgraph L2["Agent 编排层"]
        B["Agent Runtime（LangGraph / 自研）<br/>推理循环 · 工具选择 · 状态检查点"]
    end
    subgraph L3["模型网关层"]
        C["LLM Gateway（LiteLLM / Portkey）<br/>模型路由 · 故障转移 · 缓存 · 成本追踪"]
    end
    subgraph L4["数据与工具层"]
        D1["向量数据库<br/>(RAG)"]
        D2["工具服务<br/>(MCP)"]
        D3["状态存储<br/>(Redis/PG)"]
        D4["安全护栏<br/>(Guardrails)"]
    end
    subgraph L5["可观测性层"]
        E["Traces(Langfuse)<br/>Metrics(Prometheus) + Alerts"]
    end
    L1 --- L2
    L2 --- L3
    L3 --- L4
    L4 --- L5
    classDef proc fill:#e3f2fd,stroke:#1565c0,color:#0d47a1
    classDef store fill:#e8f5e9,stroke:#2e7d32,color:#1b5e20
    classDef neutral fill:#eceff1,stroke:#546e7a,color:#37474f
    class A,B,C proc
    class D1,D2,D3,D4 store
    class E neutral
```
*生产 Agent 是五层堆叠：接入、编排、模型网关、数据工具、观测——LLM 调用必须过网关层。*

#### 各层详解

```python
# 1. 接入层
api_layer = {
    "API Gateway": {
        "职责": "认证、限流、路由、CORS",
        "选择": "Kong / AWS API Gateway / Nginx",
    },
    "流式响应": {
        "协议": "SSE（Server-Sent Events）用于单向流",
        "场景": "Token 流式输出，用户无需等待完整回答",
        "实现": "FastAPI StreamingResponse / WebSocket",
    },
    "健康检查": {
        "端点": "/health 检查服务状态",
        "内容": "服务可用性 + LLM API 连通性 + DB 连通性",
    },
}

# 2. Agent 编排层
orchestration_layer = {
    "Agent Runtime": {
        "职责": "管理 Agent 的推理-行动循环",
        "框架选择": {
            "LangGraph": "最成熟，适合复杂有状态工作流",
            "CrewAI": "多 Agent 协作场景",
            "自研": "需要完全控制时",
        },
    },
    "状态管理": {
        "检查点": "每步保存状态，支持恢复和回放",
        "会话管理": "跨请求维护对话上下文",
        "存储": "Redis（短期）+ PostgreSQL（长期）",
    },
    "部署模式": {
        "Plan-then-Execute": {
            "优势": "规划和执行解耦，支持并行",
            "适用": "复杂多步任务",
        },
        "ReAct": {
            "优势": "简单灵活，逐步推理",
            "适用": "简单任务、对话式交互",
        },
    },
}

# 3. 模型网关层
model_gateway = {
    "统一 API": "一个接口调用 OpenAI/Anthropic/Google 等",
    "故障转移": "主模型不可用时自动切换到备用模型",
    "Prompt 缓存": "相同前缀的请求复用缓存，降低成本 90%",
    "模型路由": "按任务复杂度选择模型（简单→小模型，复杂→大模型）",
    "成本追踪": "实时记录每个请求的 Token 用量和费用",
    "工具": "LiteLLM, Portkey, OpenRouter",
}

# 4. 数据与工具层
data_tool_layer = {
    "向量数据库": {
        "用途": "RAG 检索",
        "选择": "Pinecone（托管）/ Weaviate（自部署）/ pgvector（嵌入PG）",
    },
    "工具服务": {
        "协议": "MCP（Model Context Protocol）",
        "说明": "标准化 LLM 与外部工具/数据源的连接",
    },
    "持久化存储": {
        "对话历史": "PostgreSQL",
        "会话缓存": "Redis",
        "文件存储": "S3 / GCS",
    },
}
```

#### 容器化部署方案

```yaml
# docker-compose.yml — 基础部署配置
version: '3.8'
services:
  agent-api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - REDIS_URL=redis://redis:6379
      - DATABASE_URL=postgresql://agent_user:${POSTGRES_PASSWORD}@postgres:5432/agent
    depends_on:
      - redis
      - postgres
    deploy:
      resources:
        limits:
          memory: 2G  # Agent 需要较多内存
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  postgres:
    image: pgvector/pgvector:pg16
    environment:
      # pgvector 镜像基于官方 postgres 镜像，POSTGRES_PASSWORD 是必填的
      # 否则容器启动会报 "you must specify POSTGRES_PASSWORD..."
      - POSTGRES_DB=agent
      - POSTGRES_USER=agent_user
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

配套 `.env` 示例：

```bash
# .env （生产中通过密钥管理服务注入，不要提交到 Git）
OPENAI_API_KEY=sk-...
POSTGRES_PASSWORD=change-me-in-production
```

#### 部署策略选择

```
你的场景是什么？
│
├── 早期/MVP（< 1000 用户）
│   → Serverless（Cloud Run / Lambda）
│   → 按用量付费，无需管理服务器
│   → 注意：冷启动延迟 + 执行时间限制
│
├── 中期/增长（1K-100K 用户）
│   → 容器化（ECS / Cloud Run 持续运行）
│   → 自动扩缩容 + 健康检查
│   → 引入 Redis 缓存和 CDN
│
├── 大规模（100K+ 用户）
│   → Kubernetes 集群
│   → 多区域部署 + 全球负载均衡
│   → 自建模型服务（vLLM）降低成本
│
└── 共同关注：
    ├── 密钥管理（Vault / AWS Secrets Manager）
    ├── 网络隔离（Agent 不直接访问公网）
    ├── CI/CD（代码 + Prompt 都走管道）
    └── 可观测性（从 Day 1 接入 Trace）
```

## 常见误区 / 面试追问

1. **误区："LLMOps 就是 MLOps 加个 Prompt 管理"** — LLMOps 引入了全新的挑战维度：非确定性输出评估、Prompt Injection 安全、运行时成本控制、向量数据库管理等。这些不是简单地在 MLOps 上"加功能"，而是需要不同的思维方式和工具链。

2. **误区："用了 LLM API 就不需要 MLOps 了"** — 大多数企业的 AI 系统同时包含传统 ML 模型和 LLM 应用。推荐系统、风控模型仍然需要 MLOps。LLMOps 和 MLOps 是互补关系，不是替代关系。

3. **误区："直接在应用代码里调 LLM API 就行"** — 生产系统需要模型网关层来处理故障转移、限流、成本控制和缓存。直接调 API 会导致：单点故障、无法切换模型、成本失控、无法追踪。LiteLLM 或 Portkey 可以一行代码解决这些问题。

4. **误区："Agent 系统和普通 Web 服务部署一样"** — Agent 有独特的部署挑战：(1) 长时间运行（一次任务可能执行数分钟）；(2) 状态管理（多步执行需要检查点）；(3) 高内存需求（上下文窗口占用大量内存）；(4) 不确定的成本（每次请求的 Token 消耗不同）。

5. **追问："LLMOps 的最大挑战是什么？"** — 评估。传统 ML 有明确的量化指标（准确率、F1），但 LLM 输出的质量是主观且多维度的。如何自动化、可靠地评估 LLM 输出质量是 LLMOps 的核心难题，也是 LLM-as-Judge 等技术兴起的原因。

6. **追问："小团队如何起步 LLMOps？"** — 最小可行方案：(1) Prompt 用 Git 版本管理；(2) 用 Langfuse（免费开源）记录所有请求和成本；(3) 维护 50 条 Golden Dataset 做回归测试；(4) 用 LiteLLM 统一多模型 API。这四步一周内可以搭好。

7. **追问："如何处理 Agent 的长时间运行任务？"** — (1) 异步执行：接收请求后立即返回任务 ID，客户端轮询或 WebSocket 推送结果；(2) 检查点机制：每步保存状态，支持断点恢复；(3) 超时保护：设置最大执行时间和最大步数；(4) 流式输出：中间结果实时推送给用户。

8. **追问："选择框架还是自研？"** — 参考原则：如果需求匹配框架能力的 80%+，用框架（LangGraph）；如果需要深度定制或框架是瓶颈，自研核心 Agent 循环但复用社区工具（LiteLLM、Langfuse）。框架选择比模型选择更重要。

## 参考资料

- [MLOps vs LLMOps: What's the Difference? (ZenML)](https://www.zenml.io/blog/mlops-vs-llmops)
- [What is LLMOps Compared to MLOps (Pluralsight)](https://www.pluralsight.com/resources/blog/ai-and-data/what-is-llmops)
- [From MLOps to LLMOps: The Evolution of Automation (CircleCI)](https://circleci.com/blog/from-mlops-to-llmops/)
- [LLMOps vs MLOps: Key Differences and Evolution (Ideas2IT)](https://www.ideas2it.com/blogs/llmops-vs-mlops-key-differences-and-evolution)
- [What is LLMOps? Key Components & Differences to MLOps (lakeFS)](https://lakefs.io/blog/llmops/)
- [LLM Agents in Production: Architectures, Challenges, and Best Practices (ZenML)](https://www.zenml.io/blog/llm-agents-in-production-architectures-challenges-and-best-practices)
- [Deploying AI Agents to Production: Architecture and Implementation Roadmap (MLM)](https://machinelearningmastery.com/deploying-ai-agents-to-production-architecture-infrastructure-and-implementation-roadmap/)
- [Architecting Efficient Context-Aware Multi-Agent Framework for Production (Google)](https://developers.googleblog.com/architecting-efficient-context-aware-multi-agent-framework-for-production/)
- [What 1,200 Production Deployments Reveal About LLMOps in 2025 (ZenML)](https://www.zenml.io/blog/what-1200-production-deployments-reveal-about-llmops-in-2025)
- [LLM Agents: The Enterprise Technical Guide 2025 (Aisera)](https://aisera.com/blog/llm-agents/)

---

> 📎 本题由原 #086（LLMOps 基础）与 #087（部署架构）合并而来（2026-05-23 重构）
