## Spark 中的 --files 参数与 ConfigFactory 工厂方法

## 1.1 使用配置库

**配置文件解析利器-Config库**

 Typesafe的Config库，纯Java写成、零外部依赖、代码精简、功能灵活、API友好。支持Java properties、JSON、JSON超集格式HOCON以及环境变量。
 
```
ublic class Configure {
    private final Config config;
 
    public Configure(String confFileName) {
        config = ConfigFactory.load(confFileName);
    }
 
    public Configure() {
        config = ConfigFactory.load();
    }
 
    public String getString(String name) {
        return config.getString(name);
    }
}
```
ConfigFactory.load()会加载配置文件，默认加载classpath下的application.conf,application.json和application.properties文件。当然也可以调用ConfigFactory.load(confFileName)加载指定的配置文件。

配置内容即可以是层级关系，也可以用”.”号分隔写成一行:

```
host{
  ip = 127.0.0.1
  port = 2282
}

```
或者

```
host.ip = 127.0.0.1
host.port = 2282
```
即json格式和properties格式。貌似:

 - *.json只能是json格式
 - *.properties只能是properties格式
 - *.conf可以是两者混合

而且配置文件只能是以上三种后缀名

如果多个config 文件有冲突时，解决方案有:

```
- a.withFallback(b) //a和b合并，如果有相同的key，以a为准 
- a.withOnlyPath(String path) //只取a里的path下的配置
- a.withoutPath(String path) //只取a里出path外的配置

Config firstConfig = ConfigFactory.load("test1.conf");
Config secondConfig = ConfigFactory.load("test2.conf");
 
//a.withFallback(b)  a和b合并，如果有相同的key，以a为准
Config finalConfig = firstConfig.withOnlyPath("host").withFallback(secondConfig);
```

### conf 文件
resources 目录下的文件如下：

```
application.conf              
application.production.conf      
application.local.conf             
log4j.properties              
metrics.properties
```
`ConfigFactory` 工厂方法默认会读取 `resources` 目录下面名为 **application.conf** 的文件：

```
# Spark 相关配置
spark {
  master                   = "local[2]"
  streaming.batch.duration = 5001  // Would normally be `ms` in config but Spark just wants the Long
  eventLog.enabled         = true
  ui.enabled               = true
  ui.port                  = 4040
  metrics.conf             = metrics.properties
  checkpoint.path          = "/tmp/checkpoint/telematics-local"
  stopper.port             = 12345
  spark.cleaner.ttl        = 3600
  spark.cleaner.referenceTracking.cleanCheckpoints = true
}

# Kafka 相关配置
kafka {

  metadata.broker.list = "localhost:9092"
  zookeeper.connect    = "localhost:2181"

  topic.dtcdata {
    name = "dc-diagnostic-report"
    partition.num = 1      
    replication.factor = 1  
  }

  group.id             = "group-rds"
  timeOut              = "3000"
  bufferSize           = "100"
  clientId             = "telematics"
  key.serializer.class = "kafka.serializer.StringEncoder"
  serializer.class     = "com.wm.dtc.pipeline.kafka.SourceDataSerializer"
//  serializer.class     = "kafka.serializer.DefaultEncoder"
}

# MySQL 配置
mysql {
  dataSource.maxLifetime              = 800000
  dataSource.idleTimeout              = 600000
  dataSource.maximumPoolSize          = 10
  dataSource.cachePrepStmts           = true
  dataSource.prepStmtCacheSize        = 250
  dataSource.prepStmtCacheSqlLimit    = 204800
  dataSource.useServerPrepStmts       = true
  dataSource.useLocalSessionState     = true
  dataSource.rewriteBatchedStatements = true
  dataSource.cacheResultSetMetadata   = true
  dataSource.cacheServerConfiguration = true
  dataSource.elideSetAutoCommits      = true
  dataSource.maintainTimeStats        = false

  jdbcUrl="jdbc:mysql://127.0.0.1:6606/wmdtc?useUnicode=true&characterEncoding=utf-8&autoReconnect=true&failOverReadOnly=false&useSSL=false"
  jdbcDriver="com.mysql.jdbc.Driver"
  dataSource.user="root"
  dataSource.password="123456"
}

```
为了验证，我创建了一个 Object 对象：

```
package allinone
import com.typesafe.config.ConfigFactory
import scopt.OptionParser

object SparkFilesArgs extends App  {
  val config = ConfigFactory.load()
  val sparkConf = config.getConfig("spark")
  val sparkMaster = sparkConf.getString("master")
  val sparkDuration = sparkConf.getLong("streaming.batch.duration")
  println(sparkMaster, sparkDuration)
}
```

如果我直接运行就会打印：

> (local[2],5001)

确实是 application.conf 文件中 Spark 的配置。

