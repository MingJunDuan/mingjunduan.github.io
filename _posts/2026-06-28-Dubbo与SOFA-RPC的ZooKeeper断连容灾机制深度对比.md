---
layout: post
title: "Dubbo 与 SOFA RPC 的 ZooKeeper 断连容灾机制深度对比"
author: "Inela"
---

ZooKeeper 挂了怎么办？这是每个用 ZK 做注册中心的团队必须回答的问题。Dubbo 和 SOFA RPC 作为国内最主流的两大 RPC 框架，都用 ZK 做服务发现，但它们的容灾策略有本质差异。

本文从一个真实的生产故障场景切入，从源码层面拆解两个框架的断连处理全链路。

## 一、场景设定

```
┌─────────────────────────────────────────────────────────────┐
│                      故障前正常状态                          │
│                                                             │
│   ZK 集群 (5节点)           Dubbo/SOFA Consumer             │
│   ┌───┐ ┌───┐              ┌─────────────────┐             │
│   │ L │ │ F ├─── 连接 ────►│ Consumer-1       │             │
│   └───┘ └───┘              │  当前Provider列表:│             │
│   ┌───┐ ┌───┐ ┌───┐       │  [10.0.0.1:20880] │             │
│   │ F │ │ F │ │ O │       │  [10.0.0.2:20880] │             │
│   └───┘ └───┘ └───┘       │  [10.0.0.3:20880] │             │
│                             └─────────────────┘             │
│                                                             │
│  消费者连接到 Follower-2 (10.0.0.2:2181)                    │
│  正在调用 OrderService，负载均衡到三个 Provider              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    故障发生: Follower-2 宕机                 │
│                                                             │
│   ZK 集群                     Dubbo/SOFA Consumer           │
│   ┌───┐ ┌───┐               ┌─────────────────┐            │
│   │ L │ │ F │✕ 宕机          │ 问题:            │            │
│   └───┘ └───┘               │ - RPC调用会断吗?  │            │
│   ┌───┐ ┌───┐ ┌───┐        │ - Provider列表丢吗?│           │
│   │ F │ │ F │ │ O │         │ - 多久恢复正常?   │            │
│   └───┘ └───┘ └───┘        └─────────────────┘            │
└─────────────────────────────────────────────────────────────┘
```

**核心问题**：消费者连接的那台 ZK 节点挂了，RPC 调用还能继续吗？

**答案**：能。RPC 框架的设计原则是——**注册中心是辅助，不是主链路**。

---

## 二、Dubbo RPC 的三层容灾架构

Dubbo 的 `ZookeeperRegistry` 继承自 `FailbackRegistry`，名字就说明了设计意图：`Failback` = 失败后自动恢复。

### 2.1 整体架构

```
┌──────────────────────────────────────────────────────────────┐
│                     Dubbo 三层容灾                             │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 第1层: 内存缓存 (ConcurrentHashMap)                     │ │
│  │                                                        │ │
│  │ RegistryDirectory.registryCache:                       │ │
│  │   key: "dubbo://10.0.0.1:20880/..."                    │ │
│  │   value: Provider URL 对象                              │ │
│  │                                                        │ │
│  │ 特性: 内存直接读取，O(1)，不经过 ZK                      │ │
│  │ 生命周期: Consumer 进程存活期间一直有效                  │ │
│  └───────────────────────────┬────────────────────────────┘ │
│                              │ ZK 变更时更新                  │
│                              ▼                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 第2层: 文件缓存 (~/.dubbo/dubbo-registry-{app}.cache)  │ │
│  │                                                        │ │
│  │ 格式 (Properties 文件):                                 │ │
│  │   com.xxx.OrderService=                               │ │
│  │     dubbo://10.0.0.1:20880/com.xxx.OrderService?...,  │ │
│  │     dubbo://10.0.0.2:20880/com.xxx.OrderService?...   │ │
│  │                                                        │ │
│  │ 特性: 进程重启后恢复，不需要 ZK 也加载              │ │
│  │ 更新: 每次内存缓存变化后，异步写入文件                  │ │
│  └───────────────────────────┬────────────────────────────┘ │
│                              │ 启动时加载                     │
│                              ▼                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 第3层: ZooKeeper 实时订阅                               │ │
│  │                                                        │ │
│  │ - 启动时全量拉取 + 注册 Watcher                         │ │
│  │ - 变更时增量更新第1层和第2层                            │ │
│  │ - 断连时: 第1层照常工作，不感知 ZK 状态                 │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 核心源码: AbstractRegistry —— 只要我不死，Provider 列表就在

```java
// Dubbo 2.7.x: AbstractRegistry.java
public abstract class AbstractRegistry implements Registry {

