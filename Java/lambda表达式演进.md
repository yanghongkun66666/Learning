# 定义

**本质价值是：把“行为/函数”当成数据传来传去**，从而让很多 API（尤其 Stream、并发、回调）变得非常自然。





## 0）先明确：sort( ) 里放的到底是什么？

`List.sort` 需要一个 **Comparator**（函数式接口）：

```java
@FunctionalInterface
public interface Comparator<T> {
    int compare(T a, T b);
}
```

lambda 只是把 `compare(a, b)` 这个方法“用更短的方式写出来”。

------

## 1）演进过程：从最原始到最简洁

### A. 单独写一个比较器类（最原始、最啰嗦）

```java
class AscComparator implements Comparator<Integer> {
    @Override
    public int compare(Integer a, Integer b) {
        return Integer.compare(a, b);
    }
}

nums.sort(new AscComparator());
```

### B. 匿名内部类（Java 7 常见写法）

```java
nums.sort(new Comparator<Integer>() {
    @Override
    public int compare(Integer a, Integer b) {
        return Integer.compare(a, b);
    }
});
```

### C. lambda：带完整类型 + 代码块

```java
nums.sort((Integer a, Integer b) -> {
    return Integer.compare(a, b);
});
```

### D. lambda：省略类型（编译器可推断）+ 代码块

```java
nums.sort((a, b) -> {
    return Integer.compare(a, b);
});
```

### E. lambda：只有一行表达式（最常见），可省略 `{}` 和 `return`

```java
nums.sort((a, b) -> Integer.compare(a, b));
```

### F. 方法引用（lambda 的再简化）

```java
nums.sort(Integer::compare);
```

> 这一步等价于：`(a, b) -> Integer.compare(a, b)`

------

## 2）“全部可能情况”：lambda 语法形式你可以怎么写

lambda 的核心语法是：

```
(参数列表) -> 表达式  只有一行表达式可以将大括号去掉
(参数列表) -> { 语句块 }
```

### 2.1 参数列表的各种写法（以 Comparator 的两个参数为例）

**① 带类型**

```java
nums.sort((Integer a, Integer b) -> Integer.compare(a, b));
```

**② 不带类型（最常用，靠推断）**

```java
nums.sort((a, b) -> Integer.compare(a, b));
```

**③ 用 var（Java 10+），想写类型但又懒**

```java
nums.sort((var a, var b) -> Integer.compare(a, b));
```

> 注意：**要么全写类型/var，要么全不写**，不能混着来：
> ❌ `(var a, b) -> ...` 不允许

------

### 2.2 函数体的各种写法

**① 单表达式（省略 return 和大括号）**

```java
nums.sort((a, b) -> Integer.compare(a, b));
```

**② 多语句（必须加 {}，return 也要写）**

```java
nums.sort((a, b) -> {
    // 你想写日志、做中间变量都行
    int r = Integer.compare(a, b);
    return r;
});
```

**③ 用三目运算写规则（也是单表达式）**

```java
nums.sort((a, b) -> a.equals(b) ? 0 : (a < b ? -1 : 1));
```

------

## 3）排序里你会遇到的“常见 lambda 版本全集”

### 3.1 升序（正向）

推荐：

```java
nums.sort((a, b) -> Integer.compare(a, b));
```

不推荐但你会看到：

```java
nums.sort((a, b) -> a - b); // 有溢出风险
```

### 3.2 降序（反向）

推荐：

```java
nums.sort((a, b) -> Integer.compare(b, a));
```

### 3.3 直接用 Comparator 的“现成工具”（也是 Comparator，只是不是 lambda 写出来）

```java
nums.sort(Comparator.naturalOrder());     // 升序
nums.sort(Comparator.reverseOrder());     // 降序
```

### 3.4 按“某个规则”排序（更能体现 lambda 的意义）

比如：按绝对值从小到大：

```java
nums.sort((a, b) -> Integer.compare(Math.abs(a), Math.abs(b)));
```

比如：按“离 target 的距离”从近到远：

```java
int t = target; // 需要是 effectively final
nums.sort((a, b) -> Integer.compare(Math.abs(a - t), Math.abs(b - t)));
```

### 3.5 组合规则：先按 A 排，再按 B 排（链式）

假设你有对象 `Person(age, name)`，更典型：

```java
people.sort(Comparator.comparingInt(Person::getAge)
                      .thenComparing(Person::getName));
```

------

## 4）你真正该记住的一句话

`nums.sort((a, b) -> ...)` 里这段 lambda **就是“compare(a,b) 的实现”**：

- 返回负数：a 在前
- 返回 0：相等
- 返回正数：a 在后

------





# lambda表达式常见应用场景

## 1) 集合处理：排序、过滤、映射、分组（配合 Stream）

- 排序：

```java
list.sort((a, b) -> a.getAge() - b.getAge());
// 或
list.sort(Comparator.comparing(User::getAge));
```

- 过滤 + 映射：

```java
List<String> names = users.stream()
    .filter(u -> u.isActive())
    .map(User::getName)
    .toList(); // 16+ 可用，8~15 用 collect(toList())
```

- 分组：

