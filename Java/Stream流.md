# 定义

**Stream 是一种对数据源（如集合、数组、I/O 等）进行“声明式、流水线式”处理的抽象序列。**
它不是存储数据的容器，而是把一系列操作（过滤、映射、排序、聚合等）按管道串起来，最后产生结果。

# 特点

- **数据源**：Stream 本身不保存数据，数据来自 `Collection`、数组、生成器等。不存数据，它不是集合。
- **流水线操作**：`filter / map / sorted ...` 等中间操作可以链式组合。
- **惰性执行**：中间操作、流水线操作不会立刻执行，只有遇到终止操作（如 `collect / forEach / reduce / count`）才真正运行。
- **一次性**：一个 Stream 通常只能消费一次，用完就不能再用。
- **可并行**：可以用 `parallelStream()` 或 `stream().parallel()` 并行处理（适合可拆分、无副作用的场景）。但要谨慎。会发生什么？什么情况会并行？并行的可能危害后果是什么？





# stream骨架

### 2.1 生成流（Source）

常见来源：

```
list.stream()
Arrays.stream(arr)
Stream.of(1,2,3)
IntStream.range(0, n)
Files.lines(path) // IO 场景
```

### 2.2 中间操作（Intermediate）——惰性

返回一个新的 Stream（仍未执行）：

- `filter`  **按条件筛选，保留满足条件的元素，不改变元素类型**。输入是 `Stream<T>`，输出还是 `Stream<T>`。
- `map / flatMap` `map` 的作用是：**把每个元素转换成另一种形式（字段提取/对象转换/类型变化），元素数量通常不变，但类型可以变**。输入 `Stream<T>`，输出 `Stream<R>`。
- `distinct`
- `sorted`
- `limit / skip`
- `peek`（调试用，别拿它做业务副作用）

### 2.3 终止操作（Terminal）——真正执行

- `collect`
- `forEach`
- `count`
- `reduce`
- `min/max`
- `anyMatch/allMatch/noneMatch`
- `findFirst/findAny`



# filter和map所需要的参数

`filter` / `map` 里面**本来就不是“随便放东西”**，它们各自要求你传入一种“函数接口对象”。
Lambda 之所以能直接写进去，是因为 **Java 会把 lambda 自动当成这个函数接口的实现**（这叫 *函数式接口 + lambda 表达式*）。

------

## 1) `filter` 里面本来应该传什么参数？

看方法签名最清楚（你不用背，只要理解）：

```java
Stream<T> filter(Predicate<? super T> predicate)
```

也就是说，`filter` 需要一个 **Predicate**，它本质就是：

> 给你一个元素 T，你返回一个 boolean：保留还是丢掉？

`Predicate<T>` 的核心抽象方法是：

```java
boolean test(T t);
```

所以 filter 里你要传的东西，本质上必须能做到：`test(User u) -> boolean`。

------

## 2) 那为什么能放 lambda：`u -> u.getAge() >= 18`？

因为这个 lambda **刚好匹配** `Predicate<User>` 的抽象方法：

- 形参：`User u`
- 返回：`boolean`

等价于你手写一个匿名类（老写法）：

```java
users.stream().filter(new java.util.function.Predicate<User>() {
    @Override
    public boolean test(User u) {
        return u.getAge() >= 18;
    }
});
```

lambda 只是把这个匿名类写法压缩了。

> 关键：`Predicate` 是 **函数式接口**（只有一个抽象方法），所以可以用 lambda 来实现。

------

## 3) map 里面本来应该传什么参数？

`map` 的签名类似：

```java
<R> Stream<R> map(Function<? super T, ? extends R> mapper)
```

它需要一个 **Function<T, R>**，核心抽象方法是：

```java
R apply(T t);
```

意思是：

> 给你一个元素 T，你返回一个新元素 R（完成转换/映射）

所以你写：

```java
.map(u -> u.getName())
```

就是把 `User -> String` 的映射函数传进去了。

等价老写法：

```java
.map(new java.util.function.Function<User, String>() {
    @Override
    public String apply(User u) {
        return u.getName();
    }
});
```

------

## 4) 为什么 `filter/map` 能“直接接收 lambda”？

总结成一句：

**因为它们的参数类型是函数式接口（Predicate / Function），lambda 表达式可以被 Java 自动转换成这些接口的实例。**