    // ====== 内存缓存 (第1层) ======
    // Properties 是线程安全的，key=service, value=provider列表
    private final Properties properties = new Properties();

    // 文件缓存路径
    private File file;

    public AbstractRegistry(URL url) {
        // 启动时从文件恢复
        this.file = new File(System.getProperty("user.home")
            + "/.dubbo/dubbo-registry-" + url.getParameter("application")
            + "-" + url.getAddress().replace(':', '-') + ".cache");

        if (file.exists()) {
            // 从文件加载到内存 Properties
            properties.load(new FileInputStream(file));
        }
    }

    // ====== 文件缓存 (第2层) ======
    // 异步保存到文件
    private void saveProperties(URL url) {
        String key = url.getServiceKey();  // com.xxx.OrderService
        String value = getCacheUrls(url);  // dubbo://10.0.0.1:20880/...,...

        properties.setProperty(key, value);

        // 异步写文件，不阻塞主流程
        long version = url.getParameter("timestamp", 0L);
        if (version > lastCacheChanged.get()) {
            // 启动一个单线程的 ScheduledExecutor 异步保存
            newSaveThread.schedule(() -> {
                synchronized (file) {
                    properties.store(new FileOutputStream(file), "Dubbo Registry Cache");
                }
            }, 500, TimeUnit.MILLISECONDS);  // 500ms 消抖
        }
    }

    // ☆ 最关键的方法: 获取缓存的 Provider 列表 ☆
    // 这个方法不访问 ZK，只读内存!
    public List<URL> getCacheUrls(URL url) {
        for (Map.Entry<Object, Object> entry : properties.entrySet()) {
            String key = (String) entry.getKey();
            if (key.equals(url.getServiceKey())) {
                // 从文件中解析 Provider URL 列表
                return URL.parseURL((String) entry.getValue(), null);
            }
        }
        return Collections.emptyList();
    }
}
```

**关键设计**：`getCacheUrls()` 这个方法的调用路径完全不经过 ZooKeeper。消费者发起 RPC 调用时，负载均衡组件直接读内存 Properties，不需要跟 ZK 交互。

### 2.3 核心源码: FailbackRegistry —— 失败后自动重试

```java
// FailbackRegistry.java
public abstract class FailbackRegistry extends AbstractRegistry {

    // ====== 三个失败重试集合 ======
    private final Set<URL> failedRegistered = new ConcurrentHashSet<>();     // 注册失败
    private final Set<URL> failedUnregistered = new ConcurrentHashSet<>();   // 注销失败
    private final ConcurrentMap<URL, NotifyListener> failedSubscribed = ...; // 订阅失败

    // 重试定时器: 每隔 retryPeriod (默认 5000ms) 重试
    private final ScheduledExecutorService retryExecutor =
        Executors.newScheduledThreadPool(1);

    public FailbackRegistry(URL url) {
        super(url);
        // 启动定时重试
        this.retryPeriod = url.getParameter("retry.period", 5000);
        retryExecutor.scheduleWithFixedDelay(() -> {
            try {
                retry();  // 重试所有失败的操作
            } catch (Throwable t) {
                log.error("Unexpected error in retry", t);
            }
        }, retryPeriod, retryPeriod, TimeUnit.MILLISECONDS);
    }

