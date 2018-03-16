## Spark算子：RDD行动Action操作(3)–aggregate、fold、lookup、countByKey、foreach、foreachPartition、sortBy

### aggregate
def aggregate[U](zeroValue: U)(seqOp: (U, T) ⇒ U, combOp: (U, U) ⇒ U)(implicit arg0: ClassTag[U]): U

aggregate用户聚合RDD中的元素，先使用seqOp将RDD中每个分区中的T类型元素聚合成U类型，再使用combOp将之前每个分区聚合后的U类型聚合成U类型，特别注意seqOp和combOp都会使用zeroValue的值，zeroValue的类型为U。

```
var rdd1 = sc.makeRDD(1 to 10,2)
rdd1.mapPartitionsWithIndex{
        (partIdx,iter) => {
          var part_map = scala.collection.mutable.Map[String,List[Int]]()
            while(iter.hasNext){
              var part_name = "part_" + partIdx;
              var elem = iter.next()
              if(part_map.contains(part_name)) {
                var elems = part_map(part_name)
                elems ::= elem
                part_map(part_name) = elems
              } else {
                part_map(part_name) = List[Int]{elem}
              }
            }
            part_map.iterator
           
        }
      }.collect
res16: Array[(String, List[Int])] = Array((part_0,List(5, 4, 3, 2, 1)), (part_1,List(10, 9, 8, 7, 6)))

------------------------------------------------------------------------------------------------------------

##第一个分区中包含5,4,3,2,1

##第二个分区中包含10,9,8,7,6

------------------------------------------------------------------------------------------------------------
scala> rdd1.aggregate(1)(
     |           {(x : Int,y : Int) => x + y}, 
     |           {(a : Int,b : Int) => a + b}
     |     )
res17: Int = 58

结果为什么是58，看下面的计算过程：

------------------------------------------------------------------------------------------------------------

##先在每个分区中迭代执行 (x : Int,y : Int) => x + y 并且使用zeroValue的值1

##即：part_0中 zeroValue+5+4+3+2+1 = 1+5+4+3+2+1 = 16

## part_1中 zeroValue+10+9+8+7+6 = 1+10+9+8+7+6 = 41

##再将两个分区的结果合并(a : Int,b : Int) => a + b ，并且使用zeroValue的值1

##即：zeroValue+part_0+part_1 = 1 + 16 + 41 = 58

------------------------------------------------------------------------------------------------------------

再比如：

scala> rdd1.aggregate(2)(
     |           {(x : Int,y : Int) => x + y}, 
     |           {(a : Int,b : Int) => a * b}
     |     )
res18: Int = 1428
------------------------------------------------------------------------------------------------------------

##这次zeroValue=2

##part_0中 zeroValue+5+4+3+2+1 = 2+5+4+3+2+1 = 17

##part_1中 zeroValue+10+9+8+7+6 = 2+10+9+8+7+6 = 42

##最后：zeroValue*part_0*part_1 = 2 * 17 * 42 = 1428

因此，zeroValue即确定了U的类型，也会对结果产生至关重要的影响，使用时候要特别注意。

```

### fold

def fold(zeroValue: T)(op: (T, T) ⇒ T): T

fold是aggregate的简化，将aggregate中的seqOp和combOp使用同一个函数op。

```
scala> rdd1.fold(1)(
     |       (x,y) => x + y    
     |     )
res19: Int = 58
 
##结果同上面使用aggregate的第一个例子一样，即：
scala> rdd1.aggregate(1)(
     |           {(x,y) => x + y}, 
     |           {(a,b) => a + b}
     |     )
res20: Int = 58
```

### countByKey

def countByKey(): Map[K, Long]

countByKey用于统计RDD[K,V]中每个K的数量。

```
scala> var rdd1 = sc.makeRDD(Array(("A",0),("A",2),("B",1),("B",2),("B",3)))
rdd1: org.apache.spark.rdd.RDD[(String, Int)] = ParallelCollectionRDD[7] at makeRDD at :21
 
scala> rdd1.countByKey
res5: scala.collection.Map[String,Long] = Map(A -> 2, B -> 3)
```

### lookup

def lookup(key: K): Seq[V]

lookup用于(K,V)类型的RDD,指定K值，返回RDD中该K对应的所有V值。

```
scala> var rdd1 = sc.makeRDD(Array(("A",0),("A",2),("B",1),("B",2),("C",1)))
rdd1: org.apache.spark.rdd.RDD[(String, Int)] = ParallelCollectionRDD[0] at makeRDD at :21
 
scala> rdd1.lookup("A")
res0: Seq[Int] = WrappedArray(0, 2)
 
scala> rdd1.lookup("B")
res1: Seq[Int] = WrappedArray(1, 2)
```

### foreach

def foreach(f: (T) ⇒ Unit): Unit

foreach用于遍历RDD,将函数f应用于每一个元素。

但要注意，如果对RDD执行foreach，只会在Executor端有效，而并不是Driver端。

比如：rdd.foreach(println)，只会在Executor的stdout中打印出来，Driver端是看不到的。

我在Spark1.4中是这样，不知道是否真如此。

这时候，使用accumulator共享变量与foreach结合，倒是个不错的选择。

```
scala> var cnt = sc.accumulator(0)
cnt: org.apache.spark.Accumulator[Int] = 0
 
scala> var rdd1 = sc.makeRDD(1 to 10,2)
rdd1: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[5] at makeRDD at :21
 
scala> rdd1.foreach(x => cnt += x)
 
scala> cnt.value
res51: Int = 55
 
scala> rdd1.collect.foreach(println)
1
2
3
4
5
6
7
8
9
10
```

### foreachPartition

def foreachPartition(f: (Iterator[T]) ⇒ Unit): Unit

foreachPartition和foreach类似，只不过是对每一个分区使用f。

```
scala> var rdd1 = sc.makeRDD(1 to 10,2)
rdd1: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[5] at makeRDD at :21
 
scala> var allsize = sc.accumulator(0)
size: org.apache.spark.Accumulator[Int] = 0
 
scala> var rdd1 = sc.makeRDD(1 to 10,2)
rdd1: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[6] at makeRDD at :21
 
scala>     rdd1.foreachPartition { x => {
     |       allsize += x.size
     |     }}
 
scala> println(allsize.value)
10
```

### sortBy

def sortBy[K](f: (T) ⇒ K, ascending: Boolean = true, numPartitions: Int = this.partitions.length)(implicit ord: Ordering[K], ctag: ClassTag[K]): RDD[T]

sortBy根据给定的排序k函数将RDD中的元素进行排序。

```
scala> var rdd1 = sc.makeRDD(Seq(3,6,7,1,2,0),2)
 
scala> rdd1.sortBy(x => x).collect
res1: Array[Int] = Array(0, 1, 2, 3, 6, 7) //默认升序
 
scala> rdd1.sortBy(x => x,false).collect
res2: Array[Int] = Array(7, 6, 3, 2, 1, 0)  //降序
 
//RDD[K,V]类型
scala>var rdd1 = sc.makeRDD(Array(("A",2),("A",1),("B",6),("B",3),("B",7)))
 
scala> rdd1.sortBy(x => x).collect
res3: Array[(String, Int)] = Array((A,1), (A,2), (B,3), (B,6), (B,7))
 
//按照V进行降序排序
scala> rdd1.sortBy(x => x._2,false).collect
res4: Array[(String, Int)] = Array((B,7), (B,6), (B,3), (A,2), (A,1))

```







