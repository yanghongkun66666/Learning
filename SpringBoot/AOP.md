# AOP定义

AOP 的标准中文说法是 **“面向切面编程”（Aspect-Oriented Programming）**。

> **AOP = 一种编程思想，用来把“横切关注点”（比如日志、事务、安全校验、监控这些跟业务无关，但到处都要写的代码）从业务代码中抽出来，集中在“切面”里统一处理。**

------

## 1. 关键几个词

- **横切关注点（Cross-cutting Concern）**
   指的是：  各自细节实现
  - 日志记录
  - 权限校验
  - 事务管理 更细节一点呢
  - 性能监控 ？
  - 缓存
     这些逻辑**不是某个业务独有**，而是“横着切过很多模块的方法、类”的通用逻辑。
- **切面（Aspect）**
   就是把上面这些“横切关注点”**单独装进一个类/模块里**，
   然后声明：
  - 在哪些地方（哪些方法）
  - 以什么方式（前置、后置、环绕）
     把这些逻辑“织入”进去。

------

## 2. 一句话类比

如果正常面向对象编程（OOP）是：

> “把不同业务职责封装成不同类”

那 **AOP** 是：

> “把横跨很多类的 *同一种* 通用逻辑（比如日志、事务），从这些类里拧出来，单独放到一个切面里，再在运行时自动给这些类的方法外面“套一层壳”来统一处理。”

------

## 3. 用 Spring 场景再说一遍（更贴地）

在 Spring 里：

- 你写业务方法：

  ```java
  public void createOrder() { ... }
  ```

- 正常情况下，要加事务 / 日志，你需要在每个方法前后写一大堆重复代码。

- 用 AOP 后，你可以写一个切面：

  ```java
  @Aspect
  class LogAspect {
      @Before("execution(* com.xxx.service..*(..))")
      public void beforeLog() { ... }
  
      @AfterReturning(...)
      public void afterLog() { ... }
  }
  ```

Spring 会在运行时：

- 用代理 / 动态字节码等技术，把 `createOrder()` 的调用**包上一层**；
- 在调用前后自动帮你调用 `beforeLog()` / `afterLog()`。

所以你业务方法本身还是干干净净，只做业务逻辑，
 而日志、事务、权限这些横切逻辑，全部集中由 AOP 统一管理。

------

### 极简总结句

> **AOP（面向切面编程）= 把“横跨很多模块的通用功能”（日志、事务、安全等）从业务代码里抽出来，放到独立的“切面”里，在运行时自动织入，从而减少重复代码、让业务逻辑更干净。**

- 





# AOP中重要术语



AOP 里最常被问、也最容易搞混的术语就是这几个：

> **Aspect、JoinPoint、Pointcut、Advice、Target Object、Proxy、Weaving（+ 有时会提 Introduction）**

我按“你写业务代码时会怎么遇到它们”的顺序讲一遍。

------

## 1. Aspect（切面）

**是什么：**

- 就是一段“横切逻辑”的封装。
- 例如：日志切面、事务切面、权限切面、监控切面……

**在 Spring 里长这样：**

```java
@Aspect
class LogAspect {
    @Before("execution(* com.xxx.service..*(..))")
    public void doLog() { ... }
}
```

可以简单记成：

> **Aspect = 横切关注点 + 告诉系统“要织到哪些地方去”的配置集合。**

------

## 2. JoinPoint（连接点）

**是什么：**

- 程序运行过程中，**可以被 AOP “插入代码”的具体位置**。
- 在 Spring AOP（基于代理）里，几乎等价于：
  - **被代理对象中，每一个可被拦截的方法的“方法调用点”**。

举例：

```java
userService.createUser()
userService.deleteUser()
```

这些方法被代理后，每一次方法调用，就是一个 JoinPoint。

可以记成：

> **JoinPoint = 可以被切面“卡住一下”的那一瞬间（方法调用、抛异常等的位置）。**

------

## 3. Pointcut（切点）

**是什么：**

- 是**一组 JoinPoint 的筛选条件**。
- 你不用关心所有 JoinPoint，只想选一部分，比如：
  - service 包下面所有 public 方法；
  - 某个注解标记的方法；
  - 名字以 `save*` 开头的方法。

