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

