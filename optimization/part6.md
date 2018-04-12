## Spark配置

### Spark配置

Spark有以下三种方式修改配置：

- Spark properties （Spark属性）可以控制绝大多数应用程序参数，而且既可以通过 SparkConf 对象来设置，也可以通过Java系统属性来设置。
- Environment variables （环境变量）可以指定一些各个机器相关的设置，如IP地址，其设置方法是写在每台机器上的conf/spark-env.sh中。
- Logging （日志）可以通过log4j.properties配置日志。

### Spark属性

Spark属性可以控制大多数的应用程序设置，并且每个应用的设定都是分开的。这些属性可以用SparkConf 对象直接设定。SparkConf为一些常用的属性定制了专用方法（如，master URL和application name），其他属性都可以用键值对做参数，调用set()方法来设置。例如，我们可以初始化一个包含2个本地线程的Spark应用，代码如下：

注意，local[2]代表2个本地线程 – 这是最小的并发方式，可以帮助我们发现一些只有在分布式上下文才能复现的bug。

```
val conf = new SparkConf()
             .setMaster("local[2]")
             .setAppName("CountingSheep")
val sc = new SparkContext(conf)
```

### 动态加载Spark属性

在某些场景下，你可能需要避免将属性值写死在 SparkConf 中。例如，你可能希望在同一个应用上使用不同的master或不同的内存总量。Spark允许你简单地创建一个空的SparkConf对象：
```
val sc = new SparkContext(new SparkConf())
```

然后在运行时设置这些属性：

```
./bin/spark-submit \
--name "My app" \
--master local[4] \
--conf spark.eventLog.enabled=false \
--conf "spark.executor.extraJavaOptions=-XX:+PrintGCDetails -XX:+PrintGCTimeStamps" myApp.jar
```

Spark shell和spark-submit工具支持两种动态加载配置的方法。第一种，通过命令行选项，如：上面提到的–master（设置master URL）。spark-submit可以在启动Spark应用时，通过–conf标志接受任何属性配置，同时有一些特殊配置参数同样可用（如，–master）。运行./bin/spark-submit –help可以展示这些选项的完整列表。

同时，bin/spark-submit 也支持从conf/spark-defaults.conf 中读取配置选项，在该文件中每行是一个键值对，并用空格分隔，如下：

```
spark.master            spark://5.6.7.8:7077
spark.executor.memory   4g
spark.eventLog.enabled  true
spark.serializer        org.apache.spark.serializer.KryoSerializer
```

这些通过参数或者属性配置文件传递的属性，最终都会在SparkConf 中合并。其优先级是：首先是SparkConf代码中写的属性值，其次是spark-submit或spark-shell的标志参数，最后是spark-defaults.conf文件中的属性。

有一些配置项被重命名过，这种情形下，老的名字仍然是可以接受的，只是优先级比新名字优先级低。

### 查看Spark属性

每个SparkContext都有其对应的Spark UI，所以Spark应用程序都能通过Spark UI查看其属性。默认你可以在这里看到：http://<driver>:4040，页面上的”Environment“ tab页可以查看Spark属性。如果你真的想确认一下属性设置是否正确的话，这个功能就非常有用了。注意，只有显式地通过SparkConf对象、在命令行参数、或者spark-defaults.conf设置的参数才会出现在页面上。其他属性，你可以认为都是默认值。

### 可用的属性

绝大多数属性都有合理的默认值。这里是部分常用的选项：

应用属性

| 属性名称 | 默认值 | 含义 
| - | :-: | -: 
|spark.app.name	| (none) |	Spark应用的名字。会在SparkUI和日志中出现。
|spark.driver.cores	| 1	| 在cluster模式下，用几个core运行驱动器（driver）进程。
|spark.driver.maxResultSize	| 1g |	Spark action算子返回的结果最大多大。至少要1M，可以设为0表示无限制。如果结果超过这一大小，Spark作业（job）会直接中断退出。但是，设得过高有可能导致驱动器OOM（out-of-memory）（取决于spark.driver.memory设置，以及驱动器JVM的内存限制）。设一个合理的值，以避免驱动器OOM。
|spark.driver.memory	| 1g	 |驱动器进程可以用的内存总量（如：1g，2g）。注意，在客户端模式下，这个配置不能在SparkConf中直接设置（因为驱动器JVM都启动完了呀！）。驱动器客户端模式下，必须要在命令行里用 –driver-memory 或者在默认属性配置文件里设置。
|spark.executor.memory	| 1g	| 单个执行器（executor）使用的内存总量（如，2g，8g）
|spark.extraListeners	|(none)	|逗号分隔的SparkListener子类的类名列表；初始化SparkContext时，这些类的实例会被创建出来，并且注册到Spark的监听总线上。如果这些类有一个接受SparkConf作为唯一参数的构造函数，那么这个构造函数会被优先调用；否则，就调用无参数的默认构造函数。如果没有构造函数，SparkContext创建的时候会抛异常。
|spark.local.dir |	/tmp	| Spark的”草稿“目录，包括map输出的临时文件，或者RDD存在磁盘上的数据。这个目录最好在本地文件系统中，这样读写速度快。这个配置可以接受一个逗号分隔的列表，通常用这种方式将文件IO分散不同的磁盘上去。注意：Spark-1.0及以后版本中，这个属性会被集群管理器所提供的环境变量覆盖：SPARK_LOCAL_DIRS（独立部署或Mesos）或者 LOCAL_DIRS（YARN）。
|spark.logConf |	false	| SparkContext启动时是否把生效的 SparkConf 属性以INFO日志打印到日志里
|spark.master	| (none)	| 集群管理器URL。参考allowed master URL’s.

