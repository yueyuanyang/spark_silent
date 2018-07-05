## Apache Spark 作业性能调优(Part 1)

当你开始编写 Apache Spark 代码或者浏览公开的 API 的时候，你会遇到各种各样术语，比如 transformation，action，RDD 等等。 了解到这些是编写 Spark 代码的基础。 同样，当你任务开始失败或者你需要透过web界面去了解自己的应用为何如此费时的时候，你需要去了解一些新的名词： job, stage, task。对于这些新术语的理解有助于编写良好 Spark 代码。这里的良好主要指更快的 Spark 程序。对于 Spark 底层的执行模型的了解对于写出效率更高的 Spark 程序非常有帮助。

### Spark 是如何执行程序的

一个 Spark 应用包括一个 driver 进程和若干个分布在集群的各个节点上的 executor 进程。

driver 主要负责调度一些高层次的任务流（flow of work）。exectuor 负责执行这些任务，这些任务以 task 的形式存在， 同时存储用户设置需要caching的数据。 task 和所有的 executor 的生命周期为整个程序的运行过程（如果使用了dynamic resource allocation 时可能不是这样的）。如何调度这些进程是通过集群管理应用完成的（比如YARN，Mesos，Spark Standalone），但是任何一个 Spark 程序都会包含一个 driver 和多个 executor 进程。

