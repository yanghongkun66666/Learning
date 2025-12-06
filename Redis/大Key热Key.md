# 定义

**大 Key / 热 Key 都不是绝对数值，而是：在某个系统 / 某层里，单个 Key 在“体积 / 访问量 / 资源占用”上远远偏离平均水平的那个点。不同场景，指标和阈值都不一样。**

------

## 一、在 Redis 里的大 Key / 热 Key（最典型场景）

### 1.1 单机 Redis 场景

- **大 Key（Big Key）**
  - 关注的是：**单个 key 的 value 体积 / 元素数是否过大**
  - 常见经验阈值（不是硬标准）：
    - String：> 1MB、5MB 就可以认为是大 key
    - Hash/List/Set/ZSet：元素数 > 几万 / 十几万
  - 关注原因：
    - `GET / SET / HGETALL / LRANGE / SMEMBERS` 等操作一次性搬很多数据
    - `DEL`、扫描、持久化（RDB / AOF）对这个 key 都会“卡一下”
- **热 Key（Hot Key）**
  - 关注的是：**QPS / 命中次数是否远高于其他 key**
  - 常见定义方式：
    - 单 key QPS 占总 QPS 的 10% / 20% 以上
    - 单 key 每秒几千、上万访问，而普通 key 每秒只有个位数
  - 关注原因：
    - 单线程 Redis：大量请求都堆到这一个 key → 这个实例变成瓶颈
    - 这个 key 若失效 / 击穿，后面一大堆请求直打 DB

### 1.2 Redis 集群 / 分片场景

概念不变，但**影响会放大**：

- **大 Key**：
  - 集群重分片 / 迁移槽位时，这个 key 会被拷贝、迁移，导致迁移超慢，甚至阻塞。
  - 所以有的公司会更严格一点，比如：
    - 单个 key > 512KB 就要报警 / 审查
- **热 Key**：
  - 某个 key 集中在某一个分片，**把这个分片的 CPU / 网络 / QPS 全打满**，整个集群看起来是“平均负载 OK，但某个节点爆了”。
  - 典型：爆款商品库存 `sku:123:stock` 全在一个分片上

> ✅ 小结（Redis 场景）：
>
> - 大 key = 单个 key 太胖 → 单次操作 & 迁移成本很高
> - 热 key = 单个 key 太火 → 单节点 QPS 压力过高

------

## 二、在数据库（MySQL / NoSQL）里的“大 key / 热点”

在 DB 世界里，一般不叫 key，而叫：**热点行 / 热点索引 / 热点分区**，但是本质是一样的。

### 2.1 MySQL

- **“大 key” 对应：大 row / 大 field / 大 blob**
  - 一行里有超大的 TEXT/BLOB/JSON 字段，几百 KB / MB
  - 单次查询 / 更新，要传输和解析特别多数据
  - 对缓冲池 / 网络 IO 影响明显
- **“热 key” 对应：热点行 / 热点索引值 / 热点分区**
  - 某一行被频繁更新（比如计数表里 `id=1` 的那一行）
  - 某个索引值（比如某个 userId）被频繁访问
  - 某一分区 / 某一页（Page）被持续读写 → 争锁 / cache 竞争
  - 典型例子：
    - 排行榜表里只有一行总分记录被疯狂更新
    - 多人同时更新同一账户余额 → 产生行锁争用

### 2.2 NoSQL（MongoDB、ES 之类）

- **大文档（Big Document）**：
  - 单个 document 太大（几十 KB / 上 MB）
  - 影响：加载慢、索引维护重
- **热点文档 / 热点 shard**：
  - 某个 document 或某个 shard 的查询/写入远远多于其他
  - ES 里叫：**热点 Shard**（Hot Shard）

> ✅ 小结（数据库场景）：
>
> - 大：一行 / 一条文档太大
> - 热：某行 / 某索引 / 某分区被高频访问或写入，造成锁竞争或负载不均

------

## 三、在缓存体系 / 本地缓存里的大 Key / 热 Key

