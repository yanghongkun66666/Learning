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





# AOP中重要术语



AOP 里最常被问、也最容易搞混的术语就是这几个：

> **Aspect、JoinPoint、Pointcut、Advice、Target Object、Proxy、Weaving（+ 有时会提 Introduction）**

我按“你写业务代码时会怎么遇到它们”的顺序讲一遍。

------

## 1. Aspect（切面）

**是什么：**

- 就是一段“横切逻辑”的封装。（通用逻辑）
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

> **Pointcut = 我想让切面生效的“方法集合”的表达式。**（一组连接点的集合）

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
  - 它实现了同样的接口：JDK（或继承同样的类：CGLIB子类方式），
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





# 底层实现

本质上，AOP 底层就是在**“某个方法调用前后，把一段额外代码（advice）插进去”**。
 实现上主要有两大路子：

1. **代理方式：靠“代理对象”接管方法调用，再自己决定先干啥再调原方法**（Spring AOP）
2. **织入方式：直接改 class 字节码，把切面逻辑“写进方法体里”**（AspectJ 编译期 / 类加载期）

你说“越底层越好”，那我就从这两个方向一路扒到 JVM / 字节码级别给你讲。

------

## 一、AOP 需要解决的两个根问题

任何 AOP 框架底层都在解决两件事：

1. **在哪些地方插？（JoinPoint / Pointcut）**
   - 也就是：哪些类 / 哪些方法 / 哪些调用，是“我要拦截的”？
   - Spring：`execution(* com.xx.service..*(..))` → 解析成 MethodMatcher、ClassFilter。
2. **插进去之后怎么跑？（Advice 执行模型）**
   - 调用顺序：前置 → 原方法 → 后置 / 异常 → 最终。
   - 多个切面时如何排顺序？（拦截器链 / 责任链模式）

底层就是：**把原本“直接调用目标方法”的路径，变成“先走一圈拦截器，再调用目标方法，再回来”**。

------

## 二、Spring AOP：基于代理的实现（最常用）

Spring AOP 不改字节码（默认情况下），它做的是：

> **用一个代理对象代替原来的 bean，所有调用都先打到代理上。**

### 1. Bean 创建 & 代理生成

大致流程（简化）：

1. Spring 容器启动，扫描到你的这些东西：
   - 普通 Bean（`UserServiceImpl`）
   - 切面 Bean（`@Aspect` 标注的类）
2. AOP 相关的 `BeanPostProcessor`（比如 `AnnotationAwareAspectJAutoProxyCreator`）在 Bean 初始化后介入：
   - 分析：这个 Bean 的方法有没有匹配到某些 Pointcut？
   - 如果匹配到了：**不给你原始 Bean，而是给你一个代理对象**。
3. 代理对象里会保存：
   - 原始目标对象（target）
   - 一串针对这个目标对象匹配到的拦截器链（advisors → interceptors）

**关键点：你在代码里注入的 userService，其实是一个 Proxy。**

------

### 2. JDK 动态代理路径（有接口时）

如果目标类**实现了接口**，Spring 优先用 **JDK 动态代理**：

```java
UserService proxy = (UserService) Proxy.newProxyInstance(
        targetClass.getClassLoader(),
        new Class<?>[]{UserService.class},
        invocationHandler  // Spring 实现的一个 InvocationHandler
);
```

用户代码调用：

```java
userService.createOrder();
```

底层变成（概念上）：

```java
// proxy 是 JDK 动态生成的类，方法体类似：
public void createOrder() {
    // 统一进 InvocationHandler
    return invocationHandler.invoke(this, method_createOrder, args);
}
```

