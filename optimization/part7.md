## Apache Spark 作业性能调优(Part 1)

当你开始编写 Apache Spark 代码或者浏览公开的 API 的时候，你会遇到各种各样术语，比如 transformation，action，RDD 等等。 了解到这些是编写 Spark 代码的基础。 同样，当你任务开始失败或者你需要透过web界面去了解自己的应用为何如此费时的时候，你需要去了解一些新的名词： job, stage, task。对于这些新术语的理解有助于编写良好 Spark 代码。这里的良好主要指更快的 Spark 程序。对于 Spark 底层的执行模型的了解对于写出效率更高的 Spark 程序非常有帮助。

### Spark 是如何执行程序的

一个 Spark 应用包括一个 driver 进程和若干个分布在集群的各个节点上的 executor 进程。

driver 主要负责调度一些高层次的任务流（flow of work）。exectuor 负责执行这些任务，这些任务以 task 的形式存在， 同时存储用户设置需要caching的数据。 task 和所有的 executor 的生命周期为整个程序的运行过程（如果使用了dynamic resource allocation 时可能不是这样的）。如何调度这些进程是通过集群管理应用完成的（比如YARN，Mesos，Spark Standalone），但是任何一个 Spark 程序都会包含一个 driver 和多个 executor 进程。


