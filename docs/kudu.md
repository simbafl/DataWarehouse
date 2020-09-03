# Kudu
![kudu](/img/kudu.png)

## 概念
Kudu专为需要对快速（快速变化的）数据进行快速分析的用例而设计。Kudu旨在利用下一代硬件和内存处理技术，显着降低了Apache Impala（正在孵化）和Apache Spark（最初是其他执行引擎）的查询延迟。

Apache Kudu是一个为了Hadoop系统环境而打造的**列存储管理器**，与一般的Hadoop生态环境中的其他应用一样，具有能在通用硬件上运行、水平扩展性佳和支持高可用性操作等功能。

在Kudu出现之前，Hadoop生态环境中的储存主要依赖HDFS和HBase，追求高吞吐批处理的用例中使用HDFS，追求低延时随机读取用例下用HBase，而Kudu正好能兼顾这两者。处于折中的一款存储器。

## Kudu的主要优点：
- 快速处理OLAP（Online Analytical Processing）任务
- 集成MapReduce、Spark和其他Hadoop环境组件
- 与Impala高度集成，使得这成为一种高效访问交互HDFS的方法
- 强大而灵活的统一性模型
- 在执行同时连续随机访问时表现优异
- 通过Cloudera Manager可以轻松管理控制
- 高可用性，tablet server和master利用Raft Consensus算法保证节点的可用
- 支持update、delete操作，这在众多OLAP引擎中是一大亮点

## 常见的应用场景：
- 刚刚到达的数据就马上要被终端用户使用访问到
- 同时支持在大量历史数据中做访问查询和某些特定实体中需要非常快响应的颗粒查询
- 基于历史数据使用预测模型来做实时的决定和刷新
- 要求几乎实时的流输入处理


kudu的主键是聚簇索引，KUDU 的存储也是通过`LSM树`（Log-Structured Merge Tree）来实现的。KUDU 的最小存储单元是 `RowSets`，KUDU 中存在两种 RowSets：`MemRowSets`、`DiskRowSets`，数据先写内存中的 `MemRowSet`，`MemRowSet` 满了后刷到磁盘成为一个 `DiskRowSet`，`DiskRowSet` 一经写入，就无法修改了。

kudu的写过程:

![kudu写](/img/kudu_w.jpg)

kudu的更新过程:

![kudu更新](/img/kudu_update.jpg)

kudu的读过程:

![kudu读](/img/kudu_read.jpg)