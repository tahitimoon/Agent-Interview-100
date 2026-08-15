# Embedding 模型选择与微调策略

> 难度：中级
> 分类：RAG

## 简短回答

Embedding 模型的选择直接决定 RAG 检索质量。2025-2026 年的格局：**Voyage AI voyage-3-large** 与 **Cohere embed-v4** 在 MTEB Benchmark 上排名相近，分别在性价比与多语言/多模态场景占优（具体名次取决于子任务）；**OpenAI text-embedding-3-large** 是最均衡的生产默认选择。开源方面 **BGE-M3** 和 **Qwen3 Embedding** 表现出色。微调可带来 10-30% 的领域特定提升，但需要注意重新索引的成本。

## 详细解析

### Embedding 模型的作用

Embedding 模型将文本转化为高维向量，使得语义相似的文本在向量空间中距离更近。在 RAG 中，它用于两个环节：
1. **索引时**：将文档块转为向量存入向量数据库
2. **查询时**：将用户问题转为向量，与文档向量做相似度匹配

**关键约束：** 索引和查询必须使用**同一个** Embedding 模型，否则向量空间不一致，检索完全失效。

### 主流商业模型对比

| 模型 | 维度 | MTEB 表现 | 价格($/M tokens) | 特色 |
|------|------|-----------|-------------------|------|
| **Voyage AI voyage-3-large** | 1024 | 综合前列 | $0.06 | 性价比最高，多领域领先 |
| **OpenAI text-embedding-3-large** | 3072(可缩) | 前列 | $0.13 | 最均衡，支持维度缩减 |
| **Cohere embed-v4** | 1536 | 多语言/多模态前列 | ~$0.10 | 100+ 语言，多模态 |
| **OpenAI text-embedding-3-small** | 1536 | 中上 | $0.02 | 最便宜，适合成本敏感场景 |

#### Voyage AI — 性价比之王

Voyage AI 的 voyage-3-large 在多项 MTEB 子任务上超越 OpenAI text-embedding-3-large 与 Cohere embed-v3（具体提升幅度因数据集而异）。1024 维 Embedding 比 OpenAI 的 3072 维节省约 3 倍存储空间，价格仅为 OpenAI 的一半。

特别值得注意：Voyage 提供领域特化模型（截至 2026-05 的最新版）：
- `voyage-law-3`：法律领域（案例检索精度大幅领先通用模型）
- `voyage-code-3`：代码搜索
- `voyage-finance-2`：金融领域（v3 尚未发布）

#### OpenAI text-embedding-3-large — 最均衡的默认选择

Agentset 的评测显示它在更多的直接对比中胜出，是最稳定的生产选择。独特优势：支持通过 `dimensions` 参数缩减维度（3072→1536→256），在精度和存储间灵活权衡。

```python
from openai import OpenAI
client = OpenAI()

# 全维度：最高精度
full = client.embeddings.create(
    model="text-embedding-3-large",
    input="查询文本"
)  # 3072 维

# 缩减维度：节省存储，轻微精度损失
compact = client.embeddings.create(
    model="text-embedding-3-large",
    input="查询文本",
    dimensions=1024  # 从 3072 缩减到 1024
)
```

#### Cohere embed-v4 — 多语言多模态之王

在 MTEB 多语言子任务上排名前列，支持 100+ 语言的跨语言检索，能在同一语义空间中嵌入文本和图像。支持 128K token 的超长输入。

### 主流开源模型

| 模型 | 维度 | 许可证 | 特色 |
|------|------|--------|------|
| **BAAI/bge-m3** | 1024 | MIT | 多语言，自托管首选 |
| **Qwen3 Embedding (4B/8B)** | 可变 | Apache 2.0 | 最新 SOTA，32K 上下文 |
| **Jina Embeddings v3** | 1024 | Apache 2.0 | 多语言，任务特定适配 |
| **all-MiniLM-L6-v2** | 384 | Apache 2.0 | 极轻量，MVP 首选 |

**选择开源模型的理由：**
- 数据隐私：不需要将文档发送到第三方 API
- 成本控制：大量数据时自托管更经济
- 离线部署：无网络环境
- 完全可控：可以微调和定制

### 微调策略

领域特定微调可带来 **10-30%** 的检索质量提升。

#### 何时需要微调？