这件事有个名字：**目标类型推断（target typing）**
编译器看到 `.filter( 这里需要 Predicate<User> )`，就知道你这个 lambda 是在实现 `test(User)`。

------

## 5) 面试官常追问的两句你也顺便记住

- `filter`：传 `Predicate<T>`，做筛选（T → boolean）
- `map`：传 `Function<T,R>`，做转换（T → R）



### 最开始要传的是实现了接口的实现类对象

对，你理解得非常准：**这确实就是 Java “把函数当参数传”能力逐步演进的结果**。不过有个小修正：并不是“后来实现了匿名内部类对象”——**匿名内部类从 Java 1.1 就有了**，只是以前没有更简洁的写法；**Java 8 引入了 lambda**，让“把行为当参数传”变得像写函数一样。

按时间线捋一遍就很清楚：

------

## 1) 最早：传“实现类对象”（显式类）

你要给 `filter` 传 `Predicate<User>`，只能写一个类实现它：

```java
class AdultPredicate implements Predicate<User> {
    @Override
    public boolean test(User u) {
        return u.getAge() >= 18;
    }
}

// 使用
users.stream().filter(new AdultPredicate()).toList();
```

**缺点**：为了一个小逻辑要新建一个类，很啰嗦。

------

## 2) 后来（其实很早就有）：匿名内部类

不用单独建文件/类了，直接“现场 new 一个实现”：

```java
users.stream().filter(new Predicate<User>() {
    @Override
    public boolean test(User u) {
        return u.getAge() >= 18;
    }
}).toList();
```

**优点**：不用单独建类
**缺点**：还是很长，而且读起来不够“像函数”。

------

## 3) Java 8：lambda 表达式（函数式接口语法糖）

因为 `Predicate` 是**函数式接口**（只有一个抽象方法），所以可以用 lambda 简写：

```java
users.stream()
     .filter(u -> u.getAge() >= 18)
     .toList();
```

这就是把“匿名内部类实现 `test`”压缩成一行的语法糖。

------

## 4) 更进一步：方法引用（lambda 的进一步简写）

当 lambda 只是“调用某个现成方法”时，可以用方法引用：

```java
users.stream()
     .filter(StreamPractice::isAdult) // isAdult(User u) -> boolean
     .toList();

static boolean isAdult(User u) {
    return u.getAge() >= 18;
}
```

或者更常见的：

```java
users.stream().map(User::getName).toList();
```

------

## 你这句话怎么在面试里讲最稳？

你可以这样说（很加分）：

> Java 一开始只能传对象，所以想传“行为”就得传实现类对象；
> 后来用匿名内部类能就地实现接口，但很冗长；
> Java 8 引入 lambda（基于函数式接口）和方法引用，把“行为”以更简洁的方式作为参数传入，从而形成 Stream 这种声明式流水线写法。

------

## 一个重要区别（面试官可能追问）

**lambda 和匿名内部类不是完全等价**，两点常考：

1. **this 指向不同**

- 匿名内部类里的 `this` 是匿名类对象
- lambda 里的 `this` 是外部类对象

1. **lambda 不是“创建一个新类”的语法糖（实现层面更轻）**
   实现细节不必展开太深，但可以点到：lambda 通常通过 `invokedynamic` 之类机制生成/调用，更偏“函数”而不是“新类型”。



# 常用操作

### 3.1 filter：过滤

```java
List<User> adults = users.stream()
    .filter(u -> u.getAge() >= 18)    filter里面本来应该传递什么参数？为什么可以放一个lambda表达式？因为filter和map里面需要的参数都是一个函数式接口对象，最开始是一个显示构建的实现类，把对象传进去，
    .toList(); // JDK16+；老版本用 collect(toList())
```

### 3.2 map：一对一转换（DO → DTO）

```java
List<UserDTO> dtos = users.stream()
    .map(u -> new UserDTO(u.getId(), u.getName()))
    .toList();
```

### 3.3 sorted：排序（写 Comparator 的能力很关键）

```java
List<User> sorted = users.stream()
    .sorted(Comparator.comparing(User::getAge).reversed()
        .thenComparing(User::getId))
    .toList();
```

### 3.4 distinct：去重（注意：依赖 equals/hashCode）

```java
List<String> uniq = list.stream().distinct().toList();
```

### 3.5 limit/skip：截取/跳过（常用于分页前处理）

