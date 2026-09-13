---
layout: post
title: "拆解 pi 的 Agent Loop：1800 行核心，怎么把“大脑、手脚、记事本、任务单”组装成会干活的 Agent"
author: "Inela"
---

上一篇 [《AgentScope 详解》](/2026-09-04/AgentScope详解-多智能体框架从ReAct循环到企业级Harness) 讲了多智能体框架从 ReAct 循环到 Harness 的全貌，[《Agent 循环与目标》](/2026-08-29/Agent循环与目标-一个Agent到底是怎么自己跑起来的) 讲了原理。这篇换个讲法：**不再画原理图，而是钻进一份真实的、生产级的源码——pi——看一个 agent loop 到底长什么样、多少行、每一行在干嘛。**

先立一句话贯穿全文，后面所有内容都在给这句话做注脚：

> **Agent = 大脑 + 手脚 + 记事本 + 任务单。Agent loop 就是把这些零件组装起来的"工作习惯"：看一眼任务单 → 翻记事本 → 决定自己答还是动手 → 动手 → 把结果记回记事本 → 再看一眼……直到任务单打勾。**

pi 是 Flask 作者 Armin Ronacher 的新项目，一年时间 104K star，全仓库 TypeScript 约 18 万行。但**组装四要素的循环核心，只有 4 个文件、约 1860 行**。这篇博客就讲这 1860 行，以及它周围的记忆与压缩代码。

---

## 一、先把四要素认全：一个工位上的 Agent

想象一个工位，上面摆着四样东西：

```
┌─────────────────────────── 工位 = Agent ───────────────────────────┐
│                                                                    │
│  大脑 LLM              手脚 工具              记事本 记忆           │
│  "什么都懂，但碰不到     "能改文件、跑命令、     "刚才说过什么、      │
│   现实世界，只会说话"     查文档、搜代码"        改过哪些文件，      │
│                                                全记在这"           │
│                                                                    │
│  任务单 目标                                                        │
│  "用户到底要什么？干到什么程度，才算把活干完？"                     │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

四样东西摆在那，工位还是不会干活。**缺的是"工作习惯"**——没有人规定：先看哪个、看完怎么判断、动手之后结果放哪、什么时候算干完。这套"习惯"，就是 agent loop。

对照到 pi 源码，四要素各有自己的"实体"：

| 零件 | 通俗说法 | pi 源码里的实体 |
|---|---|---|
| **大脑 LLM** | 什么都懂但碰不到现实 | `Model` + `StreamFn`（由 pi-ai 包提供） |
| **手脚 工具** | 改文件、跑命令、查资料 | `AgentTool`（types.ts 定义） |
| **记事本 记忆** | 上下文窗口就是它的记事本 | `AgentContext.messages` |
| **任务单 目标** | 用户要什么、怎样算完 | `systemPrompt` + 终止条件 |

问题来了：**这四个零件在源码里是四样互相独立的东西，谁负责把它们按顺序串起来、中途用户插话不炸、大脑出错不崩？**

这就是 agent loop 的活。下面去源码里找它。

---

## 二、源码地图：四要素住在哪个包

pi 是 monorepo，11 个包。先看全景，再看"1-2 千行"这个说法的真相：

```
pi（monorepo，全仓库 TypeScript 约 18 万行，不含测试/生成文件）
│
├── packages/ai            27.7K 行   ← 大脑的家：把各家模型 API 统一成一种形状
├── packages/agent         25.3K 行   ← ★ 发动机在这，核心只有 4 个文件：
│     ├── src/agent.ts          592 行   有状态的操作台（状态/队列/订阅）
│     ├── src/agent-loop.ts     803 行   纯函数的发动机（双层循环）
│     ├── src/stream-fn.ts       20 行   大脑插座（依赖注入）
│     └── src/types.ts          446 行   规则说明书（全部类型契约）
├── packages/coding-agent  88.2K 行   ← CLI、TUI、扩展系统（护栏+装修，最大）
├── packages/tui            19.1K 行   ← 终端界面
├── packages/chord           5.8K 行   ← RPC/传输层
├── packages/session-backends 2.2K 行  ← 会话落盘（SQLite 等）
└── packages/protocol / client / server / telemetry / evals（其余）
```

"核心代码就 1-2 千行"这个说法**对，也不对**：

- 整个项目远不止——18 万行 TypeScript，6352 个 commit；
- 但**组装四要素的循环核心，从头到尾就 4 个文件、1861 行**（592 + 803 + 20 + 446）；
- 更巧的是，2025-08-09 的初始 commit 里，整个 agent 包（连 CLI、渲染器、工具一起）加起来才约 1900 行。**一年过去，核心循环几乎没膨胀，膨胀的全是外围**——TUI、扩展系统、模型目录、沙箱。

这 4 个文件怎么串起四要素？答案是：**它们围绕"一条消息的旅程"分工**。你按下回车之后：

```
 你按下回车
     │
     ▼
