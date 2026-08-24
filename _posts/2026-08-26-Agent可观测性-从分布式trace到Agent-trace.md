---
layout: post
title: "Agent 可观测性：从分布式 trace 到 Agent trace"
author: "Inela"
---

微服务时代，线上出了问题，工程师的第一反应是打开链路追踪平台，搜一个 `traceId`，看这条请求在网关、订单、库存、支付之间哪个服务慢了、哪个服务报错了。这套打法在 2026 年之前几乎是可观测性的"标准答案"。

但 Agent 时代，问题变了。

你问的已经不是"哪个服务慢了"，而是：

- 模型为什么突然决定调用 `bash` 而不是先读文件？
- 这一轮（turn）为什么多花了 30 秒？是模型在想，还是某个工具卡住了？
- 它到底有没有经过审批，就去执行了那个危险操作？
- 一个子 Agent 在父 Agent 的哪个时间点被 fork 出来，它俩的上下文是怎么拼接的？

这四个问题，传统的分布式 trace 一个都答不利索。因为它们的答案不在 span 树里，而在**事件日志**里。

这篇文章，我把 DeepSeek Harness（下文简称 DSH）的会话日志/事件流——`agent/*`、`tools/*`、`turn/*`、`step/*`——当成"Agent 版的 OpenTelemetry trace"来拆：**span 怎么建、怎么回放、怎么审计、怎么脱敏**。

如果你读过我之前写的 [SLF4J 原理深度剖析](/2026-07-02/SLF4J原理深度剖析) 和 [日志脱敏：从注解到自动化的完整实现](/2026-07-02/日志脱敏-从注解到自动化的完整实现)，这篇是那两篇的自然延伸：日志是系统的眼睛，SLF4J 是视神经，脱敏是眼睛的遮光片——而这篇要讲的是，当"系统"变成了会思考的 Agent，这双眼睛该怎么重新聚焦。

---

## 一、为什么微服务的 trace 不够用了

### 1.1 OTel 三支柱的隐含假设

先回顾一下我们熟悉的那套。OpenTelemetry 定义了三大信号：**Trace**（链路追踪）、**Metrics**（指标）、**Logs**（日志）。其中 Trace 是排查问题的第一入口：

```
Trace = 一棵 span 树
       ├── span: 一次操作的时间区间 [start, end]
       ├── traceId:  整条链路的唯一标识
       ├── spanId:   单个 span 的唯一标识
       ├── parentSpanId: 父 span 的标识（构成树）
       └── attributes / events: 附属信息
```

这套模型之所以好用，是因为它背后有三个**隐含假设**：

| 假设 | 含义 |
|------|------|
| **路径确定性** | 一次请求的执行路径，在进入系统时基本就确定了（代码是写死的） |
| **一次一链** | 一次请求对应一条 trace，链上 span 数量有限（几十个级别） |
| **边界清晰** | 每个 span 是一个 RPC 调用或函数，边界明确、嵌套规整 |

### 1.2 Agent 打破了这三个假设

而一个 Agent 的一次任务，把这三个假设全打破了：

```
假设一「路径确定性」被打碎：
  微服务：请求 → 网关 → 订单服务 → 库存服务 → 返回（路径写死在代码里）
  Agent：  用户提需求 → 模型「决定」读文件 → 模型「决定」调 grep
           → 模型「决定」调 bash → 模型「决定」再读另一个文件 → ...
           ↑ 每一步都是模型「现场决定」的，不是代码写死的

假设二「一次一链」被打碎：
  微服务：一次请求 = 一条 trace，几十个 span
  Agent：  一次任务可能持续几分钟，经历几十轮 turn、
           上百次工具调用、几万 token 的流式输出

假设三「边界清晰」被打碎：
  微服务：span 边界 = RPC 边界，天然清晰
  Agent：  「模型思考」这一段是黑盒，没有天然的 span 边界
           工具执行、token 生成、审批等待……边界需要重新定义
```

### 1.3 结论：Agent 需要的是"日志即 trace"

分布式 trace 的根子是**采样**：为了省存储，它只留一部分。而 Agent 出问题，往往就出在那个"被采样丢掉的"决策瞬间——你没法事后问一句"它当时到底怎么想的"。

所以 Agent 可观测性的答案，不是"更细的 span"，而是一个方向性的转变：

```
从「采样式的、丢历史的 trace」
到「事件溯源式的、可回放、可审计的日志即 trace」
```

DSH 正是这么做的：它把 Agent 的每一次交互，都落成一条**append-only、seq 连续、lossless JSON** 的事件日志。这条日志本身，就是一条可回放的 Agent trace。

---

## 二、核心概念映射：把 DSH 会话日志当 trace 读

### 2.1 一张对照表：全篇的"翻译词典"

先把 OTel 的词汇和 DSH 的词汇对齐。这张表是全篇的锚点：

| OTel 概念 | DSH 对应 | 说明 |
|---|---|---|
| **Trace** | 一个 **Session**（`session.id` + `SessionHeader`） | 一次 Agent 交互的完整生命 |
| **Span（外层）** | `turn/start` … `turn/end` | 一次"用户输入 → Agent 完成响应"的回合 |
| **Span（内层）** | `step/start` … `step/end` | 一次模型调用 + 它触发的工具执行 |
| **Span Event / 内容** | `user/message`、`assistant/message`、`tool/call`、`tool/result` | span 里真正"发生"的事 |
| **流式原始数据** | `assistant/chunk` | token 级原始流，回放保真 |
| **Resource Attribute** | `SessionHeader` + `request/header` + `request/context` | 会话元信息、模型配置、路由 |
| **运行时信号（Metrics）** | `agent/*`、`tools/*` 总线事件 | 运行态状态、工具执行钩子 |
| **Exporter** | `session-telemetry-otel`（OTLP） | 把事件导出成 OTel logs |
| **Collector / 存储** | `session-persistence`（JSONL / SQLite） | 落盘，保证可回放 |

