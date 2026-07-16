# 什么是 12-Factor Agents？如何用它评审生产级 Agent？

> 难度：中级
> 分类：生产部署

## 简短回答

**12-Factor Agents** 是 HumanLayer 创始人 Dex Horthy 于 2025 年提出的一套「生产级 LLM 应用设计原则」，直接致敬 2011 年 Heroku 团队的经典 **12-Factor App**。项目开源在 `humanlayer/12-factor-agents`，目前在 GitHub 收获约 **2.4 万 star**，是 Agent 工程化领域传播最广的方法论文档之一。它的核心论断是：**真正能在生产环境交付给客户的 "AI Agent"，绝大多数并不是「给个 prompt + 一袋工具 + 死循环到目标」的魔幻自治体，而是「以 LLM 决策为节点、以确定性代码为边」的软件系统**。

这套方法论回答一个工程问题：*怎样构建「好到可以交付给生产客户」的 LLM 驱动软件？* 作者调研了上百位 SaaS 创业者后发现一条高度一致的路径——**抓一个框架快速冲到 70~80% 质量分 → 发现 80% 对面向客户的功能远远不够 → 想突破必须逆向工程框架的 prompt/流程/状态 → 最终推倒重写**。12-Factor Agents 的立论正是：与其 all-in 某个框架，不如把 Agent 的核心能力拆成 12 条可独立采纳的模块化原则，融进你现有的产品代码里。

12 条原则可以浓缩为一句话：**把 Agent 当作「LLM 决定下一步 + 确定性代码执行 + 结果回灌上下文」的循环，并把这个循环里的每一个关键环节（prompt、上下文、状态、控制流、错误、人工介入）都显式地「收归己有」**。它既是一份反模式清单，也是一张可逐条打勾的生产化评审表。面试中它常作为「你怎么评审一个 Agent 架构」的标准答案出现。

## 详细解析

### 核心理念：Agent 本质上是一个「确定性外壳 + LLM 决策内核」的循环

12-Factor Agents 的起点是对「Agent 是什么」的重新定义。作者指出，业界宣传的 Agent 往往是「丢掉 DAG（有向无环图），让 LLM 实时决定路径」的乌托邦；但真正跑通的 Agent，骨子里是一个稳定的循环：

```python
# 12-Factor Agents 的「最小 Agent 循环」（来自官方 README）
initial_event = {"message": "..."}
context = [initial_event]

while True:
    # 1. LLM 根据当前上下文，决定下一步（输出结构化的 tool call）
    next_step = await llm.determine_next_step(context)
    context.append(next_step)

    # 2. 如果 LLM 认为做完了，返回最终答案
    if next_step.intent == "done":
        return next_step.final_answer

    # 3. 确定性代码执行这一步，把结果追加进上下文
    result = await execute_step(next_step)
    context.append(result)
```

这个循环里只有三件事：**LLM 决定下一步 → 代码执行 → 结果进上下文 → 重复**。12 条原则中的每一条，都是在加固这个循环的某一个面——让 prompt 可控、让上下文可管、让状态可恢复、让错误可自愈、让人工可介入。理解了这个循环，就理解了 12-Factor Agents 的全部锚点。

### 12 条原则逐条解读

下表先给出全景，随后对易误解的几条展开说明。

