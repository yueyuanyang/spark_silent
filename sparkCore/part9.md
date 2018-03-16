## Spark算子：RDD创建操作 - parallelize、 makeRDD

### parallelize

def parallelize[T](seq: Seq[T], numSlices: Int = defaultParallelism)(implicit arg0: ClassTag[T]): RDD[T]

从一个Seq集合创建RDD。

- 参数1：Seq集合，必须。

- 参数2：分区数，默认为该Application分配到的资源的CPU核数

```
scala> var rdd = sc.parallelize(1 to 10)
rdd: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[2] at parallelize at :21
 
scala> rdd.collect
res3: Array[Int] = Array(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
 
scala> rdd.partitions.size
res4: Int = 15
 
//设置RDD为3个分区
scala> var rdd2 = sc.parallelize(1 to 10,3)
rdd2: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[3] at parallelize at :21
 
scala> rdd2.collect
res5: Array[Int] = Array(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
 
scala> rdd2.partitions.size
res6: Int = 3
```

### makeRDD

def makeRDD[T](seq: Seq[T], numSlices: Int = defaultParallelism)(implicit arg0: ClassTag[T]): RDD[T]

这种用法和parallelize完全相同

def makeRDD[T](seq: Seq[(T, Seq[String])])(implicit arg0: ClassTag[T]): RDD[T]

该用法可以指定每一个分区的preferredLocations。

```
scala> var collect = Seq((1 to 10,Seq("slave007.lxw1234.com","slave002.lxw1234.com")),
(11 to 15,Seq("slave013.lxw1234.com","slave015.lxw1234.com")))
collect: Seq[(scala.collection.immutable.Range.Inclusive, Seq[String])] = List((Range(1, 2, 3, 4, 5, 6, 7, 8, 9, 10),
List(slave007.lxw1234.com, slave002.lxw1234.com)), (Range(11, 12, 13, 14, 15),
List(slave013.lxw1234.com, slave015.lxw1234.com)))
 
scala> var rdd = sc.makeRDD(collect)
rdd: org.apache.spark.rdd.RDD[scala.collection.immutable.Range.Inclusive] 
= ParallelCollectionRDD[6] at makeRDD at :23
 
scala> rdd.partitions.size
res33: Int = 2
 
scala> rdd.preferredLocations(rdd.partitions(0))
res34: Seq[String] = List(slave007.lxw1234.com, slave002.lxw1234.com)
 
scala> rdd.preferredLocations(rdd.partitions(1))
res35: Seq[String] = List(slave013.lxw1234.com, slave015.lxw1234.com)
```