### 2.2 事件命名空间总览

DSH 的事件流分两大类，一类是**运行态总线事件**（不落盘，转瞬即逝），一类是**会话事件**（落盘，是 trace 的主体）：

<pre class="highlight"><code>
运行态总线事件（live bus，不落盘）:
  agent/status、agent/created、agent/error、agent/session-start、
  agent/pre-step、agent/turn-stopping
  tools/execute、tools/pre-execute、tools/post-execute、tools/change
        ↑ 这些是「正在发生什么」的瞬时信号，类比 metrics

会话事件（session log，落盘，trace 主体）:
  <span class="evt-turn">turn/start</span>   <span class="evt-turn">turn/end</span>      ← 回合的括号
  <span class="evt-step">step/start</span>   <span class="evt-step">step/end</span>      ← 步骤的括号
  <span class="evt-msg">user/message</span>               ← 用户输入 / 注入的上下文
  <span class="evt-chunk">assistant/chunk</span>            ← 原始流式 token
  <span class="evt-msg">assistant/message</span>          ← 组装好的模型回复（带 usage）
  <span class="evt-tool">tool/call</span>                  ← 模型请求调用工具
  <span class="evt-tool">tool/result</span>                ← 工具执行结果
  <span class="evt-marker">request/header</span>  <span class="evt-marker">request/context</span>  ← 模型配置与路由
  <span class="evt-marker">todo/write</span>                 ← 任务清单
  <span class="evt-marker">session/end-seed</span>           ← 种子边界（fork 相关）
        ↑ 这些是「发生过什么」的持久记录，类比 trace
</code></pre>

### 2.3 一张全景图：一个回合的完整事件流

<pre class="highlight"><code>
用户输入 "帮我看看这个目录下有哪些 TODO"
        │
        ▼
  ┌─────────────┐
  │ <span class="evt-turn">turn/start</span>  │  turn=1          ← 回合开始（外层 span open）
  └──────┬──────┘
         ▼
  ┌──────────────┐
  │ <span class="evt-msg">user/message</span> │  "帮我看看这个目录下有哪些 TODO"
  └──────┬───────┘
         ▼
  ┌─────────────┐
  │ <span class="evt-step">step/start</span>  │  turn=1, step=1  ← 第一次模型调用（内层 span open）
  └──────┬──────┘
         ▼
  ┌───────────────────┐
  │ <span class="evt-msg">assistant/message</span> │  模型决定调用 grep 工具
  │  content: tool-call│  name=grep, callId=call-1
  └──────┬────────────┘
         ▼
  ┌────────────┐
  │ <span class="evt-tool">tool/call</span>  │  grep, callId=call-1, arguments={...}
  └──────┬─────┘
         ▼
  ┌─────────────┐
  │ <span class="evt-tool">tool/result</span> │  匹配结果（callId 配对）
  └──────┬──────┘
         ▼
  ┌───────────────────┐
  │ <span class="evt-msg">assistant/message</span> │  模型基于工具结果给出最终回复（含 token usage）
  └──────┬────────────┘
         ▼
  ┌───────────┐
  │ <span class="evt-step">step/end</span>  │  turn=1, step=1  ← 内层 span close
  └──────┬────┘
         ▼
  ┌───────────┐
  │ <span class="evt-turn">turn/end</span>  │  reason=completed  ← 回合结束（外层 span close + status）
  └───────────┘
</code></pre>

### 2.4 三个贯穿全篇的设计底座

理解 DSH 的 trace，先记住三个底座，后面每一章都在用：

```
底座一：append-only
  → 事件只能追加，永不修改、永不删除
  → 想看"当时发生了什么"，历史永远在那里

底座二：seq 单调连续
  → 每个事件一个单调递增的序号，且必须连续（0,1,2,3,...）
  → 缺一个序号，后端直接拒收（下一章细讲为什么）

底座三：lossless JSON
  → 每个事件 data 都是可无损序列化的 JSON
  → 落盘字节 == 内存字节，回放时逐字节还原
```

---

## 三、Span 怎么建：turn 和 step 的"括号语义"

### 3.1 OTel span 的本质

一个 OTel span 本质上是两件事：

```
1. 一对时间戳：start（开始）和 end（结束），end - start = duration
2. 一个父子引用：parentSpanId 指向父 span，构成树
```

### 3.2 DSH 用"括号事件"建 span

DSH 没有显式的 `spanId`/`parentSpanId`，而是用**成对的边界事件**（我称之为"括号语义"）：

<pre class="highlight"><code>
<span class="evt-turn">turn/start</span> ... <span class="evt-turn">turn/end</span>      → 外层 span（一个回合）
   └── <span class="evt-step">step/start</span> ... <span class="evt-step">step/end</span>  → 内层 span（一次模型调用）
</code></pre>

源码里，这四类边界的 data 形状极简（见 `SessionEventMap`）：

```ts
'turn/start': { turn: number }                    // 打开回合 turn
'turn/end':   { turn: number; reason: TurnEndReason }  // 关闭回合 + 原因
'step/start': { turn: number; step: number }      // 打开步骤 step
'step/end':   { turn: number; step: number }      // 关闭步骤 step
```

### 3.3 事件信封：一条事件的完整结构

每个事件都是一个统一信封，`type` 是一个**判别联合**（discriminated union）的键，`switch(event.type)` 就能自动收窄 `data` 的类型：

