# 什么是 Context Engineering？它与 Prompt Engineering 有何本质区别？

> 难度：高级
> 分类：Prompt Engineering

## 简短回答

**Context Engineering** 是一种从"如何写好单条 Prompt"到"如何为 LLM 构建完整、精准上下文"的范式转变。它关注的核心问题不再是措辞技巧，而是**上下文的选择、组装与管理**。一个 LLM 调用的效果，70% 取决于喂给它的上下文质量，而非 Prompt 本身的遣词造句。Context Engineering 将上下文视为四大来源的动态组合：**System Prompt**、**Tool Results**、**Conversation History** 和 **External Knowledge**（RAG / API）。其核心挑战在于**上下文窗口是有限的"房地产"**——必须在有限的 token 预算内，为当前任务挑选信息密度最高的上下文片段。在 **Agentic 场景**中，这一挑战尤为突出：多轮 tool call 会持续积累上下文，若不加管理，关键信息会被淹没在噪声中，导致 Agent 性能急剧下降。

## 详细解析

### 从 Prompt Engineering 到 Context Engineering

Prompt Engineering 聚焦于"怎么问"，Context Engineering 聚焦于"用什么信息去问"。这是一个维度的跃升：

```
┌─────────────────────────────────────────────────────────┐
│                  Prompt Engineering                      │
│   "如何措辞、格式化一条指令以获得更好的输出"                  │
│   ┌─────────────────────────────┐                       │
│   │  System Prompt 写作技巧     │                       │
│   │  Few-shot 示例设计          │                       │
│   │  CoT / ReAct 格式           │                       │
│   └─────────────────────────────┘                       │
└─────────────────────────────────────────────────────────┘
                        │
                        ▼ 范式升级
┌─────────────────────────────────────────────────────────┐
│                 Context Engineering                      │
│   "如何为 LLM 构建正确的、完整的决策上下文"                  │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│   │ System   │ │ Tool     │ │ History  │ │ External │  │
│   │ Prompt   │ │ Results  │ │ 对话历史  │ │ Knowledge│  │
│   └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘  │
│        │            │            │            │          │
│        └────────────┴─────┬──────┴────────────┘          │
│                           ▼                              │
│               ┌───────────────────┐                      │
│               │  动态上下文组装器  │                      │
│               │ (Context Builder) │                      │
│               └─────────┬─────────┘                      │
│                         ▼                                │
│               ┌───────────────────┐                      │
│               │   LLM 调用入口    │                      │
│               └───────────────────┘                      │
└─────────────────────────────────────────────────────────┘
```

### 四大上下文来源

| 来源 | 内容 | 特征 | 管理策略 |
|------|------|------|----------|
| **System Prompt** | 角色定义、工具描述、输出格式、约束规则 | 静态、每次调用都携带 | 精简压缩，避免冗余 |
| **Tool Results** | 搜索结果、API 返回、代码执行输出 | 动态生成、体积不可控 | 截断 / 摘要 / 选择性保留 |
| **Conversation History** | 用户多轮对话、之前的 Agent 推理过程 | 线性增长、含大量噪声 | 滑动窗口 / 摘要压缩 |
| **External Knowledge** | RAG 检索文档、知识库、用户画像 | 按需注入、相关性参差 | 相关性排序 + top-k 截断 |

### 上下文窗口的"房地产"管理

```
┌────────────────── Context Window (128K tokens) ──────────────────┐
│                                                                  │
│  ┌──────────────┐  固定区域：~10%                                │
│  │ System Prompt│  角色 + 工具描述 + 格式要求                      │
│  └──────────────┘                                                │
│  ┌──────────────┐  弹性区域：~30%                                │
│  │ RAG / 知识库  │  根据查询动态检索                               │
│  └──────────────┘                                                │
│  ┌──────────────┐  增长区域：~40%                                │
│  │ 对话历史      │  多轮交互 + tool call 结果                      │
│  └──────────────┘                                                │
│  ┌──────────────┐  预留区域：~20%                                │
│  │ 输出空间      │  模型生成 token 的预算                          │
│  └──────────────┘                                                │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

关键原则：上下文不是越多越好，而是信噪比越高越好。
每加入一段上下文都要问：它对当前决策的边际贡献 > 它消耗的 token 成本吗？
```

### Prompt Engineering vs Context Engineering 对比