除了这些以外，以下还有很多可用的参数配置，在某些特定情形下，可能会用到：

### 运行时环境

| 属性名称 | 默认值 | 含义 
| - | :-: | -: 
|spark.driver.extraClassPath | (none) | 额外的classpath，将插入到到驱动器的classpath开头。注意：驱动器如果运行客户端模式下，这个配置不能通过SparkConf 在程序里配置，因为这时候程序已经启动呀！而是应该用命令行参数（–driver-class-path）或者在 conf/spark-defaults.conf 配置。
|spark.driver.extraJavaOptions | (none) | 驱动器额外的JVM选项。如：GC设置或其他日志参数。注意：驱动器如果运行客户端模式下，这个配置不能通过SparkConf在程序里配置，因为这时候程序已经启动呀！而是应该用命令行参数（–driver-java-options）或者conf/spark-defaults.conf 配置。
|spark.driver.extraLibraryPath | (none) | 启动驱动器JVM时候指定的依赖库路径。注意：驱动器如果运行客户端模式下，这个配置不能通过SparkConf在程序里配置，因为这时候程序已经启动呀！而是应该用命令行参数（–driver-library-path）或者conf/spark-defaults.conf 配置。
|spark.driver.userClassPathFirst | false | (试验性的：即未来不一定会支持该配置) 驱动器是否首选使用用户指定的jars，而不是spark自身的。这个特性可以用来处理用户依赖和spark本身依赖项之间的冲突。目前还是试验性的，并且只能用在集群模式下。
|spark.executor.extraClassPath | (none) | 添加到执行器（executor）classpath开头的classpath。主要为了向后兼容老的spark版本，不推荐使用。
|spark.executor.extraJavaOptions | (none) | 传给执行器的额外JVM参数。如：GC设置或其他日志设置等。注意，不能用这个来设置Spark属性或者堆内存大小。Spark属性应该用SparkConf对象，或者spark-defaults.conf文件（会在spark-submit脚本中使用）来配置。执行器堆内存大小应该用 spark.executor.memory配置。
|spark.executor.extraLibraryPath | (none) | 启动执行器JVM时使用的额外依赖库路径。
|spark.executor.logs.rolling.maxRetainedFiles | (none) | Sets the number of latest rolling log files that are going to be retained by the system. Older log files will be deleted. Disabled by default.设置日志文件最大保留个数。老日志文件将被干掉。默认禁用的。
|spark.executor.logs.rolling.maxSize | (none) | 设置执行器日志文件大小上限。默认禁用的。需要自动删日志请参考 spark.executor.logs.rolling.maxRetainedFiles.
|spark.executor.logs.rolling.strategy | (none) | 执行器日志滚动策略。默认禁用。可接受的值有”time”（基于时间滚动） 或者 “size”（基于文件大小滚动）。time：将使用 spark.executor.logs.rolling.time.interval设置滚动时间间隔size：将使用 spark.executor.logs.rolling.size.maxBytes设置文件大小上限
|spark.executor.logs.rolling.time.interval | daily | 设置执行器日志滚动时间间隔。日志滚动默认是禁用的。可用的值有 “daily”, “hourly”, “minutely”，也可设为数字（则单位为秒）。关于日志自动清理，请参考 spark.executor.logs.rolling.maxRetainedFiles
|spark.executor.userClassPathFirst | false | （试验性的）与 spark.driver.userClassPathFirst类似，只不过这个参数将应用于执行器
|spark.executorEnv.[EnvironmentVariableName] | (none) | 向执行器进程增加名为EnvironmentVariableName的环境变量。用户可以指定多个来设置不同的环境变量。
|spark.python.profile | false | 对Python worker启用性能分析，性能分析结果会在sc.show_profile()中，或者在驱动器退出前展示。也可以用sc.dump_profiles(path)输出到磁盘上。如果部分分析结果被手动展示过，那么驱动器退出前就不再自动展示了。默认会使用pyspark.profiler.BasicProfiler，也可以自己传一个profiler 类参数给SparkContext构造函数。
|spark.python.profile.dump | (none) | 这个目录是用来在驱动器退出前，dump性能分析结果。性能分析结果会按RDD分别dump。同时可以使用ptats.Stats()来装载。如果制定了这个，那么分析结果就不再自动展示。
|spark.python.worker.memory | 512m | 聚合时每个python worker使用的内存总量，和JVM的内存字符串格式相同（如，512m，2g）。如果聚合时使用的内存超过这个量，就将数据溢出到磁盘上。
|spark.python.worker.reuse | true | 是否复用Python worker。如果是，则每个任务会启动固定数量的Python worker，并且不需要fork() python进程。如果需要广播的数据量很大，设为true能大大减少广播数据量，因为需要广播的进程数减少了。







