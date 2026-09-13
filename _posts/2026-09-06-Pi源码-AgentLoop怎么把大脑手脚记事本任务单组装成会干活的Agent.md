---
layout: post
title: "拆解 pi 最初的两千行：还没长护栏的 Agent Loop，怎么把“大脑、手脚、记事本、任务单”组装成会干活的 Agent"
author: "Inela"
---

前面写过 [《Agent 循环与目标》](/2026-08-29/Agent循环与目标-一个Agent到底是怎么自己跑起来的)（原理篇）和 [《AgentScope 详解》](/2026-09-04/AgentScope详解-多智能体框架从ReAct循环到企业级Harness)（另一个框架的实现）。这篇钻进 pi 的源码——而且钻的不是今天的 pi，是它**出生的第一天**。

一句话贯穿全文，后面所有内容都在给这句话做注脚：

> **Agent = 大脑 + 手脚 + 记事本 + 任务单。Agent loop 就是把这些零件组装起来的"工作习惯"：看一眼任务单 → 翻记事本 → 决定自己答还是动手 → 动手 → 把结果记回记事本 → 再看一眼……直到任务单打勾。**

今天的 pi（Armin Ronacher 的项目，104K star）全仓库约 18 万行 TypeScript。我把仓库拉了个新分支，回退到最初提交 `a74c5da11`（2025-08-09）：那时的 pi 只有 3 个包、约 6000 行，agent 包约 1900 行——**整个 agent loop 就长在两个 while 循环加一个 class 里**。

```
                      pi 的两个版本
┌─────────────────────────────┬──────────────────────────────┐
│      今天的 pi（18 万行）    │    最初的 pi（a74c5da11）     │
├─────────────────────────────┼──────────────────────────────┤
│ 核心 4 文件 1860 行          │ agent 包全部约 1900 行        │
│ 8 个钩子、流式、并行、压缩…  │ 2 个 while 循环 + 1 个 class  │
│ 双层消息抽象、事件流、队列…  │ messages: any[] + 10 种事件   │
└─────────────────────────────┴──────────────────────────────┘
```

为什么拆"旧版"而不是"新版"？因为新版太安全了：18 万行里绝大部分是**护栏**（钩子、压缩、截断保护、队列……），发动机被包得严严实实。旧版没有护栏，发动机裸露在外面——四要素是怎么被组装起来的，看得一清二楚。

顺带说一句考古发现：最初提交里三个包的包名还挂在 `@mariozechner` scope 下（就是 libGDX 的作者 Mario Zechner），如今项目在 Flask 作者 Armin Ronacher 的 earendil-works 名下。

这篇博客按"**认零件 → 看装配 → 看痕迹 → 看刹车 → 看缺的护栏**"的顺序走：

- 〇章：最小版的四要素分别长什么样（裸装零件对照表）
- 一章：组装它们的两个 while 循环，逐帧拆解（核心章）
- 二章：循环的每一步留下的"痕迹"——事件系统，就是记事本的实现
- 三章：最小版唯一的刹车——中断
- 四章：数一数缺了哪些护栏，以及最小版仅有的"野生护栏"
- 五章：总结

---

## 〇、先看一眼这个工位：最小版的四要素

想象一个工位，上面摆着四样东西：

```
┌─────────────────────────── 工位 = Agent ───────────────────────────┐
│                                                                    │
│  大脑 LLM              手脚 工具              记事本 记忆           │
│  "什么都懂，但碰不到     "能读文件、跑命令、     "刚才说过什么、      │
│   现实世界，只会说话"     查文档、搜代码"        改过哪些文件，      │
│                                                全记在这"           │
│                                                                    │
│  任务单 目标                                                        │
│  "用户到底要什么？干到什么程度，才算把活干完？"                     │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

四样东西摆在那，工位还是不会干活。**缺的是"工作习惯"**——先看哪个、看完怎么判断、动手之后结果放哪、什么时候算干完。这套"习惯"，就是 agent loop。

在最小版 pi 里，四要素各有自己的"实体"，而且都很朴素：

| 零件 | 通俗说法 | 最小版 pi 里的实体 | 位置 |
|---|---|---|---|
| 大脑 | 只会说话，碰不到现实 | OpenAI SDK + 两个"打电话"函数 `callModelResponsesApi` / `callModelChatCompletionsApi` | agent.ts:43-272 |
| 手脚 | 能改文件、跑命令 | 5 个工具：`read` / `list` / `bash` / `glob` / `rg` | tools.ts（264 行） |
| 记事本 | 刚才说过什么全记下 | 内存剧本 `this.messages: any[]` + 磁盘流水账 JSONL 文件 | agent.ts:277 / session-manager.ts |
| 任务单 | 用户要什么、何时算完 | `systemPrompt` + 循环出口（`conversationDone` / `assistantResponded`） | agent.ts:52 / agent.ts:188 |

注意最小版"任务单"有多朴素：它**没有多目标追踪、没有长期计划**，任务单上只有一个格子——**把用户这句话答完**。`conversationDone` 和 `assistantResponded` 这两个布尔值，就是"任务单打勾"。

```
最初提交 a74c5da11 只有 3 个包：
  packages/tui     pi-tui    终端 UI 库（差分渲染）
  packages/agent   pi-agent  通用 agent（工具调用 + 会话持久化）★ 本篇主角
  packages/pods    pi        vLLM 部署 CLI（在 GPU 机器上拉起模型）
  依赖方向：pi-tui ← pi-agent ← pi
