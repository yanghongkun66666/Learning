 # JVM内存结构

JVM 内存结构（运行时数据区）是指 Java 虚拟机在执行 Java 程序时，为了存储字节码执行所需的各种数据，而在内存中划分出的若干功能不同的区域及其组织方式。

按照《Java 虚拟机规范》，这些运行时数据区主要包括：

1. **堆（Heap）**：线程共享，用来存放绝大多数对象实例和数组，是垃圾回收器管理的主要区域。
2. **方法区（Method Area，JDK8 起多以 Metaspace 方式实现）**：线程共享，用来存放类元数据，如类信息、常量池、字段和方法信息、JIT 编译后的代码等。
3. **Java 虚拟机栈（Java Stack）**：线程私有，按栈帧保存方法调用的局部变量、操作数栈、返回地址等。
4. **本地方法栈（Native Method Stack）**：线程私有，为 JNI 等本地方法服务。
5. **程序计数器（Program Counter, PC 寄存器）**：线程私有，记录当前线程下一条将要执行的字节码指令地址。



# OOM定义

**OOM 的定义可以这样说：**

在 Java 中，**OOM 指的是 java.lang.OutOfMemoryError（Out Of Memory Error）**，是 JVM 在运行过程中出现的一个严重错误。**当 Java 虚拟机在某个内存相关区域（如堆、方法区/元空间、直接内存、线程栈等）需要分配内存时，发现可用内存不足，并且通过垃圾回收也无法再回收出足够空间来满足本次分配请求时，虚拟机就会抛出 java.lang.OutOfMemoryError，简称 OOM。**

**OOM 是 Error，不是 Exception**：
 属于 `Error`，通常表示 JVM 运行环境或程序整体资源使用出现严重问题，**不是简单靠业务捕获就能恢复**的。

**OOM ≠ 只发生在堆**：
 常见的几种形式本质都属于 OOM，例如：

- `Java heap space`（堆内存不足）
- `Metaspace` / `PermGen space`（类元数据区不足）
- `Direct buffer memory`（直接内存不足）
- `unable to create new native thread`（无法再创建新线程，本质是本地内存/系统资源不够）

**内存泄漏是 OOM 的常见原因之一，但不是全部**：
 也可能是堆/元空间配置过小、数据量确实超大、线程太多等导致。













JVM 里**几乎每一块“会动态分配内存”的区域，都有机会触发 OOM**，只是表现形式和解决办法不一样。

我分 3 部分说：

1. JVM 运行时内存结构（跟 OOM 强相关的几块）
2. 各区域可能出现的 OOM 类型 & 常见原因
3. 一般排查 & 处理思路（线上出问题时你可以按这个 checklist 来）

------

## 一、JVM 运行时内存结构（与 OOM 相关的）

常见说法是这几块：

1. **Java 堆（Heap）**
   - 存放几乎所有对象实例，GC 的主战场。
   - 一般再划分为新生代（Eden + Survivor）和老年代。
2. **方法区（Method Area） / 元空间（Metaspace, JDK8+）**
   - 存放类的元数据（类结构、方法信息、常量池等），以及 JIT 编译后的代码等。
   - JDK8 以前是 PermGen（永久代），之后是本地内存中的 Metaspace。
3. **Java 虚拟机栈（Java Stack）**
   - 每个 Java 线程一个栈，存放栈帧：局部变量、操作数栈、返回地址等。
   - 栈太深或者栈容量不足会出事。
4. **本地方法栈（Native Method Stack）**
   - 供 JNI 等本地代码使用，概念类似 Java 栈。
5. **程序计数器（PC 寄存器）**
   - 存放当前线程执行字节码的地址，基本不会出现 OOM，一般不关心。
6. **直接内存（Direct Memory）**
   - 通常通过 `ByteBuffer.allocateDirect` 或 Netty 等框架分配，**不在 JVM 堆内**，但受总内存和 `-XX:MaxDirectMemorySize` 限制。
7. **操作系统层面的其他资源**
   - 比如**线程数上限**、进程可用虚拟内存上限等，触发 `OutOfMemoryError: unable to create new native thread` 等。

------

## 二、各区域可能发生的 OOM 类型 & 原因

下面把常见的 OOM 类型按区域归类一下。

> 注意：具体错误 message 可能因 JDK 版本略有差异。

### 1. 堆内存相关

#### 1）`java.lang.OutOfMemoryError: Java heap space`

**可能区域**：Java 堆
 **典型原因**：