┌───────────┐  轮间注入   ┌────────────┐  问大脑   ┌──────────┐
│ 排队区     │ ─────────► │ 发动机      │ ────────► │ 大脑 LLM  │
│ steering  │            │ runLoop    │ ◄──────── │ streamFn │
└───────────┘            └─────┬──────┘   回答    └──────────┘
     ▲                         │
     │                    模型要调工具吗？
     │                    ┌────┴────┐
     │               是   ▼         │ 否
     │              ┌──────────┐    │
     │              │ 手脚 工具  │    │
     │              └────┬─────┘    │
     │                   ▼          │
     │              ┌──────────┐    │
     │              │ 记事本    │◄───┘  （结果写回上下文）
     │              │ context  │
     │              └────┬─────┘
     │                   ▼
     │              ┌──────────┐
     │              │ 任务单    │── 打勾了？──► 收工 agent_end
     │              │ 终止条件  │
     │              └────┬─────┘
     │                   │ 没打勾：再来一轮
     └───────────────────┘
```

后面每一章，对应这趟旅程的一站：**规则说明书（types.ts）→ 大脑插座（stream-fn.ts）→ 发动机（agent-loop.ts）→ 操作台（agent.ts）→ 记忆的故事**。

---

## 三、第一站 types.ts（446 行）：工作台的"规则说明书"

要组装四要素，先得有规矩：什么能进记事本、大脑出错了怎么办、谁能在哪一步插手。types.ts 干的就是这个——它 446 行**全是类型和注释，一行运行时代码都没有**，却把整个系统的行为契约写死了。挑三样最重要的讲。

### 3.1 两种消息：记事本里不只有"发给模型的话"

这是整个架构的地基：

```typescript
// types.ts:317-326
export interface CustomAgentMessages {
    // Empty by default - apps extend via declaration merging
}

export type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages];
```

`AgentMessage` 是循环内部通用的消息类型，`Message` 是能发给 LLM 的严格类型（user / assistant / toolResult）。中间的 `CustomAgentMessages` 是个**空接口**，应用可以用 TypeScript 的 declaration merging 往里塞自定义消息——比如"UI 通知""渲染完成的 artifact"，这些消息**只给人看，不进模型**。

所以记事本（上下文）里混着两种消息，发往大脑之前必须过一道海关：

```typescript
// agent.ts:33-37（默认的海关实现）
function defaultConvertToLlm(messages: AgentMessage[]): Message[] {
    return messages.filter(
        (message) => message.role === "user" || message.role === "assistant" || message.role === "toolResult",
    );
}
```

不能转换的自定义消息（UI 通知之类）直接被过滤掉。**这一层抽象让"给用户看的对话"和"给模型看的对话"从此是两本账**——TUI 可以随便往上下文里插状态消息，模型永远只看到干净的三类消息。

### 3.2 "不准抛错"契约：大脑崩了，不许 throw

```typescript
// types.ts:28-32（节选）
export type StreamFn = (
    model: Model<Api>,
    context: Context,
    options?: SimpleStreamOptions,
) => AssistantMessageEventStream | Promise<AssistantMessageEventStream>;
```

注意上面那段注释里写的契约（types.ts:22-27）：

> Contract: **Must not throw** or return a rejected promise for request/model/runtime failures. Failures must be encoded in the returned stream via protocol events and a final AssistantMessage with stopReason "error" or "aborted" and errorMessage.

翻译成人话：**调用大脑失败（超时、限流、断网、token 过期），不许抛异常，必须把失败包装成一条正常的流事件**——最终消息带 `stopReason: "error"` 和一个 `errorMessage` 字段。

为什么这个约定如此关键？对比一下：

```
普通库的做法：                          pi 的做法：
                                       ┌─────────────────────────────┐
try {                                  │ 大脑崩了？不许 throw！        │
  res = await llm.call()               │ 错误编码成流事件：            │
} catch (e) {                          │   ① 流里发一个 "error" 事件   │
  handle(e)   // 每个调用点都要写       │   ② 最终消息带               │
}                                       │      stopReason: "error"    │
                                       └─────────────────────────────┘
                                       ↓
                                       循环代码里一行 try/catch 都没有
```

**循环代码因此可以零 try/catch 地写**——因为"出错"和"正常"长得一模一样，都是事件流。检查失败只需要看最后一条消息的 `stopReason`（agent-loop.ts:215-219）：

```typescript
if (message.stopReason === "error" || message.stopReason === "aborted") {
    await emit({ type: "turn_end", message, toolResults: [] });
    await emit({ type: "agent_end", messages: newMessages });
    return;
}
```

### 3.3 八个钩子：八个"插手时机"的插座

`AgentLoopConfig`（types.ts:149-294）是循环的全部扩展点，八个钩子各有明确契约（全部"must not throw"）。把它们按四要素归类，就看清了"规则说明书"的用意——**每个零件都预留了插手的口子**：

```
                     AgentLoopConfig（插座面板）