`InvocationHandler.invoke` 的实现（Spring 里是 `JdkDynamicAopProxy`）核心是：

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    // 1. 找出这个 method 对应的拦截器链（advice 列表）
    List<MethodInterceptor> chain = getInterceptors(method, targetClass);

    // 2. 把 target + method + args + chain 封装成一个 MethodInvocation
    MethodInvocation mi = new ReflectiveMethodInvocation(
            proxy, target, method, args, chain
    );

    // 3. 触发拦截器链
    return mi.proceed();
}
```

`proceed()` 的内部就是 **责任链模式**：

```java
public Object proceed() throws Throwable {
    if (this.currentInterceptorIndex == this.interceptors.size() - 1) {
        // 链表走完，调用真正目标方法
        return method.invoke(target, args);
    }

    this.currentInterceptorIndex++;
    MethodInterceptor interceptor = this.interceptors.get(currentInterceptorIndex);
    // 让下一个拦截器执行，拦截器内部再调用 mi.proceed()
    return interceptor.invoke(this);
}
```

一个典型的 `@Around` 通知对应的拦截器 pseudo-code：

```java
class AroundAdviceInterceptor implements MethodInterceptor {
    public Object invoke(MethodInvocation mi) throws Throwable {
        // 前置逻辑
        long t1 = System.currentTimeMillis();

        Object ret;
        try {
            ret = mi.proceed(); // 调下一个拦截器 or 原方法
            // 返回后逻辑
            long t2 = System.currentTimeMillis();
            log("cost", t2 - t1);
        } catch (Throwable ex) {
            // 异常逻辑
            log("error", ex);
            throw ex;
        }

        return ret;
    }
}
```

**这一套就是 Spring AOP 在 JDK 动态代理下的底层调用路径。**

------

### 3. CGLIB 代理路径（没有接口或强制使用 CGLIB 时）

如果目标类没有实现接口，或者你配置了 `proxyTargetClass = true`，Spring 使用 CGLIB：

CGLIB 怎么做？

> **在运行时生成一个“目标类的子类”，在子类里 override 需要增强的方法，在 override 的方法里做拦截，然后再去调 super。**

生成的大致结构（伪代码）：

```java
class UserServiceImpl$$EnhancedByCGLIB extends UserServiceImpl {

    MethodInterceptor callback;

    @Override
    public void createOrder() {
        // 进入统一拦截逻辑
        callback.intercept(this, method_createOrder, args, methodProxy_createOrder);
    }
}
```

`MethodInterceptor` 的实现思路和前面的 `InvocationHandler` 差不多：

```java
public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
    // 构造 MethodInvocation，带上拦截器链
    MethodInvocation mi = new CglibMethodInvocation(obj, target, method, args, interceptors, proxy);
    return mi.proceed();
}
```

`mi.proceed()` 同样是责任链，最后落到：

```java
// 调用原始父类方法（真正业务逻辑）
return methodProxy.invokeSuper(obj, args);
```

CGLIB 底层是通过 ASM 直接生成字节码的：

- 运行时动态拼一个 `class` 文件结构
- 定义方法、字段、构造器
- JVM 加载这个新类 → 生成子类实例

从 JVM 角度看，它就是一个普通类；从 Java 角度看，你拿到的是这个子类实例，而不是原类实例。

------

## 三、更底层一点：JDK 动态代理 / CGLIB 实际做了什么？

### 1. JDK 动态代理的本质

`Proxy.newProxyInstance` 做的事情：

1. **动态生成一个“实现了指定接口集合”的类：**

   类似：

   ```java
   public final class $Proxy0 extends Proxy implements UserService, OtherInterface {
       public void createOrder() {
           try {
               return h.invoke(this, method_createOrder, null);
           } catch (Throwable e) { ... }
       }
       ...
   }
   ```

   - `h` 就是 `InvocationHandler`。
   - `method_createOrder` 是一个 `java.lang.reflect.Method` 常量。

2. 把这个类的 `.class` 字节码在内存中定义加载（`defineClass`）。

3. new 出一个对象，构造时注入你传进去的 `InvocationHandler h`。

所以本质就是：

> **代理类的每个方法体里，唯一做的事就是转发到 InvocationHandler.invoke()，再由你的 AOP 框架决定怎么调拦截器/目标对象。**

------

### 2. CGLIB 的本质

CGLIB = 基于 ASM 的字节码生成工具。

做的事比 JDK Proxy 多一步：

1. 继承目标类 `class Proxy extends TargetClass`；

2. 对每个要增强的方法 override：

   ```java
   @Override
   public void foo(int a, String b) {
       methodInterceptor.intercept(this, method_foo, new Object[]{a, b}, methodProxy_foo);
   }
   ```

3. `methodProxy_foo.invokeSuper(obj, args)` 里调用的是父类（原类）的实现：

   - ASM 直接在字节码里把调用编成类似 `invokespecial TargetClass.foo`。

这样，相比 JDK 动态代理：

- **不需要依赖接口**；
- 方法调用时，可以少一层反射（因为是直接字节码调用 `invokeSuper`）。

------

## 四、AspectJ：真正“改字节码”的 AOP

Spring AOP 是“**调用时绕一圈**”；
 AspectJ 更狠，是直接“**改方法体**”。

### 1. 编译期织入（Compile-time Weaving）

用 `ajc` 编译器编译代码时：

1. 把你的业务类 `.java` 编译成 `.class`；
2. 同时解析切面（@Aspect 或 .aj 文件）；
3. 在匹配的 JoinPoint（比如某方法前）**直接插入字节码**：

原业务方法：

```java
void foo() {
    System.out.println("foo");
}
```

织入后的伪字节码逻辑：

```java
void foo() {
    Aspect.beforeFoo(this, ...);  // 前置通知
    try {
        System.out.println("foo"); // 原逻辑
        Aspect.afterReturning(this, ...);
    } catch (Throwable t) {
        Aspect.afterThrowing(this, t, ...);
        throw t;
    } finally {
        Aspect.after(this, ...);
    }
}
```

优点：

- 运行时 **没有代理**，调用开销就是普通方法调用；
- join point 不仅可以是“方法调用”，还能是“构造器调用、字段访问”等 Spring AOP 支持不了的点。

缺点：

- 编译链路变复杂（要用 ajc 或特殊插件）；
- 类加载/部署也要考虑 AspectJ 的织入器。

### 2. 类加载期织入（Load-time Weaving）

通过 Java Agent（`-javaagent:aspectjweaver.jar`）+ Instrumentation：

1. JVM 加载一个类的字节码时，先交给 AspectJ 的 `ClassFileTransformer`；
2. Transformer 修改字节码，把 advice 插进去；
3. 再交还 JVM 链接、初始化。

这就是“运行中改 class”的方式，本质还是：**改字节码，在方法体里插代码**。

------

## 五、站在 JVM 视角：一次带 AOP 的方法调用长啥样？

以 Spring AOP + JDK 代理 为例，从 JVM 的角度看调用栈（简化）：

你代码写的：

```java
orderService.createOrder("xxx");
```

真实调用栈可能是：

```text
main()
  -> Controller.create()
      -> proxy(OrderService).createOrder("xxx")           // 调的是代理对象
          -> JdkDynamicAopProxy.invoke(proxy, method, args)
              -> MethodInvocation.proceed()
                  -> AroundInterceptor1.invoke(mi)
                      -> AroundInterceptor2.invoke(mi)
                          ...
                              -> method.invoke(target, args)  // 反射调用真实 OrderServiceImpl
