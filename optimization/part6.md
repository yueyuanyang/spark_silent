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

### 应用属性

| 属性名称 | 默认值 | 含义 
| - | - | - |
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
| - | - | - |
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


### 混洗行为

| 属性名称 | 默认值 | 含义 
| - | - | - |
|spark.reducer.maxSizeInFlight | 48m | map任务输出同时reduce任务获取的最大内存占用量。每个输出需要创建buffer来接收，对于每个reduce任务来说，有一个固定的内存开销上限，所以最好别设太大，除非你内存非常大。
|spark.shuffle.compress | true | 是否压缩map任务的输出文件。通常来说，压缩是个好主意。使用的压缩算法取决于 spark.io.compression.codec
|spark.shuffle.file.buffer | 32k | 每个混洗输出流的内存buffer大小。这个buffer能减少混洗文件的创建和磁盘寻址。
|spark.shuffle.io.maxRetries | 3 | （仅对netty）如果IO相关异常发生，重试次数（如果设为非0的话）。重试能是大量数据的混洗操作更加稳定，因为重试可以有效应对长GC暂停或者网络闪断。
|spark.shuffle.io.numConnectionsPerPeer | 1 | （仅netty）主机之间的连接是复用的，这样可以减少大集群中重复建立连接的次数。然而，有些集群是机器少，磁盘多，这种集群可以考虑增加这个参数值，以便充分利用所有磁盘并发性能。
|spark.shuffle.io.preferDirectBufs | true | （仅netty）堆外缓存可以有效减少垃圾回收和缓存复制。对于堆外内存紧张的用户来说，可以考虑禁用这个选项，以迫使netty所有内存都分配在堆上。
|spark.shuffle.io.retryWait | 5s | （仅netty）混洗重试获取数据的间隔时间。默认最大重试延迟是15秒，设置这个参数后，将变成maxRetries* retryWait。
|spark.shuffle.manager | sort | 混洗数据的实现方式。可用的有”sort”和”hash“。基于排序（sort）的混洗内存利用率更高，并且从1.2开始已经是默认值了。
|spark.shuffle.service.enabled | false | 启用外部混洗服务。启用外部混洗服务后，执行器生成的混洗中间文件就由该服务保留，这样执行器就可以安全的退出了。如果 spark.dynamicAllocation.enabled启用了，那么这个参数也必须启用，这样动态分配才能有外部混洗服务可用。
|spark.shuffle.service.port | 7337 | 外部混洗服务对应端口
|spark.shuffle.sort.bypassMergeThreshold | 200 | （高级）在基于排序（sort）的混洗管理器中，如果没有map端聚合的话，就会最多存在这么多个reduce分区。
|spark.shuffle.spill.compress | true | 是否在混洗阶段压缩溢出到磁盘的数据。压缩算法取决于spark.io.compression.codec


### Spark UI

| 属性名称 | 默认值 | 含义 
| - | - | - |
|spark.eventLog.compress | false | 是否压缩事件日志（当然spark.eventLog.enabled必须开启）
|spark.eventLog.dir | file:///tmp/spark-events | Spark events日志的基础目录（当然spark.eventLog.enabled必须开启）。在这个目录中，spark会给每个应用创建一个单独的子目录，然后把应用的events log打到子目录里。用户可以设置一个统一的位置（比如一个HDFS目录），这样history server就可以从这里读取历史文件。
|spark.eventLog.enabled | false | 是否启用Spark事件日志。如果Spark应用结束后，仍需要在SparkUI上查看其状态，必须启用这个。
|spark.ui.killEnabled | true | 允许从SparkUI上杀掉stage以及对应的作业（job）
|spark.ui.port | 4040 | SparkUI端口，展示应用程序运行状态。
|spark.ui.retainedJobs | 1000 | SparkUI和status API最多保留多少个spark作业的数据（当然是在垃圾回收之前）
|spark.ui.retainedStages | 1000 | SparkUI和status API最多保留多少个spark步骤（stage）的数据（当然是在垃圾回收之前）
|spark.worker.ui.retainedExecutors | 1000 | SparkUI和status API最多保留多少个已结束的执行器（executor）的数据（当然是在垃圾回收之前）
|spark.worker.ui.retainedDrivers | 1000 | SparkUI和status API最多保留多少个已结束的驱动器（driver）的数据（当然是在垃圾回收之前）
|spark.sql.ui.retainedExecutions | 1000 | SparkUI和status API最多保留多少个已结束的执行计划（execution）的数据（当然是在垃圾回收之前）
|spark.streaming.ui.retainedBatches | 1000 | SparkUI和status API最多保留多少个已结束的批量（batch）的数据（当然是在垃圾回收之前）


### 压缩和序列化

