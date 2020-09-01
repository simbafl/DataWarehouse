Presto和Impala更像师兄弟，分别来自FaceBook和Google，也同样属于RLOAP模型设计。 细细研究还是有些区别。

# Impala与Hive
> 通过几方面对比下impala与hive

#### 数据存储
二者使用相同的存储数据池都支持把数据存储于HDFS, HBase。

#### 元数据
两者使用相同的元数据。

#### SQL解释处理
比较相似都是通过词法分析生成执行计划。

#### 执行计划
**Hive**: 依赖于MapReduce执行框架，执行计划分成 map->shuffle->reduce->map->shuffle->reduce…的模型。如果一个Query会 被编译成多轮MapReduce，则会有更多的写中间结果。由于MapReduce执行框架本身的特点，过多的中间过程会增加整个Query的执行时间。

**Impala**: 把执行计划表现为一棵完整的执行计划树，可以更自然地分发执行计划到各个Impalad执行查询，而不用像Hive那样把它组合成管道型的 map->reduce模式，以此保证Impala有更好的并发性和避免不必要的中间sort与shuffle。

#### 内存使用
**Hive**: 在执行过程中如果内存放不下所有数据，则会使用外存，以保证Query能顺序执行完。每一轮MapReduce结束，中间结果也会写入HDFS中，同样由于MapReduce执行架构的特性，shuffle过程也会有写本地磁盘的操作。

**Impala**: 在遇到内存放不下数据时，当前版本1.0.1是直接返回错误，而不会利用外存，以后版本应该会进行改进。这使用得Impala目前处理Query会受到一 定的限制，最好还是与Hive配合使用。Impala在多个阶段之间利用网络传输数据，在执行过程不会有写磁盘的操作（insert除外）。

#### 数据流
**Hive**: 采用推的方式，每一个计算节点计算完成后将数据主动推给后续节点。

**Impala**: 采用拉的方式，后续节点通过getNext主动向前面节点要数据，以此方式数据可以流式的返回给客户端，且只要有1条数据被处理完，就可以立即展现出来，而不用等到全部处理完成，更符合SQL交互式查询使用。

#### 调度
**Hive**: 任务调度依赖于Hadoop的调度策略。

**Impala**: 调度由自己完成，目前只有一种调度器simple-schedule，它会尽量满足数据的局部性，扫描数据的进程尽量靠近数据本身所在的物理机器。调度器 目前还比较简单，在SimpleScheduler::GetBackend中可以看到，现在还没有考虑负载，网络IO状况等因素进行调度。但目前Impala已经有对执行过程的性能统计分析，应该以后版本会利用这些统计信息进行调度吧。

#### 容错
**Hive**: 依赖于Hadoop的容错能力。

**Impala**: 在查询过程中，没有容错逻辑，如果在执行过程中发生故障，则直接返回错误（这与Impala的设计有关，因为Impala定位于实时查询，一次查询失败， 再查一次就好了，再查一次的成本很低）。但从整体来看，Impala是能很好的容错，所有的Impalad是对等的结构，用户可以向任何一个Impalad提交查询，如果一个Impalad失效，其上正在运行的所有Query都将失败，但用户可以重新提交查询由其它Impalad代替执行，不会影响服务。对于State Store目前只有一个，但当State Store失效，也不会影响服务，每个Impalad都缓存了State Store的信息，只是不能再更新集群状态，有可能会把执行任务分配给已经失效的Impalad执行，导致本次Query失败。

#### 适用面
**Hive**: 复杂的批处理查询任务，数据转换任务。

**Impala**：实时数据分析，因为不支持UDF，能处理的问题域有一定的限制，与Hive配合使用，对Hive的结果数据集进行实时分析。

# Impala与Presto

对比:
```
1. Impala的计算速度是其一大优点，多表查询性能和Presto差不多，单表查询方面却不如Presto好。
2. Impala不支持update、delete操作，不支持Date数据类型，不支持ORC文件格式等，并且Impala在查询时占用的内存很大。比如多表join时，Impala占用的内存比presto要多。
3. Impala对Kudu的支持非常友好。
4. presto的社区活跃度更高。
5. Presto支持数据源丰富，生态非常广泛，可以对接很多的数据库。有兴趣看官网。
```

所以从整体性能对比看，两者差不多，但是Impala更重些，使用起来没有Presto方便。

公司目前的架构是：
- Presto + Kudu 做实时计算
- Spark + HDFS + ORC 做离线计算