    // ====== 每次写操作都套了 try-catch-failback ======
    @Override
    public void register(URL url) {
        try {
            doRegister(url);  // → ZookeeperRegistry 实际写 ZK
            failedRegistered.remove(url);
        } catch (Exception e) {
            // ☆ 失败不抛异常，加入重试队列 ☆
            failedRegistered.add(url);
            log.warn("Failed to register {}, will retry", url, e);
        }
    }

    @Override
    public void subscribe(URL url, NotifyListener listener) {
        try {
            doSubscribe(url, listener);  // → ZookeeperRegistry 实际注册 Watcher
            failedSubscribed.remove(url);
        } catch (Exception e) {
            failedSubscribed.put(url, listener);  // 加入重试队列
            log.warn("Failed to subscribe {}, will retry", url, e);
        }
    }

    // ====== 重试逻辑 ======
    @Override
    protected void retry() {
        // 重试失败的注册
        Iterator<URL> it = failedRegistered.iterator();
        while (it.hasNext()) {
            URL url = it.next();
            try {
                doRegister(url);
                it.remove();
            } catch (Exception e) {
                log.warn("Retry register still failed: {}", url);
            }
        }

        // 重试失败的订阅 (最重要!)
        // Watcher 丢失后需要重新注册
        for (Map.Entry<URL, NotifyListener> entry : failedSubscribed.entrySet()) {
            try {
                doSubscribe(entry.getKey(), entry.getValue());
                failedSubscribed.remove(entry.getKey());
            } catch (Exception e) {
                log.warn("Retry subscribe still failed");
            }
        }
    }
}
```

**设计精髓**：`register()` 和 `subscribe()` 方法对调用方来说是**不会失败的**。ZK 真失败时，操作被记录到重试集合，后台线程每 5 秒重试。调用方永远拿到 `success`（或最多一条 warn 日志）。

### 2.4 核心源码: ZookeeperRegistry —— 断连恢复的精确动作

这是最精彩的部分。Dubbo 3.x 之后开始同时支持 Zookeeper、Nacos 等多种注册中心，但 ZookeeperRegistry 的模式没变：

```java
// ZookeeperRegistry.java (Dubbo 2.7.x)
public class ZookeeperRegistry extends FailbackRegistry {

    private CuratorFramework zkClient;  // Apache Curator 封装

    @Override
    protected void doSubscribe(URL url, NotifyListener listener) {
        String path = toServicePath(url);  // /dubbo/com.xxx.OrderService/providers

        // 创建一个 Cache 对象 (Curator 的 TreeCache / PathChildrenCache)
        // Cache 内部处理了 Watcher 的自动重新注册!
        PathChildrenCache cache = pathChildrenCacheMap.computeIfAbsent(path, p -> {
            PathChildrenCache c = new PathChildrenCache(zkClient, p, true);
            c.getListenable().addListener((client, event) -> {
                // 子节点变化回调
                List<URL> urls = toUrlsWithEmpty(client, path, event.getData());
                // 更新内部 Provider 列表 → 通知 Dubbo 的 NotifyListener
                notify(url, listener, urls);
                // 同时更新文件缓存
                saveProperties(url);
            });
            return c;
        });
        cache.start();
    }

    // ====== Curator 连接状态监听 ======
    private void addStateListener() {
        zkClient.getConnectionStateListenable().addListener((client, state) -> {
            switch (state) {
                case RECONNECTED:
                    // ☆ 重连成功: 核心恢复逻辑 ☆
                    log.info("ZK reconnected, recovering...");
                    recover();
                    break;

                case LOST:
                    // Session 过期
                    // Curator 内部会自动重建连接 (新的 sessionId)
                    // 但所有 Watcher 都丢失了 → 需要全量重建
                    log.warn("ZK session expired!");
                    // recover() 会在 RECONNECTED 中触发
                    break;

                case SUSPENDED:
                    // 断连中 → 不 panic，等待 Curator 自动重连
                    // 内存缓存中的 Provider 列表完全可用
                    log.info("ZK connection suspended, waiting for reconnect...");
                    break;
            }
        });
    }