```java
Map<String, List<User>> byDept = users.stream()
    .collect(Collectors.groupingBy(User::getDept));
```

## 2) 回调/策略模式：把“行为”当参数传（替代匿名内部类）

典型：根据条件选择不同策略

只要一个方法参数类型是**函数式接口**，你就能用 lambda。

常见四大函数式接口（面试必会）：

- `Predicate<T>`：`T -> boolean`（筛选）
- `Function<T,R>`：`T -> R`（转换）
- `Consumer<T>`：`T -> void`（消费/执行）
- `Supplier<T>`：`() -> T`（提供/延迟创建）

```java
`Predicate<T>`：`T -> boolean`（筛选） 就是stream里面的filter需要传递的参数，需要时间的函数式接口

Function<Order, BigDecimal> pricing =
    vip ? this::vipPrice : this::normalPrice;

BigDecimal p = pricing.apply(order);

// Consumer：forEach 里传一个“怎么处理每个元素”
users.forEach(u -> System.out.println(u.getName()));

// Supplier：延迟提供默认值
String v = map.getOrDefault(key, defaultSupplier.get());

```

## 3) 事件监听 / 观察者：GUI、消息回调、Hook

```java
button.addActionListener(e -> System.out.println("clicked"));
```

在业务里对应：消息消费、状态变化通知、异步完成回调。



很多框架/工具方法会让你传一个 callback。

比如：

- 自定义重试 `retry(() -> callRemote())`
- 统一异常处理 `doWithLock(() -> ...)`
- 通用模板 `executeInTx(() -> ...)`

你会看到大量这种写法：

```
runWithLog("createUser", () -> userService.create(dto));
```

这就是把“要执行的动作”用 lambda 传进去。

## 4) 并发与异步：Runnable/Callable、CompletableFuture 用在并发与异步 API（线程/任务）

例如 `Runnable / Callable`：

```java
new Thread(() -> doWork()).start();

ExecutorService pool = Executors.newFixedThreadPool(4);
Future<Integer> f = pool.submit(() -> 1 + 2);
```

这是 lambda 最常见的“非 Stream”使用场景之一。

- 简化线程任务：

```java
executor.submit(() -> doWork());
```

- 异步流水线：

```java
CompletableFuture.supplyAsync(this::query)
    .thenApply(this::convert)
    .thenAccept(this::persist);
```

## 5) 资源包裹/模板方法：用 Lambda 把“核心逻辑”塞进模板里

例如统一做：日志、耗时、异常处理、事务边界

```java
<T> T withLog(String name, Supplier<T> supplier) {
    long t = System.currentTimeMillis();
    try { return supplier.get(); }
    finally { System.out.println(name + " cost=" + (System.currentTimeMillis()-t)); }
}

var r = withLog("loadUser", () -> loadUser(id));
```

## 6) Optional：避免显式 null 判断（但别滥用）

```java
String city = Optional.ofNullable(user)
    .map(User::getAddress)
    .map(Address::getCity)
    .orElse("unknown");
```

## 7) Map/Cache 的惰性计算：computeIfAbsent

```java
User u = cache.computeIfAbsent(id, this::loadUser);
```

常用于缓存、对象池、分组聚合。

## 8) 校验与断言：Predicate 组合

lambda 配合 `and / or / negate`（Predicate）和 `andThen / compose`（Function）可以把规则像积木一样拼起来。

```java
Predicate<User> ok = u -> u.isActive();
Predicate<User> adult = u -> u.getAge() >= 18;

List<User> res = users.stream().filter(ok.and(adult)).toList();

Predicate<User> isAdult = u -> u.getAge() >= 18;
Predicate<User> isRD = u -> "RD".equals(u.getDept());

List<User> ans = users.stream()
    .filter(isAdult.and(isRD))
    .toList();





Function<User, String> name = User::getName;
Function<String, Integer> len = String::length;

Function<User, Integer> nameLen = name.andThen(len);
int l = nameLen.apply(users.get(0));

```



## 9) 延迟执行与性能优化（惰性/避免无用计算）

一个经典面试点：**orElse vs orElseGet**

- `orElse(x)`：不管是否需要，`x` 先算出来（可能浪费）
- `orElseGet(() -> x)`：真的为空时才执行 lambda（惰性）

```
String v = opt.orElseGet(() -> expensiveCompute());
```

这在业务里经常能省掉不必要的 DB/远程调用/大对象构造。





## 10) “自定义高阶函数”：你自己写工具方法接 lambda（大厂常见封装）

你可以写一个通用的模板方法，把变化的部分交给 lambda：

```
static <T> T withTimer(String name, Supplier<T> action) {
    long s = System.currentTimeMillis();
    try { return action.get(); }
    finally { System.out.println(name + " cost " + (System.currentTimeMillis()-s)); }
}
```

调用：

```
String res = withTimer("query", () -> queryFromDB());
```

这就是“把行为当参数”，非常大厂。



------

### 面试常用一句话总结

Lambda 最常用在：**集合流水线（Stream）**、**回调/策略**、**异步编排（CompletableFuture）**、**模板封装（统一日志/事务/异常）** 这四类场景。