┌──────────────────────────────────────────────────────────────┐
│  大脑相关   getApiKey       每次问大脑前重新拿钥匙（token 会过期）│
│             convertToLlm    海关：内部消息 → LLM 消息            │
│  记忆相关   transformContext 裁剪记事本（上下文窗口管理）         │
│  手脚相关   beforeToolCall  出手前拦截（可以 block + 给理由）     │
│             afterToolCall   出手后改写结果（字段级覆盖）          │
│  目标相关   shouldStopAfterTurn 这轮干完就停？（优雅收工）        │
│             prepareNextTurn 下一轮开始前：换模型 / 压缩记事本     │
│  排队相关   getSteeringMessages  插话便签（干到一半时用户说话）    │
│             getFollowUpMessages  追加任务（干完再处理）           │
└──────────────────────────────────────────────────────────────┘
```

这张面板后面会反复用到——**第七、八章你会发现，压缩、会话管理这些"大功能"全是往这些插座上插的插件**。

### 3.4 工具三件套：手脚的"使用说明"

`AgentTool`（types.ts:387-412）除了 LLM 工具通用的 schema 之外，多了三样东西：

- `prepareArguments`——**参数先修再验**。模型给的原始参数可能不规范（比如该传数组传了单个值），先过这个垫片修正，再过 schema 校验；
- `replay: "never" | "safe"`——声明这个工具如果**执行到一半断了、结果未知**，能不能安全重放（改文件的副作用工具要慎重，读文件的查询工具可以）；
- `execute` 的第四个参数 `onUpdate`——工具干活过程中可以**流式上报部分结果**（types.ts:384 承诺：工具 settle 之后再调用这个回调会被忽略），UI 能实时看到"正在读第 3 个文件"。

规矩讲完了，下一站看大脑怎么插进来。

---

## 四、第二站 stream-fn.ts（20 行）：大脑是怎么"插"进来的

钩子只是插座，**大脑本身从哪来？** agent-core 这个包不 import 任何一家模型厂商——它只认"形状"。这 20 行就是全部的机制，全文贴出：

```typescript
// stream-fn.ts:11-20（全文）
export function setDefaultStreamFn(streamFn: StreamFn | undefined): void {
    defaultStreamFn = streamFn;
}

export function getDefaultStreamFn(): StreamFn {
    if (!defaultStreamFn) {
        throw new Error("No default stream function configured. Pass streamFn explicitly or call setDefaultStreamFn().");
    }
    return defaultStreamFn;
}
```

一个模块级变量 + 两个函数。设计意图全在那段注释里：**宿主程序把自己默认的模型运行时注册进来，agent-core 就不必依赖任何模型目录或兼容层**。

```
  agent-core（发动机，谁的大脑都能装）         pi-ai（大脑供应商）
  ┌────────────────────────────┐           ┌───────────────────────────┐
  │ 只认形状，不认厂商：         │           │ Models.streamSimple：      │
  │ (model, context, options)  │◄──────────│ 把 OpenAI / Anthropic /   │
  │      => 事件流              │   插上     │ Bedrock / Mistral /       │
  │                            │           │ Vertex / Gemini…          │
  │ agent.ts:222 兜底：         │           │ 统一成同一个形状           │
  │ this.streamFunction =      │           │                           │
  │   options.streamFn ??      │           │                           │
  │   getDefaultStreamFn()     │           │                           │
  └────────────────────────────┘           └───────────────────────────┘
```

谁负责"插上"？宿主——coding-agent 包的启动代码（`packages/coding-agent/src/core/sdk.ts:37`）：

```typescript
setDefaultStreamFn(streamSimple);
```

这是教科书级的依赖反转：**核心循环不依赖具体大脑，具体大脑以"形状符合"的方式注入**。所以第三方可以把 pi 的循环接上任何模型——甚至接上一个假大脑做测试。这个包之所以叫"agent-core"而不是"claude-agent"或"openai-agent"，底气全在这 20 行。

---

## 五、第三站 agent-loop.ts（上）：四个入口与主循环

先说清一个事实：这个文件**不是类，而是纯函数模块**——803 行没有一个 class，全是导出的入口函数加内部辅助函数。"类"是下一章登场的 `Agent`（agent.ts），它只是把这个模块包了一层。文件本身分三层：

```
第一层 对外入口    agentLoop / agentLoopContinue         → 返回 EventStream（边跑边发事件）
                  runAgentLoop / runAgentLoopContinue    → 返回 Promise（直接 await 到结束）
第二层 主循环      runLoop                               → 双层 while，全部核心逻辑
第三层 工具管线    executeToolCalls → prepare/execute/finalize → 事件发射
```

本章讲前两层，工具管线放下一章。

### 5.1 四个入口：流式版与直连版

`agentLoop`（agent-loop.ts:32-55）与 `runAgentLoop`（96-119 行）是同一件事的两种封装：

- `agentLoop` 创建 `EventStream`，把 emit 接到 `stream.push`，然后 **fire-and-forget** 启动循环——结束时 `stream.end(messages)` 收尾。调用方拿到的是流：可以 `for await` 事件、拿结果；
- `runAgentLoop` 是直连版：调用方自己传 `emit` 回调，返回 `Promise<AgentMessage[]>`。

流里的事件长这样（事件全集在 types.ts:431-446）：

```
agent_start
   └─► turn_start
         └─► message_start（用户消息）
             message_end
             message_start（assistant，开始流式）
             message_update × N（每个 token 一帧）
             message_end
               └─► tool_execution_start ─► tool_execution_update × N ─► tool_execution_end
                     └─► message_start（toolResult）─► message_end
                           └─► turn_end
                                 ├── 还有工具调用 / 插话？──► 回到 turn_start
                                 └── 没了 ──► agent_end（收工，附全部新消息）
