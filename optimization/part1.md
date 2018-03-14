## spark 性能优化
### 一般软件调优
- 资源分配
-  序列化
- IO
- MISC

#### 资源分配 - CPU

- .executor.cores - recommend 5 cores per executor*(每个executor分配5个cores)

(1) 较少的核心数量(像每个执行者的单核心)会引入JVM开销,e.g:多个广播副本

(2) 更多的内核数量可能难以应用大型资源

(3) 在整个HDFS上实现完整写入

- number of executor per node -  cores per mode / 5 * (1-0.9) (每个节点的核心数 % 5 * (1 - 0.9))

$ 查看逻辑CPU的个数

[root@AAA ~]# cat /proc/cpuinfo| grep "processor"| wc -l

#### 资源分配 - Memory

- spark.executor.memory - memory size per executor (每个executor的内存大小)

(1) 至少留下10-15% 给系统缓存：,page cache etc

(2) 每个节点(总内存*(85-90)%) / 每个节点的executor数量

(3) 每个核心2-5 G：2-5GB : 2-5GB * spark.executor.cores

- spark.yarn.executor.memoryOverhead -  indicte for offhead memory size,increasing that to avoid killing by tarn NM
(指出额外的内存大小，增加以避免通过tarn NM杀死)

(1)有时候默认的值(384,0.7 * sparkexecutor.memory)太小,netty可能会大量使用它们

(2) yarn.noemanager.resource.memory-mb=spark.yarn.executor.memoryOerhead + spark.executor.memory

## 软件调整 - 序列化
- spark.serializer - 类使用序列化对象

(1) 为提高速度，序列化是必须的，它能带来15-20%的提高
```
KryoSerialization速度快，可以配置为任何org.apache.spark.serializer的子类。
但Kryo也不支持所有实现了 java.io.Serializable 接口的类型，
它需要你在程序中 register 需要序列化的类型，以得到最佳性能

在 SparkConf 初始化的时候调用 

conf.set(“spark.serializer”, “org.apache.spark.serializer.KryoSerializer”) 

使用 Kryo。这个设置不仅控制各个worker节点之间的混洗数据序列化格式，
同时还控制RDD存到磁盘上的序列化格式。需要在使用时注册需要序列化的类型，
建议在对网络敏感的应用场景下使用Kryo。

如果你的自定义类型需要使用Kryo序列化，可以用 registerKryoClasses 方法先注册：

val conf = new SparkConf.setMaster(...).setAppName(...)
conf.registerKryoClasses(Array(classOf[MyClass1], classOf[MyClass2]))
val sc = new SparkContext(conf)

最后，如果你不注册需要序列化的自定义类型，Kryo也能工作，
不过每一个对象实例的序列化结果都会包含一份完整的类名，这有点浪费空间。
 
```



