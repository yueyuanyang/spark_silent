
# Spark-Redis

可以从 [Redis](http://redis.io) with [Apache Spark](http://spark.apache.org/)这里查看读、 写操作


Spark-Redis提供了所有Redis数据结构的访问权限——字 符串，哈希，列表，集合和排序集合,读取数据作为Spark的RDD。 该库可以与Redis独立以及集群数据库一起使用。 与Redis集群一起使用时，Spark-Redis知道其分区方案，并根据重新分片和节点故障事件进行调整。

Spark-Redis 也支持 Spark-Streaming.

## Minimal requirements

你需要在如下条件下使用 Spark-Redis:

 - Apache Spark v1.4.0
 - Scala v2.10.4
 - Jedis v2.7
 - Redis v2.8.12 or v3.0.3

## Known limitations

* Java, Python and R API 暂时不支持
* 本包知识在如下条件下测试:
 - Apache Spark v1.4.0
 - Scala v2.10.4
 - Jedis v2.7 and v2.8 pre-release (see [below](#jedis-and-read-only-redis-cluster-slave-nodes) for details)
 - Redis v2.8.12 and v3.0.3

## Additional considerations

该库正在进行中，因此API可能会在官方发布之前更改。

## Getting the library

您可以下载官方的资源并构建它：

```
git clone https://github.com/RedisLabs/spark-redis.git
cd spark-redis
mvn clean package -DskipTests
```

### Jedis and read-only Redis cluster slave nodes

Jedis的当前版本 - v2.7 - 不支持从Redis集群的从节点读取数据。 该功能仅包含在其即将发布的版本v2.8中。

要将Spark-Redis与Redis群集的从属节点一起使用，该库的源包括在“with-slaves”分支下预发布Jedis v2.8。 在运行`mvn clean install`之前输入以下内容切换到该分支：

```
git checkout with-slaves
```

## Using the library

使用`--jars`命令行选项将Spark-Redis添加到Spark。 例如，从spark-shell中使用它，以下列方式包含它：

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

以下部分包含演示Spark-Redis使用的代码片段。 要使用示例代码，您需要分别将`your.redis.server`和`6379`替换为您的Redis数据库的IP地址或主机名和端口。

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

支持的配置keys包括：

* `redis.host` - 主机或我们连接到的初始节点的IP。 连接器将读取群集拓扑结构从初始节点开始，因此不需要提供其余的集群节点。
* `redis.port` -  初始节点TCP redis端口。
* `redis.auth` - 初始节点 AUTH password。
* `redis.db` -  可选的database库号。特别是在集群模式下,避免使用这个。

### The keys RDD

由于Redis中的数据访问基于key，因此要使用Spark-Redis，首先需要 key RDD。 以下示例显示如何将Redis中的键名读入RDD中：

```
import com.redislabs.provider.redis._

val keysRDD = sc.fromRedisKeyPattern("foo*", 5)
val keysRDD = sc.fromRedisKeys(Array("foo", "bar"), 5)

```

上面的示例通过从Redis中检索出与给定模式匹配的key(`foo *`)或者可以由数组组成的key匹配的值,返回RDD形式。 此外,设置的新值为5,覆盖了RDD中3个分区的默认设置, 每个分区由一组包含匹配的关键字名称Redis集群哈希槽组成。

### Reading data

Each of Redis' data types can be read to an RDD. The following snippet demonstrates reading Redis Strings.

#### Strings

```
import com.redislabs.provider.redis._
val stringRDD = sc.fromRedisKV("keyPattern*")
val stringRDD = sc.fromRedisKV(Array("foo", "bar"))
```

Once run, `stringRDD: RDD[(String, String)]` will contain the string values of all keys whose names are provided by keyPattern or Array[String].

#### Hashes
```
val hashRDD = sc.fromRedisHash("keyPattern*")
val hashRDD = sc.fromRedisHash(Array("foo", "bar"))
```

This will populate `hashRDD: RDD[(String, String)]` with the fields and values of the Redis Hashes, the hashes' names are provided by keyPattern or Array[String]

#### Lists
```
val listRDD = sc.fromRedisList("keyPattern*")
val listRDD = sc.fromRedisList(Array("foo", "bar"))
```
The contents (members) of the Redis Lists in whose names are provided by keyPattern or Array[String] will be stored in `listRDD: RDD[String]`

#### Sets
```
val setRDD = sc.fromRedisSet("keyPattern*")
val setRDD = sc.fromRedisSet(Array("foo", "bar"))
```

The Redis Sets' members will be written to `setRDD: RDD[String]`.

#### Sorted Sets
```
val zsetRDD = sc.fromRedisZSetWithScore("keyPattern*")
val zsetRDD = sc.fromRedisZSetWithScore(Array("foo", "bar"))
```

Using `fromRedisZSetWithScore` will store in `zsetRDD: RDD[(String, Double)]`, an RDD that consists of members and their scores, from the Redis Sorted Sets whose keys are provided by keyPattern or Array[String].

```
val zsetRDD = sc.fromRedisZSet("keyPattern*")
val zsetRDD = sc.fromRedisZSet(Array("foo", "bar"))
```

Using `fromRedisZSet` will store in `zsetRDD: RDD[String]`, an RDD that consists of members, from the Redis Sorted Sets whose keys are provided by keyPattern or Array[String].

```
val startPos: Int = _
val endPos: Int = _
val zsetRDD = sc.fromRedisZRangeWithScore("keyPattern*", startPos, endPos)
val zsetRDD = sc.fromRedisZRangeWithScore(Array("foo", "bar"), startPos, endPos)
```

Using `fromRedisZRangeWithScore` will store in `zsetRDD: RDD[(String, Double)]`, an RDD that consists of members and the members' ranges are within [startPos, endPos] of its own Sorted Set, from the Redis Sorted Sets whose keys are provided by keyPattern or Array[String].

```
val startPos: Int = _
val endPos: Int = _
val zsetRDD = sc.fromRedisZRange("keyPattern*", startPos, endPos)
val zsetRDD = sc.fromRedisZRange(Array("foo", "bar"), startPos, endPos)
```

Using `fromRedisZSet` will store in `zsetRDD: RDD[String]`, an RDD that consists of members and the members' ranges are within [startPos, endPos] of its own Sorted Set, from the Redis Sorted Sets whose keys are provided by keyPattern or Array[String].

```
val min: Double = _
val max: Double = _
val zsetRDD = sc.fromRedisZRangeByScoreWithScore("keyPattern*", min, max)
val zsetRDD = sc.fromRedisZRangeByScoreWithScore(Array("foo", "bar"), min, max)
```

Using `fromRedisZRangeByScoreWithScore` will store in `zsetRDD: RDD[(String, Double)]`, an RDD that consists of members and the members' scores are within [min, max], from the Redis Sorted Sets whose keys are provided by keyPattern or Array[String].

```
val min: Double = _
val max: Double = _
val zsetRDD = sc.fromRedisZRangeByScore("keyPattern*", min, max)
val zsetRDD = sc.fromRedisZRangeByScore(Array("foo", "bar"), min, max)
```

Using `fromRedisZSet` will store in `zsetRDD: RDD[String]`, an RDD that consists of members and the members' scores are within [min, max], from the Redis Sorted Sets whose keys are provided by keyPattern or Array[String].

### Writing data
To write data from Spark to Redis, you'll need to prepare the appropriate RDD depending on the data type you want to use for storing the data in it.

#### Strings
For String values, your RDD should consist of the key-value pairs that are to be written. Assuming that the strings RDD is called `stringRDD`, use the following snippet for writing it to Redis:

```
...
sc.toRedisKV(stringRDD)
```

#### Hashes
To store a Redis Hash, the RDD should consist of its field-value pairs. If the RDD is called `hashRDD`, the following should be used for storing it in the key name specified by `hashName`:

```
...
sc.toRedisHASH(hashRDD, hashName)
```

#### Lists
Use the following to store an RDD in a Redis List:

```
...
sc.toRedisLIST(listRDD, listName)
```

Use the following to store an RDD in a fixed-size Redis List:

```
...
sc.toRedisFixedLIST(listRDD, listName, listSize)
```

The `listRDD` is an RDD that contains all of the list's string elements in order, and `listName` is the list's key name.
`listSize` is an integer which specifies the size of the redis list; it is optional, and will default to an unlimited size.


#### Sets
For storing data in a Redis Set, use `toRedisSET` as follows:

```
...
sc.toRedisSET(setRDD, setName)
```

Where `setRDD` is an RDD with the set's string elements and `setName` is the name of the key for that set.

#### Sorted Sets
```
...
sc.toRedisZSET(zsetRDD, zsetName)
```

The above example demonstrates storing data in Redis in a Sorted Set. The `zsetRDD` in the example should contain pairs of members and their scores, whereas `zsetName` is the name for that key.

### Streaming
Spark-Redis support streaming data from Redis instance/cluster, currently streaming data are fetched from Redis' List by the `blpop` command. Users are required to provide an array which stores all the List names they are interested in. The [storageLevel](http://spark.apache.org/docs/latest/streaming-programming-guide.html#data-serialization) is `MEMORY_AND_DISK_SER_2` by default, you can change it on your demand.
`createRedisStream` will create a `(listName, value)` stream, but if you don't care about which list feeds the value, you can use `createRedisStreamWithoutListname` to get the only `value` stream.

Use the following to get a `(listName, value)` stream from `foo` and `bar` list

```
import org.apache.spark.streaming.{Seconds, StreamingContext}
import org.apache.spark.storage.StorageLevel
import com.redislabs.provider.redis._
val ssc = new StreamingContext(sc, Seconds(1))
val redisStream = ssc.createRedisStream(Array("foo", "bar"), storageLevel = StorageLevel.MEMORY_AND_DISK_2)
redisStream.print
ssc.awaitTermination()
```


Use the following to get a `value` stream from `foo` and `bar` list

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
If you want to use multiple redis clusters/instances, an implicit RedisConfig can be used in a code block to specify the target cluster/instance in that block.

## Contributing

You're encouraged to contribute to the open source Spark-Redis project. There are two ways you can do so.

### Issues

If you encounter an issue while using the Spark-Redis library, please report it at the project's [issues tracker](https://github.com/RedisLabs/spark-redis/issues).

### Pull request

Code contributions to the Spark-Redis project can be made using [pull requests](https://github.com/RedisLabs/spark-redis/pulls). To submit a pull request:

 1. Fork this project.
 2. Make and commit your changes.
 3. Submit your changes as a pull request.
