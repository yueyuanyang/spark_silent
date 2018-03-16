## Spark算子：RDD行动Action操作(3)–aggregate、fold、lookup

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

##第一个分区中包含5,4,3,2,1

##第二个分区中包含10,9,8,7,6

scala> rdd1.aggregate(1)(
     |           {(x : Int,y : Int) => x + y}, 
     |           {(a : Int,b : Int) => a + b}
     |     )
res17: Int = 58

结果为什么是58，看下面的计算过程：

##先在每个分区中迭代执行 (x : Int,y : Int) => x + y 并且使用zeroValue的值1

##即：part_0中 zeroValue+5+4+3+2+1 = 1+5+4+3+2+1 = 16

## part_1中 zeroValue+10+9+8+7+6 = 1+10+9+8+7+6 = 41

##再将两个分区的结果合并(a : Int,b : Int) => a + b ，并且使用zeroValue的值1

##即：zeroValue+part_0+part_1 = 1 + 16 + 41 = 58

再比如：

scala> rdd1.aggregate(2)(
     |           {(x : Int,y : Int) => x + y}, 
     |           {(a : Int,b : Int) => a * b}
     |     )
res18: Int = 1428

##这次zeroValue=2

##part_0中 zeroValue+5+4+3+2+1 = 2+5+4+3+2+1 = 17

##part_1中 zeroValue+10+9+8+7+6 = 2+10+9+8+7+6 = 42

##最后：zeroValue*part_0*part_1 = 2 * 17 * 42 = 1428

因此，zeroValue即确定了U的类型，也会对结果产生至关重要的影响，使用时候要特别注意。

```