```

如果是 CGLIB：

```text
main()
  -> Controller.create()
      -> OrderServiceImpl$$EnhancerByCGLIB.createOrder("xxx")
          -> MethodInterceptor.intercept(this, method, args, methodProxy)
              -> MethodInvocation.proceed()
                  -> ...
                      -> methodProxy.invokeSuper(this, args) // ASM 生成的字节码调用父类方法
```

如果是 AspectJ 编译期织入：

```text
main()
  -> Controller.create()
      -> OrderServiceImpl.createOrder("xxx")
          -> // 已被织入的字节码：
             Aspect.before(...)
             try {
                 // 原始业务代码
             } catch(Exception e) {
                 Aspect.afterThrowing(...)
             } finally {
                 Aspect.after(...)
             }
```

------

## 六、你可以自己写一个极简版 AOP（JDK 动态代理示例）

下面是一个“用最少代码模拟 Spring AOP 调用过程”的版本：

```java
import java.lang.reflect.*;

interface UserService {
    void createUser(String name);
}

class UserServiceImpl implements UserService {
    public void createUser(String name) {
        System.out.println("真正创建用户：" + name);
    }
}

// 一个简单的环绕“切面”
class LogAroundHandler implements InvocationHandler {
    private final Object target;

    public LogAroundHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 相当于 @Before
        System.out.println("[AOP-BEFORE] 调用方法: " + method.getName());

        Object ret;
        try {
            // 调用目标方法，相当于 mi.proceed()
            ret = method.invoke(target, args);
            // 相当于 @AfterReturning
            System.out.println("[AOP-AFTER-RETURN] 方法正常返回");
        } catch (Throwable ex) {
            // 相当于 @AfterThrowing
            System.out.println("[AOP-THROW] 方法抛异常: " + ex);
            throw ex;
        } finally {
            // 相当于 @After
            System.out.println("[AOP-AFTER] 方法结束");
        }
        return ret;
    }
}