```

为什么会有 pods 这个包？因为 pi 出生时的目标就是**自己部署 GPT-OSS 模型**（vLLM 部署 + Responses API 驱动）。这也解释了 agent 里为什么有两个"打电话"函数——下一章的主角。

零件认全了。接下来看装配：**两个 while 循环，怎么把这四样东西串成一套工作习惯**。

---

## 一、循环长在"打电话"的函数里

最小版没有独立的 agent-loop.ts 文件。循环**直接长在两个 API 调用函数里**：一个函数就是一个 while 循环，整个 agent 的全部"工作习惯"就写在里面。

### 1.1 为什么有两个循环

因为大脑有两种"电话线路"，代码里各写了一个适配：

- **Responses API**（`client.responses.create`）：给 GPT-OSS 这类新模型用（OpenAI 的新接口，支持 reasoning 思考过程）
- **Chat Completions API**（`client.chat.completions.create`）：给通用模型用（OpenAI 老接口，兼容性最好）

cli.ts 里用 `--api responses|completions` 参数切换（默认 `completions` + 模型 `gpt-5-mini`）。两个函数的循环骨架一模一样，只有"剧本格式"和"循环出口"不同。

### 1.2 responses 循环：逐帧拆解

先看全景流程图（agent.ts:43-177）：

```
        callModelResponsesApi(client, model, messages, signal, eventReceiver)
                            │
               ┌────────────▼─────────────┐
               │ 报 assistant_start（开灯） │
               └────────────┬─────────────┘
                            │
                 ┌──────────▼──────────┐
                 │ while (!conversationDone) │   ← 循环出口 = 任务单打勾
                 └──────────┬──────────┘
                            │
               ┌────────────▼─────────────┐
               │ ① 进门查 interrupt        │──已中断──▶ 报 interrupted + throw
               └────────────┬─────────────┘
                            │ 没中断
               ┌────────────▼─────────────┐
               │ ② responses.create(       │
               │     input: messages,      │   ← 记事本全量递给大脑
               │     tools: 5 个工具,      │
               │     tool_choice: "auto",  │
               │     parallel_tool_calls,  │
               │     reasoning: medium,    │
               │     max_output_tokens:2000│
               │   )                       │
               └────────────┬─────────────┘
                            │
               ┌────────────▼─────────────┐
               │ ③ 报 token_usage          │
               └────────────┬─────────────┘
                            │
          ┌─────────────────▼───────────────────┐
          │ ④ for item of response.output       │
          │    switch (item.type)               │
          └──┬────────────┬────────────┬────────┘
             │reasoning   │message     │function_call
      ┌──────▼─────┐┌─────▼──────────┐┌▼─────────────────────┐
      │报 thinking ││报 assistant_   ││报 tool_call           │
      │(思考过程)  ││message         ││→ executeTool() 干活   │
      │            ││conversationDone││→ 报 tool_result      │
      │            ││ = true 打勾    ││→ 结果回填记事本       │
      └────────────┘└────────────────┘└──────────────────────┘
                    回到 while 顶：打勾了就退出
```

核心代码（agent.ts:61-75）：

```ts
const response = await client.responses.create(
    {
        model,
        input: messages,
        tools: toolsForResponses as any,
        tool_choice: "auto",
        parallel_tool_calls: true,
        reasoning: {
            effort: "medium", // Use auto reasoning effort
            summary: "auto",
        },
        max_output_tokens: 2000, // TODO make configurable
    },
    { signal },
);
```

逐条看这次"打电话"的参数：

- `input: messages`——**记事本全量递过去**。最小版没有压缩，整个对话历史每次都原样发一遍，上下文只增不减；
- `tools: toolsForResponses`——把 5 个工具的 JSON Schema 告诉大脑："你有这 5 只手可用"；
- `tool_choice: "auto"`——大脑自己决定这轮是"动嘴回答"还是"动手调工具"；
- `parallel_tool_calls: true`——允许大脑一次要多个工具；
- `reasoning`——让模型输出思考过程，这就是屏幕上"thinking"灰色文字的来源；
- `max_output_tokens: 2000`——输出上限，注释写着 `// TODO make configurable`。**这个 TODO 后来长成了现代版完整的长度截断保护**；
- 第二个参数 `{ signal }`——中断的钩子，三章的主角。

拿到响应后，遍历 `response.output` 数组，每个 item **先入账、再反应**（agent.ts:93-121）：

