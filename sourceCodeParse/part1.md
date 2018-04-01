## 1. 通过WordCount实战详解Spark RDD创建

**实例**

```
val wordCount = sc.textFile("/usr/").flatMap(_.split(" ")).map((_,1))
               .reduceByKey(_+_).cache
```

**wordcount stage 图解**

![stage](https://github.com/yueyuanyang/spark_silent/blob/master/sourceCodeParse/img/1.jpg)

**查看执行过程**

cache.toDugStirng

![执行](https://github.com/yueyuanyang/spark_silent/blob/master/sourceCodeParse/img/2.jpg)

注：
1) hdfs -> textFile 阶段会从hdfs 上读取数据
如："/usr"目录下的 HadoopRDD 数据，它的作用是将物理的block块进行逐个inputSplit分片
2) textFile -> map 会将Mappartitions 转化为 MapperRDD
3) stage0 和stage1 的划分根据 shuffleRDD 的宽依赖

**重点**

spark task类型的划分：（依据shuffle划分）
- shuffle 前的为一个类型：该类型称为 shuffleMapTask
- shuffle 后的为一个类型：该类型称为 resultTask

### Spark中RDD的窄依赖NarrowDependency和宽依赖ShuffleDependency详解

宽窄依赖图解
![图解](https://github.com/yueyuanyang/spark_silent/blob/master/sourceCodeParse/img/3.jpg)

宽依赖：父RDD被多个子RDD依赖，即父RDD与子RDD为一对多的关系(groupByKey,join,reduceByKey)

窄依赖：父RDD被一个子RDD依赖，即父RDD与子RDD为一对一的关系(map,filter union)

**重点注意**

- 窄依赖(stage0):内部可以进行pipleLine操作,可以在同一个block块进行一系列的操作
- 提升效率：代码 -> 数据 -> 代码的操作，不产生中间结果
- 高校的原因：窄依赖间进行pipleLine,只需要一次读和一次写入

### 通过案例彻底详解Spark中DAG的逻辑视图的产生机制和过程

**RDD的DAG图构建的过程**

1) DAG 构建的关键： RDD之间有血统关系，即：存在lineage
- 借助于lineage关系，可以保证它计算时，所依赖的父RDD的计算都完成了
- 可以很好的容错性，部分失败，可以借助父RDD重新计算

2) stage 划分

stage 根据宽窄依赖进行划分

3) stage 与stage之间的过程

- stage中的一个结束时，要将数据写入本地文件系统（localFileSystem）
- 下一个stage从上一个stage 的本地文件系统来去数据
- stage查找文件系统目录：stage可以根据Driver端的MapOutPutTrackMaster跟踪部署

**spark的类型（stage）划分**

- 最后一个stage的类型：ResultTask
- 它前面所依赖的stage类型： shuffleMapTask

**shuffle 类型**

- hashShffle
- soreShffle

### RDD Iterator中的缓存处理内幕

缓存发现: 通过cacheManager管理，实际上cacheManager通过扫描blockManager,看看有没有缓存

**RDD partition分区**

- 缓存必须有action操作，当有action操作时，会写入blockManager
- RDD 的每个partition 对应store中的每个blok(partition 通过处理过的数据)

![图](https://github.com/yueyuanyang/spark_silent/blob/master/sourceCodeParse/img/4.jpg)




















