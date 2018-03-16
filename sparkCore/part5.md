## Spark算子：RDD基本转换操作(1)–map、flatMap、distinct、coalesce、repartition

### map

将一个RDD中的每个数据项，通过map中的函数映射变为一个新的元素。

输入分区与输出分区一对一，即：有多少个输入分区，就有多少个输出分区。

```
hadoop fs -cat /tmp/lxw1234/1.txt
hello world
hello spark
hello hive
 
 
//读取HDFS文件到RDD
scala> var data = sc.textFile("/tmp/lxw1234/1.txt")
data: org.apache.spark.rdd.RDD[String] = MapPartitionsRDD[1] at textFile at :21
 
//使用map算子
scala> var mapresult = data.map(line => line.split("\\s+"))
mapresult: org.apache.spark.rdd.RDD[Array[String]] = MapPartitionsRDD[2] at map at :23
 
//运算map算子结果
scala> mapresult.collect
res0: Array[Array[String]] = Array(Array(hello, world), Array(hello, spark), Array(hello, hive))

```

### flatMap

属于Transformation算子，第一步和 map一样，最后将所有的输出分区合并成一个(扁平化)。
```
/使用flatMap算子
scala> var flatmapresult = data.flatMap(line => line.split("\\s+"))
flatmapresult: org.apache.spark.rdd.RDD[String] = MapPartitionsRDD[3] at flatMap at :23
 
//运算flagMap算子结果
scala> flatmapresult.collect
res1: Array[String] = Array(hello, world, hello, spark, hello, hive)
```

使用flatMap时候需要注意：flatMap会将字符串看成是一个字符数组。

看下面的例子：
```
scala> data.map(_.toUpperCase).collect
res32: Array[String] = Array(HELLO WORLD, HELLO SPARK, HELLO HIVE, HI SPARK)
scala> data.flatMap(_.toUpperCase).collect
res33: Array[Char] = Array(H, E, L, L, O,  , W, O, R, L, D, H, E, L, L, O,  , S, 
P, A, R, K, H, E, L, L, O,  , H, I, V, E, H, I,  , S, P, A, R, K)

scala> data.map(x => x.split("\\s+")).collect
res34: Array[Array[String]] = Array(Array(hello, world), Array(hello, spark), Array(hello, hive), Array(hi, spark))
 
scala> data.flatMap(x => x.split("\\s+")).collect
res35: Array[String] = Array(hello, world, hello, spark, hello, hive, hi, spark)

```

这次的结果好像是预期的，最终结果里面并没有把字符串当成字符数组。这是因为这次map函数中返回的类型为Array[String]，并不是String。flatMap只会将String
扁平化成字符数组，并不会把Array[String]也扁平化成字符数组。

### coalesce

def coalesce(numPartitions: Int, shuffle: Boolean = false)(implicit ord: Ordering[T] = null): RDD[T]

该函数用于将RDD进行重分区，使用HashPartitioner。

第一个参数为重分区的数目，第二个为是否进行shuffle，默认为false;

以下面的例子来看：

```
scala> var data = sc.textFile("/tmp/lxw1234/1.txt")
data: org.apache.spark.rdd.RDD[String] = MapPartitionsRDD[53] at textFile at :21
 
scala> data.collect
res37: Array[String] = Array(hello world, hello spark, hello hive, hi spark)
 
scala> data.partitions.size
res38: Int = 2  //RDD data默认有两个分区
 
scala> var rdd1 = data.coalesce(1)
rdd1: org.apache.spark.rdd.RDD[String] = CoalescedRDD[2] at coalesce at :23
 
scala> rdd1.partitions.size
res1: Int = 1   //rdd1的分区数为1
 
 
scala> var rdd1 = data.coalesce(4)
rdd1: org.apache.spark.rdd.RDD[String] = CoalescedRDD[3] at coalesce at :23
 
scala> rdd1.partitions.size
res2: Int = 2   //如果重分区的数目大于原来的分区数，那么必须指定shuffle参数为true，//否则，分区数不便
 
scala> var rdd1 = data.coalesce(4,true)
rdd1: org.apache.spark.rdd.RDD[String] = MapPartitionsRDD[7] at coalesce at :23
 
scala> rdd1.partitions.size
res3: Int = 4
```

### repartition

def repartition(numPartitions: Int)(implicit ord: Ordering[T] = null): RDD[T]

该函数其实就是coalesce函数第二个参数为true的实现

```
scala> var rdd2 = data.repartition(1)
rdd2: org.apache.spark.rdd.RDD[String] = MapPartitionsRDD[11] at repartition at :23
 
scala> rdd2.partitions.size
res4: Int = 1
 
scala> var rdd2 = data.repartition(4)
rdd2: org.apache.spark.rdd.RDD[String] = MapPartitionsRDD[15] at repartition at :23
 
scala> rdd2.partitions.size
res5: Int = 4
```