在 Spring 里典型写法：

```java
@Pointcut("execution(* com.xxx.service..*(..))")
public void serviceMethods() {}
```

可以记成：

> **Pointcut = 我想让切面生效的“方法集合”的表达式。**

------

## 4. Advice（通知）

**是什么：**

- 真正要“插入执行”的那段代码逻辑。
- 比如“方法执行前打印日志”“方法执行后提交事务”“抛异常时打告警”。

在 Spring 里，有几种常见类型：

- `@Before`：前置通知，方法执行前
- `@After`：后置通知（无论成功/异常）
- `@AfterReturning`：返回后
- `@AfterThrowing`：抛异常后
- `@Around`：环绕通知（方法前后都能插逻辑，最灵活）

例如：

```java
@Around("serviceMethods()")
public Object around(ProceedingJoinPoint pjp) throws Throwable {
    long start = System.currentTimeMillis();
    Object ret = pjp.proceed();  // 调原方法
    long cost = System.currentTimeMillis() - start;
    System.out.println("耗时: " + cost + "ms");
    return ret;
}
```

可以记成：

> **Advice = 在某个 Pointcut 对应的方法执行“之前 / 之后 / 替代”的那段实际代码。**

------

## 5. Target Object（目标对象）

**是什么：**

- 被 AOP 增强的那个“原始对象”，也就是你写的业务类实例：
  - `UserServiceImpl`、`OrderServiceImpl` 之类。

AOP 不会直接改变这个对象，而是：

> 用一个 **Proxy（代理对象）** 把它“包起来”。

可以记成：

> **Target Object = 你真实写的业务类对象，AOP 要在它的方法前后做文章。**

------

## 6. Proxy（代理对象）

**是什么：**

- 真正暴露给你使用的，不是原始的 `UserServiceImpl`，而是一个 **代理对象**：
  - 它实现了同样的接口（或继承同样的类），
  - 但是方法调用会先经过 AOP 逻辑（Advice），然后再去调 Target Object。

Spring AOP 常见两种代理方式：

- JDK 动态代理：基于接口
- CGLIB：基于子类继承

你注入时：

```java
@Autowired
private UserService userService;
```

得到的大多是 **代理对象**，不是你 new 的原始实现类。

可以记成：

> **Proxy = 带有 AOP 功能的“中间人”，对外表现像原对象，对内多了一层拦截。**

------

## 7. Weaving（织入）

**是什么：**

- 把 “切面（Aspect）” 的逻辑 **编织进** 目标代码的过程，就叫“织入”。

根据“在什么时候织入”，一般分：

- **编译期织入**（AspectJ 的某些模式）：编译 `.java` → `.class` 时就把字节码改掉。
- **类加载时织入**：通过特殊类加载器，在字节码加载进 JVM 时织入。
- **运行时织入**（Spring AOP）：通过动态代理 / CGLIB，在运行时创建代理对象。

可以记成：

> **Weaving = 把切面真正“装到”（hook 到）目标方法上的过程。**

------

## 8. （附加）Introduction（引介 / 引入）

有时也会提到：

- **Introduction**：给现有类 **动态增加属性 / 方法 / 实现接口** 的一种 AOP 功能。
- 比如：不改源代码的情况下，让某个类“看起来”多实现了一个接口，多了一些方法。

Spring 里比较少自己玩这个，大多数使用场景还是 JoinPoint + Advice。

------

### 最后帮你压缩成一句话：

> - **Aspect**：装横切逻辑的“切面类”；
> - **JoinPoint**：可以被插入的执行点（方法调用等）；
> - **Pointcut**：从所有 JoinPoint 里挑出“我要拦”的那一批；
> - **Advice**：在这些点前 / 后 / 周围真正执行的那段代码；
> - **Target Object**：被拦截和增强的原始业务对象；
> - **Proxy**：对外暴露的代理对象，调用方法时会先跑 AOP 逻辑；
> - **Weaving**：把切面和目标对象“编织”在一起的过程。

如果你想，我可以用一个具体的 Spring AOP 小例子（带注解、代理类调用链）把这些术语标在代码上，让你一眼看到“这里是 JoinPoint，这里是 Advice，这里是 Proxy”。