```

**UI、会话存储、遥测全是旁观者**——它们订阅事件，不参与循环。发动机和仪表盘从此互不阻塞。

`runAgentLoop` 开头还有个仪式（104-115 行）：把用户 prompt 拼进 context 副本（**不修改调用方传入的 context**），然后发 `agent_start` → `turn_start` → 每条 prompt 的 `message_start/message_end`。注意 `turn_start` 在 prompt 消息**之前**发——所以第一"轮"的定义是"prompt + 第一次 assistant 回答"。

`agentLoopContinue`（65-94 行）是"续跑"入口：不加新消息、从现有上下文继续，主要用于重试。两个前置校验：

```typescript
if (context.messages.length === 0) throw new Error("Cannot continue: no messages in context");
if (context.messages[context.messages.length - 1].role === "assistant") {
    throw new Error("Cannot continue from message role: assistant");
}
```

**最后一条消息必须是 user 或 toolResult**——否则下一轮 LLM 调用没有"新的输入"可回应。注释里承认这个约束没法本地完整校验（自定义消息可能在 `convertToLlm` 时才转成 user），所以只能查最后一条的 role。

配套的 `createAgentStream`（146-151 行）只有两行，告诉 `EventStream` 两件事：什么事件算流结束（`agent_end`）、从终结事件提取什么作为结果（`agent_end` 携带的全部新消息）。

### 5.2 主循环的两本账

`runLoop`（156-273 行）开头的状态变量里藏着一个关键设计：

```typescript
let currentContext = initialContext;   // 工作上下文（会被原地 push）
let config = initialConfig;            // 循环配置（prepareNextTurn 可以替换它）
let lastCompletedTurn;                 // 上一轮完整快照
let pendingMessages = (await config.getSteeringMessages?.()) || [];  // 待注入消息
```

注意 `newMessages`（参数）和 `currentContext.messages` 是**两本账**：前者只记录"本轮循环新增的消息"，是 `agent_end` 的返回值；后者是发给模型的完整上下文。分账的目的是让调用方知道"这次 run 产生了什么"，而不用 diff 整个上下文。

### 5.3 双层 while：内层干"工具+插话"，外层管"收工后的追加任务"

核心结构（agent-loop.ts:156-177，节选）：

```typescript
async function runLoop(initialContext, newMessages, initialConfig, signal, emit, streamFunction) {
    let currentContext = initialContext;
    let config = initialConfig;
    let lastCompletedTurn: PrepareNextTurnContext | undefined;
    let pendingMessages: AgentMessage[] = (await config.getSteeringMessages?.()) || [];

    // Outer loop: continues when queued follow-up messages arrive after agent would stop
    while (true) {
        let hasMoreToolCalls = true;

        // Inner loop: process tool calls and steering messages
        while (hasMoreToolCalls || pendingMessages.length > 0) {
            // ...
        }

        // Agent would stop here. Check for follow-up messages.
        const followUpMessages = (await config.getFollowUpMessages?.()) || [];
        if (followUpMessages.length > 0) {
            pendingMessages = followUpMessages;   // 转成 pending，回到内层
            continue;
        }
        break;
    }
    await emit({ type: "agent_end", messages: newMessages });
}
```

画成结构图：

```
      ┌────────────────────────────────────────────┐
      │ 外层 while(true)：                          │
      │   "内层转完了，本应收工——还有追加任务吗？"     │
      │   followUp 队列有货 → 塞进 pending → 回去    │
      │   没货 → break → agent_end 收工             │
      │  ┌──────────────────────────────────────┐  │
      │  │ 内层 while(还有工具调用 || 有插话)：     │  │
      │  │   ① 轮间：prepareNextTurn（压缩/换模型）│  │
      │  │   ② 注入插话消息（pending，最多一条）   │  │
      │  │   ③ 问大脑 → 新的 assistant 消息      │  │
      │  │   ④ 有工具调用？→ 执行 → 结果记回记事本 │  │
      │  │   ⑤ turn_end → 任务单打勾了吗？        │  │
      │  └──────────────────────────────────────┘  │
      └────────────────────────────────────────────┘
