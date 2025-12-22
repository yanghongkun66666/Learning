

ArrayList 底层是 Object[] 存引用，允许 null 就是允许某个槽位存空引用（0 指针），不分配对象；可以存多个 null。对 Integer 这种包装类型，数组里存的是引用（4/8 字节），对象本体在堆上（通常 16 字节左右），null 的主要风险是自动拆箱导致 NPE。

# ArrayList 是什么 定义

一句话：**基于动态数组（Object[]）实现的 List**。

它的核心特性：

- **随机访问快**：`get(i)` / `set(i)` 是 O(1)
- **尾部追加快**：`add(e)` 平均 O(1)，扩容时 O(n)
- **中间插入/删除慢**：需要搬移元素，O(n)
- **允许 null**   ArrayList 里某个位置的元素可以是 `null`  可以add多个null
- **非线程安全**
- **迭代器 fail-fast**（快速失败）

------



# 继承链

## 1) 继承/实现层级（接口链）

- `Collection` 是最顶层的集合接口之一
- `List` **extends** `Collection`（List 是 Collection 的子接口，继承了集合接口） 
- `ArrayList` **implements** `List`   实现了列表接口

所以这条链是：
**ArrayList → List → Collection**

------

## 2) 但别漏了“类继承链”（源码里更关键）

`ArrayList` 不只是实现接口，它还继承了抽象类：

在 JDK 17 中大致是：

- `ArrayList` **extends** `AbstractList<E>` 抽象列表
- `AbstractList<E>` **extends** `AbstractCollection<E>` 抽象集合
- 然后 `AbstractCollection<E>` 实现了 `Collection<E>`

所以更完整的层级（类 + 接口）是：

- **接口：** `ArrayList implements List`，而 `List extends Collection`
- **类：** `ArrayList extends AbstractList extends AbstractCollection`

> ArrayList 是 List 的典型实现，List 是 Collection 的子接口；同时 ArrayList 继承自 AbstractList，通过 AbstractCollection 间接实现 Collection。



# 2. 业务开发里 ArrayList 用在什么地方

你在大厂项目里最常见的使用场景：

- **数据库/接口返回的结果集容器**（DTO 列表）
- **组装批量入参**（比如批量更新、批量调用远程服务）
- **中间结果缓存**（先 collect 到 list，再做分组/分页/排序）
- **接口返回**（JSON 序列化 List）

同时它也是最容易被“性能/并发/扩容”坑到的容器。

------

# 3. ArrayList 的底层字段（JDK 17 源码级）

ArrayList 最关键的 3 个成员：

```java
transient Object[] elementData; // 真正存数据的数组（transient：自定义序列化）
private int size;               // 当前元素个数（不是容量）
protected transient int modCount; // 来自 AbstractList，用于 fail-fast
```

## size vs capacity

- **size**：你放了多少个元素
- **capacity**：底层数组 `elementData.length` 的长度（能放多少个，不扩容的情况下）

面试必问：**ArrayList 查询 size 是 O(1)**，因为有字段直接存着。

List<Integer> list = new ArrayList<>();

这个数组的每个格子里存的不是 `Integer` 本体

存的是 **指向 Integer 对象的“引用（reference）”**

```` scss
elementData[0] -> (指向某个 Integer 对象)
elementData[1] -> null（空引用）
elementData[2] -> (指向另一个 Integer 对象)
...
````

Object[] 每个格子占多少字节？

槽位里放的是 **引用**。引用大小取决于 JVM：

- **64 位 JVM + 开启压缩指针（Compressed Oops，最常见）**：引用 **4 字节**
- **64 位 JVM + 未开启压缩指针**：引用 **8 字节**

> 所以你说“一个元素占 4 字节”只在“开启压缩指针”且说的是“数组槽位引用”时大概率成立，但不能绝对。

------

#### B) Integer 对象本身占多少字节？

`Integer` 是对象，存在堆上，通常包含：

- 对象头（mark word + klass 指针）
- 一个 `int value` 字段（4 字节）
- 对齐填充（按 8 或 16 字节对齐）

在常见的 HotSpot 64 位 + 压缩指针环境下，`Integer` 往往是 **16 字节左右**（可能随 JVM 参数变化）。

所以：`ArrayList<Integer>` 的空间大头通常是 **Integer 对象本身**，不是那 4/8 字节的引用。





##  null 在内存里存的是什么？

**null 存的是“空引用”**。

