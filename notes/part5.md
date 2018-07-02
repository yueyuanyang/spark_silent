### CDH中如何升级Spark

> 公司平时使用的CDH版本的hadoop生态，spark任务是基于yarn来管理的，而不是基于原生的spark master slave集群管理

### 因此任务的大致启动模式是：

#### 如果是Cluster模式：
- 1.A节点启动Spark-submit，这个程序即为client，client连接Resource Manager
- 2.Resource Manager指定一个Node Manager创建AppMaster，这个AppMaster就是Driver
- 3.AppMaster向Resource Manager申请资源创建Spark的Excutor
- 4.Excutor向Driver(AppMaster)报告程序结果

#### 如果是Client模式：
- 1.A节点启动Spark-submit，这个程序就是client，此时直接创建Driver。
- 2.连接Resource Manager创建AppMaster
- 3.Driver向AppMaster申请创建Excutor，AppMaster再跟Resource Manager申请资源创建Excutor
- 4.Excutor向Driver(Client)报告程序结果

## 那么这种环境下如何升级Spark呢？

通过上面的过程分析，可以知道，Spark版本存在两个地方：

- 一个是A节点提交Spark-submit的程序必须是2.3.0版本的；
- 另一个是Yarn使用的lib必须是2.3.0版本的。

虽然暂时还屡不清楚来龙去脉，但是跟着过一遍吧！

#### 第一步，在A节点下载spark2.3的jar
```
[xxx@hnode10 app]$ ls -l
total 628168
drwxrwxr-x  6 hdfs hdfs      4096 Jan  9 10:35 akita
-rw-r--r--  1 hdfs hdfs  18573432 Jan  9 10:34 akita-release.tar.gz
lrwxrwxrwx  1 hdfs hdfs        46 Jan  2 09:37 canal -> /var/lib/hadoop-hdfs/app/canal.deployer-1.0.25
drwxrwxr-x  6 hdfs hdfs      4096 Jan  2 09:36 canal.deployer-1.0.25
drwxrwxr-x  4 hdfs hdfs      4096 May 31 09:11 hadoop
lrwxrwxrwx  1 root root        50 Jun  5 12:34 spark -> /var/lib/hadoop-hdfs/app/spark-2.2.0-bin-hadoop2.6
drwxr-xr-x 14 hdfs hdfs      4096 Nov  9  2017 spark-2.1.1-bin-hadoop2.6
-rw-r--r--  1 hdfs hdfs 198804211 Oct 23  2017 spark-2.1.1-bin-hadoop2.6.tgz
drwxr-xr-x 13 hdfs hdfs      4096 Jun  5 12:33 spark-2.2.0-bin-hadoop2.6
-rw-rw-r--  1 hdfs hdfs 201706782 Jul 11  2017 spark-2.2.0-bin-hadoop2.6.tgz
drwxr-xr-x 13 hdfs hdfs      4096 Feb 23 03:46 spark-2.3.0-bin-hadoop2.6
-rw-rw-r--  1 hdfs hdfs 224121109 Feb 23 03:54 spark-2.3.0-bin-hadoop2.6.tgz
lrwxrwxrwx  1 root root        25 Jun  6 09:04 spark23 -> spark-2.3.0-bin-hadoop2.6
```
#### 第二步，修改配置文件和启动脚本

解压后，创建一个新的软连接 spark23到对应的目录：
```
ln -s /var/lib/hadoop-hdfs/app/spark-2.3.0-bin-hadoop2.6 spark23
```

然后配置对应的启动脚本：

```
[xxx@hnode10 bin]$ ls -l
total 9588
-rwxr-xr-x 1 hdfs hdfs    2991 Oct 23  2017 spark2-shell
-rwxr-xr-x 1 hdfs hdfs    1013 Oct 23  2017 spark2-submit
-rwxr-xr-x 1 root root    2993 Jun  6 17:39 spark23-shell
-rwxr-xr-x 1 root root    1015 Jun  6 17:41 spark23-submit
```

在spark23-submit中修改SPARK_HOME

```
export SPARK2_HOME=/var/lib/hadoop-hdfs/app/spark23
exec "${SPARK2_HOME}"/bin/spark-class org.apache.spark.deploy.SparkSubmit "$@"
```

在spark23-shell中修改SPARK_HOME

```
cygwin=false
case "$(uname)" in
  CYGWIN*) cygwin=true;;
esac

# Enter posix mode for bash
set -o posix

export SPARK2_HOME=/var/lib/hadoop-hdfs/app/spark23
....
```

修改Spark2.3中的配置文件spark-defaults.conf

```
spark.yarn.jars  hdfs://nameservice1/app/spark23/lib/*.jar
spark.history.fs.logDirectory  hdfs://nameservice1/user/spark/applicationHistory
```
其中spark.yarn.jars指定了yarn使用的spark jar包目录。

#### 第三步，在hdfs中上传yarn使用的lib


最后，找一个hello world启动下试试吧~