但是生产环境我们打算使用另外一个配置文件 application.production.conf:

```
spark {
  master = "yarn"
  streaming.batch.duration = 5002
  eventLog.enabled=true
  ui.enabled = true
  ui.port = 4040
  metrics.conf = metrics.properties
  checkpoint.path = "/tmp/telematics"
  stopper.port = 12345
  spark.cleaner.ttl = 3600
  spark.cleaner.referenceTracking.cleanCheckpoints = true

  trajectory.path = "hdfs://CRRCNameservice/road_matching/output/road_match_result"
  city.path = "hdfs://CRRCNameservice/user/root/telematics/data/city.csv"
}

##cassandra相关配置
cassandra {
  keyspace = wmdtc
  cardata.name = can_signal
  trip.name = trip
  latest.name = latest
  latest.interval = 15000

  connection.host = "WMBigdata2,WMBigdata3,WMBigdata4,WMBigdata5,WMBigdata6"
  write.consistency_level = LOCAL_ONE
  read.consistency_level = LOCAL_ONE
  concurrent.writes = 24
  batch.size.bytes = 65536
  batch.grouping.buffer.size = 1000
  connection.keep_alive_ms = 300000
  auth.username = cihon
  auth.password = cihon
}

kafka {
  metadata.broker.list = "WMBigdata2:9092,WMBigdata3:9092,WMBigdata4:9092,WMBigdata5:9092,WMBigdata6:9092"
  zookeeper.connect = "WMBigdata2:2181,WMBigdata3:2181,WMBigdata4:2181"

  topic.obddata {
    name = "wmdtc"
  }

  group.id = "can_signal"
  timeOut = "3000"
  bufferSize = "100"
  clientId = "telematics"

  key.serializer.class = "kafka.serializer.StringEncoder"
  serializer.class = "com.wm.telematics.pipeline.kafka.SourceDataSerializer"

}

akka {
  loglevel = INFO
  stdout-loglevel = WARNING
  loggers = ["akka.event.slf4j.Slf4jLogger"]
}

##geoService接口URL
webservice {
  url = "http://101.201.108.155:8088/map/roadmessage"
}

##geoService相关配置
geoservice {
  timeout = 3
  useRealData = false
}
```

既然 ConfigFactory 方法默认读取 application.conf 文件，但是

> val config = ConfigFactory.load()

相当于：

> val config = ConfigFactory.load("application.conf")

但是 load 方法也接受参数：resourceBasename:

> val config = ConfigFactory.load("application.production") // 加载生产环境的配置

这样在代码里面通过加载不同的配置文件实现本地、测试、生产环境的切换和部署，但是在代码里面读取配置还是不够优美！所以我们有 Spark 的 --files 命令行选项。顾名思义，显而易见，也正如官网所描述的那样, --files 参数后面的值是逗号分割的文本文件, 里面有一个 .conf 文件, load 方法会加载 --files 选项传递过来的配置文件：

```
#!/bin/sh

CONF_DIR=/root/telematics/resources
APP_CONF=application.production.conf
EXECUTOR_JMX_PORT=23339
DRIVER_JMX_PORT=2340

spark-submit \
  --name WM_telematics \
  --class allinone.SparkFilesArgs \
  --master local[*] \
  --deploy-mode client \
  --driver-memory 2g \
  --driver-cores 2 \
  --executor-memory 1g \
  --executor-cores 3 \
  --num-executors 3 \
  --conf "spark.executor.extraJavaOptions=-Dconfig.resource=$APP_CONF -Dcom.sun.management.jmxremote.port=$EXECUTOR_JMX_PORT -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Djava.rmi.server.hostname=`hostname`" \
  --conf "spark.driver.extraJavaOptions=-Dconfig.resource=$APP_CONF -Dcom.sun.management.jmxremote.port=$DRIVER_JMX_PORT -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Djava.rmi.server.hostname=`hostname`" \
  --conf spark.executor.memoryOverhead=4096 \
  --conf spark.driver.memoryOverhead=2048 \
  --conf spark.yarn.maxAppAttempts=2 \
  --conf spark.yarn.submit.waitAppCompletion=false \
  --conf spark.network.timeout=1800s \
  --conf spark.scheduler.executorTaskBlacklistTime=30000 \
  --conf spark.core.connection.ack.wait.timeout=300s \
  --files $CONF_DIR/$APP_CONF,$CONF_DIR/log4j.properties,$CONF_DIR/metrics.properties \
```
它打印：
> (local[*],5002)

因为我在命令行选项中指定了 master 为 local[\*], 配置文件为 application.production.conf。

### 如果报错： resource not found on classpath: application.conf

jar 包里面我把 application.conf 给删除了，用 --files 传参数给 spark-submit 的方式，但是报：在 classpath 下找不到 application.conf 这个文件了。