![P1](https://github.com/yueyuanyang/spark_silent/blob/master/optimization/img/p1.png)


在执行层次结构的最上方是一系列 Job。调用一个Spark内部的 action 会产生一个 Spark job 来完成它。 为了确定这些job实际的内容，Spark 检查 RDD 的DAG再计算出执行 plan 。这个 plan 以最远端的 RDD 为起点（最远端指的是对外没有依赖的 RDD 或者 数据已经缓存下来的 RDD），产生结果 RDD 的 action 为结束 。

执行的 plan 由一系列 stage 组成，stage 是 job 的 transformation 的组合，stage 对应于一系列 task， task 指的对于不同的数据集执行的相同代码。每个 stage 包含不需要 shuffle 数据的 transformation 的序列。

什么决定数据是否需要 shuffle ？RDD 包含固定数目的 partition， 每个 partiton 包含若干的 record。对于那些通过 narrow tansformation（比如 map 和 filter）返回的 RDD，一个 partition 中的 record 只需要从父 RDD 对应的 partition 中的 record 计算得到。每个对象只依赖于父 RDD 的一个对象。有些操作（比如 coalesce）可能导致一个 task 处理多个输入 partition ，但是这种 transformation 仍然被认为是 narrow 的，因为用于计算的多个输入 record 始终是来自有限个数的 partition。

然而 Spark 也支持需要 wide 依赖的 transformation，比如 groupByKey，reduceByKey。在这种依赖中，计算得到一个 partition 中的数据需要从父 RDD 中的多个 partition 中读取数据。所有拥有相同 key 的元组最终会被聚合到同一个 partition 中，被同一个 stage 处理。为了完成这种操作， Spark需要对数据进行 shuffle，意味着数据需要在集群内传递，最终生成由新的 partition 集合组成的新的 stage。

举例，以下的代码中，只有一个 action 以及 从一个文本串下来的一系列 RDD， 这些代码就只有一个 stage，因为没有哪个操作需要从不同的 partition 里面读取数据。
```
sc.textFile("someFile.txt").
  map(mapFunc).
  flatMap(flatMapFunc).
  filter(filterFunc).
  count()
```

跟上面的代码不同，下面一段代码需要统计总共出现超过1000次的字母，

```
val tokenized = sc.textFile(args(0)).flatMap(_.split(' '))
val wordCounts = tokenized.map((_, 1)).reduceByKey(_ + _)
val filtered = wordCounts.filter(_._2 >= 1000)
val charCounts = filtered.flatMap(_._1.toCharArray).map((_, 1)).
  reduceByKey(_ + _)
charCounts.collect()
```
这段代码可以分成三个 stage。recudeByKey 操作是各 stage 之间的分界，因为计算 recudeByKey 的输出需要按照可以重新分配 partition。

这里还有一个更加复杂的 transfromation 图，包含一个有多路依赖的 join transformation。

![P2](https://github.com/yueyuanyang/spark_silent/blob/master/optimization/img/P2.png)

粉红色的框框展示了运行时使用的 stage 图。

![P3](https://github.com/yueyuanyang/spark_silent/blob/master/optimization/img/P3.png)

运行到每个 stage 的边界时，数据在父 stage 中按照 task 写到磁盘上，而在子 stage 中通过网络按照 task 去读取数据。这些操作会导致很重的网络以及磁盘的I/O，所以 stage 的边界是非常占资源的，在编写 Spark 程序的时候需要尽量避免的。父 stage 中 partition 个数与子 stage 的 partition 个数可能不同，所以那些产生 stage 边界的 transformation 常常需要接受一个 numPartition 的参数来觉得子 stage 中的数据将被切分为多少个 partition。

正如在调试 MapReduce 是选择 reducor 的个数是一项非常重要的参数，调整在 stage 边届时的 partition 个数经常可以很大程度上影响程序的执行效率。我们会在后面的章节中讨论如何调整这些值。

### 选择正确的 Operator

当需要使用 Spark 完成某项功能时，程序员需要从不同的 action 和 transformation 中选择不同的方案以获得相同的结果。但是不同的方案，最后执行的效率可能有云泥之别。回避常见的陷阱选择正确的方案可以使得最后的表现有巨大的不同。一些规则和深入的理解可以帮助你做出更好的选择。

选择 Operator 方案的主要目标是减少 shuffle 的次数以及被 shuffle 的文件的大小。因为 shuffle 是最耗资源的操作，所以有 shuffle 的数据都需要写到磁盘并且通过网络传递。repartition，join，cogroup，以及任何 \*By 或者 \*ByKey 的 transformation 都需要 shuffle 数据。不是所有这些 Operator 都是平等的，但是有些常见的性能陷阱是需要注意的。

- 当进行联合的规约操作时，避免使用 groupByKey。举个例子:rdd.groupByKey().mapValues(_ .sum) 与 rdd.reduceByKey(_ + _) 执行的结果是一样的，但是前者需要把全部的数据通过网络传递一遍，而后者只需要根据每个 key 局部的 partition 累积结果，在 shuffle 的之后把局部的累积值相加后得到结果。

- 当输入和输入的类型不一致时，避免使用 reduceByKey。举个例子，我们需要实现为每一个key查找所有不相同的 string。一个方法是利用 map 把每个元素的转换成一个 Set，再使用 reduceByKey 将这些 Set 合并起来
```
rdd.map(kv => (kv._1, new Set[String]() + kv._2))
    .reduceByKey(_ ++ _)
```
这段代码生成了无数的非必须的对象，因为每个需要为每个 record 新建一个 Set。这里使用 aggregateByKey 更加适合，因为这个操作是在 map 阶段做聚合。

```
val zero = new collection.mutable.Set[String]()
rdd.aggregateByKey(zero)(
    (set, v) => set += v,
    (set1, set2) => set1 ++= set2)
```

- 避免 flatMap-join-groupBy 的模式。当有两个已经按照key分组的数据集，你希望将两个数据集合并，并且保持分组，这种情况可以使用 cogroup。这样可以避免对group进行打包解包的开销。


### 什么时候不发生 Shuffle
当然了解在哪些 transformation 上不会发生 shuffle 也是非常重要的。当前一个 transformation 已经用相同的 patitioner 把数据分 patition 了，Spark知道如何避免 shuffle。参考一下代码：
```
rdd1 = someRdd.reduceByKey(...)
rdd2 = someOtherRdd.reduceByKey(...)
rdd3 = rdd1.join(rdd2)
```
因为没有 partitioner 传递给 reduceByKey，所以系统使用默认的 partitioner，所以 rdd1 和 rdd2 都会使用 hash 进行分 partition。代码中的两个 reduceByKey 会发生两次 shuffle 。如果 RDD 包含相同个数的 partition， join 的时候将不会发生额外的 shuffle。因为这里的 RDD 使用相同的 hash 方式进行 partition，所以全部 RDD 中同一个 partition 中的 key的集合都是相同的。因此，rdd3中一个 partiton 的输出只依赖rdd2和rdd1的同一个对应的 partition，所以第三次 shuffle 是不必要的。

举个例子说，当 someRdd 有4个 partition， someOtherRdd 有两个 partition，两个 reduceByKey 都使用3个 partiton，所有的 task 会按照如下的方式执行：

![p4](https://github.com/yueyuanyang/spark_silent/blob/master/optimization/img/p4.png)

如果 rdd1 和 rdd2 在 reduceByKey 时使用不同的 partitioner 或者使用相同的 partitioner 但是 partition 的个数不同的情况，那么只用一个 RDD (partiton 数更少的那个)需要重新 shuffle。

相同的 tansformation，相同的输入，不同的 partition 个数：

![p5](https://github.com/yueyuanyang/spark_silent/blob/master/optimization/img/p5.png)

当两个数据集需要 join 的时候，避免 shuffle 的一个方法是使用 broadcast variables。如果一个数据集小到能够塞进一个 executor 的内存中，那么它就可以在 driver 中写入到一个 hash table中，然后 broadcast 到所有的 executor 中。然后 map transformation 可以引用这个 hash table 作查询。

### 什么情况下 Shuffle 越多越好
尽可能减少 shuffle 的准则也有例外的场合。如果额外的 shuffle 能够增加并发那么这也能够提高性能。比如当你的数据保存在几个没有切分过的大文件中时，那么使用 InputFormat 产生分 partition 可能会导致每个 partiton 中聚集了大量的 record，如果 partition 不够，导致没有启动足够的并发。在这种情况下，我们需要在数据载入之后使用 repartiton （会导致shuffle)提高 partiton 的个数，这样能够充分使用集群的CPU。

另外一种例外情况是在使用 recude 或者 aggregate action 聚集数据到 driver 时，如果数据把很多 partititon 个数的数据，单进程执行的 driver merge 所有 partition 的输出时很容易成为计算的瓶颈。为了缓解 driver 的计算压力，可以使用 reduceByKey 或者 aggregateByKey 执行分布式的 aggregate 操作把数据分布到更少的 partition 上。每个 partition 中的数据并行的进行 merge，再把 merge 的结果发个 driver 以进行最后一轮 aggregation。查看 treeReduce 和 treeAggregate 查看如何这么使用的例子。

这个技巧在已经按照 Key 聚集的数据集上格外有效，比如当一个应用是需要统计一个语料库中每个单词出现的次数，并且把结果输出到一个map中。一个实现的方式是使用 aggregation，在每个 partition 中本地计算一个 map，然后在 driver 中把各个 partition 中计算的 map merge 起来。另一种方式是通过 aggregateByKey 把 merge 的操作分布到各个 partiton 中计算，然后在简单地通过 collectAsMap 把结果输出到 driver 中。

### 二次排序
还有一个重要的技能是了解接口 repartitionAndSortWithinPartitions transformation。这是一个听起来很晦涩的 transformation，但是却能涵盖各种奇怪情况下的排序，这个 transformation 把排序推迟到 shuffle 操作中，这使大量的数据有效的输出，排序操作可以和其他操作合并。

举例说，Apache Hive on Spark 在join的实现中，使用了这个 transformation 。而且这个操作在 secondary sort 模式中扮演着至关重要的角色。secondary sort 模式是指用户期望数据按照 key 分组，并且希望按照特定的顺序遍历 value。使用 repartitionAndSortWithinPartitions 再加上一部分用户的额外的工作可以实现 secondary sort。

### 结论
现在你应该对完成一个高效的 Spark 程序所需的所有基本要素有了很好的了解。在 Part II 中将详细介绍资源调用、并发以及数据结构相关的调试。