在 `Object[]` 里某个槽位为 `null`，就是这个引用值为 **0（全 0 比特）**，表示“不指向任何对象”。

- **不会分配任何对象**
- 只是那个槽位里放了 “0” 这个引用值





## 一个你会在业务里踩的坑：自动拆箱 NPE  存null可能带来的坑

`Integer` 可以是 null，但你如果把它当 `int` 用，会触发拆箱：

```
Integer x = list.get(0); // x 可能是 null
int y = x;               // ❌ 如果 x==null，这里直接 NullPointerException
```





## 还有一个容易混淆的点：ArrayList 里“没用到的位置”本来就是 null

ArrayList 的底层数组容量可能比 size 大：

- `size` 之前：有效元素（可能为 null，也可能不为 null）
- `size` 之后：未使用槽位，通常就是 null（这是为了 GC 友好）

所以：

- `null` 既可能代表“这个位置存的元素就是 null”
- 也可能只是“这个槽位还没用到”（但后者不算 list 的元素，因为 `size` 不包含它）





# 4. 构造器的“隐藏行为”

### 4.1 `new ArrayList<>()`

不会立刻分配 10 个空间，而是：

- `elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA`（一个空数组标记）
- **第一次 add 时才扩容到 10**

### 4.2 `new ArrayList<>(0)`

- `elementData = EMPTY_ELEMENTDATA`（另一个空数组标记）
- 第一次 add 时扩到 **1**（再按增长策略涨）

### 4.3 `new ArrayList<>(n)`

直接分配长度为 n 的数组

> 默认构造和 initialCapacity=0 的行为不同，这是为了兼顾“默认常用容量 10”与“显式要求 0 容量”的语义。

------

# 5. 扩容机制（最核心、最常问）

当你 `add(e)` 时，如果放不下，会扩容。

## 5.1 增长策略：1.5 倍

JDK 17 的核心逻辑（简化版）：

```java
newCapacity = oldCapacity + (oldCapacity >> 1); // old * 1.5
if (newCapacity < minCapacity) newCapacity = minCapacity;
elementData = Arrays.copyOf(elementData, newCapacity);
```

- `old >> 1` 等于 `old / 2`
- 所以整体是 `old + old/2` → **1.5 倍**

## 5.2 为什么是 1.5 倍？

折中：

- 倍数太小：扩容频繁，拷贝成本高
- 倍数太大：浪费内存，GC 压力大

## 5.3 扩容成本是什么？

扩容本质是：

- 分配新数组
- `System.arraycopy` 把旧数组复制过去
  所以是 **O(n)**。

业务建议：

- 你大概知道最终数量时，**一定用 new ArrayList<>(预计大小) 或 ensureCapacity**，这是很实在的性能优化点。

------

# 6. add / get / remove 的时间复杂度（源码级理解）

## 6.1 `get(index)`：O(1)

就是数组下标访问：

```java
return (E) elementData[index];
```

## 6.2 `add(e)` 尾插：均摊 O(1)

- 不扩容：直接 `elementData[size++] = e`
- 扩容：O(n) 拷贝

均摊 O(1) 的直觉：扩容不是每次发生，发生时一次性“买更多空间”。

## 6.3 `add(index, e)`：O(n)

要把 index 右侧整体右移：

```java
System.arraycopy(elementData, index, elementData, index+1, size-index);
elementData[index] = e;
size++;
```

## 6.4 `remove(index)`：O(n)

要把 index 右侧整体左移：

```java
int numMoved = size - index - 1;
if (numMoved > 0)
    System.arraycopy(elementData, index+1, elementData, index, numMoved);
elementData[--size] = null; // 重要：帮助 GC
```

面试加分点：**最后置 null 是为了让被删除对象可被 GC 回收（避免内存泄漏式的“对象游离引用”）**。

------

# 7. fail-fast（并发修改检测）是怎么做到的？

你经常听到：“ArrayList 迭代过程中修改会抛 ConcurrentModificationException”，这就是 fail-fast。

关键是 `modCount`：

- 结构性修改（add/remove/clear 等）会 `modCount++`
- 迭代器创建时会保存一个 `expectedModCount = modCount`
- 迭代过程中每次 `next()` / `remove()` 会检查两者是否相等

伪码：

```java
if (modCount != expectedModCount) throw new ConcurrentModificationException();
```

重要理解：

- 这不是线程安全保证，只是“**尽早发现错误用法**”
- 多线程下它不是 100% 必现（只是 best-effort）

