# 定义



**分布式唯一 ID（Distributed Unique ID）** 一般指：在一个分布式系统中，由某种“ID 生成方案”产生的、**在全局范围内（跨机器、跨服务、跨时间）不重复**的标识符，用来标记请求、订单、消息、数据行等对象。 **不管有多少节点、多少服务实例、多久的时间跨度，大家一起生成 ID，也绝不会撞车。**关键特性一般包含这些

1. **全局唯一**
   - 所有节点、所有服务、所有时间段内生成的 ID **不重复**。
2. **分布式生成**
   - 多台机器各自本地生成，不依赖单机数据库自增那种集中式方案。
3. **高可用 / 无单点**
   - 某个节点/机房挂了，其他节点还是能继续发 ID。
4. **高性能**
   - QPS 很高也能扛得住（通常几十万甚至上百万 ID/s）。
5. **有序或大致有序（可选）**
   - 常见方案（例如 Snowflake）里，ID 按时间大致递增，方便做排序、分表、范围查询等。
6. **可编码业务信息（可选）**
   - 一部分位表示时间，一部分表示机房/节点，一部分表示自增序列等等。

------

# 常见唯一ID生成方案

## 1. 数据库自增 / 序列（最传统）

### 1.1 单库单表自增 ID（AUTO_INCREMENT / SEQUENCE）

**特点：**

- 简单粗暴：直接用 MySQL 自增主键、Oracle/PG 的 sequence。
- 保证同一个库内唯一，有严格递增顺序。
- 容易成为**单点瓶颈** + 不好水平扩展（多库多写就麻烦）。

**典型场景：**

- 小系统、单库单实例：
  - 内部管理系统、CRM、小型工具类项目。
- 不需要真正“分布式生成”，只要在同一个 DB 里唯一即可。
- 不适合分库分表场景

------

### 1.2 号段模式（Database Segment / 美团 Leaf-segment 思路） leaf segment思路是什么？

**核心思路：**

- **数据库只负责“发号段”，应用在内存里发“号段内的 ID”。**
- 如：DB 表中一行记录当前值 1,000,000，应用一次取“1,000,001 ~ 1,010,000”这 1 万个号段，之后这 1 万个 ID 都在本机内存自增就行。

**特点：**

- 号段分配在 DB，保证全局不重。
- 本地发 ID，无需频繁访问 DB，吞吐高。
- 存在单点 DB，但压力很小（偶尔申请号段）。

**典型场景：**

- 中大型系统，需要：
  - **关系型数据库主键**兼顾有序/性能；
  - QPS 比较高，又不想搞太复杂算法。
- 例如订单 ID、用户 ID、交易 ID 等。

------

## 2. 基于缓存 / 中间件的递增（Redis / Zookeeper 等）

### 2.1 Redis INCR / INCRBY

**特点：**

- 利用 Redis 单线程 + 原子指令 `INCR` 保证**同一个 key 上严格自增且不重复**。
- 性能比 DB 好，写在内存里。
- Redis 自己要做高可用，否则挂了要考虑恢复策略。

**典型场景：**

- 中小规模的统一 ID 服务：
  - 每次需要 ID 时，应用去 Redis 拿一个自增值，再根据自己规则拼接（比如前缀 + 时间 + 自增）。
- 不强依赖“强有序”，但要求简单可靠。

------

### 2.2 Zookeeper / Etcd 自增节点

比较少用了，简单提一句：

- 可以通过顺序节点获得一个递增数字。
- 适合**量不大**、**强一致性**要求比较高的场景（例如分布式锁/选主时的编号），不适合大规模高频 ID 生成。

------

## 3. Snowflake 及变种（Twitter 雪花算法）

**经典定义：**

> 一个 64-bit 的 long 型 ID，通过**时间戳 + 机器号 + 序列号**组合，保证在分布式环境里**大致递增且全局唯一**。

一个常见的拆分（仅示例）：

- 1bit 符号位（固定 0）
- 41bit 时间戳（毫秒级，相对某个起始时间）
- 10bit 工作机器 ID（机房 + 机器）
- 12bit 毫秒内序列号

**特点：**

- 本地计算即可，不需要中心服务 → 高性能、低延迟。
- ID 随时间大致递增，适合作为数据库主键。
- 要自己管理“时间回拨问题”（系统时钟回跳）。这是什么？如何解决？