```ts
for (const item of output) {
    // gpt-oss vLLM quirk: need to remove type from "message" events
    if (item.id === "message") {
        const { type, ...message } = item;
        messages.push(item);
    } else {
        messages.push(item);
    }

    switch (item.type) {
        case "reasoning": { ... 报 thinking 事件 ... }
        case "message": {
            ... 报 assistant_message 事件 ...
            conversationDone = true;   // 任务单打勾
        }
        case "function_call": { ... 干活，见下 ... }
        default: { 报 error 事件 }
    }
}
```

注意两件事：

1. **先入账再反应**：`messages.push(item)` 发生在 switch 之前。不管大脑这轮干了什么（思考、回答、要工具），剧本（记事本）都先记下来——这是"工作习惯"和"记事本"的耦合点；
2. `message` 分支里 `conversationDone = true` 就是任务单打勾：**大脑给出最终回答 = 活干完了**。最小版判断"目标完成"的方式就这么朴素。

`function_call` 分支是"手脚"的启动点（agent.ts:124-167，精简）：

```ts
case "function_call": {
    if (signal?.aborted) { ... throw new Error("Interrupted"); }

    try {
        await eventReceiver?.on({ type: "tool_call", toolCallId: item.call_id || "", name: item.name, args: item.arguments });
        const result = await executeTool(item.name, item.arguments, signal);
        await eventReceiver?.on({ type: "tool_result", toolCallId: item.call_id || "", result, isError: false });

        // 结果回填记事本
        const toolResultMsg = {
            type: "function_call_output",
            call_id: item.call_id,
            output: result,
        } as ResponseFunctionToolCallOutputItem;
        messages.push(toolResultMsg);
    } catch (e: any) {
        await eventReceiver?.on({ type: "tool_result", ..., result: e.message, isError: true });
        const errorMsg = {
            type: "function_call_output",
            call_id: item.id,          // ← 注意：成功用 call_id，失败用 id
            output: e.message,
            isError: true,
        };
        messages.push(errorMsg);
    }
    break;
}
```

大脑说"动手"，就调 `executeTool()`（tools.ts 的分发函数）真正去读文件、跑命令；结果无论成败都**回填进记事本**——工具失败了也不崩溃，把错误信息作为 `isError: true` 的结果喂回给大脑，让它自己看着办。这是 agent loop 里最重要的一条工作习惯：**动手的结果必须记回记事本，循环才能继续转**。

### 1.3 chat 循环：换一套剧本，同一个习惯

`callModelChatCompletionsApi`（agent.ts:179-272）的循环出口叫 `assistantResponded`：

```
        while (!assistantResponded)
                  │
                  ▼
        chat.completions.create(messages, tools, max_completion_tokens: 2000)
                  │
                  ▼
        message = response.choices[0].message
                  │
        ┌─────────┴──────────┐
        │ 有 tool_calls?     │ 有 content?
        ▼                    ▼
   1. assistant 消息     报 assistant_message
      （带 tool_calls）    → 入账
      入账                assistantResponded = true
   2. for 每个 toolCall    → 退出循环
      顺序执行
      → 回填 tool 消息
   3. 回到 while 顶
      （再问一轮）
```

骨架一致：进门查中断 → 打电话 → 记 token → **要么动手（tool_calls 分支）、要么回答（content 分支）**。但有两处细节和 responses 版不同：

1. **动手前先把"要动手的宣言"入账**（agent.ts:224-229）：chat 格式要求 assistant 的 tool_calls 消息必须先于 tool 结果出现，所以代码先把 `{role:"assistant", tool_calls:[...]}` push 进 messages，再逐个执行工具；
2. 工具执行是**纯串行**的 `for` 循环（agent.ts:232-263），并且兼容两种工具调用形状（agent.ts:240-241）：

```ts
const funcName = toolCall.type === "function" ? toolCall.function.name : toolCall.custom.name;
const funcArgs = toolCall.type === "function" ? toolCall.function.arguments : toolCall.custom.input;
```

两个函数的循环结构，翻译成"工作习惯"就是同一套口诀：

```
看一眼任务单（while 条件没打勾）
  → 翻记事本（messages 全量入参）
  → 问大脑（create）
  → 大脑说"动手" → 手脚干活 → 结果记回记事本 → 回到第一步
  → 大脑说"答完了" → 任务单打勾 → 收工
```

### 1.4 最小版循环的三个"老实"

跟现代版一比，最小版的循环有三个特别"老实"的地方：

**① 无流式——屏幕只能放转圈圈。**

`create` 是一次性 HTTP 请求，拿到完整响应才开始处理。所以界面上没有任何逐字输出，只有 tui-renderer.ts 里的一个帧动画 spinner（tui-renderer.ts:14-31）：