比如：Caffeine / Guava / Ehcache 等本地缓存，或者多级缓存系统。

- **大 Key**
  - 单个 key 的 value 占内存太多（比如几十 MB 图片、巨长 JSON）
  - 占用大量堆内存，影响 GC，缓存容量利用率变差
- **热 Key**
  - 某 key 的访问次数远高于其他：
    - 命中统计里，top 1 key > 50% 的命中数
  - 在本地缓存里，一样会导致：
    - 一直常驻内存（可能是好事）
    - 并发更新时，如果没锁好，也会竞争

------

## 四、在网关 / HTTP 层（Nginx / API Gateway）

这里我们通常不说 key，而说：**热点 URL / 热点接口 / 热点用户**，但概念是一样的。

- **大 Key 类比：大响应 / 大包**
  - 某些接口返回体特别大（几十 MB 的导出文件 / 一次查全量数据）
  - 单个请求耗时 / 带宽都特别高
- **热 Key 类比：热 URL / 热 API**
  - 某个接口被大量调用：
    - `/flashSale/buy`
    - `/hotFeed`
  - 某个用户 / IP / appKey 产生了绝大多数流量

这里“热 key”的处理通常就变成了：

- **接口限流**（QPS 限制）
- **按照 IP / 用户 / appKey 限流**
- **CDN 缓存热点静态资源**

------

## 五、在消息系统（Kafka / MQ）里的大 / 热

在 Kafka 这种系统里：

- **大 “Key / Message”**
  - 单条消息体积太大（几百 KB / MB）
  - 影响：
    - 单次发送 / 拉取耗时变长
    - Page Cache / 网络 IO 压力明显
- **热 Partition / 热 Topic**
  - 某个 partition 的 QPS / 流量远高于其他 partition
  - 导致：
    - 那个 broker / 那个 partition 的 I/O、CPU 飙高
  - 常见原因：
    - 分区键 key 设计不均匀，比如 orderId 模 4，结果 80% 订单集中在某几个号段

**这里的“热 key”可以理解为“热点分区的 key 值分布非常倾斜”。**

------

## 六、业务维度的“大 / 热”：热商品、热用户、热房源…

在业务侧，你可能会听到：

- 热商品：某 SKU 被疯狂浏览 / 下单
- 热用户：某头部主播、头部创作者
- 热房源：某几套房点击量极高

这些“热”最终在技术层面会映射成：

- Redis 热 key（缓存层）
- DB 热行 / 热索引
- API 热接口 / 热路由
- MQ 热分区

而**大 key**在业务侧也有表现：

- 一个用户画像 JSON 里塞了几千个特征 → 变成 Redis 大 key + 大 row
- 一个订单的扩展字段乱塞东西 → 变成胖行





# 大key 热key处理方案



这个问题可以直接当“八股 + 项目经验”讲了，我给你按**组件 / 场景拆开**，说说大厂常见怎么搞 Big Key / Hot Key，每块都带一点“可以在面试里讲出来的话术”。

------

## 一、Redis 场景

### 1. Big Key 的常见处理方案

**问题回顾：**

- 单个 key 很胖：value 很大或集合成员超多；
- 操作它会**阻塞 Redis 单线程**、拖慢持久化、迁移。

**常见做法：**

1. **业务层“拆 key” / “拆集合”**

   - 把一个大集合拆成多个小集合，比如：

     ```text
     big:set:{uid}  →  big:set:{uid}:0, big:set:{uid}:1, ... big:set:{uid}:N-1
     ```

   - 按哈希 / 按时间 / 按业务维度拆，比如：

     - `fans:{uid}` 拆成 `fans:{uid}:0..9`，按被粉丝 id%10 分桶；
     - 历史数据按日期拆：`history:2024-01:*`。

   **目标**：单个 key 元素数可控，增删查可以分批。