**典型场景：**

- 高并发订单号、支付流水、日志 ID、消息 ID 等等。
- 适用于：
  - 多机房、多实例；
  - 需要有序、短 ID、long 类型存储。

------

## 4. 随机型 ID：UUID / ULID 等

### 4.1 UUID（如 UUIDv4）

**特点：**

- 128 位，通常字符串表现为 36 字符。
- 完全本地随机生成，不需要任何协调，理论上重复概率极小。
- 无序，作为 DB 自增主键时索引碎片严重。
- 字符串比 long 占空间多，性能一般。

**典型场景：**

- 对性能要求不极致、但要快速、简单的唯一标识：
  - 文件 ID、API Token、Session ID 等。
- 不适合作为高写入量关系型数据库主键（除非配合优化/顺序 UUID）。

------

### 4.2 ULID / KSUID 等“时间 + 随机”的改良型

**特点：**

- 保留随机性，保证全局唯一，同时 ID 大致按时间排序。
- 常用于替代 UUID 的场景，但提供更好的排序特性。

**典型场景：**

- 需要字符串 ID，又希望有**时间顺序**：
  - 日志事件 ID
  - 业务事件流 ID
  - 依然不适合做索引？

------

## 5. 文档型数据库内置 ID（MongoDB ObjectId 等）

**Mongo ObjectId 举例：**

- 包含时间戳 + 机器 + 进程 + 自增序列。
- Mongo 默认的 `_id` 就是它，天然分布式唯一。

**典型场景：**

- 使用 Mongo、ES 等文档存储，本身就有内置 ID，不想额外搞一套 ID 生成服务。
- 对其他系统暴露时，也可以直接复用这个 ID（前提是位数长度能接受）。

------

## 6. 业务自定义规则（时间 + 业务码 + 自增/随机）

常见套路：**时间 + 业务前缀 + 自增 / 随机数**，例如：

- 订单号：`20251209 01 00000123`
  - 时间戳：20251209
  - 业务线：01
  - 日内流水号：00000123

**实现方式**可以用：

- 号段模式 / Redis 自增 / Snowflake 之类当“核心流水号”；
- 然后再拼成业务可读的 ID。

**典型场景：**

- 人类可读、有业务含义的 ID：
  - 订单号、工单号、发票号、物流单号等。

------

## 7. 分布式唯一 ID 的典型使用场景小总结

按“用 ID 来干啥”分几类：

1. **数据库主键**
   - 追求：有序、短（long）、高写入性能
   - 常用方案：号段模式、Snowflake、Leaf、Redis+时间。
2. **业务编号（订单号、交易号等）**
   - 追求：可读、有业务信息、可追踪
   - 常用方案：`时间 + 业务码 + 自增/随机`，内部自增 ID 采用号段/Snowflake/Redis 等。
3. **日志 & 请求追踪（Trace ID / Span ID）**
   - 追求：全局唯一、任意服务快速生成、不需要强有序
   - 常用方案：UUID、改良的时间+随机 ID（ULID/KSUID）、TraceID 规范。
4. **消息队列 / 事件 ID**
   - 追求：唯一 + 大致有序，有利于重放和排查
   - 常用方案：Snowflake / ULID / MQ 自带 ID。
5. **安全敏感 Token（API Token、Reset Token 等）**
   - 追求：难猜、不可预测，唯一
   - 常用方案：强随机数 / UUIDv4 / 安全随机字节编码。

------

## 如果你要选方案，可以这样想：

- 你现在最关心的是哪一项？
  - **数据库插入性能 + 主键有序** → Snowflake / 号段。
  - **简单好用，系统不大** → DB 自增 / Redis INCR。
  - **无需有序、只是唯一** → UUID / 随机 ID。
  - **要人类可读、能看出时间/业务** → 雪花或号段生成“内部 ID”，再包一层业务格式。





# 号段模式

> **数据库存“最大号段值”，应用批量取号段到内存，在内存里自增发号。**

这样既保证全局唯一（因为号段不重叠）；又不需要每次都访问数据库（性能极高）。

------

### 举个形象例子

假设数据库里维护了一张表：

| biz_tag | max_id | step | update_time |
| ------- | ------ | ---- | ----------- |
| order   | 100000 | 1000 | 2025-12-09  |

意思是：

