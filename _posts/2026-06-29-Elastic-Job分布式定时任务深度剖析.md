---
layout: post
title: "Elastic-Job 分布式定时任务深度剖析 —— 从概念到原理"
author: "Inela"
---

定时任务是后端开发中最常见的需求之一。单机时代，一个 `cron` 表达式加上 `@Scheduled` 就能解决问题。但当服务集群部署后，问题立刻复杂起来：谁执行？执行几次？执行节点宕机了怎么办？

Elastic-Job 是当当网开源（现为 Apache ShardingSphere 生态）的分布式任务调度框架，Lite 模式以**去中心化 + ZooKeeper 协调**的设计在业界独树一帜。本文从概念到原理，深入剖析 Elastic-Job Lite 的架构设计、核心机制和源码实现。

---

## 一、为什么需要分布式定时任务

### 1.1 单机定时任务的三个瓶颈

```
┌──────────────────────────────────────────────────┐
│                  单机定时任务的困境                  │
├──────────────┬──────────────┬─────────────────────┤
│   单点故障    │   性能上限    │    运维碎片化        │
├──────────────┼──────────────┼─────────────────────┤
│ 机器宕机      │ 单机处理能力  │ 10 台机器各有自己的   │
│ 所有定时任务   │ 有限，数据量  │ @Scheduled，      │
│ 全部停摆      │ 增长后无法    │ 没有统一的管理视图    │
│              │ 按时完成      │                     │
└──────────────┴──────────────┴─────────────────────┘
```

单机定时任务的根本问题是：**执行能力与可用性都绑定在一台机器上**。

### 1.2 分布式定时任务的核心挑战

当定时任务从单机走向分布式，需要回答三个问题：

| 问题 | 本质 | Elastic-Job 的答案 |
|------|------|-------------------|
| **谁执行？** | 多节点如何分工 | 分片机制：将任务拆成 N 个分片，分配给不同节点 |
| **执行几次？** | 如何避免重复执行 | 分片与实例一一对应，同一分片不会同时在两个节点执行 |
| **宕机怎么办？** | 节点故障后任务如何接管 | 失效转移 + 自诊断修复 |

### 1.3 业界方案对比

| 框架 | 架构 | 注册中心 | 分片 | 管理界面 |
|------|------|---------|------|---------|
| Quartz 集群版 | 去中心化（数据库锁） | 数据库 | ❌ 不支持 | ❌ 无 |
| XXL-JOB | 中心化（调度中心） | 数据库（自研） | ✅ 支持 | ✅ 完善 |
| **Elastic-Job Lite** | **去中心化（ZK协调）** | **ZooKeeper** | ✅ 核心特性 | ❌ 无（需配合事件追踪） |
| SchedulerX | 中心化（阿里云） | 阿里云托管 | ✅ 支持 | ✅ 阿里云控制台 |

Elastic-Job 的独特之处在于：**没有调度中心，所有节点对等，通过 ZooKeeper 协调**。这意味着没有单点故障，任何节点宕机都不影响整体调度。

---

## 二、Elastic-Job 架构全景

### 2.1 Lite vs Cloud

Elastic-Job 有两个子项目：

```
Elastic-Job
├── Elastic-Job-Lite  ← 本文重点
│   - 轻量级，jar 包嵌入业务应用
│   - 无中心化，仅依赖 ZooKeeper
│   - 适合业务系统内嵌使用
│
└── Elastic-Job-Cloud
    - 基于 Mesos/Docker 的云原生方案
    - 有调度中心（Scheduler）
    - 资源弹性伸缩，适合大规模调度
```

Lite 模式是绝大多数团队的选择：引入一个 jar 包、配置一个 ZK 地址，就能拥有分布式定时任务能力。

### 2.2 去中心化设计哲学

```
                    ┌─────────────────────┐
                    │   ZooKeeper 集群     │
                    │  (唯一的协调中心)     │
                    └──┬──────┬──────┬────┘
                       │      │      │
              ┌────────┼──────┼──────┼────────┐
              │        │      │      │        │
              ▼        ▼      ▼      ▼        ▼
         ┌────────┐┌────────┐┌────────┐┌────────┐
         │ 节点 A  ││ 节点 B  ││ 节点 C  ││ 节点 D  │
         │ Quartz ││ Quartz ││ Quartz ││ Quartz │
         │ 分片 0  ││ 分片 1  ││ 分片 2  ││ 分片 3  │
         └────────┘└────────┘└────────┘└────────┘
         
核心特点：
- 每个节点都是"完整的"：本地 Quartz 调度 + 任务执行逻辑
- 节点之间不知道彼此存在，只与 ZK 通信
- ZK 是协调中心，不是调度中心 —— 它不"发号施令"，只提供状态存储 + 变更通知
```

这个设计的关键洞察是：**调度逻辑下沉到每个节点，协调逻辑上浮到 ZK**。每个节点本地 Quartz 到点就触发，但"该不该执行"由 ZK 中的分片信息决定。

### 2.3 整体启动流程

```
应用启动
    │
    ▼
注册中心连接（ZK）
    │
    ▼
作业注册：写入 config、servers、instances 节点
    │
    ▼
主节点选举（LeaderLatch）
    │
    ▼
主节点执行分片分配
    │
    ▼
开启本地 Quartz 调度
    │
    ▼
Cron 触发 → 查 ZK 分片信息 → 执行分配给自己的分片
```

---

## 三、ZooKeeper 注册中心 —— 分布式协调的基石

### 3.1 为什么选 ZooKeeper？

Elastic-Job 的协调需求正好命中 ZK 的三大核心能力：

| ZK 特性 | Elastic-Job 用途 |
|---------|-----------------|
| **临时节点** | 实例上线建临时节点，宕机自动删除 → 集群自动感知节点变化 |
| **Watcher 机制** | 监听分片变化、配置变更、实例上下线 → 事件驱动，无需轮询 |
| **分布式锁（Curator）** | 主节点选举（LeaderLatch）、失效转移竞争 → 互斥操作保证 |

