## Spark算子：RDD行动Action操作(5)–saveAsTextFile、saveAsSequenceFile、saveAsObjectFile、saveAsHadoopFile、saveAsHadoopDataset、saveAsNewAPIHadoopFile、saveAsNewAPIHadoopDataset

### saveAsTextFile

def saveAsTextFile(path: String): Unit

def saveAsTextFile(path: String, codec: Class[_ <: CompressionCodec]): Unit

saveAsTextFile用于将RDD以文本文件的格式存储到文件系统中。

codec参数可以指定压缩的类名。

```
var rdd1 = sc.makeRDD(1 to 10,2)
scala> rdd1.saveAsTextFile("hdfs://cdh5/tmp/lxw1234.com/") //保存到HDFS
hadoop fs -ls /tmp/lxw1234.com
Found 2 items
-rw-r--r--   2 lxw1234 supergroup        0 2015-07-10 09:15 /tmp/lxw1234.com/_SUCCESS
-rw-r--r--   2 lxw1234 supergroup        21 2015-07-10 09:15 /tmp/lxw1234.com/part-00000
 
hadoop fs -cat /tmp/lxw1234.com/part-00000
1
2
3

```

注意：如果使用rdd1.saveAsTextFile(“file:///tmp/lxw1234.com”) 将文件保存到本地文件系统，那么只会保存在Executor所在机器的本地目录。

指定压缩格式保存

```
rdd1.saveAsTextFile("hdfs://cdh5/tmp/lxw1234.com/",classOf[com.hadoop.compression.lzo.LzopCodec])
 
hadoop fs -ls /tmp/lxw1234.com
-rw-r--r--   2 lxw1234 supergroup    0 2015-07-10 09:20 /tmp/lxw1234.com/_SUCCESS
-rw-r--r--   2 lxw1234 supergroup    71 2015-07-10 09:20 /tmp/lxw1234.com/part-00000.lzo
 
hadoop fs -text /tmp/lxw1234.com/part-00000.lzo
1
2
3

```
### saveAsSequenceFile

saveAsSequenceFile用于将RDD以SequenceFile的文件格式保存到HDFS上。

用法同saveAsTextFile。

```
rdd1.saveAsSequenceFile("hdfs://cdh5/tmp/lxw1234.com/") 
```

### saveAsObjectFile

def saveAsObjectFile(path: String): Unit

saveAsObjectFile用于将RDD中的元素序列化成对象，存储到文件中。

对于HDFS，默认采用SequenceFile保存。

```
var rdd1 = sc.makeRDD(1 to 10,2)
rdd1.saveAsObjectFile("hdfs://cdh5/tmp/lxw1234.com/")
 
hadoop fs -cat /tmp/lxw1234.com/part-00000
SEQ !org.apache.hadoop.io.NullWritable"org.apache.hadoop.io.BytesWritableT
```

### saveAsHadoopFile

def saveAsHadoopFile(path: String, keyClass: Class[_], valueClass: Class[_], outputFormatClass: Class[_ <: OutputFormat[_, _]], codec: Class[_ <: CompressionCodec]): Unit

def saveAsHadoopFile(path: String, keyClass: Class[_], valueClass: Class[_], outputFormatClass: Class[_ <: OutputFormat[_, _]], conf: JobConf = …, codec: Option[Class[_ <: CompressionCodec]] = None): Unit

saveAsHadoopFile是将RDD存储在HDFS上的文件中，支持老版本Hadoop API。

可以指定outputKeyClass、outputValueClass以及压缩格式。

每个分区输出一个文件。

```
var rdd1 = sc.makeRDD(Array(("A",2),("A",1),("B",6),("B",3),("B",7)))
 
import org.apache.hadoop.mapred.TextOutputFormat
import org.apache.hadoop.io.Text
import org.apache.hadoop.io.IntWritable
 
rdd1.saveAsHadoopFile("/tmp/lxw1234.com/",classOf[Text],classOf[IntWritable],classOf[TextOutputFormat[Text,IntWritable]])
 
rdd1.saveAsHadoopFile("/tmp/lxw1234.com/",classOf[Text],classOf[IntWritable],classOf[TextOutputFormat[Text,IntWritable]],
                      classOf[com.hadoop.compression.lzo.LzopCodec])
```

### saveAsHadoopDataset

def saveAsHadoopDataset(conf: JobConf): Unit

saveAsHadoopDataset用于将RDD保存到除了HDFS的其他存储中，比如HBase。

在JobConf中，通常需要关注或者设置五个参数：

文件的保存路径、key值的class类型、value值的class类型、RDD的输出格式(OutputFormat)、以及压缩相关的参数。

**使用saveAsHadoopDataset将RDD保存到HDFS中**

```
import org.apache.spark.SparkConf
import org.apache.spark.SparkContext
import SparkContext._
import org.apache.hadoop.mapred.TextOutputFormat
import org.apache.hadoop.io.Text
import org.apache.hadoop.io.IntWritable
import org.apache.hadoop.mapred.JobConf
 
 
 
var rdd1 = sc.makeRDD(Array(("A",2),("A",1),("B",6),("B",3),("B",7)))
var jobConf = new JobConf()
jobConf.setOutputFormat(classOf[TextOutputFormat[Text,IntWritable]])
jobConf.setOutputKeyClass(classOf[Text])
jobConf.setOutputValueClass(classOf[IntWritable])
jobConf.set("mapred.output.dir","/tmp/lxw1234/")
rdd1.saveAsHadoopDataset(jobConf)
 
结果：
hadoop fs -cat /tmp/lxw1234/part-00000
A       2
A       1
hadoop fs -cat /tmp/lxw1234/part-00001
B       6
B       3
B       7
```

保存数据到HBASE

HBase建表：
```
create ‘lxw1234′,{NAME => ‘f1′,VERSIONS => 1},{NAME => ‘f2′,VERSIONS => 1},{NAME => ‘f3′,VERSIONS => 1}

import org.apache.spark.SparkConf
import org.apache.spark.SparkContext
import SparkContext._
import org.apache.hadoop.mapred.TextOutputFormat
import org.apache.hadoop.io.Text
import org.apache.hadoop.io.IntWritable
import org.apache.hadoop.mapred.JobConf
import org.apache.hadoop.hbase.HBaseConfiguration
import org.apache.hadoop.hbase.mapred.TableOutputFormat
import org.apache.hadoop.hbase.client.Put
import org.apache.hadoop.hbase.util.Bytes
import org.apache.hadoop.hbase.io.ImmutableBytesWritable
 
var conf = HBaseConfiguration.create()
    var jobConf = new JobConf(conf)
    jobConf.set("hbase.zookeeper.quorum","zkNode1,zkNode2,zkNode3")
    jobConf.set("zookeeper.znode.parent","/hbase")
    jobConf.set(TableOutputFormat.OUTPUT_TABLE,"lxw1234")
    jobConf.setOutputFormat(classOf[TableOutputFormat])
    
    var rdd1 = sc.makeRDD(Array(("A",2),("B",6),("C",7)))
    rdd1.map(x => 
      {
        var put = new Put(Bytes.toBytes(x._1))
        put.add(Bytes.toBytes("f1"), Bytes.toBytes("c1"), Bytes.toBytes(x._2))
        (new ImmutableBytesWritable,put)
      }
    ).saveAsHadoopDataset(jobConf)
 
##结果：
hbase(main):005:0> scan 'lxw1234'
ROW     COLUMN+CELL                                                                                                
 A       column=f1:c1, timestamp=1436504941187, value=\x00\x00\x00\x02                                              
 B       column=f1:c1, timestamp=1436504941187, value=\x00\x00\x00\x06                                              
 C       column=f1:c1, timestamp=1436504941187, value=\x00\x00\x00\x07                                              
3 row(s) in 0.0550 seconds
```

注意：保存到HBase，运行时候需要在SPARK_CLASSPATH中加入HBase相关的jar包。