```java
List<User> page = users.stream()
    .skip((long)(pageNo - 1) * pageSize)
    .limit(pageSize)
    .toList();
```

### 3.6 anyMatch/allMatch/noneMatch：条件判断（短路）

```java
boolean hasVip = users.stream().anyMatch(User::isVip);
```

### 3.7 findFirst/findAny：找一个

```java
Optional<User> first = users.stream().filter(...).findFirst();
```

### 3.8 collect：收集到 List/Set/Map（必考）

```java
Map<Long, User> byId = users.stream()
    .collect(Collectors.toMap(User::getId, u -> u));
```





# 进阶操作 高频

### 4.1 flatMap：一对多“拍平”

场景：一个用户多个标签，想拿到所有标签的列表

```java
List<String> tags = users.stream()
    .flatMap(u -> u.getTags().stream())
    .distinct()
    .toList();
```

### 4.2 groupingBy：分组统计（非常常用）

按部门分组：

```java
Map<String, List<User>> byDept = users.stream()
    .collect(Collectors.groupingBy(User::getDept)); 这里面用的是方法引用，也是lambda表达式的最简便形式，为什么放个lambda表达式就可以，这里需要的参数是什么？
```

按部门计数：

```java
Map<String, Long> cnt = users.stream()
    .collect(Collectors.groupingBy(User::getDept, Collectors.counting()));
```

按部门取最大年龄：

```java
Map<String, Optional<User>> oldest = users.stream()
    .collect(Collectors.groupingBy(
        User::getDept,
        Collectors.maxBy(Comparator.comparing(User::getAge))
    ));
```

### 4.3 toMap：key 冲突是大坑（高频）

如果 id 可能重复，你必须提供 merge 规则：

```java
Map<Long, User> byId = users.stream()
    .collect(Collectors.toMap(
        User::getId,
        u -> u,
        (a, b) -> a // 冲突时保留哪个？按业务定
    ));
```



# 常见应用场景

给出实际代码示例？

你在真实业务里会经常看到 Stream 出现在这些“胶水逻辑”里：

### A. DTO/VO 转换与字段映射（最常见）

把数据库查出来的 DO / PO 转成返回给前端的 DTO：

- `map()` 做对象转换
- `collect(toList())` 收集结果

### B. 列表过滤/去重/排序/分页前处理

- 规则过滤：`filter`
- 去重：`distinct` 或基于 key 去重（大厂更常见）
- 排序：`sorted(Comparator…)`

### C. 分组与统计报表（最能体现 Stream 的价值）

- 按部门分组、按状态分组：`groupingBy`
- 每组计数/求和/最大值：`counting/summingInt/maxBy`

### D. 构建 Map / 索引（很常见）

把 list 变成 `Map<id,对象>`，后面做 join 或查找更快：

- `collect(toMap(key, value, merge))`

### E. 轻量 join（应用层的“关联”）

比如订单列表要补用户信息：先把用户 list 转成 map，再 `map` 填充。

> Stream 在大厂里不是为了“炫技”，而是为了**表达清晰 + 少写临时变量 + 把处理步骤组合成可读的流水线**。
>  但在极致性能或复杂循环控制时，for 仍然常见。





# 辅助案例

下面给你一套 **30 道从易到难的 Java Stream 练习题（每题都有“你该手敲的目标 + 标准答案 + 常考点/坑”）**。
你照着顺序刷，刷完你会：

- 业务里：过滤/转换/分组/统计/toMap/去重/排序/Join 都能写
- 面试里：惰性执行、短路、flatMap、toMap 冲突、并行坑、Optional 坑都能讲清楚

我用一个统一的 **User 业务模型**（大厂最常见），并附一个最小数据集，你可以直接 copy 到 IDE 里手敲。

------

## 0) 统一模型与测试数据（先复制）

