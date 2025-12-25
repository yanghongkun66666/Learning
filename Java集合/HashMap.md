## 1. HashMap 是什么

**HashMap 是基于哈希表的 Key-Value 容器，平均 O(1) 时间完成 put/get/remove**（在 hash 均匀、负载适中时）。
JDK8+（JDK17 同样）采用：**数组 + 链表/红黑树** 解决冲突。

------

## 2. 最核心的结构

HashMap 底层主要是一个数组：

```java
Node<K,V>[] table;
```

每个数组槽（bucket）里存一条链：

- 冲突少：链表（Node next）
- 冲突多：树化成红黑树（TreeNode）

### 关键字段（面试常问）

- `size`：当前键值对数量
- `threshold`：触发扩容的阈值 `capacity * loadFactor`
- `loadFactor`：负载因子，默认 `0.75`
- `modCount`：结构性修改计数（fail-fast 用）
- `table.length`：容量，始终是 2 的幂

------

## 3. 为什么容量必须是 2 的幂？

因为索引计算是：

```java
index = (n - 1) & hash
```

当 `n` 是 2 的幂时，`(n-1)` 二进制全是 1，能把 hash 的低位均匀映射到 `[0, n-1]`，比 `% n` 更快（位运算）且分布更可控。

------

## 4. hash 过程：为什么要“扰动函数”

JDK17 里核心仍是：

- 先取 `key.hashCode()`
- 再做一次混合，让高位参与低位分布（因为我们用低位取桶）

典型逻辑（等价表达）：

```java
h = key.hashCode();
hash = h ^ (h >>> 16);
```

意义：降低一些糟糕 hashCode 导致的碰撞。

------

## 5. put/get 逻辑

### put 的流程（高频面试）

1. 如果 table 还没初始化 → `resize()` 初始化
2. 计算 hash 和桶下标 `i`
3. table[i] 为空 → 直接放 Node
4. table[i] 非空：
   - 先看首节点 key 是否相等，是就覆盖 value
   - 否则沿着链表找：
     - 找到相同 key → 覆盖
     - 走到尾部 → 追加新 Node
   - 如果链表太长 → 可能树化（见树化条件）
5. `size++`，如果 `size > threshold` → `resize()` 扩容

### get 的流程

1. 计算 hash 和桶下标
2. 先比首节点（最快）
3. 再遍历链表/红黑树查找 key

------

## 6. 扩容（resize）为什么是性能关键

扩容通常是 **2 倍**，并重分布元素。JDK8+ 的一个重要优化点：

### 旧版（JDK7）问题

重哈希时会重新计算 index，且链表反转导致并发下死循环（历史坑）。

### JDK8+（JDK17）优化：分裂成两个链

因为容量翻倍，从 `n` 变 `2n`，新的 index 只可能是：

- 仍在原位置 `i`
- 或者变成 `i + n`

判断只看一位：

```java
(hash & oldCap) == 0 ? stay : move
```

这样迁移是线性分裂，不需要重新 hash，性能更好。

------

## 7. 链表什么时候树化（JDK17）

两个阈值你要记住（面试常问）：

- `TREEIFY_THRESHOLD = 8`：链表长度达到 8，**可能**树化
- `UNTREEIFY_THRESHOLD = 6`：树节点变少可能退化回链表
- 还有一个很重要的：
  - **table 容量必须 ≥ 64（MIN_TREEIFY_CAPACITY）** 才会树化
  - 否则优先扩容（因为扩容能减少冲突，比树化更划算）

所以：**链表>=8 且容量>=64 → 树化**，否则扩容优先。

------

## 8. HashMap 常见业务场景

### 场景1：缓存/去重

- 以 id 为 key 缓存对象
- 去重统计：`map.put(x, true)` 或用 `HashSet`（更语义化）

### 场景2：计数/分组（最常用）

- 计数：`map.merge(key, 1, Integer::sum)`
- 分组：`map.computeIfAbsent(groupKey, k -> new ArrayList<>()).add(item)`

### 场景3：索引加速查询（典型业务）

例如按 userId 快速找到订单列表/会话状态等

### 场景4：映射表（配置、字典）

枚举映射、状态码映射、策略路由表（策略模式常用）

------

## 9. 你必须掌握的 API（业务高频）

### put/get/contains/remove

```java
map.put(k, v);
map.get(k);
map.getOrDefault(k, def);
map.containsKey(k);
map.remove(k);
```

### compute / computeIfAbsent / merge（强烈推荐）

**最推荐掌握这三个**，因为线程安全之外，写法更简洁不出错：

```java
map.computeIfAbsent(k, kk -> new ArrayList<>()).add(v);
map.merge(k, 1, Integer::sum);           // 计数
map.compute(k, (kk, vv) -> vv == null ? 1 : vv + 1);
```

------

## 10. 坑点

### 1）HashMap 是线程安全的吗？

不是。并发 put 可能数据丢失、链表结构破坏等。
并发场景用：`ConcurrentHashMap`（或外部加锁）。

### 2）key 为什么要不可变？

如果 key 的 `hashCode/equals` 依赖可变字段，插入后修改字段：

- 查找时 hash 变了 → 找不到
- 会出现“明明 put 过但 get 取不到”的 bug

所以 key 最好：

- 不可变（String、Long、record、不可变对象）
- 正确实现 `equals/hashCode`

### 3）equals 与 hashCode 必须一致

- equals 相等 → hashCode 必须相等
  否则会出现重复 key 或无法命中的问题。

### 4）null key / null value 支持吗？

HashMap 支持（一个 null key，多 null value）。
ConcurrentHashMap 不支持 null（JDK 设计上用 null 表示“无”会有歧义）。

### 5）为什么默认负载因子 0.75？

折中：空间利用率 vs 冲突概率。过大冲突多，过小浪费内存。

### 6）遍历为什么 fail-fast？

迭代器依赖 `modCount`，结构被外部修改会抛 `ConcurrentModificationException`。
这是“尽力而为”的快速失败，不保证一定抛（但通常会）。

------

## 11. JDK17 源码读懂的“关键点”

你不需要背整个源码，抓住这些点就够：

1. **Node/TreeNode 两种节点**：冲突少链表，冲突多红黑树
2. **hash 扰动**：`h ^ (h >>> 16)`
3. **index 计算**：`(n-1) & hash`
4. **resize 迁移优化**：用 `(hash & oldCap)` 决定 stay / move
5. **树化条件**：链表>=8 且容量>=64，否则优先扩容
6. **modCount fail-fast**

面试讲这 6 条，基本就够“源码级”。

------

## 12. 回答模板

> HashMap 底层是数组 table，桶里用链表解决冲突，冲突严重时在 JDK8+ 会树化为红黑树。索引用 `(n-1)&hash`，所以容量是 2 的幂。hash 会做扰动 `h^(h>>>16)` 让高位参与。put 时定位桶，key 相等覆盖，否则追加；size 超过阈值 `capacity*loadFactor` 会扩容到 2 倍，迁移时用 `(hash&oldCap)` 判断元素留在原桶还是移动到 `i+oldCap`。链表长度到 8 且容量>=64 才会树化，否则优先扩容。HashMap 非线程安全，并发用 ConcurrentHashMap 或加锁。

------

## 13. 实战建议

- 预计容量时给初始容量：`new HashMap<>(expectedSize * 4 / 3 + 1)`（避免频繁扩容）
- key 用不可变对象
- 计数/分组优先用 `merge/computeIfAbsent`
- 并发一定别用 HashMap（除非外部锁）

