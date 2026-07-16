# 什么是 Computer Use / Browser Use Agent？它与 API Agent 有何区别？

> 难度：高级
> 分类：Agent 架构

## 简短回答

**Computer Use Agent（计算机使用智能体）** 是一类能够像人类一样"看屏幕、动鼠标、敲键盘"来完成任务的 AI Agent。它通过截取屏幕画面（或读取网页 DOM / 辅助功能树）感知图形界面（GUI），由多模态大模型规划下一步操作，再通过点击、输入、滚动等原子动作驱动真实的操作系统或浏览器，完成"订机票""填表单""操作内部 ERP"等任务。**Browser Use Agent** 是其子集，动作空间限定在浏览器内部。

它与 **API Agent（Function Calling Agent）** 的本质区别在于**操作对象**：API Agent 调用的是结构化、稳定、有 schema 的接口（如 `create_ticket(title, assignee)`），输入输出确定性强；Computer Use Agent 操作的是**为人类设计的图形界面**——没有稳定契约、布局随时变化、元素靠视觉位置定位。这一跃迁带来了巨大的能力扩展（任何带界面的软件都能被自动化），也带来了显著的成本与可靠性代价。

2026 年这一方向快速成熟：Anthropic 的 Claude Computer Use 已支持"手机发任务、电脑执行"的跨设备模式，OpenAI 推出竞品 CUA（Computer-Using Agent），开源框架 Browser Use（21K+ stars）在 WebVoyager 基准上达约 89% 成功率。Computer Use 正从"技术 demo"走向生产级 Agent，是 Agent 自主性的标志性进展。

## 详细解析

### 一、范式跃迁：从 API 到 GUI

传统 Agent 工具调用的前提是"目标系统愿意为 AI 开放 API"。现实里大量软件——遗留系统、桌面应用、带验证码的网站、企业内部门户——没有 API，或 API 能力远少于界面功能。Computer Use Agent 把"人类用 GUI"这条通用通道开放给 Agent，相当于给 Agent 一个"数字劳动力"的身体。

```
┌─────────────────────────── API Agent ───────────────────────────┐
│  Agent ──► Function Call ──► { "tool": "create_issue",          │
│                                 "args": {...} }                 │
│           结构化 schema ✓   稳定契约 ✓   可静态校验 ✓            │
└──────────────────────────────────────────────────────────────────┘

┌─────────────────────── Computer Use Agent ──────────────────────┐
│  Agent ──► 看截图 ──► 规划 ──► mouse_move(842,310) / click / type│
│           视觉感知          无契约   靠坐标/元素定位             │
│           任意 GUI 都能操作，但状态不确定、需校验                │
└──────────────────────────────────────────────────────────────────┘
```

一句话总结：API Agent 操作的是"机器为机器准备的接口"，Computer Use Agent 操作的是"机器为人准备的界面"。

### 二、感知层：Agent 如何"看见"界面

感知层是 Computer Use Agent 与 API Agent 最大的分水岭，主流分三条技术路线：

| 感知方式 | 原理 | 优势 | 劣势 | 代表方案 |
|---------|------|------|------|---------|
| **纯视觉（Screenshot + VLM）** | 截图喂给多模态模型，模型直接输出坐标 | 通用，能处理任意 GUI（桌面应用、Canvas、远程虚拟桌面） | 定位精度依赖模型，坐标易漂移，图片 token 成本高 | Claude Computer Use、OpenAI CUA |
| **结构化（DOM / Accessibility Tree）** | 读取网页 DOM 或无障碍树，拿到元素 id、role、text | 定位精准、可点元素有唯一引用，token 省 | 只能用于 Web 或有 a11y 暴露的应用，动态渲染难 | Browser Use、WebArena 系列 |
| **混合（Visual + Set-of-Mark）** | 截图叠加编号标签（SoM），模型选编号而非坐标 | 兼顾视觉泛化与定位精度 | 需要预处理标注步骤 | SeeAct、部分 Browser Use 模式 |

**Set-of-Mark（SoM）** 是关键的折中思路：先用检测模型给截图中每个可交互元素打上数字编号（[1] [2] [3]…），模型只需回答"点 3 号"，把"坐标回归"问题降维成"编号选择"问题，大幅降低定位误差。这也是纯视觉方案后来借鉴结构化思路、提升精度的常用手段。

### 三、决策层：从"点击登录按钮"到坐标