```ts
type SessionEvent = {
  type: string          // 事件类型，如 'turn/start'
  seq: number           // 会话内单调递增的序号
  time: number          // Unix epoch 毫秒时间戳
  data: { ... }         // 该类型专属的负载
  ignorable?: true      // 见第六章「前向兼容」
  // 仅消息类事件（user/message、assistant/message、tool/result）有：
  surfaceOp?: 'append' | { op: 'replace'; start: number; end: number }
  sourceEventSeqs?: number[]  // 引用的上游事件 seq
}
```

### 3.4 消息如何挂到 span 上

这是 Agent trace 和微服务 trace 最大的区别：**span 的"内容"不是 attributes 里的几串 key-value，而是挂在 span 之间的一条条消息事件**。

两个关键的"挂钩"机制：

**机制一：`tool/call` 和 `tool/result` 靠 `callId` 配对**

```ts
// 模型请求调用工具
'tool/call': { turn, step, callId, name, arguments }

// 工具返回结果，用同一个 callId 配对
'tool/result': { turn, step, message, error?, meta? }
```

`callId` 是工具调用在 trace 里的"子链 id"——它把"模型决定调工具"（`assistant/message` 里的 tool-call block）、"实际调用"（`tool/call`）、"结果返回"（`tool/result`）三件事串成一条因果链。

**机制二：`assistant/message.sourceEventSeqs` 引用原始流**

```ts
'assistant/chunk':    { turn, step, chunk }   // 原始流，可能有几万个
'assistant/message':  { turn, step, message, usage? }  // 组装结果

// assistant/message 的 sourceEventSeqs 指向构成它的 chunk 们
// → 你知道这条完整回复，是由哪些原始 token 拼出来的
```

### 3.5 边界之外：`turn/end.reason` 就是 span 的 status

OTel span 有 `status`（OK / Error），DSH 的回合结束事件用 `reason` 表达得更丰富：

```ts
type TurnEndReason =
  | { kind: 'completed' }        // 正常完成
  | { kind: 'aborted'; reason }  // 被取消（用户/父Agent/钩子）
  | { kind: 'blocked' }          // 阻塞
  | { kind: 'error'; error }     // 出错（带结构化 LlmFailure）
  | { kind: 'max-tokens' }       // 触达输出 token 上限
  | { kind: 'interrupted' }      // 崩溃遗留（见第四章自愈）
```

**关键洞察**：`interrupted` 不是运行时产生的，而是持久化层在**崩溃恢复**时合成的。这一条单独拎出来，就是第四章的案例三。

### 3.6 与 OTel 的关键差异：不存 parentSpanId

这里藏着一个反直觉的设计决策：

<pre class="highlight"><code>
OTel：显式存 parentSpanId，靠指针建树
DSH： 不存 parentSpanId，靠「seq 顺序 + turn/step 编号」推导层级

为什么 DSH 敢这么做？
  → 因为 seq 是全局单调连续的，事件顺序就是唯一的时间轴
  → turn 编号 + step 编号天然表达了嵌套关系
  → 「括号」的开闭（<span class="evt-turn">turn/start</span> 到 <span class="evt-turn">turn/end</span>）本身就框出了父子边界

好处：不用维护一个脆弱的指针，回放时靠顺序就能重建整棵树
</code></pre>

这是 Agent trace 的精髓之一：**把"关系"从显式指针降维成"顺序 + 编号"，换取可回放性和容错性**。

---

## 四、案例分析：读四条真实的 DSH 轨迹

> 这一章不虚构，逐 seq 拆 DSH 持久化契约测试里真实存在的事件序列。目的：把第三章的"括号、seq、配对"落到"打开一条 session 日志，你到底该怎么读"。

### 4.1 案例一：最小回合 —— 一条"hello world" trace

来源：DSH `session-persistence` 契约测试的 `oneTurnLog()`。无工具调用的最简完整回合：

<pre class="highlight"><code>
seq  type                data（摘要）
 0   <span class="evt-turn">turn/start</span>          { turn: 1 }
 1   <span class="evt-msg">user/message</span>        "hi"
 2   <span class="evt-step">step/start</span>          { turn: 1, step: 1 }
 3   <span class="evt-msg">assistant/message</span>   "hello"
 4   <span class="evt-step">step/end</span>            { turn: 1, step: 1 }
 5   <span class="evt-turn">turn/end</span>            { reason: completed }
</code></pre>

**读法拆解**：

1. **seq 是唯一时间轴**。0 到 5 单调连续，中间缺任何一个（比如 seq 2 丢了），后端会直接拒收——这保证了你看到的 trace 永远是"完整的一段"，不存在静默跳号。
2. **两对括号 = 两层 span**。`turn/start…turn/end` 是外层（一次完整回合），`step/start…step/end` 是内层（一次模型调用）。没有 `parentSpanId`，层级靠顺序 + 编号推导。
3. **`turn/end.reason = completed` 是这条 trace 的 status**。相当于 OTel 里 span 的 `status: OK`。

### 4.2 案例二：带工具调用的回合 —— Agent 真正的核心循环

来源：DSH 协调器契约测试的 `legacyMessageLog()`（此处用当前规范格式重新表述，序列结构一致）。展示"模型请求工具 → 执行 → 结果回填 → 结果被替换"：

<pre class="highlight"><code>
seq  type                data（摘要）
 0   <span class="evt-turn">turn/start</span>          { turn: 1 }
 1   <span class="evt-msg">user/message</span>        "hi"
 2   <span class="evt-step">step/start</span>          { turn: 1, step: 1 }
 3   <span class="evt-msg">assistant/message</span>   tool-call: read, callId=call-1     ← 模型决定调工具
 4   <span class="evt-tool">tool/call</span>           read, callId=call-1, arguments={}  ← 调用点
 5   <span class="evt-tool">tool/result</span>         "full result"   (surfaceOp: append)      ← 结果上屏
 6   <span class="evt-tool">tool/result</span>         "pruned"        (surfaceOp: replace 5..5) ← 结果被压缩替换
 7   <span class="evt-step">step/end</span>            { turn: 1, step: 1 }
 8   <span class="evt-turn">turn/end</span>            { reason: completed }
