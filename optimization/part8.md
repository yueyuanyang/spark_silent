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