- 对象创建过多、存活时间长，**GC 回收不掉**（内存泄漏或业务数据量暴涨）
- 堆配置过小（`-Xmx` 太小）
- 有些场景一次性构建超大数组或集合（比如一下子构 `new int[Integer.MAX_VALUE]`）。

**处理思路**：

1. **先保护现场**：

   - JVM 参数打开 OOM dump，例如：

     ```sh
     -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/path/to/dumps
     ```

2. **分析 dump**（MAT、VisualVM、YourKit、jprofiler 等）：

   - 看哪些类的实例占用最多内存
   - 是否存在典型的“泄漏链”（某个容器一直挂着大量对象不释放）

3. **从代码/设计层面优化**：

   - 减少不必要的缓存、避免把大结果一次性全放内存（分页、流式处理）
   - 用 `WeakReference` / `SoftReference` 协助缓存，不要手写永不淘汰的大 Map
   - 避免无界集合（无上限的队列、List 等）

4. **确认确实是“内存不够用而非泄漏”时**：

   - 合理调大 `-Xmx` / `-Xms`，同时评估机器物理内存
   - 调整 GC 策略、代大小比例等，减少碎片。

------

#### 2）`java.lang.OutOfMemoryError: GC overhead limit exceeded`

**可能区域**：仍然是堆，只是另一种触发条件。

**含义**：JVM 发现 **大部分时间在 GC，但回收不了多少内存**（默认条件大概是 98% 时间在 GC，但只回收不到 2% 堆空间）。

**典型原因**：

- 实际上跟 “Java heap space” 相同：堆太小或者泄漏。
- 对象一直创建、一直存活，GC 顶不住。

**处理**：

- 仍然是按照“Java heap space”的步骤排查。
- 极端情况下，可以暂时通过 `-XX:-UseGCOverheadLimit` 关闭这个保护机制，但**不建议作为根本解决方案**。

------

#### 3）`java.lang.OutOfMemoryError: Requested array size exceeds VM limit`

**区域**：堆
 **原因**：

- 程序试图创建一个**极大**的数组（如大小超过 `Integer.MAX_VALUE - 2`），超过 JVM 的内部限制。
- 一般是业务逻辑错误（误把数据量当成数组长度等）。

**处理**：

- 修正代码：避免用“未知/过大”的长度创建数组，改为分块、流式、分页。

------

### 2. 元空间/方法区相关

#### 4）JDK 8+：`java.lang.OutOfMemoryError: Metaspace`

#### JDK 7 及以前：`java.lang.OutOfMemoryError: PermGen space`

**区域**：方法区（类元数据）

**典型原因**：

- 大量**动态生成/加载类**（如频繁使用 CGLIB 动态代理、ASM 动态生成类、Groovy、JSP 编译等），但不卸载。
- 旧项目中使用大量热部署、类加载器泄漏（比如容器重启多次后类加载器一直被持有）。

**处理思路**：

1. 适当调大：
   - JDK 8+：`-XX:MaxMetaspaceSize`
   - JDK 7：`-XX:MaxPermSize`
2. 分析**类加载情况**：
   - 用 `jcmd <pid> VM.class_hierarchy`、`jmap -clstats` 等命令观察
   - 或使用 MAT / VisualVM 看 `ClassLoader` 是否过多、泄漏。
3. 减少动态代理/类生成：
   - 重构：避免每次业务请求都生成新类
   - 或复用 ClassLoader，确保不用时可以卸载。

------

### 3. 栈 & 线程相关

#### 5）`java.lang.StackOverflowError`（不是 OOM，但经常一起出现）

**区域**：Java 栈 / Native 栈
 **典型原因**：

- 递归调用无终止条件（或终止条件错误）
- 方法调用层级特别深。

**处理**：

- 修复递归逻辑，改为迭代/尾递归优化（JVM 不支持真正的 TCO，但可以改写算法）。
- 在特定场景可以稍微调大线程栈大小：`-Xss`，但要注意线程数 * 栈大小 * 机器内存。

------

#### 6）`java.lang.OutOfMemoryError: unable to create new native thread`

**区域**：操作系统层面（线程 + 本地栈）

**典型原因**：

- 程序创建了**过多线程**（比如用 `new Thread()` 乱开，而不是线程池）。
- OS 为 JVM 进程分配的最大线程数达到上限（如 Linux 的 `ulimit -u`）。
- 每个线程栈（`-Xss`）设置太大 + 线程数多，让可用内存耗尽。

