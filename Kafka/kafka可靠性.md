# 定义

Kafka 的可靠性就是它在面对 broker 宕机、网络故障等异常时，仍然能够保证消息被持久化到副本，并按至少一次（或配置的 exactly-once）语义被消费者读取，不会丢失或悄悄消失的能力。

# Producer端

### 1.1 ACK 机制：消息写入成功

关键配置：`acks`

- `acks=0`：
   Producer 发出去就当成功，**完全不可靠**，Broker 收没收到都不知道。
- `acks=1`（默认）：
   **Leader 副本写入本地日志后**就给 Producer 回 ACK。
  - Leader 挂掉但还没同步给 follower 的这部分数据可能丢。
- `acks=-1` 或 `acks=all`：
   只有当 **Leader + ISR 中的所有 follower 都写成功** 才应答。
  - 配合合适的 `replication.factor` + `min.insync.replicas`，可以做到“单机宕机不丢”。

> **可靠性建议：生产环境一般至少 acks=all。**

### 1.2 重试 + 幂等：防止“网络抖了一下就丢”

网络超时 / 临时故障时，Producer 可能收不到 ACK，这时候重试就很关键：

- `retries`：失败后重试次数（新版本合并到 `delivery.timeout.ms` 控制整体超时时间）。
- **问题**：重试有可能导致同一条消息写入多次（重复）。

为了解决“重试导致重复”这个问题，引入了**幂等生产者**：

- `enable.idempotence=true`
- Kafka 给每个 Producer 分配一个 PID（Producer Id），并给每个 `(PID, partition)` 的消息附加单调递增的 sequence number。
- Broker 端对同一个 `(PID, partition)`，只接受**第一次出现**的 sequence，重复的直接丢弃。
- 这样即使网络抖动 + 重试，也不会产生重复消息（在单个分区的维度上）。

> 搭配使用：
>  `enable.idempotence=true` 会自动强制：`acks=all`、`retries` 足够大、`max.in.flight.requests.per.connection` 限制在安全范围，以保证顺序和去重。

------

## 二、Broker 端：写入后的“保命机制”

### 2.1 顺序写 + 磁盘持久化

Kafka 的每个分区就是一个 **commit log**：

- 消息按顺序 append 到日志文件（顺序写，非常快）。
- 依赖**操作系统 page cache + fsync 策略**，保证持久性：
  - 即使 Broker 进程崩溃，已写入磁盘的数据可以从日志中恢复。
- 同时提供参数控制是否需要更“强”的刷盘策略（性能 vs 持久性之间的 tradeoff）。

### 2.2 多副本复制：单机挂了不丢

每个分区都有：

- 1 个 **Leader**
- 若干个 **Follower**（副本），组成 ISR（In-Sync Replicas，同步副本集合）

写流程：

1. Producer 只写 Leader。
2. Leader 写入本地日志。
3. Follower 异步从 Leader 拉取数据，写入本地日志。
4. 当 ISR 中足够多 follower 同步完成，Leader 才认为这条消息“已复制”。

可靠相关配置：

- `replication.factor`（topic 副本数，常见设置为 3）
- `min.insync.replicas`（最少有多少副本必须“在 ISR 里”且写成功，才能对 `acks=all` 的请求返回成功）

比如：

- `replication.factor=3`
- `min.insync.replicas=2`
- Producer 使用 `acks=all`

那么：
 **至少有 2 个副本写成功**才会返回成功。
 如果此时挂掉一个 Broker，仍然还有至少一个副本完整持有数据，不丢。

### 2.3 Leader 选举策略：尽量只用“没丢数据的那位”

参数：`unclean.leader.election.enable`

- 若为 `false`（推荐生产环境）：
   **发生故障时，只在 ISR 中选新的 Leader**。
  - 即只选择“数据是最新的那几个”，避免选到落后的副本造成数据回滚/丢失。
- 若为 `true`：
   在极端情况可以“牺牲数据一致性，换取可用性”，允许选一个落后很多的 follower 为 Leader，可能丢已确认的数据。

> 为了可靠性：**一般都关闭 unclean leader election**。

### 2.4 内部主题的可靠性

像：

- `__consumer_offsets`（保存消费位移）
- `__transaction_state`（事务状态）

这些内部 topic 通常也设置为：

- `replication.factor` 较高（如 3）
- `min.insync.replicas` 合理配置