```ts
private frames = ["⠋", "⠙", "⠹", "⠸", "⠼", "⠴", "⠦", "⠧", "⠇", "⠏"];
...
start() {
    this.updateDisplay();
    this.intervalId = setInterval(() => {
        this.currentFrame = (this.currentFrame + 1) % this.frames.length;
        this.updateDisplay();
    }, 80);
}
```

现代版用 stream-fn.ts 统一各家流式接口、token 级逐字渲染；最小版只能每 80ms 换一个盲文字符。**用户盯着 spinner 等——这其实是"无流式"时代 agent 体验的全部。**

**② 无并行——`parallel_tool_calls: true` 是"假并行"。**

那个参数只允许**模型**一次响应里要多个工具；代码拿到多个 function_call 后，还是 `for` 循环**一个一个顺序执行**。模型一口气要 3 个工具，第 2 个就得等第 1 个跑完。现代版用 thunk + `Promise.all` 并行执行并保持原顺序，最小版没这个待遇。

**③ 无长度保护——2000 是硬编码。**

`max_output_tokens: 2000` 两处都写着 `// TODO make configurable`。输出超了直接截断，没有现代版"length 整批拒绝 + 截断保护"那套机制。

### 1.5 最小版专属的三个坑

只有旧版有、后来都修掉的坑，值得单独拎出来：

**坑一：注释在骗人（agent.ts:94-100）。**

```ts
// gpt-oss vLLM quirk: need to remove type from "message" events
if (item.id === "message") {
    const { type, ...message } = item;
    messages.push(item);   // ← 解构了个寂寞：push 的还是原对象
} else {
    messages.push(item);
}
```

注释说要"把 message 事件的 type 字段删掉"，代码也确实解构了 `{ type, ...message }`，但 push 的是**原对象 `item`**——`message` 变量根本没被用到，是一段死代码。碰巧 vLLM 对这个多余字段不挑剔，所以一直没炸。顺带一提，判断用的还是 `item.id === "message"`：gpt-oss 的 vLLM 服务给 message 事件的 id 填的是字符串 "message" 而不是正常的 msg_xxx，所以只能这么认。

**坑二：成功和失败用了不同的字段（agent.ts:146-165）。**

成功的回填用 `call_id: item.call_id`，失败的回填用 `call_id: item.id`。对 function_call 类型的 item 来说，`id` 和 `call_id` 并不总是一回事——工具失败时回填的 `call_id` 可能对不上号。这是个真实的 bug，最小版就这样带着它跑了很久。

**坑三：chat 循环可能原地转圈（agent.ts:222-270）。**

chat 版的分支只有 `if (tool_calls)` 和 `else if (content)` 两个。如果模型返回一条**既没有 tool_calls、content 又是空**的消息（某些模型在某些情况下会这样），`assistantResponded` 永远是 false，`while` 循环原地转圈，一直打到 token 耗尽。没有 else 兜底——这就是"无护栏"的代价：**一个边界情况就能让工作习惯变成死循环**。

装配看完了。但这一章里到处都是的 `eventReceiver?.on(...)` 不是装饰——**它们是整个记忆系统的入口**。下一章专门讲：循环的每一步怎么留下痕迹，痕迹又怎么变成"记事本"。

---

## 二、事件即真相：记事本的实现

最小版 pi 里有个贯穿一切的设计：**agent 的每一个动作，都先变成一条事件（AgentEvent），再分发给所有关心它的人**。循环负责"干活"，事件负责"记账"——记事本就是这么实现的。

### 2.1 一条事件，三个消费者

```
                 Agent（agent.ts）
                      │
                      │ 每个动作都变成一条 AgentEvent
                      ▼
              comboReceiver.on(event)
                 ┌────┴────┐
                 ▼         ▼
          renderer.on   sessionManager.on
          (画到屏幕)     (appendFileSync 写进 JSONL)
                 │         │
              TUI/控制台   ~/.pi/sessions/--cwd--/xxx.jsonl
```

**事件是"流水账"（给人看、给磁盘存），messages 是"剧本"（给模型看）**。两本账：剧本随循环实时改写，流水账只追加、永不修改。最小版靠一套手工映射在两者之间翻译（`setEvents`，见 2.5）——现代版的两层消息抽象 + `convertToLlm` 海关，就是这套手工翻译的工业化版本。

### 2.2 十种事件

AgentEvent 是一个联合类型，最小版一共 10 种（agent.ts:6-23）：

| 事件 | 什么时候发 | 谁在听、干什么 |
|---|---|---|
| `session_start` | 构造 Agent 时（仅当有会话管理） | 记录本次会话的配置快照 |
| `assistant_start` | 每轮调用大脑前 | TUI 开 spinner + 锁编辑器；恢复会话时重置状态 |
| `thinking` | 模型吐思考过程（reasoning） | TUI 灰色小字显示 |
| `tool_call` | 决定调工具、动手之前 | TUI 黄色打印 `[tool] read(...)` |
| `tool_result` | 工具跑完（成功或失败） | TUI 灰色打印结果（最多 10 行） |
| `assistant_message` | 模型给出最终回答 | TUI 用 Markdown 渲染 |
| `error` | 各种错误（拒绝回答、未知输出类型…） | TUI 红字显示 |
| `user_message` | 用户每说一句 | 回显 + 写进会话文件 |
| `interrupted` | 用户按了 Esc | TUI 回显 `[Interrupted by user]` |
| `token_usage` | 每次调用后 | TUI 底栏 `↑↓⟲⟳` 计数 |