</code></pre>

**读法拆解**（三个关键洞察）：

**洞察一：工具执行是 trace 里的"子链"。**

<pre class="highlight"><code>
<span class="evt-msg">assistant/message</span>（决策：我要调 read）
      │  callId=call-1
      ▼
<span class="evt-tool">tool/call</span>（实际执行：read 被调用）
      │  callId=call-1
      ▼
<span class="evt-tool">tool/result</span>（结果：返回了什么）
</code></pre>

三者靠 `callId` 配对，构成 Agent trace 最典型的一段。排查"Agent 为什么做了 X"，第一步就是顺着 `callId` 找这条子链。

**洞察二：append-only 上的"逻辑替换"。**

注意 seq 5 和 seq 6 都是 `tool/result`，但 seq 6 的 `surfaceOp` 是 `{ op: 'replace', start: 5, end: 5 }`：

<pre class="highlight"><code>
seq 5: <span class="evt-tool">tool/result</span> "full result"   surfaceOp: append
seq 6: <span class="evt-tool">tool/result</span> "pruned"        surfaceOp: replace 5..5
        ↑ 意思是「我用 seq 6 覆盖 seq 5，但 seq 5 没被删除」
</code></pre>

历史仍在（可审计），但**当前视图只认最新一条**。这就是 compaction / prune 的落盘方式：不是删旧数据，而是用 replace 声明"这条被替换了"。这对审计极其重要——你永远能回到"它当时先看到了 full result，后来才被压缩成 pruned"。

**洞察三：为什么不能只靠"看对话"来查问题。**

如果你只看 UI 上的对话流，只能看到"模型调了 read，得到了结果"。但 `tool/call`（调用点）、`tool/result` 的 replace 动作、`callId` 的配对关系，全都被折叠掉了。事件日志才保留了完整的因果链。

### 4.3 案例三：崩溃轨迹的自愈 —— Agent trace 比微服务 trace 强在哪

来源：DSH 持久化契约测试的三个崩溃恢复用例。这是本章的高潮，回答"trace 能不能在事故后救活现场"。

#### 场景 A：turn 撕裂（进程在回合中途被杀）

<pre class="highlight"><code>
0..5  turn 1 完整（同案例一）
 6    <span class="evt-turn">turn/start</span>   { turn: 2 }
 7    <span class="evt-step">step/start</span>   { turn: 2, step: 1 }
      —— 进程崩溃，<span class="evt-step">step/end</span>、<span class="evt-turn">turn/end</span> 永远没写 ——
</code></pre>

重新 load 时，持久化层自动"补括号"：

<pre class="highlight"><code>
 8    <span class="evt-step">step/end</span>     { turn: 2, step: 1 }      ← 合成（synthetic）
 9    <span class="evt-turn">turn/end</span>     { reason: interrupted }   ← 合成（synthetic）
</code></pre>

**读法**：一个撕裂的 span 不是垃圾数据，而是被 `commitRepair` 用合成事件**闭合**。审计者一看 `reason: interrupted` 就知道"这不是正常结束，是崩溃遗留"。

对比一下微服务：一条没有 span/end 的链，要么被采样器丢掉，要么在 UI 里永久悬挂成一个"幽灵 span"。Agent trace 选择主动闭合它，并**留下一个诚实的标记**。

#### 场景 B：模型要了工具，但工具还没来得及执行就崩了

<pre class="highlight"><code>
 8    <span class="evt-msg">assistant/message</span>   tool-call: bash, callId=call-x
      —— 崩溃（工具没跑，<span class="evt-tool">tool/result</span> 不存在）——
</code></pre>

补上一条**合成的错误结果**，保证 transcript 里没有"悬空的工具调用"：

<pre class="highlight"><code>
 9    <span class="evt-tool">tool/result</span>   { isError: true, error.code: TOOL_NOT_STARTED }
10    <span class="evt-step">step/end</span>
11    <span class="evt-turn">turn/end</span>      { reason: interrupted }
</code></pre>

**读法**：`TOOL_NOT_STARTED` 是 trace 自己"承认"——这个工具请求从未真正执行。这是可观测性里最稀缺的东西：**明确区分"没做"和"做了但结果丢了"**。

#### 场景 C：工具调用了，但结果没落盘（副作用风险最高）

<pre class="highlight"><code>
... <span class="evt-msg">assistant/message</span>   tool-call: write, callId=call-risk
... <span class="evt-tool">tool/call</span>           write, callId=call-risk
      —— 崩溃，<span class="evt-tool">tool/result</span> 永远没写 ——
</code></pre>

这次不是"没做"，而是"做了，但不知道结果"。补一条**带安全提示的合成结果**：