public class Demo {
    public static void main(String[] args) {
        UserService target = new UserServiceImpl();

        // 生成代理对象
        UserService proxy = (UserService) Proxy.newProxyInstance(
                target.getClass().getClassLoader(),
                new Class[]{UserService.class},
                new LogAroundHandler(target)
        );

        proxy.createUser("Alice");
    }
}
```

运行输出大致是：

```text
[AOP-BEFORE] 调用方法: createUser
真正创建用户：Alice
[AOP-AFTER-RETURN] 方法正常返回
[AOP-AFTER] 方法结束
```

这就是 **Spring AOP 做的事情的最小缩影**，只是 Spring 把：

- 代理的创建集成到了容器生命周期；
- 拦截器链支持各种类型的 Advice（事务、缓存、权限、日志等）；
- Pointcut 决定哪些方法要套这层壳；
- 再加上各种排序、异常处理、配置解析。

------

### 最后帮你压缩成一句“底层版总结”

> Java 里的 AOP，底层要么是 **“代理接管方法调用，再走一圈拦截器（Spring AOP：JDK 动态代理 / CGLIB 子类 + ASM）”**，
>  要么是 **“直接改 class 字节码，把 advice 逻辑织进方法体（AspectJ 编译期 / 类加载期 weaving）”**。
>  而真正的“切点判断、通知顺序、调用链推进”本质上是几个对象模型：Pointcut → Advisor → InterceptorChain → `proceed()`，再加上 JVM 提供的反射 / subclass / 字节码重写能力。

如果你想，我可以专门画一张“Spring AOP 调用链类图”（`ProxyFactory`、`AdvisedSupport`、`JdkDynamicAopProxy`、`MethodInterceptor` 等）帮你把类之间的关系串一下。







# 常见应用场景

在大厂里，AOP 不是拿来“耍花活”的，而是**专门解决那种：到处都要写、又不想写在业务里的通用逻辑**。
 最最常用的场景，基本就围绕这几类：**日志、监控/埋点、事务、安全、缓存、审计**，再加一点高级用法。

我按“出现频率从高到低”给你讲，每个都带一点 Spring 风格的例子，方便你有画面感。

------

## 1. 统一日志（请求日志 / 方法调用日志 / 异常日志）

**这是绝对 Top1 场景。**

常见需求：

- 所有 Controller 的入参/出参日志
- 所有 Service 方法异常时打印统一格式日志
- 关键接口的耗时、调用链路

不用 AOP 的话，每个方法前后都 `log.info()` 一遍，既丑又难维护。

**AOP 做法：**

- 定义一个切面，匹配一批方法（比如 `@RestController` 下所有方法）
- 在 `@Around` / `@AfterThrowing` 里统一记录：
  - 接口名
  - 入参
  - 返回值
  - 耗时
  - 异常信息

示意代码（简化版）：

```java
@Aspect
@Component
public class WebLogAspect {