| # | 原则（英文原名） | 一句话释义 | 它解决的反模式 |
|---|---|---|---|
| 1 | **Natural Language to Tool Calls** | 用 LLM 把自然语言意图转成结构化的「下一步动作」 | 把决策逻辑硬编码进 if/else，丧失灵活性 |
| 2 | **Own Your Prompts** | Prompt 是一等代码，版本化、可测试、可迭代，不外包给框架黑盒 | 框架把 prompt 藏起来，出问题无法定位与调优 |
| 3 | **Own Your Context Window** | 显式管理每一轮进入上下文的内容（即 Context Engineering） | 上下文无限膨胀，关键信息被淹没，Agent 「失焦」 |
| 4 | **Tools Are Just Structured Outputs** | 工具调用 = LLM 的原生结构化输出（JSON / Pydantic），而非正则解析自由文本 | 用正则/字符串匹配抽取 LLM 输出，脆弱且易错 |
| 5 | **Unify Execution State and Business State** | 把「执行态」（loop 进度）与「业务态」（业务结果）统一成一条事件流 | 内存里有 loop 状态、数据库里有业务状态，重启即丢失或错乱 |
| 6 | **Launch/Pause/Resume with Simple APIs** | Agent 像普通程序一样，能用简单 API 启动、暂停、恢复、停止 | 长任务只能 `while...sleep` 阻塞挂起，进程挂了就从头再来 |
| 7 | **Contact Humans with Tool Calls** | 把「请求人工确认/澄清」也建模成一种工具调用（HITL as a tool） | 高风险操作只能「yolo 盲跑」或被迫降级为低风险只读任务 |
| 8 | **Own Your Control Flow** | 自己掌控循环的 break/continue：何时等审批、何时压缩上下文、何时熔断 | 把控制流交给框架，无法在「工具选中」与「工具执行」之间插入审批 |
| 9 | **Compact Errors into Context Window** | 工具失败时把错误信息压缩后回灌上下文，让 LLM 自愈 | 工具一报错整个 Agent 直接崩溃退出 |
| 10 | **Small, Focused Agents** | 单个 Agent 控制在 3~10（至多 20）步以内，做小做专 | 一个巨型 Agent 试图扛 100 步任务，上下文爆炸、反复跑偏 |
| 11 | **Trigger from Anywhere** | Agent 能从 webhook、cron、IM 消息、前端事件等任意入口触发 | Agent 只能从聊天框触发，无法嵌入业务流 |
| 12 | **Make Your Agent a Stateless Reducer** | 把 Agent 建模成无状态的 reducer（fold）：`state' = reduce(event, state)` | 状态散落在多处，无法重放、难以水平扩展与恢复 |

> 附录还有一条「**Factor 13: Pre-fetch all the context you might need**」（预先抓取所有可能需要的上下文），作为荣誉条目，强调在 LLM 决策前由确定性代码先把上下文备齐，而不是让 LLM 边走边找。

几条容易踩坑的原则展开说明：

- **Factor 5（统一状态）** 是 12 条里工程含量最高的一条。它的核心是：不要让「loop 当前进度」这种**执行态**只活在内存变量里，而业务结果活在数据库里——两者一旦分离，进程崩溃就无法恢复。正确做法是把所有发生的事（用户消息、LLM 的 tool call、工具结果、人工审批）都作为**事件（event）**追加到同一条 `thread` 里，状态就是事件的折叠结果。这样「恢复」等价于「重新 replay 这条事件流」。

- **Factor 7（人工作为工具）** 与 Factor 8 是孪生兄弟。作者反复强调他「对每个 Agent 框架的头号需求」就是：**能在工具被「选中」和「被执行」之间插入一个暂停点**，让人审核。没有这个粒度，你就只能在「只让 Agent 做只读研究」和「给它高危权限然后祈祷」之间二选一。

- **Factor 12（无状态 reducer）** 听起来玄，本质就是把 Agent 写成纯函数：`new_state = agent(events_so_far, new_event)`。任何一个时刻，只要把事件流喂进去，就能还原出与原运行完全一致的 Agent 状态。这让 Agent 天然可重放、可恢复、可水平扩展——这也是「stateless reducer」这个名字直接借用前端 Redux/`Array.reduce` 范式的原因。

### 反模式 → 原则对照表

把 12 条反过来读，就是一份「Agent 翻车常见姿势」清单。下面给出工程实践中最常见的反模式及其对应原则：