```java
import java.time.LocalDate;
import java.util.*;
import java.util.function.Function;
import java.util.stream.*;

class User {
    private final long id;
    private final String name;
    private final int age;
    private final String dept;
    private final int salary;
    private final boolean vip;
    private final LocalDate joinDate;
    private final List<String> tags;

    public User(long id, String name, int age, String dept, int salary, boolean vip, LocalDate joinDate, List<String> tags) {
        this.id = id;
        this.name = name;
        this.age = age;
        this.dept = dept;
        this.salary = salary;
        this.vip = vip;
        this.joinDate = joinDate;
        this.tags = tags;
    }

    public long getId() { return id; }
    public String getName() { return name; }
    public int getAge() { return age; }
    public String getDept() { return dept; }
    public int getSalary() { return salary; }
    public boolean isVip() { return vip; }
    public LocalDate getJoinDate() { return joinDate; }
    public List<String> getTags() { return tags; }

    @Override public String toString() {
        return "User{id=" + id + ", name='" + name + "', age=" + age + ", dept='" + dept +
                "', salary=" + salary + ", vip=" + vip + ", joinDate=" + joinDate + ", tags=" + tags + "}";
    }
}

public class StreamPractice {
    static List<User> users = List.of(
            new User(1, "Alice", 23, "RD",  25000, true,  LocalDate.of(2023, 3, 1), List.of("java", "backend")),
            new User(2, "Bob",   19, "QA",  12000, false, LocalDate.of(2024, 1, 15), List.of("test", "python")),
            new User(3, "Cindy", 31, "RD",  35000, true,  LocalDate.of(2020, 7, 20), List.of("java", "arch")),
            new User(4, "David", 27, "OPS", 20000, false, LocalDate.of(2022, 11, 2), List.of("linux", "devops")),
            new User(5, "Eva",   27, "RD",  28000, false, LocalDate.of(2021, 5, 10), List.of("go", "backend")),
            new User(6, "Frank", 40, "QA",  22000, true,  LocalDate.of(2019, 9, 1), List.of("test", "lead")),
            // 故意加一个同名、一个同dept、以及 id 冲突练 toMap merge
            new User(7, "Alice", 29, "PM",  30000, false, LocalDate.of(2022, 2, 8), List.of("product")),
            new User(3, "Cindy2", 26, "RD",  26000, false, LocalDate.of(2025, 6, 1), List.of("java"))
    );

    public static void main(String[] args) {
        // 你把每题写成一个方法，main 里调用打印验证
    }
}
```

> 注意：我故意放了 **id=3 重复** 来制造 `toMap` 冲突坑；也放了重复名字 Alice 来制造“按 name 去重”坑。

------

## Part A：基础热身（1～10）— 让你“手指有感觉”

### 1) 过滤出所有成年人（age >= 18），返回 List

**常考点**：filter + toList

```java
List<User> ans = users.stream()
        .filter(u -> u.getAge() >= 18)
        .toList();
```

### 2) 只拿出所有人的名字，返回 List

**常考点**：map

```java
List<String> ans = users.stream()
        .map(User::getName)
        .toList();
```

### 3) 找出所有 VIP 用户的 id，返回 Set

**坑**：收集到 Set 去重

```java
Set<Long> ans = users.stream()
        .filter(User::isVip)
        .map(User::getId)
        .collect(Collectors.toSet());
```

### 4) 是否存在 OPS 部门的人？

**常考点**：anyMatch（短路）

```java
boolean ans = users.stream().anyMatch(u -> "OPS".equals(u.getDept()));
```

### 5) 是否所有人 salary 都 >= 10000？

**常考点**：allMatch

```java
boolean ans = users.stream().allMatch(u -> u.getSalary() >= 10000);
```

### 6) 找出薪资最高的人（Optional）

**常考点**：max + Comparator

```java
Optional<User> ans = users.stream()
        .max(Comparator.comparingInt(User::getSalary));
```

### 7) 找出 RD 部门中薪资最高的人，返回 User（没有则返回 null）

**坑**：Optional 拆箱 orElse

```java
User ans = users.stream()
        .filter(u -> "RD".equals(u.getDept()))
        .max(Comparator.comparingInt(User::getSalary))
        .orElse(null);
```

### 8) 按 age 升序排序，若 age 相同按 salary 降序

**常考点**：Comparator 链式

```java
List<User> ans = users.stream()
        .sorted(Comparator.comparingInt(User::getAge)
                .thenComparing(Comparator.comparingInt(User::getSalary).reversed()))
        .toList();
```

### 9) 取入职最早的 3 个人

**常考点**：sorted + limit

```java
List<User> ans = users.stream()
        .sorted(Comparator.comparing(User::getJoinDate))
        .limit(3)
        .toList();
```

### 10) 跳过入职最早的 2 个，再取 3 个（模拟分页）

**常考点**：skip/limit

```java
List<User> ans = users.stream()
        .sorted(Comparator.comparing(User::getJoinDate))
        .skip(2)
        .limit(3)
        .toList();
```

