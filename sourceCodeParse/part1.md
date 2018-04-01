## 1. 通过WordCount实战详解Spark RDD创建

**实例**

```
val wordCount = sc.textFile("/usr/").flatMap(_.split(" ")).map((_,1))
               .reduceByKey(_+_).cache
```

**wordcount stage 图解**

[stage](https://github.com/yueyuanyang/spark_silent/blob/master/sourceCodeParse/img/1.jpg)

**查看执行过程**

cache.toDugStirng

[](https://github.com/yueyuanyang/spark_silent/blob/master/sourceCodeParse/img/2.jpg)

注：
1) hdfs -> textFile 阶段会从hdfs 上读取数据
如："/usr"目录下的 HadoopRDD 数据，它的作用是将物理的block块进行逐个inputSplit分片
2) textFile -> map 会将Mappartitions 转化为 MapperRDD
3) stage0 和stage1 的划分根据 shuffleRDD 的宽依赖

**重点**

spark task类型的划分：（依据shuffle划分）
- shuffle 前的为一个类型：该类型称为 shuffleMapTask
- shuffle 后的为一个类型：该类型称为 resultTask