    @Around("execution(* com.xxx.api..*(..))")
    public Object logAround(ProceedingJoinPoint pjp) throws Throwable {
        String method = pjp.getSignature().toShortString();
        Object[] args = pjp.getArgs();

        long start = System.currentTimeMillis();
        try {
            Object result = pjp.proceed();
            long cost = System.currentTimeMillis() - start;

            // 这里可以只打摘要，不要直接打印大对象
            log.info("method={}, args={}, cost={}ms, result={}", method, Arrays.toString(args), cost, result);
            return result;
        } catch (Throwable e) {
            long cost = System.currentTimeMillis() - start;
            log.error("method={}, args={}, cost={}ms, ex={}", method, Arrays.toString(args), cost, e.toString(), e);
            throw e;
        }
    }
}
```

**特点：**

- 不改任何业务代码，只要方法能被切中，就会自动打统一格式日志。
- 换日志格式 / 打点内容，只改切面即可。

------

## 2. 性能监控 & 埋点（Metrics / Tracing）

这是和日志并列的常用场景，大厂非常爱用：

- 统计某类接口的**平均耗时、TP90、TP99**
- 统计 QPS、错误率
- 给 Prometheus / Micrometer / Cat / Zipkin / SkyWalking 之类喂数据

**AOP 做法：**

- `@Around` 切一波关键方法：
  - 接口层
  - 数据访问层
  - 某些核心服务方法
- 调用前记录时间，调用后打点到 metrics 系统：

```java
@Aspect
@Component
public class MetricAspect {
    @Around("@annotation(com.xxx.Metric)")
    public Object recordMetric(ProceedingJoinPoint pjp) throws Throwable {
        String name = pjp.getSignature().toShortString();
        long start = System.nanoTime();
        try {
            Object result = pjp.proceed();
            long cost = System.nanoTime() - start;
            Metrics.timer("service.method.cost", "method", name)
                   .record(cost, TimeUnit.NANOSECONDS);
            return result;
        } catch (Throwable e) {
            Metrics.counter("service.method.error", "method", name).increment();
            throw e;
        }
    }
}
```

**为什么 AOP 很适合：**

- 监控是标准的**横切关注点**：几乎所有模块都要，但不想写在业务里；
- 动态决定：切哪些方法 → 哪些方法就有监控数据。

------

## 3. 事务管理（尤其是 Spring 声明式事务）

**这是你“每天在用、可能没意识到自己在用 AOP” 的典型场景。**

`@Transactional` 本质上就是：

> 用 AOP 代理把方法包起来，在调用前后统一处理事务逻辑。

简化后的过程：

1. Spring 启动时，发现某个 Bean 的方法有 `@Transactional`

2. 它不会直接给你这个 Bean，而是给你一个代理（加了事务拦截器）

3. 调用时变成：

   ```text
   事务切面.before() -> 调原方法 -> 事务切面.after()/afterThrowing()
   ```

伪代码：

```java
@Aspect
@Component
public class TxAspect {

    @Around("@annotation(org.springframework.transaction.annotation.Transactional)")
    public Object aroundTx(ProceedingJoinPoint pjp) throws Throwable {
        TransactionStatus status = txManager.begin(); // 开启事务
        try {
            Object ret = pjp.proceed(); // 调业务方法
            txManager.commit(status);   // 提交
            return ret;
        } catch (Throwable e) {
            txManager.rollback(status); // 回滚
            throw e;
        }
    }
}
```

**大厂里：99% 的事务管理都是靠这种代理 + AOP 做的**，只是 Spring 替你封装好了。

------

## 4. 权限校验 / 鉴权（Security）

常见需求：

- 某个接口需要管理员权限
- 某个方法只能本人操作
- 检查 token / 角色 / 数据权限

不用 AOP 会变成：

```java
if (!checkPermission(...)) throw new NoAuthException();
```

在每个方法里写一遍，很烦。

**AOP 做法：**

- 自定义注解，如 `@RequireRole("ADMIN")`
- 定义切面，切这些带注解的方法：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface RequireRole {
    String value();
}
@Aspect
@Component
public class AuthAspect {

    @Before("@annotation(requireRole)")
    public void checkRole(RequireRole requireRole) {
        String need = requireRole.value();
        String current = AuthContext.getCurrentUserRole();
        if (!need.equals(current)) {
            throw new AccessDeniedException("need role: " + need);
        }
    }
}
```

**优势：**

- 业务方法里只要写注解：`@RequireRole("ADMIN")`
- 实际逻辑集中在切面里统一实现和维护。

在大厂的网关层 / BFF 层 / 内部服务里，这类权限 AOP 非常常见。

------

## 5. 缓存（Cache）—— 读多写少的查询服务里很常见

比如你有个查询方法：

```java
public UserDTO getUserDetail(Long userId)
```

需求：

- 先查缓存（Redis、内存）
- 缓存 miss 再查 DB，并写回
- 方法本身不想写一堆 `redis.get/set`

可以用 Spring Cache（本质也是 AOP）：

```java
@Cacheable(cacheNames = "userDetail", key = "#userId")
public UserDTO getUserDetail(Long userId) {
    return userRepository.find(userId);
}
```

背后是一个 Cache 切面：

1. 方法调用前：
   - 根据 `cacheNames + key` 查缓存
   - 命中 → 直接返回缓存结果，不执行原方法
2. 未命中：
   - 执行原方法拿到结果
   - 写入缓存
   - 返回

你也可以自己写 AOP+注解做分布式缓存、热点缓存、短期缓存等等。

