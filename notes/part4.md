### Spark shuffle 机制之—— HashShuffle介绍

#### Spark 中的 HashShuffle介绍

Spark HashShuffle 是它以前的版本，现在1.6x 版本默应是 Sort-Based Shuffle，那为什么要讲 HashShuffle 呢，因为有分布式就一定会有 Shuffle，而且 HashShuffle 是 Spark以前的版本，亦即是 Sort-Based Shuffle 的前身，因为有 HashShuffle 的不足，才会有后续的 Sorted-Based Shuffle，以及现在的 Tungsten-Sort Shuffle，所以我们有必要去了解它

#### 原始的HashShuffle 机制

基于 Mapper 和 Reducer 理解的基础上，当 Reducer 去抓取数据时，它的 Key 到底是怎么分配的，核心思考点是：作为上游数据是怎么去分配给下游数据的。在这张图中你可以看到有4个 Task 在2个 Executors 上面，它们是并行运行的，Hash 本身有一套 Hash算法，可以把数据的 Key 进行重新分类，每个 Task 对数据进行分类然后把它们不同类别的数据先写到本地磁盘，然后再经过网络传输 Shuffle，把数据传到下一个 Stage 进行汇聚。

下图有3个 Reducer，从 Task 开始那边各自把自己进行 Hash 计算，分类出3个不同的类别，每个 Task 都分成3种类别的数据，刚刚提过因为分布式的关系，我们想把不同的数据汇聚然后计算出最终的结果，所以下游的 Reducer 会在每个 Task 中把属于自己类别的数据收集过来，汇聚成一个同类别的大集合，抓过来的时候会首先放在内存中，但内存可能放不下，也有可能放在本地 (这也是一个调优点。可以参考上一章讲过的一些调优参数)，

**每1个 Task 输出3份本地文件，这里有4个 Mapper Tasks，所以总共输出了4个 Tasks x 3个分类文件 = 12个本地小文件**。