------

## Part B：业务常用（11～20）— 大厂项目高频

### 11) 统计一共有多少个 RD

**常考点**：count

```java
long ans = users.stream().filter(u -> "RD".equals(u.getDept())).count();
```

### 12) 所有人的 salary 总和（用原始流）

**面试点**：避免装箱，mapToInt

```java
int ans = users.stream().mapToInt(User::getSalary).sum();
```

### 13) 计算 RD 部门平均薪资（OptionalDouble）

```java
OptionalDouble ans = users.stream()
        .filter(u -> "RD".equals(u.getDept()))
        .mapToInt(User::getSalary)
        .average();
```

### 14) 找出所有出现过的 tag（去重后 List）

**面试点**：flatMap

```java
List<String> ans = users.stream()
        .flatMap(u -> u.getTags().stream())
        .distinct()
        .toList();
```

### 15) 统计每个部门人数 Map<String, Long>

**常考点**：groupingBy + counting

```java
Map<String, Long> ans = users.stream()
        .collect(Collectors.groupingBy(User::getDept, Collectors.counting()));
```

### 16) 统计每个部门薪资总和 Map<String, Integer>

**常考点**：groupingBy + summingInt

```java
Map<String, Integer> ans = users.stream()
        .collect(Collectors.groupingBy(User::getDept, Collectors.summingInt(User::getSalary)));
```

### 17) 每个部门薪资最高的人 Map<String, User>

**坑**：maxBy 返回 Optional；要 collectingAndThen 转成 User

```java
Map<String, User> ans = users.stream()
        .collect(Collectors.groupingBy(
                User::getDept,
                Collectors.collectingAndThen(
                        Collectors.maxBy(Comparator.comparingInt(User::getSalary)),
                        opt -> opt.orElse(null)
                )
        ));
```

### 18) 按部门分组后，每组只要名字列表 Map<String, List>

```java
Map<String, List<String>> ans = users.stream()
        .collect(Collectors.groupingBy(
                User::getDept,
                Collectors.mapping(User::getName, Collectors.toList())
        ));
```

### 19) 把 users 转成 Map<id, User>（注意 id 重复！）

**面试大坑**：toMap 默认会抛异常，必须 merge

```java
Map<Long, User> ans = users.stream()
        .collect(Collectors.toMap(
                User::getId,
                Function.identity(),
                (oldV, newV) -> oldV // 业务决定保留哪个
        ));
```

### 20) 把 users 转成 Map<dept, List>（其实就是 groupingBy）

```java
Map<String, List<User>> ans = users.stream()
        .collect(Collectors.groupingBy(User::getDept));
```

------

## Part C：高频坑点与面试加分（21～30）

### 21) “按 name 去重”并保留 salary 更高的那一个

**常见业务需求**：同名只保留最强的
**坑**：distinct 不行（除非你重写 equals/hashCode）

```java
Collection<User> ans = users.stream()
        .collect(Collectors.toMap(
                User::getName,
                Function.identity(),
                (a, b) -> a.getSalary() >= b.getSalary() ? a : b
        ))
        .values();
```

> 面试加分：讲清楚 distinct 依赖 equals/hashCode；按 key 去重用 toMap 更常用。

### 22) 找出“每个部门薪资 Top2” Map<String, List>

**业务常见**：部门榜单

```java
Map<String, List<User>> ans = users.stream()
        .collect(Collectors.groupingBy(User::getDept))
        .entrySet().stream()
        .collect(Collectors.toMap(
                Map.Entry::getKey,
                e -> e.getValue().stream()
                        .sorted(Comparator.comparingInt(User::getSalary).reversed())
                        .limit(2)
                        .toList()
        ));
```

> 坑：分组后再对每组排序/limit，别指望一次 sorted 解决。

### 23) 统计所有 tag 出现次数 Map<String, Long>

**面试点**：flatMap + groupingBy counting

```java
Map<String, Long> ans = users.stream()
        .flatMap(u -> u.getTags().stream())
        .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));
```

### 24) 找到“tag 出现最多的那个 tag”（Optional<Map.Entry<String, Long>>）

```java
Optional<Map.Entry<String, Long>> ans = users.stream()
        .flatMap(u -> u.getTags().stream())
        .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()))
        .entrySet().stream()
        .max(Map.Entry.comparingByValue());
```