决策层负责把自然语言意图（"点击登录按钮"）映射到可执行的动作。核心是 **Grounding（视觉接地）**——把语义概念锚定到界面上的具体位置。

```
意图层：   "我要登录"
   │
   ▼  语义解析
目标识别： "找到标着 Login / 登录 的可点击元素"
   │
   ▼  Grounding（视觉接地）
定位策略： 纯视觉 → 输出坐标 (x=842, y=310)
           DOM   → 输出 selector "#login-btn"
           SoM   → 输出编号 "click [3]"
   │
   ▼  动作生成
原子动作： click(842, 310)
```

决策通常不是一次性的，而是 **"观察—思考—行动"（Observe-Think-Act）** 的 ReAct 循环：模型每一步先描述看到了什么，再决定下一步做什么，遇到歧义还会自我提问（"这里有两个 Login 按钮，该选导航栏的还是表单里的？"）。这与普通 Agent 的 ReAct 循环同构（参见 #006 Agent Loop 设计），区别仅在于"观察"的来源从工具返回值变成了截图/DOM。

### 四、执行层：动作空间与状态机

Computer Use Agent 的动作空间（Action Space）是一组原子操作：

```
click(x, y)            # 左键单击
double_click(x, y)
right_click(x, y)
type(text)             # 在焦点元素输入文字
key("Return")          # 按键 / 组合键 ctrl+c
scroll(direction, amt) # 滚动
mouse_move(x, y)
drag(x1,y1, x2,y2)     # 拖拽
screenshot()           # 主动截屏（重新感知）
wait(seconds)          # 等待加载
```

**操作原子化**有两个好处：一是便于校验每一步（出了错能精确定位是哪一拍），二是便于回放与回滚。整个执行过程被建模为一个隐式的**状态机**：每个动作改变环境状态，Agent 重新感知后判断是否进入期望状态，未达预期则触发修复分支。这与"一段宏脚本一口气跑完"的早期 RPA 思路本质不同——后者一旦中间出错就全线崩溃，前者允许逐步校验与自我修复。

### 五、架构总览：感知→规划→执行→校验循环

```
        ┌─────────────────────────────────────────────────┐
        │                  用户任务                        │
        │     "帮我在携程订一张明天北京到上海的机票"        │
        └──────────────────────┬──────────────────────────┘
                               ▼
   ┌────────────────────────────────────────────────────────┐
   │                    Agent Loop                          │
   │                                                        │
   │  ┌──────────┐    ┌──────────┐    ┌──────────┐         │
   │  │  感知     │───►│  规划     │───►│  执行     │         │
   │  │ Perceive │    │ Plan     │    │ Execute  │         │
   │  │ 截图/DOM │    │ ReAct    │    │ click/... │         │
   │  └────┬─────┘    └──────────┘    └─────┬────┘         │
   │       ▲                                 │              │
   │       │           ┌──────────┐          ▼              │
   │       └───────────┤  校验     │◄─────────┘              │
   │                   │ Verify   │                         │
   │                   │ 是否达成 │                         │
   │                   │ 子目标?  │                         │
   │                   └─────┬────┘                         │
   │                  达成 ↓     ↓ 未达成→修复/回溯         │
   └─────────────────────────┼────────────────────────────┘
                             ▼
                      任务完成 / 主动放弃
```

**校验环节**是 Computer Use Agent 区别于"盲操作脚本"的关键：每一步执行后都要重新感知界面，确认操作生效（按钮真的按下去了？页面真的跳转了？），否则进入修复分支。没有校验的 GUI 自动化本质上是脆弱的录制回放，稍有界面变化就崩。

### 六、可靠性挑战

GUI 的不确定性是 Computer Use Agent 落地的最大障碍：

1. **动态界面与时序问题**：元素异步加载，点击时按钮可能还没渲染完。需要显式等待条件（"等到登录按钮可见再点"），而非固定 `sleep`。
2. **加载等待**：网络抖动导致截图是"旧画面"，Agent 在过期状态上决策。需要加载完成检测，否则会点到已经不存在的元素。
3. **误点击与误操作**：坐标漂移点到错误元素（如点到广告而非目标按钮），且动作可能不可逆（点了"删除""转账""提交订单"）。这是安全红线。
4. **回溯与自我修复**：走错路后如何识别并退回。成熟的方案会维护"操作栈 + 检查点"，结合 DOM diff / 截图对比判断状态变化，必要时回退到上一个检查点。
5. **验证码与人机校验**：很多站点主动识别并拦截自动化访问，这是合规与反爬的灰色地带，生产方案通常在此处转人工。

