## 1）XXL-JOB 是什么？你先把它当成“企业级定时任务平台”

**一句话定义：**
XXL-JOB 是一个**分布式任务调度平台**：你在网页上配置“什么时候做什么事”，平台会在合适的时间把任务派发到某台（或多台）业务机器上执行，并提供日志、告警、重试、分片、路由等能力。 ([GitHub](https://github.com/xuxueli/xxl-job/))

它解决的核心痛点：

- 单机 `cron`/`@Scheduled`：**看不见、管不住、扩不动、失败不好追**
- 多机器部署：容易出现**重复执行、错过执行、无法故障转移**
- 需要统一：**任务管理、审计、告警、日志、权限**等

------

## 2）核心架构：两大角色 + 一个数据库

你只要记住这张“脑图”：

### A. 调度中心（xxl-job-admin）

- 提供 Web 控制台：任务 CRUD、手动触发、查看日志、失败告警、统计报表等
- 负责按 Cron/事件等触发策略进行调度（并支持集群 HA） ([GitHub](https://github.com/xuxueli/xxl-job/))

### B. 执行器（executor）

- 你的业务服务（Spring Boot 项目）接入 xxl-job-core 后，暴露执行接口
- 调度中心把任务“派发”给执行器执行；执行器可集群部署，实现执行 HA ([GitHub](https://github.com/xuxueli/xxl-job/))

### C. DB（非常关键）

- 调度中心用 DB 做任务、触发记录、日志等存储
- 调度中心集群一致性：通过 DB 锁保证“一次调度只触发一次” ([GitHub](https://github.com/xuxueli/xxl-job/))

------

## 3）最重要的名词

- **任务（Job）**：一条定时/触发规则 + 运行参数 + 执行策略
- **执行器（Executor）**：实际跑代码的服务
- **执行器 AppName**：执行器集群的“逻辑名字”（同名表示同一组机器）
- **JobHandler**：你写的“任务入口方法”
- **路由策略**：同一组执行器多台机器时，派发到谁（轮询/随机/一致性 hash/故障转移/忙碌转移等） ([GitHub](https://github.com/xuxueli/xxl-job/))
- **阻塞策略**：任务太密集，上一次还没跑完怎么办（串行/丢弃/覆盖等） ([GitHub](https://github.com/xuxueli/xxl-job/))
- **分片广播**：一次触发让集群里每台执行器都执行一次，并拿到分片参数做分治 ([GitHub](https://github.com/xuxueli/xxl-job/))
- **失败重试/超时控制/告警**：平台内建能力 ([GitHub](https://github.com/xuxueli/xxl-job/))

------

## 4）10 分钟跑起来（推荐用 Docker 示例理解整体链路）

你最初只需要跑通：**控制台能看到执行器 → 新建任务 → 手动触发 → 看到日志**。

参考一个可运行示例仓库：控制台地址、默认账号密码等写得很清楚（例如 `admin/123456`）。 ([GitHub](https://github.com/qct/xxl-job-example))

### 跑通链路的最小步骤（概念版）

1. 启动 **xxl-job-admin**（它需要连 MySQL，并初始化表结构）
2. 启动你的 **executor**（Spring Boot 服务，接入 xxl-job-core）
3. 在 admin 控制台：
   - 新增“执行器”（填 AppName、注册方式等）
   - 新增“任务”（选择 BEAN/GLUE 等运行模式、Cron、路由/阻塞策略）
4. 点“执行一次”，去日志页面看执行输出

> 真正上手时你会发现：XXL-JOB 的学习曲线主要在“控制台配置项”和“执行器注册/网络连通性”。

------

## 5）执行器端怎么写（业务开发 90% 都在这里）

下面按“你最常用的 Spring Boot 接入方式”讲清楚。

### 5.1 引入依赖

你会用到 `xxl-job-core`（版本按公司选型，建议跟随公司或选最新稳定）。例如历史上常见 2.4.0： ([Maven Repository](https://mvnrepository.com/artifact/com.xuxueli/xxl-job-core/2.4.0?utm_source=chatgpt.com))
（但生产环境务必关注安全公告/漏洞修复，后面会讲。）

### 5.2 配置（你需要理解的关键参数）

- `adminAddresses`：调度中心地址（可多个）
- `appname`：执行器组名（同组多实例 = 执行器集群）
- `address` / `ip` / `port`：执行器对外可访问地址（容器/多网卡时最容易踩坑）
- `accessToken`：调度中心与执行器通信鉴权（建议开）
- 日志路径、日志保留天数

### 5.3 写一个任务 Handler（BEAN 模式最常用）

典型写法是给方法打注解（不同版本注解/类名略有差异，但思想一致）：

- 在 handler 里打印日志（控制台能滚动查看日志是 XXL-JOB 的强项之一） ([GitHub](https://github.com/xuxueli/xxl-job/))
- 做好**幂等**（失败重试/重复触发时不出事故）

### 5.4 分片任务怎么写（大厂高频）

当你把路由策略选为**分片广播**时，一次触发会广播到每台执行器，每台拿到 `分片序号/分片总数`，你按这个把数据切片处理。 ([GitHub](https://github.com/xuxueli/xxl-job/))
适用场景：**全量数据修复、离线批处理、按用户分桶推送**等。

------

## 6）控制台配置项：把“面试高频点”一次讲透

### 6.1 触发策略（什么时候触发）

官方列了很多：Cron、固定间隔、固定延时、API 事件触发、人工触发、父子任务触发等。 ([GitHub](https://github.com/xuxueli/xxl-job/))

面试怎么答：

- **Cron**：严格按时间点
- **固定延时**：更像“上次结束后再等 N 秒”（避免挤压）
- **事件/API 触发**：用于“业务事件来了就跑一次”（例如补偿、对账）

### 6.2 路由策略（派发给谁）

常见：轮询、随机、一致性 HASH、故障转移、忙碌转移等。 ([GitHub](https://github.com/xuxueli/xxl-job/))

面试怎么答（给场景）：

- **一致性 HASH**：同一 key（如用户ID）尽量落到同一台，提升缓存命中/减少并发冲突
- **故障转移**：某台挂了自动换一台，适合高可用任务 ([GitHub](https://github.com/xuxueli/xxl-job/))
- **忙碌转移**：优先找空闲机器，适合耗时波动大的任务

### 6.3 阻塞策略（上一次没跑完）

- 串行（默认）、丢弃后续、覆盖之前等 ([GitHub](https://github.com/xuxueli/xxl-job/))

面试怎么答：

- **串行**：保证不并发，适合非幂等/有锁任务
- **丢弃后续**：宁可少跑也不堆积，适合低优先级统计
- **覆盖之前**：只关心最新一次，适合“刷新缓存/刷新报表”

### 6.4 超时、失败重试、告警

- 支持任务超时控制（超时主动中断） ([GitHub](https://github.com/xuxueli/xxl-job/))
- 支持失败重试（分片任务还能“分片粒度重试”） ([GitHub](https://github.com/xuxueli/xxl-job/))
- 默认邮件告警，并预留扩展接口接短信/钉钉等 ([GitHub](https://github.com/xuxueli/xxl-job/))

面试必加一句：

- “**重试不等于可靠**”：要结合幂等、去重、业务补偿、死信/人工介入。

------

## 7）大厂常见落地场景（“场景 → 为什么用 XXL-JOB → 配什么策略”）

1. **离线批处理/ETL**
   - 特点：数据量大、耗时长、需要扩容	
   - 配置：分片广播 + 合理路由 + 超时 + 失败告警
2. **对账/结算/报表**
   - 特点：强一致、不能重复扣款
   - 配置：串行/加分布式锁 + 幂等 + 失败告警 + 手动补跑入口
3. **缓存预热/定时刷新**
   - 特点：只要最新结果
   - 配置：覆盖之前 + 固定间隔/延时





------

## 8）优缺点（选型）

### 优点

- 上手快：Web 管理、动态改任务、立即生效 ([GitHub](https://github.com/xuxueli/xxl-job/))
- 分布式能力完整：注册中心、执行器/调度中心 HA、路由/分片/Failover ([GitHub](https://github.com/xuxueli/xxl-job/))
- 可观测：Rolling 日志、任务进度监控 ([GitHub](https://github.com/xuxueli/xxl-job/))
- 生态：脚本任务、GLUE 在线编辑、命令行任务等 ([GitHub](https://github.com/xuxueli/xxl-job/))

### 缺点/风险点

- **网络与地址配置容易踩坑**（容器、NAT、多网卡、反向代理）
- **任务语义主要是“调度触发”**：想要强一致的“工作流编排/Exactly Once”，需要你自己做幂等、去重、补偿
- **安全风险要重视**：历史版本出现过 RCE 等漏洞（例如 NVD 记录的 CVE-2023-48089 指向 xxl-job-admin 2.4.0 的 RCE）。([国家漏洞数据库](https://nvd.nist.gov/vuln/detail/CVE-2023-48089?utm_source=chatgpt.com))
  生产建议：升级到较新版本、开启鉴权、限制管理端访问、谨慎使用 GLUE/脚本/命令行等高危能力

------

