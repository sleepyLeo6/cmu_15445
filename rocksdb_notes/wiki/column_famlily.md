[rocksdb wiki: column family](https://github.com/facebook/rocksdb/wiki/Column-Families)

# column family 是什么？

在 RocksDB 中，列族（Column Family，CF）通常用于把“属于同一个数据库、但存储和访问特征不同”的数据分开管理，同时保留跨组原子写入的能力。

理解 RocksDB 时，把 CF 看成“有独立存储配置的 KV 集合”更直接。

每个 CF 都是一个独立的有序 key-value 空间，拥有自己的 MemTable 和 SST 文件，但共享数据库的预写日志（WAL）。同一个 key 可以出现在不同 CF 中，表示不同的记录。官方说明 (https://github.com/facebook/rocksdb/wiki/Column-Families)

```
一个 RocksDB 实例
│
├── users CF
│   └── 自己的 MemTable → 自己的 SST / LSM 层级
│
├── email_index CF
│   └── 自己的 MemTable → 自己的 SST / LSM 层级
│
└── 共享 WAL
```

# 使用 column family 的场景

**最典型的场景：主数据与二级索引。**

假设应用需要根据用户 ID 和邮箱查询用户：

```
users CF：
    user:42 → {name: "小明", email: "a@example.com"}

email_index CF：
    a@example.com → 42
```

创建用户时，两份数据必须一起更新，否则可能出现“用户存在，但邮箱索引找不到”的状态。可以把两次写入放进同一个 WriteBatch：

```
一个原子写入批次：
    users CF       写入 user:42 → 用户信息
    email_index CF 写入 邮箱 → 42
```

RocksDB 支持跨 CF 的原子批量写入。
- 它不会自动维护索引，索引内容仍由应用生成；
- 若要分多次读取并保持一致视图，可使用同一个快照（Snapshot）。

**另一类场景：数据需要不同的存储策略。**

下面是设计示例，并非固定配置建议：

| 数据组 | 特征 | 分成 CF 的目的 |
|---|---|---|
| 用户资料与访问计数 | 资料较少修改，计数频繁更新 | 分别设置写缓冲和压缩策略，避免两类数据混在同一批 SST 中被整理 |
| 小型索引与大型正文 | 索引频繁随机读取，正文较大 | 分别选择块大小、压缩和过滤器配置 |
| 不同业务的数据集合 | 需要独立遍历、配置或整体删除 | 为每组数据提供独立命名空间和管理边界 |

CF 支持独立配置；具体哪些选项可设置、能否动态修改，需要看对应版本的 ColumnFamilyOptions。配置边界 (https://github.com/facebook/rocksdb/wiki/Column-Families#reference)

但 CF 提供的资源隔离有限。

它们仍共享进程、设备和部分数据库级资源；Block Cache 可以配置为共享，也可以分开。一个 CF 的 compaction 跟不上，触发写停顿（Write Stall）时，还可能影响整个 DB 的写入。因此，把不同租户放进不同 CF，并不自动获得严格的性能隔离。写停顿的影响范围 (https://github.com/facebook/rocksdb/wiki/Write-Stalls#causes-of-write-stalls)

选择时可以这样判断：

- 只是区分 key 的含义，配置和生命周期都相同：一个 CF 加 key 前缀通常就够了。
- 需要独立存储配置、独立 SST 管理，或跨组原子更新：适合使用多个 CF。
- 需要独立打开、关闭、恢复或更强的故障与资源隔离：考虑多个 DB，必要时使用独立进程。

这里的“列族”也不意味着每个关系型字段都要建一个 CF。RocksDB 的 CF 本质上管理一组完整的 key-value；是否把用户的 name、email 分开保存，由应用的数据模型决定。