| 反模式（典型翻车现场） | 对应原则 | 不遵守的后果 |
|---|---|---|
| 框架 `Agent(role, goal, personality).run(task)` 黑盒，prompt 看不见 | F2 Own Your Prompts | 质量卡在 80%，想突破必须逆向框架，最终重写 |
| 把 tool 描述和系统提示一股脑塞进去，从不裁剪 | F3 Own Your Context | 多轮后上下文膨胀，模型「Lost in the Middle」失焦 |
| 让 LLM 输出一段自然语言，再用正则抠出 `action:xxx` | F4 Structured Outputs | 模型换个说法就解析失败，链路脆弱 |
| loop 状态只在内存，DB 只存最终结果 | F5 Unify State | 进程一崩，跑了 30 分钟的任务彻底丢失 |
| 部署等长任务用 `while True: sleep(5)` 阻塞 | F6 Pause/Resume | 一个 Pod 重启 = 全部重来；无法横向扩展 |
| 危险操作（删库、打款）要么盲跑要么干脆不开放 | F7 + F8 | 要么出事故，要么 Agent 沦为只能查不能做的玩具 |
| 工具抛异常直接冒泡终止整个 Agent | F9 Compact Errors | 一次网络抖动就全盘失败，无自愈能力 |
| 一个「超级 Agent」端到端做完百步业务 | F10 Small Focused | 上下文爆炸、反复跑偏、难以调试 |
| Agent 只能从一个 Web 聊天框唤起 | F11 Trigger Anywhere | 无法嵌入 cron、webhook、IM 等真实业务流 |
| 状态散在内存/Redis/DB 三处，互相不同步 | F12 Stateless Reducer | 无法重放、无法 debug「为什么这步这么决策」 |

### 与经典 12-Factor App 的呼应与差异

12-Factor Agents 刻意沿用了「12 条」的形式与命名，但两者的**关注层完全不同**：12-Factor App（2011）面向 SaaS 云原生时代的**部署与可移植性**（代码库、依赖、配置、后端服务、并发、日志），而 12-Factor Agents（2025）面向 LLM 时代的**应用内循环设计**（prompt、上下文、状态、控制流）。可以这样对应理解：

| 12-Factor App（2011，云原生运维） | 12-Factor Agents（2025，Agent 应用设计） | 呼应关系 |
|---|---|---|
| I. Codebase：一份代码库多次部署 | F2 Own Your Prompts | prompt 也是 codebase，需要版本化 |
| III. Config：配置放环境变量，不进代码 | F3 Own Your Context Window | 上下文是「运行时配置」，需显式组装 |
| IV. Backing Services：把 DB/缓存当可替换资源 | F4 Tools as Structured Outputs | 工具是可替换的外部服务，通过结构化输出对接 |
| VI. Processes：无状态进程，状态外置 | F5 + F12 Unify State / Stateless Reducer | 都强调状态不留在进程内存，可重建 |
| IX. Disposability：可快速启动、优雅退出 | F6 Launch/Pause/Resume | 都追求进程可被随意杀掉再恢复 |
| XI. Logs：日志作为事件流 | F9 Compact Errors | 都把「事件」作为一等公民灌进流 |
| XII. Admin Processes：一次性管理进程 | F11 Trigger from Anywhere | 都强调进程能从任意入口按需启动 |

差异同样关键：原版 12-Factor App **不关心业务逻辑长什么样**，它假设你写的是普通确定性代码；而 12-Factor Agents **专门处理 LLM 引入的非确定性**——如何约束 LLM 的输出（F4）、如何在它跑偏时纠错（F8/F9）、如何在它「失忆」前缩小任务范围（F10）。可以认为：**12-Factor App 解决「软件怎么上云」，12-Factor Agents 解决「带 LLM 的软件怎么上生产」**，两者是互补而非替代关系。

### 落地方法：用 12 条评审自己的 Agent 架构

面试中如果被问「你会怎么评审一个 Agent 架构」，12 条本身就是现成的 checklist。下面给出一个可操作的评审流程，先看整体评审链路图：

```
┌─────────────────────────────────────────────────────────────────┐
│              12-Factor Agents 架构评审流程                        │
└─────────────────────────────────────────────────────────────────┘

  ① 先找「循环」      你的 Agent 是否有清晰的 LLM决策→执行→回灌循环？
       │              └─ 没有 → 先补 Factor 1（NL→Tool Call）
       ▼
  ② 审「可控性」      F2 Prompt / F3 Context / F4 Structured Output
       │              逐条问：prompt 在哪？上下文谁在裁？输出有 schema 吗？
       ▼
  ③ 审「可恢复性」    F5 统一状态 / F6 暂停恢复 / F12 无状态 reducer
       │              └─ kill -9 一次进程，能否从断点 resume？能否 replay？
       ▼
  ④ 审「可介入性」    F7 人工即工具 / F8 自控控制流
       │              └─ 工具「选中」和「执行」之间，能否插审批点？
       ▼
  ⑤ 审「鲁棒性」      F9 错误压缩 / F10 小而专注
       │              └─ 工具失败会自愈吗？单 Agent 步数是否 ≤ 20？
       ▼
  ⑥ 审「可触达性」    F11 触发入口 / F13 预取上下文
                      └─ 能否从 webhook/cron/IM 触发？上下文是否预取？
```