```mermaid
flowchart TD
    A{"通用 Embedding<br/>在你的数据上满意？"}
    A -->|"满意"| B["不需要微调"]
    A -->|"不满意"| C{"原因是什么？"}
    C -->|"数据质量问题"| D["先优化数据，不是模型"]
    C -->|"领域术语不理解"| E["微调"]
    C -->|"检索逻辑需要定制"| F["微调"]
    classDef decision fill:#fffde7,stroke:#f9a825,color:#795548
    classDef proc fill:#e3f2fd,stroke:#1565c0,color:#0d47a1
    class A,C decision
    class B,D,E,F proc
```
*微调前先测通用模型：不满意先区分原因——数据质量问题先修数据，领域术语或检索逻辑不匹配才值得微调。*

#### 微调方法

```python
# 1. 准备训练数据：(query, positive_doc, negative_doc) 三元组
training_data = [
    {
        "query": "RLHF 的奖励模型如何训练？",
        "positive": "奖励模型通过人类偏好标注数据训练...",
        "negative": "强化学习是一种机器学习方法..."
    },
    # ... 至少 1000 个样本
]

# 2. 使用对比学习进行微调
from sentence_transformers import SentenceTransformer, losses

model = SentenceTransformer("BAAI/bge-base-en-v1.5")
train_loss = losses.TripletLoss(model)
model.fit(
    train_objectives=[(train_dataloader, train_loss)],
    epochs=3
)
```

#### 微调的关键注意事项

1. **必须重新索引**：微调改变了 Embedding 空间，所有已索引的向量都需要重新生成。这是不可忽略的成本。
2. **数据量要求**：至少 1000 个标注样本，理想情况 5000-10000 个
3. **评估闭环**：用 Recall@k、MRR 等指标对比微调前后的效果
4. **基础模型选择**：选与你的领域最接近的基础模型微调，效果更好

### 选型决策框架

```python
def choose_embedding_model(requirements: dict) -> str:
    if requirements["stage"] == "mvp":
        return "all-MiniLM-L6-v2"        # 免费、快、够用

    if requirements["multilingual"]:
        return "cohere-embed-v4"          # 100+ 语言

    if requirements["privacy_critical"]:
        return "bge-m3 (self-hosted)"     # MIT 许可，完全可控

    if requirements["budget"] == "tight":
        return "voyage-3-large"           # 最高性价比

    if requirements["domain_specific"]:
        if requirements["domain"] in ["legal", "code", "finance"]:
            return f"voyage-{domain}-2"   # 领域特化模型
        else:
            return "fine-tune bge-m3"     # 自定义微调

    return "openai-text-embedding-3-large"  # 最均衡的默认选择
```

### 关键原则

> "在你的数据上做 Benchmark——通用分数不一定适用于你的领域。"

> "价格更高的模型不一定准确率更高。最佳模型是在准确率和成本之间取得最好平衡的模型。"

## 常见误区 / 面试追问

1. **误区："选 MTEB 排名第一的就对了"** — MTEB 是通用基准，你的领域数据分布可能完全不同。必须在自己的数据集上评估。

2. **误区："维度越高越好"** — 3072 维比 1024 维精度更高，但存储 3 倍、检索更慢。很多场景下 1024 维甚至 768 维就足够了。OpenAI 的维度缩减功能正是为此设计。

3. **追问："如果换了 Embedding 模型怎么办？"** — 必须重新索引所有文档。索引和查询必须使用完全相同的模型。这是换模型的最大成本。

4. **追问："Embedding 和 LLM 需要来自同一家吗？"** — 不需要。Embedding 模型和 LLM 是独立组件。可以用 Voyage 的 Embedding + Anthropic 的 Claude 生成，完全没问题。

## 参考资料

- [Best Embedding Models 2025: MTEB Scores & Leaderboard (Ailog)](https://app.ailog.fr/en/blog/guides/choosing-embedding-models)
- [Top Embedding Models 2026: Complete Guide (ArtSmart)](https://artsmart.ai/blog/top-embedding-models-in-2025/)
- [Embedding Models Comparison 2026: OpenAI vs Cohere vs Voyage vs BGE (Reintech)](https://reintech.io/blog/embedding-models-comparison-2026-openai-cohere-voyage-bge)
- [9 Best Embedding Models for RAG (ZenML)](https://www.zenml.io/blog/best-embedding-models-for-rag)
- [Embedding Model Leaderboard (Agentset)](https://agentset.ai/embeddings)
