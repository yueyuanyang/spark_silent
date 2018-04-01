## 通过WordCount实战详解Spark RDD创建

**实例**

```
val wordCount = sc.textFile("/usr/").flatMap(_.split(" ")).map((_,1))
               .reduceByKey(_+_).cache
```