    // ☆ 恢复方法: 重连后必须做的三件事 ☆
    @Override
    protected void recover() {
        // 1. 重新注册本机的 Provider URL
        Set<URL> recoverRegistered = new HashSet<>(getRegistered());
        if (!recoverRegistered.isEmpty()) {
            log.info("Recovering registered URLs: {}", recoverRegistered.size());
            for (URL url : recoverRegistered) {
                try {
                    // 先检查 ZK 上是否还存在 (可能已被自动清理)
                    if (zkClient.checkExists().forPath(toUrlPath(url)) == null) {
                        doRegister(url);  // 重新创建临时节点
                    }
                } catch (Exception e) {
                    failedRegistered.add(url);
                }
            }
        }

        // 2. 重新订阅: 重新拉取全量 Provider 列表 + 重新注册 Watcher
        Set<URL> recoverSubscribed = new HashSet<>(getSubscribed().keySet());
        if (!recoverSubscribed.isEmpty()) {
            log.info("Recovering subscribed URLs: {}", recoverSubscribed.size());
            for (URL url : recoverSubscribed) {
                try {
                    // 重新订阅 (内部会全量拉取 + 重新设置 Watcher)
                    doSubscribe(url, getSubscribed().get(url));
                } catch (Exception e) {
                    failedSubscribed.put(url, getSubscribed().get(url));
                }
            }
        }
    }
}
```

### 2.5 Dubbo 消费者在 ZK 故障下的完整时间线

```
sessionTimeout = 12000ms (Dubbo 的 zk.session.timeout 默认值)

时间线:
══════════════════════════════════════════════════════════════════►

T=0ms              T=0~几秒               T=sessionTimeout      T=重连后
  │                   │                       │                    │
  ▼                   ▼                       ▼                    ▼
┌──────┐         ┌──────────┐          ┌──────────┐         ┌──────────┐
│ZK节点│         │Curator   │          │Session   │         │全量恢复  │
│宕机  │         │→SUSPENDED│          │过期→LOST │         │          │
│      │         │          │          │          │         │1.recover │
│      │         │消费者：   │          │临时节点被 │         │  注册    │
│      │         │继续用内存 │          │Leader清理 │         │2.recover │
│      │         │缓存调RPC  │          │          │         │  订阅    │
│      │         │          │          │Curator：  │         │3.全量拉取│
│      │         │RPC不受影响│          │自动取新   │         │  Provider│
│      │         │          │          │sessionId  │         │ 列表     │
│      │         │          │          │重连       │         │          │
└──────┘         └──────────┘          └──────────┘         └──────────┘
                         ▲                      ▲                    ▲
                         │                      │                    │
                    Watcher丢失              Watcher全丢           全量恢复
                    但PathChildrenCache      ZK上临时节点被清       Provider列表
                    内部会自动重建            （Provider注册信息）   重新拉取
                                                                   文件缓存也被
                                                                   更新
