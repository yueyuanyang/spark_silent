### spark杂记之一  ——  如何设置spark日志的打印级别

#### 在集群模式中的代码中打印个别内容
```
Logger.getRootLogger.info(message)

```

目前spark日志打印级别设置有三种方法(**推荐使用第三种**)，如下：

#### 第一种 ： 通过配置文件

```
#log4j.rootLogger=WARN,console
log4j.rootLogger=DEBUG, stdout
# console
log4j.appender.console=org.apache.log4j.ConsoleAppender
log4j.appender.console.encoding=utf-8
log4j.appender.console.layout=org.apache.log4j.PatternLayout
log4j.appender.console.layout.ConversionPattern =%-d{yyyy-MM-dd HH:mm:ss} [ %t:%r ] - [ %p ] %l %m%n

```

#### 第二种：通过代码中设置
```
val sc: SparkContext = new SparkContext(sparkConf)
sc.setLogLevel("WARN")
//sc.setLogLevel("DEBUG")
//sc.setLogLevel("ERROR")
//sc.setLogLevel("INFO")

```

#### 第三种：代码中使用代理设置(推荐使用)
```
import org.apache.log4j.Logger
Logger.getLogger("org.apache.spark").setLevel(Level.ERROR)
Logger.getLogger("org.apache.hadoop").setLevel(Level.ERROR)
Logger.getLogger("org.apache.zookeeper").setLevel(Level.WARN)
Logger.getLogger("org.apache.hive").setLevel(Level.WARN)

```