------

## 6. 参数校验 / 防御性检查 / 统一异常转换

比如：

- 入参为空 / 不合法，要抛业务异常
- 日志里要统一打印哪些非法参数
- 除了业务异常外，其它异常统一转成 `ErrorResponse`

**AOP 做法：**

- 切 Controller 层方法：
  - 在 `@Before` 检查参数是否符合某些规则；
  - 发现问题统一抛自定义异常；
- 切 Service 层方法：
  - 拦截抛出的异常，转成标准错误码，打监控日志等。

例子（参数非空校验注解）：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
public @interface NotBlankParam {
    String name();
}
@Aspect
@Component
public class ParamCheckAspect {

    @Before("execution(* com.xxx.service..*(..)) && args(..)")
    public void check(JoinPoint jp) {
        Object[] args = jp.getArgs();
        // 通过反射拿参数上的 @NotBlankParam，做统一校验
        // 不合法就抛 BusinessException
    }
}
```

------

## 7. 审计日志（谁在什么时候对什么做了什么）

例如：

- 记录“谁改了订单状态”
- “谁修改了配置中心某个配置”
- 数据安全合规要求的操作日志

**AOP 做法：**

- 在写操作（update / delete / 特定接口）方法上加一个审计注解；
- 切面里：
  - 拿当前登录用户信息
  - 拿方法名、入参、执行结果
  - 按统一格式写到审计表 / 审计日志系统

这样即使今后变更审计字段，只改切面逻辑即可。

------

## 8. 分布式锁 / 幂等控制（稍微进阶一点，但也挺常用）

比如：

- 某个方法不能并发执行（同一个 key 下）
- 需要基于 Redis/Zookeeper/数据库实现分布式锁

**AOP 做法：**

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface WithDistributedLock {
    String key();
    long expire() default 30_000;
}
@Aspect
@Component
public class DistributedLockAspect {

    @Around("@annotation(lockAnno)")
    public Object around(ProceedingJoinPoint pjp, WithDistributedLock lockAnno) throws Throwable {
        String key = parseKey(lockAnno.key(), pjp.getArgs());
        if (!lockClient.tryLock(key, lockAnno.expire())) {
            throw new BusyException("try later");
        }
        try {
            return pjp.proceed();
        } finally {
            lockClient.unlock(key);
        }
    }
}
```

业务里只要加：

```java
@WithDistributedLock(key = "'order:'+ #orderId")
public void handleOrder(Long orderId) { ... }
```

逻辑就被套上了“先加锁再执行”的模板。

------

## 9. 多租户 / 数据隔离（进阶）

一些大厂的 SaaS 系统里：

- 同一套代码支撑多个租户
- 需要在所有查询、更新操作前自动加上 “tenantId = 当前租户” 这样的条件

**AOP 做法：**

- 拦截 Repository / Mapper 调用：
  - 在 SQL 生成前自动拼上租户条件
- 或者拦截 Service 层方法：
  - 自动在 `ThreadLocal` 里传递当前租户信息，底层使用时直接拿

这个一般会结合 MyBatis 拦截器、JPA 的 Filter 一起用。

------

## 最后，给你一个“简短记忆版总结”（可以说给面试官听）

> 在大厂日常开发里，AOP 最常用在这些地方：
>  1）接口和服务的**统一日志**（入参/出参/异常）；
>  2）**性能监控和埋点**，统计耗时、QPS、错误率；（框架？skywalking？）
>  3）**声明式事务**（`@Transactional` 背后就是 AOP 代理）；
>  4）**权限 / 鉴权**（统一做 `@RequireRole`、`@PreAuthorize` 之类的拦截）；
>  5）**缓存**（`@Cacheable` 等，方法调用前后查写缓存）；
>  6）**参数校验、异常统一处理、审计日志**；
>  7）稍微进阶一点的：**分布式锁、幂等控制、多租户数据隔离**。
>
> 本质就是：这些逻辑都属于“横切关注点”，如果写进每个业务方法会大量重复且难维护，用 AOP 把它们抽成切面，通过代理或字节码织入统一管理，让业务代码更干净、更聚焦。

如果你愿意，我可以帮你针对其中一个场景（比如“统一接口日志 + 耗时监控”）写一整套从注解、切面到实际调用链的完整示例，你可以直接拿到项目里改一改就用。