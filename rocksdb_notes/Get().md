
# 一次 point lookup 会经历什么？

## 流程

Full Bloom Filter 的 SST，典型的点查询逻辑是：

```mermaid
%%{init: {"flowchart": {"nodeSpacing": 20, "rankSpacing": 25}}}%%
flowchart TD
    A["Get(key)"] --> B["查询 Mutable MemTable"]
    B --> C("已获得确定结果?")
    C -->|是| Z["返回结果"]
    C -->|否| D["查询 Immutable MemTables"]
    D --> E("已获得确定结果?")
    E -->|是| Z
    E -->|否| F["定位候选 SST 文件"]
    F --> G("Bloom Filter 排除?")
    G -->|是| H["跳过该 SST"]
    G -->|否| I["Index Block 定位 Data Block"]
    I --> J["Block Cache / 磁盘读取"]
    J --> K["Data Block 内搜索 Key"]
    K --> L["检查 Sequence Number 和记录类型"]
    L --> M["返回结果或继续搜索"]
    H --> M
```

## 二、通过一个具体例子理解查询

假设执行：
```
std::string value;

db->Get(
    rocksdb::ReadOptions(),
    "cat",
    &value
);
```

我们假设：
1. Mutable 和 Immutable MemTable 都没有 cat。
2. RocksDB 找到了一个可能包含 cat 的 SST。
3. 该 SST 配置了 Full Bloom Filter。
SST 内包含以下记录：

```mermaid
%%{init: {"flowchart": {"nodeSpacing": 16, "rankSpacing": 16, "padding": 8, "wrappingWidth": 480}}}%%
flowchart TB
    subgraph SST["000123.sst · 按 Internal Key 排序"]
        direction TB
        D1["Data Block 1<br/>apple@105 → A<br/>banana@103 → B"]
        D2["Data Block 2<br/>cat@102 → C<br/>dog@101 → D"]
        D3["Data Block 3<br/>egg@99 → E<br/>fish@98 → F"]
        IDX["Index Block<br/>索引边界 1 → Block 1 的位置<br/>索引边界 2 → Block 2 的位置<br/>索引边界 3 → Block 3 的位置"]
        FILTER["Full Bloom Filter<br/>包含 apple、banana、cat、dog、egg、fish<br/>等 User Key 的过滤信息"]
        D1 ~~~ IDX
        D2 ~~~ IDX
        D3 ~~~ IDX
        IDX ~~~ FILTER
    end

    classDef data fill:#eaf5ee,stroke:#addfbb,color:#111;
    classDef target fill:#e2f2e7,stroke:#16a34a,stroke-width:3px,color:#111;
    classDef index fill:#edf2ff,stroke:#edf2ff,color:#111;
    classDef filter fill:#f5efff,stroke:#f5efff,color:#111;
    class D1,D3 data;
    class D2 target;
    class IDX index;
    class FILTER filter;
    style SST fill:#fcfcfc,stroke:#ddd,rx:16,ry:16
```

图中 `key@seq` 表示 User Key 与 Sequence Number；绿色加粗框标出本例查询 `cat` 的目标数据块。此图展示逻辑组成，省略其他元数据和 Footer，不表示文件中的物理排列顺序。

下面正式开始查询。

### **Step 1：找到候选 SST 文件**

在调用 SST 内部的 Index Block 之前，RocksDB 必须先知道：

应该去哪个 SST 文件查找？

RocksDB 使用 LSM-Tree 的 Version 元数据，维护每个 SST 文件的 Key 范围。

例如：
```
SST	所在 Level	User Key 范围
000121.sst	L0	apple – dog
000122.sst	L0	banana – fish
000123.sst	L0	apple – fish
000098.sst	L1	apple – mango
```

由于 cat 落在多个文件的 Key 范围内，可能需要检查多个文件。

其中：
- L0 的 SST 文件允许 Key 范围重叠，因此一个 Key 可能对应多个候选文件。
- L1 及更深层级在通常的 Leveled Compaction 下，同一层的 SST Key 范围互不重叠，所以每层至多定位到一个候选文件。

