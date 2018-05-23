## 使用Apache Spark将数据写入ElasticSearch

ElasticSearch是一个基于Lucene的搜索服务器。它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口。Elasticsearch是用Java开发的，并作为Apache许可条款下的开放源码发布，是当前流行的企业级搜索引擎。设计用于云计算中，能够达到实时搜索，稳定，可靠，快速，安装使用方便。

直接介绍如何使用Apache Spark将数据写入到ElasticSearch中。本文使用的是类库是elasticsearch-spark-20_2.11，其从2.1版本开始提供了内置支持Apache Spark的功能，在使用elasticsearch-hadoop之前，本文elasticsearc 2.1.0 和 spark 2.x,我们需要引入依赖：
```
<dependency>
    <groupId>org.elasticsearch</groupId>
    <artifactId>elasticsearch-spark-20_2.11</artifactId>
    <version>5.2.0</version>
</dependency>
```
### 文章目录

1. 将Map对象写入ElasticSearch
2. 将case class对象写入ElasticSearch
3. 将Json字符串写入ElasticSearch
4. 动态设置插入的type
5. 自定义id
6. 自定义记录的元数据

如果你直接将代码写入文件，那么你可以在初始化SparkContext之前设置好ElasticSearch相关的参数，如下：
```
import org.apache.spark.sql.SparkSession

val spark = SparkSession.builder().appName("Words Segmentation")
              .config("es.index.auto.create", "true")
              .config("es.nodes", url)
              .config("es.port", "9200")
              .getOrCreate()
```

**将Map对象写入ElasticSearch**

```
import org.elasticsearch.spark.rdd.EsSpark
 
val numbers = Map("one" -> 1, "two" -> 2, "three" -> 3)
val airports = Map("OTP" -> "Otopeni", "SFO" -> "San Fran")

sc.makeRDD(Seq(numbers, airports)).saveToEs("book/docs")

```
上面构建了两个Map对象，然后将它们写入到ElasticSearch中；其中saveToEs里面参数的book表示索引(indexes)，而docs表示type。

**将case class对象写入ElasticSearch**

我们还可以将Scala中的case class对象写入到ElasticSearch；Java中可以写入JavaBean对象，如下：
```
import org.elasticsearch.spark.rdd.EsSpark
case class Trip(departure: String, arrival: String) 

val upcomingTrip = Trip("OTP", "SFO")
val lastWeekTrip = Trip("MUC", "OTP")
 
val rdd = sc.makeRDD(Seq(upcomingTrip, lastWeekTrip))  
rdd.saveToEs("book/class")

```

上面的代码片段将upcomingTrip和lastWeekTrip写入到名为book的_index中，type是class。上面都是通过隐式转换才使得rdd拥有saveToEs方法

还提供显式方法来把RDD写入到ElasticSearch中，如下：
```
import org.elasticsearch.spark.rdd.EsSpark 

val rdd = sc.makeRDD(Seq(upcomingTrip, lastWeekTrip)) 

EsSpark.saveToEs(rdd, "spark/docs")

```
**将Json字符串写入ElasticSearch**

我们可以直接将Json字符串写入到ElasticSearch中，如下:

```
import org.elasticsearch.spark.rdd.EsSpark 

val json1 = """{"id" : 1, "blog" : "www.iteblog.com", "weixin" : "iteblog_hadoop"}"""
val json2 = """{"id" : 2, "blog" : "books.iteblog.com", "weixin" : "iteblog_hadoop"}"""

val rdd = sc.makeRDD(Seq(json1, json2))
EsSpark.saveJsonToEs(rdd,"book/json")

```

**动态设置插入的type**

上面的示例都是将写入的type写死。有很多场景下同一个Job中有很多类型的数据，我们希望一次就可以将不同的数据写入到不同的type中，比如属于book的信息全部写入到type为book里面；而属于cd的信息全部写入到type为cd里面。很高兴的是elasticsearch-hadoop为我们提供了这个功能，如下：

```
import org.elasticsearch.spark.rdd.EsSpark 

val game = Map("media_type"->"game","title" -> "FF VI","year" -> "1994")
val book = Map("media_type" -> "book","title" -> "Harry Potter","year" -> "2010")
 
val cd = Map("media_type" -> "music","title" -> "Surfing With The Alien")
val rdd = sc.makeRDD(Seq(game, book, cd))

EsSpark.saveToEs(rdd,"title/{media_type}")

```
type是通过{media_type}通配符设置的，这个在写入的时候可以获取到，然后将不同类型的数据写入到不同的type中。

**自定义id**

在ElasticSearch中，`_index/_type/_id`的组合可以唯一确定一个Document。如果我们不指定id的话，ElasticSearch将会自动为我们生产全局唯一的id，自动生成的ID有20个字符长如下：

```
{
    "_index": "iteblog", 
    "_type": "docs", 
    "_id": "AVZy3d5sJfxPRwCjtWM-", 
    "_score": 1, 
    "_source": {
        "arrival": "Otopeni", 
        "SFO": "San Fran"
    }
}
```

很显然，这么长的字符串没啥意义，而且也不便于我们记忆使用。不过我们可以在插入数据的时候手动指定id的值，如下：

```
val otp = Map("iata" -> "OTP", "name" -> "Otopeni")
val muc = Map("iata" -> "MUC", "name" -> "Munich")
val sfo = Map("iata" -> "SFO", "name" -> "San Fran")
val airportsRDD = sc.makeRDD(Seq((1, otp), (2, muc), (3, sfo))) 
 
airportsRDD.saveToEsWithMeta("book/2015")

```
上面的Seq((1, otp), (2, muc), (3, sfo))语句指定为各个对象指定了id值，分别为1、2、3。然后你可以通过/iteblog/2015/1 URL搜索到otp对象的值。我们还可以如下方式指定id：

```
val json1 = """{"id" : 1, "blog" : "www.iteblog.com", "weixin" : "iteblog_hadoop"}"""
val json2 = """{"id" : 2, "blog" : "books.iteblog.com", "weixin" : "iteblog_hadoop"}"""

val rdd = sc.makeRDD(Seq(json1, json2))
 
EsSpark.saveToEs(rdd, "book/docs", Map("es.mapping.id" -> "id"))
```
上面通过es.mapping.id参数将对象中的id字段映射为每条记录的id。

**自定义记录的元数据**

自定义记录的元数据
```
import org.elasticsearch.spark.rdd.Metadata._         

val otp = Map("iata" -> "OTP", "name" -> "Otopeni")
val muc = Map("iata" -> "MUC", "name" -> "Munich")
val sfo = Map("iata" -> "SFO", "name" -> "San Fran")

val otpMeta = Map(ID -> 1, TTL -> "3h")  
val mucMeta = Map(ID -> 2, VERSION -> "23")
 
val sfoMeta = Map(ID -> 3) 
val airportsRDD = sc.makeRDD(Seq((otpMeta, otp), (mucMeta, muc), (sfoMeta, sfo)))
 
airportsRDD.saveToEsWithMeta("iteblog/2015")

```
上面代码片段分别为otp、muc和sfo设置了不同的元数据，这在很多场景下是非常有用的。




