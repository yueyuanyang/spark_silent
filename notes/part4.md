### Spark shuffle 机制之—— HashShuffle介绍

#### 1.Spark 中的 HashShuffle介绍

Spark HashShuffle 是它以前的版本，现在1.6x 版本默应是 Sort-Based Shuffle，那为什么要讲 HashShuffle 呢，因为有分布式就一定会有 Shuffle，而且 HashShuffle 是 Spark以前的版本，亦即是 Sort-Based Shuffle 的前身，因为有 HashShuffle 的不足，才会有后续的 Sorted-Based Shuffle，以及现在的 Tungsten-Sort Shuffle，所以我们有必要去了解它

#### 2.原始的HashShuffle 机制

基于 Mapper 和 Reducer 理解的基础上，当 Reducer 去抓取数据时，它的 Key 到底是怎么分配的，核心思考点是：作为上游数据是怎么去分配给下游数据的。在这张图中你可以看到有4个 Task 在2个 Executors 上面，它们是并行运行的，Hash 本身有一套 Hash算法，可以把数据的 Key 进行重新分类，每个 Task 对数据进行分类然后把它们不同类别的数据先写到本地磁盘，然后再经过网络传输 Shuffle，把数据传到下一个 Stage 进行汇聚。

下图有3个 Reducer，从 Task 开始那边各自把自己进行 Hash 计算，分类出3个不同的类别，每个 Task 都分成3种类别的数据，刚刚提过因为分布式的关系，我们想把不同的数据汇聚然后计算出最终的结果，所以下游的 Reducer 会在每个 Task 中把属于自己类别的数据收集过来，汇聚成一个同类别的大集合，抓过来的时候会首先放在内存中，但内存可能放不下，也有可能放在本地 (这也是一个调优点。可以参考上一章讲过的一些调优参数)，

**每1个 Task 输出3份本地文件，这里有4个 Mapper Tasks，所以总共输出了4个 Tasks x 3个分类文件 = 12个本地小文件**。

![t10](https://github.com/yueyuanyang/spark_silent/blob/master/notes/img/t10.png)

HashShuffle 也有它的弱点：
- Shuffle前在磁盘上会产生海量的小文件，此时会产生大量耗时低效的 IO 操作 (因為产生过多的小文件）
- 内存不够用，由于内存中需要保存海量文件操作句柄和临时信息，如果数据处理的规模比较庞大的话，内存不可承受，会出现 OOM 等问题。

#### 3.优化后的 HashShuffle 机制(Consoldiated Hash-Shuffle)

在刚才 HashShuffle 的基础上思考该如何进行优化，这是优化后的实现：

![t11](https://github.com/yueyuanyang/spark_silent/blob/master/notes/img/t11.png)

这里还是有4个Tasks，数据类别还是分成3种类型，因为Hash算法会根据你的 Key 进行分类，在同一个进程中，无论是有多少过Task，都会把同样的Key放在同一个Buffer里，然后把Buffer中的数据写入以Core数量为单位的本地文件中，(一个Core只有一种类型的Key的数据)，**每1个Task所在的进程中，分别写入共同进程中的3份本地文件，这里有4个Mapper Tasks，所以总共输出是 2个Cores x 3个分类文件 = 6个本地小文件**。Consoldiated Hash-Shuffle的优化有一个很大的好处就是假设现在有200个Mapper Tasks在同一个进程中，也只会产生3个本地小文件； 如果用原始的 Hash-Based Shuffle 的话，200个Mapper Tasks 会各自产生3个本地小文件，在一个进程已经产生了600个本地小文件。3个对比600已经是一个很大的差异了。

这个优化后的 HashShuffle 叫 ConsolidatedShuffle，在实际生产环境下可以调以下参数：

> spark.shuffle.consolidateFiles=true

**Consolidated HashShuffle 也有它的弱点**： 
- 如果 Reducer 端的并行任务或者是数据分片过多的话则 Core * Reducer Task 依旧过大，也会产生很多小文件。




