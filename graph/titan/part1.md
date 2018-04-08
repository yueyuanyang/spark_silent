## 图数据库(titan) —— 安装及配置

### 概念

Titan 是一个在服务器集群搭建的分布式的图形数据库，特别为存储和处理大规模图形而优化。集群很容易扩展以支持更大的数据集，Titan有一个很好的插件式性能，这个性能让它搭建在一些成熟的数据库技术上像 Apache Cassandra、Apache HBase、 Oracle BerkeleyDB。插件式索引架构可以整合 ElasticSearch 和Lucene技术。内置实现 Blueprints  graph API，支持 TinkerPop所有的技术。

### 特性
1、支持不同的分布式存储层
- Apache Cassandra (distributed)
- Apache HBase (distributed)
- Oracle BerkeleyDB (local)
- Persistit (local)

2、可以更加数据集的大小和用户基数弹性扩展

3、分布式存储的复制，高容错性

4、支持很多字符集和热备份

5、支持 ACID 和 eventual consistency（最终一致性）

6、支持的索引

 - ElasticSearch
 - Apache Lucene
 
7、内置实现 TinkerPop graph API

 - Gremlin graph query language
 - Frames object-to-graph mapper
 - Rexster graph server
 - Blueprints standard graph API

### 安装部署

| 产品 | 版本 | 环境 
| - | :-: | -: 
| titan-1.0.0-hadoop2 | 1.0.0|
| hbase-1.2.3 | 1.0* |
| elasticsearch-1.5.2 | 1.0* |