```

**关键发现**：

1. **RPC 调用不受影响**：Provider 列表在内存中，负载均衡直接读，不走 ZK
2. **SUSPENDED 阶段**：Consumer 不知道 Provider 下线了（因为收不到 ZK 通知），但 Dubbo 有额外的 **健康检查/心跳机制** 来摘除故障 Provider——这个不依赖 ZK
3. **LOST 后**：Provider 在 ZK 上的临时节点会被清理，但其他 Consumer 会在重连后全量拉取，自动过滤掉故障 Provider

---

## 三、SOFA RPC 的容灾架构

### 3.1 整体架构

SOFA RPC 的 ZK 容灾与 Dubbo 有相似之处（内存缓存 + 自动重连），但在架构上更强调**分层解耦**：

```
┌──────────────────────────────────────────────────────────────┐
│                   SOFA RPC 容灾架构                            │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 订阅层: ZookeeperRegistry                               │ │
│  │                                                        │ │
│  │   职责: 连接 ZK + 订阅目录 + 注册 Watcher               │ │
│  │   断连: 记录断连状态，通知下层                           │ │
│  │   重连: 全量拉取 + 重新注册 Watcher                     │ │
│  │                                                        │ │
│  │   SOFA 使用自研的 ZkClient (基于原生 ZK 客户端封装)      │ │
│  │   而非 Curator，对断连的控制更底层                       │ │
│  └──────────────────────┬─────────────────────────────────┘ │
│                         │ 通知地址变更                        │
│                         ▼                                    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 地址持有层: AddressHolder                               │ │
│  │                                                        │ │
│  │   职责: 持有当前可用的 Provider 列表 (内存)              │ │
│  │   特点: 与 ZK 完全解耦，ZK 挂了不影响                    │ │
│  │                                                        │ │
│  │   核心属性:                                             │ │
│  │     List<ProviderInfo> currentProviders;                │ │
│  │     List<ProviderGroup> currentGroups;                  │ │
│  │                                                        │ │
│  │   核心方法:                                             │ │
│  │     getProviders() → 直接返回内存中的列表               │ │
│  │     updateProviders(List) → 全量替换                    │ │
│  │     addProvider(ProviderInfo) → 增量添加                │ │
│  │     removeProvider(ProviderInfo) → 增量删除             │ │
│  └──────────────────────┬─────────────────────────────────┘ │
│                         │ 返回地址列表                        │
│                         ▼                                    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 调用层: Consumer + 负载均衡                              │ │
│  │                                                        │ │
│  │   getProviders() → AddressHolder.getProviders()        │ │
│  │   完全感知不到 ZK 是否存活                               │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

### 3.2 核心源码: SOFA 自研 ZkClient 的连接管理

SOFA RPC 没有使用 Curator，而是封装了自己的 ZkClient：

```java
// SOFA RPC: ZkClient.java (简化版)
public class ZkClient {
    private ZooKeeper zk;
    private volatile State state = State.DISCONNECTED;
    private final List<StateListener> stateListeners = new CopyOnWriteArrayList<>();

    // 连接状态枚举
    enum State {
        DISCONNECTED,  // 未连接
        CONNECTING,    // 连接中
        CONNECTED,     // 已连接
        RECONNECTING,  // 重连中
        CLOSED         // 已关闭
    }

    public ZkClient(String connectString, int sessionTimeout) {
        connect(); // 首次连接
    }

    private void connect() {
        this.zk = new ZooKeeper(connectString, sessionTimeout, event -> {
            switch (event.getState()) {
                case SyncConnected:
                    state = State.CONNECTED;
                    // 通知所有 StateListener: 连接成功
                    for (StateListener listener : stateListeners) {
                        listener.onConnected();
                    }
                    break;

                case Disconnected:
                    state = State.RECONNECTING;
                    // ☆ 注意: SOFA 在断连时通知，但不 panic
                    //         因为 AddressHolder 仍持有有效数据
                    for (StateListener listener : stateListeners) {
                        listener.onDisconnected();
                    }
                    break;

                case Expired:
                    state = State.DISCONNECTED;
                    // Session 过期: 通知上层需要重建
                    for (StateListener listener : stateListeners) {
                        listener.onSessionExpired();
                    }
                    // 自动重连
                    reconnect();
                    break;
            }
        });
    }

    // ☆ 断连自动重连 ☆
    private void reconnect() {
        state = State.CONNECTING;
        try {
            // 关闭旧连接
            zk.close();
        } catch (Exception ignored) {}

        while (state != State.CLOSED) {
            try {
                // 重新建立 ZooKeeper 连接
                connect();
                break;
            } catch (Exception e) {
                log.warn("ZK reconnect failed, retrying...");
                Thread.sleep(3000);  // 固定 3s 重试
            }
        }
    }
}
```

### 3.3 核心源码: ZookeeperRegistry 的订阅与恢复

