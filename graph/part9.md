### graphx操作实例08-connectedComponents
利用connectedComponents求图中的连通图
```
<span style="font-size:14px;">import org.apache.spark._
import org.apache.spark.SparkContext
import org.apache.spark.SparkContext._
import org.apache.spark.graphx._
import org.apache.spark.rdd.RDD

import scala.reflect.ClassTag

object Day11 {

  def main(args: Array[String]) = {

    val conf = new SparkConf()
    val sc = new SparkContext("local", "test", conf)

    // day11.tsv
    // 1 2
    // 2 3
    // 3 1
    // 4 5
    // 5 6
    // 6 7
    val graph = GraphLoader.edgeListFile(sc, "graphdata/day11.tsv").cache()

    println("\n\n~~~~~~~~~ Confirm Vertices Internal of graph ")
    graph.vertices.collect.foreach(println(_))
    // (4,1)
    // (6,1)
    // (2,1)
    // (1,1)
    // (3,1)
    // (7,1)
    // (5,1)

    println("\n\n~~~~~~~~~ Confirm Edges Internal of graph ")
    graph.edges.collect.foreach(println(_))
    // Edge(1,2,1)
    // Edge(2,3,1)
    // Edge(3,1,1)
    // Edge(4,5,1)
    // Edge(5,6,1)
    // Edge(6,7,1)
    // 1,2,3相连，4,5,6,7相连

    // 取连通图，连通图以图中最小Id作为label给图中顶点打属性
    val cc:Graph[Long, Int] = graph.connectedComponents

    println("\n\n~~~~~~~~~ Confirm Vertices Connected Components ")
    cc.vertices.collect.foreach(println(_))
    // (4,4)
    // (6,4)
    // (2,1)
    // (1,1)
    // (3,1)
    // (7,4)
    // (5,4)

    // 取出id为2的顶点的label
    val cc_label_of_vid_2:Long = cc.vertices.filter{case (id, label) => id == 2}.first._2

    println("\n\n~~~~~~~~~ Confirm Connected Components Label of Vertex id 2")
    println(cc_label_of_vid_2)
    // 1

    // 取出相同类标的顶点
    val vertices_connected_with_vid_2:RDD[(Long, Long)] = cc.vertices.filter{case (id, label) => label == cc_label_of_vid_2}

    println("\n\n~~~~~~~~~ Confirm vertices_connected_with_vid_2")
    vertices_connected_with_vid_2.collect.foreach(println(_))
    // (2,1)
    // (1,1)
    // (3,1)

    val vids_connected_with_vid_2:RDD[Long] = vertices_connected_with_vid_2.map(v => v._1)
    println("\n\n~~~~~~~~~ Confirm vids_connected_with_vid_2")
    vids_connected_with_vid_2.collect.foreach(println(_))
    // 2
    // 1
    // 3

    val vids_list:Array[Long] = vids_connected_with_vid_2.collect
    // 取出子图
    val graph_include_vid_2 = graph.subgraph(vpred = (vid, attr) => vids_list.contains(vid))

    println("\n\n~~~~~~~~~ Confirm graph_include_vid_2 ")
    graph_include_vid_2.vertices.collect.foreach(println(_))
    // (2,1)
    // (1,1)
    // (3,1)

    sc.stop
  }

}</span>
```
