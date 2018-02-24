### graphx操作实例04-使用mapReduceTriplets、mapEdges、mapVertices修改属性
用mapReduceTriplets、mapEdges、mapVertices操作修改属性
```
import org.apache.spark._  
import org.apache.spark.SparkContext  
import org.apache.spark.SparkContext._  
import org.apache.spark.graphx._  
import org.apache.spark.rdd.RDD  
  
object Day05 {  
  
    def main(args: Array[String]) = {  
  
        val conf = new SparkConf()  
        val sc = new SparkContext("local", "test", conf)  
  
        // day05-vertices.csv  
        // 1,Taro,100  
        // 2,Jiro,200  
        // 3,Sabo,300  
        val vertexLines: RDD[String] = sc.textFile("graphdata/day05-vertices.csv")  
        val vertices: RDD[(VertexId, (String, Long))] = vertexLines.map(line => {  
                val cols = line.split(",")  
                (cols(0).toLong, (cols(1), cols(2).toLong))  
            })  
  
        // day05-edges.csv  
        // 1,2,100,2014/12/1  
        // 2,3,200,2014/12/2  
        // 3,1,300,2014/12/3  
        val format = new java.text.SimpleDateFormat("yyyy/MM/dd")  
        val edgeLines: RDD[String] = sc.textFile("graphdata/day05-edges.csv")  
        val edges:RDD[Edge[((Long, java.util.Date))]] = edgeLines.map(line => {  
                val cols = line.split(",")  
                Edge(cols(0).toLong, cols(1).toLong, (cols(2).toLong, format.parse(cols(3))))  
            })  
  
        // 使用vertices和edges生成图  
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
  
        // 使用mapVertices修改顶点的属性，由原先的(String, Long)修改为（String的length*Long的值）  
        val graph2:Graph[Long, (Long, java.util.Date)] = graph.mapVertices((vid:VertexId, attr:(String, Long)) => attr._1.length * attr._2)  
  
        println("\n\n~~~~~~~~~ Confirm Vertices Internal of graph2 ")  
        graph2.vertices.collect.foreach(println(_))  
        // (2,800) Jiro的长度为4，乘以200得到800，下同  
        // (1,400)  
        // (3,1200)  
  
        // 使用mapEdges将edge的属性由(100,Mon Dec 01 00:00:00 EST 2014)变为100  
        val graph3:Graph[(String, Long), Long] = graph.mapEdges( edge => edge.attr._1 )  
  
        println("\n\n~~~~~~~~~ Confirm Vertices Internal of graph3 ")  
        graph3.edges.collect.foreach(println(_))  
        // Edge(1,2,100)  
        // Edge(2,3,200)  
        // Edge(3,1,300)  
  
        println("\n\n~~~~~~~~~ Confirm triplets Internal of graph ")  
        graph.triplets.collect.foreach(println(_))  
        // ((1,(Taro,100)),(2,(Jiro,200)),(100,Mon Dec 01 00:00:00 EST 2014))  
        // ((2,(Jiro,200)),(3,(Sabo,300)),(200,Tue Dec 02 00:00:00 EST 2014))  
        // ((3,(Sabo,300)),(1,(Taro,100)),(300,Wed Dec 03 00:00:00 EST 2014))  
        // 到这里可以观察到，上述操作对graph本身并没有影响  
  
        // 使用mapTriplets对三元组整体进行操作，即可以利用srcAttr attr dstAttr来修改attr的信息  
        val graph4:Graph[(String, Long), Long] = graph.mapTriplets(edge => edge.srcAttr._2 + edge.attr._1 + edge.dstAttr._2)  
  
        println("\n\n~~~~~~~~~ Confirm Vertices Internal of graph4 ")  
        graph4.edges.collect.foreach(println(_))  
        // Edge(1,2,400) //400 = 100+200+100  
        // Edge(2,3,700)  
        // Edge(3,1,700)  
  
        // 使用mapReduceTriplets来生成新的VertexRDD  
        // 利用map对每一个三元组进行操作  
        // 利用reduce对相同Id的顶点属性进行操作
        /****2.0+ 被弃用****/
        val newVertices:VertexRDD[Long] = graph.mapReduceTriplets(  
                mapFunc = (edge:EdgeTriplet[(String, Long), (Long, java.util.Date)]) => {  
                    val toSrc = Iterator((edge.srcId, edge.srcAttr._2 - edge.attr._1))  
                    val toDst = Iterator((edge.dstId, edge.dstAttr._2 + edge.attr._1))  
                    toSrc ++ toDst  
                },  
                reduceFunc = (a1:Long, a2:Long) => ( a1 + a2 )  
            )  
        /****aggregateMessages 改写 ***/     
       val newVertices = graph.aggregateMessages[Long](
           triplet =>{
               val toSrc : Long = triplet.srcAttr._2 - triplet.attr._1
               val toDst : Long = triplet.dstAttr._2 + triplet.attr._1
               triplet.sendToDst(toSrc + toDst)
           },
            (a1:Long, a2:Long) => ( a1 + a2 )
          )    
  
        println("\n\n~~~~~~~~~ Confirm Vertices Internal of newVertices ")  
        newVertices.collect.foreach(println(_))  
        // (2,300)  
        // (1,400)  
        // (3,500)  
  
        sc.stop  
    }  
}  
```
