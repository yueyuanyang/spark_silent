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

(2) 每个节点总内存*(85-90)% / 每个节点的executor数量

(3)  每个核心2-5 G：2-5GB : 2-5 * sparkk.executor.cores

- spark.yarn.executor.memoryOverhead -