针对可靠性，工程上常用 **"先断言再操作"（assert-before-act）** 范式：每一步执行前先验证前置条件（目标元素存在、可见、可点），执行后验证后置条件（页面状态如期变化），任一不满足即暂停或转 HITL。

### 七、主流方案对比

| 方案 | 提供方 | 感知方式 | 运行范围 | 特点 |
|------|--------|---------|---------|------|
| **Claude Computer Use** | Anthropic | 纯视觉（截图） | 桌面 + 浏览器（Docker 容器） | 官方 `computer` tool；2026.3 起支持跨设备"手机发任务、电脑执行"；OS 级操作，泛化最强 |
| **OpenAI CUA / Operator** | OpenAI | 纯视觉（截图） | 浏览器为主 | CUA（Computer-Using Agent）模型，早期以 Operator 产品形态提供研究预览 |
| **Browser Use** | 开源社区 | DOM + 视觉混合 | 浏览器（Playwright） | Python 框架，21K+ stars，WebVoyager 约 89%；可自选底层 LLM |
| **学术 DOM/SoM 方案** | 学术界 | DOM / Set-of-Mark | 浏览器 | WebVoyager、SeeAct 等，定位精准但泛化弱，多为研究基线 |

选型经验：**需要操作桌面应用/远程系统 → Claude Computer Use 这类纯视觉方案；只在浏览器内且追求精准高效 → Browser Use 这类 DOM/混合方案**。注意"Anthropic Computer Use"与"Claude Computer Use"指的是同一个产品——Claude 是 Anthropic 的模型，Computer Use 是其提供的 GUI 操作能力。

### 八、代码示例：Computer Use Agent 主循环（伪代码）

以 Anthropic Computer Use API 风格为例，核心是"截图—让模型决策—执行—再截图"的循环：

```python
def computer_use_loop(task: str, max_steps: int = 50):
    """Computer Use Agent 的感知-决策-执行-校验主循环"""
    messages = [{"role": "user", "content": task}]
    for step in range(max_steps):
        # 1. 调用模型，传入 computer 工具定义（工具类型版本以官方文档为准）
        response = client.messages.create(
            model="claude-sonnet-5",
            tools=[{
                "type": "computer",          # Anthropic computer-use 工具
                "name": "computer",
                "display_width_px": 1280,
                "display_height_px": 800,
            }],
            messages=messages,
        )
        messages.append({"role": "assistant", "content": response.content})

        # 2. 没有工具调用 = 模型认为任务完成
        if response.stop_reason == "end_turn":
            return "done"

        # 3. 执行工具返回的动作，并把结果（通常是新截图）回传
        for block in response.content:
            if block.type == "tool_use":
                action = block.input          # {"action": "click", "coordinate": [842, 310]}
                result = execute_action(action)   # 真正去 click / type / screenshot
                messages.append({
                    "role": "user",
                    "content": [{"type": "tool_result",
                                 "tool_use_id": block.id,
                                 "content": result}],  # 下一轮感知的输入
                })


def execute_action(action: dict):
    """把模型输出的高层动作映射到底层执行，并返回新截图"""
    act = action["action"]
    if act == "click":
        pyautogui.click(*action["coordinate"])
    elif act == "type":
        pyautogui.typewrite(action["text"])
    elif act == "scroll":
        scroll_page(action["direction"])
    elif act == "key":
        pyautogui.hotkey(*action["key"].split("+"))
    elif act == "wait":
        time.sleep(action.get("duration", 2))
    # 关键：操作后返回新截图，供下一轮感知形成闭环
    return capture_screenshot()
```

注意 `execute_action` 的返回值是一张新截图——感知与执行通过"动作改变画面，画面反馈给模型"形成闭环。这正是前文架构图中 **Verify → Perceive** 回路在代码层面的体现。

## 常见误区 / 面试追问

1. **误区："Computer Use 是 API Agent 的升级版，能取代 API Agent"** — 恰恰相反，能用 API 就不要用 Computer Use。API Agent 的确定性、成本、速度、可审计性都远优于 GUI 操作。Computer Use 是"没有 API 时的最后手段"，适用于遗留系统、闭源软件、必须人类界面交互的场景。选型原则一句话：**API 优先，GUI 兜底**。