这就是最小版的全部"通信协议"——**引擎和界面、引擎和磁盘之间，只靠这 10 种事件对话**。

### 2.3 comboReceiver：一个插线板

Agent 里没有复杂的订阅系统，只有一个"插线板"（agent.ts:294-299）：

```ts
this.comboReceiver = {
    on: async (event: AgentEvent): Promise<void> => {
        await this.renderer?.on(event);
        await this.sessionManager?.on(event);
    },
};
```

一条事件进来，先给渲染器画屏幕，再给会话管理器写磁盘，两个消费者按顺序各收一份。**新增一个消费者 = 在这个对象里加一行**。现代版的 EventStream 订阅系统，就是这个插线板长大后的样子。

### 2.4 session-manager：追加式 JSONL

磁盘上的"记事本"是追加式 JSONL 文件。每个工作目录有自己的会话目录（session-manager.ts:59-69）：

```ts
private getSessionDirectory(): string {
    const cwd = process.cwd();
    const safePath = "--" + cwd.replace(/^\//, "").replace(/\//g, "-") + "--";

    const piConfigDir = resolve(process.env.PI_CONFIG_DIR || join(homedir(), ".pi"));
    const sessionDir = join(piConfigDir, "sessions", safePath);
    if (!existsSync(sessionDir)) {
        mkdirSync(sessionDir, { recursive: true });
    }
    return sessionDir;
}
```

路径翻译很有趣：cwd 里的 `/` 全部换成 `-`，首尾再包上 `--`：

```
/opt/code/idea/workspace-github/pi
        ↓
~/.pi/sessions/--opt-code-idea-workspace-github-pi--/2025-08-09T12-00-00-000Z_3f2a....jsonl
                                                   └────── 时间戳_uuid ──────┘
```

写入方式极简——每个事件一行 JSON，同步追加（session-manager.ts:124-131）：

```ts
async on(event: AgentEvent): Promise<void> {
    const entry: SessionEvent = {
        type: "event",
        timestamp: new Date().toISOString(),
        event: event,
    };
    appendFileSync(this.sessionFile, JSON.stringify(entry) + "\n");
}
```

文件长这样：

```
{"type":"session","id":"3f2a...","timestamp":"...","cwd":"/opt/...","config":{...}}
{"type":"event","timestamp":"...","event":{"type":"user_message","text":"帮我看看 package.json"}}
{"type":"event","timestamp":"...","event":{"type":"tool_call","toolCallId":"call_1","name":"read","args":"{\"path\":\"package.json\"}"}}
{"type":"event","timestamp":"...","event":{"type":"tool_result","toolCallId":"call_1","result":"{...}","isError":false}}
```

为什么用**追加式**而不是"整个状态存一份"？三个好处：崩溃安全（写一半不影响已有内容）、无需重写全文件、恢复时按行顺序读就是完整历史。`getSessionData()` 恢复时逐行解析：`session` 行取配置，`event` 行累积进事件列表，最后一条 `token_usage` 就是累计用量。

恢复入口在 cli.ts：`--continue` 参数让 SessionManager 找当前目录**最近修改**的会话文件，然后干两件事（cli.ts:179-189）：

```ts
if (sessionData) {
    agent.setEvents(sessionData ? sessionData.events.map((e) => e.event) : []);
    for (const sessionEvent of sessionData.events) {
        const event = sessionEvent.event;
        if (event.type === "assistant_start") {
            renderer.renderAssistantLabel();   // 历史重放时不启动 spinner
        } else {
            await renderer.on(event);
        }
    }
}
```

先 `setEvents` 把历史事件翻译回 messages（给模型看的剧本），再把事件**重放**给渲染器（把历史画到屏幕上）。有个细节：`assistant_start` 在重放时被特殊处理成静态标签——因为正常渲染它会启动 spinner 动画，历史画面里不该有一堆转圈圈。

### 2.5 setEvents：把流水账翻译回剧本

`setEvents`（agent.ts:366-483）是两本账之间的手工翻译器，因为**两套 API 的剧本格式完全不同**：

```
                ┌── api = "responses" ──▶ 直接顺序翻译：
                │      user_message  → {type:"user", content:[{input_text}]}
                │      thinking      → {type:"reasoning", ...}
                │      tool_call     → {type:"function_call", ...}
  事件流水账 ───┤      tool_result   → {type:"function_call_output", ...}
                │      assistant_message → {type:"message", ...}
                │
                └── api = "chat" ────────▶ 带状态机（见下图）：
                      因为 chat 格式要求 tool 消息前面必须有
                      对应的 assistant(tool_calls) 消息，
                      而事件流里只有零散的 tool_call / tool_result
```

