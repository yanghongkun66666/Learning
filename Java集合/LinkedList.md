下面带你把 **LinkedList（JDK 17）从 0 到源码级精通**。我会按：**是什么 → 怎么用 → 业务场景 → 源码结构 → 复杂度 → 坑点 → 面试话术** 来讲，尽量“深入但不乱”。

------

# 1) LinkedList 是什么

一句话：**双向链表实现的 List + Deque**。

它在 JDK 17 里的定位非常明确：

- 实现 `List<E>`：可按下标访问（但慢）
- 实现 `Deque<E>`：可当**双端队列**用（这是它最适合的用法）

**你在业务里应该把 LinkedList 当成“队列/双端队列”来用，而不是当成 ArrayList 的替代品。**

------

# 2) 业务开发里 LinkedList 常用在哪？

大厂项目里 LinkedList 常见位置很集中：

## A. 队列/双端队列（最常见、最推荐）

- 任务队列（单线程/局部队列）
- BFS、拓扑排序等算法（面试/工具逻辑）
- 需要频繁 `addFirst/removeFirst/addLast/removeLast` 才考虑使用LinkedList，它是线程不安全的，不过实际上也是`ArrayDeque`，官方推荐的。

> 但注意：多线程队列一般用 `ConcurrentLinkedQueue` （无锁/弱一致，吞吐好）、`LinkedBlockingQueue` （需要阻塞/限流时），或者说：**简单包一层锁**：`Collections.synchronizedDeque(new ArrayDeque<>())`、而不是使用LinkedList。
>
> 线程安全可以用Collections.synchronizedList 或者 CopyOnWriteArrayList

## B. 很少用于“随机访问 list”

因为 `get(i)` 是 O(n)，真实业务中几乎不如 ArrayList。

------

# 3) 继承/实现层级（面试必会）

- `LinkedList<E>` **extends** `AbstractSequentialList<E>`
- `implements`：`List<E>, Deque<E>, Cloneable, Serializable`
- 它也实现了List接口

关键点：

- **AbstractSequentialList**：就是为“顺序访问结构”（链表）准备的抽象类，暗示它**不适合随机访问**。

------

# 4) LinkedList 的底层数据结构（JDK 17 源码核心）

LinkedList 内部是双向链表节点 `Node<E>`： 

```java
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;
    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

LinkedList 维护两个指针：

```java
transient int size = 0;
transient Node<E> first;
transient Node<E> last;
```

解释：

- `first` 指向头结点
- `last` 指向尾结点
- `size` 是元素个数（O(1)）
- `transient`：序列化时走自定义逻辑（类似 ArrayList）

------

# 5) 最核心方法：add / remove / get 的源码级逻辑

插入和删除节点效率较高（但也得先找到对应节点才行，好处在于不用移动元素）

随机访问需要遍历链表，效率较低

额外占用指针空间

## 5.1 add(e)（尾插）

在 LinkedList 里，`add(e)` 等价于 `linkLast(e)`：

- new 一个节点
- 改 last 指针
- size++、modCount++

伪码（接近源码）：

```java
void linkLast(E e) {
    Node<E> l = last;
    Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null) first = newNode;
    else l.next = newNode;
    size++;
    modCount++;
}
```

**复杂度：O(1)**（这是它的优势之一）

## 5.2 addFirst(e) / addLast(e)

同理，都是 O(1)，只改头尾指针。

## 5.3 removeFirst() / removeLast()

删除头/尾也都是 O(1)，并且会把删除节点的 item/next/prev 断开帮助 GC：

伪码：

```java
E unlinkFirst(Node<E> f) {
    E element = f.item;
    Node<E> next = f.next;
    f.item = null;
    f.next = null; // help GC
    first = next;
    if (next == null) last = null;
    else next.prev = null;
    size--;
    modCount++;
    return element;
}
```

## 5.4 get(index)（关键：慢！）

LinkedList 的 `get(i)` 会先判断 index 靠近头还是靠近尾：

- `index < size/2`：从 `first` 往后走
- 否则从 `last` 往前走

伪码：

```java
Node<E> node(int index) {
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++) x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--) x = x.prev;
        return x;
    }
}
```

**复杂度：O(n)**

> 面试加分：它做了“从近的一端开始走”的优化，但本质仍是线性。

## 5.5 add(index, e) / remove(index)

- 先 `node(index)` 找到位置（O(n)）
- 再 link/unlink（O(1)）
  所以整体还是 **O(n)**

------

# 6) 空间与性能：LinkedList 的“真实代价”（面试很爱问）

LinkedList 的每个元素不是只存一个值，还要存 **两个引用（prev/next）**，再加对象头。

粗略理解（常见 HotSpot 64 位 + 压缩指针）：

- 一个 Node 对象：对象头 + 3 个引用字段（item/prev/next）+ 对齐
- 通常会是 **24~32 字节级别**（随 JVM 变化）

相比之下 ArrayList：

- 每个元素槽位只是 4/8 字节引用（对象本体另算）

所以 LinkedList 的缺点：

- **内存占用大**
- **节点分散在堆上**，CPU cache 命中率差，很多场景即使复杂度看起来一样也更慢

> 这就是为什么现实里很多队列场景也更推荐 `ArrayDeque`。

------

# 7) fail-fast（ConcurrentModificationException）同样存在

LinkedList 也继承 `modCount`（来自 AbstractList），迭代器同样是：

- `expectedModCount`
- 每次 next/remove 检查

所以：

- for-each 遍历时直接 `list.remove(x)` 也会 CME
- 正确是 `Iterator.remove()`

------

# 8) LinkedList 允许 null 吗？允许多个吗？

允许。因为它存的是引用，节点的 `item` 字段可以为 null。

- `add(null)` ✅
- 多个 null ✅

------

# 9) LinkedList vs ArrayList：你怎么选（业务标准答案）

### 用 LinkedList 的场景（少）

- 你真的要当 **Deque** 用：`addFirst/removeFirst/addLast/removeLast`
- 你要频繁在头尾操作，并且数据量不大/性能要求不苛刻

### 不要用 LinkedList 的场景（多）

- 你需要大量 `get(i)`、`for(i...)` 访问 → **ArrayList**
- 你做队列但追求性能 → **ArrayDeque** 更好（内存连续、cache 友好）

------

# 10) 面试高频问答（给你可背话术）

**Q1：LinkedList 底层是什么？**

> 双向链表，节点 Node 有 item/prev/next，维护 first/last 指针与 size。

**Q2：为什么 get 慢？复杂度？**

> get(index) 需要从头或尾走到该节点，O(n)，只是做了从近端遍历的优化。

**Q3：哪些操作是 O(1)？**

> 头尾插入删除（addFirst/removeFirst/addLast/removeLast）是 O(1)。

**Q4：为什么很多时候 LinkedList 反而比 ArrayList 慢？**

> 节点分散、内存占用大、cache 不友好；ArrayList 连续内存随机访问快。

**Q5：队列用 LinkedList 还是 ArrayDeque？**

> 单线程高性能一般用 ArrayDeque；并发用并发队列；LinkedList 主要是功能上实现 Deque 但性能不占优。

**Q6：subList/迭代删除怎么处理？**

> 迭代过程中结构性修改会 fail-fast，删除要用 Iterator.remove。

------