这一步使用的是 SST 文件级别的元数据，还没有进入某个 SST 文件内部的 Data Block。

### **Step 2：Bloom Filter 判断 Key 是否可能存在**

假设现在检查 000123.sst。

Bloom Filter 会对 cat 做成员查询：
$$
\operatorname{MayContain}(\text{"cat"})
$$

可能得到两种结果。

```mermaid
%%{init: {"flowchart": {"nodeSpacing": 16, "rankSpacing": 20, "padding": 10}}}%%
flowchart LR
    Q("Bloom Filter 查询")
    Q --> M("Maybe<br/>Key 可能存在<br/>继续查询 Index Block 和 Data Block")
    Q --> N("Definitely Not<br/>Key 一定不存在<br/>跳过该 SST，无须读取 Data Block")

    classDef maybe fill:#eaf5ee,stroke:#addfbb,color:#111;
    classDef absent fill:#fdf0f0,stroke:#ffbcbc,color:#111;
    class M maybe;
    class N absent;
```


Bloom Filter 内部到底在做什么？
假设有一个 16-bit 的 Bloom Filter：

```mermaid
%%{init: {"flowchart": {"nodeSpacing": 4, "rankSpacing": 4, "padding": 6}, "themeVariables": {"fontSize": "14px"}}}%%
flowchart LR
    subgraph BITS["16-bit Bloom Filter · 格内上方为值，下方为下标"]
        direction LR
        subgraph COL0[" "]
            direction TB
            B0["0<br/>[0]"] ~~~ B8["1<br/>[8]"]
        end
        subgraph COL1[" "]
            direction TB
            B1["1<br/>[1]"] ~~~ B9["0<br/>[9]"]
        end
        subgraph COL2[" "]
            direction TB
            B2["0<br/>[2]"] ~~~ B10["1<br/>[10]"]
        end
        subgraph COL3[" "]
            direction TB
            B3["1<br/>[3]"] ~~~ B11["0<br/>[11]"]
        end
        subgraph COL4[" "]
            direction TB
            B4["0<br/>[4]"] ~~~ B12["0<br/>[12]"]
        end
        subgraph COL5[" "]
            direction TB
            B5["0<br/>[5]"] ~~~ B13["1<br/>[13]"]
        end
        subgraph COL6[" "]
            direction TB
            B6["1<br/>[6]"] ~~~ B14["0<br/>[14]"]
        end
        subgraph COL7[" "]
            direction TB
            B7["0<br/>[7]"] ~~~ B15["1<br/>[15]"]
        end
        COL0 ~~~ COL1 ~~~ COL2 ~~~ COL3 ~~~ COL4 ~~~ COL5 ~~~ COL6 ~~~ COL7
    end

    classDef zero fill:#fcfcfc,stroke:#e5e7eb,color:#111;
    classDef one fill:#dceedd,stroke:#80c98d,color:#111;
    class B0,B2,B4,B5,B7,B9,B11,B12,B14 zero;
    class B1,B3,B6,B8,B10,B13,B15 one;
    style BITS fill:#fcfcfc,stroke:#ddd,rx:12,ry:12
    style COL0 fill:transparent,stroke:transparent
    style COL1 fill:transparent,stroke:transparent
    style COL2 fill:transparent,stroke:transparent
    style COL3 fill:transparent,stroke:transparent
    style COL4 fill:transparent,stroke:transparent
    style COL5 fill:transparent,stroke:transparent
    style COL6 fill:transparent,stroke:transparent
    style COL7 fill:transparent,stroke:transparent
```


教学示例，真实 Bloom Filter 通常更大，并使用专门的哈希计算。
写入时，对某个 Key 使用多个哈希函数：
$$
h_1(k),h_2(k),\ldots,h_r(k)
$$

对应的 Bit 被设置为 1。

查询时，再计算这些哈希位置：
- 只要有一个 Bit 是 0，就能确定 Key 不存在。
- 如果全部是 1，也不能保证 Key 存在，因为可能是其他 Key 共同设置了这些 Bit。
这就是 Bloom Filter 存在 False Positive（假阳性），却不会正常产生 False Negative（假阴性）的原因。