保证**连 offset / 事务状态都尽量不丢**。

------

## 三、Consumer 端：怎么“读得对”，既不丢也不乱

### 3.1 Offset 管理：读到哪儿、处理到哪儿？

Kafka Consumer 本质上自己维护一个指针：offset。
 可靠性关键点是：**offset 何时提交？提交到哪儿？**

默认是提交到 Kafka 的内部 topic：`__consumer_offsets`。

三种常见语义：

1. **至少一次（at-least-once）**（Kafka 默认推荐）
   - **先处理消息，再提交 offset**：
     - 消费者拉到消息 → 处理业务 → 确认成功 → 提交 offset。
   - 故障场景：
     - 如果处理成功但 offset 提交前挂了，重启后会从旧 offset 重新消费这条消息 → **消息不丢，但可能重复**。
   - 所以需要业务逻辑具备 **幂等性** 或做去重。
2. **至多一次（at-most-once）**
   - **先提交 offset，再处理**：
     - 优点：不会重复消费。
     - 缺点：如果提交后挂了，业务没处理成功 → **消息会丢**。
   - 一般不建议用于对可靠性要求高的业务。
3. **精确一次（exactly-once）**（Kafka 从 0.11 开始支持 EOS）
   - 需要结合 **幂等生产者 + 事务性写入 + 消费端事务提交 offset**。
   - 下面单独讲。

### 3.2 Rebalance 机制与可靠性

- 同一个消费组里的多个 consumer 会“分片”消费不同分区。
- 当消费者上下线、分区增加时，会触发 rebalance：
  - Kafka 会把分区重新分配给不同的 Consumer。
- offset 存在 Kafka 里，可由新 Consumer 继续接着之前的位置消费，不会从头开始，也减少了丢消息的风险。

------

## 四、Exactly Once：生产→处理→再生产的全链路不重不丢

Kafka 的 **EOS（Exactly-Once Semantics）** 主要用于“流式处理 / ETL：topicA → 处理 → topicB” 这种场景，核心是：

> “要么结果消息和 offset 一起成功，要么都不成功，不会出现只处理不提交 offset、或者只提交 offset 不处理成功。”

关键技术点：

1. **幂等生产者（Idempotent Producer）**
   - 前面说过，保证一个分区内 Producer 重试不会导致重复数据。
2. **事务性 Producer（Transactional Producer）**
   - Producer 获取一个 `transactional.id`，在多个分区上开启事务：
     - 在事务中往多个 topic/partition 写消息；
     - 同时往 `__consumer_offsets` 写“提交的 offset”；
     - 最后 `commitTransaction()`：
        → 要么所有写入 + offset 提交都一起生效，
        → 要么在失败时回滚，不对外可见。
   - Consumer 端设置 `isolation.level=read_committed`：
     - 只会读取已经提交事务的数据，跳过未提交或中途 abort 的写入。
3. **配合流框架（如 Kafka Streams）**
   - 默认就可以利用 EOS，做到：
      “从 input topic 读 → 处理 → 写到 output topic + 更新 offset” 在同一个事务中完成，实现 end-to-end exactly once。

> 总结：
>  Kafka 通过
>  **幂等 Producer + 事务写消息 + 事务提交 offset + read_committed 消费**
>  实现了真正的「端到端精确一次」语义。

------

## 五、把全链路串起来，再说一遍“可靠性”的路径

从生产到消费，全链路上的“保命点”：

1. **Producer 发消息不轻易丢**
   - `acks=all`：确认至少多个副本持久化才算成功。
   - `enable.idempotence=true`：重试不产生重复（分区内）。
   - `retries` + 合理的超时控制：网络抖动不致于直接失败。
2. **Broker 存消息不轻易丢**
   - 顺序写日志 + 磁盘持久化，进程挂了可从日志恢复。
   - 多副本：`replication.factor>=3`，leader 宕机还有 follower 可顶上。
   - ISR + `min.insync.replicas` + `unclean.leader.election=false`：
      防止在数据不同步时选了落后的副本当 leader 导致回滚/丢。
3. **Consumer 读消息不轻易丢 & 不乱**
   - 通过合适的 offset 提交时机实现 **至少一次** 或配合事务实现 **精确一次**。
   - offset 持久化在 Kafka 内部 topic，多副本保护 offset 自身不丢。
   - rebalance 机制 + group.id：
      consumer 挂了，新 consumer 从上次的 offset 接着读。
