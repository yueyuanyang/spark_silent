
# Spark-Redis

可以从 [Redis](http://redis.io) with [Apache Spark](http://spark.apache.org/)这里查看读、 写操作


Spark-Redis提供了所有Redis数据结构的访问权限——字 符串，哈希，列表，集合和排序集合,读取数据转为Spark的RDD。 该库可以与Redis独立以及集群数据库一起使用。当与Redis集群一起使用时，Spark-Redis知道其分区方案，并根据重新分片和节点故障事件进行调整。

Spark-Redis 也支持 Spark-Streaming.

## Minimal requirements

使用 Spark-Redis 基本环境:

 - Apache Spark v1.4.0
 - Scala v2.10.4
 - Jedis v2.7
 - Redis v2.8.12 or v3.0.3

## Known limitations

* Java, Python and R API 暂时不支持

* 测试在下面的环境下:

 - Apache Spark v1.4.0
 - Scala v2.10.4
 - Jedis v2.7 and v2.8 pre-release (see [below](#jedis-and-read-only-redis-cluster-slave-nodes) for details)
 - Redis v2.8.12 and v3.0.3

## Additional considerations

该库正仍在进行中，因此API可能会在官方发布之前更改。

## Getting the library

您可以从官方下载资源并构建它：

```
git clone https://github.com/RedisLabs/spark-redis.git
cd spark-redis
mvn clean package -DskipTests
```

### Jedis and read-only Redis cluster slave nodes

Jedis的当前版本- v2.7 - 不支持从Redis集群的从节点读取数据。 该功能仅包含在其即将发布的版本v2.8中。

要将Spark-Redis与Redis群集的从属节点一起使用，该库的源包括在 “with-slaves” 分支下预发布Jedis v2.8。 在运行`mvn clean install`之前输入以下内容切换到该分支：

```
git checkout with-slaves
```

## Using the library

使用`--jars`命令行选项将Spark-Redis添加到Spark。 例如，以下列方式从spark-shell中使用它：

```
$ bin/spark-shell --jars <path-to>/spark-redis-<version>.jar,<path-to>/jedis-<version>.jar

Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 1.4.0
      /_/

Using Scala version 2.10.4 (OpenJDK 64-Bit Server VM, Java 1.7.0_79)
...
```

以下部分包含演示Spark-Redis使用的代码块。 要使用示例代码，您需要分别将`your.redis.server`和`6379`替换为您的Redis数据库的IP地址或主机名和端口。

### Configuring Connections to Redis using SparkConf

下面是带有redis配置的SparkContext的示例配置：

```scala

import com.redislabs.provider.redis._

...

sc = new SparkContext(new SparkConf()
      .setMaster("local")
      .setAppName("myApp")

      // initial redis host - can be any node in cluster mode
      .set("redis.host", "localhost")

      // initial redis port
      .set("redis.port", "6379")

      // optional redis AUTH password
      .set("redis.auth", "")
  )
```

支持的配置k-v参数包括：

* `redis.host` - 主机或我们连接到的初始节点的IP。 连接器将读取群集拓扑结构从初始节点开始，因此不需要提供其余的集群节点。
* `redis.port` -  初始节点TCP redis端口。
* `redis.auth` - 初始节点 AUTH password。
* `redis.db` -  可选的database库号。特别是在集群模式下,避免使用这个。

### The keys RDD

由于Redis中的数据访问都是基于key值,因此要使用Spark-Redis，首先需要 key RDD。 以下示例显示如何利用 key将Redis中的值读入RDD中：

```
import com.redislabs.provider.redis._

val keysRDD = sc.fromRedisKeyPattern("foo*", 5)
val keysRDD = sc.fromRedisKeys(Array("foo", "bar"), 5)

```

上面的示例通过 RDD 从Redis中检索出数据，返回RDD形式，key值形式包括：
- 模式匹配的key(`foo *`) 
- 数组形式Array("foo", "bar")。 

此外，后面的参数为RDD分区数，本例中设置为5.其中,RDD默认设置为3, 每个分区由一组包含匹配的关键字名称Redis集群哈希槽组成。

### Reading data

每个Redis的数据类型都可以读取成 RDD形式。 

以下代码片段演示了如何阅读Redis字符串。

#### Strings

```
import com.redislabs.provider.redis._
val stringRDD = sc.fromRedisKV("keyPattern*")
val stringRDD = sc.fromRedisKV(Array("foo", "bar"))
```

一旦运行，`字符串RDD：RDD [(String，String)]`将包含名称由键Pattern或Array [String]提供的所有key对应的字符串值。

#### Hashes

```
val hashRDD = sc.fromRedisHash("keyPattern*")
val hashRDD = sc.fromRedisHash(Array("foo", "bar"))
```

这将使用Redis哈希的key和value,形成 `hashRDD：RDD [(String，String)]`,哈希的名称由keyPattern或Array [String]

#### Lists
```
val listRDD = sc.fromRedisList("keyPattern*")
val listRDD = sc.fromRedisList(Array("foo", "bar"))
```
redis列表中其名称由keyPattern或Array [String]提供的内容（成员）将存储在 `listRDD：RDD [String]`

#### Sets
```
val setRDD = sc.fromRedisSet("keyPattern*")
val setRDD = sc.fromRedisSet(Array("foo", "bar"))
```

Redis集合的成员将被写入 `setRDD：RDD [String]`。

#### Sorted Sets

```
val zsetRDD = sc.fromRedisZSetWithScore("keyPattern*")
val zsetRDD = sc.fromRedisZSetWithScore(Array("foo", "bar"))
```

使用`fromRedisZSetWithScore`将从Redis Sorted集合中存储为 `zsetRDD：RDD [(String，Double)]`，一个由member及其scores组成的RDD，其中键由keyPattern或Array [String]提供。

```
val zsetRDD = sc.fromRedisZSet("keyPattern*")
val zsetRDD = sc.fromRedisZSet(Array("foo", "bar"))
```

使用`fromRedisZSet`将存储在`zsetRDD：RDD [String]`中，一个由member构成的RDD,来自Redis Sorted集，其键由KeyPattern或Array [String]提供。

```
val startPos: Int = _
val endPos: Int = _
val zsetRDD = sc.fromRedisZRangeWithScore("keyPattern*", startPos, endPos)
val zsetRDD = sc.fromRedisZRangeWithScore(Array("foo", "bar"), startPos, endPos)
```

使用`fromRedisZRangeWithScore`将存储在`zsetRDD：RDD [(String,Double)]`中，在其自己分类集的[startPos，endPos]范围内的member构成的RDD和member的构成.其中,Redis Sorted Sets 键由keyPattern或Array [String]。

```
val startPos: Int = _
val endPos: Int = _
val zsetRDD = sc.fromRedisZRange("keyPattern*", startPos, endPos)
val zsetRDD = sc.fromRedisZRange(Array("foo", "bar"), startPos, endPos)
```

使用`fromRedisZSet`将存储在`zsetRDD：RDD [String]`中，一个由member构成的RDD和member的范围位于其自己分类集的[startPos，endPos]之内，来自Redis分类集 keyPattern或Array [String]。

```
val min: Double = _
val max: Double = _
val zsetRDD = sc.fromRedisZRangeByScoreWithScore("keyPattern*", min, max)
val zsetRDD = sc.fromRedisZRangeByScoreWithScore(Array("foo", "bar"), min, max)
```

使用`fromRedisZRangeByScoreWithScore`将存储在`zsetRDD：RDD [（String，Double）]`中，一个由member组成且member的分数在[min，max]内的RDD，来自Redis Sorted集合，其键由keyPattern 或Array [String]。

```
val min: Double = _
val max: Double = _
val zsetRDD = sc.fromRedisZRangeByScore("keyPattern*", min, max)
val zsetRDD = sc.fromRedisZRangeByScore(Array("foo", "bar"), min, max)
```

使用`fromRedisZSet`将存储在`zsetRDD：RDD [String]`中，一个由member组成并且member的分数在[min，max]之内的RDD，来自Redis Sorted Sets，其键由KeyPattern或Array [String]。

### Writing data

要将Spark中的数据写入Redis，您需要根据要用于存储数据的数据类型准备适当的RDD。

#### Strings

对于字符串值，您的RDD应该包含要写入的键值对。 假设字符串RDD被称为`stringRDD`，请使用以下代码片段将其写入Redis：

```
...
sc.toRedisKV(stringRDD)
```

#### Hashes

要存储Redis哈希，RDD应该由其字段值对组成。 如果RDD被称为`hashRDD`，则应使用以下内容将其存储在由`hashName`指定的key名称中：

```
...
sc.toRedisHASH(hashRDD, hashName)
```

#### Lists

使用以下命令将RDD存储在Redis列表中：

```
...
sc.toRedisLIST(listRDD, listName)
```

使用以下命令将RDD存储在固定大小的Redis列表中：

```
...
sc.toRedisFixedLIST(listRDD, listName, listSize)
```

listRDD是一个RDD，它按顺序包含所有列表的字符串元素，而listName是列表的键名。`listSize`是一个整数，它指定了redis列表的大小; 它是可选的，并将默认为无限大小。


#### Sets

要将数据存储在Redis集中，请按如下所示使用`toRedisSET`：

```
...
sc.toRedisSET(setRDD, setName)
```

其中`setRDD`是一个RDD，集合的字符串元素和`setName`是该集合的键名。

#### Sorted Sets
```
...
sc.toRedisZSET(zsetRDD, zsetName)
```

上面的例子演示了如何在Redis中将数据存储在Sorted Set中。 示例中的`zsetRDD`应该包含成员对和它们的分数，而`zsetName`是该键的名称。

### Streaming

Spark-Redis支持来自 Redis 实例/集群的流式数据，目前流式数据是通过 `blpop` 命令从 Redis'List 中获取的。 用户需要提供一个数组来存储他们关心的所有List名称。 默认情况下存储方式为[storageLevel]（http://spark.apache.org/docs/latest/streaming-programming-guide.html#data-serialization）是 `MEMORY_AND_DISK_SER_2`,您可以根据您的需求进行更改。

用法：

- `createRedisStream` 将创建一个 `(listName，value)`流
- 如果你不关心哪个列表提供了这个值,你可以使用 `createRedisStreamWithoutListname` 来获得唯一的 `value` 流

1) 使用以下命令从`foo`和`bar`列表中获取 `(listName，value)` 流

```
import org.apache.spark.streaming.{Seconds, StreamingContext}
import org.apache.spark.storage.StorageLevel
import com.redislabs.provider.redis._
val ssc = new StreamingContext(sc, Seconds(1))
val redisStream = ssc.createRedisStream(Array("foo", "bar"), storageLevel = StorageLevel.MEMORY_AND_DISK_2)
redisStream.print
ssc.awaitTermination()

```

2) 使用以下命令从`foo`和`bar`列表中获取`value`流

```
import org.apache.spark.streaming.{Seconds, StreamingContext}
import org.apache.spark.storage.StorageLevel
import com.redislabs.provider.redis._
val ssc = new StreamingContext(sc, Seconds(1))
val redisStream = ssc.createRedisStreamWithoutListname(Array("foo", "bar"), storageLevel = StorageLevel.MEMORY_AND_DISK_2)
redisStream.print
ssc.awaitTermination()
```


### Connecting to Multiple Redis Clusters/Instances

```
def twoEndpointExample ( sc: SparkContext) = {
  val redisConfig1 = new RedisConfig(new RedisEndpoint("127.0.0.1", 6379, "passwd"))
  val redisConfig2 = new RedisConfig(new RedisEndpoint("127.0.0.1", 7379))
  val rddFromEndpoint1 = {
    //endpoint("127.0.0.1", 6379) as the default connection in this block
    implicit val c = redisConfig1
    sc.fromRedisKV("*")
  }
  val rddFromEndpoint2 = {
    //endpoint("127.0.0.1", 7379) as the default connection in this block
    implicit val c = redisConfig2
    sc.fromRedisKV("*")
  }
}

```
如果您想使用多个Redis集群/实例，则可以在代码块中使用隐式RedisConfig来指定该块中的目标集群/实例。