理解重点：Bloom Filter 不是寻找 Key 的具体位置，而是提前判断有没有必要继续寻找。


### **Step 3：Index Block 定位 Data Block**

假设 Bloom Filter 返回 Maybe。

现在 RocksDB 需要知道 cat 究竟在哪一个 Data Block 中。

Index Block 通常保存有序的索引 Key 和 BlockHandle。

对于刚才的 SST，我们可以把 Index Block 简化理解为：

|索引覆盖范围（示意）|BlockHandle|
|---|---|
|apple – banana|(offset=0, size=4096)|
|cat – dog|(offset=4101, size=4096)|
|egg – fish|(offset=8202, size=4096)|

这张表只是帮助理解查找关系。真实 Index Key 通常是根据相邻 Data Block 的边界构造的分隔 Key，不一定直接存储这样的起止范围。

RocksDB 在 Index Block 中搜索 cat，最终定位：
$$
\boxed{\text{BlockHandle}=(4101,4096)}
$$

表示：
- Data Block 位于文件偏移 4101 字节处。
- 其大小为 4096 字节。
这是一个关键区别：

Index Block 定位的是 Data Block，并不是直接定位到其中某条 Key-Value 的字节地址。

因此，它通常还需要在目标 Data Block 内执行一次查找。

### **Step 4：先检查 Block Cache**

得到了 BlockHandle，是否就需要读取磁盘？

不一定。

RocksDB 会尝试从 Block Cache 中获取对应的数据块。

```mermaid
%%{init: {"flowchart": {"nodeSpacing": 16, "rankSpacing": 20, "padding": 10}}}%%
flowchart LR
    NEED("需要 Data Block 2") --> CACHE("Block Cache 查询")
    CACHE -->|Cache Hit| HIT("直接使用内存中的 Block")
    CACHE -->|Cache Miss| MISS("读取 SST 文件<br/>校验、必要时解压<br/>按配置填充 Cache")

    classDef hit fill:#eaf5ee,stroke:#addfbb,color:#111;
    classDef miss fill:#fff5e9,stroke:#f4c99b,color:#111;
    class HIT hit;
    class MISS miss;
```

这里可以联系你之前学习的 CMU 15-445 Buffer Pool。

相似之处在于：
- 都避免反复从磁盘读取相同的数据。
- 都有 Cache Hit 和 Cache Miss。
- 都需要内存淘汰策略。
但 RocksDB 的 Block Cache 主要缓存 SST Block，不等同于传统 DBMS 对可修改 Page 的缓存。

### **Step 5：在 Data Block 内查找 Key**

现在已经拿到 Data Block 2：
(cat, 102) → C
(dog, 101) → D


如何快速找到 cat？

这里就要用到你前面看到的两个概念：
- Prefix Compression
- Restart Point

#### **5.1 为什么引入 Prefix Compression？**

考虑下面几个 Key：
```
student001
student002
student003
student004
student005
```

如果每次都完整存储，就有大量重复的前缀。

例如 student001 和 student002 共有 9 个字节。

RocksDB 可以编码为：
|Key|Shared Bytes|Key Delta|
|---|---|---|
|student001|0|student001|
|student002|9|2|
|student003|9|3|
|student004|9|4|
|student005|9|5|

这样可以显著节省空间。

但同时出现一个新问题：

如果后一个 Key 依赖前一个 Key 才能解码，还怎么进行二分查找？

这就是 Restart Point 的设计动机。

#### **5.2 Restart Point 如何解决这个问题？**

RocksDB 每隔一定数量的 Entry 设置一个 Restart Point。

在 Restart Point 上：
$$
\text{shared\_bytes}=0
$$

意味着该 Key 需要完整存储，不依赖之前的 Key。

例如假设 Restart Interval = 3：

