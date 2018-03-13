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

- number od executor per node - 

