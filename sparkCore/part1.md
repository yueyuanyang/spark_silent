## spark Action 算子

**spark Action 算子**

**RDD行动Action操作(1) – first、count、reduce、collect**

**first**

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

**count**

def count(): Long

count返回RDD中的元素数量。

```
scala> var rdd1 = sc.makeRDD(Array(("A","1"),("B","2"),("C","3")),2)
rdd1: org.apache.spark.rdd.RDD[(String, String)] = ParallelCollectionRDD[34] at makeRDD at :21
 
scala> rdd1.count
res15: Long = 3
```

**reduce**

def reduce(f: (T, T) ⇒ T): T

根据映射函数f，对RDD中的元素进行二元计算，返回计算结果。

```
scala> var rdd1 = sc.makeRDD(1 to 10,2)
rdd1: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[36] at makeRDD at :21
 
scala> rdd1.reduce(_ + _)
res18: Int = 55
 
scala> var rdd2 = sc.makeRDD(Array(("A",0),("A",2),("B",1),("B",2),("C",1)))
rdd2: org.apache.spark.rdd.RDD[(String, Int)] = ParallelCollectionRDD[38] at makeRDD at :21
 
scala> rdd2.reduce((x,y) => {
     |       (x._1 + y._1,x._2 + y._2)
     |     })
res21: (String, Int) = (CBBAA,6)
```

**collect**

def collect(): Array[T]

collect用于将一个RDD转换成数组

```
scala> var rdd1 = sc.makeRDD(1 to 10,2)
rdd1: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[36] at makeRDD at :21
 
scala> rdd1.collect
res23: Array[Int] = Array(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
```