<pre class="highlight"><code>
<span class="evt-tool">tool/result</span>  { error.code: TOOL_OUTCOME_UNKNOWN,
               text: "retry only if the operation is read-only or idempotent;
                      if it may have side effects, first verify external state
                      or ask the user" }
</code></pre>

**读法**：这是整章最亮的一笔。trace 不止记录"发生了什么"，还**在崩溃自愈时给模型注入"重试安全边界"**。`write` 可能已经产生副作用（比如已经写了一半文件），系统不假装知道结果，而是把"未知结果"这个事实如实交给下一轮模型去判断。

可回放 + 可自愈，这是 Agent trace 对微服务 trace 的降维打击——它不只是"事后看"，还能"事故后救"。

### 4.4 案例四：跨会话 fork / subagent 的 trace 拼接

> 这是 Agent trace 独有的场景：微服务 trace 里没有"从一个 trace 里 fork 出一个新 trace，且新 trace 继承了旧 trace 的前缀"这回事。而 Agent 里，子 Agent 就是这样的存在。

**背景**：DSH 里，父 Agent 可以 fork 出子 Agent（subagent）。子 Agent 的会话是一个**独立的 Session**，但它不是从零开始的——它继承父会话的一部分历史作为"种子"（seed）。

源码里，子会话的元信息是这样构建的（`child-agent.ts` 的 `childSessionMeta`）：

```ts
// 子会话的 SessionHeader 携带四个关键字段
{
  parentSession: parentHeader.id,        // 父会话的 id
  origin: 'subagent',                    // 产品级分类标记
  delegationDepth: childDepth,           // 委派深度 = 父深度 + 1
  seedLength: lineageSeedLength,         // 前多少条事件来自父日志
}
```

**案例：父 Agent 有 6 条事件，fork 出一个子 Agent**

<pre class="highlight"><code>
父会话 (session.id = root)：
  seq 0..5   完整的一轮 turn（同案例一，共 6 条事件）

子会话 (session.id = child，parentSession = root，seedLength = 6)：
  seq 0..5   ← 继承自父日志的种子前缀（与父的 0..5 逐字节相同）
  seq 6      <span class="evt-marker">session/end-seed</span>           ← 种子边界标记
  seq 7      <span class="evt-turn">turn/start</span>  { turn: 2 }     ← 子 Agent 自己的第一轮开始
  seq 8      ...                         ← 之后全是子 Agent 自己的工作
</code></pre>

**读法拆解**（三个关键点）：

**关键点一：`session/end-seed` 是拼接的"分界线"。**

`session/end-seed` 是一个空负载的边界事件，由 `Session` 构造器写入，含义是：**"我之前的 seq 都是种子（继承来的），从这个边界之后才是本生命周期自己产生的"**。源码里对应的运行时概念是 `firstLiveSeq`：

```
seq < seedLength（即 firstLiveSeq）  → 种子历史（继承自父）
seq >= seedLength                    → 子自己的实时工作
```

**关键点二：`seedLength` + `parentSession` 让接收端能把父子 trace 拼起来。**

子会话的日志里，前缀是父历史的**逐字节拷贝**。接收端（比如 telemetry 导出）在导出子会话事件时，会带上这两个身份属性（`session-telemetry` 的 `identityOf`）：

```ts
// 子会话事件的 identity attributes 里会带上：
{
  'session.id': 'child',
  'session.parent_id': 'root',     // 父是谁
  'session.seed_length': 6,        // 前缀多长
  ...
}
```

接收端拿到这两个字段就知道：**"child 的 0..5 在 root 那里已经见过了，我不需要重复存，只需在 child 上挂一个指向 root 的指针，把 6 条前缀和 child 自己的尾部拼起来"**。这就是跨会话 trace 的拼接——类比微服务里"一个 trace 跨越多个服务，靠 traceId 串联"，这里靠的是 `parentSession` + `seedLength`。

**关键点三：`delegationDepth` 是"递归预算"，必须能扛过重启。**

`delegationDepth = 父深度 + 1`，它被持久化到 header 里，而不是只存在运行时。为什么？因为一个子 Agent 被 resume（进程重启后恢复）时，如果深度只在运行时，重启后它就"忘记"自己是第几层了，可能突破递归上限。所以深度必须落盘，成为 trace 的"层级"信息的一部分。

```
root (depth 0)
  └── child (depth 1, parentSession = root)
        └── grandchild (depth 2, parentSession = child)
```

于是，整个 Agent 集群的 trace 不是一棵扁平的 span 树，而是一张**跨会话的有向无环图（DAG）**：节点是 Session，边是 `parentSession` 指针，边的权重是 `seedLength`。

### 4.5 三个案例串成一句方法论

> 读一条 DSH 轨迹 = 沿 seq 走时间轴，用 `turn/step` 括号切 span，用 `callId` 串工具子链，用 `surfaceOp` 看视图演变，用 `turn/end.reason` 读结局，用 `parentSession` + `seedLength` 拼跨会话的父子关系——崩溃时再看持久化层怎么"补括号自愈"。

---

## 五、Span 怎么回放：事件溯源是 Agent trace 的杀手锏

第四章的案例三已经埋了伏笔：DSH 的 trace 能在崩溃后自愈。这一章系统讲"回放"这件事。

### 5.1 为什么分布式 trace 几乎不能回放

```
微服务 trace：
  采样丢了 → 只剩 1% 的链，回放无从谈起
  span 里只有几个 attributes → 没有"状态"，无法重建历史
  → 只能「看」，不能「重演」

DSH Agent trace：
  事件全量落盘 → 一条都不丢
  事件是事件溯源（event sourcing）的日志 → 重放日志 = 重建状态
  → 既能「看」，也能「重演」
```

### 5.2 回放 = 重放事件日志重建状态

事件溯源（event sourcing）的核心思想：**不存当前状态，存导致状态变化的事件序列；重放事件，就能重建任意时刻的状态**。

DSH 的 `Session` 就是这么做的：

```ts
// 从一个持久化的会话恢复，传入完整事件日志 + header
const resumed = Session.create(sessionId, loadedEvents, loadedMeta)

// 从事件日志重建「模型可见的历史消息」
const messages = resumed.deriveMessages()
```

`deriveMessages()` 遍历 `user/message`、`assistant/message`、`tool/result`，按 `surfaceOp` 应用 append/replace，重建出模型看到的那份对话历史。**消息历史不是存出来的，是算出来的**——这就是回放的威力。

### 5.3 落盘格式：JSONL + zstd

DSH 的 JSONL 持久化后端，每个会话一个文件，格式极简：

```
第 1 行：header 行（type: 'session'）
第 2 行起：一条事件一行
```

```json
{"type":"session","version":0,"id":"abc123","createdAt":1724650000000,"cwd":"/work","delegationDepth":0}
{"type":"turn/start","seq":0,"time":1,"data":{"turn":1}}
{"type":"user/message","seq":1,"time":2,"data":{...},"surfaceOp":"append"}
...
```

要点：

- **header 行与事件行区分**：靠 `type: 'session'` 标记，读第一行就能拿到元信息。
- **zstd 压缩**：可选，`.jsonl.zstd`，省空间但不损回放性。
- **路径安全**：`SessionId` 是未校验的 branded string，落盘前必须编码（防目录穿越、防碰撞）。

### 5.4 恢复与断点：`firstLiveSeq` 与 `session/end-seed`

第四章案例四已经见过 `session/end-seed` 了。它的作用是标记"种子边界"：

```
一个 Session 的事件日志：
  seq 0..(firstLiveSeq-1)   → 种子（resume 的历史、fork 的父前缀）
  seq firstLiveSeq..        → 本生命周期实时产生
```

这个边界为什么重要？因为**resume 和 fork 都会把已有历史作为种子**。回放时需要知道"哪些是我自己产生的，哪些是继承来的"，否则：

- 一个 resumed 的会话会把自己的历史当成新工作重复处理；
- 一个 fork 的子会话会把父前缀当成自己的输出。

`session/end-seed` 是唯一合法的边界写入者（只有 `Session` 构造器），这个约束保证了边界不会被插件伪造。

### 5.5 崩溃恢复：撕裂的括号被自动闭合

第四章案例三的三种自愈，源头在持久化层的 `commitRepair`。它的逻辑一句话：**读到一个"开了没关"的括号（turn/step 只有 start 没有 end），就合成对应的 end 事件把它关上**：

<pre class="highlight"><code>
读取到撕裂的日志：
  <span class="evt-turn">turn/start</span> (seq 6)
  <span class="evt-step">step/start</span> (seq 7)
  ← 没有 <span class="evt-step">step/end</span>、<span class="evt-turn">turn/end</span>

commitRepair 合成：
  <span class="evt-step">step/end</span>   { turn: 2, step: 1 }        ← 补内层括号
  <span class="evt-turn">turn/end</span>   { reason: interrupted }     ← 补外层括号 + 标记崩溃

结果：日志重新「平衡」，可以继续 append（下一条 seq = 10）
</code></pre>

**关键设计**：合成事件不是"篡改历史"，而是"闭合历史"——它保留了原始事件，只追加了必要的闭合标记，且用 `reason: interrupted` 诚实标注。

### 5.6 resume / fork / compaction：三种"非从头"的回放

| 场景 | 机制 | trace 上的表现 |
|------|------|--------------|
| **resume**（重启恢复） | 加载已有日志，从 `firstLiveSeq` 继续 | 同一条 trace 跨进程延续 |
| **fork**（派生子会话） | `seedLength` 记录继承前缀长度 | 新 trace 继承了父前缀（案例四） |
| **compaction**（压缩） | `surfaceOp: replace` 替换消息 | 视图变短，但历史事件保留 |

三者的共同点：**都不破坏 append-only 和 seq 连续性**。

### 5.7 精确到 token 的回放：`assistant/chunk`

最后，DSH 的 trace 回放能精确到什么程度？答案是 token 级：

<pre class="highlight"><code>
<span class="evt-chunk">assistant/chunk</span>: { turn, step, chunk }   ← 原始流式 chunk，一个不落
<span class="evt-msg">assistant/message</span>: { message, usage }    ← 组装结果 + token 计数
</code></pre>

`assistant/message` 的 `sourceEventSeqs` 指向构成它的所有 `assistant/chunk`。这意味着：

- 你能回放出"模型第 37 个 token 是什么时候吐出来的"；
- 你能统计"这一轮到底消耗了多少 token"（`usage` 就挂在 `assistant/message` 上，而不是单独一条 usage 记录——"输出和它的计费一起走"）。

这种保真度，是微服务 trace 想都不敢想的。

---

## 六、Span 怎么审计：append-only 日志的问责链

可观测性的一半是"看得见"，另一半是"信得过"。Agent 会自己调工具、自己读文件、自己执行命令——审计就成了刚需：**它做的那件事，到底是谁允许的？**

### 6.1 审计四问

一条合格的 Agent trace，要能回答四个问题：

<pre class="highlight"><code>
谁        在什么时候      做了什么          被允许吗       结果如何
  │            │              │                │              │
session.id   event.time   <span class="evt-tool">tool/call</span>.name   approval/decided  <span class="evt-turn">turn/end</span>.reason
  │            │              │                │              │
agent.id    event.seq    <span class="evt-tool">tool/call</span>.args    permission/preset <span class="evt-tool">tool/result</span>.error
</code></pre>

### 6.2 不可变性：三条防线

审计的前提是"日志不可被事后篡改"。DSH 用三条防线保证：

```
防线一：append-only
  → 只能追加，不能改、不能删
  → 想抹掉一条历史记录？做不到

防线二：seq 连续
  → 缺一个序号就拒收
  → 想偷偷抽掉中间一条？做不到，序号会断

防线三：SESSION_FORMAT_VERSION 版本门
  → 每个 header 盖一个格式版本号
  → 老 reader 遇到不认识的格式，拒绝读而不是读错
```

### 6.3 安全决策留痕：审批与权限

DSH 把安全相关的决策也落成事件（见 `KNOWN_SESSION_EVENT_TYPES`）：

```
approval/asked      ← 向用户发起了审批请求
approval/decided    ← 审批结果（同意 / 拒绝）
permission/preset   ← 权限预设
sandbox/mode        ← 沙箱模式（文件访问策略）
```

这意味着"模型执行 `bash` 之前，有没有经过审批"，是一个**能被事件日志回答的问题**，而不是事后扯皮。审计者可以直接下钻：

<pre class="highlight"><code>
<span class="evt-tool">tool/call</span>: bash, callId=call-x
   ↑ 这条调用之前，有没有 approval/asked + approval/decided = 同意？
</code></pre>

### 6.4 请求演进的留痕：`request/header.reason`

Agent 运行中，模型配置、system prompt、工具集都可能变。`request/header` 记录每次请求的完整配置快照，并用 `reason` 说明它为什么出现：

```ts
type RequestHeaderReason = 'initial' | 'resume' | 'change'
// initial: 新会话的第一份 header
// resume:  重启/fork 后，循环实例的第一份 header
// change:  后续请求换了一套 header
```

`request/header` 还附带 system prompt 和工具 schema。审计"它当时拿的是哪套工具、哪个 system prompt"时，不需要猜，直接读最近一份 `request/header`。

### 6.5 前向兼容：`ignorable` 标记

这是审计里容易被忽略、但极其重要的一环。事件词汇是会演进的——今天只有 20 种事件，明天插件可能新增 10 种。老 reader 读到不认识的事件怎么办？

DSH 的答案是 `ignorable` 标记：

```
事件带 ignorable: true
  → 纯信息记录，丢了不影响重建 → reader 可以安全跳过

事件不带 ignorable（默认）
  → 必须理解，否则会读错整条日志
  → reader 遇到不认识的非 ignorable 事件，必须拒绝，而不是静默跳过
```

**设计哲学**：默认"拒绝"，而不是默认"跳过"。因为静默跳过一条影响重建的事件，比报错更危险——它会让你在不知不觉中拿到一个"gutted session"（被掏空的会话）。

### 6.6 检索：`session-query` 与语义文本提取

审计最后一步是"能从海量事件里捞出我想要的那条"。DSH 的 `session-query` 用 `extractSessionEventText` 提取每个事件的可搜索语义文本：

<pre class="highlight"><code>
<span class="evt-msg">user/message</span>       → 提取 content 里的 text
<span class="evt-msg">assistant/message</span>  → 提取 message.content 里的 text
<span class="evt-tool">tool/call</span>          → 提取 name + arguments
<span class="evt-tool">tool/result</span>        → 提取结果文本 + error name/code
<span class="evt-marker">todo/write</span>         → 提取 todo 的 status + content
<span class="evt-turn">turn/end</span>           → 提取 error reason
<span class="evt-turn">turn/start</span>、<span class="evt-step">step/start</span>、<span class="evt-step">step/end</span>、<span class="evt-chunk">assistant/chunk</span>、<span class="evt-marker">request/header</span>
                   → 结构边界，不提取（它们没有"语义文本"）
</code></pre>

---

## 七、Span 怎么脱敏：Agent trace 的隐私难题

这是和前一篇[日志脱敏](/2026-07-02/日志脱敏-从注解到自动化的完整实现)直接接续的一章。Agent trace 里，脱敏不只是"字段打码"，它有自己更难的三个点。

### 7.1 Agent 场景脱敏更难的三个点

<pre class="highlight"><code>
难点一：工具参数/结果里含明文密钥
  模型调用 shell 工具时，arguments 里可能就是 "export API_KEY=sk-..."
  工具读文件时，result 里可能就是整个 config 文件的明文密码

难点二：模型回显 PII
  模型把用户输入的身份证号、手机号「复述」进 <span class="evt-msg">assistant/message</span>
  这不走你的 DTO toString()，直接就是自由文本

难点三：推理链里可能泄密
  模型在 reasoning 里可能把敏感上下文「想」出来
  这些内容如果原样导出，就是合规事故
</code></pre>

### 7.2 铁律：canonical log 永不改写

先立一条最重要的铁律：

```
canonical log（本地落盘的会话日志）→ 永不脱敏、永不改写
  ↑ 因为它是事件溯源的本体，改了就无法精确回放

脱敏只作用于「导出副本」
  ↑ 即 session-telemetry 往外发的那份
```

这条铁律对应前一篇的一个核心矛盾：**回放保真 vs 隐私合规**。DSH 的选择是"两头都要"——本地日志全量保真（为了能回放、能审计），导出的 telemetry 才脱敏（为了合规）。

### 7.3 DSH 的脱敏挂载点：`session-telemetry/record` waterfall

DSH 在导出侧留了一个明确的脱敏扩展点——`session-telemetry/record` 瀑布（waterfall）：

```ts
// 每个待导出的 record，都会经过这个 waterfall
// 部署方挂载的 listener 可以改写它（脱敏），再交给后端
ctx.waterfall('session-telemetry/record', record, () => record)
```

三个关键性质（对应 `SessionTelemetryCoordinator` 的 `redact`）：

```
性质一：fail-closed
  → listener 抛异常 → 这条 record 被丢弃，绝不带伤发出

性质二：只改导出副本
  → canonical log 永远不被改写

性质三：同步执行、结果深拷贝
  → listener 拿到的是 coordinator 自己的深拷贝，改不坏原数据
```

### 7.4 把前一篇的脱敏体系"搬家"到 Agent 事件流

前一篇日志脱敏讲了四层方案：注解声明层 → TurboFilter 拦截层 → Converter 正则兜底层 → 审计 Appender 层。搬到 Agent 事件流上，对应关系如下：

| 前一篇的层级 | Agent 事件流的对应 | 作用 |
|-------------|------------------|------|
| **注解层**（`@Desensitize`） | 工具 schema 里的敏感字段声明 | 声明"哪些工具参数/结果是敏感的" |
| **拦截层**（TurboFilter） | `tools/pre-execute` 钩子 + 事件 append 时 | 在事件进入日志前拦截 |
| **兜底层**（Converter 正则） | `session-telemetry/record` waterfall 的正则规则 | 导出副本的全局兜底 |
| **审计层**（SensitiveDataChecker） | 对导出 record 的扫描 | 发版前确认零漏报 |

### 7.5 三层脱敏策略

落到工程上，DSH 的 Agent trace 脱敏是三层：

<pre class="highlight"><code>
第一层：入口层（执行前）
  tools/pre-execute 钩子里，对进入工具的参数先脱敏/拦截
  → 敏感数据根本不该进工具

第二层：事件层（append 时）
  在 Session.append 之前，对 <span class="evt-tool">tool/result</span>、<span class="evt-msg">assistant/message</span> 的敏感内容处理
  → 注意：这一层要权衡「回放保真」和「脱敏」——见 7.2 铁律

第三层：导出层（telemetry 时）
  session-telemetry/record waterfall 对导出副本脱敏
  → 这是最安全的一层，因为不碰 canonical log
</code></pre>

**推荐**：把真正的脱敏放在**第三层（导出层）**，第一层做"拦截"（不让敏感数据进入执行），第二层只在有明确合规要求时才做。

### 7.6 异步一致性的老坑

前一篇讲过一个深刻的坑：异步日志里 RingBuffer 存的是**对象引用**，不是字符串快照，导致格式化时刻 ≠ 引用放入时刻，日志内容可能被业务线程事后修改。

Agent trace 有完全一样的坑，而且更隐蔽：

<pre class="highlight"><code>
业务线程：<span class="evt-msg">assistant/message</span> 的 message 对象被 append 进 session
         ↓
         （message 对象是可变引用？如果导出时用的是引用而非快照……）
异步导出线程：结构化克隆/序列化时，对象可能已经变了
</code></pre>

DSH 的解法在 `captureEvent` 里，就是 7.3 提到的 `structuredClone(event.data)`：

```ts
// coordinator 在导出时做深拷贝
body: structuredClone(event.data),
```

**一句话**：导出的那一刻做**深拷贝快照**，断绝引用。这跟前一篇"让 RingBuffer 里存不可变字符串而不是可变 DTO 引用"是同一个道理，只是 Agent 这里用的是 `structuredClone`。

---

## 八、从"会看"到"会用"：落地清单

最后，把整篇收束成一套能落地的分层方案。

### 8.1 分层收口

<pre class="highlight"><code>
会话事件（session log）   → 当 trace 用
  turn/*、step/*、<span class="evt-msg">user/message</span>、<span class="evt-msg">assistant/message</span>、<span class="evt-tool">tool/call</span>、<span class="evt-tool">tool/result</span>
  职责：回放、审计、排障

运行态信号（live bus）    → 当 metrics 用
  agent/*、tools/*
  职责：实时状态、告警

决策留痕（audit events）  → 当 audit log 用
  approval/*、permission/preset、sandbox/mode、<span class="evt-marker">request/header</span>
  职责：问责、合规
</code></pre>

### 8.2 四个落地动作

```
动作一：事件落盘
  → 选 JSONL 或 SQLite 持久化，保证 append-only + seq 连续
  → 这是"可回放"的物理基础

动作二：关联 id
  → 确保 session.id、turn、step、callId、parentSession、seedLength 都打全
  → 这是"能拼起来"的逻辑基础

动作三：脱敏
  → 在导出层（session-telemetry/record）挂脱敏规则
  → canonical log 保真，导出副本合规

动作四：导出 OTel
  → session-telemetry-otel 把事件映射成 OTLP logs，送进你现有的 collector
  → Agent trace 和微服务 trace 在同一个平台汇合
```

### 8.3 一句收束

回到 SLF4J 那篇的结尾：**日志是分布式系统的眼睛**。

Agent 时代，这双眼睛要完成一次进化：

```
从「看请求」→ 看一条请求在哪个服务慢了
到「看思考」→ 看一个 Agent 为什么做了这个决策、调了这个工具、
            走了这条路、最后为什么是这个结果
```

而支撑这次进化的，不是更花哨的仪表盘，而是一个朴素到近乎固执的设计：**把 Agent 的每一步，都变成一条 append-only、可回放、可审计、可脱敏的事件**。

分布式 trace 让我们能"事后看"一条链路；Agent trace 让我们能"事后重演、甚至自愈"一段思考。这就是从 trace 到 Agent trace 的跨越。

---

**参考**：

- [OpenTelemetry 官方文档](https://opentelemetry.io/docs/)
- DeepSeek Harness 源码（事件模型与持久化）：
  - `packages/core/session/src/types.ts` — `SessionEventMap` 事件词汇表
  - `packages/session/session-telemetry/src/coordinator.ts` — 事件 → 遥测 record 的映射
  - `packages/session/session-telemetry-otel/src/index.ts` — OTel OTLP 导出后端
  - `packages/session/session-persistence-jsonl/src/format.ts` — JSONL 落盘格式
  - `packages/session/session-persistence/tests/contract.ts` — 崩溃自愈契约用例
  - `packages/subagent/subagent/src/child-agent.ts` — fork/subagent 会话元信息
- [SLF4J 原理深度剖析](/2026-07-02/SLF4J原理深度剖析)
- [日志脱敏：从注解到自动化的完整实现](/2026-07-02/日志脱敏-从注解到自动化的完整实现)