2. **分页 / 局部读取，禁止全量操作**

   - 避免 `SMEMBERS / HGETALL / LRANGE 0 -1` 这种全量拉；
   - 改成**分页 scan** / limit 读取：
     - `SSCAN / HSCAN / ZSCAN`；
     - List 用 `LRANGE key offset limit` 分页。
   - 有些公司直接规定：**不允许线上用 HGETALL / SMEMBERS 查大结构**。

3. **异步删除 / 分段删除**

   - Redis 4 开始有 `UNLINK`，改用异步删除大 key，避免一次 `DEL` 卡线程；
   - 特别大的集合还会在业务层做“逻辑删除 + 后台慢慢分批删成员”。

4. **容量/大小监控 + 预警**

   - 定期跑脚本扫描 key 的大小（`MEMORY USAGE`、采样）；
   - 一旦某个 key 体积 / 元素数超阈值，就报警 → 查业务逻辑。

5. **数据迁移出 Redis**

   - 对那种**天然就很大的 value**（大 JSON、大列表）：
     - 提前评估：是不是应该放到 MySQL / ES / 对象存储（OSS）；
     - Redis 只存索引 ID、摘要、TopN 等。

> 一句话：**大 key 的主方向是“拆 + 限制 + 监控 + 部分挪走”。**

------

### 2. Hot Key 的常见处理方案

**问题回顾：**

- 一个 key 的 QPS 特别高（抢购库存、热榜、配置）；
- 集中在某个分片 / 实例，导致这台 Redis 顶不住；
- key 一失效还会击穿 DB。

**常见做法：**

1. **多级缓存：本地缓存 + Redis**

   - 热 key 读操作先走本地缓存（Caffeine/Guava），本机命中直接返回；

   - miss 再查 Redis / DB。

   - 大厂常见三层：

     ```text
     Jvm 本地缓存 → Redis → DB
     ```

2. **热 key 复制 / 拆 key 分散读压力**

   - 把一个热 key 复制成多个 key：

     ```text
     hot:key      →  hot:key:0, hot:key:1, ... hot:key:N-1
     ```

   - 读的时候随机 / hash 到其中一个，写的时候多 key 同步写：

     - 或“一个主 key + 多个副本 key”，读随机读副本。

   **目的**：让热点分散到多个分片 / 多个实例。

3. **热点发现 + 自动降级策略**

   - 使用中间件 / Agent 统计 Redis key 的访问频率；
   - 一旦发现某 key QPS 过高：
     - 对这个 key 启用本地缓存；
     - 提前预热值；
     - 对写入/刷新频率做限流。

4. **防止击穿：加互斥锁 / SingleFlight**

   - 对同一个热 key，缓存 miss 时：

     - 只允许一个线程去 DB 回源；
     - 其他线程阻塞/自旋等待，避免“上千个请求同时打 DB”。

   - 伪代码：

     ```java
     // pseudo single-flight
     Value v = localCache.get(key);
     if (v == null) {
         Lock lock = getLock(key);
         lock.lock();
         try {
             v = localCache.get(key);
             if (v == null) {
                 v = loadFromDB();
                 localCache.put(key, v);
                 redis.set(key, v);
             }
         } finally {
             lock.unlock();
         }
     }
     ```

5. **接口限流 / 特定业务降级**

   - 对某个热 key 对应的接口加 QPS 限制；
   - 抢购 / 秒杀场景：排队削峰、预热静态页面、只允许请求进入“排队队列”。

> 一句话：**热 key 的主方向是“分散 + 本地缓存 + 防击穿 + 限流”。**

------

## 二、MySQL 的热点行 / 大行

### 1. 热点行 / 热点索引

**常见处理：**

1. **打散单行写入：分桶计数 / 分表**

   - 例如一个计数表，不把所有计数写在一行：

     ```text
     (id = 1, count = ?)  ❌
     ```

   - 改成多行分桶：

     ```text
     (id = 1, bucket = 0..N-1, count = ?)
     ```

   - 写入时随机/按 hash 落在不同 bucket，读的时候 SUM。

   - 用在：点赞数、播放量、浏览量、热度值等。

2. **改同步写为异步累加**

   - 前端操作只写入 MQ / 缓存；
   - 后台批量汇总写回 DB（比如每秒/每分钟归档一次）。

