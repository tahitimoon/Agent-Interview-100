# 结构化输出（Structured Output）：如何让 LLM 返回 JSON/XML？

> 难度：基础
> 分类：Prompt Engineering

## 简短回答

结构化输出是让 LLM 返回机器可解析格式（JSON、XML 等）而非自由文本的技术，是 Agent 系统和数据管道的基础能力。主要实现方法有四种：(1) **Prompt 指令**——在 Prompt 中描述期望格式，最简单但不保证合规；(2) **JSON Mode**——API 级别强制输出合法 JSON，保证语法正确但不保证 Schema 匹配；(3) **Function Calling / Tool Use**——定义函数签名让 LLM 填参数，最适合 Agent 工具调用；(4) **Constrained Decoding**——在解码层面约束 token 生成，100% 保证格式合规（如 Outlines、Guidance）。生产环境推荐 **Function Calling + Pydantic 验证**：Function Calling 处理格式，Pydantic 验证语义。OpenAI 自 openai-python 1.92 起 `client.chat.completions.parse` 已 GA，Responses API 提供 `client.responses.parse` 作为新项目首选；**Anthropic 也已于 2025-11-14 公开 Beta 原生 Structured Outputs**（`anthropic-beta: structured-outputs-2025-11-13` + `client.messages.parse`），底层 constrained decoding 100% Schema 合规——这是 2026 面试的核心新原语。

## 详细解析

### 方法 1：Prompt 指令（最简单）

```python
# 通过 Prompt 要求输出 JSON
prompt = """
从以下文本中提取实体信息，以 JSON 格式输出。

文本：张三是 ABC 公司的技术总监，他在北京工作了 5 年。

输出格式：
{
    "name": "人名",
    "company": "公司名",
    "title": "职位",
    "location": "工作地点",
    "years": 数字
}

请只输出 JSON，不要添加任何其他文本。
"""

# 问题：
# - LLM 可能添加 "```json" 标记或解释文字
# - 可能输出不合法的 JSON（缺少引号、多余逗号）
# - 需要后处理和错误处理
```

### 方法 2：JSON Mode（API 级别）

```python
# OpenAI JSON Mode
from openai import OpenAI
client = OpenAI()

response = client.chat.completions.create(
    model="gpt-4o",
    response_format={"type": "json_object"},  # 强制 JSON 输出
    messages=[
        {"role": "system", "content": "你是一个数据提取助手。始终以 JSON 格式回答。"},
        {"role": "user", "content": "提取：张三是 ABC 公司的技术总监"}
    ]
)

# 保证输出是合法的 JSON
data = json.loads(response.choices[0].message.content)

# 注意：JSON Mode 保证语法合法，但不保证 Schema 匹配
# 可能返回 {"person": "张三"} 而不是 {"name": "张三"}
```

### 方法 3：Function Calling / Structured Outputs

```python
from pydantic import BaseModel

# 用 Pydantic 定义 Schema（推荐方式）
class PersonInfo(BaseModel):
    name: str
    company: str
    title: str
    location: str
    years: int

# OpenAI Structured Outputs（openai-python>=1.92，parse 已 GA）
response = client.chat.completions.parse(
    model="gpt-4o",
    response_format=PersonInfo,  # 直接传 Pydantic 模型
    messages=[
        {"role": "user", "content": "提取：张三是 ABC 公司的技术总监，在北京工作了5年"}
    ]
)

person = response.choices[0].message.parsed
# person.name == "张三"
# person.company == "ABC"
# person.years == 5
# 类型安全，100% Schema 合规

# 注：2024-08 由 client.beta.chat.completions.parse 提升为 GA（openai-python 1.92+）
#     旧路径 client.beta.* 仍可用，但新代码推荐 client.chat.completions.parse

# OpenAI Responses API（新项目首选，2025-03 发布）
# 把 Chat Completions + Assistants 合并为一个新 endpoint
response = client.responses.parse(
    model="gpt-4o",
    text_format=PersonInfo,
    input=[{"role": "user", "content": "提取：张三是 ABC 公司的技术总监"}]
)
person = response.output_parsed

# Anthropic 原生 Structured Outputs（公开 Beta，2025-11-14 发布）
# 需要 anthropic-beta header，底层是 constrained decoding，100% Schema 合规
response = anthropic_client.messages.parse(
    model="claude-sonnet-4-5",
    max_tokens=1024,
    response_format=PersonInfo,           # 直接传 Pydantic 模型
    extra_headers={
        "anthropic-beta": "structured-outputs-2025-11-13"
    },
    messages=[
        {"role": "user", "content": "提取：张三是 ABC 公司的技术总监"}
    ]
)
person = response.parsed                  # 直接拿到 Pydantic 实例

# 兜底方案：Anthropic Tool Use（在原生 SO Beta 之前的事实标准，仍适用于无 Beta 权限场景）
response = anthropic_client.messages.create(
    model="claude-sonnet-4-5",
    tools=[{
        "name": "extract_person",
        "description": "提取人物信息",
        "input_schema": PersonInfo.model_json_schema()
    }],
    tool_choice={"type": "tool", "name": "extract_person"},
    messages=[{"role": "user", "content": "提取：张三是 ABC 公司的技术总监"}]
)
```

### 方法 4：Constrained Decoding（底层约束）

```python
# Outlines 框架：在 token 级别约束输出
# 新版 API（outlines >= 1.0）：from_transformers / from_vllm 等装载器
import outlines
from transformers import AutoModelForCausalLM, AutoTokenizer