> 结合[前文 ZK Watcher 机制分析](https://mingjunduan.github.io)的知识：Elastic-Job 依赖 Curator 的 `PathChildrenCache` 和 `NodeCache` 自动处理 Watch 重注册，开发者无需关心 ZK 一次触发的问题。

### 3.2 命名空间与节点树全景

每个作业在 ZK 中的完整目录结构：

```
/${namespace}/${jobName}/
│
├── config                          ← 持久节点，JSON 格式
│   └── 存储：cron、分片总数、分片参数、作业类型、监听器配置等
│
├── servers/                        ← 持久节点
│   ├── 192.168.1.101              ← 持久节点，存服务器状态
│   └── 192.168.1.102
│
├── instances/                      ← 临时节点（核心！）
│   ├── 192.168.1.101@-@8372       ← 实例下线后自动删除
│   └── 192.168.1.102@-@9231       ← 格式：IP@进程ID
│
├── sharding/                       ← 分片信息
│   ├── 0/
│   │   ├── instance               ← 持久节点：该分片分配给哪个实例
│   │   ├── running                ← 临时节点：分片正在执行
│   │   ├── misfire                ← 持久节点：错过执行标记
│   │   └── disabled               ← 持久节点：禁用标记
│   ├── 1/
│   │   └── ...
│   └── 2/
│       └── ...
│
├── leader/                         ← 主节点相关
│   ├── election/
│   │   ├── latch/                 ← 选举用临时顺序节点（Curator 管理）
│   │   └── instance               ← 临时节点：当前主节点实例 ID
│   ├── sharding/
│   │   ├── necessary              ← 持久节点：标记需要重新分片
│   │   └── processing             ← 临时节点：正在执行分片（阻塞任务）
│   └── failover/
│       ├── latch/                 ← 失效转移竞争锁
│       └── items/                 ← 待转移的分片项
│           ├── 0
│           └── 2
│
└── guarantee/                      ← 分布式栅栏
    ├── started/                    ← 标记已启动的实例
    └── completed/                  ← 标记已完成的实例
```

### 3.3 节点类型与生命周期的对应关系

一个核心设计原则：

```
临时节点 (EPHEMERAL)
  ├── instances/*       → 实例存活 = ZK Session 存活
  ├── sharding/*/running → 任务正在执行 = Session 存活
  ├── leader/election/instance → 主节点存活 = Session 存活
  └── leader/sharding/processing → 正在分片 = Session 存活

持久节点 (PERSISTENT)
  ├── config            → 只要作业存在就存在
  ├── servers/*         → 服务器注册信息持久化
  ├── sharding/*/instance → 分片分配关系持久化（超出 Session 生命周期）
  └── leader/sharding/necessary → 重分片标记持久化
```

**关键设计意图**：临时节点绑定 Session 生命周期，宕机即消失 → 触发集群感知。持久节点记录分配关系，超出单次 Session → 用于一致性校验和恢复。

### 3.4 连接 ZK 的代码入口

```java
// 注册中心配置
CoordinatorRegistryCenter regCenter = new ZookeeperRegistryCenter(
    new ZookeeperConfiguration("zk1:2181,zk2:2181,zk3:2181", "elasticjob-namespace")
);
regCenter.init();

// 作业配置
JobConfiguration jobConfig = JobConfiguration.newBuilder("myJob", 3)
    .cron("0/5 * * * * ?")
    .shardingItemParameters("0=Beijing,1=Shanghai,2=Guangzhou")
    .build();

// 启动
new ScheduleJobBootstrap(regCenter, new MySimpleJob(), jobConfig).schedule();
```

几行代码就完成了一个分布式定时任务的搭建。背后是 ZK 节点树的全自动管理。

### 3.5 单节点执行 vs 多节点执行 —— ZK 数据结构的差异

很多开发者以为"单节点执行"就是只有一个实例注册到 ZK。实际上并非如此——差异不在 `instances/`，而在 `sharding/` 子树。

#### 场景设定

```
3 个实例：节点 A (IP 101)、节点 B (IP 102)、节点 C (IP 103)
作业名：myJob，命名空间：elasticjob
```

#### 单节点执行（shardingTotalCount = 1）

```
/${namespace}/myJob/
│
├── config
│   └── {"cron":"0/5 * * * * ?", "shardingTotalCount":1, ...}
│
├── servers/
│   ├── 192.168.1.101
│   ├── 192.168.1.102
│   └── 192.168.1.103
│
├── instances/                         ← 3 个实例都注册了！
│   ├── 192.168.1.101@-@8372
│   ├── 192.168.1.102@-@9231
│   └── 192.168.1.103@-@5721
│
├── sharding/
│   └── 0/                             ← 只有一个分片
│       ├── instance → "192.168.1.102@-@9231"  ← 只分给 B
│       ├── running  → "192.168.1.102@-@9231"  ← 只有 B 在执行
│       ├── misfire
│       └── disabled
│
├── leader/                            ← 与多节点完全相同
│   ├── election/...
│   ├── sharding/...
│   └── failover/...
│
└── guarantee/...
```

执行时刻：

```
10:00:00 Cron 触发
  节点 A → 查 /sharding/0/instance = "B@9231" ≠ 我  → skip
  节点 B → 查 /sharding/0/instance = "B@9231" = 我  → 执行!
  节点 C → 查 /sharding/0/instance = "B@9231" ≠ 我  → skip
```

**关键**：3 个实例都在 ZK 注册，3 个 Quartz 都触发，但只有 1 个真正执行。任意一个宕机，分片可以 failover 到另一个实例——这就是"弹性"。

#### 多节点执行（shardingTotalCount = 3）

```
/${namespace}/myJob/
│
├── config
│   └── {"cron":"0/5 * * * * ?", "shardingTotalCount":3,
│         "shardingItemParameters":"0=Beijing,1=Shanghai,2=Guangzhou", ...}
│
├── instances/                         ← 完全相同！
│   ├── 192.168.1.101@-@8372
│   ├── 192.168.1.102@-@9231
│   └── 192.168.1.103@-@5721
│
├── sharding/                          ← 这里有 3 个分片子目录
│   ├── 0/
│   │   ├── instance → "192.168.1.101@-@8372"  ← 分给 A
│   │   ├── running  → "192.168.1.101@-@8372"  ← A 在执行
│   │   ├── misfire
│   │   └── disabled
│   ├── 1/
│   │   ├── instance → "192.168.1.102@-@9231"  ← 分给 B
│   │   ├── running  → "192.168.1.102@-@9231"  ← B 在执行
│   │   ├── misfire
│   │   └── disabled
│   └── 2/
│       ├── instance → "192.168.1.103@-@5721"  ← 分给 C
│       ├── running  → "192.168.1.103@-@5721"  ← C 在执行
│       ├── misfire
│       └── disabled
│
├── leader/                            ← 完全相同
│   ├── election/...
│   ├── sharding/...
│   └── failover/...
│
└── guarantee/                         ← 子节点数量不同
    ├── started/
    │   ├── 0 → 101
    │   ├── 1 → 102
    │   └── 2 → 103
    └── completed/
        ├── 0 → 101
        ├── 1 → 102
        └── 2 → 103
```

#### 差异对照一览

```
ZK 路径                    shardingTotalCount=1    shardingTotalCount=3
─────────────────────────  ─────────────────────   ─────────────────────
config                     相同（仅 totalCount 不同）  相同
servers/*                  3 个持久节点             3 个持久节点（相同）
instances/*                3 个临时节点             3 个临时节点（相同）
leader/election/*          相同                    相同
leader/sharding/*          相同                    相同
leader/failover/*          相同                    相同

sharding/                  只有 0/                  有 0/、1/、2/
  {item}/instance          1 个                    3 个，各自指向不同实例
  {item}/running           最多 1 个同时存在         最多 3 个同时存在
  {item}/misfire           1 个                     3 个

guarantee/started/         最多 1 个子节点           最多 3 个子节点
guarantee/completed/       最多 1 个子节点           最多 3 个子节点
```

#### 特殊情况：实例数 > 分片数

```
shardingTotalCount = 2，实例 = 3 (A, B, C)

平均分配结果：
  A: 分片 0
  B: 分片 1
  C: 无分片

ZK 上的 sharding/:
  sharding/0/instance → A@8372
  sharding/1/instance → B@9231

执行时：
  节点 A → 执行分片 0 ✓
  节点 B → 执行分片 1 ✓
  节点 C → instances 里有它，但 sharding/ 下没有它的分片 → skip
```

核心认知：

```
┌────────────────────────────────────────────────────────┐
│                                                       │
│  instances/ 反映"谁在线"                                │
│  sharding/  反映"谁执行什么"                             │
│                                                       │
│  两者是独立的。在线不一定有分片，有分片一定在线。          │
│                                                       │
│  单节点执行 = sharding/ 下只有 1 个子目录                 │
│  多节点执行 = sharding/ 下有 N 个子目录                   │
│                                                       │
│  其他 ZK 路径（instances、leader、servers）完全一样。      │
│  分片数 ≠ 实例数，两者设计上就解耦。                       │
└────────────────────────────────────────────────────────┘
```

---

## 四、主节点选举 —— 谁来做全局决策

### 4.1 为什么需要主节点？

去中心化架构中，大部分操作（触发调度、执行任务）各节点独立完成。但有一个操作需要唯一决策者：**分片分配**。

如果每个节点都自己算分片，可能出现 A 算出来分片 0 给自己、B 也算出来分片 0 给自己 —— 造成重复执行。所以需要一个"临时协调者"来做分片计算并把结果写入 ZK。

**主节点不是调度中心**，它不做任务调度，只做分片分配这一件事。

### 4.2 LeaderLatch 选举原理

Elastic-Job 使用 Curator 的 `LeaderLatch` 实现选举，底层是 ZK 的临时顺序节点：

```java
// 源码位置：LeaderService.electLeader()
public void electLeader() {
    log.debug("Elect a new leader now.");
    jobNodeStorage.executeInLeader(LeaderNode.ELECTION_LATCH, new LeaderElectionExecutionCallback());
    log.debug("Leader election completed.");
}

// LeaderElectionExecutionCallback.execute()
public void execute() {
    if (!hasLeadership()) {
        return;
    }
    // 在 /leader/election/instance 写入自己的实例 ID
    jobNodeStorage.fillEphemeralJobNode(LeaderNode.ELECTION_INSTANCE, 
        JobRegistry.getInstance().getJobInstance(jobName).getJobInstanceId());
}
```

```
选举过程：
时刻 T0: 3 个节点同时启动，各自在 /leader/election/latch/ 下创建临时顺序节点
         latch/_c_0000000001  ← 节点 A (最小，获得 Leader)
         latch/_c_0000000002  ← 节点 B
         latch/_c_0000000003  ← 节点 C
         
时刻 T1: 节点 A 发现自己序号最小 → 获取 Leadership
         └→ 在 /leader/election/instance 写入 "192.168.1.101@-@8372"

时刻 T2: 节点 A 宕机 → 临时节点 _c_0000000001 和 instance 自动删除
         └→ 节点 B 通过 Watcher 感知 → 成为新 Leader
         └→ 写入 /leader/election/instance = "192.168.1.102@-@9231"
```

### 4.3 以作业为维度的选举

**不同作业的 Leader 可以是不同节点**。例如：

```
集群 3 节点: A(101), B(102), C(103)

作业 "orderJob"  → Leader: A
作业 "userJob"   → Leader: B
作业 "reportJob"  → Leader: C
```

这避免了"所有作业的主节点都在一台机器上"的热点问题。每个作业独立选举，选举范围只在同一 `/${namespace}/${jobName}/leader/election/` 路径下。

### 4.4 选举触发时机

| 触发场景 | 触发方式 | 说明 |
|---------|---------|------|
| 作业初始化 | `registerStartUpInfo()` 中调用 `leaderService.electLeader()` | 每个节点启动时都参与选举 |
| 主节点宕机 | Watcher 感知 `leader/election/instance` 删除 | 剩余节点重新竞争 |
| 主节点放弃 | 主节点主动 `close()` 释放 LeaderLatch | 正常下线 |

### 4.5 主节点的职责范围

```
主节点职责（仅两件事）：
┌─────────────────────────────────────────────┐
│ 1. 分片分配                                  │
│    - 感知到 necessary 标记后执行分片分配       │
│    - 将分片结果写入 /sharding/{item}/instance │
│                                             │
│ 2. 失效转移标记                              │
│    - 感知到实例下线后标记失效分片              │
│    - 写入 /leader/failover/items/{item}      │
└─────────────────────────────────────────────┘

主节点 NOT 做的事：
✗ 不调度任务（各节点本地 Quartz 独立触发）
✗ 不执行失效转移（所有空闲节点竞争执行权）
✗ 不收集执行结果（各节点独立回写 ZK）
```

---

## 五、分片机制 —— 弹性扩容的核心

### 5.1 分片概念澄清

**Elastic-Job 的分片不是数据分片**。它分的是"执行权"（即谁执行哪部分逻辑）。数据怎么分，由开发者在 `process()` 方法中根据分片参数自行处理。

```
shardingTotalCount: 3
shardingItemParameters: 0=Beijing,1=Shanghai,2=Guangzhou

分片结果（平均分配，2 个实例）：
  实例 A: 分片 0, 1  →  处理 Beijing + Shanghai 的数据
  实例 B: 分片 2      →  处理 Guangzhou 的数据
```

### 5.2 分片执行全流程

```
Step 1: 标记阶段
    ┌─ 触发条件发生后，主节点在 /leader/sharding/necessary 创建持久节点
    └─ 这是一个"信号量"，所有节点监听这个节点

Step 2: 阻塞阶段
    ┌─ 主节点创建 /leader/sharding/processing 临时节点
    └─ 所有执行节点发现 processing 存在 → 阻塞等待（暂停任务执行）
    
Step 3: 计算阶段
    ┌─ 主节点读取 /instances 获取在线实例列表
    │─ 主节点读取 /config 获取分片总数
    │─ 主节点按策略计算分片分配方案
    └─ 实例列表按 IP 排序（保证多次计算结果一致）

Step 4: 写入阶段
    ┌─ 主节点事务写入 /sharding/0/instance, /sharding/1/instance ...
    └─ 各节点通过 Watcher 感知分片变更，读取自己负责的分片

Step 5: 解除阶段
    ┌─ 主节点删除 /leader/sharding/processing
    └─ 所有节点退出阻塞，按新分片方案执行任务
```

**关键设计**：任务执行过程中即使有节点上下线，也**不会立即重新分片**，只标记 `necessary`。等到**下次任务触发前**才执行分片。这保证了单次执行周期内的分片稳定性。

### 5.3 分片策略详解

```java
// 策略接口
public interface JobShardingStrategy {
    Map<JobInstance, List<Integer>> sharding(List<JobInstance> jobInstances, 
                                              String jobName, 
                                              int shardingTotalCount);
}
```

#### 5.3.1 平均分配策略（默认）

```
分片数: 5, 实例: [A, B, C]

算法:
 每个实例分到: 5 / 3 = 1 个，余数 2
 前 2 个实例各多分 1 个

结果:
 A: [0, 1]
 B: [2, 3]
 C: [4]

源码逻辑（AverageAllocationJobShardingStrategy）:
 int shardingCount = shardingTotalCount / instancesCount;   // 1
 int remain = shardingTotalCount % instancesCount;           // 2
 // 前 remain 个实例各多分配 1 片
```

#### 5.3.2 轮询分配策略

以 **作业名哈希** 对实例列表轮转后平均分配。优点：同一实例不会在所有作业中都分配到相同分片号。

#### 5.3.3 奇偶分配策略

根据实例数量的奇偶性决定 IP 排序方向（升序/降序），轮转后再平均分配。在多作业场景下，分片分布更均匀。

#### 5.3.4 自定义策略

```java
public class MyShardingStrategy implements JobShardingStrategy {
    @Override
    public Map<JobInstance, List<Integer>> sharding(
            List<JobInstance> instances, String jobName, int shardingTotalCount) {
        // 根据机房标签、机器负载等自定义分配逻辑
        Map<JobInstance, List<Integer>> result = new HashMap<>();
        // ... 自定义逻辑
        return result;
    }
}
```

### 5.4 触发重新分片的四个条件

| 条件 | ZK 事件 | 触发者 |
|------|---------|--------|
| 实例上线 | `instances/` 下新增临时节点 | 新启动节点的 Watcher |
| 实例下线 | `instances/` 下临时节点被删除 | 主节点的 Watcher |
| 分片总数变更 | `config` 节点数据变化 | 配置变更 Watcher |
| 自诊断修复 | ReconciliationService 检测到不一致 | 主节点 |

所有触发最终都执行同一个操作：**在 `/leader/sharding/necessary` 创建标记节点**。

### 5.5 实例排序的稳定性保证

分片分配使用 `JobInstance` 列表，排序依据是 **IP 地址**：

```java
// 源码中所有策略都以相同的顺序排列实例
Collections.sort(jobInstances);  // JobInstance 实现了 Comparable，按 IP 排序
```

这保证了：只要在线实例集合相同，每次计算的结果完全一致。避免了"抖动"——分片不会在无变化的情况下重新分配。

### 5.6 源码关键路径

```java
// ShardingService.shardingIfNecessary() —— 分片入口
public void shardingIfNecessary() {
    // 获取可用的分片执行器（需要获取分布式锁）
    Optional<LeaderElectionExecutionCallback> optional = 
        createShardingExecutionCallback();
    if (!optional.isPresent()) {
        return;  // 未获取到锁，其他主节点在处理
    }
    // 检查是否真的需要分片
    if (!isNeedSharding()) {
        return;
    }
    // 执行分片（在分布式锁保护下）
    optional.get().execute();
}

// ShardingExecuteCallback.execute() —— 核心分片逻辑
public void execute() {
    // 1. 获取在线实例
    List<String> instances = jobNodeStorage.getJobNodeChildrenKeys(InstanceNode.ROOT);
    // 2. 获取分片策略并计算
    Map<JobInstance, List<Integer>> shardingResult = 
        jobShardingStrategy.sharding(availableJobInstances, jobName, shardingTotalCount);
    // 3. 事务写入 ZK
    for (Map.Entry<JobInstance, List<Integer>> entry : shardingResult.entrySet()) {
        for (int shardItem : entry.getValue()) {
            jobNodeStorage.replaceJobNode(
                ShardingNode.getInstanceNode(shardItem), 
                entry.getKey().getJobInstanceId());
        }
    }
}
```

---

## 六、任务执行全流程 —— 从 Cron 触发到结果回写

### 6.1 整体时序

```
时间轴 →

T-0:      所有节点的本地 Quartz 同时触发 Cron 信号
              │
T-1:      每个节点检查 /leader/sharding/necessary 是否需要分片
              │   ├─ 需要 → 等待主节点完成分片
              │   └─ 不需要 → 继续
              │
T-2:      每个节点读取 /sharding/*/instance，获取自己负责的分片号
              │   ├─ "我有分片 [0, 1]" → 执行
              │   └─ "我没有分片"  → 跳过（这是正常情况!）
              │
T-3:      创建 /sharding/0/running 临时节点（执行锁）
              │
T-4:      执行 before 监听器（分布式栅栏：所有分片就绪才放行）
              │
T-5:      执行 process(ShardingContext) 业务逻辑
              │
T-6:      执行 after 监听器（分布式栅栏：所有分片完成才放行）
              │
T-7:      删除 /sharding/0/running 临时节点
              │
T-8:      等待下一个 Cron 触发
```

### 6.2 "所有节点同时触发，但只有部分执行"的设计

这是 Elastic-Job 最有意思的设计之一：

```
┌──────────────────────────────────────────────┐
│ 为什么不让主节点触发，然后分发给执行节点？        │
├──────────────────────────────────────────────┤
│ 1. 去中心化：主节点宕机不影响任务调度触发        │
│ 2. 降低延迟：不需要"调度→分发→确认"的链路       │
│ 3. 简化模型：每个节点的 Quartz 独立运行          │
│                                              │
│ 代价：节点时钟不一致可能导致触发偏差             │
│ 解决：依赖 ZK 的分片信息做"执行守卫"             │
└──────────────────────────────────────────────┘
```

每个节点触发后做的第一件事是**查 ZK 上的分片分配**：我是分片 0 的执行者吗？是就执行，不是就跳过。这保证了即使所有节点同时触发，每个分片只被执行一次。

### 6.3 分布式栅栏 —— 执行前/后监听器

```
场景：3 个分片分布在 2 台机器上
  机器 A: 分片 0, 1
  机器 B: 分片 2

before 监听器（分布式栅栏）:
  机器 A 分片 0 done ─────┐
  机器 A 分片 1 done ─────┼──→ 全部就绪 → 放行
  机器 B 分片 2 done ─────┘

作用：确保所有分片的数据准备就绪后再开始执行业务逻辑
```

实现原理：

```java
// GuaranteeService.registerStart() —— 分布式开始栅栏
public void registerStart(List<Integer> shardingItems) {
    for (int each : shardingItems) {
        // 在 /guarantee/started/{item} 创建临时节点
        jobNodeStorage.fillEphemeralJobNode(
            GuaranteeNode.getStartedNode(each), "");
    }
}

// 等待所有分片就绪的判断
public boolean isAllStarted() {
    return jobNodeStorage.getJobNodeChildrenKeys(GuaranteeNode.STARTED_ROOT)
        .size() >= configService.load(true).getTypeConfig().getCoreConfig()
            .getShardingTotalCount();
}
```

### 6.4 错过任务重执行（Misfire）

当任务执行时间超过 Cron 间隔时，Quartz 会标记 Misfire。Elastic-Job 的处理：

```
Cron: 每 5 秒执行，但上次执行花了 8 秒
         │
    0s   ├─ 触发#1 (执行 8s, 到 8s 结束)
    5s   ├─ 触发#2 应该发生但 #1 还在跑 → Misfire!
   10s   ├─ 触发#3 (正常)
    
Elastic-Job 处理:
    触发#2 被标记为 misfire → 写入 /sharding/{item}/misfire
    → 如果配置了 misfire=true，在下次触发时补执行
```

```
Misfire 配置:
  misfire: true   → 错过触发的任务会被补偿执行
  misfire: false  → 错过就错过了，等下一次正常触发
```

### 6.5 Cron 表达式深度解析 —— 四个常见追问

#### 6.5.1 Cron 是谁控制的？每个节点各自执行吗？

**答案：每个节点有自己的 Quartz 实例，各自独立触发，互不通信。**

```
┌──────────────────────────────────────────────────┐
│          常见的误解："主节点触发然后分发"            │
│                                                  │
│  ✗ 错误理解：                                     │
│    主节点 Quartz 触发 → 通知 A 执行分片 0           │
│                      → 通知 B 执行分片 1           │
│                                                  │
│  ✓ 实际机制：                                     │
│    节点 A Quartz 触发 → 查 ZK → "我的分片是 0" → 执行│
│    节点 B Quartz 触发 → 查 ZK → "我的分片是 1" → 执行│
│    节点 C Quartz 触发 → 查 ZK → "我没有分片"   → 跳过│
│                                                  │
│    三个 Quartz 同时响、各自判、互不依赖              │
└──────────────────────────────────────────────────┘
```

Cron 表达式的流转路径：

```
开发者配置 → 写入 ZK /config（持久节点）→ 每个节点启动时读 ZK → 设置本地 Quartz
```

所以 Cron 表达式的控制链是：**你配置 → ZK 存储 → 节点自取 → Quartz 自调**。没有"中心调度器"。

#### 6.5.2 后启动的实例会晚于先启动的实例吗？

**会，但仅限于第一个周期。之后完全同步。**

```
场景：Cron = "0/5 * * * * ?"（在 :00, :05, :10, :15... 触发）

节点 A 启动于 10:00:00
  10:00:00 → 触发 ✓
  10:05:00 → 触发 ✓

节点 B 启动于 10:02:00
  10:00:00 → 已过时，不触发（或标记 misfire）
  10:05:00 → 触发 ✓（跟 A 同时）
```

关键认知：Quartz 的 Cron 是基于**绝对壁钟时间**，不是"启动后每 5 分钟"。`0/5 * * * * ?` 的含义是"在分钟数为 0、5、10、15… 的时间点触发"，不是"启动后经过 5 分钟触发"。只要两个节点的时钟同步（NTP），它们会在**同一时刻**触发。

#### 6.5.3 Cron 是什么时候写入 ZK 的？每个实例都会写吗？

**第一个实例写入，后续实例只读不写。**

```java
// 源码关键路径：SchedulerFacade.registerStartUpInfo()
// → ConfigService.setUpConfiguration()
public void setUpConfiguration(LiteJobConfiguration liteJobConfig) {
    if (jobNodeStorage.isJobNodeExisted(ConfigurationNode.ROOT)) {
        // ZK 上已存在 → 仅校验，不覆盖
        LiteJobConfiguration existing = LiteJobConfigurationGsonFactory.fromJson(
            jobNodeStorage.getJobNodeData(ConfigurationNode.ROOT));
        if (!existing.equals(liteJobConfig)) {
            log.warn("Job configuration on ZK is different from local, using ZK's.");
        }
        return;  // 直接返回，不写入
    }
    // ZK 上不存在 → 写入（通常是第一个实例执行）
    jobNodeStorage.replaceJobNode(ConfigurationNode.ROOT, 
        LiteJobConfigurationGsonFactory.toJson(liteJobConfig));
}
```

```
时序：
  节点 A 启动 (10:00:00)
      ├─ 查 ZK /config → 不存在
      └─ 写入 /config = {"cron":"0/5 * * * * ?", ...}
      
  节点 B 启动 (10:02:00)
      ├─ 查 ZK /config → 已存在 ✓
      ├─ 比较本地 cron 与 ZK cron → 一致 → 跳过写入
      └─ 读 ZK /config → 设置本地 Quartz

  节点 C 启动 (10:03:00)
      └─ 同上，只读不写
```

关键结论：**`/config` 是持久节点，写一次，永久存在。** 如果启动时发现本地 cron 与 ZK 不一致，以 **ZK 为准**（并打印警告日志）。

#### 6.5.4 后台修改 Cron 表达式，怎么生效的？

**ZK Watcher 通知 → 每个实例本地 reschedule Quartz。**

```
完整链路：

  [管理后台] 修改 cron: "0/5" → "0/10"
      │
      ▼
  [ZooKeeper] /config 节点数据变更
      │
      ├──── Watcher 触发（NodeCache 感知）────┐
      │                                       │
      ▼                                       ▼
  [节点 A]                               [节点 B]
      │                                       │
      ▼                                       ▼
  比较新旧 cron:                        比较新旧 cron:
  "0/5" ≠ "0/10"                       "0/5" ≠ "0/10"
      │                                       │
      ▼                                       ▼
  Quartz 重新调度:                      Quartz 重新调度:
  scheduler.rescheduleJob(             scheduler.rescheduleJob(
    oldTrigger, newTrigger)              oldTrigger, newTrigger)
      │                                       │
      ▼                                       ▼
  下次按新 cron 触发:                   下次按新 cron 触发:
  10:10 → 10:20 → 10:30              10:10 → 10:20 → 10:30
```

时效性分析：

```
假设当前时间 10:08，修改 cron:

  旧 cron: "0/5 * * * * ?"  → 下次触发 10:10
  新 cron: "0/10 * * * * ?" → 下次触发 10:10
  
  → cron 变化但下次触发恰好重合（10:10），延迟约 2 分钟

  旧 cron: "0/5 * * * * ?"  → 下次触发 10:10
  新 cron: "0/7 * * * * ?"  → 下次触发 10:14
  
  → 延迟约 6 分钟，期间不会执行

关键细节：
  - 正在执行中的任务不会被中断，新 Cron 只影响下一次触发
  - 如果分片总数也变了，会同时标记 leader/sharding/necessary
```

#### 6.5.5 后台手动触发定时任务，是怎么实现的？

**写一个触发标记到 ZK，每个实例 Watcher 感知后立即调用 `triggerJob()`。**

```
完整链路：

  [管理后台] 点击 "立即执行"
      │
      ▼
  [ZooKeeper] 写入临时节点: /namespace/myJob/trigger
      │
      ├──── Watcher 触发 ────┐
      │                      │
      ▼                      ▼
  [节点 A]              [节点 B]              [节点 C]
      │                      │                      │
      ▼                      ▼                      ▼
  查 ZK 分片表:          查 ZK 分片表:          查 ZK 分片表:
  我的分片 = [0]         我的分片 = [1]         我的分片 = [2]
      │                      │                      │
      ▼                      ▼                      ▼
  triggerJob()           triggerJob()           triggerJob()
  立即执行 分片 0!       立即执行 分片 1!       立即执行 分片 2!
  (不等 Cron)            (不等 Cron)            (不等 Cron)
```

跟 Cron 触发的区别只有一个：Cron 是 Quartz 定时器到点自动调用，手动触发是 ZK Watcher 收到信号后手动调用 `triggerJob()`。**执行逻辑完全一样**（都要查分片表，都只执行自己的分片）。

---

### 6.6 缓存机制与 ZK 压力分析 —— Cron 触发会"打爆"ZK 吗？

如果每次 Cron 触发都要去 ZK 读一堆数据，高频任务确实能把 ZK 打爆。Elastic-Job 的设计对此有充分考虑：**用多层本地缓存 + Curator 缓存机制，将 ZK 读取降到零，只保留必不可少的写入。**

#### 6.6.1 每次 Cron 触发，到底读了什么？

答案：**几乎全是本地内存读取，不访问 ZK。**

```
Cron 触发一次，ZK 操作分解：

  ┌────────────────────────────────────────────────────────┐
  │ 操作                  是否访问 ZK        频率/分片       │
  ├────────────────────────────────────────────────────────┤
  │ 读取分片分配表          否 (TreeCache 缓存)   0          │
  │ 读取 config            否 (NodeCache 缓存)    0          │
  │ 检查 necessary 标记    否 (NodeCache 缓存)    0          │
  │ 创建 running 临时节点   是 (必须! 分布式执行锁) 1 次写    │
  │ 删除 running 临时节点   是 (必须! 执行完成)    1 次删    │
  └────────────────────────────────────────────────────────┘
```

**只有 running 节点的创建和删除需要访问 ZK，这是分布式执行锁，不能用任何缓存替代。** 其他所有读操作都被 Curator 的 TreeCache / NodeCache 拦截，走本地内存。

#### 6.6.2 缓存体系全景

```
                     ┌──────────────────────────┐
                     │     ZooKeeper 集群        │
                     └────┬──────┬──────┬───────┘
                          │      │      │
              ┌───────────┼──────┼──────┼───────────┐
              │           │      │      │           │
              ▼           ▼      ▼      ▼           ▼
       ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
       │ 节点 A    │ │ 节点 B    │ │ 节点 C    │ │ 节点 D    │
       ├──────────┤ ├──────────┤ ├──────────┤ ├──────────┤
       │ TreeCache│ │ TreeCache│ │ TreeCache│ │ TreeCache│
       │ /sharding│ │ /sharding│ │ /sharding│ │ /sharding│
       │ 全量缓存  │ │ 全量缓存  │ │ 全量缓存  │ │ 全量缓存  │
       ├──────────┤ ├──────────┤ ├──────────┤ ├──────────┤
       │NodeCache │ │NodeCache │ │NodeCache │ │NodeCache │
       │ /config  │ │ /config  │ │ /config  │ │ /config  │
       ├──────────┤ ├──────────┤ ├──────────┤ ├──────────┤
       │本地变量   │ │本地变量   │ │本地变量   │ │本地变量   │
       │ myItems  │ │ myItems  │ │ myItems  │ │ myItems  │
       │ [0,1]    │ │ [2,3]    │ │ [4,5]    │ │ [6,7]    │
       └──────────┘ └──────────┘ └──────────┘ └──────────┘
       
数据流：
  读分片分配   → TreeCache (内存) → 不访问 ZK
  读配置       → NodeCache (内存) → 不访问 ZK
  创建 running → 必须访问 ZK (执行锁，不能缓存)
  
数据变更时：
  ZK /config 变化  → Watcher 推送 → NodeCache 更新 → 本地重新调度 Quartz
  ZK /sharding 变化 → Watcher 推送 → TreeCache 更新 → myItems 重新计算
```

#### 6.6.3 TreeCache 原理 —— 关键优化

```java
// 节点启动时，TreeCache 把 /sharding/ 子树全量加载到本地内存
TreeCache treeCache = new TreeCache(client, "/namespace/myJob/sharding");
treeCache.start();

// Watcher 持续监听 ZK 变更，自动增量更新本地缓存
// 当 Cron 触发时：
// getLocalShardingItems() → 直接读 TreeCache → 纯内存操作，零网络调用
List<Integer> myItems = cachedShardingItems.getInstanceItems(instanceId);
```

这就是 Curator `TreeCache` 的价值：**启动时一次全量拉取 + Watcher 持续增量同步**。之后的所有读取都是内存操作。它解决了[前文 ZK Watcher 分析](./)中"每次触发需要重新注册 Watcher"的问题——Curator 在底层自动处理了 Watcher 的重新注册。

#### 6.6.4 running 节点会给 ZK 造成压力吗？

算一笔账：

```
假设场景：10 个作业，每个 3 个分片，Cron = 每 5 秒

每秒 ZK 写入量：
  每次触发：3 个 running 创建 + 3 个 running 删除 = 6 次写
  每分钟：6 × (60/5) = 72 次写
  每秒：72 / 60 = 1.2 次写

ZK 的写能力：
  单节点 ZK：~10,000-20,000 TPS（顺序写入 ZAB 日志）
  3 节点 ZK 集群：~5,000-10,000 TPS（受限于 ZAB 协议广播到 Follower）

结论：1.2 TPS vs 5,000+ TPS → 不到 0.03%，完全可以忽略
```

**极端场景推演**：

```
场景：100 个作业，每个 10 个分片，Cron = 每秒 1 次（这本身就不合理）

每秒 running 写 = 100 × 10 × 2 = 2,000 次/秒
2,000 / 5,000 = 40% 的 ZK 写容量 → 开始有压力

但现实是：
  - 不会有 100 个作业都每秒执行（Cron 精度通常在分钟级）
  - running 节点分散在不同路径 /{namespace}/{jobName}/sharding/{item}/running
    → 不是热点写入，分散在不同 ZNode 上
```

#### 6.6.5 真正的 ZK 压力来自哪里？

**不是 Cron 触发，而是实例频繁上下线和 Session 超时重建。**

```
┌─────────────────────────────────────────────────────────┐
│ 高压力操作              压力来源              真实风险    │
├─────────────────────────────────────────────────────────┤
│ Cron 触发 running 写    每次任务执行时         低         │
│ Watcher 注册/触发       实例上下线/配置变更    低-中      │
│ 选举 (LeaderLatch)      实例上下线时           低         │
│ 分片写入                实例上下线/配置变更    低         │
├─────────────────────────────────────────────────────────┤
│ ⚠️ 大量实例频繁上下线   每次上下线触发全流程    中         │
│ ⚠️ 大量作业同时启动     瞬时大量节点写入        中-高      │
│ ⚠️ ZK Session 超时      所有临时节点全删+重建  高         │
└─────────────────────────────────────────────────────────┘
```

最需要警惕的场景：**ZK Session 过期后所有实例同时重连**——几百个临时节点瞬间全部重建 + 所有 Watcher 同时触发 + 所有作业同时选举 + 所有作业同时分片。这是[前文分析 ZK Session 机制](./)时提到的经典高负载场景。

设置建议：

```
sessionTimeoutMilliseconds: 60000-120000
  - 太短：轻微网络抖动就 Session 超时 → 大规模重建 → ZK 压力瞬间飙升
  - 太长：节点真实宕机后要等太久 → 失效转移延迟大

经验值：60s 适合大多数场景，网络不稳定可调到 120s
```

---

## 七、失效转移 —— 节点宕机后谁接手

### 7.1 触发条件

失效转移不是默认开启的，需要同时满足：

```
failover = true           ← 作业级别配置
monitorExecution = true   ← 作业级别配置（默认 true）
+ 某个节点在执行任务期间宕机
+ 该节点的 running 临时节点因 Session 过期被 ZK 删除
```

> **为什么需要 `monitorExecution=true`？** 
> 因为它决定了是否创建 `running` 临时节点来监控执行状态。没有 `running` 节点，就无法区分"节点在执行中宕机"和"节点根本没执行"。

### 7.2 完整六阶段流程

```
时刻 T0: 节点 A(分片0) + 节点 B(分片1) 正常执行中
         /sharding/0/running → "A@8372" (临时节点)
         /sharding/1/running → "B@9231" (临时节点)

时刻 T1: 节点 A 宕机，进程崩溃
         └→ ZK Session 超时 → /sharding/0/running 自动删除
         
阶段 1 - 故障检测:
         主节点(节点B) Watcher 感知到 /sharding/0/running 被删除
         └→ 检查 /instances 中节点 A 的临时节点也已消失
         └→ 确认节点 A 宕机（不是任务正常完成）
         
阶段 2 - 标记待转移:
         主节点写入 /leader/failover/items/0 (持久节点)
         └→ 含义："分片 0 需要被接管"

阶段 3 - 选举执行者:
         所有空闲节点竞争 /leader/failover/latch 分布式锁
         └→ 节点 C 抢到锁（节点 B 正在执行分片 1，不参与竞争）

阶段 4 - 转移执行:
         节点 C:
         ① 创建 /sharding/0/failover 标记节点
         ② 创建 temporary /sharding/0/running → "C@5721"
         ③ 删除 /leader/failover/items/0
         ④ 立即触发分片 0 的任务执行（不等待 Cron）

阶段 5 - 清理:
         执行完成后删除 /sharding/0/running、/sharding/0/failover

阶段 6 - 分片恢复:
         下次 Cron 触发前，主节点检测到实例变更
         └→ 重新分片，分片 0 正式分配给节点 C
```

### 7.3 失效转移 vs 重新分片 —— 本质区别

```
失效转移 (Failover)                  重新分片 (Resharding)
─────────────────────              ─────────────────────
触发时机：任务执行中，节点宕机        触发时机：下次任务触发前
响应速度：秒级（立即接管）            响应速度：分钟级（等下次 Cron）
作用范围：仅故障分片                  作用范围：全部分片重新分配
实现方式：分布式锁竞争                 实现方式：主节点统一计算
配置要求：failover=true            自动触发（无需额外配置）
             + monitorExecution=true
```

失效转移是"急救"，重新分片是"康复"。两者互补：失效转移保证当前任务不中断，重新分片保证长期运行均衡。

### 7.4 关键源码

```java
// FailoverService.failoverIfNecessary() —— 每个节点定时检查
public void failoverIfNecessary() {
    if (!needFailover()) {
        return;
    }
    // 竞争失效转移执行权
    jobNodeStorage.executeInLeader(FailoverNode.LATCH, new FailoverLeaderExecutionCallback());
}

// FailoverLeaderExecutionCallback.execute() —— 获取锁后执行
public void execute() {
    // 获取一个待转移的分片
    List<String> items = jobNodeStorage.getJobNodeChildrenKeys(FailoverNode.ITEMS_ROOT);
    if (items.isEmpty()) return;
    int crashedItem = Integer.parseInt(items.get(0));
    
    // 标记当前实例为执行者
    jobNodeStorage.fillEphemeralJobNode(
        FailoverNode.getExecutionFailoverNode(crashedItem), 
        JobRegistry.getInstance().getJobInstance(jobName).getJobInstanceId());
    
    // 移除失效转移标记
    jobNodeStorage.removeJobNodeIfExisted(FailoverNode.getItemsNode(crashedItem));
    
    // 立即触发执行（不等 Cron）
    jobScheduleController.triggerJob();
}
```

### 7.5 双节点级联故障 —— 已知的边界问题

```
场景：仅 2 个节点，节点 B 宕机后又恢复，恢复期间节点 A 也宕机

问题：
  1. 节点 B 异常终止时，/sharding/1/running 残留（未正常清理）
  2. 节点 A 等待 B 的 running 节点被删除…但它永远不会自动删除
     因为 B 恢复后会创建新的 Session，旧的 running 节点已"无主"
  3. LegacyCrashedRunningItemListener 仅在单实例场景做清理

影响：分片 1 的 running 节点一直存在 → 调度永久卡住

应急修复：
  zkCli.sh -server localhost:2181
  delete /namespace/jobName/sharding/1/running
```

这个问题的本质是：临时节点的 Session 已经过期，但 ZK 客户端重连时没有清理"上一个 Session 创建的临时节点"。这也是我们[前文分析 ZK Session 机制](https://mingjunduan.github.io)时提到的经典场景。

---

## 八、自诊断修复 —— 最终一致性的兜底

### 8.1 为什么需要自诊断？

失效转移是**事件驱动**的（Watcher 触发），但分布式环境下有些场景 Watcher 可能漏掉：

```
场景 1: 网络抖动
  节点 A 短暂断连 → ZK 临时节点删除 → 分片重新分配给 B
  节点 A 恢复 → 但分片分配已变，A 不知道自己"丢"了分片
  (Watcher 在断连期间丢失，前文 ZK 系列详细分析过)

场景 2: ZK 数据残留
  节点直接 kill -9，running 节点没来得及清理
  Session 过期后 ZK 删除临时节点，但 sharding/{item}/instance 是持久节点
  导致分片分配给了"已不在线"的实例

场景 3: 时间窗口竞态
  分片过程中发生网络分区
  分片写入与实例上下线穿插发生
```

失效转移处理**单点故障的即时接管**，自诊断处理**跨周期的状态不一致**。两者互补。

### 8.2 ReconcileService —— 定时巡检

```java
// ReconcileService 继承 Guava AbstractScheduledService
@Override
protected Scheduler scheduler() {
    // 每分钟检查一次
    return Scheduler.newFixedDelaySchedule(0, 1, TimeUnit.MINUTES);
}

@Override
protected void runOneIteration() throws Exception {
    LiteJobConfiguration config = configService.load(true);
    int reconcileIntervalMinutes = null == config ? -1 : config.getReconcileIntervalMinutes();
    
    // 到达配置的修复间隔才执行（不是每分钟都修）
    if (reconcileIntervalMinutes > 0 
        && (System.currentTimeMillis() - lastReconcileTime 
            >= reconcileIntervalMinutes * 60 * 1000)) {
        
        lastReconcileTime = System.currentTimeMillis();
        
        // 三个条件必须同时满足
        if (leaderService.isLeaderUntilBlock()           // ① 当前是主节点
            && !shardingService.isNeedSharding()          // ② 没有正在进行的分片
            && shardingService.hasShardingInfoInOfflineServers()) { // ③ 存在不一致
            log.warn("Elastic Job: job status node has inconsistent value, "
                + "start reconciling...");
            shardingService.setReshardingFlag();
        }
    }
}
```

### 8.3 不一致检测的核心逻辑

```java
// ShardingService.hasShardingInfoInOfflineServers()
public boolean hasShardingInfoInOfflineServers() {
    // 从 ZK 获取在线实例列表（临时节点）
    List<String> onlineInstances = 
        jobNodeStorage.getJobNodeChildrenKeys(InstanceNode.ROOT);
    
    // 遍历所有分片，检查分片分配的实例是否在线
    int shardingTotalCount = configService.load(true)
        .getTypeConfig().getCoreConfig().getShardingTotalCount();
    
    for (int i = 0; i < shardingTotalCount; i++) {
        String assignedInstance = 
            jobNodeStorage.getJobNodeData(ShardingNode.getInstanceNode(i));
        if (!onlineInstances.contains(assignedInstance)) {
            return true;  // 发现"幽灵分片"：分配给已下线实例
        }
    }
    return false;
}
```

```
关键对比：
  /instances/                   ← 临时节点，反映"当前谁在线"
  /sharding/{item}/instance     ← 持久节点，记录"分片分配给谁"
  
  不一致 = 持久分片记录指向了不在临时实例列表中的实例
       → 说明该实例已下线，但分片未被重新分配
       → 设置 necessary 标记 → 触发重新分片
```

### 8.4 自诊断 vs 失效转移 vs 事件驱动—— 三层保障

```
              ┌─────────────────────────────┐
              │    事件驱动 (Watcher)         │  ← 第一层：实时感知
              │    实例变化 → 立即标记重分片    │
              ├─────────────────────────────┤
              │    失效转移 (Failover)        │  ← 第二层：即时接管
              │    运行中宕机 → 秒级接管        │
              ├─────────────────────────────┤
              │    自诊断 (Reconciliation)    │  ← 第三层：周期兜底
              │    定时巡检 → 修复遗漏         │
              └─────────────────────────────┘
```

---

## 九、作业类型生态

Elastic-Job 3.x 基于 ShardingSphere 可插拔架构，将作业解耦为**作业接口**和**执行器接口**，内置四种作业类型并支持 SPI 扩展。

### 9.1 Simple 作业 —— 最简单模式

```java
public class MySimpleJob implements SimpleJob {
    @Override
    public void execute(ShardingContext shardingContext) {
        // shardingContext 提供：
        //   getShardingItem()       → 当前分片号
        //   getShardingTotalCount() → 分片总数
        //   getShardingParameter()  → 分片参数（如 "Beijing"）
        
        int shardNo = shardingContext.getShardingItem();
        String city = shardingContext.getShardingParameter();
        log.info("分片 {} 处理城市 {}", shardNo, city);
        // 执行实际业务逻辑
    }
}
```

适用：普通定时任务，不需要流式数据处理的场景。

### 9.2 Dataflow 作业 —— 流式处理

Dataflow 作业是处理**持续流入数据**的最佳选择：

```java
public class MyDataflowJob implements DataflowJob<Order> {
    
    @Override
    public List<Order> fetchData(ShardingContext context) {
        // 拉取待处理数据
        // context 告诉你是哪个分片，你据此决定拉取哪部分数据
        int shardNo = context.getShardingItem();
        int totalShard = context.getShardingTotalCount();
        return orderService.fetchPending(shardNo, totalShard, 100);
    }
    
    @Override
    public void processData(ShardingContext context, List<Order> data) {
        // 处理拉取到的数据
        for (Order order : data) {
            orderService.process(order);
        }
    }
}
```

核心配置 `streamingProcess`：

```
streamingProcess = true:
  Cron 触发
    → fetchData() → 返回数据?
    ├→ 有数据 → processData() → fetchData() → ...循环
    └→ 无数据 → 本次触发结束，等下次 Cron
  
streamingProcess = false (默认):
  Cron 触发
    → fetchData() → processData() → 结束
```

**适用场景**：订单超时关闭、消息推送、数据同步等需要"持续消费"的场景。`streamingProcess=true` 等价于"只要还有数据就干活，干完为止"。

### 9.3 Script 作业 —— 零 Java 代码接入

```yaml
elasticjob:
  jobs:
    cleanLogJob:
      job-type: SCRIPT
      cron: "0 0 2 * * ?"
      sharding-total-count: 1
      props:
        script.command.line: "/opt/scripts/clean_logs.sh"
```

支持 Shell、Python、Perl 等，脚本中通过环境变量获取分片上下文：
```bash
#!/bin/bash
echo "分片号: ${SHARDING_ITEM}, 总分片: ${SHARDING_TOTAL_COUNT}"
# 清理对应分片的日志
```

### 9.4 HTTP 作业（3.x 新增）

无需编写 Java 代码，通过配置调用 HTTP 接口即可：

```yaml
elasticjob:
  jobs:
    httpJob:
      job-type: HTTP
      cron: "0 */10 * * * ?"
      sharding-total-count: 1
      props:
        http.url: "http://internal-api/task/execute"
        http.method: "POST"
        http.data: '{"type":"report","ts":"${EXECUTION_TIME}"}'
```

### 9.5 SPI 扩展 —— 自定义作业类型

```
实现步骤:
  1. 实现 Job 接口（或 ElasticJob 接口）
  2. 实现 JobExecutor 接口
  3. 在 META-INF/services/ 下添加 SPI 声明
  4. 配置中指定 job-type 名称即可
```

---

## 十、连接状态与容灾

### 10.1 ZK 连接状态机

```
                  ┌──────────────┐
          ┌──────►│  CONNECTED   │◄─────────┐
          │       │  (正常状态)    │          │
          │       └──────┬───────┘          │
          │              │ 网络闪断           │
          │              ▼                  │
          │       ┌──────────────┐          │
          │       │  SUSPENDED   │          │
          │       │  (连接挂起)   │          │
          │       └──────┬───────┘          │
          │              │                  │
          │     ┌────────┴────────┐         │
          │     ▼                 ▼         │
          │  Session 未过期    Session 过期   │
          │     │                 │         │
          │     ▼                 ▼         │
          │  ┌──────────┐  ┌──────────┐    │
          │  │RECONNECTED│  │   LOST   │    │
          │  │  (重连成功) │  │(彻底断开) │────┘
          │  └──────────┘  └──────────┘   (重新注册)
          │       │
          └───────┘
```

### 10.2 各状态下的作业行为

| 连接状态 | 作业行为 | 原因 |
|---------|---------|------|
| CONNECTED | 正常运行 | ZK 通信正常，分片信息准确 |
| SUSPENDED | **暂停调度** | 可能发生脑裂，暂停执行避免重复 |
| LOST | **暂停调度 + 清理本地状态** | 彻底断连，清空运行信息 |
| RECONNECTED | **重新注册 + 恢复调度** | 重新写入实例节点、分片状态，恢复执行 |

```java
// AbstractScheduleTracker —— 连接状态处理
public void stateChanged(CuratorFramework client, ConnectionState newState) {
    switch (newState) {
        case SUSPENDED:
        case LOST:
            // 暂停作业调度
            jobScheduleController.pauseJob();
            break;
        case RECONNECTED:
            // 重新持久化上线信息
            serverService.persistOnline(true);
            // 清理上次的 running 标记
            executionService.clearRunningInfo(shardingItems);
            // 恢复作业调度
            jobScheduleController.resumeJob();
            break;
    }
}
```

> 结合[前文 ZK Session 分析](https://mingjunduan.github.io)：SUSPENDED 期间是"黄金重连窗口"，只要在 sessionTimeout 内重连成功，临时节点不会消失，集群不会感知到实例变化。

---

## 十一、与 XXL-JOB 的对比

### 11.1 架构对比

```
Elastic-Job (去中心化)              XXL-JOB (中心化)
─────────────────────              ─────────────────
                                   
  [节点A](Quartz+执行)              [调度中心] (单点/集群)
  [节点B](Quartz+执行)               │    │    │
  [节点C](Quartz+执行)               ▼    ▼    ▼
       │    │    │               [执行器A][执行器B][执行器C]
       └────┼────┘
            ▼
      [ZooKeeper 集群]
```

| 维度 | Elastic-Job Lite | XXL-JOB |
|------|-----------------|---------|
| **架构** | 去中心化，节点对等 | 中心化，调度中心 + 执行器 |
| **单点故障** | 无（ZK 集群保证高可用） | 调度中心故障则全部停摆（可集群化但仍有状态同步问题） |
| **注册中心** | ZooKeeper | 数据库（自研） |
| **分片策略** | 平均、轮询、奇偶、自定义（4 种内置） | 分片广播、固定分片 |
| **失效转移** | ✅ 自动接管（秒级） | ✅ 故障转移（依赖调度中心检测） |
| **作业类型** | Simple/Dataflow/Script/HTTP + SPI | 内建 10+ 种（GLUE、Shell、Python、Node.js 等） |
| **管理界面** | ❌ 无内置（需事件追踪系统） | ✅ 完善 Web 控制台 |
| **依赖** | ZK（必须） | 数据库（必须） |
| **运维复杂度** | 低（只需 ZK） | 中（数据库 + 调度中心部署） |
| **社区** | Apache ShardingSphere 子项目 | 大众点评开源，社区活跃 |

### 11.2 选型建议

```
选 Elastic-Job，如果：
  ✅ 团队已有 ZK 集群（不想引入新依赖）
  ✅ 不需要管理界面（或已有自己的监控平台）
  ✅ 看重去中心化的高可用设计
  ✅ 需要灵活的分片策略（平均分配）
  ✅ 系统对延迟敏感（去中心化调度延迟更低）

选 XXL-JOB，如果：
  ✅ 团队没有 ZK，但有数据库
  ✅ 需要管理界面（任务管理、日志查看、手动触发）
  ✅ 需要 GLUE 模式（在线编辑代码）
  ✅ 需要子任务依赖编排
  ✅ 中小团队，运维要简单
```

---

## 十二、实战最佳实践

### 12.1 分片数与实例数的关系

```
原则：分片数 > 实例数（预留扩容空间）

  反例：3 个实例，分片数 = 3
    扩容到 4 个实例时，无法继续拆分 → 需要修改配置 → 可能影响运行中任务
  
  正例：3 个实例，分片数 = 6
    扩容到 4 个实例：每个实例 1~2 个分片 → 自动重新分配
    扩容到 6 个实例：每个实例恰好 1 个分片 → 最大化并行度
    缩容到 2 个实例：每个实例 3 个分片 → 自动合并

建议：分片数 = 期望最大实例数 × 2
```

### 12.2 failover 与 monitorExecution 的配置策略

```
长任务（执行时间 > 30s）:
  failover: true   ← 开启，避免长任务中断
  monitorExecution: true  ← 必须开启

短任务（执行时间 < 5s）:
  failover: false  ← 关闭，等下次调度更简单
  monitorExecution: true  ← 仍建议开启，用于自诊断

高频任务（Cron < 10s）:
  failover: false  ← 频繁转移的开销 > 收益
  misfire: false   ← 错过一次马上就有下次，不补偿
```

### 12.3 reconcileIntervalMinutes 设置

```
默认：-1（不自动修复，仅靠事件驱动）
建议：10 分钟（生产环境默认足够）
若网络不稳定：5 分钟（更快发现不一致）
若 Job 数量巨大（> 100）：15~30 分钟（减少 ZK 读取压力）
```

### 12.4 ZK 连接参数调优

```java
new ZookeeperConfiguration("zk1:2181,zk2:2181,zk3:2181", "elasticjob-namespace")
    .setSessionTimeoutMilliseconds(60000)    // 默认 60s，网络不稳可调大
    .setConnectionTimeoutMilliseconds(15000) // 默认 15s
    .setMaxRetries(3)                        // 重试次数
    .setBaseSleepTimeMilliseconds(1000)      // 重试间隔基数
    .setDigest("user:password");             // ZK ACL 认证

// Session 超时建议：60s ~ 120s
// 太短：轻微网络抖动就触发重新分片 → 集群不稳定
// 太长：节点真实宕机后要等太久才发现 → 失效转移延迟大
```

### 12.5 监控要点

```
关键 ZK 节点路径监控：
  /namespace/jobName/leader/election/instance  ← 主节点是否正常
  /namespace/jobName/instances/                ← 在线实例数是否符合预期?
  /namespace/jobName/leader/sharding/necessary ← 长期存在 = 分片卡住
  /namespace/jobName/leader/failover/items/    ← 长期存在 = 失效转移卡住

应用层监控：
  - 每个分片的执行耗时 (process 方法耗时分布)
  - 每次调度的触发延迟 (Cron 触发时间 vs 实际开始执行时间)
  - 分片分配变化次数 (频繁变化 = 集群不稳定)
  - ZK 连接状态变化次数 (频繁变化 = 网络问题)
```

---

## 十三、总结

### 核心设计理念回顾

```
Elastic-Job 的核心设计决策：

1. 去中心化 + ZK 协调
   "没有调度中心，就没有调度中心的单点故障"
   ZK 只做状态存储和变更通知，不做调度

2. 分片 = 执行权的分布式
   "分片不是数据分片，是谁执行哪部分的分工"
   通过分片将任务拆解为 N 个独立执行单元
   分片数 ≠ 实例数，两者设计上充分解耦

3. 本地 Quartz + ZK 守卫
   "所有节点同步触发，ZK 分片信息决定谁执行"
   不依赖中心触发，靠 ZK 状态做执行守卫
   Cron 是绝对壁钟时间，不是相对启动时间

4. 多层缓存，ZooKeeper 无压力
   TreeCache 缓存 /sharding 子树的全部数据
   NodeCache 缓存 /config 节点
   Cron 触发时只有 running 节点的一次创建和一次删除访问 ZK
   其余操作全部走本地内存

5. 三层一致性保障
   事件驱动(Watcher) → 失效转移(Failover) → 自诊断(Reconciliation)
   实时 → 秒级 → 分钟级，层层兜底
```

### 适用场景总结

```
┌─────────────────────────────────────────────┐
│ Elastic-Job 最适合：                          │
│ • 数据分片处理（按城市/用户ID区间/时间分段）    │
│ • 已有 ZK 集群的团队（零额外依赖）             │
│ • 对高可用要求高的核心业务（去中心化无单点）     │
│ • 需要弹性扩容的定时任务（实例增减自动重分片）   │
├─────────────────────────────────────────────┤
│ 不太适合：                                   │
│ • 需要 Web 管理界面（没有开箱即用的控制台）     │
│ • 小团队快速上手（需要先理解 ZK）             │
│ • 需要任务编排/依赖（不支持 DAG 工作流）       │
└─────────────────────────────────────────────┘
```

---

Elastic-Job 的设计哲学是"**简单即鲁棒**"：没有调度中心就没有中心化故障，依赖 ZK 的三大特性（临时节点、Watcher、分布式锁）就解决了分布式协调。它对开发者的要求是理解分片概念和 ZK 节点树——一旦掌握，你就能搭建一个高可用的分布式定时任务系统。