| 属性名称 | 默认值 | 含义 
| - | - | - |
|spark.broadcast.compress | true | 是否在广播变量前使用压缩。通常是个好主意。
|spark.closure.serializer | org.apache.spark.serializer.JavaSerializer | 闭包所使用的序列化类。目前只支持Java序列化。
|spark.io.compression.codec | snappy | 内部数据使用的压缩算法，如：RDD分区、广播变量、混洗输出。Spark提供了3中算法：lz4，lzf，snappy。你也可以使用全名来指定压缩算法：org.apache.spark.io.LZ4CompressionCodec,org.apache.spark.io.LZFCompressionCodec,org.apache.spark.io.SnappyCompressionCodec.
|spark.io.compression.lz4.blockSize | 32k | LZ4算法使用的块大小。当然你需要先使用LZ4压缩。减少块大小可以减少混洗时LZ4算法占用的内存量。
|spark.io.compression.snappy.blockSize | 32k | Snappy算法使用的块大小（先得使用Snappy算法）。减少块大小可以减少混洗时Snappy算法占用的内存量。
|spark.kryo.classesToRegister | (none) | 如果你使用Kryo序列化，最好指定这个以提高性能（tuning guide）。本参数接受一个逗号分隔的类名列表，这些类都会注册为Kryo可序列化类型。
|spark.kryo.referenceTracking | true (false when using Spark SQL Thrift Server) | 是否跟踪同一对象在Kryo序列化的引用。如果你的对象图中有循环护着包含统一对象的多份拷贝，那么最好启用这个。如果没有这种情况，那就禁用以提高性能。
|spark.kryo.registrationRequired | false | Kryo序列化时，是否必须事先注册。如果设为true，那么Kryo遇到没有注册过的类型，就会抛异常。如果设为false（默认）Kryo会序列化未注册类型的对象，但会有比较明显的性能影响，所以启用这个选项，可以强制必须在序列化前，注册可序列化类型。
|spark.kryo.registrator | (none) | 如果你使用Kryo序列化，用这个class来注册你的自定义类型。如果你需要自定义注册方式，这个参数很有用。否则，使用 spark.kryo.classesRegister更简单。要设置这个参数，需要用KryoRegistrator的子类。详见：tuning guide 。
|spark.kryoserializer.buffer.max | 64m | 最大允许的Kryo序列化buffer。必须必你所需要序列化的对象要大。如果你在Kryo中看到”buffer limit exceeded”这个异常，你就得增加这个值了。
|spark.kryoserializer.buffer | 64k | Kryo序列化的初始buffer大小。注意，每台worker上对应每个core会有一个buffer。buffer最大增长到 spark.kryoserializer.buffer.max
|spark.rdd.compress | false | 是否压缩序列化后RDD的分区（如：StorageLevel.MEMORY_ONLY_SER）。能节省大量空间，但多消耗一些CPU。
|spark.serializer | org.apache.spark.serializer.JavaSerializer (org.apache.spark.serializer.KryoSerializer when using Spark SQL Thrift Server) | 用于序列化对象的类，序列化后的数据将通过网络传输，或从缓存中反序列化回来。默认的Java序列化使用java的Serializable接口，但速度较慢，所以我们建议使用usingorg.apache.spark.serializer.KryoSerializer and configuring Kryo serialization如果速度需要保证的话。当然你可以自定义一个序列化器，通过继承org.apache.spark.Serializer.
|spark.serializer.objectStreamReset | 100 | 如果使用org.apache.spark.serializer.JavaSerializer做序列化器，序列化器缓存这些对象，以避免输出多余数据，然而，这个会打断垃圾回收。通过调用reset来flush序列化器，从而使老对象被回收。要禁用这一周期性reset，需要把这个参数设为-1，。默认情况下，序列化器会每过100个对象，被reset一次。

### 内存管理

| 属性名称 | 默认值 | 含义 
| - | - | - |
|spark.memory.fraction | 0.75 | 堆内存中用于执行、混洗和存储（缓存）的比例。这个值越低，则执行中溢出到磁盘越频繁，同时缓存被逐出内存也更频繁。这个配置的目的，是为了留出用户自定义数据结构、内部元数据使用的内存。推荐使用默认值。请参考this description.
|spark.memory.storageFraction | 0.5 | 不会被逐出内存的总量，表示一个相对于 spark.memory.fraction的比例。这个越高，那么执行混洗等操作用的内存就越少，从而溢出磁盘就越频繁。推荐使用默认值。更详细请参考 this description.
|spark.memory.offHeap.enabled | true | 如果true，Spark会尝试使用堆外内存。启用 后，spark.memory.offHeap.size必须为正数。
|spark.memory.offHeap.size | 0 | 堆外内存分配的大小（绝对值）。这个设置不会影响堆内存的使用，所以你的执行器总内存必须适应JVM的堆内存大小。必须要设为正数。并且前提是 spark.memory.offHeap.enabled=true.
|spark.memory.useLegacyMode | false | 是否使用老式的内存管理模式（1.5以及之前）。老模式在堆内存管理上更死板，使用固定划分的区域做不同功能，潜在的会导致过多的数据溢出到磁盘（如果不小心调整性能）。必须启用本参数，以下选项才可用：spark.shuffle.memoryFractionspark.storage.memoryFractionspark.storage.unrollFraction
|spark.shuffle.memoryFraction | 0.2 | （废弃）必须先启用spark.memory.useLegacyMode这个才有用。混洗阶段用于聚合和协同分组的JVM堆内存比例。在任何指定的时间，所有用于混洗的内存总和不会超过这个上限，超出的部分会溢出到磁盘上。如果溢出台频繁，考虑增加spark.storage.memoryFraction的大小。
|spark.storage.memoryFraction | 0.6 | （废弃）必须先启用spark.memory.useLegacyMode这个才有用。Spark用于缓存数据的对内存比例。这个值不应该比JVM 老生代（old generation）对象所占用的内存大，默认是60%的堆内存，当然你可以增加这个值，同时配置你所用的老生代对象占用内存大小。
|spark.storage.unrollFraction | 0.2 | （废弃）必须先启用spark.memory.useLegacyMode这个才有用。Spark块展开的内存占用比例。如果没有足够的内存来完整展开新的块，那么老的块将被抛弃。







