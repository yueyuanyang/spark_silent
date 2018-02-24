### graphx操作实例03-导入顶点和边生成图
#### 导入顶点和边的信息，使用Graph生成图
```
import org.apache.spark._  
import org.apache.spark.SparkContext  
import org.apache.spark.SparkContext._  
import org.apache.spark.graphx._  
import org.apache.spark.rdd.RDD  
  
object Day04 {  
  
    def main(args: Array[String]) = {  
  
        val conf = new SparkConf()  
        val sc = new SparkContext("local", "test", conf)  
  
        // day04-vertices.csv  
        // 1,Taro,100  
        // 2,Jiro,200  
        // 3,Sabo,300  
        val vertexLines: RDD[String] = sc.textFile("graphdata/day04-vertices.csv")  
        val vertices: RDD[(VertexId, (String, Long))] = vertexLines.map(line => {  
                val cols = line.split(",")  
                (cols(0).toLong, (cols(1), cols(2).toLong))  
            })  
  
        // day04-edges.csv  
        // 1,2,100,2014/12/1  
        // 2,3,200,2014/12/2  
        // 3,1,300,2014/12/3  
        val format = new java.text.SimpleDateFormat("yyyy/MM/dd")  
        val edgeLines: RDD[String] = sc.textFile("graphdata/day04-edges.csv")  
        val edges:RDD[Edge[((Long, java.util.Date))]] = edgeLines.map(line => {  
                val cols = line.split(",")  
                Edge(cols(0).toLong, cols(1).toLong, (cols(2).toLong, format.parse(cols(3))))  
            })  
  
        val graph:Graph[(String, Long), (Long, java.util.Date)] = Graph(vertices, edges)  
  
        println("\n\n~~~~~~~~~ Confirm Edges Internal of graph ")  
        graph.edges.collect.foreach(println(_))  
        // Edge(1,2,(100,Mon Dec 01 00:00:00 EST 2014))  
        // Edge(2,3,(200,Tue Dec 02 00:00:00 EST 2014))  
        // Edge(3,1,(300,Wed Dec 03 00:00:00 EST 2014))  
  
  
        println("\n\n~~~~~~~~~ Confirm Vertices Internal of graph ")  
        graph.vertices.collect.foreach(println(_))  
        // (2,(Jiro,200))  
        // (1,(Taro,100))  
        // (3,(Sabo,300))  
  
        sc.stop  
    }  
  
}  
```
