# 内存满了触发OOM会发生什么？

这个问题里其实有 **两种完全不同的 “OOM”**，很多人会混在一起：

1. **Java 的 OOM（java.lang.OutOfMemoryError）**：JVM 在“它自己能管理/申请的内存或资源”里分配失败，于是抛 Error。 [Oracle 文档](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/OutOfMemoryError.html?utm_source=chatgpt.com)
2. **Linux 的 OOM（kernel OOM killer）**：内核在系统（或 cgroup）层面已经回收不出足够内存来维持运行，只能“杀一个进程救全局”。 [Linux内核文档+1](https://docs.kernel.org/admin-guide/mm/concepts.html)



把**定义、触发过程、杀谁的标准、JVM 又会怎样**系统讲清楚。

------

## 一、先把“内存满了”这句话说精确：Linux 到底什么叫满？

### 1) 虚拟内存 vs 物理内存

- **虚拟内存（VIRT / address space）**：进程“看到的”可用地址空间，不等于实际占用。
- **物理内存（RAM）**：真实内存页。
- **RSS**：进程当前真正驻留在 RAM 里的部分（常用来衡量“吃内存”）。

### 2) Linux 里“空闲内存很少”不等于要 OOM

Linux 会尽量把内存用来做 **page cache（页缓存）**，提升 IO 性能。缓存是可以回收的。内核把可回收页称为 reclaimable，回收过程叫 reclaim。([Linux内核文档](https://docs.kernel.org/admin-guide/mm/concepts.html))

**所以真正危险的是**：当需要分配内存时，内核已经**回收（reclaim）+ 必要的 swap/回写**都做了，还是拿不出足够页给这次分配。

------

## 二、Linux 内存紧张到触发 OOM killer：系统会发生什么？

### 触发链路（从“紧张”到“杀进程”）

1. **kswapd 异步回收**：当空闲页低于阈值，分配会唤醒 `kswapd` 去扫描并回收（丢 page cache 或把匿名页 swap 出去）。([Linux内核文档](https://docs.kernel.org/admin-guide/mm/concepts.html))
2. **direct reclaim 同步回收**：更紧张时，发起分配的线程会被“卡住”，直到回收到足够页。([Linux内核文档](https://docs.kernel.org/admin-guide/mm/concepts.html))
3. **必要时 compaction**：尝试整理碎片，满足大块连续内存需求。([Linux内核文档](https://docs.kernel.org/admin-guide/mm/concepts.html))
4. **仍然失败 → OOM killer**：内核无法再回收足够内存，为了保住系统，启动 OOM killer，选择一个“牺牲者”杀掉，希望释放内存恢复正常。([Linux内核文档](https://docs.kernel.org/admin-guide/mm/concepts.html))

### OOM killer “根据什么标准”杀进程？

Linux 会给每个候选进程算一个 **badness score（坏度/牺牲分）**，范围大致 **0～1000**：用的内存/交换越多，越容易被杀。([man7.org](https://man7.org/linux/man-pages/man5/proc_pid_oom_score_adj.5.html))

关键可观测指标（你排障会用到）：

- `/proc/<pid>/oom_score`：当前分数（越高越危险）
- `/proc/<pid>/oom_score_adj`：人为“加/减权重”的调节值，范围 **-1000～+1000**
  - `-1000` 基本等价于“不要杀我”（分数会被压到 0）
  - `+1000` 基本等价于“优先杀我”
    ([man7.org](https://man7.org/linux/man-pages/man5/proc_pid_oom_score_adj.5.html))

> man page 还明确说：badness 是根据进程的 **当前内存和 swap 使用量**估算的；并且 root 进程会有额外权重因素。([man7.org](https://man7.org/linux/man-pages/man5/proc_pid_oom_score_adj.5.html))

### 发生 OOM kill 后系统会输出什么？

- 内核日志会记录“Out of memory… Killed process …”
- 还可以通过 sysctl 打开更详细的 task dump（包含 pid、rss、swap、oom_score_adj 等）。([Linux内核档案馆](https://www.kernel.org/doc/Documentation/admin-guide/sysctl/vm.rst?utm_source=chatgpt.com))

你在机器上排查最常用的命令：

```bash
dmesg -T | grep -i -E "out of memory|killed process|oom"
journalctl -k | grep -i -E "out of memory|killed process|oom"

cat /proc/<pid>/oom_score
cat /proc/<pid>/oom_score_adj
```

------

## 三、容器 / cgroup 场景：为什么“宿主机没满，容器却 OOMKilled”？

在 cgroup v2 下，OOM 可能是 **“这个 cgroup 达到了 memory.max 限制”**，并不是整机 RAM 真没了。`oom_score_adj` 的 man page 也说明：当 OOM 是因为 **memory limit reached**，allowed memory 就是那个限制值。([man7.org](https://man7.org/linux/man-pages/man5/proc_pid_oom_score_adj.5.html))

当 `memory.current` 顶到 `memory.max` 且回收不够时，OOM killer 会在该 cgroup 内执行（常见行为是干掉该 cgroup 里“最大的进程/占用最多的进程”）。([Server Fault](https://serverfault.com/questions/1192733/actual-sequence-of-events-from-memory-pressure-to-oom-for-cgroups-v2?utm_source=chatgpt.com))

另外还有 `memory.oom.group`：开启后，OOM 发生时会把整个 cgroup 当成一个不可分割工作负载，“一起杀或都不杀”，避免只杀子进程导致服务半残。这个特性在 Kubernetes 场景里很常见。([GitHub](https://github.com/kubernetes/kubernetes/issues/117070?utm_source=chatgpt.com))

------

## 四、Java / JVM 的 OOM：到底是什么、会发生什么？

### 1) `OutOfMemoryError` 的“官方定义”

JDK 17 的 API 文档：当 JVM **无法分配对象**，且 GC 也无法再腾出更多可用内存时抛出。([Oracle 文档](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/OutOfMemoryError.html?utm_source=chatgpt.com))

### 2) Java 里“常见 OOM 类型”你需要能读懂（业务+面试必备）

面试官通常不要求你背全，但要求你能把**症状→根因方向**说清楚：

- `java.lang.OutOfMemoryError: Java heap space`：**堆满了**（对象太多/泄漏/`-Xmx` 太小）。
- `java.lang.OutOfMemoryError: GC overhead limit exceeded`：GC 花了极高比例时间但每次回收很少（典型是堆已接近被“活对象”塞满）。Oracle 故障排查指南解释了触发条件（98% 时间 GC、回收 <2% 等）。([Oracle 文档](https://docs.oracle.com/javase/jp/8/docs/technotes/guides/troubleshoot/memleaks002.html?utm_source=chatgpt.com))
- `java.lang.OutOfMemoryError: Metaspace`：类元数据/动态生成类太多（框架、代理、热部署、类加载器泄漏等）。
- `java.lang.OutOfMemoryError: Direct buffer memory`：NIO direct buffer 相关（堆外）。
- `java.lang.OutOfMemoryError: unable to create new native thread`：更多时候是 **线程/系统资源耗尽**（进程能创建的线程数、虚拟内存、ulimit 等），不一定是堆满。([Stack Overflow](https://stackoverflow.com/questions/16789288/java-lang-outofmemoryerror-unable-to-create-new-native-thread?utm_source=chatgpt.com))

### 3) JVM 发生 OOM 后“接下来会发生什么”？

- **OOME 是 Error**：如果没被捕获，会导致抛出它的线程终止；如果在主线程等关键线程未捕获，通常进程就结束。
- 即使你 catch 住（很多人会 try-catch `Throwable`），程序也往往处在**不可预测状态**（因为内存已经极端紧张，继续分配/日志/告警都可能失败），生产上通常建议：**采集证据后快速失败并重启**。

### 4) JVM 和 Linux OOM killer 的关系：谁先发生？

有三种典型情况：

**A. JVM 先 OOME，系统没 OOM**
比如 `-Xmx` 配得太小：堆先满，JVM抛 OOME，但系统 RAM 还有余量。

**B. 系统（或 cgroup）先 OOM kill，JVM 来不及抛 OOME**
这是线上很常见的：内核直接把 Java 进程 kill 掉（K8s 里表现为 `OOMKilled`，退出码常见 137），**你不会看到 Java 栈里的 OOME**，也可能来不及打 heap dump。

**C. JVM 抛了 OOME，但随后进程仍可能被内核 OOM kill**
比如 OOME 之后你继续分配、继续起线程、继续打日志，反而触发系统层面的更大内存压力。

------

## 五、两边各自“如何自救”：你要能给面试官/同事一个可执行方案

### 1) Linux / 容器侧：怎么确认是 kernel OOM kill？

- 查内核日志：`dmesg -T` / `journalctl -k`
- 看进程的 `oom_score` / `oom_score_adj`（如果进程还活着）
- 容器里看 cgroup：`memory.current`、`memory.max`（以及 Kubernetes 事件）

### 2) JVM 侧：怎么在 OOM 时留下证据？

JDK 17 官方文档里有两个对生产很有用的选项：

- **OOM 时自动生成 heap dump**：`-XX:+HeapDumpOnOutOfMemoryError`（并可配 `-XX:HeapDumpPath=...`）([Oracle 文档](https://docs.oracle.com/en/java/javase/17/troubleshoot/command-line-options1.html))
- **OOM 时执行脚本/命令**：`-XX:OnOutOfMemoryError=...`（比如打 jstack、发告警、抓 /proc 等）([Oracle 文档](https://docs.oracle.com/javase/jp/17/troubleshoot/command-line-options1.html))

> 注意：如果是 **内核直接 SIGKILL**（容器 OOMKilled），这些 JVM 钩子可能来不及运行，所以还要配合系统层日志/监控。

------

## 六、面试“加分回答模板”：一句话把两种 OOM 讲清楚

> Java 的 `OutOfMemoryError` 是 JVM 在堆/元空间/堆外/线程等资源分配失败后的错误；Linux 的 OOM killer 是内核在无法回收足够内存维持系统（或 cgroup）运行时，按 `oom_score`（可被 `oom_score_adj` 调整）选择“最该死”的进程杀掉来恢复系统。([man7.org](https://man7.org/linux/man-pages/man5/proc_pid_oom_score_adj.5.html))

------

如果你愿意，我可以基于你的真实环境再把“精通”部分补齐：

- 你是在 **裸机/VM** 还是 **Docker/Kubernetes**？
- Java 用的是 **JDK17 HotSpot** 吗？是否有容器内存限制（比如 Pod limit）？

​    