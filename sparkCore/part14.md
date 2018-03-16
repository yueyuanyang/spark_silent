## Spark算子：RDD键值转换操作(4)–cogroup、join

### cogroup

- 参数为1个RDD

def cogroup[W](other: RDD[(K, W)]): RDD[(K, (Iterable[V], Iterable[W]))]

def cogroup[W](other: RDD[(K, W)], numPartitions: Int): RDD[(K, (Iterable[V], Iterable[W]))]

def cogroup[W](other: RDD[(K, W)], partitioner: Partitioner): RDD[(K, (Iterable[V], Iterable[W]))]

- 参数为2个RDD

def cogroup[W1, W2](other1: RDD[(K, W1)], other2: RDD[(K, W2)]): RDD[(K, (Iterable[V], Iterable[W1], Iterable[W2]))]

def cogroup[W1, W2](other1: RDD[(K, W1)], other2: RDD[(K, W2)], numPartitions: Int): RDD[(K, (Iterable[V], Iterable[W1], Iterable[W2]))]

def cogroup[W1, W2](other1: RDD[(K, W1)], other2: RDD[(K, W2)], partitioner: Partitioner): RDD[(K, (Iterable[V], Iterable[W1], Iterable[W2]))]

- 参数为3个RDD

def cogroup[W1, W2, W3](other1: RDD[(K, W1)], other2: RDD[(K, W2)], other3: RDD[(K, W3)]): RDD[(K, (Iterable[V], Iterable[W1], Iterable[W2], Iterable[W3]))]

def cogroup[W1, W2, W3](other1: RDD[(K, W1)], other2: RDD[(K, W2)], other3: RDD[(K, W3)], numPartitions: Int): RDD[(K, (Iterable[V], Iterable[W1], Iterable[W2], Iterable[W3]))]

def cogroup[W1, W2, W3](other1: RDD[(K, W1)], other2: RDD[(K, W2)], other3: RDD[(K, W3)], partitioner: Partitioner): RDD[(K, (Iterable[V], Iterable[W1], Iterable[W2], Iterable[W3]))]


cogroup相当于SQL中的全外关联full outer join，返回左右RDD中的记录，关联不上的为空。

- 参数numPartitions用于指定结果的分区数。

- 参数partitioner用于指定分区函数。

#### 参数为1个RDD的例子
```
var rdd1 = sc.makeRDD(Array(("A","1"),("B","2"),("C","3")),2)
var rdd2 = sc.makeRDD(Array(("A","a"),("C","c"),("D","d")),2)
 
scala> var rdd3 = rdd1.cogroup(rdd2)
rdd3: org.apache.spark.rdd.RDD[(String, (Iterable[String], Iterable[String]))] = MapPartitionsRDD[12] at cogroup at :25
 
scala> rdd3.partitions.size
res3: Int = 2
 
scala> rdd3.collect
res1: Array[(String, (Iterable[String], Iterable[String]))] = Array(
(B,(CompactBuffer(2),CompactBuffer())), 
(D,(CompactBuffer(),CompactBuffer(d))), 
(A,(CompactBuffer(1),CompactBuffer(a))), 
(C,(CompactBuffer(3),CompactBuffer(c)))
)
 
 
scala> var rdd4 = rdd1.cogroup(rdd2,3)
rdd4: org.apache.spark.rdd.RDD[(String, (Iterable[String], Iterable[String]))] = MapPartitionsRDD[14] at cogroup at :25
 
scala> rdd4.partitions.size
res5: Int = 3
 
scala> rdd4.collect
res6: Array[(String, (Iterable[String], Iterable[String]))] = Array(
(B,(CompactBuffer(2),CompactBuffer())), 
(C,(CompactBuffer(3),CompactBuffer(c))), 
(A,(CompactBuffer(1),CompactBuffer(a))), 
(D,(CompactBuffer(),CompactBuffer(d))))
```

#### 参数为2个RDD的例子
```

var rdd1 = sc.makeRDD(Array(("A","1"),("B","2"),("C","3")),2)
var rdd2 = sc.makeRDD(Array(("A","a"),("C","c"),("D","d")),2)
var rdd3 = sc.makeRDD(Array(("A","A"),("E","E")),2)
 
scala> var rdd4 = rdd1.cogroup(rdd2,rdd3)
rdd4: org.apache.spark.rdd.RDD[(String, (Iterable[String], Iterable[String], Iterable[String]))] = 
MapPartitionsRDD[17] at cogroup at :27
 
scala> rdd4.partitions.size
res7: Int = 2
 
scala> rdd4.collect
res9: Array[(String, (Iterable[String], Iterable[String], Iterable[String]))] = Array(
(B,(CompactBuffer(2),CompactBuffer(),CompactBuffer())), 
(D,(CompactBuffer(),CompactBuffer(d),CompactBuffer())), 
(A,(CompactBuffer(1),CompactBuffer(a),CompactBuffer(A))), 
(C,(CompactBuffer(3),CompactBuffer(c),CompactBuffer())), 
(E,(CompactBuffer(),CompactBuffer(),CompactBuffer(E))))

```

#### join

def join[W](other: RDD[(K, W)]): RDD[(K, (V, W))]

def join[W](other: RDD[(K, W)], numPartitions: Int): RDD[(K, (V, W))]

def join[W](other: RDD[(K, W)], partitioner: Partitioner): RDD[(K, (V, W))]

 

join相当于SQL中的内关联join，只返回两个RDD根据K可以关联上的结果，join只能用于两个RDD之间的关联，如果要多个RDD关联，多关联几次即可。

- 参数numPartitions用于指定结果的分区数

- 参数partitioner用于指定分区函数

```
var rdd1 = sc.makeRDD(Array(("A","1"),("B","2"),("C","3")),2)
var rdd2 = sc.makeRDD(Array(("A","a"),("C","c"),("D","d")),2)
 
scala> rdd1.join(rdd2).collect
res10: Array[(String, (String, String))] = Array((A,(1,a)), (C,(3,c)))

```




