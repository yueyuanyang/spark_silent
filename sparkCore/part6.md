## Spark算子：RDD基本转换操作(3) – randomSplit、glom、union、intersection、subtract

### randomSplit

def randomSplit(weights: Array[Double], seed: Long = Utils.random.nextLong): Array[RDD[T]]

该函数根据weights权重，将一个RDD切分成多个RDD。

该权重参数为一个Double数组

第二个参数为random的种子，基本可忽略。

```
scala> var rdd = sc.makeRDD(1 to 10,10)
rdd: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[16] at makeRDD at :21
 
scala> rdd.collect
res6: Array[Int] = Array(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)  
 
scala> var splitRDD = rdd.randomSplit(Array(1.0,2.0,3.0,4.0))
splitRDD: Array[org.apache.spark.rdd.RDD[Int]] = Array(MapPartitionsRDD[17] at randomSplit at :23, 
MapPartitionsRDD[18] at randomSplit at :23, 
MapPartitionsRDD[19] at randomSplit at :23, 
MapPartitionsRDD[20] at randomSplit at :23)
 
//这里注意：randomSplit的结果是一个RDD数组
scala> splitRDD.size
res8: Int = 4
//由于randomSplit的第一个参数weights中传入的值有4个，因此，就会切分成4个RDD,
//把原来的rdd按照权重1.0,2.0,3.0,4.0，随机划分到这4个RDD中，权重高的RDD，划分到//的几率就大一些。
//注意，权重的总和加起来为1，否则会不正常
 
scala> splitRDD(0).collect
res10: Array[Int] = Array(1, 4)
 
scala> splitRDD(1).collect
res11: Array[Int] = Array(3)                                                    
 
scala> splitRDD(2).collect
res12: Array[Int] = Array(5, 9)
 
scala> splitRDD(3).collect
res13: Array[Int] = Array(2, 6, 7, 8, 10)
```

### glom

def glom(): RDD[Array[T]]

该函数是将RDD中每一个分区中类型为T的元素转换成Array[T]，这样每一个分区就只有一个数组元素。

```
scala> var rdd = sc.makeRDD(1 to 10,3)
rdd: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[38] at makeRDD at :21
scala> rdd.partitions.size
res33: Int = 3  //该RDD有3个分区
scala> rdd.glom().collect
res35: Array[Array[Int]] = Array(Array(1, 2, 3), Array(4, 5, 6), Array(7, 8, 9, 10))
//glom将每个分区中的元素放到一个数组中，这样，结果就变成了3个数组
```

### union

def union(other: RDD[T]): RDD[T]

该函数比较简单，就是将两个RDD进行合并，不去重。

 ```
 scala> var rdd1 = sc.makeRDD(1 to 2,1)
rdd1: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[45] at makeRDD at :21
 
scala> rdd1.collect
res42: Array[Int] = Array(1, 2)
 
scala> var rdd2 = sc.makeRDD(2 to 3,1)
rdd2: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[46] at makeRDD at :21
 
scala> rdd2.collect
res43: Array[Int] = Array(2, 3)
 
scala> rdd1.union(rdd2).collect
res44: Array[Int] = Array(1, 2, 2, 3)
 ```

### intersection

def intersection(other: RDD[T]): RDD[T]
def intersection(other: RDD[T], numPartitions: Int): RDD[T]
def intersection(other: RDD[T], partitioner: Partitioner)(implicit ord: Ordering[T] = null): RDD[T]

该函数返回两个RDD的交集，并且去重。
- 参数numPartitions指定返回的RDD的分区数。
- 参数partitioner用于指定分区函数

```
scala> var rdd1 = sc.makeRDD(1 to 2,1)
rdd1: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[45] at makeRDD at :21
 
scala> rdd1.collect
res42: Array[Int] = Array(1, 2)
 
scala> var rdd2 = sc.makeRDD(2 to 3,1)
rdd2: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[46] at makeRDD at :21
 
scala> rdd2.collect
res43: Array[Int] = Array(2, 3)
 
scala> rdd1.intersection(rdd2).collect
res45: Array[Int] = Array(2)
 
scala> var rdd3 = rdd1.intersection(rdd2)
rdd3: org.apache.spark.rdd.RDD[Int] = MapPartitionsRDD[59] at intersection at :25
 
scala> rdd3.partitions.size
res46: Int = 1
 
scala> var rdd3 = rdd1.intersection(rdd2,2)
rdd3: org.apache.spark.rdd.RDD[Int] = MapPartitionsRDD[65] at intersection at :25
 
scala> rdd3.partitions.size
res47: Int = 2
```

### subtract

def subtract(other: RDD[T]): RDD[T]
def subtract(other: RDD[T], numPartitions: Int): RDD[T]
def subtract(other: RDD[T], partitioner: Partitioner)(implicit ord: Ordering[T] = null): RDD[T]

该函数类似于intersection，但返回在RDD中出现，并且不在otherRDD中出现的元素，不去重。
- 参数含义同intersection

```
scala> var rdd1 = sc.makeRDD(Seq(1,2,2,3))
rdd1: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[66] at makeRDD at :21
 
scala> rdd1.collect
res48: Array[Int] = Array(1, 2, 2, 3)
 
scala> var rdd2 = sc.makeRDD(3 to 4)
rdd2: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[67] at makeRDD at :21
 
scala> rdd2.collect
res49: Array[Int] = Array(3, 4)
 
scala> rdd1.subtract(rdd2).collect
res50: Array[Int] = Array(1, 2, 2)
```