### 25) 找入职时间在 2022-01-01 之后且是 VIP 的人，按入职时间升序返回名字

```java
List<String> ans = users.stream()
        .filter(User::isVip)
        .filter(u -> u.getJoinDate().isAfter(LocalDate.of(2022, 1, 1)))
        .sorted(Comparator.comparing(User::getJoinDate))
        .map(User::getName)
        .toList();
```

### 26) 用 reduce 实现 salary 求和（练面试题）

**面试点**：reduce 三种形式；但业务更推荐 sum

```java
int ans = users.stream()
        .map(User::getSalary)
        .reduce(0, Integer::sum);
```

### 27) 找到第一个满足：RD 且 salary >= 30000 的人（Optional）

**短路**：findFirst

```java
Optional<User> ans = users.stream()
        .filter(u -> "RD".equals(u.getDept()))
        .filter(u -> u.getSalary() >= 30000)
        .findFirst();
```

### 28) 演示惰性：用 peek 打印链路（只用于调试）

**面试点**：中间操作不执行，终止才执行

```java
List<String> ans = users.stream()
        .filter(u -> u.getAge() >= 20)
        .peek(u -> System.out.println("after filter: " + u.getName()))
        .map(User::getName)
        .peek(n -> System.out.println("after map: " + n))
        .toList();
```

> 坑：peek 不要写业务副作用，尤其别在并行流里改共享对象。

### 29) 并行流坑：不要往同一个 ArrayList add（示例错误写法）

**面试点**：parallelStream + 共享可变状态 = 数据错/并发问题

```java
List<Long> bad = new ArrayList<>();
users.parallelStream().forEach(u -> bad.add(u.getId())); // ❌ 可能错
```

正确写法（用 collect）：

```java
List<Long> good = users.parallelStream()
        .map(User::getId)
        .toList(); // 或 collect(toList())
```

### 30) 业务 join 模拟：给订单列表补全用户名字（典型大厂场景）

先构建索引 Map，再 map 填充（**大厂非常常见**）

```java
class Order {
    long orderId;
    long userId;
    String userName; // 需要补
    Order(long orderId, long userId) { this.orderId = orderId; this.userId = userId; }
    public long getUserId() { return userId; }
    public void setUserName(String userName) { this.userName = userName; }
    @Override public String toString(){ return "Order{orderId=" + orderId + ", userId=" + userId + ", userName=" + userName + "}"; }
}

List<Order> orders = List.of(new Order(1001, 1), new Order(1002, 3), new Order(1003, 999));

Map<Long, String> idToName = users.stream()
        .collect(Collectors.toMap(
                User::getId,
                User::getName,
                (a, b) -> a // id 冲突处理
        ));

List<Order> filled = orders.stream()
        .peek(o -> o.setUserName(idToName.getOrDefault(o.getUserId(), "UNKNOWN")))
        .toList();
```

> 这题面试你可以讲：
>
> 1. 先把 users 建索引 map（O(n)）
> 2. 再遍历 orders 去补字段（O(m)）
> 3. 避免双重循环 O(n*m)

------

## 你必须吃透的“坑 & 常考点”总结（背这段就很稳）

1. **惰性执行**：中间操作不跑，终止操作才跑
2. **短路**：`limit/findFirst/anyMatch` 能提前结束
3. **distinct** 依赖 `equals/hashCode`，按 key 去重用 `toMap`
4. **toMap 冲突**：不写 merge 默认抛异常
5. **Optional**：`orElse/orElseGet/orElseThrow` 区别要懂（orElse 会提前计算）
6. **peek** 仅调试，不做业务副作用
7. **parallelStream**：不要共享可变状态；不一定更快
8. **性能**：能用 `mapToInt/sum` 就别装箱 `reduce`

------

## 给你一个“练习方法”（这样练最有效）

- 你先把每题的“题目”抄到方法里，把答案注释掉，自己手敲实现
- 每写完一题就 `System.out.println` 打印结果对照
- **第 19、21、29、30** 这四题是面试最喜欢挖坑的，一定多敲两遍

------

如果你愿意，我可以再追加一份“**面试官追问脚本**”：
比如你写了 `groupingBy + maxBy`，面试官会怎么追问你 Optional、并行、性能、Collectors 内部原理，你应该怎么答（按一句一句可背的方式）。你想先要这份“追问脚本”，还是先要“每题的预期输出/验证结果”？