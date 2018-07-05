## Apache Spark 作业性能调优(Part 2)

在Apache Spark 作业性能调优(Part 1)文章中介绍了 Spark 的一些调优方法。作者将尽量覆盖到影响 Spark 程序性能的方方面面，你们将会了解到资源调优，或者如何配置 Spark 以压榨出集群每一分资源。然后我们将讲述调试并发度，这是job性能中最难也是最重要的参数。最后，你将了解到数据本身的表达形式，Spark 读取在磁盘的上的形式（主要是Apache Avro和 Apache Parquet)以及当数据需要缓存或者移动的时候内存中的数据形式。

### 调试资源分配
Spark 的用户邮件邮件列表中经常会出现 “我有一个500个节点的集群，为什么但是我的应用一次只有两个 task 在执行”，鉴于 Spark 控制资源使用的参数的数量，这些问题不应该出现。但是在本章中，你将学会压榨出你集群的每一分资源。推荐的配置将根据不同的集群管理系统（ YARN、Mesos、Spark Standalone）而有所不同，我们将主要集中在 YARN 上，因为这个 Cloudera 推荐的方式。

我们先看一下在 YARN　上运行　Spark 的一些背景。查看之前的博文：点击这里查看

Spark（以及YARN） 需要关心的两项主要的资源是 CPU 和 内存， 磁盘 和 IO 当然也影响着 Spark 的性能，但是不管是 Spark 还是 Yarn 目前都没法对他们做实时有效的管理。

在一个 Spark 应用中，每个 Spark executor 拥有固定个数的 core 以及固定大小的堆大小。core 的个数可以在执行 spark-submit 或者 pyspark 或者 spark-shell 时，通过参数 --executor-cores 指定，或者在 spark-defaults.conf 配置文件或者 SparkConf 对象中设置 spark.executor.cores 参数。同样地，堆的大小可以通过 --executor-memory 参数或者 spark.executor.memory 配置项。core 配置项控制一个 executor 中task的并发数。 --executor-cores 5 意味着每个 executor 中最多同时可以有5个 task 运行。memory 参数影响 Spark 可以缓存的数据的大小，也就是在 group aggregate 以及 join 操作时 shuffle 的数据结构的最大值。

--num-executors 命令行参数或者spark.executor.instances 配置项控制需要的 executor 个数。从 CDH 5.4/Spark 1.3 开始，你可以避免使用这个参数，只要你通过设置 spark.dynamicAllocation.enabled 参数打开 动态分配 。动态分配可以使的 Spark 的应用在有后续积压的在等待的 task 时请求 executor，并且在空闲时释放这些 executor。

同时 Spark 需求的资源如何跟 YARN 中可用的资源配合也是需要着重考虑的，YARN 相关的参数有：

- yarn.nodemanager.resource.memory-mb 控制在每个节点上 container 能够使用的最大内存；
- yarn.nodemanager.resource.cpu-vcores 控制在每个节点上 container 能够使用的最大core个数；请求5个 core 会生成向 YARN 要5个虚拟core的请求。从 YARN　请求内存相对比较复杂因为以下的一些原因：
- --executor-memory/spark.executor.memory 控制 executor 的堆的大小，但是 JVM 本身也会占用一定的堆空间，比如内部的 String 或者直接 byte buffer，executor memory 的 spark.yarn.executor.memoryOverhead 属性决定向 YARN 请求的每个 executor 的内存大小，默认值为max(384, 0.7 * spark.executor.memory);
- YARN 可能会比请求的内存高一点，YARN 的 yarn.scheduler.minimum-allocation-mb 和 yarn.scheduler.increment-allocation-mb 属性控制请求的最小值和增加量。
下面展示的是 Spark on YARN 内存结构

