## Spark算子：RDD键值转换操作(5)–leftOuterJoin、rightOuterJoin、subtractByKey

## leftOuterJoin

def leftOuterJoin[W](other: RDD[(K, W)]): RDD[(K, (V, Option[W]))]

def leftOuterJoin[W](other: RDD[(K, W)], numPartitions: Int): RDD[(K, (V, Option[W]))]

def leftOuterJoin[W](other: RDD[(K, W)], partitioner: Partitioner): RDD[(K, (V, Option[W]))]

leftOuterJoin类似于SQL中的左外关联left outer join，返回结果以前面的RDD为主，关联不上的记录为空。只能用于两个RDD之间的关联，如果要多个RDD关联，多关联几次即可。

- 参数numPartitions用于指定结果的分区数

- 参数partitioner用于指定分区函数

```
var rdd1 = sc.makeRDD(Array(("A","1"),("B","2"),("C","3")),2)
var rdd2 = sc.makeRDD(Array(("A","a"),("C","c"),("D","d")),2)
 
scala> rdd1.leftOuterJoin(rdd2).collect
res11: Array[(String, (String, Option[String]))] = Array((B,(2,None)), (A,(1,Some(a))), (C,(3,Some(c))))

```

### rightOuterJoin

def rightOuterJoin[W](other: RDD[(K, W)]): RDD[(K, (Option[V], W))]

def rightOuterJoin[W](other: RDD[(K, W)], numPartitions: Int): RDD[(K, (Option[V], W))]

def rightOuterJoin[W](other: RDD[(K, W)], partitioner: Partitioner): RDD[(K, (Option[V], W))]

rightOuterJoin类似于SQL中的有外关联right outer join，返回结果以参数中的RDD为主，关联不上的记录为空。只能用于两个RDD之间的关联，如果要多个RDD关联，多关联几次即可。

- 参数numPartitions用于指定结果的分区数

- 参数partitioner用于指定分区函数

```
var rdd1 = sc.makeRDD(Array(("A","1"),("B","2"),("C","3")),2)
var rdd2 = sc.makeRDD(Array(("A","a"),("C","c"),("D","d")),2)
scala> rdd1.rightOuterJoin(rdd2).collect
res12: Array[(String, (Option[String], String))] = Array((D,(None,d)), (A,(Some(1),a)), (C,(Some(3),c)))

```


### subtractByKey

ubtractByKey
def subtractByKey[W](other: RDD[(K, W)])(implicit arg0: ClassTag[W]): RDD[(K, V)]

def subtractByKey[W](other: RDD[(K, W)], numPartitions: Int)(implicit arg0: ClassTag[W]): RDD[(K, V)]

def subtractByKey[W](other: RDD[(K, W)], p: Partitioner)(implicit arg0: ClassTag[W]): RDD[(K, V)]

 

subtractByKey和基本转换操作中的subtract类似，只不过这里是针对K的，返回在主RDD中出现，并且不在otherRDD中出现的元素。

- 参数numPartitions用于指定结果的分区数

- 参数partitioner用于指定分区函数

```
var rdd1 = sc.makeRDD(Array(("A","1"),("B","2"),("C","3")),2)
var rdd2 = sc.makeRDD(Array(("A","a"),("C","c"),("D","d")),2)
 
scala> rdd1.subtractByKey(rdd2).collect
res13: Array[(String, String)] = Array((B,2))
```

