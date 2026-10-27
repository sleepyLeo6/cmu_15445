[rocksdb wiki: column family](https://github.com/facebook/rocksdb/wiki/Column-Families)

# column family 是什么？

在 RocksDB 中，列族（Column Family，CF）通常用于把“属于同一个数据库、但存储和访问特征不同”的数据分开管理，同时保留跨组原子写入的能力。

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

最典型的场景：主数据与二级索引。

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
