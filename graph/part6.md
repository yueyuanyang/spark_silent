### graphx操作实例05-VertexRDD和EdgeRDD属性测试
filter、mapValues、diff（测试之后感觉有问题）、leftJoin、innerJoin、aggregateUsingIndex、reverse
```
import org.apache.spark._
import org.apache.spark.SparkContext
import org.apache.spark.SparkContext._
import org.apache.spark.graphx._
import org.apache.spark.rdd.RDD

object Day06_08 {

    def main(args: Array[String]) = {

    	// **************************************************************************
    	//  VertexRDD 和 EdgeRDD
    	// **************************************************************************

		val conf = new SparkConf()
		val sc = new SparkContext("local", "test", conf)

		// day06-vertices
		// 1,Taro,100
		// 2,Jiro,200
		// 3,Sabo,300
		val vertexLines: RDD[String] = sc.textFile("graphdata/day06-vertices.csv")
		val v: RDD[(VertexId, (String, Long))] = vertexLines.map(line => {
				val cols = line.split(",")
				(cols(0).toLong, (cols(1), cols(2).toLong))
			})

		// day06-edges.csv
		// 1,2,100,2014/12/1
		// 2,3,200,2014/12/2
		// 3,1,300,2014/12/3
		val format = new java.text.SimpleDateFormat("yyyy/MM/dd")
		val edgeLines: RDD[String] = sc.textFile("graphdata/day06-edges.csv")
		val e:RDD[Edge[((Long, java.util.Date))]] = edgeLines.map(line => {
				val cols = line.split(",")
				Edge(cols(0).toLong, cols(1).toLong, (cols(2).toLong, format.parse(cols(3))))
			})

		val graph:Graph[(String, Long), (Long, java.util.Date)] = Graph(v, e)

		val vertices:VertexRDD[(String, Long)] = graph.vertices

		println(" Confirm Vertices Internal of graph ")
		vertices.collect.foreach(println(_))
		// (2,(Jiro,200))
		// (1,(Taro,100))
		// (3,(Sabo,300))

		val edges:EdgeRDD[((Long, java.util.Date))] = graph.edges

		println("Confirm Edges Internal of graph ")
		edges.collect.foreach(println(_))
		// Edge(1,2,(100,Mon Dec 01 00:00:00 EST 2014))
		// Edge(2,3,(200,Tue Dec 02 00:00:00 EST 2014))
		// Edge(3,1,(300,Wed Dec 03 00:00:00 EST 2014))

    	// ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    	// Day07 : VertexRDD
    	// ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    	// 使用filter筛选出属性value>150的顶点
		val filteredVertices:VertexRDD[(String, Long)] = vertices.filter{ case (vid:VertexId, (name:String, value:Long)) => value > 150 }

		println("Confirm filtered vertices ")
		filteredVertices.collect.foreach(println(_))
		// (2,(Jiro,200))
		// (3,(Sabo,300))

		// 除了mapVertices之外，也可以使用VertexRDD的mapValues来修改顶点的属性
		val mappedVertices:VertexRDD[Long] = vertices.mapValues((vid:VertexId, attr:(String, Long)) => attr._2 * attr._1.length)

		println(" Confirm mapped vertices ")
		mappedVertices.collect.foreach(println(_))
		// (2,800)
		// (1,400)
		// (3,1200)

		println("Confirm diffed vertices ")
		// Remove vertices from this set that appear in the other set
		// val diffedVertices:VertexRDD[(String, Long)] = filteredVertices.diff(vertices)
		val diffedVertices:VertexRDD[(String, Long)] = vertices.diff(filteredVertices)

		// diffedVertices.collect.foreach(println(_))
		println("vertices : " + vertices.count)
		println("filteredVertices : " + filteredVertices.count)
		println("diffedVertices : " + diffedVertices.count)
		// vertices : 3
		// filteredVertices : 2
		// diffedVertices : 0 这个结果感觉不正确

		// 0L until 2L 生成 Range(0, 1)，再使用VertexRDD的map来修改顶点的属性
		val setA: VertexRDD[Int] = VertexRDD(sc.parallelize(0L until 2L).map(id => (id, id.toInt)))
		println("set A ")
		setA.collect.foreach(println(_))
		// (0,0)
		// (1,1)
		val setB: VertexRDD[Int] = VertexRDD(sc.parallelize(1L until 3L).map(id => (id, id.toInt)))
		println("set B ")
		setB.collect.foreach(println(_))
		// (2,2)
		// (1,1)
		val diff = setA.diff(setB)
		println("diff ")
		diff.collect.foreach(println(_))
		// diff输出为空？？这里是否使用有误？？

		// day07-01-vertices.csv
		// 1,Japan
		// 2,USA
		val verticesWithCountry: RDD[(VertexId, String)] = sc.textFile("graphdata/day07-01-vertices.csv").map(line => {
				(line.split(",")(0).toLong, line.split(",")(1))
			})

		// 使用leftJoin来合并两个数据集中的属性，以left为主，如果verticesWithCountry存在则赋值，不存在，则赋值为"World"
		val leftJoinedVertices = vertices.leftJoin(verticesWithCountry){
			(vid, left, right) => (left._1, left._2, right.getOrElse("World"))
		}

		println(" Confirm leftJoined vertices ")
		leftJoinedVertices.collect.foreach(println(_))
		// (2,(Jiro,200,USA))
		// (1,(Taro,100,Japan))
		// (3,(Sabo,300,World))

		// 使用innerJoin来合并两个顶点属性，以verticesWithCountry为主
		val innerJoinedVertices = vertices.innerJoin(verticesWithCountry){
			(vid, left, right) => (left._1, left._2, right)
		}

		println("Confirm innerJoined vertices ")
		innerJoinedVertices.collect.foreach(println(_))
		// (2,(Jiro,200,USA))
		// (1,(Taro,100,Japan))
		// 因为verticesWithCountry只有两个值，所以这里也只有两个值

		// day07-02-vertices.csv
		// 1,10
		// 2,20
		// 2,21
		val verticesWithNum: RDD[(VertexId, Long)] = sc.textFile("graphdata/day07-02-vertices.csv").map(line => {
				(line.split(",")(0).toLong, line.split(",")(1).toLong)
			})

		// 使用aggregateUsingIndex将verticesWithNum中相同Id中的属性相加
		val auiVertices:VertexRDD[Long] = vertices.aggregateUsingIndex(verticesWithNum, _ + _)
		
		println(" Confirm aggregateUsingIndexed vertices ")
		auiVertices.collect.foreach(println(_))
		// (2,41)
		// (1,10)


    	// ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    	// Day08 : EdgeRDD
    	// ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    	// 使用mapValues对edge中的属性进行修改
		val mappedEdges:EdgeRDD[Long] = edges.mapValues(edge => edge.attr._1 + 1)

		println("Confirm mapped edges ")
		mappedEdges.collect.foreach(println(_))
		// Edge(1,2,101)
		// Edge(2,3,201)
		// Edge(3,1,301)

		// The reverse operator returns a new graph with all the edge directions reversed
		val reversedEdges:EdgeRDD[(Long, java.util.Date)] = edges.reverse

		println(" Confirm reversed edges ")
		reversedEdges.collect.foreach(println(_))
		// Edge(2,1,(100,Mon Dec 01 00:00:00 EST 2014))
		// Edge(3,2,(200,Tue Dec 02 00:00:00 EST 2014))
		// Edge(1,3,(300,Wed Dec 03 00:00:00 EST 2014))

		// day08-edges.csv
		// 1,2,love
		// 2,3,hate
		// 3,1,even
		val e2:RDD[Edge[String]] = sc.textFile("graphdata/day08-edges.csv").map(line => {
				val cols = line.split(",")
				Edge(cols(0).toLong, cols(1).toLong, cols(2))
			})

		val graph2:Graph[(String, Long), String] = Graph(v, e2)
		val edges2 = graph2.edges
		val innerJoinedEdge:EdgeRDD[(Long, java.util.Date, String)] = edges.innerJoin(edges2)((v1, v2, attr1, attr2) => (attr1._1, attr1._2, attr2))

		println("\n\n~~~~~~~~~ Confirm innerJoined edge ")
		innerJoinedEdge.collect.foreach(println(_))
		// Edge(1,2,(100,Mon Dec 01 00:00:00 EST 2014,love))
		// Edge(2,3,(200,Tue Dec 02 00:00:00 EST 2014,hate))
		// Edge(3,1,(300,Wed Dec 03 00:00:00 EST 2014,even))

		sc.stop
	}
}
```
