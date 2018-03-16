## spark Action 算子

**spark Action 算子**

**RDD行动Action操作(1)–first、count、reduce、collect**

def first(): T

first返回RDD中的第一个元素，不排序。

```
scala> var rdd1 = sc.makeRDD(Array(("A","1"),("B","2"),("C","3")),2)
rdd1: org.apache.spark.rdd.RDD[(String, String)] = ParallelCollectionRDD[33] at makeRDD at :21
 
scala> rdd1.first
res14: (String, String) = (A,1)
 
scala> var rdd1 = sc.makeRDD(Seq(10, 4, 2, 12, 3))
rdd1: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[0] at makeRDD at :21
 
scala> rdd1.first
res8: Int = 10

```

