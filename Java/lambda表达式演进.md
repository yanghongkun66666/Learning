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

