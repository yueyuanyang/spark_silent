####  Spark shuffle 机制之—— Sort-Based Shuffle 介绍

#### 1. 为什么需要Sort-Based Shuffle? 

1. Shuffle一般包含两个阶段任务： 
- 第一部分：产生Shuffle数据的阶段(Map阶段，额外补充，需要实现ShuffleManager中的getWriter来写数据(数据可以通过BlockManager写到Memory，Disk，Tachyon等，例如想非常快的Shuffle，此时可以考虑把数据写在内存中，但是内存不稳定，所以可以考虑增加副本。建议采用MEMONY_AND_DISK方式)； 
- 第二部分：使用Shuffle数据的阶段(Reduce阶段，额外补充，Shuffle读数据：需要实现ShuffleManager的getReader，Reader会向Driver去获取上一个Stage产生的Shuffle数据)。

2. Spark的Job会被划分成很多Stage: 

如果只要一个Stage，则这个Job就相当于只有一个Mapper段，当然不会产生Shuffle，适合于简单的ETL。如果不止一个Stage，则最后一个Stage就是最终的Reducer，最左侧的第一个Stage就仅仅是整个Job 的 Mapper，中间所有的任意一个Stage是其父Stage的Reducer且是其子Stage的 Mapper;

#### 2. Sort-Based Shuffle 

1. 为了让Spark在更大规模的集群上更高性能处理更大规模的数据，于是就引入了Sort-based Shuffle!从此以后(Spark1.1版本开始)，Spark可以胜任任何规模(包括PB级别及PB以上的级别)的大数据的处理，尤其是钨丝计划的引入和优化，Spark更快速的在更大规模的集群处理更海量的数据的能力推向了一个新的巅峰！ 

2. Spark1.6版本支持最少三种类型Shuffle：

```
val shortShuffleMgrNames = Map(
  "hash" -> "org.apache.spark.shuffle.hash.HashShuffleManager",
  "sort" -> "org.apache.spark.shuffle.sort.SortShuffleManager",
  "tungsten-sort" -> "org.apache.spark.shuffle.sort.SortShuffleManager")
val shuffleMgrName = conf.get("spark.shuffle.manager", "sort")
val shuffleMgrClass = shortShuffleMgrNames.getOrElse(shuffleMgrName.toLowerCase, shuffleMgrName)
val shuffleManager = instantiateClass[ShuffleManager](shuffleMgrClass)

```

实现ShuffleManager接口可以根据自己的业务实际需要最优化的使用自定义的Shuffle实现； 

3. Spark1.6默认采用的就是Sort-based Shuffle的方式：

```
val shuffleMgrName = conf.get("spark.shuffle.manager", "sort")
```

上述源码说明，你可以在Spark配置文件中配置Spark框架运行时要使用的具体的ShuffleManager的实现。可以在conf/spark-default.conf加入如下内容： 
spark.shuffle.manager SORT 配置Shuffle方式是SORT.

4. Sort-based Shuffle的工作方式如下：

**Shuffle的目的就是**：数据分类，然后数据聚集。 

![t12](https://github.com/yueyuanyang/spark_silent/blob/master/notes/img/t12.png)

1) 首先每个ShuffleMapTask不会为每个Reducer单独生成一个文件，相反，Sort-based Shuffle会把Mapper中每个ShuffleMapTask所有的输出数据Data只写到一个文件中。因为每个ShuffleMapTask中的数据会被分类，所以Sort-based Shuffle使用了index文件存储具体ShuffleMapTask输出数据在同一个Data文件中是如何分类的信息！

2) 基于Sort-base的Shuffle会在Mapper中的每一个ShuffleMapTask中产生两个文件：Data文件和Index文件，其中Data文件是存储当前Task的Shuffle输出的。而index文件中则存储了Data文件中的数据通过Partitioner的分类信息，此时下一个阶段的Stage中的Task就是根据这个Index文件获取自己所要抓取的上一个Stage中的ShuffleMapTask产生的数据的，Reducer就是根据index文件来获取属于自己的数据。 

涉及问题：Sorted-based Shuffle：会产生 2 x M(M代表了Mapper阶段中并行的Partition的总数量，其实就是ShuffleMapTask的总数量)个Shuffle临时文件。 

**Shuffle产生的临时文件的数量的变化一次为**： 

**Basic Hash Shuffle:** M x R; 
**Consalidate方式的Hash Shuffle:** C x R; 
**Sort-based Shuffle:**  2 x M; 

---

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

### 5.Shuffle 性能调优思考
Shuffle可能面临的问题，运行 Task 的时候才会产生 Shuffle (Shuffle 已经融化在 Spark 的算子中)

- 几千台或者是上万台的机器进行汇聚计算，数据量会非常大，网络传输会很大
- 数据如何分类其实就是 partition，即如何 Partition、Hash 、Sort 、计算
- 负载均衡 (数据倾斜）
- 网络传输效率，需要压缩或解压缩之间做出权衡，序列化 和 反序列化也是要考虑的问题

具体的 Task 进行计算的时候尽一切最大可能使得数据具备 Process Locality 的特性，退而求其次是增加数据分片，减少每个 Task 处理的数据量，基于Shuffle 和数据倾斜所导致的一系列问题，可以延伸出很多不同的调优点，比如说：

Mapper端的 Buffer 应该设置为多大呢？
- Reducer端的 Buffer 应该设置为多大呢？如果 Reducer 太少的话，这会限制了抓取多少数据
- 在数据传输的过程中是否有压缩以及该用什么方式去压缩，默应是用 snappy 的压缩方式。
- 网络传输失败重试的次数，每次重试之间间隔多少时间。

> Spark HashShuffle 源码鉴赏: https://www.cnblogs.com/jcchoiling/p/6431969.html





