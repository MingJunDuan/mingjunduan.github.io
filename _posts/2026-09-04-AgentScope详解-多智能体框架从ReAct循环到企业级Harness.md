---
layout: post
title: "阿里 AgentScope 详解：一个多智能体框架，从 ReAct 循环到企业级 Harness"
author: "Inela"
---

你写了一个"客服智能体"，能查订单、能退款、能安抚用户，单机跑得好好的。可产品经理说：再给我加一个"质检智能体"，让它审客服的回复；再加一个"运营智能体"，让它看质检结论出报表。三个智能体要互相喊话，还要部署到线上——多个实例、多个租户、数据互相隔离、跑挂了能定位。这时候你发现，手写 `if-else` 和几段 `async/await`，已经撑不住这套东西了。

这不是你一个人的困境。单智能体（Single Agent）早就不是瓶颈，真正难的是**多智能体怎么协作、怎么上线、怎么治理**。而这，正是阿里 **AgentScope** 这类框架要解决的问题。

这篇文章延续之前《从大模型到 Agent》《Agent 的循环与目标》的剖析风格，回答三个问题：

1. **是什么**：AgentScope 是什么，它提供了哪些功能。
2. **解决什么问题**：它到底在解决多智能体/企业级落地里的哪几个痛点，尤其是 2.0 引入的 Harness 层。
3. **怎么用、和谁比**：给出真实可跑的代码例子，并说清它和 **Cursor**、**DeepSeek-Harness** 的差异在哪、什么时候该选谁。

---

## 一、AgentScope 是什么：定位与出身

### 1.1 一句话定位

> **AgentScope 是阿里（通义实验室）开源的多智能体框架**：它给你一套"定义 Agent、让多个 Agent 通过消息协作、把 Agent 部署成企业级服务"的工程底座。

几个关键事实：

