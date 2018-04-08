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

**安装环境**

| 产品 | 版本 | 环境 
| - | :-: | -: 
| titan-1.0.0-hadoop2 | 1.0.0|
| jdk | 1.8 |
| hbase-1.2.3 | 1.0* |
| elasticsearch-1.5.2 | 1.0* |

搭建环境(hbase+es+titan1.0.0-hadoop2为例）

下载安装

- 下载安装titan-1.0-hadoop1http://s3.thinkaurelius.com/downloads/titan/titan-1.0.0-hadoop2.zip
- 下载安装最新JDKhttp://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html，设置环境变量PATH和CLASSPATH
- elasticsearch下载1.x版本https://www.elastic.co/downloads/past-releases能够兼容1.x的hbase
- 下载hbase1.x版本https://mirrors.tuna.tsinghua.edu.cn/apache/hbase/

**步骤一**

1) 删除titan-1.0.0-hadoop2/lib目录下的hadoop-core-1.2.1.jar
2) 拷贝titan-hadoop-1.0.0.jar，titan-hadoop-core-1.0.0.jar到titan-1.0.0-hadoop2/lib目录
3) 可以使用命令$bin/titan.sh start测试是否安装成功

**步骤二**

编辑conf/titan-hbase-es.properties文件
```
storage.backend=hbase  
storage.hostname=host-10,host-11,host-12 // zookeeper 地址
index.search.backend=elasticsearch  
index.search.hostname=host-10 // 节点IP 
index.search.elasticsearch.client-only=true 
#index.search.elasticsearch.interface=TRANSPORT_CLIENT   
#index.search.elasticsearch.cluster-name=data 

```

**测试**

```
$bin/gremlin.sh  
gremlin> graph=TitanFactory.open('conf/titan-hbase-es.properties')  
==>standardtitangraph[hbase:[host-115]]  
gremlin> GraphOfTheGodsFactory.load(graph)  
==>null  
gremlin> g = graph.traversal()  
graphtraversalsource[standardtitangraph[hbase:[host-115]], standard]  
gremlin> saturn=g.V().has('name','saturn').next()  
==>v[256]
gremlin> g.V(saturn).valueMap()  
==>[name:[saturn],age:[10000]]  
gremlin> g.V(saturn).in('father').in('father').values('name')  
==>hercules  
gremlin> g.E().has('place',geoWithin(Geoshape.circle(37.97,23.72,50)))  
==>e[a9x-co8-9hx-39s][16424-battled->4240]  
==>e[9vp-co8-9hx-9ns][16424-battled->12520]  
gremlin> g.E().has('place',geoWithin(Geoshape.circle(37.97,23.72,50))).as('source').inV().as('god2').select('source').outV().as('god1')
.select('god1','god2').by('name')  
==>[god1:hercules,god2:hydra]  
==>[god1:hercules,god2:nemean]  
gremlin> hercules=g.V(saturn).repeat(__.in('father')).times(2).next()  
==>v[1536]  
gremlin> g.V(hercules).out('father','mother')  
==>v[1024]  
==>v[1792]  
gremlin> g.V(hercules).out('father','mother').values('name')  
==>jupiter  
==>alcmene  
gremlin> g.V(hercules).out('father','mother').label()  
==>god  
==>human  
gremlin> hercules.label()  
==>demigod  
gremlin> g.V(hercules).out('father','mother')  
==>v[1024]  
==>v[1792]  
gremlin> g.V(hercules).out('father','mother').values('name')  
==>jupiter  
==>alcmene
gremlin>g.V(hercules).out('father','mother').label()  
==>god  
==>human  
gremlin> hercules.label()  
==>demigod  
gremlin> g.V(hercules).outE('battled').has('time',gt(1)).inV().values('name').toString()
==>[GraphStep([v[24744]],vertex),VertexStep(OUT,[battled],edge),HasStep([time.gt(1)]),EdgeVertexStep(IN),PropertiesStep  
gremlin> pluto=g.V().has('name','pluto').next()  
==>v[2048]  
gremlin> g.V(pluto).out('lives').in('lives').values('name')  
==>pluto  
==>cerberus  
gremlin>// pluto can't be his own cohabitant  
gremlin> g.V(pluto).out('lives').in('lives').where(is(neq(pluto))).values('name')  
==>cerberus
gremlin> g.V(pluto).as('x').out('lives').in('lives').where(neq('x')).values('name')  
==>cerberus  
gremlin>// where do pluto's brothers live?  
gremlin> g.V(pluto).out('brother').out('lives').values('name')  
==>sky  
==>sea  
gremlin>// which brother lives in which place?  
gremlin> g.V(pluto).out('brother').as('god').out('lives').as('place').select()  
==>[god:v[1024],place:v[512]]  
==>[god:v[1280],place:v[768]]  
gremlin>// what is the name of the brother and the name of the place?  
gremlin>g.V(pluto).out('brother').as('god').out('lives').as('place').select().by('name')  
==>[god:jupiter,place:sky]  
==>[god:neptune,place:sea]  
gremlin> g.V(pluto).outE('lives').values('reason')  
==>nofearofdeathgremlin>g.E().has('reason',textContains('loves'))  
==>e[6xs-sg-m51-e8][1024-lives->512]  
==>e[70g-zk-m51-lc][1280-lives->768]  
gremlin> g.E().has('reason',textContains('loves')).as('source').values('reason').as('reason')
.select('source').outV().values('name').as('god').select('source').inV().values('name').as('thing').select('god','reason','thing')  
==>[god:neptune,reason:loveswaves,thing:sea]  
==>[god:jupiter,reason:lovesfreshbreezes,thing:sky]  

```




