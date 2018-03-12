## spark 分词

### Spark下四种中文分词工具使用

- hanLP
- ansj
- jieba
- fudannlp

本文使用maven 导入四种分词工具

```
 <dependency>
        <groupId>org.ansj</groupId>
        <artifactId>ansj_seg</artifactId>
        <version>5.1.3</version>
    </dependency>
    <dependency>
        <groupId>com.hankcs</groupId>
        <artifactId>hanlp</artifactId>
        <version>portable-1.3.4</version>
    </dependency>
    <dependency>
        <groupId>com.huaban</groupId>
        <artifactId>jieba-analysis</artifactId>
        <version>1.0.2</version>
    </dependency>
```
fudannlp github地址：https://github.com/FudanNLP/fnlp 

模型文件国内网盘地址：https://pan.baidu.com/disk/home#list/vmode=list&path=%2F%E6%88%91%E7%9A%84%E8%B5%84%E6%BA%90%2Ffnlp%E7%BD%91%E7%9B%98%E9%95%9C%E5%83%8F

### spark下四种具体使用

>  jieba 分词使用

```
import com.huaban.analysis.jieba.JiebaSegmenter
import org.apache.spark.{SparkConf, SparkContext}
object WordSp {
  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setAppName("input").setMaster("local[*]")
    val sc = new SparkContext(conf)
    val rdd=sc.textFile("C:\\Users\\Administrator\\Desktop\\icwb2-data\\testing\\pku_test.utf8")
      .map { x =>
        var str = if (x.length > 0)
          new JiebaSegmenter().sentenceProcess(x)
        str.toString
      }.top(50).foreach(println)
  }
}
```

