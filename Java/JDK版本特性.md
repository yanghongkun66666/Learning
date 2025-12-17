先给个总览：目前业界常说的 **LTS** 版本是 **8 / 11 / 17 / 21 / 25**。([oracle.com](https://www.oracle.com/java/technologies/java-se-support-roadmap.html?utm_source=chatgpt.com))

------

## Java 8（LTS）

- **Lambda / 方法引用 / 默认方法**（函数式写法、接口可演进）([oracle.com](https://www.oracle.com/java/technologies/javase/8-whats-new.html?utm_source=chatgpt.com)) 常用

  - 契机：Java 写回调、策略模式、Comparator、监听器时样板代码爆炸（匿名内部类一大坨），并且难以表达“把行为当参数传”。

    简化：把“实现一个只含一个抽象方法的接口”压缩成一行表达式；方法引用进一步省掉 `(x)->foo(x)` 这种重复。

    好处：代码更短、更可读；配合 Stream 能把集合处理写成声明式流水线；也让并行计算/异步组合更自然。

- **Stream API**（集合流水线处理）常用

  - 契机：集合处理大量 for 循环 + 临时变量 + 易写错（过滤/映射/分组/去重/排序组合起来很臃肿）。

    简化：`filter/map/flatMap/collect/groupingBy` 这类通用操作变成组合式 API。

    好处：表达意图更直接；减少中间容器；更容易做并行（`parallelStream`，尽管要慎用）；减少重复工具代码。

- **新的日期时间 API（java.time）** 

  - 契机：`Date/Calendar` 设计老、易用性差、可变对象线程不安全、时区/格式化坑多。

    简化：用 `LocalDate/LocalDateTime/Instant/ZonedDateTime/Duration/Period` 等更贴合概念的类型。

    好处：不可变、线程安全；时区语义清晰；API 更不容易写出“差一天/差一小时”的 bug。

## Java 9

- **模块系统（JPMS / Project Jigsaw）**：`module-info.java`、依赖/封装更清晰([openjdk.org](https://openjdk.org/jeps/261?utm_source=chatgpt.com))
- **JShell**（交互式 REPL）([oracle.com](https://www.oracle.com/java/technologies/javase/9-new-features.html?utm_source=chatgpt.com))
- 还带来 **HTTP/2 Client（早期）**、统一 JVM 日志等([openjdk.org](https://openjdk.org/projects/jdk9/))

## Java 10

- **var：局部变量类型推断**（更少样板代码）([openjdk.org](https://openjdk.org/projects/jdk/10/?utm_source=chatgpt.com))

  - 契机：泛型/链式调用下类型声明又长又重复（右边已经写得很清楚了，左边再写一遍很啰嗦）。

    简化：把“显而易见”的局部变量类型交给编译器推断。

    好处：减少样板代码、提升可读性（但也可能被滥用导致类型不清晰，所以团队要有规范）。

## Java 11（LTS）

- **标准化 HTTP Client（java.net.http）** ([openjdk.org](https://openjdk.org/projects/jdk/11/))

  - 契机：JDK 自带的 `HttpURLConnection` 体验差；很多项目不得不依赖 Apache HttpClient/OkHttp；HTTP/2、WebSocket 支持不统一。

    简化：用 `HttpClient/HttpRequest/HttpResponse` 提供同步/异步、HTTP/2、WebSocket。

    好处：减少第三方依赖；异步用 `CompletableFuture` 更顺；更现代的协议支持。

- **Flight Recorder（JFR）**（诊断/性能分析）([openjdk.org](https://openjdk.org/projects/jdk/11/))

- **Lambda 参数允许 var** ([openjdk.org](https://openjdk.org/projects/jdk/11/))

- **移除 Java EE / CORBA 模块**（JDK 自带内容更“瘦身”）([openjdk.org](https://openjdk.org/projects/jdk/11/))

## Java 12

- **Switch Expressions（预览）** ([openjdk.org](https://openjdk.org/projects/jdk/12/))
- **Shenandoah GC（实验）** ([openjdk.org](https://openjdk.org/projects/jdk/12/))

## Java 13

- **Text Blocks（预览，多行字符串）** ([openjdk.org](https://openjdk.org/projects/jdk/13/))
- **Switch Expressions（预览延续/增强）** ([openjdk.org](https://openjdk.org/projects/jdk/13/))

## Java 14

- **Switch Expressions 转正（标准特性）** ([openjdk.org](https://openjdk.org/projects/jdk/14/))

  - 契机：switch 语句容易忘写 break、逻辑分散，想返回值还要临时变量。

    简化：`switch (...) { case ... -> ... }` 直接产生值，`yield` 处理复杂分支。

    好处：减少 bug（break 漏写）；分支更紧凑；更贴近函数式风格。

- **Records（预览）** ([openjdk.org](https://openjdk.org/projects/jdk/14/))

  - 契机：大量“数据载体类”只有字段+构造+getter+equals/hashCode/toString，写起来烦、容易漏。

    简化：`record User(String name, int age) {}` 一句声明不可变数据模型。

    好处：少写大量样板；语义明确（就是数据）；更适合 DTO/消息/配置等场景。

- **更友好的 NullPointerException 提示** ([openjdk.org](https://openjdk.org/projects/jdk/14/))

## Java 15

- **Text Blocks 转正**（多行字符串“”“） ([openjdk.org](https://openjdk.org/projects/jdk/15/))

  - 契机：SQL/JSON/HTML 这种多行文本在 Java 里拼接、转义字符满天飞，可读性差。

    简化：原样写多行文本，少转义。

    好处：字符串更接近原文；减少错误（转义、换行、缩进）；模板文本更舒服。

- **Sealed Classes（预览）**、**Hidden Classes** ([openjdk.org](https://openjdk.org/projects/jdk/15/))

## Java 16

- **Records 转正**、**instanceof 模式匹配转正** ([openjdk.org](https://openjdk.org/projects/jdk/16/))

  - 契机：类型判断后还要强转：`if (obj instanceof String) { String s=(String)obj; }`

    简化：`if (obj instanceof String s) { ... }`

    好处：更少样板、更少强转错误；为后续更强的模式匹配铺路。

- **Unix-Domain Socket Channels**、打包工具（jpackage）等也常用([openjdk.org](https://openjdk.org/projects/jdk/16/))

## Java 17（LTS）

- **Sealed Classes 转正** ([openjdk.org](https://openjdk.org/projects/jdk/17/?utm_source=chatgpt.com))
- **switch 的模式匹配（预览）** ([openjdk.org](https://openjdk.org/projects/jdk/17/?utm_source=chatgpt.com))
- **更强的 JDK 内部封装**（对反射/内部 API 依赖更敏感）([openjdk.org](https://openjdk.org/projects/jdk/17/?utm_source=chatgpt.com))

## Java 18

- **默认字符集 UTF-8** ([openjdk.org](https://openjdk.org/projects/jdk/18/))
- **简单静态文件 Web Server（jwebserver）** ([openjdk.org](https://openjdk.org/projects/jdk/18/))
- **Javadoc 代码片段 @snippet** ([openjdk.org](https://openjdk.org/projects/jdk/18/))

## Java 19

- **Virtual Threads（预览）** ([openjdk.org](https://openjdk.org/projects/jdk/19/))
- **FFM（Foreign Function & Memory）（预览）** ([openjdk.org](https://openjdk.org/projects/jdk/19/))
- **Structured Concurrency（孵化）** ([openjdk.org](https://openjdk.org/projects/jdk/19/))

## Java 20

- 上面并发/FFM/模式匹配继续迭代（多为第 N 次预览/孵化）([openjdk.org](https://openjdk.org/projects/jdk/20/))

## Java 21（LTS）

- **Virtual Threads 转正（JEP 444）**：并发编程体验大升级([openjdk.org](https://openjdk.org/projects/jdk/21/?utm_source=chatgpt.com))

  - 契机：传统线程“一请求一线程”扩展性差（线程栈、上下文切换成本高）；而纯异步又把代码写得很复杂（回调地狱/链式 future）。

    简化：用“像写同步一样”的阻塞代码获得接近异步的伸缩性（尤其 I/O 密集服务）。

    好处：并发量上去更容易；代码仍然直观；减少为了性能被迫写复杂异步框架的场景（当然也要理解阻塞点与线程绑定/pinning等细节）。

- **Sequenced Collections**（新增“首/尾”语义接口）([openjdk.org](https://openjdk.org/projects/jdk/21/?utm_source=chatgpt.com))

- **Record Patterns**、**switch 模式匹配**（数据解构更顺）([openjdk.org](https://openjdk.org/projects/jdk/21/?utm_source=chatgpt.com))

- **Generational ZGC** ([openjdk.org](https://openjdk.org/projects/jdk/21/?utm_source=chatgpt.com))

## Java 22

- **FFM API 转正**（与 native 交互更正规）([openjdk.org](https://openjdk.org/projects/jdk/22/))
- **Stream Gatherers（预览）**、**Scoped Values（预览）**、**Structured Concurrency（预览）**继续推进([openjdk.org](https://openjdk.org/projects/jdk/22/))

## Java 23

- **Markdown 文档注释** ([openjdk.org](https://openjdk.org/projects/jdk/23/))
- **Module Import Declarations（预览）**、**Structured Concurrency / Scoped Values（继续预览）**([openjdk.org](https://openjdk.org/projects/jdk/23/))
- **ZGC 默认启用分代模式** ([openjdk.org](https://openjdk.org/projects/jdk/23/))

## Java 24

- **Stream Gatherers / Class-File API**（相关能力完善）([openjdk.org](https://openjdk.org/projects/jdk/24/))
- **虚拟线程同步但不“pinning”**（并发细节优化）([openjdk.org](https://openjdk.org/projects/jdk/24/))
- **Security Manager 永久禁用**（安全模型进一步收敛）([openjdk.org](https://openjdk.org/projects/jdk/24/))

## Java 25（LTS，2025-09-16 GA）

- **Scoped Values 转正**（替代 ThreadLocal 的重要方向）([openjdk.org](https://openjdk.org/projects/jdk/25/))

  - 契机：线程上下文传递（traceId、用户信息等）常用 ThreadLocal，但在高并发/线程复用/虚拟线程时代，ThreadLocal 容易造成泄漏、污染、传播困难。

    简化：用“有作用域的上下文值”在调用链中安全传递，生命周期由作用域控制。

    好处：比 ThreadLocal 更可控、更容易推理；与虚拟线程/结构化并发更契合；降低上下文泄漏类问题。

- **Structured Concurrency（第 5 次预览）**继续成熟([openjdk.org](https://openjdk.org/projects/jdk/25/))

- **Module Import Declarations**、**Compact Source Files & Instance Main**、**AOT 相关改进**等([openjdk.org](https://openjdk.org/projects/jdk/25/))

- 25 被多数厂商视作 LTS 版本([openjdk.org](https://openjdk.org/projects/jdk/25/))