对应的最小可运行骨架（伪代码，体现 F1/F4/F5/F8/F9 几条核心原则）：

```python
from pydantic import BaseModel
from typing import Literal, Union

# ── F4: Tools as Structured Outputs ──
# 把「下一步」定义成带 schema 的结构化输出，而非自由文本
class Done(BaseModel):
    intent: Literal["done"]
    final_answer: str

class DeployBackend(BaseModel):
    intent: Literal["deploy_backend"]
    tag: str
    env: Literal["staging", "production"]

class RequestApproval(BaseModel):
    intent: Literal["request_approval"]
    reason: str
    risky_action: str

NextStep = Union[Done, DeployBackend, RequestApproval]

# ── F5: 统一执行态与业务态为一条事件流 ──
thread = {"events": [initial_event]}

consecutive_errors = 0  # ── F9: 错误计数，防止无限自愈打转 ──

while True:
    # F1 + F2: LLM 把自然语言决策成结构化的「下一步」，prompt 是你自己的
    next_step: NextStep = await determine_next_step(thread)
    thread["events"].append({"type": next_step.intent, "data": next_step})

    if isinstance(next_step, Done):
        break  # 任务完成

    # ── F8: 自己掌控控制流 ──
    if isinstance(next_step, RequestApproval):
        await db.save_thread(thread)          # F5/F6: 落盘后暂停
        await notify_human(next_step)         # F7: 人工作为工具
        break  # 等 webhook 回来再 resume，而不是阻塞 sleep

    # ── F9: 错误压缩进上下文，触发自愈 ──
    try:
        result = await execute(next_step)
        thread["events"].append({"type": "result", "data": result})
        consecutive_errors = 0
    except Exception as e:
        consecutive_errors += 1
        if consecutive_errors >= 3:
            thread["events"].append({"type": "escalate_to_human", "data": str(e)})
            break  # 超过阈值，升级人工
        thread["events"].append({"type": "error", "data": compact(e)})

# ── F12: 整个 thread 可被重放 ──
# 任何时候 new_state = replay(thread["events"])，Agent 即恢复到该状态
```

读这段骨架时把它对着 12 条逐行标注：prompt 你自己写（F2）、输出有 Pydantic schema（F4）、状态全在 `thread` 事件流里（F5）、审批会落盘后暂停而非阻塞（F6/F7/F8）、错误会压缩回灌并计数熔断（F9）、整个 thread 可重放（F12）——这就是「评审通过」的样子。

## 常见误区 / 面试追问

1. **误区：「12 条必须全部遵守，少一条就不算生产级」** — 这是最大的误读。作者明确说这些原则「可以独立采纳」，你可以只取其中一条（例如只做 F9 错误压缩）而不做其他 11 条。它的定位是**菜单而非教条**。正确的理解是：把它当 checklist 逐条自检「我有没有刻意思考过这一面」，而不是追求集齐 12 个徽章。

2. **误区：「12-Factor Agents 是反框架的，用了 LangGraph 就违背它」** — 不准确。作者的态度是「框架能帮你拿到大多数原则，但一旦需要深度控制就会成为负担」。事实上 LangGraph 1.0 GA 后的 per-node 超时、checkpointing、interrupt-before 等特性，恰好分别对应了 F6（暂停恢复）、F5（状态持久化）、F7/F8（在节点执行前插入人工审批）。问题不是「能不能用框架」，而是「框架的黑盒你是否逆向得动」。自研 vs 框架的差异在于：自研天然满足全部 12 条（因为循环你自己写），框架则满足其中大部分但可能对 F2（prompt 透明）和 F8（控制流）做了封装，需要你额外花力气「打开」。

3. **追问：「12 条里优先级最高的是哪几条？」** — 如果只能选三条，工程界共识倾向 **F5（统一状态）+ F8（自控控制流）+ F10（小而专注）**。F5 决定了 Agent 能不能从崩溃中恢复，F8 决定了你能不能在关键步骤插审批和熔断，F10 决定了上下文会不会爆炸。F1/F4 是基础前提（没有结构化输出谈不上生产化），而 F11/F12 更多是规模化后的优化项。面试时给出带排序的理由，比背诵 12 条更有说服力。