- 2024 年开源，GitHub 仓库 [agentscope-ai/agentscope](https://github.com/agentscope-ai/agentscope)，配套论文《AgentScope: A Flexible yet Robust Multi-Agent Platform》（arXiv:2402.14034）。
- 从诞生起它就主打两件事：**多智能体协作** 与 **鲁棒/可容错**（flexible yet robust）。
- 多语言实现：**Python**（最初、文档最全）、**Java**（[agentscope-java](https://github.com/agentscope-ai/agentscope-java)，主打企业级 Harness）、**TypeScript**，Go 也在开发中。
- 在阿里内部它已经是使用最广的 Agent 框架之一，覆盖飞猪、淘宝闪购、千问 App、高德、阿里云、1688 等业务线；开源侧也进入了金融、物流、零售、制造、能源、医疗等行业。

### 1.2 它在技术栈里的坐标

把上一篇文章《从大模型到 Agent》的分层图往上一格，AgentScope 站的位置就清楚了：

```
┌──────────────────────────────────────────────────────┐
│   企业级 Harness（AgentScope 2.0 新加的一层）           │
│   Workspace · 多租户 · 分布式 · 记忆 · 沙箱 · 观测       │
├──────────────────────────────────────────────────────┤
│   多智能体框架（AgentScope 的"框架"本体）               │
│   ReAct 循环 · 消息传递 · 编排(Pipeline/MsgHub) · 工具   │
├──────────────────────────────────────────────────────┤
│   大模型 LLM（内核）          f(tokens) -> tokens       │
└──────────────────────────────────────────────────────┘
```

注意这层结构的精妙之处：**下面两层回答"怎么把模型变成能干活的多智能体"，最上面一层回答"怎么把多智能体变成能上生产的企业服务"**。很多人只把 AgentScope 当成 LangChain 那样的"写 Agent 的库"，其实 2.0 之后，它真正的重心已经移到了最上面那层 Harness。

### 1.3 与同类框架的一笔带过

- **LangChain / LlamaIndex**：重心是"给单 Agent 拼积木"（链、检索、工具），多智能体不是一等公民。
- **AutoGen（微软）**：同样是多智能体框架，以"对话式多 Agent"和"群聊 GroupChat"见长，但企业级治理（多租户、分布式文件系统、观测）不是它的主线。
- **AgentScope**：把"多智能体编排"和"企业级 Harness"两件事都做成一等公民，且背靠阿里云生态（通义/百炼、Higress、Nacos）。

这三者的对比后面第六节再展开，先看它本身长什么样。

---

## 二、核心功能拆解：它到底提供了什么

### 2.1 内核：ReAct Agent 循环

AgentScope 的"心脏"是 **ReActAgent**——即"推理（Reasoning）→ 行动（Tool Call）→ 观察（结果喂回）"的循环。这和上一篇文章讲过的 Agentic Loop 完全同构：

```
         ┌──────────────────────────────────────┐
  输入 ──▶│  Reasoning（模型想下一步做什么）         │
         │      │                               │
         │      ▼                               │
         │  Tool Call（调哪个工具、传什么参数）      │
         │      │                               │
         │      ▼                               │
         │  观察（工具结果作为新消息喂回模型）         │
         └──────┴──▶ 继续循环 / 输出最终回答          │
```

在 AgentScope 里，这个循环的内置能力相当全：支持**实时打断**（asyncio 取消机制）、**记忆压缩**、**并行工具调用**、**结构化输出**、**细粒度 MCP 控制**、**工具动态管理**、**自控长期记忆**、**自动状态管理**等。核心就是"这套循环你一行都不用自己写"。

### 2.2 Agent 与消息：两大基本抽象

AgentScope 的世界里，只有两样最核心的东西：

**（1）Agent**：一切智能体的基类。内置了 `ReActAgent`（主力）、`UserAgent`（人在环中，让真人以"一个 Agent"的身份参与对话）等。

**（2）Msg（消息）**：Agent 之间交流的唯一媒介。一条消息 = 角色（role）+ 内容（content，由 TextBlock 等块组成）+ 名字（name）。Agent 的输入是一条 `Msg`，输出也是一条 `Msg`——这为"多 Agent 互相传话"打下了统一地基。

### 2.3 多智能体编排：这是它的看家本领

这是 AgentScope 区别于"单 Agent 框架"的核心。它提供了三层编排手段：

| 手段 | 作用 | 一句话 |
|---|---|---|
| **MsgHub** | 消息广播中心 | 多个 Agent 进同一个"聊天室"，谁说话大家都听得见 |
| **Pipeline** | 结构化流水线 | sequential / if-else / for / while / msg 五种管道，把 Agent 串成有分支有循环的流程 |
| **模式** | 高层协作范式 | 辩论（Debate）、路由（Routing）、交接（Handoffs）、并发（Concurrent）等开箱即用 |

以 `MsgHub` 为例——这是"多 Agent 对话"最自然的形式：

```python
async with MsgHub(participants=[alice, bob, moderator]):
    await alice(Msg("user", "你是正方，请陈述观点。", "user"))
    await bob(Msg("user", "你是反方，请反驳并给出理由。", "user"))
```

`MsgHub` 的语义是"**广播**"：任一参与者发出的消息，会广播给其余所有人。两个辩手 `alice`、`bob` 在一个群里你来我往，而裁判 `moderator` 在一旁听全场——这正是"多智能体辩论"这个经典模式的底层机制（完整例子见第五节）。

### 2.4 工具、知识与记忆

Agent 要能干真活，还得有手脚和脑子里的存货：

- **Tool / Toolkit**：定义工具函数并注册给 Agent（`toolkit.register_tool_function(...)`）。
- **MCP**：作为"工具接入标准"被原生支持，一个 MCP Server 可被多个 Agent 复用。
- **Agent Skill**：把可复用的流程/脚本打包成"技能"，按需加载、在沙箱里执行。
- **RAG**：内置检索增强，给 Agent 补外部知识。
- **长期记忆**：跨会话的持久记忆（2.0 里演进成下节讲的"双层长期记忆"）。

### 2.5 观测与调试：AgentScope Studio

多 Agent 一多起来，最怕的是"它们内部到底怎么传的话、谁卡住了"没人看得见。AgentScope 给出了两件武器：

- **AgentScope Studio**：一个**拖拽式 Web UI**——你可以像搭积木一样把 Agent 拖到一起、连线、跑起来、实时看它们之间的消息流。做原型和调试时非常直观。
- **Tracing**：框架默认埋 OpenTelemetry 点，观测数据可上报到任何兼容平台（如开源的 LangFuse，或阿里云上的可观测产品），做"Agent 版分布式 trace"。

### 2.6 容错与分布式

"Flexible yet Robust"里的 Robust 不是白叫的：Agent 框架内建了**重试机制**、**容错处理**、**异步执行**，以及把 Agent **分布到多进程/多机**上去跑的能力。单机跑通的那套代码，加几行就能往分布式上走——这也是 2.0 Harness 要彻底解决的主线。

---

## 三、AgentScope 2.0 的 Harness：从"写单个 Agent"到"企业级底座"

这一节是"AgentScope 到底解决什么问题"的核心答案。**2.0 在原来 ReActAgent 内核之上，包了一层叫 Harness 的工程化层**。官方一句话讲得很清楚：

> 开发者既可以继续用轻量的 ReAct 循环，也可以按需启用 Workspace、持久记忆、Session、Sandbox、Skill、Subagent 等能力，把同一套 Agent 逻辑落地到企业级分布式服务中。

也就是说：**内核没变，变的是"从能跑 → 能上生产"之间缺的那一大堆工程件，被 Harness 一次性补齐了。**

### 3.1 Workspace：Agent 的"Source of Truth"

Harness 里最核心的抽象是 **Workspace（工作区）**。它是一个 Agent 的"全部家当"所在地：

```
Workspace
├── 静态资产（随镜像打包走）
│   ├── AGENTS.md          # 这个 Agent 的"人格/规范"定义
│   ├── Skills/            # 技能
│   └── Sub-agents/        # 子 Agent 定义
└── 运行时资产（运行中沉淀下来）
    ├── sessions/          # 会话日志
    ├── tasks/ plans/      # 任务与计划状态
    └── MEMORY.md          # 长期记忆
```

一句话记住：**Workspace 是一个 Agent 的"源码 + 记忆 + 状态"的统一落点**，是它的 Source of Truth。

### 3.2 抽象文件系统：这是"分布式"能成立的关键

Workspace 是逻辑概念，物理上存哪？最直白的答案是磁盘——但磁盘有个致命限制：**一个 Agent 部署多个实例时，每个实例都要看到同一个 Workspace**，磁盘做不到跨机共享。

AgentScope 的做法是抽象出一层 **Abstract File System（抽象文件系统接口）**，Agent 操作 Workspace 时只面向这个接口，接口下面可插拔三种物理后端：

| 后端 | 适用场景 |
|---|---|
| 本地磁盘（On-premise） | 单机、个人助手 |
| 树形文件系统 | 需要用户维度隔离 |
| MySQL / Redis / OSS | 生产分布式：多实例共享同一 Workspace |
| Sandbox | 需要强隔离（一个 Workspace 映射一个沙箱） |

因为有了这层抽象，**"多实例 + 共享状态 + 多租户隔离"** 这套企业级的硬需求才真正落地。这是 AgentScope 2.0 和绝大多数"单机 Agent 框架"拉开差距的地方。

### 3.3 上下文压缩：四道防线

上下文窗口有限，Agent 长期运行时必须管住上下文。Harness 内置了分层的压缩策略，只列几条典型的：

- 工具输出过大 → 截断 + 落盘，只把文件引用留在上下文里；
- 工具入参过大 → 字数截断；
- 历史消息 → 前缀结构化摘要 + 尾部保留原文（"留首尾 + 指针"）；
- 压缩用独立小模型、零额外 LLM 成本、自动重试一次……

更关键的是它懂"**什么不能压**"：任务规划、子 Agent 的异步任务状态、待办清单、权限授权记录——这些"全局要持续追踪的状态"会被排除在压缩之外，避免一压缩就把"正在进行的计划"压丢了。

### 3.4 双层长期记忆：事实自动沉淀

压缩会丢信息，丢掉的精华要沉淀成长期记忆。Harness 的做法是**双层**：

```
第一层  每天记一个流水账  memory/YYYY-MM-DD.md    （会话压缩前先做 Flush 分拣）
                        │
                        ▼  （后台任务定期蒸馏、去重）
第二层  全局 MEMORY.md                             （每次请求都注入 System Prompt）
```

配套还给了几个记忆工具（Memory Search / Memory Get / Session Search），模型会在需要时主动去查流水账。并且每一层的提取/蒸馏 Prompt 都可定制。

### 3.5 子 Agent 编排、沙箱、Skill、Plan 模式、Channel

Harness 还补齐了这些企业级能力：

- **子 Agent 编排**：主 Agent 通过 `Agent Fork / Agent Spawn` 拉起子 Agent，支持**同步/异步/远程**三种；异步子 Agent 完成后再主动把结果通知回主 Agent；用户甚至可以直接切进去和某个子 Agent 单独对话。
- **沙箱管理**：工具执行、Skill 脚本执行都可在沙箱里闭环，一套沙箱生命周期管理。
- **Skill 注册中心**：对接 Nacos 这类中心化管理，Skill 可做用户级隔离 + 审批式共享。
- **Plan 模式**：像 Claude Code / Codex 的 Plan 一样——先想清楚、写下来、再动手，还能自动切入/切出。
- **Channel**：把后台 Agent 对接企业 IM（消息平台 → Gateway → Agent）。

---

## 四、它到底解决什么问题（痛点 → 解法对照）

把上面的功能收拢成"它到底在替谁解围"，一张表说清：

| 你遇到的痛点 | AgentScope 的解法 | 对应能力 |
|---|---|---|
| 单 Agent 好写，**多 Agent 协作**难（怎么传话、怎么分支、怎么辩论） | 消息抽象 + MsgHub + Pipeline + 内置协作模式 | 2.2 / 2.3 |
| "本地能跑"容易，**分布式生产部署**难（多实例共享状态、多租户隔离） | Workspace + 抽象文件系统（磁盘/DB/OSS/Sandbox 可切换） | 3.1 / 3.2 |
| **上线后看不懂、没法管**（多个 Agent 内部黑盒） | Studio + Tracing（OpenTelemetry/LangFuse） | 2.5 |
| **记忆与上下文失控**（越聊越乱、关键状态被压丢） | 四道防线压缩 + 双层长期记忆 | 3.3 / 3.4 |
| **安全**（工具乱执行、Skill 带脚本、权限失控） | 沙箱 + 权限系统 + Human-in-the-loop | 3.5 |
| 任务复杂需要**先规划再动手** | Plan 模式 | 3.5 |

一句话总结它的核心价值主张：**让"多智能体"从 demo 变成"能分布式、能隔离、能观测、能治理"的企业级系统。**

---

## 五、实际例子（真实可跑的代码）

下面三段代码都取自官方教程/快速开始，做精简后能直接跑通（Python 为 v1.0 稳定 API，Java 为 2.0 Harness）。

### 5.1 Python：一个带工具的最小 ReAct Agent

```python
import asyncio, os
from agentscope.agent import ReActAgent
from agentscope.formatter import DashScopeChatFormatter
from agentscope.memory import InMemoryMemory
from agentscope.message import Msg, TextBlock
from agentscope.model import DashScopeChatModel
from agentscope.tool import Toolkit, ToolResponse


# 1) 定义一个工具函数：返回值必须是 ToolResponse
async def get_weather(city: str) -> ToolResponse:
    """查询某个城市的天气。"""
    return ToolResponse(
        content=[TextBlock(type="text", text=f"{city} 今天晴，25°C，微风。")]
    )


# 2) 注册到工具箱
toolkit = Toolkit()
toolkit.register_tool_function(get_weather)


# 3) 创建 ReAct Agent
agent = ReActAgent(
    name="Jarvis",
    sys_prompt="You are a helpful assistant.",
    model=DashScopeChatModel(
        model_name="qwen-max",
        api_key=os.environ["DASHSCOPE_API_KEY"],
    ),
    memory=InMemoryMemory(),
    formatter=DashScopeChatFormatter(),
    toolkit=toolkit,
)


async def main():
    # 4) 发一条消息，Agent 内部会自己决定要不要调 get_weather
    reply = await agent(Msg("user", "北京今天天气怎么样？", "user"))
    print(reply.get_text_content())


asyncio.run(main())
```

这段代码里，Agent 的"循环"全被 `ReActAgent` 包掉了：你只定义工具 + 发消息，剩下的"模型判断要不要调工具 → 执行 → 把结果喂回去 → 最终回答"都由框架完成。换个模型（OpenAI / DeepSeek / GLM / Kimi / Ollama 等）只需换 `model` 那一行的 Model 类。

### 5.2 Python：两个 Agent 用 MsgHub 辩论

```python
import asyncio, os
from agentscope.agent import ReActAgent
from agentscope.formatter import DashScopeMultiAgentFormatter
from agentscope.message import Msg
from agentscope.model import DashScopeChatModel
from agentscope.pipeline import MsgHub

topic = "地球是平的还是圆的？请给出论证。"


def make_debater(name: str, stance: str) -> ReActAgent:
    return ReActAgent(
        name=name,
        sys_prompt=f"你是辩手 {name}，立场：{stance}。围绕话题「{topic}」辩论。",
        model=DashScopeChatModel(
            model_name="qwen-max",
            api_key=os.environ["DASHSCOPE_API_KEY"],
            stream=False,
        ),
        # 多 Agent 场景要用 MultiAgent 版 formatter
        formatter=DashScopeMultiAgentFormatter(),
    )


alice = make_debater("Alice", "正方")
bob = make_debater("Bob", "反方")


async def main():
    # MsgHub 把两个辩手拉进同一个"聊天室"，消息互广播
    async with MsgHub(participants=[alice, bob]):
        await alice(Msg("user", "请陈述正方观点。", "user"))
        await bob(Msg("user", "请反驳正方，并给出反方观点。", "user"))


asyncio.run(main())
```

两个细节值得注意：其一，多 Agent 场景必须用 `DashScopeMultiAgentFormatter`（它靠消息里的 `name` 字段区分"谁说的"，而不是靠 `role`）；其二，`MsgHub` 的广播语义是"多智能体协作"的地基——辩论、讨论、群聊，本质都是它。

### 5.3 Java 2.0：HarnessAgent + Workspace

企业级 Harness 层的用法（[官方 quickstart](https://java.agentscope.io/v2/zh/docs/quickstart.html) 精简）：

```java
import io.agentscope.core.agent.RuntimeContext;
import io.agentscope.core.message.UserMessage;
import io.agentscope.harness.agent.HarnessAgent;
import io.agentscope.harness.agent.memory.compaction.CompactionConfig;
import java.nio.file.Paths;

public class FirstAgent {
    public static void main(String[] args) {
        HarnessAgent agent = HarnessAgent.builder()
                .name("note-taker")
                .sysPrompt("你是一个帮助用户做笔记的助手。")
                // "dashscope:qwen-plus" 由 ModelRegistry 解析，换厂商改这里即可
                .model("dashscope:qwen-plus")
                .workspace(Paths.get(".agentscope/workspace"))
                .compaction(CompactionConfig.builder()
                        .triggerMessages(30)   // 满 30 条触发压缩
                        .keepMessages(10)      // 保留最近 10 条不压
                        .build())
                .build();

        // 多租户：同一个 agent 实例，不同用户/会话传不同 RuntimeContext
        RuntimeContext ctx = RuntimeContext.builder()
                .sessionId("demo-session")
                .userId("alice")
                .build();

        agent.call(new UserMessage("我叫天宇，今天准备一个关于 ReAct 的分享。"), ctx).block();
        // 第二轮：同 sessionId，自动恢复上一轮状态
        agent.call(new UserMessage("我叫什么？我今天要干什么？"), ctx).block();
    }
}
```

注意第 3.2 节讲的抽象文件系统在这里就落地了：`.workspace(...)` 指定的 Workspace 是可切换的物理后端，而 `RuntimeContext` 里的 `userId` / `sessionId` 就是多租户隔离的入口——**同一个 `agent` 实例，不同用户各写各的 Workspace、各记各的记忆**。

### 5.4 官方企业级示例一瞥

官方仓库还放了几个"验证 AgentScope 能上生产"的完整示例，值得知道：

- **个人助手**：类 QwenPaw 的产品原型，Workspace 绑本地磁盘，验证"个人场景怎么搭"；
- **多租户 Agent 平台（Agent Builder）**：零代码建 Agent 的 SaaS，全公司共用同一 Agent 但数据隔离——就是 Claude Managed Agents / Qoder Cloud Agents 的原型；
- **Data Agent**：per-用户数据隔离 + 审批式 Skill 共享；
- **企业 Coding Agent**：集中部署后对接 GitLab，每个 Issue/PR 拉一个独立沙箱处理，用户间互不影响，可当 CI/CD 平台用。

这些示例共同说明一件事：**AgentScope 2.0 的目标用户不是"想快速跑个 demo 的人"，而是"要把多 Agent 变成企业服务的团队"。**

---

## 六、与 Cursor、DeepSeek-Harness 的区别

这是很多人最想搞清楚的部分。先把结论砸在前面：

> **这三个东西根本不是一个物种。** Cursor 是"产品"（一个现成的编码 Agent，你用它）；AgentScope 是"多智能体开发框架"（你基于它造 Agent）；DeepSeek-Harness 是"agent harness / 运行时"（你基于它组装 Agent，偏 CLI/编码/工作流）。

### 6.1 定位先行：先分清"产品 / 框架 / 运行时"

| | Cursor | AgentScope | DeepSeek-Harness (dsh) |
|---|---|---|---|
| 物种 | **产品**（AI 编码编辑器） | **多智能体框架** | **agent harness / 运行时** |
| 谁写的 | Anysphere，闭源 | 阿里通义实验室，开源 | DeepSeek AI，开源 |
| 一句话 | 装好就能用的 AI 写代码工具 | 造"多智能体协作"的零件库 + 企业底座 | 组装"能干活 Agent"的可插拔骨架 |
| 你扮演的角色 | 使用者 | 开发者 | 开发者/组装者 |

### 6.2 维度对比大表

| 维度 | Cursor | AgentScope | DeepSeek-Harness |
|---|---|---|---|
| **语言/生态** | 桌面应用（VS Code 生态） | Python（主力）/ Java / TypeScript | TypeScript/Node（另有 Python SDK runtime） |
| **Agent 形态** | 单 Agent（Composer/Agent 模式） | 多 Agent 是一等公民（MsgHub/Pipeline/辩论/路由/交接） | 单 Agent 循环 + 委派（subagent/workflow/Ralph） |
| **多智能体** | 不直接暴露 | 核心卖点，含分布式 | 有，但以"主子委派"为主，非广播式 |
| **记忆/状态** | messages 数组 + 压缩 + `.cursorrules` | Workspace + 双层长期记忆 + 状态存储（MySQL/Redis/OSS） | 会话日志（append-only SessionEvent，"模型可见 ⟺ 已记录"） |
| **可观测** | 界面内较有限的展示 | Studio + OpenTelemetry/LangFuse | 事件流 + 会话查询/回放 |
| **部署形态** | 本地桌面（+团队计划） | 分布式、多租户、企业级 | 偏本地/单机 CLI，也可远程事件 |
| **编程哲学** | 产品封装，你不可改内核 | Actor + 消息传递 + ReAct + Harness 层 | 一切皆插件 + 能力接缝 + 会话日志真理源 |
| **上手门槛** | 最低（装完即用） | 中（要写代码定义 Agent） | 中高（要理解插件/接缝体系） |

### 6.3 AgentScope vs DeepSeek-Harness：同为"框架/harness"，最需要辨清

这俩最容易被摆在一起比，因为它们都自称"框架/底座"。差异在**哲学**层面最深：

**（1）第一性原理不同**

- AgentScope 的第一性原理是 **"Actor + 消息"**：一切皆 Agent，Agent 之间靠 `Msg` 显式通信，多 Agent 是天然默认。
- DeepSeek-Harness 的第一性原理是 **"一切皆插件 + 会话日志真理源"**：连"Agent 循环本身"都是插件，模型能看到的一切都必须能从 append-only 的会话日志里重建出来（`模型可见 ⟺ 已记录`）。

**（2）多智能体的组织方式不同**

- AgentScope：**广播式/编排式**——`MsgHub` 让 N 个 Agent 同处一室互听，`Pipeline` 把 Agent 串成流水线，辩论/路由/交接是内置模式。
- DeepSeek-Harness：**委派式**——主 Agent 用 `subagent` 派生子 Agent、用 `workflow` 编排、用 Ralph 做多轮新 Agent 迭代，重心在"一个主脑 + 委派子任务"，而不是"N 个平级 Agent 互相辩论"。

**（3）"状态的家"不同**

- AgentScope 把状态放进 **Workspace**（文件 + 记忆 + 会话，可换物理后端、可多租户）。
- DeepSeek-Harness 把状态放进 **会话日志**（append-only SessionEvent，上下文由此派生，可回放、可审计）。

**（4）语言与面向场景不同**

- AgentScope 是 **Python 优先**（面向 AI 应用/研究/企业业务 Agent），Java/TS 面向企业后端。
- DeepSeek-Harness 是 **TypeScript/Node 优先**（面向 CLI 编码 Agent 工具链、可插拔工作流），它和你现在正在用的这个 Web/TUI 会话就是同源产物。

一句话总结：**AgentScope 更"面向多智能体应用与业务"，DeepSeek-Harness 更"面向可插拔的 Agent 运行时与工具链"；两者不是替代关系，而是解决"Agent 工程化"这个大类下的不同子问题。**

### 6.4 AgentScope vs Cursor：框架 vs 产品

这俩的差异其实是"**零件 vs 成品**"：

- **Cursor** 是一个已经封装好的**编码 Agent 产品**：你敲需求，它读代码库、改文件、跑命令、做 diff。它的 Agent 循环、工具、记忆、压缩全都替你做好了，但你**改不了它的内核**，也**没法用它去造一个"客服智能体"或"辩论智能体"**。
- **AgentScope** 是一盒**造 Agent 的零件**：你可以用它造出"编码 Agent"，但更常见的是造出"客服+质检+运营"这种 Cursor 造不了的多智能体业务系统。

类比一下：**Cursor 是一辆现成的车，AgentScope 是发动机 + 变速箱 + 底盘的零件库。** 你要"上路"就买 Cursor，要"造一辆特种车"就得用 AgentScope。

### 6.5 什么时候选谁

| 你的诉求 | 选谁 |
|---|---|
| 就想让 AI 帮我写代码、改项目，装好即用 | **Cursor** |
| 要造"多个 Agent 协作"的业务系统（客服/质检/内容/研究），Python/Java 技术栈 | **AgentScope** |
| 要一个可插拔、可组合、可观测的 Agent 运行时/工具链，深度定制循环与能力 | **DeepSeek-Harness** |
| 既要造业务 Agent、又要企业级多租户/分布式/观测 | **AgentScope 2.0（Harness）** |

---

## 七、总结

**一句话记住 AgentScope：**

```
AgentScope = 阿里开源的多智能体框架
           = ReAct 循环（内核） + 消息/编排（多智能体） + Harness（企业级工程化层）
```

- **功能**：定义 Agent → 用 `Msg`/`MsgHub`/`Pipeline` 编排多 Agent → 挂工具/MCP/Skill/RAG/记忆 → 用 Studio/Tracing 观测 → 用 Harness 落成分布式多租户服务。
- **解决什么问题**：把"多智能体"从**能跑**的 demo，变成**能分布式、能隔离、能观测、能治理**的企业级系统。
- **和 Cursor / DeepSeek-Harness 的区别**：Cursor 是**产品**，AgentScope 是**多智能体框架**，DeepSeek-Harness 是**可插拔的 Agent 运行时**——三者不在一个物种上，各解各的子问题。

判断"我该不该用 AgentScope"，套一个简单 checklist：我是不是要**多个 Agent 协作**？是不是要**上生产、多租户、分布式**？是不是 **Python/Java 技术栈**？三条里中两条，AgentScope 就值得认真看；如果只是"想让 AI 帮我改代码"，那 Cursor 更直接；如果是要"深度定制 Agent 循环与工具链、一切皆插件"，那 DeepSeek-Harness 才是对的那把刀。

---

## 附录：AgentScope 相关词表

| 概念 | 一句话 |
|---|---|
| ReActAgent | 内置的"推理→行动→观察"循环 Agent，框架主力 |
| Msg | Agent 间通信的消息，= role + content + name |
| MsgHub | 消息广播中心，多 Agent "群聊"的地基 |
| Pipeline | 顺序/分支/循环流水线，把 Agent 串成流程 |
| Workspace | Agent 的"源码+记忆+状态"统一落点，Source of Truth |
| Abstract File System | Workspace 的抽象存储层，可切磁盘/DB/OSS/Sandbox |
| RuntimeContext | 2.0 多租户隔离入口（userId / sessionId） |
| MEMORY.md | 全局长期记忆，每次请求注入 System Prompt |
| Studio | 拖拽式 Web UI，多 Agent 可视化编排与调试 |
| Tracing | OpenTelemetry 埋点，接 LangFuse 等做 Agent trace |
| HarnessAgent | Java 2.0 的入口 Agent，ReAct + Harness 能力 |

**参考资料**

- AgentScope 官方文档（Python）：<https://doc.agentscope.io/>
- AgentScope Java 2.0 文档：<https://java.agentscope.io/v2/zh/docs/>
- AgentScope 2.0 企业级 Harness 技术解析：<https://java.agentscope.io/v2/zh/blogs/agentscope-v2-explained.html>
- 论文《AgentScope: A Flexible yet Robust Multi-Agent Platform》：arXiv:2402.14034