![p7](https://github.com/yueyuanyang/spark_silent/blob/master/optimization/img/p7.png)

- 如果这些还不够决定Spark executor 个数，还有一些概念还需要考虑的：
- 应用的master，是一个非 executor 的容器，它拥有特殊的从 YARN 请求资源的能力，它自己本身所占的资源也需要被计算在内。在 yarn-client 模式下，它默认请求 1024MB 和 1个core。在 yarn-cluster 模式中，应用的 master 运行 driver，所以使用参数 --driver-memory 和 --driver-cores 配置它的资源常常很有用。
- 在 executor 执行的时候配置过大的 memory 经常会导致过长的GC延时，64G是推荐的一个 executor 内存大小的上限。
- 我们注意到 HDFS client 在大量并发线程是时性能问题。大概的估计是每个 executor 中最多5个并行的 task 就可以占满的写入带宽。
- 在运行微型 executor 时（比如只有一个core而且只有够执行一个task的内存）扔掉在一个JVM上同时运行多个task的好处。比如 broadcast 变量需要为每个 executor 复制一遍，这么多小executor会导致更多的数据拷贝。

为了让以上的这些更加具体一点，这里有一个实际使用过的配置的例子，可以完全用满整个集群的资源。假设一个集群有6个节点有NodeManager在上面运行，每个节点有16个core以及64GB的内存。那么 NodeManager的容量：yarn.nodemanager.resource.memory-mb 和 yarn.nodemanager.resource.cpu-vcores 可以设为 63 * 1024 = 64512 （MB） 和 15。我们避免使用 100% 的 YARN container 资源因为还要为 OS 和 hadoop 的 Daemon 留一部分资源。在上面的场景中，我们预留了1个core和1G的内存给这些进程。Cloudera Manager 会自动计算并且配置。

所以看起来我们最先想到的配置会是这样的：
```
-num-executors 6 --executor-cores 15 --executor-memory 63G
```

但是这个配置可能无法达到我们的需求，因为：
- 63GB+ 的 executor memory 塞不进只有 63GB 容量的 NodeManager；
- 应用的 master 也需要占用一个core，意味着在某个节点上，没有15个core给 executor 使用；
- 15个core会影响 HDFS IO的吞吐量。配置成 --num-executors 17 --executor-cores 5 --executor-memory 19G 可能会效果更好，因为：
- 这个配置会在每个节点上生成3个 executor，除了应用的master运行的机器，这台机器上只会运行2个 executor
- --executor-memory 被分成3份（63G/每个节点3个executor）=21。 21 * （1 - 0.07） ~ 19。

调试并发
我们知道 Spark 是一套数据并行处理的引擎。但是 Spark 并不是神奇得能够将所有计算并行化，它没办法从所有的并行化方案中找出最优的那个。每个 Spark stage 中包含若干个 task，每个 task 串行地处理数据。在调试 Spark 的job时，task 的个数可能是决定程序性能的最重要的参数。

那么这个数字是由什么决定的呢？在之前的博文中介绍了 Spark 如何将 RDD 转换成一组 stage。task 的个数与 stage 中上一个 RDD 的 partition 个数相同。而一个 RDD 的 partition 个数与被它依赖的 RDD 的 partition 个数相同，除了以下的情况： coalesce transformation 可以创建一个具有更少 partition 个数的 RDD，union transformation 产出的 RDD 的 partition 个数是它父 RDD 的 partition 个数之和， cartesian 返回的 RDD 的 partition 个数是它们的积。

如果一个 RDD 没有父 RDD 呢？ 由 textFile 或者 hadoopFile 生成的 RDD 的 partition 个数由它们底层使用的 MapReduce InputFormat 决定的。一般情况下，每读到的一个 HDFS block 会生成一个 partition。通过 parallelize 接口生成的 RDD 的 partition 个数由用户指定，如果用户没有指定则由参数 spark.default.parallelism 决定。

要想知道 partition 的个数，可以通过接口 rdd.partitions().size() 获得。

这里最需要关心的问题在于 task 的个数太小。如果运行时 task 的个数比实际可用的 slot 还少，那么程序解没法使用到所有的 CPU 资源。

过少的 task 个数可能会导致在一些聚集操作时， 每个 task 的内存压力会很大。任何 join，cogroup，\*ByKey 操作都会在内存生成一个 hash-map或者 buffer 用于分组或者排序。join， cogroup ，groupByKey 会在 shuffle 时在 fetching 端使用这些数据结构， reduceByKey ，aggregateByKey 会在 shuffle 时在两端都会使用这些数据结构。

当需要进行这个聚集操作的 record 不能完全轻易塞进内存中时，一些问题会暴露出来。首先，在内存 hold 大量这些数据结构的 record 会增加 GC的压力，可能会导致流程停顿下来。其次，如果数据不能完全载入内存，Spark 会将这些数据写到磁盘，这会引起磁盘 IO和排序。在 Cloudera 的用户中，这可能是导致 Spark Job 慢的首要原因。

那么如何增加你的 partition 的个数呢？如果你的问题 stage 是从 Hadoop 读取数据，你可以做以下的选项：

- 使用 repartition 选项，会引发 shuffle；
- 配置 InputFormat 用户将文件分得更小；
- 写入 HDFS 文件时使用更小的block。

如果问题 stage 从其他 stage 中获得输入，引发 stage 边界的操作会接受一个 numPartitions 的参数，比如
```
val rdd2 = rdd1.reduceByKey(_ + _, numPartitions = X)
```

X 应该取什么值？最直接的方法就是做实验。不停的将 partition 的个数从上次实验的 partition 个数乘以1.5，直到性能不再提升为止。

同时也有一些原则用于计算 X，但是也不是非常的有效是因为有些参数是很难计算的。这里写到不是因为它们很实用，而是可以帮助理解。这里主要的目标是启动足够的 task 可以使得每个 task 接受的数据能够都塞进它所分配到的内存中。

每个 task 可用的内存通过这个公式计算：spark.executor.memory * spark.shuffle.memoryFraction * spark.shuffle.safetyFraction)/spark.executor.cores 。 memoryFraction 和 safetyFractio 默认值分别 0.2 和 0.8.