```mermaid
%%{init: {"flowchart": {"nodeSpacing": 16, "rankSpacing": 16, "padding": 8}}}%%
flowchart LR
    R0("Entry 0 · Restart<br/>存储 student001 · shared=0")
    E1("Entry 1 · student002<br/>存储后缀 2 · shared=9")
    E2("Entry 2 · student003<br/>存储后缀 3 · shared=9")
    R0 -->|前缀解码| E1 -->|前缀解码| E2

    R3("Entry 3 · Restart<br/>存储 student004 · shared=0")
    E4("Entry 4 · student005<br/>存储后缀 5 · shared=9")
    E5("Entry 5 · student006<br/>存储后缀 6 · shared=9")
    R3 -->|前缀解码| E4 -->|前缀解码| E5

    R6("Entry 6 · Restart<br/>存储 student007 · shared=0")

    classDef restart fill:#e6eeff,stroke:#80a7ff,stroke-width:2px,color:#111;
    classDef delta fill:#fcfcfc,stroke:#ddd,color:#111;
    class R0,R3,R6 restart;
    class E1,E2,E4,E5 delta;
```

蓝色节点是独立解码的重启点；箭头表示后一条记录需要前一条记录的完整 Key 才能还原。`shared` 表示与前一条 Key 共享的前缀字节数，此例共享前缀为 `student00`（9 字节）。图中省略 Value 和其他编码字段。

这样 RocksDB 就可以先对 Restart Point 对应的完整 Key 进行二分查找。

假设寻找：

student005

过程为：
1. 对 student001、student004、student007 这几个 Restart Key 进行二分搜索。
2. 确定目标位于以 student004 开始的区间。
3. 从 student004 开始逐条解码。
4. 解码 student005，完成匹配。

因此，Data Block 内部不是对所有压缩记录直接进行普通二分搜索，而是通常采用：

Restart Points 上的二分查找 + 区间内的顺序扫描。

这也是为什么 Restart Point 可以同时兼顾压缩率与查询效率。

## 三、找到 Key 后，为什么还要检查 Sequence Number？

到这里，我们成功在 SST 内定位了 cat。

但是，RocksDB 还不能完全忽略 Internal Key 中的其他字段。

假设某个 SST 中有：
```
(cat, 120, Value) → "new"
(cat, 100, Value) → "old"
(cat, 90, Value)  → "older"
```

如果执行普通的最新版本读取，通常应该得到：new

但是，如果使用 Snapshot 呢？

假设快照对应：
$$
\text{Snapshot Sequence Number}=110
$$

此时：

|Internal Key|对 Snapshot 110 是否可见|
|---|---|
|cat@120|不可见|
|cat@100|可见|
|cat@90|可见，但较旧|

最终返回：old

也就是说，在普通的版本记录模型中，RocksDB 寻找满足
$$
\boxed{\text{SequenceNumber}\leq\text{SnapshotSequenceNumber}}
$$
的最新可见版本。

如果遇到的是可见的 Delete Tombstone，则表示该 Key 在该读取视图中已被删除，不能简单地忽略 Tombstone 并返回更旧的 Value。

此外，如果一个 SST 中没有足够的信息完成读取，RocksDB 可能需要继续检查其他 SST。

这正是 RocksDB 的多版本读取与 LSM-Tree 多层存储之间的重要联系。

## 四、一次 Get() 的成本究竟来自哪里？

这部分建议重点掌握，因为你后续学习 CacheLib 和 Kangaroo 时也会再次遇到。

|操作|可能的开销|对应优化|
|---|---|---|
|查找候选 SST|CPU、元数据访问|Level 文件范围索引|
|Bloom Filter|哈希计算、内存访问|快速排除不存在的 Key|
|Index Block 查询|比较、索引块读取|Index Cache、分区 Index|
|Data Block 读取|磁盘 I/O|Block Cache、预取|
|Block 解压|CPU|压缩算法选择|
|Data Block 内查找|CPU、Key 解码|Restart Point、Data Block Hash Index|
|多个 SST 查询|CPU、I/O|Compaction 减少重叠和文件数|

可以观察到三个不同层次的优化：
- Bloom Filter：减少不必要的查找和 I/O。
- Index Block：缩小查询范围，定位目标 Data Block。
- Block Cache：避免重复读取已经访问过的 Block。

它们不是互相替代的关系，而是共同优化一次查询。

