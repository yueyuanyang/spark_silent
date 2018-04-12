### Spark RDD的理解

#### 1. 定义

RDD(Resilient Distributed Datasets,弹性分布式数据集)是一个分区的只读记录的集合。

RDD只能通过在稳定的存储器或其他RDD的数据上的确定性操作来创建。我们把这些操作称作变换以区别其他类型的操作。例如 map,filter和join。

RDD在任何时候都不需要被”物化”(进行实际的变换并最终写入稳定的存储器上)。实际上，一个RDD有足够的信息描述着其如何从其他稳定的存储器上的数据生成。它有一个强大的特性：从本质上说，若RDD失效且不能重建，程序将不能引用该RDD。

而用户可以控制RDD的其他两个方面：持久化和分区。用户可以选择重用哪个RDD，并为其制定存储策略(比如，内存存储)。也可以让RDD中的数据根据记录的key分布到集群的多个机器。 这对位置优化来说是有用的，比如可用来保证两个要jion的数据集都使用了相同的哈希分区方式。

#### 2. RDD的五大特性

- A list of partitions

RDD是一个由多个partition（某个节点里的某一片连续的数据）组成的的list；将数据加载为RDD时，一般会遵循数据的本地性（一般一个hdfs里的block会加载为一个partition）。

- A function for computing each split

RDD的每个partition上面都会有function，也就是函数应用，其作用是实现RDD之间partition的转换。

- A list of dependencies on other RDDs

RDD会记录它的依赖 ，为了容错（重算，cache，checkpoint），也就是说在内存中的RDD操作时出错或丢失会进行重算。

- Optionally,a Partitioner for Key-value RDDs

  可选项，如果RDD里面存的数据是key-value形式，则可以传递一个自定义的Partitioner进行重新分区，例如这里自定义的Partitioner是基于key进行分区，那则会将不同RDD里面的相同key的数据放到同一个partition里面
  
- Optionally, a list of preferred locations to compute each split on

最优的位置去计算，也就是数据的本地性。      

