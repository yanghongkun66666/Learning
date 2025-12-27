在 **JDK 17** 里，`java.util.Arrays.sort(...)` 并不是“一种排序算法打天下”，而是 **按数组元素类型走不同实现**（而且 Javadoc 也强调这些属于 *implementation note*，未来实现允许替换，只要满足规范，比如对象排序必须稳定）。 ([Freedocs](https://freedocs.mi.hdm-stuttgart.de/doc/openjdk-17-doc/api/java.base/java/util/Arrays.html))

## 1) 基本类型数组（int/long/double…）：Dual-Pivot Quicksort

像 `sort(int[]) / sort(long[]) / sort(double[]) ...` 这类 **基本类型数组**，JDK 17 的实现说明是 **Dual-Pivot Quicksort（双轴快速排序）**。 ([Freedocs](https://freedocs.mi.hdm-stuttgart.de/doc/openjdk-17-doc/api/java.base/java/util/Arrays.html))

它的核心思路：

- **选两个 pivot**（p、q，p ≤ q），一次划分成 3 段：`< p`、`p..q`、`> q`
- 对三段递归/迭代继续分区直到足够小
- 一般是 **原地排序（in-place）**，因此 **不保证稳定性**（相等元素相对顺序可能变化——对基本类型通常也没“对象身份”的稳定性需求）
- JDK 文档层面给出的性能描述是 **O(n log n)**，并且通常比传统单轴快排更快 ([Freedocs](https://freedocs.mi.hdm-stuttgart.de/doc/openjdk-17-doc/api/java.base/java/util/Arrays.html))

> 你在源码里会看到它通常还会对“小分区”走更轻量的处理（比如插入排序类的优化），这是快排工程实现里常见的常数优化；但对外实现说明的“主算法”就是 Dual-Pivot Quicksort。 ([Freedocs](https://freedocs.mi.hdm-stuttgart.de/doc/openjdk-17-doc/api/java.base/java/util/Arrays.html))

## 2) 对象数组（Object[] / T[]）：TimSort（稳定、可利用局部有序）

像 `sort(Object[]) / sort(T[], Comparator)` 这类 **对象数组**，JDK 17 明确保证：**稳定（stable）**，并采用 **稳定、自适应、迭代的归并排序变体**，并注明该实现改编自 Python 的 **TimSort**。 ([Freedocs](https://freedocs.mi.hdm-stuttgart.de/doc/openjdk-17-doc/api/java.base/java/util/Arrays.html))

TimSort 的关键机制（理解“底层原理”最重要的几步）：

- **识别天然有序的 run**：从左到右扫描，找连续递增/递减段；遇到递减段会反转成递增段（因此能同时利用升序/降序局部有序）。 ([Freedocs](https://freedocs.mi.hdm-stuttgart.de/doc/openjdk-17-doc/api/java.base/java/util/Arrays.html))
- **run 太短则扩展**：用低成本方法（常见是二分插入式的扩展）把 run 补到一个最小长度，减少后续合并次数（工程优化）。
- **用栈管理 runs 并按规则合并**：维持一些合并不变量，让整体合并更高效，避免退化。
- **额外空间**：JDK 17 文档写得很直白——临时空间从“几乎有序时的小常数”到“随机序时约 n/2 个引用”。 ([Freedocs](https://freedocs.mi.hdm-stuttgart.de/doc/openjdk-17-doc/api/java.base/java/util/Arrays.html))
- **性能**：部分有序时比较次数可能远少于 n log n；几乎有序时接近 n 次比较。 ([Freedocs](https://freedocs.mi.hdm-stuttgart.de/doc/openjdk-17-doc/api/java.base/java/util/Arrays.html))

同时，Javadoc 也提醒对象排序可能抛：

- `ClassCastException`：元素彼此不可比较/比较器不支持 ([Freedocs](https://freedocs.mi.hdm-stuttgart.de/doc/openjdk-17-doc/api/java.base/java/util/Arrays.html))
- （可选）`IllegalArgumentException`：比较器/Comparable 合约被违反时实现可能检测出来 ([Freedocs](https://freedocs.mi.hdm-stuttgart.de/doc/openjdk-17-doc/api/java.base/java/util/Arrays.html))

## 3) 你怎么快速判断自己调用会走哪条路？

- `int[]/long[]/double[]...` → **Dual-Pivot Quicksort** ([Freedocs](https://freedocs.mi.hdm-stuttgart.de/doc/openjdk-17-doc/api/java.base/java/util/Arrays.html))
- `Integer[]/String[]/MyObj[]`（对象数组）→ **TimSort（稳定）** ([Freedocs](https://freedocs.mi.hdm-stuttgart.de/doc/openjdk-17-doc/api/java.base/java/util/Arrays.html))
- 另外还有 `parallelSort`：会走并行框架（ForkJoinPool）并调用相应实现（但你问的是 `sort`，这里不展开）。 ([Freedocs](https://freedocs.mi.hdm-stuttgart.de/doc/openjdk-17-doc/api/java.base/java/util/Arrays.html))  并行排序？