```java
// SOFA RPC: ZookeeperRegistry.java
public class ZookeeperRegistry extends Registry {

    private ZkClient zkClient;
    private final Map<String, List<ProviderInfoListener>> providerListeners = ...;

    @Override
    public void subscribe(ConsumerConfig config) {
        String interfaceId = config.getInterfaceId();
        String providerPath = "/sofa-rpc/" + interfaceId + "/providers";

        // 1. 全量拉取 Provider 列表
        List<String> children = zkClient.getChildren(providerPath);
        List<ProviderInfo> providers = convertToProviderInfos(children);

        // 2. 交给 AddressHolder 持有
        AddressHolder holder = getAddressHolder(interfaceId);
        holder.updateAllProviders(providers);

        // 3. 注册 Watcher: 监听子节点变化 (Provider 上线/下线)
        zkClient.watchChildChanges(providerPath, (parentPath, currentChildren) -> {
            List<ProviderInfo> newProviders = convertToProviderInfos(currentChildren);
            Set<ProviderInfo> added = diff(newProviders, holder.getAllProviders());
            Set<ProviderInfo> removed = diff(holder.getAllProviders(), newProviders);

            // 增量更新 AddressHolder
            holder.addProviders(added);
            holder.removeProviders(removed);

            // 通知所有 ProviderInfoListener
            notifyProviderListeners(interfaceId, added, removed);
        });

        // 4. 注册连接状态监听
        zkClient.addStateListener(new StateListener() {
            @Override
            public void onConnected() {
                log.info("ZK connected");
            }

            @Override
            public void onDisconnected() {
                // ☆ 关键: 断连时什么都不做!
                //   AddressHolder 继续持有当前列表
                //   消费者感知不到
                log.info("ZK disconnected, holding cached providers: {}",
                    holder.getProviderCount());
            }

            @Override
            public void onSessionExpired() {
                // Session 过期: 所有临时节点已丢失
                log.warn("ZK session expired!");
            }
        });
    }

    // ☆ 重连后的恢复: 与 Curator 不同，SOFA 需要手动处理 ☆
    private void recoverAfterReconnect() {
        // 遍历所有已订阅的服务
        for (String interfaceId : subscribedServices) {
            try {
                // 1. 全量拉取最新 Provider 列表
                String path = "/sofa-rpc/" + interfaceId + "/providers";
                List<String> children = zkClient.getChildren(path);
                List<ProviderInfo> providers = convertToProviderInfos(children);

                // 2. 更新 AddressHolder
                AddressHolder holder = getAddressHolder(interfaceId);
                holder.updateAllProviders(providers);

                // 3. 重新注册 Watcher (一次性机制!)
                zkClient.watchChildChanges(path, ...);

                log.info("Recovered subscription for {}", interfaceId);
            } catch (Exception e) {
                log.error("Recover failed for {}", interfaceId, e);
            }
        }
    }
}
```

### 3.4 SOFA RPC 的 Provider 级 Watcher 细节

SOFA RPC 有一个 Dubbo (2.7.x) 没有的细节：对每个 Provider 节点单独注册 Watcher，感知 Provider 自身的属性变化（如权重调整、warmup 状态）：

```java
// SOFA RPC 特有的: 对每个 Provider 节点单独注册 Watcher
private void watchProviderNode(String providerPath, ProviderInfo info) {
    // 监听单个 Provider 节点的数据变化
    zkClient.watchData(providerPath, (path, data) -> {
        // Provider 的权重、warmup 等属性变化了
        ProviderInfo updated = parseProviderInfo(data);
        AddressHolder holder = getAddressHolder(updated.getInterfaceId());
        holder.updateProvider(updated);  // 增量更新这一个 Provider
        // ↓
        // 负载均衡组件拿到新的权重，重新计算权重列表
    });
}
```

**这意味着**：SOFA RPC 恢复时需要重建两层 Watcher：
1. `/providers` 目录的子节点变化 Watcher
2. 每个 Provider 节点的数据变化 Watcher

恢复成本比 Dubbo 稍高（多了 N 个 Provider 级 Watcher 的重建），但粒度更细。

---

## 四、两边对比

### 4.1 架构差异一览

