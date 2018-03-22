## Spark优雅的操作Redis

Spark的优势在于内存计算，然而在计算中难免会用到一些元数据或中间数据，有的存在关系型数据库中，有的存在HDFS上，有的存在HBase中，但其读写速度都和Spark计算的速度相差甚远，而Redis基于内存的读写则可以完美解决此类问题，下面介绍Spark如何与Redis交互。
在Spark计算的时候如何加载Redis中的数据，其实官方有现成的包和文档，文档是全英文，好在东西不多，下面介绍如何使用。

首先把jar包引入工程，在maven上居然找不到这个包。。。所以使用Maven和SBT的同学自行解决。
[下载地址](https://spark-packages.org/package/RedisLabs/spark-redis)(打不开的同学可以尝试翻墙)

可以看出提供的功能还是挺全面的，有单独的redis分区，redisRDD，SQLAPI以及StreamingAPI

下面我们一点一点来做一个示例：

## 现在我们启动SparkContext

```
先引入Redis相关的隐试转换
import com.redislabs.provider.redis._

//这里直接使用yarn-cluster模式
val conf = new SparkConf().setMaster("yarn-cluster").setAppName("sparkRedisTest")
conf.set("redis.host", "10.1.11.70")    //host,随便一个节点，自动发现
conf.set("redis.port", "6379")  //端口号，不填默认为6379
//conf.set("redis.auth","null")  //用户权限配置
//conf.set("redis.db","0")  //数据库设置
//conf.set("redis.timeout","2000")  //设置连接超时时间
val sc = new SparkContext(conf)

```
之后可以看到IDEA给出的提示，sc通过导入的饮食转换可以调出的读取Redis的方法，都是以fromRedis开头的，都是redis可以存储的数据结构，这里以常见的KV进行示例

**还是先扒一下源码看看：**
```
def fromRedisKV[T](keysOrKeyPattern: T,
                     partitionNum: Int = 3)
                    (implicit redisConfig: RedisConfig = new RedisConfig(new RedisEndpoint(sc.getConf))):
  RDD[(String, String)] = {
    keysOrKeyPattern match {
      case keyPattern: String => fromRedisKeyPattern(keyPattern, partitionNum)(redisConfig).getKV
      case keys: Array[String] => fromRedisKeys(keys, partitionNum)(redisConfig).getKV
      case _ => throw new scala.Exception("KeysOrKeyPattern should be String or Array[String]")
    }
  }
  
```

**先看传入的参数：**

- 泛型类型keysOrKeyPattern
从的模式匹配代码中可以看出，这里的T可是是两种类型，一个是String，另一个是Array[String],如果传入其他类型则会抛出运行时异常，其中String类型的意思是匹配键，这里可以用通配符比如foo*，所以返回值是一个结果集RDD[(String, String)]，当参数类型为Array[String]时是指传入key的数组，返回的结果则为相应的的结果集，RDD的内容类型也是KV形式。
- Int类型partitionNum
生成RDD的分区数，默认为3，如果传入的第一个参数类型是Array[String]，这个参数可以这样设置，先预估一下返回结果集的大小，使用keyArr.length / num + 1，这样则保证分区的合理性，以防发生数据倾斜。若第一个参数类型为String，能预估尽量预估，如果实在没办法，比如确实在这里发生了数据倾斜，可以尝试考虑使用sc.fromRedisKeys()返回key的集合，提前把握返回结果集的大小，或者根据集群机器数量，把握分区数。
- 柯里化形式隐式参数redisConfig
由于我们之前在sparkConf里面set了相应的参数，这里不传入这个参数即可。如要调整，则可以按照源码中的方式传入，其中RedisEndpoint是一个case class类，而且很多参数都有默认值（比如6379的端口号），所以自己建立一个RedisEndpoint也是非常方便的。

打包之后运行，命令为：
```
spark-submit \
--master yarn \
--deploy-mode cluster  \
--class test.SparkRedis \
--jars jedis-2.9.0.jar,spark-redis-0.3.2.jar,/opt/cloudera/parcels/CDH/jars/commons-pool2-2.2.jar  \
--driver-class-path jedis-2.9.0.jar,spark-redis-0.3.2.jar,/opt/cloudera/parcels/CDH/jars/commons-pool2-2.2.jar spark-redis.jar

```

命令中指明了依赖的资源包:jedis-2.9.0.jar,spark-redis-0.3.2.jar,commons-pool2-2.2.jar其中commons-pool2-2.2.jar是spark-redis依赖的包，如果集群环境为CDH发行版，可在/opt/cloudera/parcels/CDH/jars/commons-pool2-2.2.jar下找到该包，，而且yarn的运行环境里面没有默认引入该包；如果为自建环境，则需要自行下载该包，Maven上搜索commons-pool2即可。

### 如果传入的是key数组

```
import org.apache.spark.{SparkConf, SparkContext}
import com.redislabs.provider.redis._

object SparkRedis extends App {
  val conf = new SparkConf().setMaster("yarn-cluster").setAppName("sparkRedisTest")
  conf.set("redis.host", "10.1.11.70")
  val sc = new SparkContext(conf)
  val keys = Array[String]("high", "abc", "together")
  sc.fromRedisKV(keys).coalesce(1).saveAsTextFile("hdfs://nameservice1/spark/test/redisResult2")
}

```

### k-v 操作

```
import org.apache.spark.{SparkConf, SparkContext}
import com.redislabs.provider.redis._
import org.apache.spark.rdd.RDD

object SparkRedis extends App {
  val conf = new SparkConf().setMaster("yarn-cluster").setAppName("sparkRedisTest")
  conf.set("redis.host", "10.1.11.70")
  val sc = new SparkContext(conf)
  val data = Seq[(String,String)](("high","111"), ("abc","222"), ("together","333"))
  val redisData:RDD[(String,String)] = sc.parallelize(data)
  sc.toRedisKV(redisData)
}

```