4. **需要时，进一步用事务串起来**
   - 生产 → 处理 → 再生产/写 DB（如果也参与分布式事务） 可以组合成“统一事务”，达到真正的 end-to-end exactly once。

------

如果你愿意，可以告诉我你现在是更关心：

- “仅仅保证不丢，能接受重复”（典型业务），
   还是
- “不能重复，必须精确一次”（比如计费）

我可以按这两种场景，给你一套具体的 Kafka 参数和代码层面的实践建议。





# 实际场景



现实里大家不是“看文档随便配”，而是结合**可靠性要求 + 成本**来折中。下面我用**几个真实/典型的企业做法**来说明（会分：自建集群 vs 云托管）。

> 先说结论：
>  绝大多数认真用 Kafka 的公司（尤其是大厂 / 金融）**生产写入一定是 acks=all，replication.factor>=3，min.insync.replicas>=2**，差别主要在：
>
> - 是否开启幂等 / 事务
> - 是否完全用云厂商托管

------

## 一、自建集群场景：传统大厂 / 互联网公司

### 1）LinkedIn（Kafka 的“亲爹”）

LinkedIn 自己的工程文章里说过，他们的 Kafka 集群：

- 副本数一般 **3 副本** 起步，用多机房部署；
- 写入使用 **acks=all**，靠副本保证可靠性；
- 严格要求 `min.insync.replicas=2`，保证**至少两个副本写成功才算成功**；
- 关闭 `unclean.leader.election`，宁可短暂不可用也不接受数据回滚。

> 这套配置实际上已经变成“业界经典模板”：
>  `acks=all`, `replication.factor=3`, `min.insync.replicas=2`, `unclean.leader.election=false`。

------

### 2）金融类/银行（国内外都类似）

金融行业对“**不丢消息**”极度敏感，常见做法（你在很多银行/券商的中间件技术分享里都能看到类似配置）：

- 生产者：
  - `acks=all`
  - `enable.idempotence=true`
  - `retries` 很大（或用 `delivery.timeout.ms` 上限控制）
- Topic：
  - `replication.factor=3`（有的核心业务甚至 5）
  - `min.insync.replicas=2`
  - `unclean.leader.election=false`
- Consumer：
  - 坚持 **“至少一次” + 业务端幂等**，
  - 或在重要链路上使用 **事务性 Producer + read_committed** 做到“精确一次”。

很多银行用的是 **自建 Kafka 集群 + 机房内多机房部署**，因为合规要求数据不能随便上公有云；
 部分会用云厂商提供的**专有云 / 金融云 Kafka 服务**（本质还是托管版 Kafka，但物理机和网络是隔离的）。

------

### 3）大互联网公司（举个典型：类似滴滴、美团、字节这种）

这些公司公开技术分享里基本也都提到类似配置：

- 日志 / 埋点 / 行为数据这类“可少量容忍丢失”的：
  - 一般也会用 `acks=all` + `replication.factor=3`
  - 但 `min.insync.replicas` 有时会是 1 或 2，根据业务重要性调；
- 订单、支付、风控等强一致业务：
  - 一样是 `acks=all`，有的强制 `min.insync.replicas=2`；
  - 部分链路启用幂等 + 事务，或者在下游 DB 层做幂等去重。

**自建还是云？**

- 早期：几乎全是**自建集群**（尤其日 PV 非常大的公司，一般认为自己管更可控、更便宜）；
- 现在：有些公司新业务、边缘业务会直接用**云上的 Kafka 托管服务**（如 Confluent Cloud / AWS MSK / 阿里云消息队列 Kafka 版等），核心业务仍然自建或专有云。

------

## 二、云厂商托管 Kafka 的典型配置

很多公司现在会直接用云服务，然后按照厂商推荐的“高可用”模板配置。

### 1）Confluent Cloud / Confluent Platform

Confluent 的官方最佳实践里：

- 推荐生产环境：
  - `acks=all`
  - `retries` 较大
  - `enable.idempotence=true`（幂等 Producer）
- Topic：
  - `replication.factor=3` 起步
  - 建议 `min.insync.replicas=2`
  - 严格关闭 `unclean.leader.election`，强调**数据可靠优先**。

Confluent Cloud 是**完全托管**：