chat 格式的翻译是个小状态机，靠一个 `pendingToolCalls` 数组攒着（agent.ts:430-472）：

```
chat 格式重建（事件流 → messages）
  assistant_start ──▶ 清空 pendingToolCalls
  tool_call ────────▶ 累积进 pendingToolCalls（先不落账）
  tool_result ──────▶ 如果 pendingToolCalls 非空：
                         ① 一次性 push {role:"assistant", tool_calls:[...]}
                         ② 清空 pendingToolCalls
                         ③ push {role:"tool", tool_call_id, content}
  assistant_message ─▶ push {role:"assistant", content}
  thinking/error/interrupted/token_usage ─▶ 跳过
```

对应的代码片段（agent.ts:430-472，精简）：

```ts
let pendingToolCalls: any[] = [];

for (const event of events) {
    switch (event.type) {
        case "assistant_start":
            pendingToolCalls = [];
            break;
        case "tool_call":
            pendingToolCalls.push({
                id: event.toolCallId,
                type: "function",
                function: { name: event.name, arguments: event.args },
            });
            break;
        case "tool_result":
            if (pendingToolCalls.length > 0) {
                this.messages.push({
                    role: "assistant",
                    content: null,
                    tool_calls: pendingToolCalls,
                });
                pendingToolCalls = [];
            }
            this.messages.push({
                role: "tool",
                tool_call_id: event.toolCallId,
                content: event.result,
            });
            break;
        // ...
    }
}
```

这个翻译器是**有损的**：responses 格式能还原 reasoning（thinking 事件翻译成 reasoning item），chat 格式直接丢弃 thinking；`interrupted`、`error` 两种事件两边都不翻译。**流水账里记了十种事件，翻译回剧本时只保住了六种**——事件溯源在最小版里已经丢信息了。现代版用类型化消息 + 完整事件流堵上了这些洞。

记忆讲完了。但一个只会转的循环是危险的——**最小版唯一的刹车，是中断**。下一章看这条刹车线怎么从键盘一路通到子进程。

---

## 三、中断：最小版唯一的刹车

### 3.1 全景链路

用户按一下 Esc，信号怎么一路传导？整条链路是这样的：

```
  Esc 键（TUI 原始模式收到 \x1b）
   │
   ▼
  tui-renderer.ts 全局按键回调
   │  只当 currentLoadingAnimation 存在时拦截
   ▼
  onInterruptCallback ──▶ agent.interrupt()   （cli.ts 里接线）
   │                          │
   │                          ▼
   │                   abortController.abort()
   │                          │  AbortSignal 传导
   │            ┌─────────────┴──────────────┐
   │            ▼                            ▼
   │   ① 循环进门检查             ② executeTool → execWithAbort
   │   （while 顶部 if aborted）    （spawn 时传 signal）
   │   → 报 interrupted + throw    → child.kill("SIGTERM")
   │            └─────────────┬──────────────┘
   │                          ▼
   │                ask() 的 catch 吞掉 "Interrupted"
   ▼
  spinner 停、编辑器解锁、界面回显 [Interrupted by user]
```

一张 `AbortController` 的 `signal`，同时喂给 HTTP 请求和子进程——这就是最小版的全部刹车系统。

### 3.2 ask() 的一次性刹车片

Agent 每次 `ask()` 都造一个新的 AbortController，用完即弃（agent.ts:322-360，精简）：

```ts
async ask(userMessage: string): Promise<void> {
    // 渲染用户消息 + 入账
    this.comboReceiver.on({ type: "user_message", text: userMessage });
    const userMsg = { role: "user", content: userMessage };
    this.messages.push(userMsg);

    // Create a new AbortController for this chat session
    this.abortController = new AbortController();

    try {
        if (this.config.api === "responses") {
            await callModelResponsesApi(this.client, this.config.model, this.messages, this.abortController.signal, this.comboReceiver);
        } else {
            await callModelChatCompletionsApi(this.client, this.config.model, this.messages, this.abortController.signal, this.comboReceiver);
        }
    } catch (e: any) {
        // Check if this was an interruption
        if (e.message === "Interrupted" || this.abortController.signal.aborted) {
            return;   // ← 中断不是错误，静默收工
        }
        throw e;     // ← 真错误继续抛，由 cli 上层显示
    } finally {
        this.abortController = null;
    }
}
```

设计上三个要点：

1. **一次一问一刹车片**：每次 ask 的 AbortController 是独立的，中断只影响当前这一轮，`finally` 里置空，不污染下一轮；
2. **中断不是错误**：catch 里专门检查 `"Interrupted"`（循环里 throw 的约定错误）或 signal 已中断，命中就静默 return——用户按 Esc 是正常操作，不是事故；
3. **真错误照抛**：网络挂了、API 报错等，re-throw 给 cli.ts 的 catch，最终以 `error` 事件显示红字。

