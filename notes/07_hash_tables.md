# intro 

这节课的核心不只是理解哈希表如何实现，而是回答三个数据库系统问题：
1. 为什么 DBMS 需要 Hash Table？
2. 当 Hash Collision（哈希冲突）发生时，DBMS 如何高效地处理？
3. 当存储的数据量不断增长时，Hash Table 如何动态扩容，而不必重建整个数据结构？


**Part 1：Background & Hash Functions**<br>
Pages 3–17 · <br>
哈希表在 DBMS 中的作用、时间复杂度、Hash Function 和 Hashing Scheme。

**Part 2：Linear Probe Hashing**<br>
Pages 18–65 · <br>
线性探测、碰撞处理、Load Factor、Tombstone、键值存储与非唯一键。

**Part 3：Cuckoo Hashing**<br>
Pages 66–80 · <br>
多哈希函数、元素驱逐、重新定位，以及为什么查询能保持固定数量的位置检查。

**Part 4：Chained Hashing**<br>
Pages 81–95 · <br>
Bucket 链表、Overflow Bucket，以及利用 Filter 优化查询。

**Part 5：Extendible Hashing**<br>
Pages 96–119 · <br>
Global Depth、Local Depth、Directory 和 Bucket Split。这是本讲的重要难点。

**Part 6：Linear Hashing**<br>
Pages 120–149 · <br>
Split Pointer、渐进式扩容、多哈希函数切换和删除后的收缩。

**Part 7：Summary & Comparison**<br>
Pages 150–152 · <br>
比较各种 Hashing Scheme，并解释为什么一般的数据库索引更倾向于使用 B+ Tree。

# part 1 background & hash functions

## 1 为什么 DBMS 需要 Hash Table？

### 1.1 Hash table 在哪里被使用

这次学习的 Hash Table 属于 Access Methods（访问方法） 的重要数据结构。

![hash tables](./img/07_access_methods.png)

这里需要建立一个区别：
- Buffer Pool Manager 解决的是：如何通过 Page ID 获取 Page，以及如何管理缓存。
- Access Methods 解决的是：如何利用数据结构快速找到所需记录或数据项。
因此，Hash Table 并不是要替代之前学过的 Page Layout 或 Buffer Pool，而是在这些基础机制之上提供更高效的访问能力。

### 1.2 Hash table 被用于什么数据结构？

| 类型 | 作用 | 例子 |
| --- | --- | --- |
|Internal Meta-data | 管理数据库内部状态 | Page Table、Page Directory |
|Core Data Storage | 存储实际记录 | Tuple Storage |
|Temporary Data Structures | 查询期间临时构建 | Hash Join 使用的 Hash Table |
|Table Indexes | 加速查找特定 Tuple | Hash Index、B+ Tree Index |

例如，执行以下 SQL：
```sql
SELECT *
FROM Students S
JOIN Scores C
  ON S.id = C.student_id;
```

一种执行方式是使用 Hash Join：
1. 扫描其中一张表，将连接键及对应数据插入 Hash Table。
2. 扫描另一张表，对连接键计算 Hash。
3. 在 Hash Table 中查找匹配项，完成 Join。

## 2 Hash Table 基本原理

### 2.1 Hash Table 基本原理

### 2.2 一个简单例子

## 3. Hash Collision

### 3.1 理想 Hash Table 的三个不现实假设

PPT Page 13 给出了三个假设：
1. 元素数量提前已知，而且不变。
2. 每个 Key 都是唯一的。
3. 存在无冲突的 Perfect Hash Function。

## 4. Hash Table 的性能为什么很好

|性能指标|时间复杂度|
|---|---|
|Space Complexity| $O(n)$ |
|Average Lookup| $O(1)$ |
|Worst-case Lookup| $O(n)$ |

### 4.1 为什么平均时间复杂度是 O(1)？

### 4.2 为什么最坏情况下是 O(n)？

## 5. Hash Table 的两个核心设计决策

PPT 把设计 Hash Table 的问题分成两个方面：
![hash table design](./img/07_hash_table_design.png)

**Design Decision 1：Hash Function**<br>
解决的问题是：<br>
如何把一个非常大的 Key Space 映射到一个有限的数值空间？<br>
重点考虑：
$$
\boxed{\text{Hash Speed}\quad\text{vs.}\quad\text{Collision Rate}}
$$

理想情况下，我们希望 Hash Function：
- 计算速度快。
- 输出分布较均匀。
- 尽可能减少碰撞。

**Design Decision 2：Hashing Scheme**<br>
解决的问题是：<br>
当多个 Key 映射到冲突的位置时，数据结构应该如何存储它们？<br>
例如：
- Linear Probing：寻找下一个可用 Slot。
- Cuckoo Hashing：使用多个候选位置，必要时移动已有元素。
- Chained Hashing：使用 Bucket Chain 存储冲突元素。

这几种方法有不同的空间开销、查询性能与更新成本。

## 6. DBMS 如何选择 Hash Function？

### 6.1 为什么 DBMS 通常不使用 SHA-256？

因为 DBMS 内部 Hash Table 的主要需求通常是：
- 高吞吐量的 Hash 计算。
- 足够好的分布特性。

而 SHA-256 这样的密码学 Hash Function 还追求抗碰撞攻击、原像抗性等安全性质，因此通常具有不必要的额外计算开销。

