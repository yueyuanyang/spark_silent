## Spark算子：RDD基本转换操作(6)–zip、zipPartitions、zipWithIndex、zipWithUniqueId

### zip

def zip[U](other: RDD[U])(implicit arg0: ClassTag[U]): RDD[(T, U)]

zip函数用于将两个RDD组合成Key/Value形式的RDD,这里默认两个RDD的partition数量以及元素数量都相同，否则会抛出异常。

```
scala> var rdd1 = sc.makeRDD(1 to 10,2)
rdd1: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[0] at makeRDD at :21
 
scala> var rdd1 = sc.makeRDD(1 to 5,2)
rdd1: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[1] at makeRDD at :21
 
scala> var rdd2 = sc.makeRDD(Seq("A","B","C","D","E"),2)
rdd2: org.apache.spark.rdd.RDD[String] = ParallelCollectionRDD[2] at makeRDD at :21
 
scala> rdd1.zip(rdd2).collect
res0: Array[(Int, String)] = Array((1,A), (2,B), (3,C), (4,D), (5,E))           
 
scala> rdd2.zip(rdd1).collect
res1: Array[(String, Int)] = Array((A,1), (B,2), (C,3), (D,4), (E,5))
 
scala> var rdd3 = sc.makeRDD(Seq("A","B","C","D","E"),3)
rdd3: org.apache.spark.rdd.RDD[String] = ParallelCollectionRDD[5] at makeRDD at :21
 
scala> rdd1.zip(rdd3).collect
java.lang.IllegalArgumentException: Can't zip RDDs with unequal numbers of partitions
//如果两个RDD分区数不同，则抛出异常
 
```

### zipPartitions

zipPartitions函数将多个RDD按照partition组合成为新的RDD，该函数需要组合的RDD具有相同的分区数，但对于每个分区内的元素数量没有要求。

该函数有好几种实现，可分为三类：

- 参数是一个RDD
def zipPartitions[B, V](rdd2: RDD[B])(f: (Iterator[T], Iterator[B]) => Iterator[V])(implicit arg0: ClassTag[B], arg1: ClassTag[V]): RDD[V]

def zipPartitions[B, V](rdd2: RDD[B], preservesPartitioning: Boolean)(f: (Iterator[T], Iterator[B]) => Iterator[V])(implicit arg0: ClassTag[B], arg1: ClassTag[V]): RDD[V]

这两个区别就是参数preservesPartitioning，是否保留父RDD的partitioner分区信息

映射方法f参数为两个RDD的迭代器。

```
scala> var rdd1 = sc.makeRDD(1 to 5,2)
rdd1: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[22] at makeRDD at :21
 
scala> var rdd2 = sc.makeRDD(Seq("A","B","C","D","E"),2)
rdd2: org.apache.spark.rdd.RDD[String] = ParallelCollectionRDD[23] at makeRDD at :21
 
//rdd1两个分区中元素分布：
scala> rdd1.mapPartitionsWithIndex{
     |         (x,iter) => {
     |           var result = List[String]()
     |             while(iter.hasNext){
     |               result ::= ("part_" + x + "|" + iter.next())
     |             }
     |             result.iterator
     |            
     |         }
     |       }.collect
res17: Array[String] = Array(part_0|2, part_0|1, part_1|5, part_1|4, part_1|3)
 
//rdd2两个分区中元素分布
scala> rdd2.mapPartitionsWithIndex{
     |         (x,iter) => {
     |           var result = List[String]()
     |             while(iter.hasNext){
     |               result ::= ("part_" + x + "|" + iter.next())
     |             }
     |             result.iterator
     |            
     |         }
     |       }.collect
res18: Array[String] = Array(part_0|B, part_0|A, part_1|E, part_1|D, part_1|C)
 
//rdd1和rdd2做zipPartition
scala> rdd1.zipPartitions(rdd2){
     |       (rdd1Iter,rdd2Iter) => {
     |         var result = List[String]()
     |         while(rdd1Iter.hasNext && rdd2Iter.hasNext) {
     |           result::=(rdd1Iter.next() + "_" + rdd2Iter.next())
     |         }
     |         result.iterator
     |       }
     |     }.collect
res19: Array[String] = Array(2_B, 1_A, 5_E, 4_D, 3_C)

```

- 参数是两个RDD
def zipPartitions[B, C, V](rdd2: RDD[B], rdd3: RDD[C])(f: (Iterator[T], Iterator[B], Iterator[C]) => Iterator[V])(implicit arg0: ClassTag[B], arg1: ClassTag[C], arg2: ClassTag[V]): RDD[V]

def zipPartitions[B, C, V](rdd2: RDD[B], rdd3: RDD[C], preservesPartitioning: Boolean)(f: (Iterator[T], Iterator[B], Iterator[C]) => Iterator[V])(implicit arg0: ClassTag[B], arg1: ClassTag[C], arg2: ClassTag[V]): RDD[V]

用法同上面，只不过该函数参数为两个RDD，映射方法f输入参数为两个RDD的迭代器。

```
scala> var rdd1 = sc.makeRDD(1 to 5,2)
rdd1: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[27] at makeRDD at :21
 
scala> var rdd2 = sc.makeRDD(Seq("A","B","C","D","E"),2)
rdd2: org.apache.spark.rdd.RDD[String] = ParallelCollectionRDD[28] at makeRDD at :21
 
scala> var rdd3 = sc.makeRDD(Seq("a","b","c","d","e"),2)
rdd3: org.apache.spark.rdd.RDD[String] = ParallelCollectionRDD[29] at makeRDD at :21
 
//rdd3中个分区元素分布
scala> rdd3.mapPartitionsWithIndex{
     |         (x,iter) => {
     |           var result = List[String]()
     |             while(iter.hasNext){
     |               result ::= ("part_" + x + "|" + iter.next())
     |             }
     |             result.iterator
     |            
     |         }
     |       }.collect
res21: Array[String] = Array(part_0|b, part_0|a, part_1|e, part_1|d, part_1|c)
 
//三个RDD做zipPartitions
scala> var rdd4 = rdd1.zipPartitions(rdd2,rdd3){
     |       (rdd1Iter,rdd2Iter,rdd3Iter) => {
     |         var result = List[String]()
     |         while(rdd1Iter.hasNext && rdd2Iter.hasNext && rdd3Iter.hasNext) {
     |           result::=(rdd1Iter.next() + "_" + rdd2Iter.next() + "_" + rdd3Iter.next())
     |         }
     |         result.iterator
     |       }
     |     }
rdd4: org.apache.spark.rdd.RDD[String] = ZippedPartitionsRDD3[33] at zipPartitions at :27
 
scala> rdd4.collect
res23: Array[String] = Array(2_B_b, 1_A_a, 5_E_e, 4_D_d, 3_C_c)
 
```
- 参数是三个RDD
def zipPartitions[B, C, D, V](rdd2: RDD[B], rdd3: RDD[C], rdd4: RDD[D])(f: (Iterator[T], Iterator[B], Iterator[C], Iterator[D]) => Iterator[V])(implicit arg0: ClassTag[B], arg1: ClassTag[C], arg2: ClassTag[D], arg3: ClassTag[V]): RDD[V]

def zipPartitions[B, C, D, V](rdd2: RDD[B], rdd3: RDD[C], rdd4: RDD[D], preservesPartitioning: Boolean)(f: (Iterator[T], Iterator[B], Iterator[C], Iterator[D]) => Iterator[V])(implicit arg0: ClassTag[B], arg1: ClassTag[C], arg2: ClassTag[D], arg3: ClassTag[V]): RDD[V]

用法同上面，只不过这里又多了个一个RDD而已。



