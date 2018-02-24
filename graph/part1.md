## spark 基础知识
### 1.构建Graph
graphx的Graph对象是用户操作图的入口, 它包含了边(edge)和顶点(vertices)两部分. 边由点组成，所以, 所有的边中包含的点的就是点的全集。但是这个全集包含重复的点, 去重复后就是VertexRDD点和边都包含一个attr属性，可以携带额外信息
#### 1.1 构建一个图的方法
1.根据边构建图(Graph.fromEdges)
```
def fromEdges[VD: ClassTag, ED: ClassTag](
      edges: RDD[Edge[ED]],
      defaultValue: VD,
      edgeStorageLevel: StorageLevel = StorageLevel.MEMORY_ONLY,
      vertexStorageLevel: StorageLevel = StorageLevel.MEMORY_ONLY): Graph[VD, ED]
```
2.根据边的两个点元数据构建(Graph.fromEdgeTuples)
```
def fromEdgeTuples[VD: ClassTag](
  rawEdges: RDD[(VertexId, VertexId)],
  defaultValue: VD,
  uniqueEdges: Option[PartitionStrategy] = None,
  edgeStorageLevel: StorageLevel = StorageLevel.MEMORY_ONLY,
  vertexStorageLevel: StorageLevel = StorageLevel.MEMORY_ONLY): Graph[VD, Int] =
```
#### 1.2 第一步：构建边EdgeRDD
![EdgeRDD](https://github.com/yueyuanyang/spark/blob/master/graph/doc/graphx_build_edge.jpg)
#### 1.2.1 加载边的文本信息
从持久化系统（HDFS）中加载边的文本信息，按行处理生成tuple, 即(srcId, dstId)

api：
```
    val rawEdgesRdd: RDD[(Long, Long)] = sc.textFile(input, partitionNum).filter(s => s != "0,0").repartition(partitionNum).map {
      case line =>
        val ss = line.split(",")
        val src = ss(0).toLong
        val dst = ss(1).toLong
        if (src < dst)
          (src, dst)
        else
          (dst, src)
    }.distinct()
```
数据形如：
```
107,109
108,109
110,111
110,112
111,112
113,114
115,116
117,79
117,118
79,118
```
#### 1.2.2	第二步：初步生成Graph
入口：Graph.fromEdgeTuples(rawEdgesRdd)
元数据为,分割的两个点ID，把元数据映射成Edge(srcId, dstId, attr)对象, attr默认为null。这样元数据就构建成了RDD[Edge[ED]]

RDD[Edge[ED]]要进一步转化成EdgeRDDImpl[ED, VD]
首先遍历RDD[Edge[ED]]的分区partitions，对分区内的边重排序new Sorter(Edge.edgeArraySortDataFormat[ED]).sort(edgeArray, 0, edgeArray.length, Edge.lexicographicOrdering)即：按照srcId从小到大排序。

问：为何要重排序？
答：为了遍历时顺序访问。采用数组而不是Map，数组是连续的内存单元，具有原子性，避免了Map的hash问题，访问速度快

填充localSrcIds,localDstIds, data, index, global2local, local2global, vertexAttrs

数组localSrcIds,localDstIds中保存的是经过global2local.changeValue(srcId/dstId)转变的本地索引，即：localSrcIds、localDstIds数组下标对应于分区元素，数组中保存的索引位可以定位到local2global中查到具体的VertexId

global2local是spark私有的Map数据结构GraphXPrimitiveKeyOpenHashMap, 保存vertextId和本地索引的映射关系。global2local中包含当前partition所有srcId、dstId与本地索引的映射关系。

data就是当前分区的attr属性数组

index索引最有意思，按照srcId重排序的边数据, 会看到相同的srcId对应了不同的dstId, 见图中index desc部分。index中记录的是相同srcId中第一个出现的srcId与其下标。

local2global记录的是所有的VertexId信息的数组。形如：srcId,dstId,srcId,dstId,srcId,dstId,srcId,dstId。其中会包含相同的ID。即：当前分区所有vertextId的顺序实际值
```
＃	用途：
＃ 根据本地下标取VertexId
localSrcIds/localDstIds -> index -> local2global -> VertexId

＃	根据VertexId取本地下标，取属性
VertexId -> global2local -> index -> data -> attr object
```
### v1.3 第二步：构建顶点(VertexRDD)
![vertexRDD](https://github.com/yueyuanyang/spark/blob/master/graph/doc/graphx_build_vertex.jpg)
入口：GraphImpl365行。 val vertices = VertexRDD.fromEdges(edgesCached, edgesCached.partitions.size, defaultVertexAttr).withTargetStorageLevel(vertexStorageLevel)

根据边EdgeRDD[ED, VD]构建出点VertexRDD, 点是孤岛，不像边一样保存点与点之间的关系。点只保存属性attr。所以需要对拿到的点做分区。

为了能通过点找到边，每个点需要保存点所在到边信息即分区Id(pid)，这些新保存在路由表RoutingTablePartition中。

**构建的过程：**

1.创建路由表

根据EdgeRDD，map其分区，对edge partition中的数据转换成RoutingTableMessage数据结构。

特别激动的是: 为节省内存，把edgePartitionId和一个标志位通过一个32位的int表示。

如图所示，RoutingTableMessage是自定义的类型类, 一个包含vid和int的tuple(VertexId, Int)。 int的32~31位表示一个标志位,01: isSrcId 10: isDstId。30～0位表示边分区ID。赞这种做法，目测作者是山西人。

RoutingTableMessage想表达这样的信息：一个顶点ID，不管未来你到天涯海角，请带着你女朋友Edge的地址: edge分区ID。并且带着一个标志你在女友心中的位置是：01是左边isSrcId，10是右边isDstId。这样就算你流浪到非洲，也能找到小女友约会...blabla...

2. 根据路由表生成分区对象vertexPartitions

1) 上（1）中生成的消息路由表信息，重新分区，分区数目根edge分区数保持一致。

2) 在新分区中，map分区中的每条数据，从RoutingTableMessage解出数据：vid, edge pid, isSrcId/isDstId。这个三个数据项重新封装到三个数据结构中:pid2vid,srcFlags,dstFlags