```
┌───────────────────────────────────────────────────────────────┐
│                     Dubbo vs SOFA RPC 对比                      │
│                                                               │
│  维度               Dubbo                     SOFA RPC        │
│  ────────────────  ────────────────────────  ──────────────── │
│  ZK客户端           Curator (Netflix开源)     自研ZkClient     │
│                     ↑封装度高                  ↑控制粒度细     │
│                                                               │
│  内存缓存            ConcurrentHashMap         AddressHolder   │
│                      (Properties)              (独立组件)      │
│                                                               │
│  文件缓存            ✓ ~/.dubbo/dubbo-          ✗ 无           │
│                      registry-*.cache                         │
│                                                               │
│  断连处理            SUSPENDED→等待           DISCONNECTED→   │
│                      Curator自动重连           手动重连        │
│                                                               │
│  Watcher重建         PathChildrenCache        手动重建两层     │
│                      (内部自动处理)            Watcher         │
│                                                               │
│  失败重试            内置 retryExecutor        依赖上层业务     │
│                      每5s自动重试             重试              │
│                                                               │
│  负载均衡感知        不依赖ZK                  不依赖ZK        │
│                      (读内存)                  (读AddressHolder)│
│                                                               │
│  ZK全挂+进程重启     文件缓存恢复              丢失Provider列表 │
│                      可继续调用                调用失败         │
└───────────────────────────────────────────────────────────────┘
```

### 4.2 最关键的差异：文件缓存

这是 Dubbo 在极端场景下的"最后一道防线"：

```
场景: ZK 集群全挂 + Consumer 进程被 OOM Killer 杀掉后重启

Dubbo:
  1. Consumer 进程重启
  2. 连接 ZK → 全失败
  3. AbstractRegistry 构造函数:
     → 读 ~/.dubbo/dubbo-registry-xxx.cache
     → 恢复 Provider 列表到内存
  4. Consumer 能用上一次的 Provider 列表继续 RPC 调用!
  5. retryExecutor 每 5s 重试连接 ZK
  6. ZK 恢复后 → 全量拉取 → 更新缓存

SOFA RPC:
  1. Consumer 进程重启
  2. 连接 ZK → 全失败
  3. AddressHolder 初始为空
  4. Consumer 没有 Provider 列表 → RPC 调用失败!

  SOFA 的设计假设: ZK 是高可用的，不会全挂
  Dubbo 的设计假设: ZK 可能全挂，需要兜底
```

### 4.3 Curator vs 自研 ZkClient

```
Curator (Dubbo 选择):
  优点:
    - 断连自动重连 (内部维护重连循环)
    - RECONNECTED 后自动重新注册所有 Watcher
    - Session 过期自动重建连接
    - 久经 Netflix 生产验证
  
  缺点:
    - 封装度高，出了问题难调试
    - 引入额外依赖 (curator-framework + curator-recipes)
    - ZK 版本兼容性问题 (Curator 3.x vs ZK 3.4.x)

自研 ZkClient (SOFA 选择):
  优点:
    - 完全可控，细粒度管理
    - 没有额外依赖
    - Session 过期/重连的每一步都可插桩
  
  缺点:
    - 需要自己处理 Watcher 重建
    - 需要自己处理重连循环
    - 断连处理逻辑由开发人员保证正确性
```

### 4.4 异常重试策略差异

```java
// ===== Dubbo: 框架级内置重试 =====
// FailbackRegistry 对所有 ZK 操作都有 try-catch-failback
register(url)     → 失败 → failedRegistered.add(url)     → 定时重试
subscribe(url)    → 失败 → failedSubscribed.put(url)      → 定时重试
unregister(url)   → 失败 → failedUnregistered.add(url)    → 定时重试

// 结论: 开发者完全不用关心 ZK 临时故障

// ===== SOFA RPC: 抛给上层处理 =====
// ZookeeperRegistry 的操作如果失败会抛出异常
subscribe(config)  → ZK 挂了 → 抛异常 → 上层业务代码决定怎么处理
register(provider) → ZK 挂了 → 抛异常 → SOFA 的 ProviderBootstrap 处理

// SOFA 的 ProviderBootstrap 中有内置重试:
public class ProviderBootstrap {
    private void doRegister() {
        // 首次注册
        try {
            registry.register(providerConfig);
        } catch (Exception e) {
            // 失败后定时重试
            scheduledExecutor.scheduleWithFixedDelay(() -> {
                try {
                    registry.register(providerConfig);
                } catch (Exception ex) {
                    log.warn("Provider register retry failed");
                }
            }, 5, 5, TimeUnit.SECONDS);
        }
    }
}
```

