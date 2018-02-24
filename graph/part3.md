### graphx操作实例02-joinVertices
### 例子说明：利用joinVertices和outJoinVertices对graph的顶点属性进行修改
```
import org.apache.spark._  
import org.apache.spark.SparkContext  
import org.apache.spark.SparkContext._  
import org.apache.spark.graphx._  
import org.apache.spark.rdd.RDD  
  
object Day03 {  
  
    def main(args: Array[String]) = {  
  
        val conf = new SparkConf()  
        val sc = new SparkContext("local", "test", conf)  
  
        // 利用edge信息生成图  
        // dataset info  
        // 1 2  
        // 2 3  
        // 3 1  
        val graph = GraphLoader.edgeListFile(sc, "graphdata/day03-edges.tsv").cache()  
  
        // 以[vid, name]形式读取vertex信息  
        // day03-vertices.csv  
        // 1,Taro  
        // 2,Jiro  
        val vertexLines = sc.textFile("graphdata/day03-vertices.csv")  
        val users: RDD[(VertexId, String)] = vertexLines.map(line => {  
                val cols = line.split(",")  
                (cols(0).toLong, cols(1))  
            })  
  
        // 将users中的vertex属性添加到graph中，生成graph2  
        // 使用joinVertices操作，用user中的属性替换图中对应Id的属性  
        // 先将图中的顶点属性置空  
        val graph2 = graph.mapVertices((id, attr) => "").joinVertices(users){(vid, empty, user) => user}  
  
        println("\n\n~~~~~~~~~ Confirm Vertices Internal of graph2 ")  
        graph2.vertices.collect.foreach(println(_))  
        // (1,Taro)  
        // (2,Jiro)  
        // (3,)  
  
        // 使用outerJoinVertices将user中的属性赋给graph中的顶点，如果图中顶点不在user中，则赋值为None  
        val graph3 = graph.mapVertices((id, attr) => "").outerJoinVertices(users){(vid, empty, user) => user.getOrElse("None")}  
  
        println("\n\n~~~~~~~~~ Confirm Vertices Internal of graph3 ")  
        graph3.vertices.collect.foreach(println(_))  
        // (2,Jiro)  
        // (1,Taro)  
        // (3,None)  
        // 结果表明，如果graph的顶点在user中，则将user的属性赋给graph中对应的顶点，否则赋值为None。  
  
        sc.stop  
    }  
}
```
