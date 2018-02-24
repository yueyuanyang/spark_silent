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
![graph](https://github.com/yueyuanyang/spark/blob/master/graph/doc/graphx_build_graph.jpg)

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

### 2.1.1 aggregateMessages Map阶段
![graph map](https://github.com/yueyuanyang/spark/blob/master/graph/doc/graphx_aggmsg_map.jpg)

从入口函数进入aggregateMessagesWithActiveSet，首先使用VertexRDD[VD]更新replicatedVertexView, 只更新其中vertexRDD中attr对象。

问：为啥更新replicatedVertexView？ 答：replicatedVertexView就是个点和边的视图，点的属性有变化，要更新边中包含的点的attr

replicatedVertexView这里对edgeRDD做mapPartitions操作，所有的操作都在每个边分区的迭代中完成。

1. 进入aggregateMessagesEdgeScan

前文中提到edge partition中包含的五个重要数据结构之一：localSrcIds, 顶点vertixId在当前分区中的索引.

1) 遍历localSrcIds, 根据其下标去localSrcIds中拿到srcId在全局local2global中的索引位，然后拿到srcId； 同理，根据下标，去localDstIds中取到local2global中的索引位, 取出dstId
有了srcId和dstId，你就可以blabla....

问： 为啥用localSrcIds的下标
答： 用localDstIds的也可以。一条边必然包含两个点:srcId, dstId

2) 发消息

看上图:

根据接口中定义的tripletFields，拿到发消息的方向: 1) 向dstId发；2) 向srcId发；3) 向两边发；4) 向其中一边发
发消息的过程就是遍历到一条边，向以srcId/dstId在本分区内的本地IDlocalId为下标的数组中添加数据，如果localId为下标数组中已经存在数据，则执行合并函数mergeMsg
每个点之间在发消息的时候是独立的，即：点单纯根据方向，向以相邻点的localId为下标的数组中插数据，互相独立，在并行上互不影响。 完事，返回消息RDDmessages: RDD[(VertexId, VD2)]
### 2.1.2 aggregateMessages Reduce阶段
![reduce](https://github.com/yueyuanyang/spark/blob/master/graph/doc/graphx_aggmsg_reduce.jpg)


因为对边上，对点发消息，所以在reduce阶段主要是VertexRDD的菜。

入口(Graphmpl 260行)： vertices.aggregateUsingIndex(preAgg, mergeMsg)

收到messages: RDD[(VertexId, VD2)]消息RDD，开始：

1. 对messages做shuffled分区，分区器使用VertexRDD的partitioner。

因为VertexRDD的partitioner根据点VertexID做分区，所以vertexId->消息分区后的pid根VertextRDD完全相同，这样用zipPartitions高效的合并两个分区的数据

2. 根据对等合并attr, 聚合函数使用API传入的mergeMsg函数

**小技巧：遍历节点时，遍历messagePartition。并不是每个节点都会收到消息，所以messagePartition集合最小，所以速度会快。遍历小集合取大集合的数据。
前文提到根据routingTables路由表生成VertexRDD的vertexPartitions时, vertexPartitions中重新封装了ShippableVertexPartition对象，其定义为：**
```
ShippableVertexPartition[VD: ClassTag](
val index: VertexIdToIndexMap,
val values: Array[VD],
val mask: BitSet,
val routingTable: RoutingTablePartition)
```
最后生成对象： new ShippableVertexPartition(map.keySet, map._values, map.keySet.getBitSet, routingTable)

所以这里用到的index就是map.keySet, map就是映射vertexId->attr

index: map.keySet, hashSet, vertexId->下标

values: map._valuers, Array[Int], 根据下标保存attr。

so so，根据vetexId从index中取到其下标，再根据下标，从values中取到attr，存在attr就用API传入的函数mergeMsg合并属性attr; 不存在就直接赋值。

最后得到的是收到消息的VertexRDD

到这里，整个map/reduce过程就完成了。
![golde](https://github.com/yueyuanyang/spark/blob/master/graph/doc/graphx_global.jpg)