2. **误区："纯视觉方案比 DOM 方案更先进"** — 两者各有适用域。纯视觉能处理任意 GUI（桌面应用、Canvas、远程虚拟桌面），但定位精度和 token 成本吃亏；DOM 方案在 Web 内精准高效，但碰到 Canvas / 图片 / 桌面应用就失效。生产中常按目标系统选型，甚至混合使用（Web 走 DOM、桌面走视觉），不存在"谁取代谁"。

3. **追问："如何评估一个 Computer Use Agent？"** — 行业有专门基准：**OSWorld**（真实操作系统上的多模态 Agent 任务，369 个任务，覆盖 Ubuntu/Windows/macOS 多类应用）测的是"全能型"难度，整体分数偏低，是公认最难的 GUI Agent 基准之一；**WebArena / VisualWebArena**（真实网站环境的 Web Agent 任务）专注浏览器；**WebVoyager**（真实网站端到端任务）是常见实测基准，Browser Use 在其上约 89%。评估维度除成功率外，还包括**平均步数**（效率）、**回溯次数**（鲁棒性）、**token 成本**（经济性）。务必警惕静态 benchmark gaming（参见 #076）——高分不等于在你的目标系统上可靠，最终要在真实环境回归测试。

4. **追问："如何防止误操作？"** — 这是生产化的核心安全命题，常见手段分层叠加：(1) **沙箱隔离**——在容器 / 虚拟机里运行，限制可访问的文件、网络、进程；(2) **权限最小化**——只给完成任务所需的最小操作权限，敏感目录只读；(3) **破坏性操作确认（HITL 卡点）**——删除、转账、提交订单等不可逆动作必须人工确认（参见 #080）；(4) **全链路操作日志**——记录每一步截图 + 动作，可回放可追责（参见 #074）；(5) **预算上限**——限制最大步数与成本，避免死循环烧钱。

5. **追问："Computer Use 的成本和延迟怎么样？"** — 显著高于 API Agent。每一步都要截图（一张图几百到上千 token）+ 多模态推理 + 执行 + 再截图，单任务动辄几十轮。一个订票任务可能消耗数万 token、耗时数分钟。这决定了它不适合高频、对延迟敏感的在线场景，更适合异步、批处理、人工发起的任务（这也解释了为何 Anthropic 把它做成"手机发任务、电脑慢慢执行"的跨设备形态）。

## 参考资料

- [Introducing Computer Use, a new Claude 3.5 Sonnet capability (Anthropic)](https://www.anthropic.com/news/claude-3-5-sonnet-computer-use)
- [Computer Use Tool — Anthropic Docs](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/computer-use-tool)
- [OSWorld: Benchmarking Multimodal Agents for Open-Ended Tasks in Real Computer Environments (Xie et al., 2024)](https://arxiv.org/abs/2404.07972)
- [WebArena: A Realistic Web Environment for Building Autonomous Agents (Zhou et al., 2023)](https://arxiv.org/abs/2307.13854)
- [WebVoyager: Building an End-to-End Web Agent via Large Multimodal Models (He et al., 2024)](https://arxiv.org/abs/2401.13919)
- [Browser Use — Open Source Web Agent Framework (GitHub)](https://github.com/browser-use/browser-use)
- [Anthropic Computer Use vs OpenAI CUA 对比 (WorkOS)](https://workos.com/blog/anthropics-computer-use-versus-openais-computer-using-agent-cua)

---

### 延伸思考 / 交叉引用

- **工具调用基础**：Computer Use 本质是一种“非结构化工具调用”，理解它需要先掌握结构化工具调用（#021 Function Calling、#027 MCP），“API 优先”是通用最佳实践。
- **安全与合规**：破坏性操作的 HITL 卡点参见 #080 Human-in-the-Loop；全链路 trace 审计参见 #074 Trace 和 Span；Agent 合规要求（如 EU AI Act 高风险系统）参见 #115。
- **评估方法**：GUI Agent 的基准陷阱与静态 benchmark 局限参见 #076 静态 Benchmark 陷阱、#072 Agent Benchmark 体系。
- **生产化**：Computer Use Agent 的循环架构与生产级 Agent 的循环设计（#010 生产级 Agent 系统设计）一脉相承；其可靠性工程实践可对照 #104 生产故障排查。
- **选型决策**："何时用 Computer Use vs 传统 RPA vs API"是高频面试追问——核心判据是"目标系统是否有稳定 API"与"操作是否需要视觉泛化"：有 API 用 API Agent，纯 Web 且追求精准用 Browser Use 类方案，桌面应用/远程系统/无 API 才上纯视觉 Computer Use。
