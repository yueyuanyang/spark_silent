## spark DataSet 创建方式

### spark DataSet 几种创建方式

**1）通过read 读取文件创建**

```
import com.spark.DataFrameRDDApp.SetLogger
import org.apache.spark.sql.SparkSession
 
/**
  * Created by *** 2017/12/23 16:17
  * DataSet的使用
  */
  
case class Sales(transactionId: Int, customerId: Int, itemId: Int, amountPaid: Double)
    
object DataSetApp {
  def main(args: Array[String]): Unit = {
    SetLogger()
    val spark = SparkSession.builder()
      .master("local[2]")
      .appName("DataFrameRDDApp").getOrCreate()
 
    val path = "src/data/sales.csv"
 
    // 解析CSV文件
    val salesDF = spark.read.option("header", "true").option("inferSchema", "true").csv(path)
    salesDF.show()
 
    // 注意：需要导入隐式转换
    import spark.implicits._
    //转换成DataSet
    val salesDS = salesDF.as[Sales]
    
    salesDS.show()
    salesDS.map(line => line.itemId).show()
    salesDF.select("itemId").show()
    salesDS.select("transactionId").show()
 
    spark.stop()
  }
}
```

**2.根据反射的DataSet去转化**

```
case class Person(name : String,age : Int)

object CreateDataSet {
  def main(args: Array[String]) {
    val spark = SparkSession.builder()
      .appName("createDataSet")
      .master("local[1]")
      .getOrCreate()

    import spark.implicits._
    val person = Seq(Person("Andy", 32),Person("tom", 31),Person("Andy", 23)).toDS()
    person.show()
    spark.stop()
   }
 }
```


**3.根据已有的DataSet去转化**

```
case class Person(name : String,age : Int)

object CreateDataSet {
  def main(args: Array[String]) {
    val spark = SparkSession.builder()
      .appName("createDataSet")
      .master("local[1]")
      .getOrCreate()

    import spark.implicits._
    val person = Seq(Person("Andy", 32),Person("tom", 31),Person("Andy", 23)).toDS()
    
    // 根据已有的DataSet 去生成新的
    val other_person = person.map(_.name)
    
    other_person.show()
    spark.stop()
  }    
}
```