| 维度 | Prompt Engineering | Context Engineering |
|------|-------------------|-------------------|
| 核心关注 | 单条指令的措辞与格式 | 整体上下文的选择与组装 |
| 优化对象 | Prompt 模板 | 上下文管道 (Context Pipeline) |
| 技能要求 | 语言直觉 + 试错调优 | 系统设计 + 信息架构 |
| 适用场景 | 单轮问答、简单任务 | Agentic 系统、复杂工作流 |
| 评估指标 | 输出质量 | 上下文信噪比 + 输出质量 |
| 可扩展性 | 低（人工逐条调优） | 高（程序化、可自动化） |
| 类比 | 写好一封邮件 | 管理整个项目的信息流 |

### 动态上下文组装器：Python 实现

```python
from dataclasses import dataclass, field
from enum import Enum
from typing import Optional


class ContextSource(Enum):
    """上下文来源类型"""
    SYSTEM = "system"
    TOOL_RESULT = "tool_result"
    HISTORY = "history"
    KNOWLEDGE = "knowledge"


@dataclass
class ContextBlock:
    """单个上下文块"""
    source: ContextSource
    content: str
    priority: int = 5          # 1-10，越高越重要
    token_count: int = 0       # 预估 token 数
    metadata: dict = field(default_factory=dict)


class ContextBuilder:
    """动态上下文组装器——Context Engineering 的核心实现"""

    def __init__(self, max_tokens: int = 128_000, reserve_output: float = 0.2):
        self.max_tokens = max_tokens
        # 为模型输出预留空间
        self.available_tokens = int(max_tokens * (1 - reserve_output))
        self.blocks: list[ContextBlock] = []

        # 各来源的 token 预算上限（百分比）
        self.budget = {
            ContextSource.SYSTEM: 0.12,      # System Prompt 最多占 12%
            ContextSource.KNOWLEDGE: 0.35,   # RAG 知识最多占 35%
            ContextSource.HISTORY: 0.35,     # 对话历史最多占 35%
            ContextSource.TOOL_RESULT: 0.18, # 工具结果最多占 18%
        }

    def add(self, source: ContextSource, content: str,
            priority: int = 5, metadata: Optional[dict] = None) -> "ContextBuilder":
        """添加一个上下文块"""
        token_count = self._estimate_tokens(content)
        self.blocks.append(ContextBlock(
            source=source,
            content=content,
            priority=priority,
            token_count=token_count,
            metadata=metadata or {},
        ))
        return self  # 链式调用

    def build(self) -> list[dict]:
        """
        核心方法：根据优先级和预算，组装最终上下文。
        返回 OpenAI 兼容的 messages 格式。
        """
        # 第一步：按来源分组
        grouped: dict[ContextSource, list[ContextBlock]] = {}
        for block in self.blocks:
            grouped.setdefault(block.source, []).append(block)

        # 第二步：每组内按优先级降序排列
        for source in grouped:
            grouped[source].sort(key=lambda b: b.priority, reverse=True)

        # 第三步：按预算分配，优先级高的先占位
        selected: list[ContextBlock] = []
        for source, blocks in grouped.items():
            budget_tokens = int(self.available_tokens * self.budget.get(source, 0.1))
            used = 0
            for block in blocks:
                if used + block.token_count <= budget_tokens:
                    selected.append(block)
                    used += block.token_count
                else:
                    # 尝试截断以填满预算
                    remaining = budget_tokens - used
                    if remaining > 100:  # 至少保留 100 token 才值得截断
                        truncated = self._truncate(block.content, remaining)
                        selected.append(ContextBlock(
                            source=block.source,
                            content=truncated,
                            priority=block.priority,
                            token_count=remaining,
                        ))
                    break

        # 第四步：组装为 messages 格式
        return self._to_messages(selected)

    def _to_messages(self, blocks: list[ContextBlock]) -> list[dict]:
        """将上下文块转为 LLM messages 格式"""
        messages = []
        # System Prompt 放最前面
        sys = [b for b in blocks if b.source == ContextSource.SYSTEM]
        if sys:
            messages.append({"role": "system",
                             "content": "\n\n".join(b.content for b in sys)})
        # RAG 知识注入
        know = [b for b in blocks if b.source == ContextSource.KNOWLEDGE]
        if know:
            messages.append({"role": "system",
                             "content": "[相关知识]\n" + "\n---\n".join(b.content for b in know)})
        # 对话历史按原始顺序
        hist = sorted([b for b in blocks if b.source == ContextSource.HISTORY],
                       key=lambda b: b.metadata.get("turn_index", 0))
        for b in hist:
            messages.append({"role": b.metadata.get("role", "user"), "content": b.content})
        return messages

    @staticmethod
    def _estimate_tokens(text: str) -> int:
        """粗略估算 token 数（中文约 1.5 字符/token，英文约 4 字符/token）"""
        cn_chars = sum(1 for c in text if '\u4e00' <= c <= '\u9fff')
        en_chars = len(text) - cn_chars
        return int(cn_chars / 1.5 + en_chars / 4)

    @staticmethod
    def _truncate(text: str, max_tokens: int) -> str:
        """按 token 预算截断文本，保留前部内容"""
        # 简化实现：按字符比例截断
        ratio = max_tokens / max(ContextBuilder._estimate_tokens(text), 1)
        cut_point = int(len(text) * min(ratio, 1.0))
        return text[:cut_point] + "\n...[已截断]"


# ── 使用示例 ──────────────────────────────────────────────
if __name__ == "__main__":
    ctx = ContextBuilder(max_tokens=128_000)
    ctx.add(ContextSource.SYSTEM, "你是资深数据分析师 Agent...", priority=10)
    ctx.add(ContextSource.KNOWLEDGE, "[文档] Q3 营收同比增长 23%...", priority=8)
    ctx.add(ContextSource.HISTORY, "帮我分析上季度销售数据", priority=7,
            metadata={"role": "user", "turn_index": 0})
    ctx.add(ContextSource.TOOL_RESULT, "SQL 结果：| 7月 | 520万 | +15% |...", priority=9)
    messages = ctx.build()  # 自动按预算裁剪、按优先级排序
    print(f"组装完成，共 {len(messages)} 条 messages")
```