3. **缓存 + 限流**

   - 热数据加缓存（Redis），DB 只作为落盘；
   - 接口 QPS 超过阈值时，部分降级/丢弃。

4. **避免自增主键变成聚簇页热点**

   - 高并发写入时，MySQL 自增主键会导致聚簇索引最后一页成为热点；
   - 有的会：
     - 换成雪花 ID / 随机 ID（分摊到多个页）；
     - 或者用分表、分区缓解。

------

### 2. 大行 / 大字段

**常见处理：**

1. **垂直拆分：冷热字段分表**

   - 把大 JSON、大 TEXT 字段拆到附表：

     ```text
     user_main (id, name, age, ...)
     user_ext  (user_id, big_profile_json, ...)
     ```

   - 常用字段在主表（查询频繁但小），大字段不参与大多数查询。

2. **不做 SELECT ，只查需要的字段*

   - 尤其避免把大字段无脑查出来；
   - 导致网络 IO、反序列化都很大。

3. **存对象存储 + URL**

   - 真正很大的内容（长文本、大二进制）放 OSS / 文件系统；
   - DB 只存一个 URL / key。

------

## 三、本地缓存 / 多级缓存中的 big/hot

### Big 值

- 应对方案：
  - 对象拆小，只缓存核心字段；
  - 对大的复杂结构拆成多个 key；
  - 做压缩（如 gzip），但要权衡 CPU。

### Hot 值

- 本地缓存 + 远端缓存（上面 Redis 那套）
- 对特别热的热点 key：
  - 本地设较短 TTL + 读写锁更新；
  - 允许短时间“读旧值”，换取抗压能力。

------

## 四、网关 / HTTP 层：热点接口 & 大响应

### 热点接口（Hot URL）：

1. **网关限流 + 业务降级**
   - 使用 Nginx / API Gateway 的限流模块：
     - 针对 URL / 某用户 / 某 IP 限流；
   - 超限时直接返回 429 或走降级策略。
2. **静态化 + CDN**
   - 热门商品详情、内容页提前静态化为 HTML；
   - 由 CDN 承载大部分 QPS。
3. **接入层缓存**
   - 网关层做短 TTL 接口缓存（如 1s/2s），少量不一致换高抗压。

### 大响应（Big Response）：

1. **分页 / 游标，禁止一次性全量拉取**
2. **流式/分块下载**（比如大文件导出，用异步任务 + 下载链接）
3. **压缩传输**（gzip/deflate），适当减小网络压力。

------

## 五、Kafka / MQ 的大消息 / 热分区

### 大消息

- 通常的经验：单条消息不要超过几百 KB。
- 治理方式：
  - 大 payload 放对象存储，只把 URL 放进消息；
  - 开启压缩（snappy / lz4），但也要看 CPU；
  - 设置 Producer 端的 `max.request.size`、Broker 端的 `message.max.bytes` 做硬限制。

### 热分区（Hot Partition）

- 避免某个分区堆满流量：
  - 设计分区键时要均匀 hash，比如：
    - 原来用 `orderId % 4`，发现业务上某些 ID 段特别集中；
    - 改为引入随机因子或更多分区。
- 对真正需要“按 key 有序”的场景：
  - 使用“局部有序”，或将热点 key 分桶（e.g. 把 userId 变成 userId#bucket）。

------

## 六、业务层的“热商品 / 热用户”治理

这个更多是**技术 + 产品策略结合**：

- 秒杀 / 抢购：
  - 静态页 + 预加载库存到 Redis；
  - 预减库存 + 队列排队；
  - 严格限流（“排队中，请稍后再试”）。
- 热榜 / 热搜：
  - 定时离线计算 + 小批次增量更新；
  - 展示读缓存，不实时读 DB。



# 实际案例

有的，而且还不少，完全可以当成你面试时的“实战素材库”。我给你挑几类**真·大厂落地方案 / 中间件**，每个都带上“可以怎么讲”的点，方便你往自己项目里迁移、包装。