base = AutoModelForCausalLM.from_pretrained("Qwen/Qwen3-7B-Instruct")
tok  = AutoTokenizer.from_pretrained("Qwen/Qwen3-7B-Instruct")
model = outlines.from_transformers(base, tok)

# 用 Pydantic / JSON Schema 约束生成
class Person(BaseModel):
    name: str
    age: int
    skills: list[str]

result = model("提取信息：张三,25岁,会Python和Java", Person)
# 100% 保证输出符合 Schema
# 原理：在每步 token 采样时，只允许合法的下一个 token
# 注：旧版 outlines.generate.json + Llama-3 8B 已被 from_transformers 替代
```

### 方法对比

| 方法 | 格式保证 | Schema 保证 | 实现难度 | 适用场景 |
|------|----------|------------|----------|----------|
| Prompt 指令 | ✗ 不保证 | ✗ 不保证 | 最简单 | 原型验证 |
| JSON Mode | ✓ 保证 | ✗ 不保证 | 简单 | 简单场景 |
| Function Calling | ✓ 保证 | ✓ 保证 | 中等 | Agent |
| Structured Outputs | ✓ 保证 | ✓ 保证 | 中等 | 生产环境 |
| Constrained Decode | ✓ 保证 | ✓ 保证 | 复杂 | 本地模型 |

### 生产环境最佳实践

```python
from pydantic import BaseModel, Field, field_validator

class ExtractedData(BaseModel):
    """用 Pydantic 做双重保障：格式 + 语义验证"""
    name: str = Field(..., min_length=1, description="人名")
    age: int = Field(..., ge=0, le=150, description="年龄")
    email: str | None = Field(None, description="邮箱")

    @field_validator("email")
    @classmethod
    def validate_email(cls, v):
        if v and "@" not in v:
            raise ValueError("邮箱格式不正确")
        return v

async def extract_with_retry(text: str, max_retries=3) -> ExtractedData:
    """带重试的结构化输出提取（openai-python>=1.92，parse 已 GA）"""
    for attempt in range(max_retries):
        try:
            response = await client.chat.completions.parse(
                model="gpt-4o",
                response_format=ExtractedData,
                messages=[{"role": "user", "content": f"提取信息：{text}"}]
            )
            return response.choices[0].message.parsed
        except ValidationError as e:
            if attempt == max_retries - 1:
                raise
            # 将验证错误反馈给模型重试
            continue
```

### Agent 系统中的结构化输出

```python
# Agent 的工具调用就是结构化输出的典型应用
tools = [
    {
        "name": "search_web",
        "description": "搜索网页信息",
        "parameters": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "搜索关键词"},
                "max_results": {"type": "integer", "default": 5}
            },
            "required": ["query"]
        }
    }
]

# LLM 输出结构化的工具调用参数
# {"name": "search_web", "arguments": {"query": "Python 最佳实践", "max_results": 3}}
# Agent 框架解析后直接调用对应函数
```

## 常见误区 / 面试追问

1. **误区："在 Prompt 里说'输出 JSON'就够了"** — Prompt 指令无法保证输出是合法 JSON。LLM 可能添加 markdown 标记、额外解释、或生成语法错误的 JSON。生产环境必须用 JSON Mode 或 Function Calling。

2. **误区："JSON Mode 能保证字段完整"** — JSON Mode 只保证输出是合法 JSON，不保证包含你需要的字段。需要用 Structured Outputs（传入 JSON Schema 或 Pydantic 模型）来保证 Schema 合规。

3. **追问："Structured Output 会影响模型的生成质量吗？"** — 会有轻微影响。格式约束限制了模型的自由度，在某些创意任务上可能降低质量。但对于数据提取、工具调用等结构化任务，约束反而提升了准确率。

4. **追问："如何处理嵌套和复杂的输出结构？"** — Pydantic 支持嵌套模型、列表、可选字段、枚举等复杂结构。关键是 Schema 设计要清晰——字段名用语义化命名，description 写清楚每个字段的含义，帮助 LLM 理解期望。

## 参考资料

- [Structured Model Outputs (OpenAI API Docs)](https://developers.openai.com/api/docs/guides/structured-outputs/)
- [Introducing Structured Outputs (Anthropic, 2025-11-14)](https://www.anthropic.com/news/structured-outputs)
- [Anthropic API: Structured Outputs Beta Reference](https://docs.anthropic.com/en/docs/build-with-claude/structured-outputs)
- [OpenAI Responses API Reference](https://platform.openai.com/docs/api-reference/responses)
- [The Guide to Structured Outputs and Function Calling with LLMs (Agenta)](https://agenta.ai/blog/the-guide-to-structured-outputs-and-function-calling-with-llms)
- [Prompting vs JSON Mode vs Function Calling vs Constrained Decoding (BoundaryML)](https://boundaryml.com/blog/schema-aligned-parsing)
- [LLM Structured Outputs: The Silent Hero of Production AI (DecodingAI)](https://www.decodingai.com/p/llm-structured-outputs-the-only-way)
- [Structured Outputs in LLMs: Definition, Techniques, Applications (LeewayHertz)](https://www.leewayhertz.com/structured-outputs-in-llms/)