---

## 五、框架开发者可以学什么

### 5.1 Dubbo 的"失败不阻塞"哲学

```
写操作: register / unregister / subscribe
  → 不保证一定成功 (ZK 可能挂了)
  → 但保证不报错 (静默加入重试队列)
  → 后台默默重试直到成功

读操作: getProviders
  → 不查 ZK
  → 直接读内存缓存
  → 保证一定有结果 (即使结果是旧数据)

这就是 Dubbo 的设计精髓:
  "注册中心是辅助设施，不能影响主链路"
```

### 5.2 SOFA RPC 的"分层解耦"设计

```
ZookeeperRegistry:  只管"和 ZK 通信"
AddressHolder:      只管"持有地址列表"
Consumer + 负载均衡: 只管"从 AddressHolder 取地址做调用"

三层之间只通过接口通信:
  ZookeeperRegistry → AddressHolder: updateProviders(List)
  Consumer → AddressHolder: getProviders() → List

好处:
  - 换注册中心 (ZK → Nacos) 只需换 Registry 实现
  - AddressHolder 可以做单元测试 (mock Registry)
  - 职责清晰，各层独立演进
```

### 5.3 通用最优实践

```
1. 注册中心 = 辅助设施，不能成为主链路的单点
   - 内存缓存是必须的，不是可选的
   - Provider 列表的读取不能经过网络

2. 文件缓存是极端场景的兜底
   - ZK 全挂 + 进程重启 = 文件缓存的用武之地
   - 代价: 数据可能是旧的
   - 配合 Provider 端的健康检查来纠正

3. 区分 "短暂断连" 和 "永久断连"
   - SUSPENDED / DISCONNECTED: 不改动内存数据
   - EXPIRED / LOST: 全量重建
   - 触发时机由 sessionTimeout 控制

4. Watcher 是一次性的，重连后必须重建
   - Curator 的 PathChildrenCache 内部处理了
   - 自研客户端必须显式处理
   - 这是最常见的 bug: 重连后 Watcher 丢了，消费者收不到变更

5. 连接超时设置要合理
   - ZK sessionTimeout: Dubbo 默认 30s, SOFA 默认 30s
   - 太短: 频繁 LOST → 频繁全量拉取
   - 太长: 真正的故障感知慢
```

---

## 六、总结

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│   Dubbo 与 SOFA RPC 的 ZK 容灾策略回答了同一个问题:           │
│                                                              │
│   "ZK 是服务发现的核心吗?"                                     │
│                                                              │
│   两个框架的回答都是:                                         │
│                                                              │
│   "不是。ZK 是通知渠道，数据本身在本地。                       │
│    通知渠道断了，数据还在。                                    │
│    数据丢了，进程启动时从文件恢复 (Dubbo)                      │
│    或者等待恢复后全量拉取 (SOFA PRC)。"                        │
│                                                              │
│   底层原理回到上中下三篇的核心:                                 │
│   客户端利用 ZK 的 sessionTimeout 窗口自动重连，               │
│   利用 Curator/ZkClient 的状态回调做有状态的恢复，             │
│   利用本地缓存保证主链路不受 ZK 状态影响。                      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**阅读建议**：本文假设读者已经掌握了 ZooKeeper 的 Session 机制、Watcher 机制和断连重连原理（参考系列文章的上中下三篇）。如果对 Curator 的 `ConnectionState` 状态机或 SOFA RPC 的 `AddressHolder` 设计有疑问，欢迎在评论区讨论。
