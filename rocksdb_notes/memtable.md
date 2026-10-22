
# 实现方式

- Skiplist MemTable
- Hash Skiplist MemTable


# Skiplist MemTable

## 特点

### concurrent insert

### insert with hint

insert with hint 是什么意思

Insert with hint（带提示的插入），就是插入时提供一个“新元素可能应该放在哪里”的线索，减少查找插入位置的开销。

结合前面的 LSM Tree 语境，它通常指 MemTable 的跳表（Skip List）插入优化。

普通插入需要先从跳表顶部开始搜索，找到各层的前驱节点，再插入：

已有 key：10 → 20 → 30 → 40
插入 25：先搜索，确定它在 20 和 30 之间

如果连续插入的 key 比较接近，就可以保留上一次搜索得到的位置，作为下一次插入的 hint：

插入 25：正常搜索，并保留位置信息
插入 26：利用 hint，从附近开始定位
插入 27：继续复用附近的位置

因此：

- hint 准确：少走很多搜索路径，插入更快。
- hint 不适用：检查后重新搜索，仍然保证有序性和正确性。
- 顺序写入或局部有序写入：通常更容易受益。

Hint 是一种可复用的位置线索；在跳表里，它可能记录多层的前驱、后继信息，而不只是一个节点。

### skiplist memtable 内部结构

它的核心结构是：底层存放所有有序条目，上层用稀疏指针加速查找。

```
Head
L3  ──────────────────────→ 40
L2  ──────────→ 20 ────────→ 40
L1  ──→ 10 ───→ 20 ────────→ 40 ──→ 50
L0  ──→ 10 ───→ 20 ──→ 30 ─→ 40 ──→ 50
```

这里同一个 key 在不同层的出现，表示同一个节点拥有多层指针，数据不会在每一层重复存储。

一个节点里有什么？

概念上可以理解为：

```
Node
├── Entry
│   ├── Internal Key
│   │   ├── User Key
│   │   ├── Sequence Number（版本号）
│   │   └── Type（值 / 删除标记）
│   └── Value
└── Next pointers
    ├── next[0]
    ├── next[1]
    └── ...         ← 节点越高，指针越多
```

具体实现可能把 Entry 放在节点后面的连续内存里，也可能由节点指向另一个内存位置。

为什么要存版本号？

MemTable 中，同一个 user key 可以有多个版本。例如：

| User Key | Sequence | Type | Value |
| -------- | -------- | ---- | ----- |
| apple    | 103 | DELETE | — |
| apple    | 102 | VALUE | red |
| apple    | 100 | VALUE | green |
| banana | 101 | VALUE | yellow |

在 LevelDB/RocksDB 常见的排序方式中，user key 升序，同一个 user key 的版本号降序。这样查找 apple 时，就能根据读取时的快照版本，找到应当可见的记录。删除操作也通过插入删除标记来表示。

查找和插入怎么进行？

- 查找：从最高层开始，向右走；快超过目标时，下降一层，最终在底层定位。
- 插入：先找到每一层的前驱，再随机决定新节点高度，将它接入对应层。
- 遍历 / Flush：沿着 next[0] 就能按序访问全部条目。

查找和插入的期望复杂度都是 O(log n)。节点内存通常由 Arena 统一分配，整个 MemTable 释放时再批量回收。

### rocksdb 插入节点的流程

rocksdb 插入节点的流程是怎么样的

结合前面的 Arena 和 MemTable，RocksDB 的跳表节点插入流程是：分配内存 → 填入数据 → 定位各层位置 → 从底层向上链接。

这里讨论跳表 MemTable 使用的 InlineSkipList，从 MemTable::Add() 开始。

1. 随机决定高度，分配节点内存

调用关系大致是：

MemTable::Add()
  → SkipListRep::Allocate()
    → InlineSkipList::AllocateKey()
      → RandomHeight()
      → AllocateNode()

节点高度决定它有多少层 next 指针。节点和条目存储空间一起通过 allocator 分配，最终由 MemTable 的 Arena 管理。源码：SkipListRep (https://github.com/facebook/rocksdb/blob/main/memtable/skiplistrep.cc)、InlineSkipList (https://github.com/facebook/rocksdb/blob/main/memtable/inlineskiplist.h)

2. 填入 key、sequence 和 value

分配完成后，MemTable::Add() 将记录编码到这块内存中：

[Internal Key 长度]
[User Key][Sequence + Type]
[Value 长度][Value]
[可选校验信息]

此时节点尚未链接进跳表，读线程还不能通过跳表找到它。源码：MemTable::Add (https://github.com/facebook/rocksdb/blob/main/db/memtable.cc)

3. 找到各层的插入位置

比较 Internal Key，找到新节点 x 在各层的前驱、后继，记录在 Splice 中：

prev[i] → next[i]

需要变成：

prev[i] → x → next[i]

有 hint 时，会检查并复用缓存的位置；不适用的部分重新搜索。

4. 从 Level 0 开始，逐层链接

非并发插入的核心操作可以简化为：

```cpp
for (int i = 0; i < height; ++i) {
    x->next[i] = next[i];   // 先设置自己的后继
    prev[i]->next[i] = x;   // 再让前驱指向自己
}
```

先接入底层，再接入上层：底层承载全部条目，上层只是搜索捷径。发布指针使用相应的内存序，保证读线程看到节点时，也能看到已经初始化的数据。

并发插入使用 CAS 更新前驱指针；如果其他线程先修改了链接，就重新定位该层并重试。普通 Insert() 则要求外部保证写入之间的同步。源码：插入实现及并发约定 (https://github.com/facebook/rocksdb/blob/main/memtable/inlineskiplist.h)

例如，新插入 (apple, seq=110) 后，底层可能变成：

原来：
Head → (apple,100) → (banana,108)

插入后：
Head → (apple,110) → (apple,100) → (banana,108)

旧版本节点仍然保留；这次插入主要做的是分配一个新节点并调整指针。