## spark-sql dataFrame 创建方式

### spark-SQL中的dataFrame 几种种创建方式

**1. 非json格式的RDD创建DataFrame（重要）**

**1) 动态创建Schema将非json格式的RDD转换成DataFrame（`建议使用`）**

**代码实现**

```
val conf = new SparkConf()
conf.setMaster("local").setAppName("rddStruct")
val sc = new SparkContext(conf)
val sqlContext = new SQLContext(sc)
val lineRDD = sc.textFile("./sparksql/person.txt")
val rowRDD = lineRDD.map { x => {
  val split = x.split(",")
  RowFactory.create(split(0),split(1),Integer.valueOf(split(2)))
} }
 
val schema = StructType(List(
  StructField("id",StringType,true),
  StructField("name",StringType,true),
  StructField("age",IntegerType,true)
))
 
val df = sqlContext.createDataFrame(rowRDD, schema)
df.show()
df.printSchema()
sc.stop()
```

**2) 通过反射的方式将非json格式的RDD转换成DataFrame（`不建议使用`） **

**代码实现**

```
case class Person(name : String,address : String,age : Int)

val conf = new SparkConf()
conf.setMaster("local").setAppName("rddreflect")
val sc = new SparkContext(conf)
val sqlContext = new SQLContext(sc)
val lineRDD = sc.textFile("./sparksql/person.txt")
/**
 * 将RDD隐式转换成DataFrame
 */
import sqlContext.implicits._
 
val personRDD = lineRDD.map { x => {
  val person = Person(x.split(",")(0),x.split(",")(1),Integer.valueOf(x.split(",")(2)))
  person
} }
val df = personRDD.toDF();
df.show()
 
/**
 * 将DataFrame转换成PersonRDD
 */
val rdd = df.rdd
val result = rdd.map { x => {
  Person(x.getAs("id"),x.getAs("name"),x.getAs("age"))
} }
result.foreach { println}
sc.stop()

```


**2. 读取json格式的文件创建DataFrame**

**代码实现**

```
val conf = new SparkConf()
conf.setMaster("local").setAppName("jsonfile")
 
val sc = new SparkContext(conf)
val sqlContext = new SQLContext(sc)
val df = sqlContext.read.json("sparksql/json") 
//val df1 = sqlContext.read.format("json").load("sparksql/json")
 
df.show()
df.printSchema()
//select * from table
df.select(df.col("name")).show()
//select name from table where age>19
df.select(df.col("name"),df.col("age")).where(df.col("age").gt(19)).show()
//select count(*) from table group by age
df.groupBy(df.col("age")).count().show();
  
/**
 * 注册临时表
 */
df.registerTempTable("jtable")
val result  = sqlContext.sql("select  * from jtable")
result.show()
sc.stop()
```
**3. 通过json格式的RDD创建DataFrame**

**代码实现**

```
val conf = new SparkConf()
conf.setMaster("local").setAppName("jsonrdd")
val sc = new SparkContext(conf)
val sqlContext = new SQLContext(sc)
 
val nameRDD = sc.makeRDD(Array(
  "{\"name\":\"zhangsan\",\"age\":18}",
  "{\"name\":\"lisi\",\"age\":19}",
  "{\"name\":\"wangwu\",\"age\":20}"
))
val scoreRDD = sc.makeRDD(Array(
        "{\"name\":\"zhangsan\",\"score\":100}",
        "{\"name\":\"lisi\",\"score\":200}",
        "{\"name\":\"wangwu\",\"score\":300}"
        ))
val nameDF = sqlContext.read.json(nameRDD)
val scoreDF = sqlContext.read.json(scoreRDD)
nameDF.registerTempTable("name")       
scoreDF.registerTempTable("score")     
val result = sqlContext.sql("select name.name,name.age,score.score from name,score where name.name = score.name")
result.show()
sc.stop()

```

***4. 读取parquet文件创建DataFrame*

**注意**：

可以将DataFrame存储成parquet文件。保存成parquet文件的方式有两种

```
df.write().mode(SaveMode.Overwrite).format("parquet").save("./sparksql/parquet");
df.write().mode(SaveMode.Overwrite).parquet("./sparksql/parquet");
```

SaveMode指定文件保存时的模式。

- Overwrite：覆盖
- Append：追加
- ErrorIfExists：如果存在就报错
- Ignore：如果存在就忽略

**代码实现**

```
val conf = new SparkConf()
 conf.setMaster("local").setAppName("parquet")
 val sc = new SparkContext(conf)
 val sqlContext = new SQLContext(sc)
 val jsonRDD = sc.textFile("sparksql/json")
 val df = sqlContext.read.json(jsonRDD)
 df.show()
  /**
  * 将DF保存为parquet文件
  */
df.write.mode(SaveMode.Overwrite).format("parquet").save("./sparksql/parquet")
 df.write.mode(SaveMode.Overwrite).parquet("./sparksql/parquet")
 /**
  * 读取parquet文件
  */
 var result = sqlContext.read.parquet("./sparksql/parquet")
 result = sqlContext.read.format("parquet").load("./sparksql/parquet")
 result.show()
 sc.stop()
```

**5. 读取JDBC中的数据创建DataFrame(MySql为例)**

**代码实现**

```
val conf = new SparkConf()
conf.setMaster("local").setAppName("mysql")
val sc = new SparkContext(conf)
val sqlContext = new SQLContext(sc)
/**
 * 第一种方式读取Mysql数据库表创建DF
 */
val options = new HashMap[String,String]();
options.put("url", "jdbc:mysql://192.168.179.4:3306/spark")
options.put("driver","com.mysql.jdbc.Driver")
options.put("user","root")
options.put("password", "123456")
options.put("dbtable","person")
val person = sqlContext.read.format("jdbc").options(options).load()
person.show()
person.registerTempTable("person")
/**
 * 第二种方式读取Mysql数据库表创建DF
 */
val reader = sqlContext.read.format("jdbc")
reader.option("url", "jdbc:mysql://192.168.179.4:3306/spark")
reader.option("driver","com.mysql.jdbc.Driver")
reader.option("user","root")
reader.option("password","123456")
reader.option("dbtable", "score")
val score = reader.load()
score.show()
score.registerTempTable("score")
val result = sqlContext.sql("select person.id,person.name,score.score from person,score where person.name = score.name")
result.show()
/**
 * 将数据写入到Mysql表中
 */
val properties = new Properties()
properties.setProperty("user", "root")
properties.setProperty("password", "123456")
result.write.mode(SaveMode.Append).jdbc("jdbc:mysql://192.168.179.4:3306/spark", "result", properties)
 
sc.stop()
```