3) 有意思的地方来了，想一下，shuffle以后新分区中的点来自于之前edge不同分区，那么一个点要找到边，就需要先确定边的分区号pid, 然后在确定的edge分区中确定是srcId还是dstId, 这样就找到了边。
```
val pid2vid = Array.fill(numEdgePartitions)(new PrimitiveVector[VertexId])
val srcFlags = Array.fill(numEdgePartitions)(new PrimitiveVector[Boolean])
val dstFlags = Array.fill(numEdgePartitions)(new PrimitiveVector[Boolean])
```
上面表达的是：当前vertex分区中点在edge分区中的分布。新分区中保存这样的记录(vids.trim().array, toBitSet(srcFlags(pid)), toBitSet(dstFlags(pid))) vid, srcFlag, dstFlag, flag通过BitSet存储，很省。

如此就生成了vertex的路由表routingTables

4) 生成ShippableVertexPartition

根据上面routingTables, 重新封装路由表里的数据结构为：ShippableVertexPartition

ShippableVertexPartition会合并相同重复点的属性attr对象，补全缺失的attr对象。

关键是：根据vertexId生成map:GraphXPrimitiveKeyOpenHashMap，这个map跟边中的global2local是不是很相似？这个map根据long vertxId生成下标索引，目测：相同的点会有相同的下标。// todo..
#### 1.4 第三步 生成Graph对象［finished］
把上述edgeRDD和vertexRDD拿过来组成Graph
```
new GraphImpl(vertices, new ReplicatedVertexView(edges.asInstanceOf[EdgeRDDImpl[ED, VD]]))
```
3. 创建VertexRDDImpl对象
new VertexRDDImpl(vertexPartitions)，这就完事了
## 2. 常用函数分析
**下面分析一下常用的graph函数aggregateMessages**
### 2.1 aggregateMessages
aggregateMessages是Graphx最重要的API，1.2版本添加的新函数，用于替换mapReduceTriplets。目前mapReduceTriplets最终也是使用兼容的aggregateMessages

据说改用aggregateMessages后，性能提升30%。

它主要功能是向邻边发消息，合并邻边收到的消息，返回messageRDD

aggregateMessages的接口如下：
```
def aggregateMessages[A: ClassTag](
      sendMsg: EdgeContext[VD, ED, A] => Unit,
      mergeMsg: (A, A) => A,
      tripletFields: TripletFields = TripletFields.All)
    : VertexRDD[A] = {
    aggregateMessagesWithActiveSet(sendMsg, mergeMsg, tripletFields, None)
  }

```
sendMsg： 发消息函数
```
private def sendMsg(ctx: EdgeContext[KCoreVertex, Int, Map[Int, Int]]): Unit = {
	ctx.sendToDst(Map(ctx.srcAttr.preKCore -> -1, ctx.srcAttr.curKCore -> 1))
	ctx.sendToSrc(Map(ctx.dstAttr.preKCore -> -1, ctx.dstAttr.curKCore -> 1))
      }
```
mergeMsg：合并消息函数。用于Map阶段，每个edge分区中每个点收到的消息合并，以及reduce阶段，合并不同分区的消息。合并vertexId相同的消息
tripletFields：定义发消息的方向