在内存中所有 shuffle 数据的大小很难确定。最可行的是找出一个 stage 运行的 Shuffle Spill（memory） 和 Shuffle Spill(Disk) 之间的比例。在用所有shuffle 写乘以这个比例。但是如果这个 stage 是 reduce 时，可能会有点复杂：

![p8](https://github.com/yueyuanyang/spark_silent/blob/master/optimization/img/p8.png)

在往上增加一点因为大多数情况下 partition 的个数会比较多。

试试在，在有所疑虑的时候，使用更多的 task 数（也就是 partition 数）都会效果更好，这与 MapRecuce 中建议 task 数目选择尽量保守的建议相反。这个因为 MapReduce 在启动 task 时相比需要更大的代价。

### 压缩你的数据结构

Spark 的数据流由一组 record 构成。一个 record 有两种表达形式:一种是反序列化的 Java 对象另外一种是序列化的二进制形式。通常情况下，Spark 对内存中的 record 使用反序列化之后的形式，对要存到磁盘上或者需要通过网络传输的 record 使用序列化之后的形式。也有计划在内存中存储序列化之后的 record。

spark.serializer 控制这两种形式之间的转换的方式。Kryo serializer，org.apache.spark.serializer.KryoSerializer 是推荐的选择。但不幸的是它不是默认的配置，因为 KryoSerializer 在早期的 Spark 版本中不稳定，而 Spark 不想打破版本的兼容性，所以没有把 KryoSerializer 作为默认配置，但是 KryoSerializer 应该在任何情况下都是第一的选择。

你的 record 在这两种形式切换的频率对于 Spark 应用的运行效率具有很大的影响。去检查一下到处传递数据的类型，看看能否挤出一点水分是非常值得一试的。

过多的反序列化之后的 record 可能会导致数据到处到磁盘上更加频繁，也使得能够 Cache 在内存中的 record 个数减少。点击这里查看如何压缩这些数据。

过多的序列化之后的 record 导致更多的 磁盘和网络 IO，同样的也会使得能够 Cache 在内存中的 record 个数减少，这里主要的解决方案是把所有的用户自定义的 class 都通过 SparkConf#registerKryoClasses 的API定义和传递的。

### 数据格式

任何时候你都可以决定你的数据如何保持在磁盘上，使用可扩展的二进制格式比如：Avro，Parquet，Thrift或者Protobuf，从中选择一种。当人们在谈论在Hadoop上使用Avro，Thrift或者Protobuf时，都是认为每个 record 保持成一个 Avro/Thrift/Protobuf 结构保存成 sequence file。而不是JSON。

每次当时试图使用JSON存储大量数据时，还是先放弃吧…