```

**为什么要两层？** 因为"用户中途说的话"有两种语义：干到一半时说的话（steering，插话），和"这活干完了我再补一句"（follow-up，追加）。内层循环负责前者，外层循环负责后者。一个 `while` 干不了这活——插话要**尽快**处理，追加要**干完才**处理，两者的触发时机正好相反。

### 5.4 一轮的五个节拍

内层循环每一轮要依次做五件事，轮间顺序藏着设计意图。

**节拍一：prepareNextTurn 快照**（176-190 行，第一轮不跑）。返回的快照可以替换三样东西：

```typescript
currentContext = nextTurnSnapshot.context ?? currentContext;   // 整个上下文（压缩后换新的）
config = {
    ...config,
    model: nextTurnSnapshot.model ?? config.model,              // 换模型
    reasoning: /* thinkingLevel → reasoning 映射 */,
};
```

一个细节：`thinkingLevel: "off"` 被显式映射成 `undefined`——config 里 `reasoning: undefined` 才代表"不指定思考档位"，字符串 `"off"` 是给 UI 的概念。两层语义在边界处做一次翻译。

**节拍二：补捡一次插话**（194-196 行）：

```typescript
// Preparation can be long-running (for example, compaction). Pick up steering
// queued while it ran. Only poll again if the earlier poll returned nothing;
// otherwise one-at-a-time mode would deliver two messages in this turn.
if (pendingMessages.length === 0) {
    pendingMessages = (await config.getSteeringMessages?.()) || [];
}
```

注释里藏着个坑：压缩可能跑很久，期间用户可能又说话了，所以要补捡一次；**但如果循环开头已经捡到了，就不能再捡**——否则"一次只注入一条"模式下，一轮会注入两条消息。

**节拍三：注入 pending 消息**（201-209 行）：发事件、push 进两本账。时机是"问大脑之前"，所以插话会作为**最新输入**出现在下一轮请求里，而不是插进历史中间。

**节拍四：问大脑 + 错误短路**（212-219 行）：

```typescript
const message = await streamAssistantResponse(...);
if (message.stopReason === "error" || message.stopReason === "aborted") {
    await emit({ type: "turn_end", message, toolResults: [] });
    await emit({ type: "agent_end", messages: newMessages });
    return;
}
```

这就是第三章"不抛错"契约的消费端：**大脑失败不是异常，是消息上的一个字段**。检查到就补完仪式直接 return——整个主循环零 try/catch。

**节拍五：轮末收尾**（243-257 行）的顺序很讲究：先 `turn_end` → 记快照 → 问 `shouldStopAfterTurn` → 最后才捡 steering。**"宿主喊停"优先于"用户插话"**——喊停了，插话就不捡了。

### 5.5 工具调用怎么处理 + 目标怎么判定

问完大脑，看模型有没有要调工具（222-241 行）：

```typescript
const toolCalls = message.content.filter((c) => c.type === "toolCall");
if (toolCalls.length > 0) {
    const executedToolBatch =
        message.stopReason === "length"
            ? await failToolCallsFromTruncatedMessage(toolCalls, emit)
            : await executeToolCalls(currentContext, message, config, signal, emit);
    toolResults.push(...executedToolBatch.messages);
    hasMoreToolCalls = !executedToolBatch.terminate;
    // 结果逐条 push 进两本账
}
```

两个分支：`stopReason === "length"`（输出撞 token 上限）→ 整批拒绝执行；正常 → 走工具管线（下一章讲）。

**为什么"length"要整批拒绝？** 流式 tool call 参数经过"抢救解析"（best-effort JSON salvage parser），**可能解析出"校验通过但内容残缺"的参数**（比如文件路径少了最后一个字符）。执行这种参数就是闯祸。所以 `failToolCallsFromTruncatedMessage`（379-404 行）不做任何执行，对每个 tool call 发一条错误结果，措辞直接告诉模型发生了什么：

> Tool call "xxx" was not executed: the response hit the output token limit, so its arguments may be truncated. Re-issue the tool call with complete arguments.

模型收到后会带着完整参数重发。**用"失败-重试"换"执行残缺命令"的安全**。

最后是"任务单"的落地，三个地方：

1. **模型不再调工具**：没有 toolCall，且两个队列都空 → 内层循环退出；
2. **`shouldStopAfterTurn` 钩子**：每轮结束被问一次"这轮干完就停？"（252-255 行），宿主可以据此实现"上下文快满了，见好就收"的优雅停止；
3. **`terminate` 信号**：工具结果可以带 `terminate: true`，但有个微妙规则（589-591 行）：

```typescript
function shouldTerminateToolBatch(finalizedCalls: FinalizedToolCallOutcome[]): boolean {
    return finalizedCalls.length > 0 && finalizedCalls.every((finalized) => finalized.result.terminate === true);
}
```

**整批所有工具都置位才停。** 因为工具是并行执行的——一个工具想停、另一个工具正在改文件，如果立刻停，另一半活就丢了。全批置位意味着"这轮的所有手脚都认为可以收工"，才安全地停。

---

## 六、第三站 agent-loop.ts（下）：LLM 边界与工具执行管线

### 6.1 streamAssistantResponse：AgentMessage 世界的唯一出口

`streamAssistantResponse`（279-370 行）是内部消息世界到 LLM 消息世界的唯一过境口，五步走完。

**第一步，两次变换**（287-300 行）：

```typescript
let messages = context.messages;
if (config.transformContext) {
    messages = await config.transformContext(messages, signal);   // 裁剪记事本
}
const llmMessages = await config.convertToLlm(messages);          // 海关过滤
const llmContext: Context = {
    systemPrompt: context.systemPrompt,
    messages: llmMessages,
    tools: context.tools,
};
```

先裁剪再过滤，**任务单（systemPrompt）和手脚（tools）也在这里随行**——每轮请求都完整携带。

**第二步，每轮重拿钥匙**（302-310 行）：`getApiKey` 每次调用都重新解析——短命 OAuth token 可能在工具执行期间过期，所以每轮问大脑前都重拿，而不是 run 开始时拿一次。

**第三步，调 streamFunction**，拿到事件流。

**第四步，流事件状态机**。核心是 `partialMessage` + `addedPartial` 两个变量的配合：

| 事件 | 动作 |
|---|---|
| `start` | 记下 partial，**push 进 context 末尾**，发 `message_start` |
| `text_delta` 等 8 种 delta | 更新 partial，**原地替换** context 最后一条消息，发 `message_update` |
| `done` / `error` | 取最终消息：有 partial 就替换、没有就 push，发 `message_end` 并 return |

两个容易漏的边界：

1. **没有 start 直接 done 的流**（某些提供商不发 start）：`addedPartial` 为 false，走"push + 补发 message_start"的路径（352-355 行）——事件序列永远完整；
2. **`for await` 正常耗尽但没见到 done**（流被异常截断）：循环后面的兜底代码（361-369 行）再调一次 `result()`，同样走"替换或 push + message_end"。

**第五步，"活消息"语义**。delta 事件里这行：

```typescript
partialMessage = event.partial;
context.messages[context.messages.length - 1] = partialMessage;
```

context 里的 assistant 消息**逐帧生长**：任何时刻中断（abort），留下的都是最新 partial，不会出现半条消息。UI 渲染和上下文更新能共用一个事件源，靠的就是这个。

### 6.2 工具执行管线：prepare → execute → finalize

每个工具调用都走同一条管线，返回 `FinalizedToolCallOutcome`（工具调用 + 结果 + 是否错误）：

```
prepareToolCall（607-675 行）
  找工具（找不到 → 立即错误结果）
  → prepareArguments 垫片修参数
  → validateToolArguments 校验
  → beforeToolCall 钩子（可 block 拦截，拦截结果还能带 terminate）
  → abort 检查
  所有失败路径都返回"立即结果"而不是抛异常
      ↓
