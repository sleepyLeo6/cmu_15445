# sst file 内部是怎么组织的？

https://chatgpt.com/s/t_6ae2b8f84d8c8191980066eda5e3c432

## sst file 整体结构

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontSize": "16px", "fontFamily": "Microsoft YaHei, Arial, sans-serif", "clusterBkg": "#fafbfc", "clusterBorder": "#d9dde3"}, "flowchart": {"htmlLabels": true, "useMaxWidth": true, "wrappingWidth": 700, "nodeSpacing": 8, "rankSpacing": 8, "padding": 12}}}%%
flowchart TB
    subgraph sst["一个 SST 文件（例如 000123.sst）· 磁盘文件"]
        direction TB
        %% 显式指定 HTML 标签宽度，避免默认窄节点导致中文频繁换行。
        d0["<div style='width:680px;text-align:left;line-height:1.7'><b style='font-size:20px'>Data Block 0</b><span style='float:right;color:#555'>Key A–F</span><br/><span style='color:#666'>按 Internal Key 排序的 Key-Value 记录</span></div>"]
        d1["<div style='width:680px;text-align:left;line-height:1.7'><b style='font-size:20px'>Data Block 1</b><span style='float:right;color:#555'>Key G–M</span><br/><span style='color:#666'>每个 Block 包含多条记录，可独立压缩</span></div>"]
        d2["<div style='width:680px;text-align:left;line-height:1.7'><b style='font-size:20px'>Data Block 2</b><span style='float:right;color:#555'>Key N–Z</span><br/><span style='color:#666'>后面还可以继续有更多 Data Blocks</span></div>"]
        filter["<div style='width:680px;text-align:left;line-height:1.7'><b style='font-size:20px'>Filter Block（可选）</b><br/><span style='color:#666'>Bloom Filter / Ribbon Filter 等，判断 Key 是否可能存在</span></div>"]
        index["<div style='width:680px;text-align:left;line-height:1.7'><b style='font-size:20px'>Index Block</b><br/><span style='color:#666'>索引项 → 对应 Data Block 的文件 Offset 和 Size</span></div>"]
        meta["<div style='width:680px;text-align:left;line-height:1.7'><b style='font-size:20px'>其他 Meta Blocks</b><br/><span style='color:#666'>Properties、Range Deletion、Compression Dictionary 等</span></div>"]
        metaindex["<div style='width:680px;text-align:left;line-height:1.7'><b style='font-size:20px'>Metaindex Block</b><br/><span style='color:#666'>记录各类元数据 Block 的位置</span></div>"]
        footer["<div style='width:680px;text-align:left;line-height:1.7'><b style='font-size:20px'>Footer</b><br/><span style='color:#666'>文件末尾的固定长度结构，包含定位和格式识别信息</span></div>"]

        d0 ~~~ d1 ~~~ d2 ~~~ filter ~~~ index ~~~ meta ~~~ metaindex ~~~ footer
    end

    classDef data fill:#fafbfc,stroke:none,color:#222222;
    classDef filterStyle fill:#eaf5f0,stroke:#96cdb9,color:#222222,rx:10,ry:10;
    classDef indexStyle fill:#eaf1fb,stroke:#9ebfee,color:#222222,rx:10,ry:10;
    classDef metaStyle fill:#f7f2e9,stroke:#dec39d,color:#222222,rx:10,ry:10;
    classDef footerStyle fill:#f3e8dc,stroke:#d5ac7a,color:#222222,rx:10,ry:10;
    class d0,d1,d2,meta data;
    class filter filterStyle;
    class index indexStyle;
    class metaindex metaStyle;
    class footer footerStyle;
    style sst fill:#fafbfc,stroke:#d9dde3,stroke-width:1px
```

> 概念示意图：各类元数据 Block 的实际排列顺序、索引组织及 Footer 编码与长度随格式版本变化，并非严格固定为图中顺序。Key 范围仅作示例。

### data block


```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontSize": "16px", "fontFamily": "Microsoft YaHei, Arial, sans-serif"}, "flowchart": {"htmlLabels": true, "useMaxWidth": true, "nodeSpacing": 6, "rankSpacing": 6, "padding": 10}}}%%
flowchart TB
    subgraph lengths[" "]
        direction LR
        shared["<div style='width:200px;text-align:center;line-height:1.6'>shared_bytes<br/><span style='color:#666'>公共前缀长度</span></div>"]
        unshared["<div style='width:200px;text-align:center;line-height:1.6'>unshared_bytes<br/><span style='color:#666'>剩余 Key 长度</span></div>"]
        valueLength["<div style='width:200px;text-align:center;line-height:1.6'>value_length<br/><span style='color:#666'>Value 长度</span></div>"]
        shared ~~~ unshared ~~~ valueLength
    end

    subgraph payload[" "]
        direction LR
        suffix["<div style='width:313px;text-align:center'>Key Suffix</div>"]
        value["<div style='width:313px;text-align:center'>Value</div>"]
        suffix ~~~ value
    end

    lengths ~~~ payload

    classDef header fill:#dfeaf7,stroke:none,color:#111,rx:8,ry:8;
    classDef keyData fill:#e0eeea,stroke:none,color:#111,rx:8,ry:8;
    classDef valueData fill:#f2e9df,stroke:none,color:#111,rx:8,ry:8;
    class shared,unshared,valueLength header;
    class suffix keyData;
    class value valueData;
    style lengths fill:none,stroke:none
    style payload fill:none,stroke:none
```

前三个长度通常使用 varint32 编码，这是经典的 Key-Value 交替存储布局。图中分两行展示，实际字段按 `shared_bytes → unshared_bytes → value_length → Key Suffix → Value` 顺序编码。
### index block