### 3.3 工具侧：SIGTERM 杀子进程

工具执行侧的 `execWithAbort`（tools.ts:101-169）是这个刹车系统最容易被忽略的一半：

```ts
async function execWithAbort(command: string, signal?: AbortSignal): Promise<string> {
    return new Promise((resolve, reject) => {
        const child = spawn(command, {
            shell: true,
            signal,
        });
        ...
        child.on("close", (code) => {
            if (signal?.aborted) {
                reject(new Error("Interrupted"));
            } else if (...) { ... }
        });

        // Kill the process if signal is aborted
        if (signal) {
            signal.addEventListener(
                "abort",
                () => {
                    child.kill("SIGTERM");
                },
                { once: true },
            );
        }
    });
}
```

`signal` 双保险：既直接传给 `spawn`，又手动监听 abort 事件补一发 `child.kill("SIGTERM")`。**如果没有这半截，按 Esc 只能打断 HTTP 请求，打断不了正在跑的 `npm install`**——agent 就会"以为停了，其实子进程还在后台干活"。

### 3.4 界面侧：Esc 的拦截条件

TUI 里 Esc 不是无条件生效的（tui-renderer.ts:106-130，精简）：

```ts
if (data === "\x1b" && this.currentLoadingAnimation) {
    if (this.onInterruptCallback) {
        this.onInterruptCallback();
    }
    if (this.currentLoadingAnimation) {
        this.currentLoadingAnimation.stop();
        this.statusContainer.clear();
        this.currentLoadingAnimation = null;
    }
    this.editor.disableSubmit = false;
    return false;
}
```

`this.currentLoadingAnimation` 非空 = **正在处理中**。只有转圈圈亮着的时候 Esc 才被拦截；空闲时 Esc 是普通按键，直接放行给编辑器。这个"用 spinner 是否存在当作忙闲状态"的小技巧，是最小版的状态管理：**没有状态机，spinner 就是状态**。

配套的还有两个输入控制细节：

- 处理中 `editor.disableSubmit = true`——TUI 模式下干脆锁键盘，不让用户在 agent 干活时再提交新消息；
- Ctrl+C 单按清空编辑器、500ms 内双击退出进程（`lastSigintTime` 记上次时间戳）。

对比一下：**TUI 模式锁输入，JSON 模式留一个单槽队列**（cli.ts:140-146）：

```ts
if (isProcessing) {
    // Queue the message for when the agent is done
    pendingMessage = command.content;
} else {
    processMessage(command.content);
}
```

这就是最小版的全部"并发控制"——一个布尔锁 + 一个单槽队列。现代版的 steering / followUp 双队列，就是从这个单槽长出来的。

还有一个诚实的代价值得说：中断是一刀切。如果中断发生在"工具跑完、结果还没回填"的瞬间，messages 里可能留着没有配对的 function_call——恢复会话后，模型会收到一本"缺页的剧本"。最小版不处理这种半截状态，现代版有更细的事件流和 prepareNextTurn 钩子来收拾。

刹车只有一套，护栏几乎没有。**下一章数一数：跟现代版比，最小版到底缺了什么。**

---

## 四、那些还没有的护栏

### 4.1 缺席清单

把最小版和现代版（18 万行版）放在一起对照，差距一目了然：

| 现代版机制 | 最小版状态 |
|---|---|
| 流式输出（stream-fn.ts 统一各家） | 无流式：一次性 create，TUI 放 spinner |
| 并行工具执行（thunk + Promise.all 保序） | `for` 循环串行执行，parallel 参数只对模型有效 |
| 长度截断保护（length 整批拒绝） | `max_output_tokens: 2000` 硬编码，TODO 挂着 |
| 8 个钩子（convertToLlm / transformContext / beforeToolCall / afterToolCall / shouldStopAfterTurn / prepareNextTurn / steering / followUp…） | 0 个钩子，扩展靠改源码 |
| 双层消息抽象 + convertToLlm 海关 | `messages: any[]`，无类型裸奔 |
| 记忆压缩 compaction | 无压缩，上下文只增不减直到 token 爆炸 |
| steering / followUp 双队列 | 仅 JSON 模式单槽 pendingMessage |
| 不抛错契约（stopReason: "error"） | throw + ask() 里 catch 吞掉 |

### 4.2 野生护栏：防现实不防模型

但最小版**不是完全没有护栏**——它有几条"野生护栏"，散落在 tools.ts 和 tui-renderer.ts 里：

