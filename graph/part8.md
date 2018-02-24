### graphx操作实例07-degrees和neighbors 
求图中对应顶点的度以及邻居节点

```
import org.apache.spark._
import org.apache.spark.SparkContext
import org.apache.spark.SparkContext._
import org.apache.spark.graphx._
import org.apache.spark.rdd.RDD

object Day10 {

    def main(args: Array[String]) = {

		val conf = new SparkConf()
		val sc = new SparkContext("local", "test", conf)

		// day10.tsv
		// 2 1
		// 3 1
		// 4 1
		// 5 1
		// 1 2
		// 4 3
		// 5 3
		// 1 4
		val graph = GraphLoader.edgeListFile(sc, "graphdata/day10.tsv").cache()

		println("\n\n~~~~~~~~~ Confirm Vertices Internal of graph ")
		graph.vertices.collect.foreach(println(_))
		// (4,1)
		// (2,1)
		// (1,1)
		// (3,1)
		// (5,1)

		println("\n\n~~~~~~~~~ Confirm Edges Internal of graph ")
		graph.edges.collect.foreach(println(_))
		// Edge(2,1,1)
		// Edge(3,1,1)
		// Edge(4,1,1)
		// Edge(5,1,1)
		// Edge(1,2,1)
		// Edge(1,4,1)
		// Edge(4,3,1)
		// Edge(5,3,1)

		// degrees ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

		println("\n\n~~~~~~~~~ Confirm inDegrees ")
		graph.inDegrees.collect.foreach(d => println(d._1 + "'s inDegree is " + d._2))
		// 4's inDegree is 1
		// 2's inDegree is 1
		// 1's inDegree is 4
		// 3's inDegree is 2

		println("\n\n~~~~~~~~~ Confirm outDegrees ")
		graph.outDegrees.collect.foreach(d => println(d._1 + "'s outDegree is " + d._2))
		// 4's outDegree is 2
		// 2's outDegree is 1
		// 1's outDegree is 2
		// 3's outDegree is 1
		// 5's outDegree is 2

		println("\n\n~~~~~~~~~ Confirm degrees ")
		// 求出对应点所有的度
		graph.degrees.collect.foreach(d => println(d._1 + "'s degree is " + d._2))
		// 4's degree is 3
		// 2's degree is 2
		// 1's degree is 6
		// 3's degree is 3
		// 5's degree is 2

		def max(a: (VertexId, Int), b: (VertexId, Int)): (VertexId, Int) = {
		  if (a._2 > b._2) a else b
		}

		println("\n\n~~~~~~~~~ Confirm max inDegrees ")
		println(graph.inDegrees.reduce(max))
		// (1,4)

		// collectNeighborIds ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

		println("\n\n~~~~~~~~~ Confirm collectNeighborIds(IN) ")
		graph.collectNeighborIds(EdgeDirection.In).collect.foreach(n => println(n._1 + "'s in neighbors : " + n._2.mkString(",")))
		// 4's in neighbors : 1
		// 2's in neighbors : 1
		// 1's in neighbors : 2,3,4,5
		// 3's in neighbors : 4,5
		// 5's in neighbors : 

		println("\n\n~~~~~~~~~ Confirm collectNeighborIds(OUT) ")
		graph.collectNeighborIds(EdgeDirection.Out).collect.foreach(n => println(n._1 + "'s out neighbors : " + n._2.mkString(",")))
		// 4's out neighbors : 1,3
		// 2's out neighbors : 1
		// 1's out neighbors : 2,4
		// 3's out neighbors : 1
		// 5's out neighbors : 1,3

		println("\n\n~~~~~~~~~ Confirm collectNeighborIds(Either) ")
		graph.collectNeighborIds(EdgeDirection.Either).collect.foreach(n => println(n._1 + "'s neighbors : " + n._2.distinct.mkString(",")))
		// 4's neighbors : 1,3
		// 2's neighbors : 1
		// 1's neighbors : 2,3,4,5
		// 3's neighbors : 1,4,5
		// 5's neighbors : 1,3

		// collectNeighbor ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

		// println("\n\n~~~~~~~~~ Confirm collectNeighbors(IN) ")
		// graph.collectNeighbors(EdgeDirection.In).collect.foreach(n => println(n._1 + "'s in neighbors : " + n._2.mkString(",")))

		println("\n\n~~~~~~~~~ Confirm collectNeighbors(OUT) ")
		graph.collectNeighbors(EdgeDirection.Out).collect.foreach(n => println(n._1 + "'s out neighbors : " + n._2.mkString(",")))
		// 4's out neighbors : (1,1),(3,1)
		// 2's out neighbors : (1,1)
		// 1's out neighbors : (2,1),(4,1)
		// 3's out neighbors : (1,1)
		// 5's out neighbors : (1,1),(3,1)

		println("\n\n~~~~~~~~~ Confirm collectNeighbors(Either) ")
		graph.collectNeighbors(EdgeDirection.Either).collect.foreach(n => println(n._1 + "'s neighbors : " + n._2.distinct.mkString(",")))
		// 4's neighbors : (1,1),(3,1)
		// 2's neighbors : (1,1)
		// 1's neighbors : (2,1),(3,1),(4,1),(5,1)
		// 3's neighbors : (1,1),(4,1),(5,1)
		// 5's neighbors : (1,1),(3,1)

		sc.stop
	}
}
```