### Agentic 场景的特殊挑战

在 Agent 多轮执行过程中，上下文管理面临独特挑战：

```
Agent 执行循环中的上下文膨胀问题：

Turn 1:  System(2K) + User(0.5K)                          = 2.5K tokens
Turn 3:  System(2K) + History(3K) + ToolResult(5K)         = 10K tokens
Turn 7:  System(2K) + History(12K) + ToolResults(25K)      = 39K tokens
Turn 12: System(2K) + History(28K) + ToolResults(60K)      = 90K tokens  ⚠️
Turn 15: System(2K) + History(40K) + ToolResults(85K)      = 127K tokens 💥
                                                             ↑ 接近窗口上限

常见应对策略：
┌────────────────────────────────────────────┐
│ 策略 1：滑动窗口（Sliding Window）           │
│   只保留最近 N 轮对话，丢弃早期历史           │
│                                            │
│ 策略 2：摘要压缩（Summarization）            │
│   用 LLM 将早期对话压缩为摘要                │
│                                            │
│ 策略 3：工具结果裁剪（Result Pruning）        │
│   只保留工具结果的关键字段，丢弃原始数据       │
│                                            │
│ 策略 4：分层记忆（Hierarchical Memory）       │
│   短期记忆 → 工作记忆 → 长期记忆分级存储      │
└────────────────────────────────────────────┘
```

在 Multi-Agent 系统中，挑战进一步放大：每个 Agent 有独立的上下文窗口，Agent 之间的信息传递需要精心设计——传太多导致 token 浪费，传太少导致信息丢失。Context Engineering 在此扮演的是**信息路由器**的角色，决定哪些信息传递给哪个 Agent、以什么粒度传递。

### Context Engineering 的行业共识：Anthropic 官方工程指南

2026 年，Context Engineering 已从"新概念"成长为**行业共识与独立工程学科**。Anthropic 发布的官方工程指南将其定义为"为 LLM 的上下文窗口填入恰好够用的信息"的精密工程，核心是把上下文窗口视为一个**需要持续维护的有限资源**，而非静态容器。

```
Context Engineering 的信息生命周期（Anthropic 框架）

  写入上下文           管理上下文           跨会话持久化
  (Write)             (Manage)            (Agentic Memory)
     │                    │                     │
     ▼                    ▼                     ▼
 ┌─────────┐       ┌───────────┐        ┌──────────────┐
 │ 检索 RAG │       │ 压缩 / 摘要 │        │ 文件式记忆    │
 │ 工具结果 │ ────► │ 裁剪 / 截断│ ────►  │ 写笔记到窗口外 │
 │ 系统提示 │       │ 滑动窗口   │        │ 下次会话读回   │
 └─────────┘       └───────────┘        └──────────────┘
                          │
                          ▼
                   上下文窗口 ≠ 记忆
                   它是"工作台"，不是"档案柜"
```

三个关键认知升级：

