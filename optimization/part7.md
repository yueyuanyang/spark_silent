## Apache Spark 作业性能调优(Part 1)

当你开始编写 Apache Spark 代码或者浏览公开的 API 的时候，你会遇到各种各样术语，比如 transformation，action，RDD 等等。 了解到这些是编写 Spark 代码的基础。 同样，当你任务开始失败或者你需要透过web界面去了解自己的应用为何如此费时的时候，你需要去了解一些新的名词： job, stage, task。对于这些新术语的理解有助于编写良好 Spark 代码。这里的良好主要指更快的 Spark 程序。对于 Spark 底层的执行模型的了解对于写出效率更高的 Spark 程序非常有帮助。

### Spark 是如何执行程序的

一个 Spark 应用包括一个 driver 进程和若干个分布在集群的各个节点上的 executor 进程。

driver 主要负责调度一些高层次的任务流（flow of work）。exectuor 负责执行这些任务，这些任务以 task 的形式存在， 同时存储用户设置需要caching的数据。 task 和所有的 executor 的生命周期为整个程序的运行过程（如果使用了dynamic resource allocation 时可能不是这样的）。如何调度这些进程是通过集群管理应用完成的（比如YARN，Mesos，Spark Standalone），但是任何一个 Spark 程序都会包含一个 driver 和多个 executor 进程。

![P1]()


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

![P2]()

粉红色的框框展示了运行时使用的 stage 图。

![P3]()

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