- 当前 `order` 业务已经分配到了 ID = 100000；
- 每次“号段”大小是 1000；
- 下一个号段是 [100001, 101000]。

一个应用启动时，会：

1. 查询当前号段信息；
2. 把 `max_id` 更新为 `max_id + step`（即 101000）；（这个过程是什么样的？事务吗？还是借助锁，mysql锁机制）
3. 拿到这段区间放进本地内存；
4. 后续生成 ID 时，在内存中从 100001 开始递增，直到 101000 用完；
5. 用完后，再去数据库取下一个号段。

------

## 二、为什么这么设计

- 如果用数据库自增主键，每次生成 ID 都要执行 SQL，QPS 低；
- 号段模式把“发号”拆为两层：
  - **号段分配层**（访问数据库，低频操作）
  - **号段使用层**（本地内存，极高速）

只要控制号段的大小、提前预取，就能实现：

- 每秒生成几十万甚至上百万 ID；
- 仍然是全局唯一；
- 不依赖外部服务（因为 DB 只是协调号段边界）。

------

## 三、数据库表设计

美团 Leaf 的号段表定义：

```sql
CREATE TABLE `leaf_alloc` (
  `biz_tag` varchar(128) NOT NULL DEFAULT '',  -- 业务标识（比如 order、user、ticket）
  `max_id` bigint(20) NOT NULL DEFAULT '1',    -- 当前最大 ID（已分配出去的上限）
  `step` int(11) NOT NULL,                     -- 每次取号段的步长
  `description` varchar(256) DEFAULT NULL,
  `update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`biz_tag`)
);
```

**一行记录对应一个“业务类型”的发号器**。

------

## 四、取号段的数据库操作（源码级）

Leaf Segment 模式的核心类是：

`com.sankuai.inf.leaf.segment.dao.IDAllocDao`
 主要方法：

```java
public SegmentBuffer updateMaxIdAndGetLeafAlloc(String tag)
```

### 操作逻辑（源码伪流程）

1. **更新号段上限：**

```sql
UPDATE leaf_alloc SET max_id = max_id + step WHERE biz_tag = #{tag}
```

1. **查询新值：**

```sql
SELECT biz_tag, max_id, step FROM leaf_alloc WHERE biz_tag = #{tag}
```

1. **返回这一段的范围：**
   - 旧号段区间：`[old_max_id - step + 1, old_max_id]`
   - 新号段区间：`[max_id - step + 1, max_id]`

应用得到这个区间后，就可以在内存中自增。

------

## 五、Leaf 的双缓冲机制（源码级重点）

Leaf 号段模式真正的亮点在于“**双 buffer 异步预加载**”。

### 为什么要双缓冲？

如果一个号段快用完了，再去数据库取下一个号段，会有停顿（冷却时间）。
 为了避免发号中断，Leaf 在后台线程里**提前异步加载下一个号段**。

------

### 数据结构：`SegmentBuffer`

源码位置：`com.sankuai.inf.leaf.segment.model.SegmentBuffer`

```java
public class SegmentBuffer {
    private String key;                 // 业务标识
    private Segment[] segments;         // 双buffer：[0]当前号段, [1]预加载号段
    private volatile boolean nextReady; // 下一个号段是否准备好
    private volatile boolean initOk;    // 是否初始化完成
    private int currentPos;             // 当前使用哪个 buffer
    private AtomicBoolean threadRunning;// 是否有线程在加载下一个号段
    private int step;                   // 步长
}
```

其中：

- `segments[0]`：当前正在使用的号段；
- `segments[1]`：下一个号段；
- 当当前号段剩余比例 < 10% 时，会异步线程预取下一段。

------

### 加载号段的核心逻辑：

```
com.sankuai.inf.leaf.segment.service.SegmentService#getNextSegmentFromDB()
```

大致逻辑：

```java
public Segment getNextSegmentFromDB(String key, SegmentBuffer buffer) {
    // 1. 从 DB 更新 max_id
    LeafAlloc leafAlloc = idAllocDao.updateMaxIdAndGetLeafAlloc(key);
    
    long newMaxId = leafAlloc.getMaxId();
    long step = leafAlloc.getStep();
    
    // 2. 设置号段范围
    Segment segment = buffer.getCurrent(); // 或者 buffer.getNext()
    segment.setValue(newMaxId - step, newMaxId);
    
    return segment;
}
```

------

### 发号逻辑（核心算法）

`Segment.getAndIncrement()`：

```java
public long getAndIncrement() {
    return value.getAndIncrement(); // AtomicLong，自增返回旧值
}
```

结合 `SegmentBuffer`：

```java
public long getId(String key) {
    SegmentBuffer buffer = cache.get(key);
    Segment segment = buffer.getCurrent();

    long id = segment.getAndIncrement();
    if (id < segment.getMax()) {
        return id;
    } else {
        // 当前段用完，切换到下一个
        synchronized(buffer) {
            if (buffer.isNextReady()) {
                buffer.switchPos(); // 切换
                buffer.setNextReady(false);
            } else {
                // 等下一段加载完或阻塞取新段
                ...
            }
        }
        return getId(key);
    }
}
```

------

### 异步预加载逻辑

触发条件：
 当前号段剩余 < 10%，且没有线程正在加载。

```java
if (!buffer.isNextReady()
    && segment.getIdle() < 0.1 * buffer.getStep()
    && buffer.getThreadRunning().compareAndSet(false, true)) {
    threadPool.execute(() -> {
        Segment next = buffer.getNext();
        updateSegmentFromDB(key, next);
        buffer.setNextReady(true);
        buffer.getThreadRunning().set(false);
    });
}
```

------

## 六、Leaf-Segment 的核心优势

| 特性     | 表现                                                   |
| -------- | ------------------------------------------------------ |
| 高性能   | 号段在内存中递增发号，无锁或低锁，自增速度可达数十万/s |
| 高可用   | 即使 DB 挂了，已有号段还能用完（暂时不影响服务）       |
| 有序     | 每个号段内递增，整体大致按时间递增                     |
| 简单可靠 | DB 只存号段状态，逻辑清晰                              |
| 可调节   | 动态调整 `step` 以适应不同负载                         |

------

## 七、潜在问题与优化策略

| 问题           | 解决方案                                       |
| -------------- | ---------------------------------------------- |
| DB 是单点      | 可用双主热备或用高可用 DB                      |
| 时钟问题（无） | 本地自增，不依赖时间戳                         |
| 动态步长调整   | Leaf 支持动态调整步长，号段消耗快则增大 step   |
| 内存丢失       | 应用重启只会损失当前段未使用完的少量号，能容忍 |

------

## 八、和 Snowflake 的对比

| 特性       | 号段模式              | Snowflake        |
| ---------- | --------------------- | ---------------- |
| 依赖       | 数据库（轻度）        | 本地时钟         |
| 有序性     | 强递增                | 时间有序         |
| 故障恢复   | 重启不影响            | 时间回拨要注意   |
| 实现复杂度 | 中等                  | 高               |
| 性能       | 高                    | 更高（纯内存）   |
| 可控性     | step 可调节，人工可控 | 依赖算法位数划分 |

------

## 九、小结

> **号段模式（Segment ID 生成）** 的核心原理是：
>
> 1. 数据库存号段（持久化边界）；
> 2. 应用分配号段（本地缓存、自增使用）；
> 3. 通过双 buffer 异步预取避免停顿；
> 4. 结合 `AtomicLong` 实现高并发线程安全。

**Leaf Segment 模式源码主链路：**

```
LeafSegmentService
  ├── SegmentIDGenImpl
  │     ├── getId()                    ← 主入口
  │     ├── updateSegmentFromDB()      ← 取号段逻辑
  │     └── asyncLoadNextSegment()     ← 异步预加载
  ├── SegmentBuffer
  │     ├── current / next Segment
  │     └── threadRunning, nextReady
  ├── Segment
  │     ├── value: AtomicLong
  │     ├── max
  │     └── getAndIncrement()
  └── IDAllocDao
        ├── updateMaxIdAndGetLeafAlloc()
        └── getAllLeafAllocs()
```

------

✅ **一句总结：**

> “号段模式”本质是：
>  **用数据库协调号段边界 + 应用本地内存递增发号 + 异步预取缓冲**
>  既保留分布式唯一性，又实现极高性能和稳定性，是工程上最常用的 ID 生成方案之一。

------

如果你希望我进一步带你看 **Leaf 源码里的核心类调用链（带方法名 + 行号 + 调用关系图）**，我可以帮你画一张完整流程图（从 HTTP 请求到最终发号返回 ID）。要不要我帮你画这个？