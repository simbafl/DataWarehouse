# shuffle调优

## 前述
大多数Spark作业的性能主要就是消耗在shuffle环节，因为该环节包含了大量的磁盘IO、序列化、网络传输等操作。因此，如果要让作业的性能更上一层楼，就有必要对shuffle过程进行调优。但是影响一个Spark作业性能的因素，主要还是代码开发、资源参数以及数据倾斜，shuffle调优只能在整个Spark作业的性能调优中占到一小部分。

## ShuffleManager发展
在Spark的源码中，负责shuffle过程的`执行`、`计算`和`处理`的组件主要就是ShuffleManager，也即**shuffle管理器**。而随着Spark的版本的发展，ShuffleManager也在不断迭代，变得越来越先进。

在Spark 1.2以前，默认的shuffle计算引擎是`HashShuffleManager`。该ShuffleManager即HashShuffleManager有着一个非常严重的弊端，就是会产生大量的中间磁盘文件，进而由大量的磁盘IO操作影响了性能。

因此在Spark1.2以后的版本中，默认的ShuffleManager改成了`SortShuffleManager`。SortShuffleManager相较于HashShuffleManager来说，有了一定的改进。主要就在于，每个Task在进行shuffle操作时，虽然也会产生较多的临时磁盘文件，但是最后会将所有的临时文件合并（merge）成一个磁盘文件，因此每个Task就只有一个磁盘文件。在下一个stage的shuffle read task拉取自己的数据时，只要根据索引读取每个磁盘文件中的部分数据即可。

## HashShuffleManager原理

### 未经优化的HashShuffleManager

> 方便理解，假设一个前提：每个Executor只有1个CPU core，也就是说，无论这个Executor上分配多少个task线程，同一时间都只能执行一个task线程。

大概流程如下图：
![HashShuffleManager1](/img/hashshuffle1.png)

**shuffle write阶段**
```
1. 主要就是在一个stage结束计算之后，为了 下一个stage可以执行shuffle类的算子（比如reduceByKey），而将每个task处理的数据按key 进行“分类”。
2. 。所谓“分类”，就是对相同的key执行hash算法，从而将相同key都写入同一个
磁盘文件中，而每一个磁盘文件都只属于下游stage的一个task。在将数据写入磁盘之前，会先 将数据写入内存缓冲中，当内存缓冲填满之后，才会溢写到磁盘文件中去。
3. 那么每个执行shuffle write的task，要为下一个stage创建多少个磁盘文件呢？下一个stage的task有多少个，当前stage的每个task就要创建多少份磁盘文件。
4. 比如下一个stage总共有100个task，那么当前stage的每个task都要创建100份磁盘文件。如果当前stage有50个task，总共有10个Executor，每个Executor执行5个Task，那么每个Executor上总共就要创建 500个磁盘文件，所有Executor上会创建5000个磁盘文件。由此可见，未经优化的shuffle write操作所产生的磁盘文件的数量是极其惊人的。
```
