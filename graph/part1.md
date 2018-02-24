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