**处理思路**：

1. **从应用层面限制线程数**：
   - 使用**线程池**（`Executors.newFixedThreadPool` / `ThreadPoolExecutor` 等），设置合理 maxPoolSize 和队列长度。
   - 避免为每个请求开启新线程 / 为每个任务开启新线程。
2. **检查系统层面配置**：
   - Linux 上检查 `ulimit -u`、`/etc/security/limits.conf`、内存/Swap 使用情况。
3. **必要时调整 JVM 栈大小**：
   - 使用较小的 `-Xss`（比如从 1M 改为 256k），在业务允许的情况下增加可创建线程数。

------

### 4. 直接内存相关

#### 7）`java.lang.OutOfMemoryError: Direct buffer memory`

**区域**：直接内存（即 NIO 直接缓冲区）

**典型原因**：

- 大量使用 `ByteBuffer.allocateDirect()`，但长期不释放、且频繁分配大块。
- 使用 Netty / NIO 等框架时，没有合理回收 buffer，或 `-XX:MaxDirectMemorySize` 配太小。

**处理**：

1. 调整参数：
   - 合理设置 `-XX:MaxDirectMemorySize`（不设置时，默认跟 `-Xmx` 差不多）。
2. 检查代码：
   - 避免频繁分配大块 direct buffer；能复用就复用。
   - 使用的网络框架（如 Netty）要正确 `release()` buffer。
3. 监控：
   - 通过 `jcmd VM.native_memory` 等命令看本地内存各部分使用情况。

------

### 5. 其他系统级 OOM

#### 8）操作系统级别的 OOM（进程被 OS 干掉）

比如 Linux 的 OOM Killer 杀死 Java 进程，但 JVM 日志中不一定有 `OutOfMemoryError`，而是在 `dmesg` / 系统日志里看到。

**典型原因**：

- 总体内存占用（JVM 堆 + 直接内存 + 本地库 + 系统 cache 等）超过物理内存 + swap 能承受的上限。
- Docker / 容器环境中 cgroup 内存限制偏小，而 JVM 按为“裸机”来配置。

**处理**：

- 总体评估**容器/机器内存上限**与 JVM 配置（`Xmx`、Metaspace、DirectMemory 等）的关系。
- 在 Kubernetes / Docker 中，设置合理的 `memory limit`，并使用 `-XX:MaxRAMPercentage` 等参数让 JVM 自适应。

------

## 三、通用排查 & 处理流程（可作为线上 OOM checklist）

遇到任何形式的 OOM，大致可以按下面步骤来：

1. **确保 JVM 在 OOM 时生成 dump 和日志**

   ```sh
   -XX:+HeapDumpOnOutOfMemoryError
   -XX:HeapDumpPath=/path/to/dumps
   -XX:+PrintGCDetails
   -XX:+PrintGCDateStamps
   -Xloggc:/path/to/gc.log   # JDK8
   # 或 JDK9+ 的统一日志参数
   ```

2. **先看错误类型和 message**

   - `Java heap space` / `GC overhead limit exceeded` → 堆问题
   - `Metaspace` / `PermGen space` → 类元数据
   - `Direct buffer memory` → 直接内存 / NIO
   - `unable to create new native thread` → 线程 & 栈
   - OS 日志里 OOM Killer → 容器/机器整体内存不够

3. **根据类型选择对应分析工具**

   - 堆 OOM：MAT / VisualVM 分析 heap dump，看**最大占用对象**、**泄漏链**
   - Metaspace：关注 `ClassLoader` 树，检测大量重复的类加载器
   - Direct Memory / Native：使用 `jcmd VM.native_memory summary`（JDK8u40+）
   - 线程问题：
     - `jstack` 看线程数量 & 状态
     - `jconsole` / `VisualVM` 也能看到线程数变化

4. **结合应用特点，做代码级和配置级优化**

   - **代码层面**：缓存策略、集合大小、对象生命周期、线程创建策略等
   - **配置层面**：`-Xmx`, `-Xms`, `-Xss`, `-XX:MaxMetaspaceSize`, `-XX:MaxDirectMemorySize` 等
   - **部署层面**：容器内存 limit、机器规格、服务实例数等

5. **加监控，防止“突然炸锅”**

   - 堆使用率、Metaspace 使用率、GC 时间 & 次数
   - 线程数、直接内存使用、容器/机器整体内存
   - 结合告警：接近阈值时提前发现问题。