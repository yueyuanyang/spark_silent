## Spark算子：RDD基本转换操作(5)–mapPartitions、mapPartitionsWithIndex

### mapPartitions

def mapPartitions[U](f: (Iterator[T]) => Iterator[U], preservesPartitioning: Boolean = false)(implicit arg0: ClassTag[U]): RDD[U]

该函数和map函数类似，只不过映射函数的参数由RDD中的每一个元素变成了RDD中每一个分区的迭代器。如果在映射的过程中需要频繁创建额外的对象，使用mapPartitions要比map高效的过。

比如，将RDD中的所有数据通过JDBC连接写入数据库，如果使用map函数，可能要为每一个元素都创建一个connection，这样开销很大，如果使用mapPartitions，那么只需要针对每一个分区建立一个connection。

参数preservesPartitioning表示是否保留父RDD的partitioner分区信息。

```
var rdd1 = sc.makeRDD(1 to 5,2)
//rdd1有两个分区
scala> var rdd3 = rdd1.mapPartitions{ x => {
     | var result = List[Int]()
     |     var i = 0
     |     while(x.hasNext){
     |       i += x.next()
     |     }
     |     result.::(i).iterator
     | }}
rdd3: org.apache.spark.rdd.RDD[Int] = MapPartitionsRDD[84] at mapPartitions at :23
 
//rdd3将rdd1中每个分区中的数值累加
scala> rdd3.collect
res65: Array[Int] = Array(3, 12)
scala> rdd3.partitions.size
res66: Int = 2
```
### mapPartitionsWithIndex

def mapPartitionsWithIndex[U](f: (Int, Iterator[T]) => Iterator[U], preservesPartitioning: Boolean = false)(implicit arg0: ClassTag[U]): RDD[U]

函数作用同mapPartitions，不过提供了两个参数，第一个参数为分区的索引。

```
ar rdd1 = sc.makeRDD(1 to 5,2)
//rdd1有两个分区
var rdd2 = rdd1.mapPartitionsWithIndex{
        (x,iter) => {
          var result = List[String]()
            var i = 0
            while(iter.hasNext){
              i += iter.next()
            }
            result.::(x + "|" + i).iterator
           
        }
      }
//rdd2将rdd1中每个分区的数字累加，并在每个分区的累加结果前面加了分区索引
scala> rdd2.collect
res13: Array[String] = Array(0|3, 1|12)
```