executePreparedToolCall（677-718 行）
  真正调 tool.execute()
  acceptingUpdates 标志：工具 settle 后再调 onUpdate 会被忽略
  并发 update 事件收进数组，最后 Promise.all 等齐
      ↓
finalizeExecutedToolCall（720-765 行）
  afterToolCall 钩子按字段覆盖结果（content/details/usage/terminate/isError，无深合并）
  钩子自己抛异常 → 结果整体替换成错误结果
```

这就是"不抛错"契约在工具侧的落地：**层层吞异常、转错误结果**，所以主循环才敢零 try/catch。

### 6.3 串行 vs 并行：三种并发策略

管线不变，变的只是编排方式。串行版（431-485 行）一个 for 循环把四步走完才轮到下一个；并行版（487-561 行）的技巧在于：prepare 阶段就定性的（被拦截、参数错误）**当场 emit 定值**，能执行的塞**延迟函数（thunk）**进数组，最后：

```typescript
const orderedFinalizedCalls = await Promise.all(
    finalizedCalls.map((entry) => (typeof entry === "function" ? entry() : Promise.resolve(entry))),
);
```

三个阶段刻意用不同的并发策略：

```
阶段一  prepare：串行
        逐个过 beforeToolCall 拦截钩子
        （拦截是"决策"，必须一个一个来，顺序可预测）

阶段二  execute：并发
        允许并行的工具用 Promise.all 一起跑
        （执行是"干活"，怎么快怎么来）

阶段三  emit：按大脑安排的原始顺序
        Promise.all 保持数组顺序，toolResult 消息
        按 assistant 消息里的 toolCall 原顺序一条条记回记事本
        （上下文是"记录"，顺序必须稳定，模型才读得懂）
```

三种并发策略对应三个目标：**决策要可预测、干活要快、记录要稳**。

### 6.4 收尾的辅助函数

- `shouldTerminateToolBatch`（589-591 行）：`every()` 全批置位才停，见 5.5 节；
- `createToolResultMessage`（784-798 行）：`content: finalized.result.content ?? []`——未类型化工具（JS 扩展）可能返回没有 content 的结果，这里归一化，防止 `null` 混进会话历史和提供商载荷；
- `emitToolResultMessage`（800-803 行）：toolResult 进上下文前也走完整的 `message_start/message_end` 仪式——**所有进上下文的消息，无一例外都发事件**，这是 UI/session 状态机一致性的保证。

### 6.5 这个文件的三条铁律

回头看，803 行的每一段都在服务三个约束：

1. **契约式不抛错**——prepare/execute/finalize 层层吞异常转错误结果，主循环零 try/catch；
2. **事件仪式完整**——任何进上下文的消息（prompt、assistant、toolResult）都发成对事件，任何退出路径都补完 turn_end/agent_end；
3. **顺序语义清晰**——决策串行、执行并发、记录按源顺序；插话轮间注入、追加收工前检查。

这些约束单独看都不难，难的是 803 行里**没有一处违反**。这就是"发动机"和"能跑起来的循环"的区别。

---

## 七、第四站 agent.ts（592 行）：操作台——状态、队列、订阅

发动机是纯函数：每次调用给它 context 和 config，它跑完发事件。**但"现在干到哪了"得有人记着**——这就是 `Agent` 类（agent.ts:173），它把无状态循环包装成一个可交互对象。四件事值得看。

### 7.1 记事本的当前态，setter 偷偷复制

```typescript
// agent.ts:68-95（节选）
function createMutableAgentState(initialState?) {
    let tools = initialState?.tools?.slice() ?? [];
    let messages = initialState?.messages?.slice() ?? [];
    return {
        get tools() { return tools; },
        set tools(nextTools: AgentTool<any>[]) { tools = nextTools.slice(); },
        get messages() { return messages; },
        set messages(nextMessages: AgentMessage[]) { messages = nextMessages.slice(); },
        ...
    };
}
```

外部代码拿到 `state` 引用后**改不了内部数组**——setter 强制复制。防止"UI 那边顺手 push 一条消息，这边上下文就乱了"这类共享可变状态的经典事故。

### 7.2 两个队列：插话便签 vs 追加任务

`PendingMessageQueue`（agent.ts:125-159）的核心是 `drain()`，支持两种模式：

```typescript
drain(): AgentMessage[] {
    if (this.mode === "all") {
        const drained = this.messages.slice();
        this.messages = [];
        return drained;          // 一次全排空
    }
    const first = this.messages[0];
    if (!first) { return []; }
    this.messages = this.messages.slice(1);
    return [first];             // 一次只取最旧一条
}
```

对应 5.3 节双层循环里的 steering / followUp 两个队列（`Agent` 构造时各建一个，agent.ts:231-232）：

```
 steering（干活中插话的便签条）：
   你："顺便把 README 也改了"
   → 排队 → 等这轮工具跑完 → 注入 → 模型看到

 followUp（干完再处理的追加任务）：
   只有 agent 本应收工时才被捞出来 → 开始新的一轮

 QueueMode 两种排空策略：
   "all"           一次全部注入
   "one-at-a-time" 一次只注入最旧一条，其余等下个轮间
                   （默认值——插话一条一条来，模型消化得过来）
