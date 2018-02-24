### graphx操作实例06-subgraph和groupEdges

使用subgraph生成子图

groupEdges用来合并相同Id的边
```
import org.apache.spark._
import org.apache.spark.SparkContext
import org.apache.spark.SparkContext._
import org.apache.spark.graphx._
import org.apache.spark.rdd.RDD

object Day09 {

    def main(args: Array[String]) = {

		val conf = new SparkConf()
		val sc = new SparkContext("local", "test", conf)

		// day09-vertices.csv
		// 1,Taro,100
		// 2,Jiro,200
		// 3,Sabo,300
		val vertexLines: RDD[String] = sc.textFile("graphdata/day09-vertices.csv")
		val v: RDD[(VertexId, (String, Long))] = vertexLines.map(line => {
				val cols = line.split(",")
				(cols(0).toLong, (cols(1), cols(2).toLong))
			})

		// day09-01-edges.csv
		// 1,2,100,2014/12/1
		// 2,3,200,2014/12/2
		// 3,1,300,2014/12/3
		val format = new java.text.SimpleDateFormat("yyyy/MM/dd")
		val edgeLines: RDD[String] = sc.textFile("graphdata/day09-01-edges.csv")
		val e:RDD[Edge[((Long, java.util.Date))]] = edgeLines.map(line => {
				val cols = line.split(",")
				Edge(cols(0).toLong, cols(1).toLong, (cols(2).toLong, format.parse(cols(3))))
			})

		val graph:Graph[(String, Long), (Long, java.util.Date)] = Graph(v, e)

		println("\n\n~~~~~~~~~ Confirm Vertices Internal of graph ")
		graph.vertices.collect.foreach(println(_))
		// (2,(Jiro,200))
		// (1,(Taro,100))
		// (3,(Sabo,300))

		println("\n\n~~~~~~~~~ Confirm Edges Internal of graph ")
		graph.edges.collect.foreach(println(_))
		// Edge(1,2,(100,Mon Dec 01 00:00:00 EST 2014))
		// Edge(2,3,(200,Tue Dec 02 00:00:00 EST 2014))
		// Edge(3,1,(300,Wed Dec 03 00:00:00 EST 2014))

		// reverse ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

		println("\n\n~~~~~~~~~ Confirm Edges reversed graph ")
		graph.reverse.edges.collect.foreach(println(_))
		// Edge(2,1,(100,Mon Dec 01 00:00:00 EST 2014))
		// Edge(3,2,(200,Tue Dec 02 00:00:00 EST 2014))
		// Edge(1,3,(300,Wed Dec 03 00:00:00 EST 2014))

		// subgraph ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

		println("\n\n~~~~~~~~~ Confirm Subgraphed vertices graph ")
		// 利用subgraph根据顶点和边的条件建立子图
		graph.subgraph(vpred = (vid, v) => v._2 >= 200).vertices.collect.foreach(println(_))
		// (2,(Jiro,200))
		// (3,(Sabo,300))

		println("\n\n~~~~~~~~~ Confirm Subgraphed edges graph ")
		graph.subgraph(epred = edge => edge.attr._1 >= 200).edges.collect.foreach(println(_))
		// Edge(2,3,(200,Tue Dec 02 00:00:00 EST 2014))
		// Edge(3,1,(300,Wed Dec 03 00:00:00 EST 2014))

		// 对顶点和边同时加限制
		val subGraph = graph.subgraph(vpred = (vid, v) => v._2 >= 200, epred = edge => edge.attr._1 >= 200)

		println("\n\n~~~~~~~~~ Confirm vertices of Subgraphed graph ")
		subGraph.vertices.collect.foreach(println(_))
		// (2,(Jiro,200))
		// (3,(Sabo,300))

		println("\n\n~~~~~~~~~ Confirm edges of Subgraphed graph ")
		subGraph.edges.collect.foreach(println(_))
		// Edge(2,3,(200,Tue Dec 02 00:00:00 EST 2014))

		// mask ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
		// The mask operator constructs a subgraph by returning a graph that contains the vertices and edges that are also found in the input graph
		// 返回一个子图，该子图由取原子图和原图的并集
		val maskedGraph = graph.mask(subGraph)

		println("\n\n~~~~~~~~~ Confirm Masked Graph vertices graph ")
		maskedGraph.vertices.collect.foreach(println(_))
		// (2,(Jiro,200))
		// (3,(Sabo,300))

		println("\n\n~~~~~~~~~ Confirm Masked Graph edges graph ")
		maskedGraph.edges.collect.foreach(println(_))
		// Edge(2,3,(200,Tue Dec 02 00:00:00 EST 2014))

		// groupEdge ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

		// day09-02-edges.csv
		// 1,2,100,2014/12/1
		// 1,2,110,2014/12/11
		// 2,3,200,2014/12/21
		// 2,3,210,2014/12/2
		// 3,1,300,2014/12/3
		// 3,1,310,2014/12/31
		val edgeLines2: RDD[String] = sc.textFile("graphdata/day09-02-edges.csv")
		val e2:RDD[Edge[((Long, java.util.Date))]] = edgeLines2.map(line => {
				val cols = line.split(",")
				Edge(cols(0).toLong, cols(1).toLong, (cols(2).toLong, format.parse(cols(3))))
			})

		val graph2:Graph[(String, Long), (Long, java.util.Date)] = Graph(v, e2)

		// 使用groupEdges语句将edge中相同Id的数据进行合并
		val edgeGroupedGraph:Graph[(String, Long), (Long, java.util.Date)] = graph2.groupEdges(merge = (e1, e2) => (e1._1 + e2._1, if(e1._2.getTime < e2._2.getTime) e1._2 else e2._2))

		println("\n\n~~~~~~~~~ Confirm merged edges graph ")
		edgeGroupedGraph.edges.collect.foreach(println(_))
		// Edge(1,2,(210,Mon Dec 01 00:00:00 EST 2014))
		// Edge(2,3,(200,Sun Dec 21 00:00:00 EST 2014))
		// Edge(2,3,(210,Tue Dec 02 00:00:00 EST 2014))
		// Edge(3,1,(610,Wed Dec 03 00:00:00 EST 2014))

		sc.stop
	}
}
```