4. **追问：「Factor 10 说 Agent 要小，那 LLM 变强后这条不就过时了？」** — 作者专门回答过这个问题，结论是：**即使 LLM 能跑 100 步，保持小而专注依然是更优策略**。原因是确定性代码比 LLM 决策便宜、可靠、可测试几个数量级，把能确定化的部分留在 DAG 里、只把真正需要灵活判断的环节交给 Agent，永远比「全塞给一个大 Agent」更经济。随着模型能力提升，单个 Agent 的 scope 可以「慢慢长大」，但「刻意控制规模」的工程纪律不会消失——这与大型代码库重构的直觉一致。

5. **追问：「12-Factor Agents 和 MCP / A2A 是什么关系？」** — 正交且互补。MCP（Model Context Protocol）解决的是「Agent 怎么发现和调用工具」，对应 F4 中「工具作为结构化输出」的**工具侧标准化**；A2A 解决的是「Agent 之间怎么通信」，对应 F10/F11 的**多 Agent 编排与触发**。12-Factor Agents 不规定你用哪种工具协议，但 F1/F4/F8 会约束你「无论用什么协议，工具调用必须是结构化的、控制流必须在你手里」。作者在 README 里也半开玩笑地说「我不会专门讨论 MCP，但你应该看得出它插在哪里」。

## 参考资料

- [12-Factor Agents 官方仓库（humanlayer/12-factor-agents）](https://github.com/humanlayer/12-factor-agents) — 12 条原则原文与配套图示，Apache 2.0 / CC BY-SA 4.0
- [12 Factor Agents — HumanLayer 官方博客](https://www.humanlayer.dev/blog/12-factor-agents) — 作者 Dex Horthy 的原始阐述
- [The Short Version: The 12 Factors（README 目录）](https://github.com/humanlayer/12-factor-agents#the-short-version-the-12-factors) — 12 条原则索引与逐条链接
- [Factor 8: Own your control flow](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-08-own-your-control-flow.md) — 控制流与「工具选中↔执行之间插审批」详解
- [Factor 10: Small, Focused Agents](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-10-small-focused-agents.md) — 「LLM 变强后是否还需要小 Agent」的论证
- [Factor 9: Compact Errors into Context Window](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-09-compact-errors.md) — 错误压缩与自愈、连续错误熔断的代码示例
- [12-Factor App（Heroku, 2011）](https://12factor.net/) — 被 12-Factor Agents 致敬的经典方法论
- [Building Effective Agents — Anthropic](https://www.anthropic.com/engineering/building-effective-agents) — 与 12-Factor Agents 精神一致的官方工程指南（Agent = loop）
- [12-Factor Agents: Production Principles for Reliable AI Agents — Developers Digest](https://www.developersdigest.tech/blog/12-factor-agents-production-principles) — 第三方对 12 条的工程化解读与 star 数据引用
- [12-Factor Agents: Patterns of reliable LLM applications — Hacker News 讨论](https://news.ycombinator.com/item?id=43699271) — 社区对方法论的正反讨论

---

### 延伸思考 / 交叉引用

- 12 条中 **F3 Own Your Context Window** 即本项目的 **#102 Context Engineering**，建议合并阅读——F3 是 Context Engineering 在生产原则层面的精炼表述。
- **F7 Contact Humans with Tool Calls** 关联 **#080 Human-in-the-Loop**，「人工即工具」（含高风险操作的审批卡点）是 HITL 的核心理念。
- **F10 Small, Focused Agents** 关联多 Agent 拆分主题 **#031 多 Agent 系统** 与 **#036 多 Agent 框架对比**（LangGraph / CrewAI / OpenAI Agents SDK / Microsoft Agent Framework 在原则遵循度上的差异）。
- **F8 Own Your Control Flow + F9 Compact Errors** 关联 **#104 生产故障排查** 与 **#079 Guardrails**，控制流熔断与安全护栏是同一枚硬币的两面。
- 生产化落地方面，关联 **#010 生产级 Agent 系统设计**、**#074 Trace 和 Span**（F8 中 logging/tracing/metrics 的归属）以及 **#027 MCP**（工具侧标准化与 F4 的交叉点）。