```

### 7.3 单飞保护：连按回车不崩、不打断

```typescript
// agent.ts:348-357（节选）
async prompt(input: string | AgentMessage | AgentMessage[], images?: ImageContent[]): Promise<void> {
    if (this.activeRun) {
        throw new Error(
            "Agent is already processing a prompt. Use steer() or followUp() to queue messages, or wait for completion.",
        );
    }
    ...
}
```

一次只允许一个 run 在跑（`activeRun`，agent.ts:204）。用户连按回车不会打断正在跑的循环，而是得到一句提示：**用 `steer()` 排队**。这就是整套设计的哲学——**排队优先于中断**。abort 只留给显式信号（Ctrl+C），普通交互全部转成队列语义。

### 7.4 订阅者参与结算：UI 渲染完了才算真收工

```typescript
// agent.ts:328-330
waitForIdle(): Promise<void> {
    return this.activeRun?.promise ?? Promise.resolve();
}
```

注意注释里的约定（agent.ts:247-249）：`agent_end` 是最后一个事件，**但 agent 要等所有订阅者处理完 `agent_end` 之后才算 idle**。也就是说：TUI 把最后一帧渲染完、session 把最后一条消息落盘，`waitForIdle()` 才 resolve。**订阅者不是旁观的日志，而是生命周期的一部分**——它们阻塞式地参与结算。

### 7.5 失败路径：发动机崩了，也要走完仪式

```typescript
// agent.ts:511-527（节选）
private async handleRunFailure(error: unknown, aborted: boolean): Promise<void> {
    const failureMessage = {
        role: "assistant",
        content: [{ type: "text", text: "" }],
        ...
        stopReason: aborted ? "aborted" : "error",
        errorMessage: error instanceof Error ? error.message : String(error),
        ...
    } satisfies AgentMessage;
    await this.processEvents({ type: "message_start", message: failureMessage });
    await this.processEvents({ type: "message_end", message: failureMessage });
    await this.processEvents({ type: "turn_end", message: failureMessage, toolResults: [] });
    await this.processEvents({ type: "agent_end", messages: [failureMessage] });
}
```

万一有东西绕过了"不抛错"契约、run 真的抛了异常：**伪造一条空的 assistant 消息，把 message_start → message_end → turn_end → agent_end 的完整仪式走一遍**。这样 UI 的状态机永远不会卡在"正在输入……"，session 里也永远有一条带 `errorMessage` 的完整记录，而不是一个断头会话。

---

## 八、记忆的完整故事：记事本满了怎么办

前面一直说"上下文即记忆"（对照 [《企业级记忆知识库》](/2026-09-01/企业级记忆知识库-短期上下文与四层记忆的RAG向量检索实现) 里的分层记忆，那篇讲企业知识库的四层，这篇看单个 agent 会话内的三层）。但记事本无限写下去会爆——pi 的答案是分三层：

```
┌──────────────────────────────────────────────────────────┐
│ 第一层 活动上下文   AgentContext.messages                 │
│   正在用的记事本，每轮完整发给模型（或经 transformContext │
│   裁剪后发给模型）                                        │
│        │ 太长了？                                        │
│        ▼                                                │
│ 第二层 压缩 compaction                                   │
│   把老对话总结成一条摘要消息（createCompactionSummary-    │
│   Message），原始消息归档；同时记录这期间读过/改过哪些     │
│   文件（CompactionDetails.readFiles / modifiedFiles）    │
│        │ 会话结束                                        │
│        ▼                                                │
│ 第三层 落盘 session 存储（SQLite / 自定义后端）           │
│   重开终端还能接着聊；支持分支、回放、导出 HTML            │
└──────────────────────────────────────────────────────────┘
```

第二层挂在哪？还记得三章的插座面板吗——**就挂在 `prepareNextTurn` 上**。coding-agent 的 `agent-session.ts:559-580` 把这个钩子包装了一层：

```typescript
this.agent.prepareNextTurnWithContext = async (turn, signal) => {
    const context = await this._compactBeforeNextAssistantResponse(turn.context);
    const previousSnapshot = await previousPrepareNextTurnWithContext?.({ ...turn, context }, signal);
    const nextContext = previousSnapshot?.context ?? context;

    return {
        ...previousSnapshot,
        context: {
            ...nextContext,
            systemPrompt: this._systemPromptOverride ?? this._baseSystemPrompt,
            tools: this.agent.state.tools.slice(),
        },
        model: this.agent.state.model,
        thinkingLevel: this.agent.state.thinkingLevel,
    };
};
```

翻译一下：**每轮开始前，先检查记事本满没满（阈值触发 `_runAutoCompaction("threshold")`），满了就先把老对话压缩成摘要、再拿压缩后的上下文去问大脑**。压缩、换模型、换思考档位——这些"大功能"对发动机来说，全都只是"下一轮开始前的例行准备"。

这和 [《Agent 循环与目标》](/2026-08-29/Agent循环与目标-一个Agent到底是怎么自己跑起来的) 里讲的"压缩是记忆管理"结论完全对上——那篇讲原理，这里你能看到原理在真实工程里的挂载点：**一行 `prepareNextTurn`，就是记忆系统的全部入口**。

---

## 九、总结：回到四要素，一张总图

把全文收束成一张对照大图：

```
四要素        源码文件               关键机制
──────────────────────────────────────────────────────────
大脑  →   stream-fn.ts (20行)     插座式注入：核心不认厂商
          pi-ai streamSimple      统一各家 API 成一个形状
          "不抛错"契约             错误编码成 stopReason
