### graphx操作实例09-Pregel学习
#### 第一个demo的功能是在设置传播方向延src到dst时，找到3步之内能形成环路的顶点
```
import org.apache.spark._
import org.apache.spark.SparkContext
import org.apache.spark.SparkContext._
import org.apache.spark.graphx._
import org.apache.spark.rdd.RDD

import scala.reflect.ClassTag

object Day14 {

  def main(args: Array[String]) = {

    val conf = new SparkConf()
    val sc = new SparkContext("local", "test", conf)

    // day14.tsv
    // 0 1
    // 1 2
    // 2 3
    // 3 4
    // 2 4
    // 4 1
    // 4 5
    // 5 6
    // 6 1
    // 5 7
    // 7 8
    val graph:Graph[Int, Int] = GraphLoader.edgeListFile(sc, "graphdata/day14.tsv").cache()

    println("\n\n~~~~~~~~~ Confirm Vertices Internal of graph ")
    graph.vertices.collect.foreach(println(_))
    // (4,1)
    // (0,1)
    // (6,1)
    // (8,1)
    // (2,1)
    // (1,1)
    // (3,1)
    // (7,1)
    // (5,1)

    println("\n\n~~~~~~~~~ Confirm Edges Internal of graph ")
    graph.edges.collect.foreach(println(_))
    // Edge(0,1,1)
    // Edge(1,2,1)
    // Edge(2,3,1)
    // Edge(2,4,1)
    // Edge(3,4,1)
    // Edge(4,1,1)
    // Edge(4,5,1)
    // Edge(5,6,1)
    // Edge(5,7,1)
    // Edge(6,1,1)
    // Edge(7,8,1)

	val circleGraph = Pregel(
		// 各顶点置空
		graph.mapVertices((id, attr) => Set[VertexId]()),//初始信息
		// 最初值为空
		Set[VertexId](),
		//最大迭代次数为3
		3,
		// 发送消息（默认出边）的方向
		EdgeDirection.out) (
			// 用户定义的接收消息
			(id, attr, msg) => (msg ++ attr),
			// 计算消息
			edge => Iterator((edge.dstId, (edge.srcAttr + edge.srcId))),
			// 合并消息
			(a, b) => (a ++ b)
		// 取出包含自身id的点
		).subgraph(vpred = (id, attr) => attr.contains(id))

    println("\n\n~~~~~~~~~ Confirm Vertices of circleGraph ")
    circleGraph.vertices.collect.foreach(println(_))
    // (4,Set(0, 1, 6, 2, 3, 4))
    // (2,Set(0, 5, 1, 6, 2, 3, 4))
    // (1,Set(0, 5, 1, 6, 2, 3, 4))

    sc.stop
  }
}
```


#### 第二个demo的功能是找到一跳节点和二跳节点

```
import org.apache.spark._
import org.apache.spark.SparkContext
import org.apache.spark.SparkContext._
import org.apache.spark.graphx._
import org.apache.spark.rdd.RDD

import scala.reflect.ClassTag

object Day13 {

  def main(args: Array[String]) = {

    val conf = new SparkConf()
    val sc = new SparkContext("local", "test", conf)

    // Day13.tsv
    // 1 2
    // 2 3
    // 1 4
    // 3 5
    // 2 8
    // 8 7
    // 4 5
    // 5 6
    // 6 7
    // 3 7
    // 4 3
    val graph:Graph[Int, Int] = GraphLoader.edgeListFile(sc, "graphdata/Day13.tsv").cache()

    println("\n\n~~~~~~~~~ Confirm Vertices Internal of graph ")
    graph.vertices.collect.foreach(println(_))
    // (4,1)
    // (6,1)
    // (8,1)
    // (2,1)
    // (1,1)
    // (3,1)
    // (7,1)
    // (5,1)

    println("\n\n~~~~~~~~~ Confirm Edges Internal of graph ")
    graph.edges.collect.foreach(println(_))
    // Edge(1,2,1)
    // Edge(1,4,1)
    // Edge(2,3,1)
    // Edge(2,8,1)
    // Edge(3,5,1)
    // Edge(8,7,1)
    // Edge(3,7,1)
    // Edge(4,3,1)
    // Edge(4,5,1)
    // Edge(5,6,1)
    // Edge(6,7,1)

	def sendMsgFunc(edge:EdgeTriplet[Int, Int]) = {
		if(edge.srcAttr <= 0){
			if(edge.dstAttr <= 0){
				// 如果双方都小于0，则不发送信息
				Iterator.empty
			}else{
				// srcAttr小于0，dstAttr大于零，则将dstAttr-1后发送
				Iterator((edge.srcId, edge.dstAttr - 1))
			}
		}else{
			if(edge.dstAttr <= 0){
				// srcAttr大于0，dstAttr<0,则将srcAttr-1后发送
				Iterator((edge.dstId, edge.srcAttr - 1))
			}else{
				// 双方都大于零，则将属性-1后发送
				val toSrc = Iterator((edge.srcId, edge.dstAttr - 1))
				val toDst = Iterator((edge.dstId, edge.srcAttr - 1))
				toDst ++ toSrc
			}
		}
	}

	val friends = Pregel(
		graph.mapVertices((vid, value)=> if(vid == 1) 2 else -1),

		// 发送初始值
		-1,
		// 指定阶数
		2,
		// 双方向发送
		EdgeDirection.Either
	)(
		// 将值设为大的一方
		vprog = (vid, attr, msg) => math.max(attr, msg),
		// 
		sendMsgFunc,
		// 
		(a, b) => math.max(a, b)
	).subgraph(vpred = (vid, v) => v >= 0)

    println("\n\n~~~~~~~~~ Confirm Vertices of friends ")
    friends.vertices.collect.foreach(println(_))
    // (4,1)
    // (8,0)
    // (2,1)
    // (1,2)
    // (3,0)
    // (5,0)

    sc.stop
  }

}
```
