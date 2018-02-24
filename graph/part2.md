### graphx操作实例01-edgeListFile导入数据
```
import org.apache.spark._  
import org.apache.spark.SparkContext  
import org.apache.spark.SparkContext._  
import org.apache.spark.graphx._  
import org.apache.spark.rdd.RDD  
  
object Day02 {  
  
    def main(args: Array[String]) = {  
  
        val conf = new SparkConf()  
        val sc = new SparkContext("local", "test", conf)  
  
        // day02.tsv  
        // 1 2  
        // 2 3  
        // 3 1  
        val graph = GraphLoader.edgeListFile(sc, "graphdata/day02.tsv").cache()  
        // 利用edgeListFile导入day02.tsv数据  
  
        println("\n\n~~~~~~~~~ Confirm Edges Internal of graph ")  
        graph.edges.collect.foreach(println(_))  
        // print  
        // Edge(1,2,1)  
        // Edge(2,3,1)  
        // Edge(3,1,1)  
        // 结果表明，不带属性的边，其属性会自动赋值为1  
  
        println("\n\n~~~~~~~~~ Confirm Vertices Internal of graph ")  
        graph.vertices.collect.foreach(println(_))  
        // print  
        // (2,1)  
        // (1,1)  
        // (3,1)  
        // 结果表明：如果只输入边的Id信息，则顶点属性默认为1  
  
        sc.stop  
    }  
}  
```
