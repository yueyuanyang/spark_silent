### SVM 分类算法

```
import com.huaban.analysis.jieba.JiebaSegmenter
import com.huaban.analysis.jieba.JiebaSegmenter.SegMode
import org.apache.spark.mllib.feature.{HashingTF, IDF}
import org.apache.spark.mllib.regression.LabeledPoint
import org.apache.spark.{SparkConf, SparkContext}
import scala.collection.mutable
import org.apache.spark.mllib.classification.SVMWithSGD
import org.apache.spark.mllib.evaluation.BinaryClassificationMetrics
import scala.collection.JavaConversions._

object SVMWithSGDForComment {
  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setAppName("SVMWithSGDExample")
    val sc = new SparkContext(conf)
    if (args.length < 4) {
      println("Please input 4 args: datafile numIterations train_percent(0.6)!")
      System.exit(1)
    }
    val datafile =   args.head.toString
    val numIterations = Integer.parseInt(args(1))
    val train_percent = args(2).toDouble
    val test_percent = 1.0 - train_percent
    val model_file = args(3)
//    数据预处理
//    数据载入到 Spark 系统，抽象成为一个 RDD
    val originData = sc.textFile(datafile)
//    distinct 方法对数据去重
    val originDistinctData = originData.distinct()
//    将每一行文本变成一个 list，并且只保留长度大于2 的数据。
    val rateDocument = originDistinctData.map(line => line.split('\t')).filter(line => line.length > 2)

//    打五分的毫无疑问是好评；考虑到不同人对于评分的不同偏好，对于打四分、三分的数据，本文无法得知它是好评还是坏评；对于三分以下的是坏评
    val fiveRateDocument = rateDocument.filter(arrline => arrline(0).equalsIgnoreCase("5"))
    System.out.println("************************5 score num:" +fiveRateDocument.count())

    val fourRateDocument = rateDocument.filter(arrline => arrline(0).equalsIgnoreCase("4"))
    val threeRateDocument = rateDocument.filter(arrline => arrline(0).equalsIgnoreCase("3"))
    val twoRateDocument = rateDocument.filter(arrline => arrline(0).equalsIgnoreCase("2"))
    val oneRateDocument = rateDocument.filter(arrline => arrline(0).equalsIgnoreCase("1"))
    
//    合并负样本数据 1.2星
    val negRateDocument = oneRateDocument.union(twoRateDocument)
    negRateDocument.repartition(1)
//    生̧成训练数̧据集
    val posRateDocument = sc.parallelize(fiveRateDocument.take(negRateDocument.count().toInt)).repartition(1)
    val allRateDocument = negRateDocument.union(posRateDocument)
    allRateDocument.repartition(1)
    val rate = allRateDocument.map(s => ReduceRate(s(0)))
    val document = allRateDocument.map(s => s(1))
//    文本的向量表示和文本特征提取  每一句评论转化为词
    val words = document.map(sentence => cut_for_calc(sentence)).map(line => line.split("/").toSeq)
    words.foreach(seq =>{
        val arr = seq.toList
        val line = new StringBuilder
        arr.foreach(item => {
          line ++= (item +' ')
        })
    })
 //    训练词频矩阵
    val hashingTF = new HashingTF()
    val tf = hashingTF.transform(words)
    tf.cache()

//    计算 TF-IDF 矩阵
    val idf = new IDF().fit(tf)
    val tfidf = idf.transform(tf)

//    生成训练集和测试集
    val zipped = rate.zip(tfidf)
    val data = zipped.map(tuple => LabeledPoint(tuple._1,tuple._2))

    val splits = data.randomSplit(Array(train_percent, test_percent), seed = 11L)
    val training = splits(0).cache()

    val test = splits(1)
    val model = SVMWithSGD.train(training, numIterations)
    model.clearThreshold()

    // Compute raw scores on the test set.
    val topicsArray = new mutable.MutableList[String]

    val scoreAndLabels = test.map { point =>
      val score = model.predict(point.features)
      (score, point.label)
    }
    scoreAndLabels.coalesce(1).saveAsTextFile("file:///data/1/usr/local/services/spark/helh/comment_test_predic/")

    // Get evaluation metrics.
    val metrics = new BinaryClassificationMetrics(scoreAndLabels)
    val auROC = metrics.areaUnderROC()
    println("Area under ROC = " + auROC)
  }

  def ReduceRate(rate_str:String):Int = {
    if (rate_str.toInt > 4)
      return 1
    else
      return 0;
  }
  def cut_for_calc(str:String):String = {
    val jieba = new JiebaSegmenter();
    val lword_info = jieba.process(str, SegMode.SEARCH);
    lword_info.map(item => item.word).mkString("/")
  }
}
```
