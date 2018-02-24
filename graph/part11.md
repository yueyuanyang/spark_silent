## graphFrame 操作
### 什么是GraphFrames

- GraphFrame支持通用的图处理，和Apache Spark的GraphX库很像，除此之外，GraphFrames基于Spark DataFrames构建，从而有以下几个优点。

- Python，Java和Scala API：GraphFrames为三种语言提供了通用的API接口。首次实现了所有在GraphX中实现的算法都能在python和Java中使用。
强力的查询：GraphFrames允许用于使用简短的查询，就像和Spark SQL和DataFrame中强力的查询语句一样。

- 保存和载入图模型：GraphFrames完全支持DataFrame结构的数据源，允许使用熟悉的Parquet、JSON、和CSV读写图。

### maven 包导入
```
    <dependency>
      <groupId>com.typesafe.scala-logging</groupId>
      <artifactId>scala-logging-slf4j_2.11</artifactId>
      <version>2.1.1</version>
    </dependency>
    <dependency>
      <groupId>graphframes</groupId>
      <artifactId>graphframes</artifactId>
      <version>0.5.0-spark2.1-s_2.11</version>
    </dependency>
```
#### 操作实例
```
import org.apache.log4j.{Level, Logger}
import org.apache.spark.sql.SparkSession
import org.graphframes.GraphFrame

object GraphFrameTest {

  def main(args: Array[String]) {

    Logger.getLogger("org.apache.spark").setLevel(Level.WARN)

    val spark = SparkSession.builder()
    .appName("graphFrame")
    .master("local[2]")
    .getOrCreate()

    val vertex = spark.createDataFrame(List(
      ("1","Jacob",48),
      ("2","Jessica",45),
      ("3","Andrew",25),
      ("4","Ryan",53),
      ("5","Emily",22),
      ("6","Lily",52)
    )).toDF("id","name","age")
    vertex.show()

    val edges = spark.createDataFrame(List(
      ("6","1","Sister"),
      ("1","2","Husband"),
      ("2","1","Wife"),
      ("5","1","Daughter"),
      ("5","2","Daughter"),
      ("3","1","Son"),
      ("3","2","Son"),
      ("4","1","Friend"),
      ("1","5","Father"),
      ("1","3","Father"),
      ("2","5","Mother"),
      ("2","3","Mother")
    )).toDF("src","dst","relationship")
    edges.show()

    val graph = GraphFrame(vertex,edges)
    graph.vertices.show()
    graph.edges.show()
    graph.vertices.groupBy().min("age").show()
    val motifs = graph.find("(a)-[e]->(b); (b)-[e2]->(a)")
    motifs.show()

    motifs.filter("b.age > 30").show()

    /***longing and save **/
    graph.vertices.write.parquet("file:///Users/sws/IdeaProjects/JavaScala/src/main/scala/Data/vertices")
    graph.edges.write.parquet("file:///Users/sws/IdeaProjects/JavaScala/src/main/scala/Data/edges")

    val verticesDF = spark.read.parquet("file:///Users/sws/IdeaProjects/JavaScala/src/main/scala/Data/vertices")
    val edgesDF = spark.read.parquet("file:///Users/sws/IdeaProjects/JavaScala/src/main/scala/Data/edges")
    val sameGraph = GraphFrame(verticesDF, edgesDF)

  }
}
```
