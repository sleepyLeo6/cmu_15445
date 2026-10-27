理解角度：
- 数据库整体管理
- 内存结构
- 磁盘结构
- 读写流程
- 后台任务

核心观点：**RocksDB 不只是一个 LSM-Tree，而是一个围绕 LSM-Tree 构建的完整嵌入式存储引擎。**

## 写入与存储结构

下面按 [参考图](https://user-images.githubusercontent.com/62277872/119747261-310fb300-be47-11eb-92c3-c11719fa8a0c.png) 重绘，展示典型的 **Leveled Compaction** 结构。箭头表示数据流或元数据关系，不表示文件的物理排列。

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontSize": "14px", "lineColor": "#555555", "clusterBkg": "#ffffff", "clusterBorder": "#cccccc", "edgeLabelBackground": "#ffffff"}, "flowchart": {"curve": "basis", "nodeSpacing": 18, "rankSpacing": 32, "padding": 12}}}%%
flowchart TB
    writes["Writes / 写入请求"]

    subgraph shared["数据库共享的持久化文件"]
        direction LR
        wal[("Write-Ahead Log<br/>WAL：追加记录更新")]
        manifest[("MANIFEST<br/>记录版本与文件集合的变化")]
    end

    subgraph cf["每个 Column Family 各自维护"]
        direction TB

        subgraph memory["Memory / 内存"]
            direction LR
            mem("Mutable MemTable<br/>接收新写入")
            imm("Immutable MemTables<br/>冻结后等待刷盘")
            mem -->|"达到阈值后冻结并切换新表"| imm
        end

        subgraph disk["Persistent Media / 持久存储：不可变 SST 文件"]
            direction TB

            subgraph l0["L0 · Key 范围可重叠"]
                direction LR
                s00("SST file")
                s01("SST file")
                s02("SST file")
                s00 ~~~ s01 ~~~ s02
            end

            subgraph l1["L1 · 同层 Key 范围不重叠"]
                direction LR
                s10("SST file")
                s11("SST file")
                s12("SST file")
                s10 ~~~ s11 ~~~ s12
            end

            subgraph l2["L2 · 同层 Key 范围不重叠"]
                direction LR
                s20("SST file")
                s21("SST file")
                s22("SST file")
                s23("SST file")
                s20 ~~~ s21 ~~~ s22 ~~~ s23
            end

            more["⋮"]

            subgraph ln["Ln · 同层 Key 范围不重叠"]
                direction LR
                sn0("SST file")
                sn1("SST file")
                sn2("SST file")
                sn3("SST file")
                sn4("SST file")
                sn0 ~~~ sn1 ~~~ sn2 ~~~ sn3 ~~~ sn4
            end

            %% 跨层连线只连接分组，保留每层内部的 LR 横向排列。
            l0 -->|"Compaction / 归并"| l1
            l1 -->|"Compaction"| l2
            l2 --> more --> ln
        end

        memory -->|"Flush / 刷盘"| l0
    end

    writes -->|"1. 追加日志（启用 WAL 时）"| wal
    wal -->|"2. 更新 MemTable"| memory

    classDef input fill:#fff4d6,stroke:#b7791f,color:#222;
    classDef ram fill:#e8f2ff,stroke:#3973ac,color:#222;
    classDef sst fill:#eaf6eb,stroke:#43834a,color:#222;
    classDef metadata fill:#f2ebfa,stroke:#8055a6,color:#222;
    classDef spacer fill:none,stroke:none,color:#666;
    class writes input;
    class mem,imm ram;
    class s00,s01,s02,s10,s11,s12,s20,s21,s22,s23,sn0,sn1,sn2,sn3,sn4 sst;
    class wal,manifest metadata;
    class more spacer;
    style cf fill:none,stroke:#777,stroke-width:2px,stroke-dasharray:5 5
    style shared fill:#faf7fd,stroke:#c4b2d6
    style memory fill:#f5f9ff,stroke:#a3bdd9
    style disk fill:#fafcf9,stroke:#a7c4aa
    style l0 fill:#ffffff,stroke:#b9cbb9
    style l1 fill:#ffffff,stroke:#b9cbb9
    style l2 fill:#ffffff,stroke:#b9cbb9
    style ln fill:#ffffff,stroke:#b9cbb9
```

- **WAL 与 MemTable**：正常启用 WAL 的写入先记录日志，再更新 MemTable；同步写还会在返回成功前等待 WAL 持久化。WAL 由多个 Column Family 共享。
- **Flush**：MemTable 冻结后不再接收更新，后台将其刷成 SST；新的写入由新的 Mutable MemTable 接收。
- **Compaction**：读取输入 SST 并生成新 SST，而不是修改已有文件；只有不再需要的旧版本和删除标记才可被清理。图中连线只示意层间流向，实际一次合并可能涉及多个文件。
- **层级范围**：L1 到 Ln 的“不重叠”指同一层内不同文件的 Key 范围；不同层之间仍可能重叠。
- **MANIFEST**：记录恢复 SST 文件集合与层级版本所需的元数据；尚未刷盘的更新由 WAL 重放恢复。

