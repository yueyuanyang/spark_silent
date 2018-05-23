
## 目录

### spark 杂记
1.[spark杂记之一 —— 如何设置spark日志的打印级别](https://github.com/yueyuanyang/spark/blob/master/notes/part1.md)

## 第一部分

### spark 概念

1.[Apache Spark RDD 详解](https://github.com/yueyuanyang/spark_silent/blob/master/notes/part3.md)

2.[Apache Spark 统一内存管理模型详解](https://github.com/yueyuanyang/spark/blob/master/notes/part2.md)

3.[Apache Spark Shuffle模型——HashShuffle和SortShuffle](https://github.com/yueyuanyang/spark/blob/master/notes/part4.md)


## 第二部分 
### spark-core
#### Action操作
1. [Spark RDD行动Action操作(1) – first、count、reduce、collect、take、top、takeOrdered](https://github.com/yueyuanyang/spark/tree/master/sparkCore/part1.md)
2. [Spark RDD行动Action操作(2) - aggregate、fold、lookup、countByKey、foreach、foreachPartition、sortBy](https://github.com/yueyuanyang/spark/tree/master/sparkCore/part2.md)
3. [Spark RDD行动Action操作(3) - saveAsNewAPIHadoopFile、saveAsNewAPIHadoopDataset](https://github.com/yueyuanyang/spark/tree/master/sparkCore/part4.md)
4. [Spark RDD行动Action操作(4) - saveAsTextFile、saveAsSequenceFile、saveAsObjectFile、saveAsHadoopFile、saveAsHadoopDataset](https://github.com/yueyuanyang/spark/tree/master/sparkCore/part3.md)

#### Transformation操作
1. [Spark RDD行动Transformation操作(1) - map、flatMap、distinct、coalesce、repartition](https://github.com/yueyuanyang/spark/tree/master/sparkCore/part5.md)
2. [Spark RDD行动Transformation操作(2) -  randomSplit、glom、union、intersection、subtract](https://github.com/yueyuanyang/spark/tree/master/sparkCore/part6.md)
3. [Spark RDD行动Transformation操作(3) - mapPartitions、mapPartitionsWithIndex](https://github.com/yueyuanyang/spark/tree/master/sparkCore/part7.md)
4. [Spark RDD行动Transformation操作(5) - zip、zipPartitions、zipWithIndex、zipWithUniqueId](https://github.com/yueyuanyang/spark/tree/master/sparkCore/part8.md)
5. [Spark RDD行动Transformation操作(6) - parallelize、makeRDD](https://github.com/yueyuanyang/spark/tree/master/sparkCore/part9.md)

#### 键值转换

1. [Spark RDD行动键值操作(1) - partitionBy、mapValues、flatMapValues](https://github.com/yueyuanyang/spark/tree/master/sparkCore/part10.md)
2. [Spark RDD行动键值操作(1) - combineByKey、foldByKey](https://github.com/yueyuanyang/spark/tree/master/sparkCore/part11.md)
3. [Spark RDD行动键值操作(1) - groupByKey、reduceByKey、reduceByKeyLocally](https://github.com/yueyuanyang/spark/tree/master/sparkCore/part12.md)
4. [Spark RDD行动键值操作(1) - cogroup、join](https://github.com/yueyuanyang/spark/tree/master/sparkCore/part13.md)
5. [Spark RDD行动键值操作(1) - leftOuterJoin、rightOuterJoin、subtractByKey](https://github.com/yueyuanyang/spark/tree/master/sparkCore/part15.md)


## 第二部分 
### spark graph(图计算)
1. [spark graph 构建graph和聚合消息](https://github.com/yueyuanyang/spark/blob/master/graph/part1.md)
2. [spark graph 操作01 —— edgeListFile导入数据](https://github.com/yueyuanyang/spark/blob/master/graph/part2.md)
3. [spark graph 操作02 —— joinVertices](https://github.com/yueyuanyang/spark/blob/master/graph/part3.md)
4. [spark graph 操作03 —— 导入顶点和边生成图](https://github.com/yueyuanyang/spark/blob/master/graph/part4.md)
5. [spark graph 操作04 —— 使用mapReduceTriplets、mapEdges、mapVertices、aggregateMessages修改属性](https://github.com/yueyuanyang/spark/blob/master/graph/part5.md)
6. [spark graph 操作05 —— VertexRDD和EdgeRDD属性测试](https://github.com/yueyuanyang/spark/blob/master/graph/part6.md)
7. [spark graph 操作06 —— subgraph和groupEdges](https://github.com/yueyuanyang/spark/blob/master/graph/part7.md)
8. [spark graph 操作07 —— degrees和neighbors](https://github.com/yueyuanyang/spark/blob/master/graph/part8.md)
9. [spark graph 操作08 —— connectedComponents](https://github.com/yueyuanyang/spark/blob/master/graph/part9.md)
10. [spark graph 操作09 —— Pregel学习](https://github.com/yueyuanyang/spark/blob/master/graph/part10.md)
11. [spark graph 操作10 —— GraphFrame学习(类Sql第三方库)](https://github.com/yueyuanyang/spark/blob/master/graph/part11.md)

## 第N部分
### spark性能优化
1. [最优化spark应用的性能](https://github.com/yueyuanyang/spark/blob/master/optimization/part1.md)
2. [Spark性能优化指南——基础篇](https://github.com/yueyuanyang/spark/blob/master/optimization/part2.md)
3. [Spark性能优化指南——高级篇](https://github.com/yueyuanyang/spark/blob/master/optimization/part3.md)
4. [spark-submit提交参数列表](https://github.com/yueyuanyang/spark/blob/master/optimization/part4.md)
5. [spark submit参数调优](https://github.com/yueyuanyang/spark/blob/master/optimization/part5.md)
6. [spark 配置详解列表](https://github.com/yueyuanyang/spark_silent/blob/master/optimization/part6.md)

## 第N+1部分
### spark 操作篇

[1. spark 优雅的操作redis](https://github.com/yueyuanyang/spark_silent/blob/master/operation/part1.md)

[2. spark 优雅的操作redis(官方文档)](https://github.com/yueyuanyang/spark_silent/blob/master/operation/part2.md)

[3. Apache Spark将数据写入\读取 ElasticSearch](https://github.com/yueyuanyang/spark_silent/blob/master/operation/part3.md)

## 第N+2部分
### spark 面试篇
[1. spark面试题一](https://github.com/yueyuanyang/spark_silent/blob/master/operation/part2.md)