手脚  →   types.ts AgentTool      prepareArguments 先修再验
          agent-loop.ts 执行       截断保护：length 整批拒绝
                                  决策串行/干活并发/记录有序
记事本 →  agent.ts state           setter 复制防偷改
          agent-loop.ts context    流式原地更新最后一条
          prepareNextTurn 钩子     压缩（compaction）挂载点
任务单 →  systemPrompt             任务描述
          shouldStopAfterTurn      优雅收工
          terminate 全批置位        并行下安全停止
──────────────────────────────────────────────────────────
发动机 →  agent-loop.ts (803行)    入口/主循环/工具管线三层
操作台 →  agent.ts (592行)         队列/单飞/订阅结算
```

**四个可以抄走的设计**：

1. **两层消息抽象**——`AgentMessage`（内部，可扩展）→ `Message[]`（LLM 边界，严格），中间过海关 `convertToLlm`。"给用户看的"和"给模型看的"是两本账；
2. **契约式不抛错**——所有用户钩子承诺不 throw，失败编码进事件流。循环代码因此零 try/catch，错误处理收敛在一个地方；
3. **纯循环 + 有状态壳**——发动机是纯函数（可单测、可复用），状态、队列、订阅全在壳里；
4. **排队优先于中断**——用户输入永远排队（steer/followUp），不 abort 正在跑的循环。

**三个诚实的弱点**：

1. "must not throw"是注释里的契约，不是类型系统保证——违反了循环会静默中断事件流，不会报错；
2. `CustomAgentMessages` 的 declaration merging 是**全局的**，两个第三方扩展可能互相冲突；
3. `defaultStreamFn` 是进程级单例——一个进程只能有一个默认大脑（CLI 无所谓，嵌入场景必须显式传 streamFn）。

回到开篇的工位比喻收尾：**组装四要素的"工作习惯"，1860 行足矣；剩下的 18 万行，都是在给这个习惯加护栏和装修**——TUI 是装修，扩展系统是护栏，模型目录是大脑的说明书。下次你写自己的 agent 时，先问问：我的"工作习惯"这 2000 行，写清楚了吗？

---

**相关阅读**

- [《Agent 循环与目标：一个 Agent 到底是怎么自己跑起来的》](/2026-08-29/Agent循环与目标-一个Agent到底是怎么自己跑起来的)——本篇的原理篇：循环、压缩、目标判定
- [《AgentScope 详解：多智能体框架从 ReAct 循环到企业级 Harness》](/2026-09-04/AgentScope详解-多智能体框架从ReAct循环到企业级Harness)——另一个框架的做法：配置+插件驱动 vs pi 的代码即编排
- [《企业级记忆知识库：短期上下文与四层记忆的 RAG 向量检索实现》](/2026-09-01/企业级记忆知识库-短期上下文与四层记忆的RAG向量检索实现)——记忆的分层视角，与本篇第七节互补
- [《Transformer 解码器：大模型是怎么"一个字一个字"写出来的》](/2026-09-05/Transformer解码器-大模型一字一句生成的工作原理详解)——大脑内部：streamFn 收到请求后发生了什么