| 护栏 | 位置 | 防的现实问题 |
|---|---|---|
| bash 输出 1MB 截断 | tools.ts execWithAbort | 命令输出撑爆内存和上下文 |
| read 超过 1MB 只读前 1MB | tools.ts:184-190 | 大文件整个读进上下文直接爆 |
| rg exit code 1 视为"无匹配" | tools.ts:146-148 | rg 的正常语义：没找到也返回 1 |
| rg 输入重定向 `< /dev/null` | tools.ts:248 | rg 在 shell 里卡住等 stdin |
| 工具结果屏幕最多显示 10 行 | tui-renderer.ts:208-222 | 屏幕被工具输出刷屏 |

摘两条最典型的（tools.ts:146-148 和 243-248）：

```ts
if (code === 1 && command.includes("rg")) {
    resolve(""); // No matches for ripgrep
}
```

```ts
// Force ripgrep to never read from stdin by redirecting stdin from /dev/null
const cmd = `rg ${args} < /dev/null`;
```

注意这些护栏的对象——**全是"现实世界"**：文件太大、命令卡 stdin、屏幕太小。**没有一条是防"模型"的**。模型要发疯（无限调工具、输出超长、跑飞命令），最小版只有两个兜底：2000 token 上限和 Esc 键。现代版新增的护栏几乎全是防模型的：length 整批拒绝、shouldStopAfterTurn 优雅收工、压缩、steering 队列……这就是"野生"和"工业"的分界线：

> **最小版的护栏防现实不防模型；现代版的护栏主要防模型。**

### 4.3 为什么裸机版值得读

拆完这一圈，反过来问：既然最小版又没护栏又有 bug，为什么值得读？

1. **发动机裸露**：两个 while + 一个 class + 事件系统，核心三件套（agent.ts + tools.ts + session-manager.ts）一共 924 行，能整个装进脑子里；
2. **它划定了一个下限**：一个能跑的工具型 Agent，最少只需要这么多——大脑（SDK 直连）、手脚（5 个工具）、记事本（数组 + JSONL）、任务单（一个布尔出口）、工作习惯（while + switch）；
3. **对照现代版能分清"本质"与"补丁"**：循环骨架、事件广播、中断隔离——这些是最小版就有的，是本质；钩子、压缩、流式、截断保护——这些是后来长出来的，是补丁。学 agent loop，先认本质，再看补丁。

---

## 五、总结：两千行发动机，十八万行护栏

把全文收束成一张对照大图：

```
四要素     最小版源码位置           干了什么
──────────────────────────────────────────────────────────────
大脑   →  agent.ts:43-272          两个"打电话"函数，各包一个 while
手脚   →  tools.ts（264 行）       5 个工具 × 2 套 schema，executeTool 分发
记事本 →  agent.ts:277 messages   内存剧本 + JSONL 流水账 + setEvents 还原
任务单 →  systemPrompt + 循环出口  最小版的目标 = "把这句话答完"
──────────────────────────────────────────────────────────────
工作习惯 → while + switch + AbortController + 事件广播
```

**三个可以抄走的设计**：

1. **事件广播器 comboReceiver**——一个动作多个消费者，UI 与持久化解耦，新消费者插线即可。现代版的 EventStream 就是它长大后的样子；
2. **事件溯源 setEvents**——不存"状态快照"，存"流水账"，剧本随时可重建。最小版已经验证了这条路走得通，虽然翻译有损；
3. **per-ask AbortController**——中断隔离在一轮里，用完即弃，不污染全局。现代版的中断模型更细，但"一次一问一刹车片"的骨架没变。

**三个要修的坑**（后都修了）：

1. 死代码解构——注释说要删 type 字段，代码 push 的却是原对象；
2. `call_id` / `id` 用错字段——工具失败时的回填对不上号；
3. chat 循环缺 else 分支——模型返回"既无工具调用又无内容"的消息时原地死循环。

最后收个尾：**最初的两千行已经是一台能跑的发动机**——大脑、手脚、记事本、任务单一个不少，工作习惯就是两个 while 循环。后来长出来的 18 万行，绝大部分不是发动机本身，而是**刹车、安全气囊、仪表盘和导航**。但护栏再豪华，也得先有发动机——学 agent loop，先拆裸机，再装护栏。

---

## 相关阅读

- [《Agent 循环与目标：一个 Agent 到底是怎么自己跑起来的》](/2026-08-29/Agent循环与目标-一个Agent到底是怎么自己跑起来的)——本篇的原理篇：循环、压缩、目标判定
- [《AgentScope 详解：多智能体框架从 ReAct 循环到企业级 Harness》](/2026-09-04/AgentScope详解-多智能体框架从ReAct循环到企业级Harness)——另一个框架的做法：配置+插件驱动 vs pi 的代码即编排
- [《企业级记忆知识库：短期上下文与四层记忆的 RAG 向量检索实现》](/2026-09-01/企业级记忆知识库-短期上下文与四层记忆的RAG向量检索实现)——记忆的分层视角，与本篇"记事本"一章互补
- [《Transformer 解码器：大模型是怎么"一个字一个字"写出来的》](/2026-09-05/Transformer解码器-大模型一字一句生成的工作原理详解)——大脑内部：create 之后模型在干什么