------

# 8. Iterator.remove() 为什么是“合法姿势”？

很多人踩坑：

```java
for (User u : list) {
  if (...) list.remove(u); // 常见炸：ConcurrentModificationException
}
```

正确姿势：

```java
Iterator<User> it = list.iterator();
while (it.hasNext()) {
  User u = it.next();
  if (...) it.remove(); // ✅
}
```

原因：`it.remove()` 会同步维护迭代器的 `expectedModCount`，不会触发 fail-fast。

------

# 9. removeIf 为什么比你手写循环更好（JDK 17）

ArrayList 在 JDK 17 **重写了 removeIf**，实现是“标记 + 压缩”的批量删除思路（常见是 BitSet/标记数组）：

- 先扫描一遍标记要删除的元素
- 再一次性把保留的元素拷贝/覆盖到前面
- 最后把尾部置 null

优点：

- 少调用 `remove(index)`（否则每删一个都搬移，最坏 O(n^2)）
- 批量删除接近 O(n)

业务建议：大量删除用 `removeIf` 很香。

------

# 10. subList 是“视图”还是“拷贝”？（面试超级高频）

`subList(from, to)` 返回的是 **视图（view）**，不是新列表拷贝。

特点：

- 子列表和原列表共享底层数组区域（逻辑上共享）
- 修改子列表会影响原列表，反之亦然
- 原列表结构性修改（非通过 subList）会导致 subList 后续操作可能抛 `ConcurrentModificationException`

面试回答模板：

> subList 是视图，持有父列表引用与偏移量，结构变更会触发 fail-fast。

如果你真要独立拷贝：

```java
List<E> copy = new ArrayList<>(list.subList(a, b));
```

------

# 11. ArrayList 的序列化为什么 elementData 是 transient？

因为 ArrayList 自己实现了 `writeObject/readObject`：

- 只序列化 `size` 个元素，而不是整个 capacity（否则会把一堆 null 也写进去，大于size的那些未使用元素）
- `elementData` 标记 transient，避免默认序列化浪费空间

这属于“源码加分点”。

------

# 12. 和 LinkedList / Vector / CopyOnWriteArrayList 的对比（面试必会）

## ArrayList vs LinkedList

- ArrayList：随机访问快；中间插入删除慢（搬移）
- LinkedList：随机访问慢（O(n)）；插入删除如果你已经定位到节点可 O(1)，但“定位”通常就 O(n)
- **现实中：99% 场景 ArrayList 更快**（因为 CPU 缓存友好、内存连续）

## ArrayList vs Vector

- Vector 方法大多带 synchronized，线程安全但开销大
- 现在一般不用 Vector

## ArrayList vs CopyOnWriteArrayList

- 写时复制（每次写都复制新数组），写很贵
- 读非常快且线程安全
- 适合：**读多写少**（比如配置监听器列表）

------

# 13. 业务最佳实践（你写项目时照着做）

1. **预估大小就预分配**

```java
List<User> list = new ArrayList<>(expectedSize);
```

1. 批量 add 用 `addAll`（减少多次扩容）
2. 大量删除用 `removeIf` 或者“新建列表收集保留项”（避免 O(n^2)）标记压缩的方式，提高性能
3. 迭代删除用 `Iterator.remove()`，别直接 `list.remove` 避免并发修改异常
4. 多线程不要裸用 ArrayList  并非线程安全的容器
   - 读写都有：`Collections.synchronizedList(new ArrayList<>())` 或 `CopyOnWriteArrayList` 或更合适的并发结构

------

# 14. 面试官连环追问你怎么答（给你现成话术）

**Q：ArrayList 为什么快？**

> 底层是连续数组，随机访问 O(1)，而且内存连续对 CPU cache 友好。

**Q：扩容怎么做？**

> 不够放时按 1.5 倍增长，用 Arrays.copyOf 拷贝旧数组到新数组，扩容是 O(n)。

**Q：为什么 add 是均摊 O(1)？**

> 大多数 add 只是赋值，只有少量触发扩容 O(n)，摊到每次就是均摊常数。

**Q：为什么 for-each remove 会 CME？**

> for-each 走迭代器，迭代器记录 expectedModCount，list.remove 修改 modCount，检测到不一致抛异常；正确做法是 iterator.remove。

**Q：subList 是拷贝吗？**

> 不是，是视图，共享父列表区间；结构性修改可能导致 fail-fast。