注意，这只是针对一般 DBMS 内部非安全用途的 Hash Table，并不代表密码学 Hash 在数据库系统的所有场景中都不适用。

### 6.2 PPT 提及的 Hash Functions

|Hash Function|PPT 强调的特点|
|---|---|
|[CRC-64](https://create.stephan-brumme.com/crc32/)	| 最初用于网络错误检测 |
|[MurmurHash](https://github.com/aappleby/smhasher)	| 快速通用哈希 |
|[CityHash](https://github.com/google/cityhash)	| 针对短 Key 优化 |
|[XXHash](https://xxhash.com/)	| 高性能的通用哈希 |
|[FarmHash](https://github.com/google/farmhash)	| CityHash 的后续改进 |
|[RapidHash](https://github.com/Nicoshev/rapidhash)	| 避免依赖特定体系结构指令 |

XXHash 是 state-of-the-art

**Hash Function 的选择会影响 Hash Table 的计算成本和冲突分布，而 Hashing Scheme 决定如何处理实际发生的冲突。**

# part 2 linear probe hashing

这一部分讨论的核心问题是：

**当两个 Key 被 Hash Function 映射到同一个 Slot 时，如何在不增加链表的情况下存储和查找它们？**

## 1. linear probing 设计动机

## 2. lookup: 如何查找一个 key

## 3. load factor 为什么哈希表不能太满

课程使用 Load Factor（负载因子）衡量哈希表的填充程度：
$$
\boxed{\alpha=\frac{\text{Active Keys}}{\text{Number of Slots}}}
$$

例如，100 个 Slots 存储 70 个有效 Key：
$$
\alpha=\frac{70}{100}=0.7
$$

随着 $\alpha$ 上升，空闲位置越来越少，Linear Probing 通常需要检查更多 Slots。

尤其容易出现 Primary Clustering（一次聚集）：

多个 Key 逐渐形成一段连续被占用的区域，新发生的碰撞又可能使这个连续区域继续增长。

结果是，查询和插入所需的 Probe 次数增加。

![hash slots occupancy](./img/07_hash_slots_occupancy.png)

### 3.1 什么时候扩容

系统可以在负载因子达到预设阈值后：

首先，分配一个通常为原来两倍大小的新哈希表；然后，重新对已有 Key 计算地址并插入新表。

为什么不能简单复制原数组？

因为 Slot Index 依赖表的大小：
$$
\operatorname{index}(k)=h(k)\bmod N
$$

当 \(N\) 从 8 变为 16，原先映射到 Slot 2 的 Key 18：
$$
18\bmod8=2
$$

现在会映射到：
$$
18\bmod16=2
$$

而 Key 10 仍映射到 10，Key 2 仍映射到 2。原来的碰撞关系就可能改变。

因此需要 Rehashing（重新散列并放置记录）。

这也解释了静态哈希表的局限：扩容可能需要重新处理大量已有记录

## 4. Deletion：为什么删除是 Linear Probing 的难点？

这是本部分需要重点掌握的内容。

继续使用之前的哈希表：
|Slot|Key|
|---|---|
|2|2|
|3|10|
|4|18|
|5|EMPTY|

假设我们执行：`Delete(10)`

### 4.1 错误的删除方式：直接清空 Slot
如果把 Slot 3 直接设成 EMPTY，哈希表变成：

|Slot|Key|
|---|---|
|2|2|
|3|EMPTY|
|4|18|
|5|EMPTY|

现在执行：`Get(18)`


系统首先检查 Slot 2，发现不是 18。

继续检查 Slot 3，发现 EMPTY，于是认为目标不存在，停止查找。

但实际上 Key 18 就在 Slot 4。

问题的本质在于：**删除操作破坏了原来的 Probe Sequence（探测序列）的连续性。**

### 4.2 Approach 1：Movement
PPT Page 35 介绍第一种方案：

删除元素后，重新组织受影响的后续元素，恢复有效的探测路径。

例如，移除 Key 10 后，可以将原来存储在 Slot 4 的 Key 18 向前移动到 Slot 3，因为其探测序列允许这一位置。

但这种方法可能需要移动大量记录，成本较高。课程 Notes 指出，这不是通常采用的方案。

### 4.3 Approach 2：Tombstone（墓碑标记）
PPT Page 52 引入更常见的方案。

删除一个 Key 时，不立即把 Slot 标记为 EMPTY，而是把它标记为 DELETED（Tombstone）。

这样哈希表中的 Slot 就有三种逻辑状态：
|Slot 状态|含义|Lookup 操作|
|---|---|---|
|EMPTY|从未被占用的空位 | 可以停止查找|
|OCCUPIED|存储有效记录|检查 Key|
|DELETED|原有记录已经删除|继续探测|


### 4.4 Tombstone 的代价
Tombstone 虽然可以解决删除问题，但不能无成本地长期保留。

因为 Lookup 必须跳过 Tombstone 继续探测，因此大量 Tombstones 会导致查询效率下降。

PPT 还指出 Tombstone 对应的 Slot 可以被新的 Key 复用，系统也可能需要周期性地进行 Garbage Collection。

补充一个实现细节：如果 Hash Table 要保证 Key 唯一，那么插入时遇到 Tombstone，也不能一律立即插入；还需要遵守查重规则，避免后续探测位置已有相同 Key。

## 5. Hash Table 如何存储 Key/Value？