1. **上下文窗口 ≠ 记忆**。窗口是 Agent 的"工作台"（scratchpad），用完即清；长期信息必须**外化**到窗口之外的文件、数据库、知识图谱。这与 [041 — 上下文窗口管理与压缩](../05-memory-and-state/041-context-window-management.md) 讨论的压缩策略一脉相承。
2. **Agentic Memory**：Agent 定期把学到的经验写成笔记（如 `.md` 文件），持久化到上下文窗口外，下次会话开始时读回。这把"记忆"从一次性 token 变成可累积的资产（详见 [043 — 持久化记忆](../05-memory-and-state/043-persistent-memory.md)）。
3. **信息衰减是常态**。长程 Agent 运行中，早期轮次的工具结果、已完成子任务的细节会从"有用"变成"噪声"。Context Engineering 要求主动清理（context editing / compaction），而非被动等待窗口溢出。

> **参考资料**：[Effective Context Engineering for AI Agents (Anthropic 官方工程指南)](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

### Prompt Caching：上下文缓存的经济学

当 Agent 在多轮执行中反复携带相同的 System Prompt、工具定义、知识库前缀时，**Prompt Caching**（上下文缓存）能把这些稳定前缀的计算结果缓存起来，命中时以极低成本读取——这是 Context Engineering 在"成本维度"的关键杠杆，也是长程 Agent 能"烧得起 token"的根本原因。

#### 核心机制：前缀匹配

```
Prompt 渲染顺序（Anthropic）：tools → system → messages

  ┌─ tools ──────────────┐  位置 0（最稳定）
  │ 工具定义（确定性排序） │
  ├─ system ─────────────┤
  │ System Prompt（冻结） │  ◄── cache_control 断点①
  ├─ messages ───────────┤
  │ 对话历史（逐轮增长）   │  ◄── cache_control 断点②（最新轮）
  │ ...                  │
  │ 当前用户问题（易变）   │  位置 N（最不稳定）
  └──────────────────────┘

缓存命中 = 从 prompt 开头到断点的每一个字节完全一致
任何一个字节的改变 → 该断点之后的所有缓存全部失效
```

#### Anthropic vs OpenAI 缓存机制对比

| 维度 | Anthropic Prompt Caching | OpenAI Prompt Caching |
|------|--------------------------|----------------------|
| 触发方式 | 手动 `cache_control` 断点 / 顶层自动缓存 | **全自动**（无需标记，系统自动识别高频前缀） |
| 最小缓存前缀 | 1024–4096 tokens（因模型而异） | 1024 tokens |
| 命中读取折扣 | ~90% off（读取价 ≈ 0.1× 基础输入价） | ~50% off（读取价 ≈ 0.5× 基础输入价） |
| 写入溢价 | 5 分钟 TTL：1.25×；1 小时 TTL：2× | 无额外写入溢价 |
| TTL（存活时间） | 5 分钟（默认）/ 1 小时（可选） | 约 5–10 分钟（系统管理，不可配置） |
| 断点上限 | 4 个 / 请求 | 不适用（自动管理） |
| 命中验证 | `usage.cache_read_input_tokens` | `usage.prompt_tokens_details.cached_tokens` |

> ⚠️ 上述折扣率与 TTL 为公开 API 文档截至 2026.7 的参数；各厂商可能随定价调整，使用前以官方文档为准。

#### 成本/延迟收益量化

以一个携带 20K token System Prompt + 10K token 知识库前缀的 Agent 为例（Anthropic）：

```
                    无缓存         有缓存（命中后）
输入 token          30,000         30,000（其中 ~28,000 命中缓存）
输入成本（相对值）   1.0×           ≈ 0.19×  (28000×0.1 + 2000×1) / 30000
首字延迟（TTFT）    基线           显著降低（跳过前缀的 prefill 计算）

盈亏平衡点：
  · 5 分钟 TTL：2 次请求即回本 (1.25× + 0.1× = 1.35×  vs  2× 无缓存)
  · 1 小时 TTL：至少需 3 次请求  (2× + 0.2× = 2.2×  vs  3× 无缓存)
```

对于 Agentic 场景（同一会话内 10+ 轮调用，每轮都携带相同 System Prompt + 累积历史），缓存命中率通常可达 **80%+**——没有 Prompt Caching，多轮 Agent 的 token 成本将高到难以承受。

#### 缓存失效管理：静默杀手

缓存失效**不报错**——`cache_read_input_tokens` 静默归零，你只会在账单上发现成本飙升。常见"静默失效"根因：

| 反模式 | 为什么破坏缓存 |
|--------|---------------|
| System Prompt 内插 `datetime.now()` / 时间戳 | 每次请求前缀都不同 |
| `json.dumps(d)` 未加 `sort_keys=True` | 序列化顺序不确定 → 字节级差异 |
| 工具列表按用户动态增减 / 顺序随机 | tools 在位置 0，一处变动全盘失效 |
| System Prompt 内插 `uuid4()` / 请求 ID | 每次请求唯一 |
| 中途切换模型 | 缓存按模型隔离，换模型 = 冷启动 |

**工程实践**：把动态内容（时间、用户 ID、会话状态）放到 `messages` 末尾，**绝不要**插进 System Prompt；工具列表按 `name` 确定性排序；构建 prompt 后用 `count_tokens` 比对前后两次请求的字节差异来排查静默失效。

#### Agent 多轮场景的断点策略

```python
# Anthropic: 冻结前缀 + 最新轮各放一个断点
# 每次请求复用之前所有轮次的前缀缓存
response = client.messages.create(
    model="claude-sonnet-5",
    max_tokens=4096,
    system=[{
        "type": "text",
        "text": STABLE_SYSTEM_PROMPT,
        "cache_control": {"type": "ephemeral"},   # 断点①：冻结前缀
    }],
    messages=[
        # ... 历史对话（前缀已被断点①缓存）...
        {"role": "user", "content": [
            {"type": "text", "text": latest_tool_result},  # 累积历史
        ]},
        {"role": "user", "content": new_question},          # 易变，放最后
    ],
    extra_headers={"cache_control": "ephemeral"},  # 顶层自动在最新 block 放断点②
)

# 响应中验证命中：>0 即命中
print(f"缓存读取: {response.usage.cache_read_input_tokens} tokens")
print(f"缓存写入: {response.usage.cache_creation_input_tokens} tokens")
```

> **进阶陷阱**：单个断点最多回溯 **20 个 content block** 找前一次缓存。长 Agent 轮次（一轮内大量 tool_use/tool_result）若超过 20 个 block，下一个断点会静默 miss——需每 ~15 个 block 补一个中间断点。冷启动场景可用 `max_tokens=0` 的空请求**预热**缓存，消除首次请求的 prefill 延迟。

## 常见误区 / 面试追问

1. **误区："Context Engineering 就是写更好的 Prompt"** — 这是最常见的混淆。Prompt Engineering 是 Context Engineering 的一个子集。Context Engineering 不仅关注 Prompt 本身，更关注 Prompt 之外的所有信息——工具返回值如何裁剪、对话历史如何压缩、外部知识如何检索与排序。类比来说，Prompt Engineering 是写好一封邮件，Context Engineering 是管理整个项目的信息流。

2. **误区："上下文越多越好，反正模型支持 128K"** — 研究表明 LLM 存在"Lost in the Middle"现象：当上下文过长时，模型对中间位置信息的注意力显著下降。此外，无关上下文会稀释模型对关键信息的注意力，降低输出质量。正确的做法是追求**高信噪比**而非高信息量——每一段上下文都应该对当前任务有明确的边际贡献。

3. **追问："如何衡量上下文质量？"** — 可以从三个维度评估：(1) **相关性**——上下文中每段信息与当前任务的相关度（可用 embedding 相似度量化）；(2) **充分性**——是否包含完成任务所需的全部关键信息（通过 ablation test 验证）；(3) **效率**——信息密度，即有效 token 占总 token 的比例。实践中常用 A/B 测试对比不同上下文策略对最终输出质量的影响。

4. **追问："Context Engineering 在 Multi-Agent 中如何应用？"** — 在 Multi-Agent 架构中，Context Engineering 承担三重角色：(1) **上下文隔离**——每个 Agent 只接收与其职责相关的上下文，避免信息过载；(2) **上下文传递**——设计 Agent 之间的信息传递协议，决定传递摘要还是原始数据；(3) **共享上下文管理**——维护全局状态（如 Blackboard 模式），让多个 Agent 读写共享的结构化上下文，而非互相透传完整历史。

## 参考资料

- [Context Engineering for AI Agents — Lessons from Building Real Systems (Philipp Schmid)](https://www.philschmid.de/context-engineering)
- [Prompt Design != Context Engineering (Torantulino / LangChain Blog)](https://blog.langchain.dev/context-engineering/)
- [Lost in the Middle: How Language Models Use Long Contexts (Nelson Liu et al., 2023)](https://arxiv.org/abs/2307.03172)
- [Effective Context Engineering for AI Agents (Anthropic 官方工程指南)](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — Context Engineering 作为独立工程学科的系统化框架、Agentic Memory 与信息生命周期
- [Prompt Caching (Anthropic 官方文档)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching) — cache_control 机制、断点放置、静默失效排查清单
- [Building Effective Agents (Anthropic)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview)
- [The Shift from Prompt Engineering to Context Engineering (Andrej Karpathy)](https://x.com/karpathy/status/1937902205765607918)