- Broker、ZK / KRaft、监控、扩容、跨区复制都由 Confluent 管理；
- 使用方主要关注：topic 的副本数、分区数、以及 producer/consumer 的参数。

很多国外公司（SaaS、金融科技、游戏）会直接用 Confluent Cloud 来减少运维压力。

------

### 2）AWS MSK（Managed Streaming for Apache Kafka）

在 AWS MSK 的官方文档和架构推荐里：

- 集群默认就是多 AZ + 多副本；
- 官方示例生产者代码基本都是：
  - `acks=all`
  - `enable.idempotence=true`（推荐开启）
- 建议 topic：
  - `replication.factor=3`
  - 对重要 topic 设置 `min.insync.replicas=2`。

很多在 AWS 上跑的公司（比如北美一些互联网公司、中小企业）会这样用：

- 生产者：`acks=all`，重试打开，幂等打开；
- Consumer：业务级幂等处理 + Kafka offset 提交；
- Kafka 运维由 AWS MSK 负责，自己主要管监控和参数调优。

------

### 3）阿里云 Kafka（消息队列 Kafka 版）

在阿里云的官方最佳实践 & Demo 里，也类似：

- 官方推荐配置：
  - 写入：`acks=all`（有些样例写的是 `-1`）；
  - 建议开启幂等生产者，避免重试产生重复；
- Topic：
  - 对高可靠业务推荐 3 副本；
  - 云产品底层默认多副本和多可用区。

很多国内公司小团队 / 新项目会直接用：

- 阿里云 Kafka + Spring Cloud Stream / Spring Kafka；
- Producer 配 `acks=all`，Consumer 走“至少一次”语义；
- 核心金融用户则会在其“金融云/专有云”里使用类似配置。

------

## 三、实际“分级”做法：不是所有业务都一样

真实公司里，一般会按业务重要性分级配置，而不是“一刀切”：

### 1）低价值日志 / 埋点

- 写入是海量但允许少量丢失：
  - 仍然多数使用 `acks=all`（因为现在机器/网络成本相对可接受）；
  - 也有极端追求写入吞吐的场景可能用 `acks=1`（但大厂越来越少这样干了）。
- `replication.factor=2` 或 `3`；
- `min.insync.replicas=1` 或 `2`；
- 不启用幂等 / 事务，消费端大多是离线统计，不怕重复。

### 2）普通业务事件（用户操作事件、业务流水）

- 配置基本统一成“可靠标准版”：
  - `acks=all`
  - `replication.factor=3`
  - `min.insync.replicas=2`
  - `unclean.leader.election=false`
- Consumer：
  - 使用“至少一次语义”，业务侧做幂等（比如根据业务 id 去重）。

### 3）强一致业务（订单、支付、计费、风控）

- Producer：
  - `acks=all`
  - `enable.idempotence=true`
  - `retries` 较大
- Topic：
  - `replication.factor=3` 或 5
  - `min.insync.replicas=2` 或 3
- 有些会：
  - 使用 **事务性 Producer**（`transactional.id`）+ Kafka Streams / 事务消费；
  - 或者通过**下游数据库唯一键 + 去重表**达到幂等效果。

------

## 四、如果你在公司里落地，怎么“抄作业”

给你一份 **可直接照抄 + 再按业务微调** 的建议配置（非常接近大厂常态）：

### 生产者（重要业务）

```properties
acks=all
enable.idempotence=true
retries=2147483647   # 或由 delivery.timeout.ms 控制
max.in.flight.requests.per.connection=5  # 或 1（要求严格顺序时）
linger.ms=5~20       # 视吞吐和延迟需求调
batch.size=32KB~128KB
```

### Topic（重要业务）

```text
replication.factor = 3
min.insync.replicas = 2
unclean.leader.election.enable = false
```

### Consumer

- 使用“至少一次语义”：
  - 先处理再提交 offset；
  - 业务层用唯一键/幂等逻辑抵消重复。
- 若你走 EOS 路线：
  - `isolation.level=read_committed`
  - 处理逻辑整合进 Kafka Streams 或自写事务性逻辑。

------

如果你愿意，可以告诉我你们业务的场景（比如：订单系统 / 日志埋点 / 风控 / 实时数仓），以及是**自建 Kafka 还是用某家云服务**，我可以帮你把上面这些“通用大厂配置”收缩成一份更针对你业务的推荐参数表。