**cat spark-submit.sh:**
```
#!/bin/sh

CONF_DIR=/Users/ohmycloud/work/cihon/gac/sources
APP_CONF=application.conf
EXECUTOR_JMX_PORT=23333
DRIVER_JMX_PORT=2334

spark-submit \
  --class $1 \
  --master local[2] \
  --deploy-mode client \
  --driver-memory 2g \
  --driver-cores 2 \
  --executor-memory 2g \
  --executor-cores 2 \
  --num-executors 4 \
  --conf "spark.executor.extraJavaOptions=-Dconfig.resource=$APP_CONF -Dcom.sun.management.jmxremote.port=$EXECUTOR_JMX_PORT -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Djava.rmi.server.hostname=`hostname`" \
  --conf "spark.driver.extraJavaOptions=-Dconfig.resource=$APP_CONF -Dcom.sun.management.jmxremote.port=$DRIVER_JMX_PORT -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Djava.rmi.server.hostname=`hostname`" \
  --conf spark.yarn.executor.memoryOverhead=1024 \
  --conf spark.yarn.driver.memoryOverhead=1024 \
  --conf spark.yarn.maxAppAttempts=2 \
  --conf spark.yarn.submit.waitAppCompletion=false \
  --files $CONF_DIR/$APP_CONF \
  /Users/ohmycloud/demo/Spark/WriteParquet2Kafka/target/socket-structured-streaming-1.0-SNAPSHOT.jar
```

原因是 application.conf 文件所在的路径 /Users/ohmycloud/work/cihon/gac/sources 不在 classpath 里面！

使用

> --driver-class-path /Users/ohmycloud/work/cihon/gac/sources 

而非

>  --driver-class-path /Users/ohmycloud/work/cihon/gac/sources/application.conf 

来添加 class path。
```
#!/bin/sh

CONF_DIR=/Users/ohmycloud/work/cihon/gac/sources
APP_CONF=application.conf
EXECUTOR_JMX_PORT=23333
DRIVER_JMX_PORT=2334

spark-submit \
  --class $1 \
  --master local[2] \
  --deploy-mode client \
  --driver-memory 2g \
  --driver-cores 2 \
  --executor-memory 2g \
  --executor-cores 2 \
  --num-executors 4 \
  --conf "spark.executor.extraJavaOptions=-Dconfig.resource=$APP_CONF -Dcom.sun.management.jmxremote.port=$EXECUTOR_JMX_PORT -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Djava.rmi.server.hostname=`hostname`" \
  --conf "spark.driver.extraJavaOptions=-Dconfig.resource=$APP_CONF -Dcom.sun.management.jmxremote.port=$DRIVER_JMX_PORT -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Djava.rmi.server.hostname=`hostname`" \
  --conf spark.yarn.executor.memoryOverhead=1024 \
  --conf spark.yarn.driver.memoryOverhead=1024 \
  --conf spark.yarn.maxAppAttempts=2 \
  --conf spark.yarn.submit.waitAppCompletion=false \
  --driver-class-path /Users/ohmycloud/work/cihon/gac/sources \
  --files $CONF_DIR/$APP_CONF \
  /Users/ohmycloud/demo/Spark/WriteParquet2Kafka/target/socket-structured-streaming-1.0-SNAPSHOT.jar

```

yarn 模式

yarn 模式下，不需要添加 driver-class-path 了:

```
#!/bin/sh

CONF_DIR=/root/resources
APP_CONF=application.test.conf
EXECUTOR_JMX_PORT=23333
DRIVER_JMX_PORT=2334

spark2-submit \
  --class $1 \
  --master yarn \
  --deploy-mode cluster \
  --driver-memory 2g \
  --driver-cores 2 \
  --executor-memory 2g \
  --executor-cores 2 \
  --num-executors 4 \
  --packages org.apache.spark:spark-sql-kafka-0-10_2.11:2.3.0 \
  --conf "spark.executor.extraJavaOptions=-Dconfig.resource=$APP_CONF -Dcom.sun.management.jmxremote.port=$EXECUTOR_JMX_PORT -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Djava.rmi.server.hostname=`hostname`" \
  --conf "spark.driver.extraJavaOptions=-Dconfig.resource=$APP_CONF -Dcom.sun.management.jmxremote.port=$DRIVER_JMX_PORT -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Djava.rmi.server.hostname=`hostname`" \
  --conf spark.executor.memoryOverhead=1024 \
  --conf spark.driver.memoryOverhead=1024 \
  --conf spark.yarn.maxAppAttempts=2 \
  --conf spark.yarn.submit.waitAppCompletion=false \
  --files $CONF_DIR/$APP_CONF,$CONF_DIR/log4j.properties,$CONF_DIR/metrics.properties \
```