------

## 1. Redis 维度：云厂商自带的 Big Key / Hot Key 方案

### 1）阿里云 ApsaraDB for Redis

阿里云在 Redis 上专门做了：

- **大 Key / 热 Key 诊断分析功能**：在控制台可以直接看到：
  - 哪些 key 占用空间最大（big key）
  - 哪些 key QPS 最高（hot key）([support.huaweicloud.com](https://support.huaweicloud.com/intl/en-us/ae-ad-1-usermanual-dcs/dcs-ug-190808001.html?utm_source=chatgpt.com))
- 官方最佳实践里推荐的方案包括：
  - 引入 **本地缓存 + Redis 二级缓存** 组合解决热点问题；
  - 对热点 key 做拆分、预热、限制 TTL 等。([AlibabaCloud](https://www.alibabacloud.com/blog/redis-hotspot-key-discovery-and-common-solutions_594446?utm_source=chatgpt.com))

**你可以在面试里说：**

> 像阿里云的 Redis 就内置了大 Key 和热 Key 分析能力，我们可以在控制台直接看到哪些 Key 占用内存最多、QPS 最高，然后按这些 Key 做精细化治理，比如拆分集合 / 改数据模型 / 对热 Key 加本地缓存和限流策略。

------

### 2）华为云 DCS Redis

华为云的 DCS 也有同样的功能：

- 提供**大 Key 分析 + 热 Key 分析**页面：
  - 扫描实例中占空间最大的 key；
  - 扫描访问次数最高的 key；([华为支持](https://support.huawei.com/enterprise/en/doc/EDOC1100370592/d01764d9?utm_source=chatgpt.com))
- 官方文档甚至直接把 hot key 和 cache breakdown（缓存击穿）联系在一起，建议配合本地缓存、限流保护 DB。([华为支持](https://support.huawei.com/enterprise/en/doc/EDOC1100370592/d01764d9?utm_source=chatgpt.com))

**你可以说：**

> 云上 Redis 都已经把大 Key/热 Key 分析做成了标准能力，比如华为云 DCS 的控制台可以一键分析，然后配合应用层的改造来解决这些 Key 的问题。

------

## 2. 真实业务：Twitter 处理时间线 Hot Key 的方案

Twitter 在自己的技术分享里专门讲过**时间线存储的 hot key 问题**：

- 他们在 timeline 用 Redis 存储时，会遇到“某些大 V 的时间线被疯狂访问”的情况；
- 实际方案大致是两步：([matthewtejo.substack.com](https://matthewtejo.substack.com/p/handling-hotkeys-in-timeline-storage?utm_source=chatgpt.com))
  1. **Redis 端检测热 Key**：通过采样统计，对访问频次极高的 key 打标为 hotkey，并通过某种通道（RPC/消息）通知上游服务；
  2. **API 层本地缓存 + 特殊处理**：
     - 对这些被标记的 hot key，在 API 层启用本地缓存（如内存 LRU）；
     - 降低对 Redis 的直接访问频率；
     - 必要时对这些热 Key 的写/刷缓存走异步或特殊限流。

**适合你面试时说：**

> 像 Twitter 的时间线服务就是典型的热点 Key 场景，他们是 Redis 在后面，前面再配一层逻辑：Redis 负责检测 hot key，然后 API 层收到信号后，对这些 key 做本地缓存和特殊限流处理，尽量把流量拦在 Redis 之前。

------

## 3. 全局缓存系统：Netflix 的 EVCache（抗热点 + 多级缓存）

Netflix 的缓存体系很经典，他们用的是自己封装的 **EVCache**（基于 memcached）：([techblog.netflix.com](https://techblog.netflix.com/2012/01/ephemeral-volatile-caching-in-cloud.html?utm_source=chatgpt.com))

### 核心特点：

- **多副本 + 多 AZ/多 Region 分布**：
   每份数据会写入多个 memcached 集群副本，读的时候可以从最近的一个读，既抗热点又提升可用性；
- **客户端分片 + 本地逻辑**：
   客户端 SDK 做一致性哈希、负载均衡，应用端可以在必要时再叠加本地缓存；
- **针对热点/大流量场景的优化**：
  - 比如热门内容（Top10、个性化推荐）会提前预热；
  - 出现节点故障时，会用**副本读** + 后台渐进式预热，避免一下子压垮下游。([InfoQ](https://www.infoq.com/articles/netflix-global-cache/?utm_source=chatgpt.com))

**你可以这样讲：**

> Netflix 用的是自己的一套 EVCache，相当于“客户端 + 多集群 memcached”的组合。对于热门内容，它通过多副本 + 本地缓存的方式分散热点；频繁访问的数据被缓存到离用户最近的节点，在 Region 之间还做了复制和预热，这样热点不会压垮单个缓存节点，某个节点挂了也不会导致大面积缓存击穿。

------

## 4. 网关 / 接口层：阿里 Sentinel 的“热点参数限流”

阿里开源的 **Sentinel**（现在基本是 Spring Cloud Alibaba 标配）有一个专门的功能叫：

> **热点参数限流（Hot Parameter Flow Control）**

用途：

- 对某个接口的**某个参数值**做限流，比如：
  - `/item/detail?itemId=xxx`
  - 对 `itemId` 这个维度做统计，识别出特定的几个 `itemId` 访问量远高于其他；
- 对这些热点参数值做单独 QPS 限制，防止某个爆款商品把整体资源耗尽。([sentinelguard.io](https://sentinelguard.io/en-us/docs/parameter-flow-control.html?utm_source=chatgpt.com))

这就是“**接口层的 hot key**”：不是 Redis key，而是“参数值 = 某几个 id”。

> 阿里在网关层用 Sentinel 做热点参数限流，比如对商品详情接口，会统计某几个商品 ID 的访问频率，如果某个 ID 达到热点阈值就单独限流，不影响其他商品。这个思路可以类比 Redis 热 Key 的治理：都是针对“小部分极热点对象做特别处理”。

------

## 5. 云厂商托管 Redis 的“内建中间件化”能力

很多云 Redis 实际上已经把 hot key/big key 的处理做成“产品级能力”：

- 华为云 / 阿里云 的 Redis 产品都提供：
  - 大 key 分析；
  - 热 key 分析；
  - 基于分析结果做报警、建议拆分 / 优化方案。([华为支持](https://support.huawei.com/enterprise/en/doc/EDOC1100370592/d01764d9?utm_source=chatgpt.com))
- 有的还提供“客户端本地缓存 SDK”：
  - 访问先查本地，再查 Redis，帮助你处理热点问题。([static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com](https://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/download%2Fpdf%2F26365%2FBest_Practices_intl_en-US.pdf?utm_source=chatgpt.com))

**你可以讲成：**

> 现在很多云厂商在 Redis 层已经提供了 big key/hot key 的分析，以及一些本地缓存 SDK。我自己在设计时会参考他们的做法：
>  1）定期跑大 key/hot key 分析，定位问题 key；
>  2）对大 key 做模型调整和拆分；
>  3）对热 key 则在业务侧加本地缓存 + 限流，并考虑 key 打散或复制。

------

## 6. 你可以怎么组织成“面试故事”

你完全可以把上面内容整合成一段自己的“方案演进”：

> 在我理解中，大厂对热 Key、大 Key 的治理，基本都是分层做的：
>
> - **缓存层（Redis/Memcached）**：借鉴像阿里云、华为云那样的大 Key / 热 Key 分析能力，先把“谁大谁热”识别出来；然后对大 Key 做拆分和分页，对热 Key 做本地缓存、多副本、打散 key。Twitter 的时间线就是用 Redis 检测热点，然后在 API 层对这些用户时间线做本地缓存。
> - **应用层 / 网关层**：类似阿里的 Sentinel，用热点参数限流的方式，对某些爆款商品 ID / 热门用户 ID 做单独限流或降级，防止热点拖垮整体服务。
> - **全局缓存架构**：参考 Netflix 的 EVCache，通过多级缓存 + 多 Region 副本，把热点数据分散在多个节点和本地内存中，既提高性能也避免单点热点。
>
> 实际项目里，我会先用云Redis的大 Key / 热 Key分析功能或自研采样，找出问题 key；再用这些思路落地：要么拆模型、拆集合，要么在业务层对这些 key 分桶、复制、副本缓存，同时在接口层用限流和降级兜底。

------



- **Sentinel 热点参数限流**（直接能在 Spring Cloud 里用）
- 或 **EVCache / 本地缓存 + Redis 的多级缓存实践**



## 7. JD-Hotkey

京东开源的 JD-Hotkey

自动找到热 key（甚至可以扩展到热用户/热接口），然后自动把这些热点数据推到各个应用的 JVM 本地缓存里。

### 1. 很可能是它：京东 JD-Hotkey

JD-Hotkey 是京东 App 后台团队开源的**高性能热数据探测中间件**：

- 做的事就是：
  1. **实时探测热 key**（包括 Redis 热数据、MySQL 热数据、热用户、热接口等）；
  2. 把探测出来的热 key 在 **毫秒级推送** 到各个业务服务节点；
  3. 在每个服务节点的 **JVM 内存里建一份本地缓存**（通常 LRU，本地 Map/Caffeine）；
  4. 之后访问这个 key 时，优先从本地 JVM 读，尽量不打 Redis / DB。([CSDN](https://blog.csdn.net/qq_45260619/article/details/134835378?utm_source=chatgpt.com))  



但是怎么保证jvm本地缓存和实际redis和DB里面数据一致性？



官方描述里就是：

> “实时探测出系统的热数据，并将热数据毫秒内推送至系统的业务集群服务器的 JVM 内存。”([developer.jdcloud.com](https://developer.jdcloud.com/article/2854?utm_source=chatgpt.com))

这和你说的**“自动找到热key，然后在 JVM 里自动构建本地缓存”**是一模一样的。

> 🔁 **注意**：JD-Hotkey 主要解决的是 **热 key**，不是 Big Key。大 key 一般还是靠 Redis 端扫描/监控来发现然后做拆分。

------

### 2. 其他类似的也有（你可能看到过）

如果不是 JD-Hotkey，还有几个也很像你说的模式：

1. **有赞 TMC（多级缓存 + 热点探测）**
   - 做热点 key 探测，然后把热点数据缓存在 JVM 本地，结合多级缓存结构。([CSDN](https://blog.csdn.net/tianyaleixiaowu/article/details/102794451?utm_source=chatgpt.com))
2. **得物的 Burning 热点探测组件**
   - 实时探测 Redis 热 key；
   - 对这些热点数据做**本地缓存预热 + 实时本地缓存**，明显降低 Redis / DB 压力。([tech.dewu.com](https://tech.dewu.com/article?id=23&utm_source=chatgpt.com))
3. 有些公司自己做的热 key 探测框架
   - 典型套路都是：
     - agent/SDK 上报访问日志 → 后台 worker 统计 → 判断是否热 key；
     - 热 key 列表通过 MQ/etcd/Redis pubsub 等**推送到应用 JVM**；
     - SDK 收到后，把这些 key 值塞进本地 LRU 缓存（Guava/Caffeine），后续请求就直接打 JVM。

------

### 3. 面试时你可以这么说（一句话版）

> 我了解过京东开源的 **JD-Hotkey**，它会实时统计整个集群对 key/用户/接口的访问频率，一旦识别出热 key，就会通过中心集群把这些热 key 推送到每个业务节点的 JVM 本地缓存里。这样后续对这些热 key 的访问可以直接命中本地缓存，把大量 QPS 挡在应用层，极大减轻 Redis/DB 压力。
>  这个思路相比我们自己手写“本地 + Redis 多级缓存”的区别在于：**它可以自动发现热 key，并自动在 JVM 中维护本地缓存，而不是完全依赖人工预估哪些是热点。**

