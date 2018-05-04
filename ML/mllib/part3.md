## Spark2.0机器学习系列之9： 聚类算法(LDA）

### 什么是LDA主题建模？

隐含狄利克雷分配（LDA，Latent Dirichlet Allocation）是一种主题模型(Topic Model，即从所收集的文档中推测主题)。 甚至可以说LDA模型现在已经成为了主题建模中的一个标准，是实践中最成功的主题模型之一。那么何谓“主题”呢？，就是诸如一篇文章、一段话、一个句子所表达的中心思想。不过从统计模型的角度来说， 我们是用一个特定的词频分布来刻画主题的，并认为一篇文章、一段话、一个句子是从一个概率模型中生成的。也就是说 在主题模型中，主题表现为一系列相关的单词，是这些单词的条件概率。形象来说，主题就是一个桶，里面装了出现概率较高的单词(参见下面的图)，这些单词与这个主题有很强的相关性。

LDA可以用来识别大规模文档集（document collection）或语料库（corpus）中潜藏的主题信息。它采用了词袋（bag of words）的方法，这种方法将每一篇文档视为一个词频向量，从而将文本信息转化为了易于建模的数字信息。但是词袋方法没有考虑词与词之间的顺序，这简化了问题的复杂性，同时也为模型的改进提供了契机。每一篇文档代表了一些主题所构成的一个概率分布，而每一个主题又代表了很多单词所构成的一个概率分布。 
LDA可以被认为是如下的一个聚类过程： 
（1）各个主题（Topics）对应于各类的“质心”，每一篇文档被视为数据集中的一个样本。 
（2）主题和文档都被认为存在一个向量空间中，这个向量空间中的每个特征向量都是词频（词袋模型） 
（3）与采用传统聚类方法中采用距离公式来衡量不同的是，LDA使用一个基于统计模型的方程，而这个统计模型揭示出这些文档都是怎么产生的。

它基于一个常识性假设：文档集合中的所有文本均共享一定数量的隐含主题。基于该假设，它将整个文档集特征化为隐含主题的集合，而每篇文本被表示为这些隐含主题的特定比例的混合。

LDA的这三位作者在原始论文中给了一个简单的例子。比如给定这几个主题：Arts、Budgets、Children、Education，在这几个主题下，可以构造生成跟主题相关的词语，如下图所示： 

表面上理解LDA比较简单，无非就是：当看到一篇文章后，我们往往喜欢推测这篇文章是如何生成的，我们可能会认为某个作者先确定这篇文章的几个主题，然后围绕这几个主题遣词造句，表达成文。 
前面说了这么多，在推导模型前，总结几条核心思想： 
（1）隐含主题，形象的说就是一个桶，里面装了出现概率较高的单词，从聚类的角度来说，各个主题（Topics）对应于各类的“质心”，主题和文档都被认为存在于同一个词频向量空间中。（2）在文档集合中的所有文本均共享一定数量的隐含主题的假设下，我们将寻找一个基于统计模型的方程。 
LDA的核心公式如下：

```
p(w|d)=∑i=1Kp(w|zk)∗p(zk|d)
``` 
d代表某篇文档，w代表某个单词，zk代表第i主题，共有K个主题。通俗的理解是：文档d以一定概率属于主题zk，即p(zk|d)，而主题zk下出现单词w的先验概率是p(w|zk)，因此在主题zk下，文档出现单词w的概率是p(w|zk)∗p(zk|d)，自然把文档在所有主题下ti:K出现单词w的概率加起来，就是文档d中出现单词w的概率p(w|d)（词频）。 
上面式子的左边，就是文档的词频，是很容易统计得到的。如果一篇文章中有很多个词语，那么就有很多个等式了。再如果我们收集了很多的文档，那么就有更多的等式了。这时候就是一个矩阵了，等式左边的矩阵是已知的，右边其实就是我们要求解的目标-与隐含主题相关，图示如下： 

Spark 代码分析、参数设置及结果评价
 SPARK中可选参数 
 （1）K：主题数量（或者说聚簇中心数量） 
 （2）maxIterations：EM算法的最大迭代次数，设置足够大的迭代次数非常重要，前期的迭代返回一些无用的（极其相似的）话题，但是继续迭代多次后结果明显改善。我们注意到这对EM算法尤其有效。，至少需要设置20次的迭代，50-100次是更合理的设置，取决于你的数据集。 
 （3）docConcentration（Dirichlet分布的参数α)：文档在主题上分布的先验参数（超参数α)。当前必须大于1，值越大，推断出的分布越平滑。默认为-1，自动设置。 
 （4）topicConcentration（Dirichlet分布的参数β)：主题在单词上的先验分布参数。当前必须大于1，值越大，推断出的分布越平滑。默认为-1，自动设置。 
 （5）checkpointInterval：检查点间隔。maxIterations很大的时候，检查点可以帮助减少shuffle文件大小并且可以帮助故障恢复。 
 SPARK中模型的评估 
（1）perplexity是一种信息理论的测量方法，b的perplexity值定义为基于b的熵的能量（b可以是一个概率分布，或者概率模型），通常用于概率模型的比较
  
  ### 详细代码注释
  
```
import org.apache.spark.sql.SparkSession
import org.apache.log4j.{Level, Logger}
import org.apache.spark.ml.clustering.LDA

object myClusters {
  def main(args:Array[String]){

    //屏蔽日志
    Logger.getLogger("org.apache.spark").setLevel(Level.ERROR)
    Logger.getLogger("org.eclipse.jetty.server").setLevel(Level.OFF)

    val warehouseLocation = "/Java/Spark/spark-warehouse"
    val spark=SparkSession
            .builder()
            .appName("myClusters")
            .master("local[4]")            
            .config("spark.sql.warehouse.dir",warehouseLocation)            
            .getOrCreate();   

    val dataset_lpa=spark.read.format("libsvm")
                          .load("/spark-2.0.0-bin-hadoop2.6/data/mllib/sample_lda_libsvm_data.txt")                    
    //------------------------------------1 模型训练-----------------------------------------
    /** 
     * k: 主题数，或者聚类中心数 
     * DocConcentration：文章分布的超参数(Dirichlet分布的参数)，必需>1.0，值越大，推断出的分布越平滑 
     * TopicConcentration：主题分布的超参数(Dirichlet分布的参数)，必需>1.0，值越大，推断出的分布越平滑 
     * MaxIterations：迭代次数，需充分迭代，至少20次以上
     * setSeed：随机种子 
     * CheckpointInterval：迭代计算时检查点的间隔 
     * Optimizer：优化计算方法，目前支持"em", "online" ，em方法更占内存，迭代次数多内存可能不够会抛出stack异常
     */  
    val lda=new LDA()
                .setK(3)
                .setTopicConcentration(3)
                .setDocConcentration(3)
                .setOptimizer("online")
                .setCheckpointInterval(10)
                .setMaxIter(100)

    val model=lda.fit(dataset_lpa)   

    /**生成的model不仅存储了推断的主题，还包括模型的评价方法。*/
    //---------------------------------2 模型评价-------------------------------------

    //模型的评价指标：ogLikelihood，logPerplexity
    //（1）根据训练集的模型分布计算的log likelihood，越大越好。
    val ll = model.logLikelihood(dataset_lpa)

    //（2）Perplexity评估，越小越好
    val lp = model.logPerplexity(dataset_lpa)

    println(s"The lower bound on the log likelihood of the entire corpus: $ll")
    println(s"The upper bound bound on perplexity: $lp")

    //---------------------------------3 模型及描述------------------------------
    //模型通过describeTopics、topicsMatrix来描述 

    //（1）描述各个主题最终的前maxTermsPerTopic个词语（最重要的词向量）及其权重
    val topics=model.describeTopics(maxTermsPerTopic=2)
    println("The topics described by their top-weighted terms:")
    topics.show(false)


    /**主题    主题包含最重要的词语序号                     各词语的权重
        +-----+-------------+------------------------------------------+
        |topic|termIndices  |termWeights                               |
        +-----+-------------+------------------------------------------+
        |0    |[5, 4, 0, 1] |[0.21169509638828377, 0.19142090510443274]|
        |1    |[5, 6, 1, 2] |[0.12521929515791688, 0.10175547561034966]|
        |2    |[3, 10, 6, 9]|[0.19885345685860667, 0.18794498802657686]|
        +-----+-------------+------------------------------------------+
     */    

    //（2） topicsMatrix: 主题-词分布，相当于phi。
    val topicsMat=model.topicsMatrix
    println("topicsMatrix")
    println(topicsMat.toString())
     /**topicsMatrix
        12.992380082908886  0.5654447550856024  16.438154549631257  
        10.552480038361052  0.6367807085306598  19.81281695100224   
        2.204054885551135   0.597153999004713   6.979803589429554 
    * 
    */

    //-----------------------------------4 对语料的主题进行聚类--------------------- 
    val topicsProb=model.transform(dataset_lpa)
    topicsProb.select("label", "topicDistribution")show(false)

    /** label是文档序号 文档中各主题的权重
        +-----+--------------------------------------------------------------+
        |label|topicDistribution                                             |
        +-----+--------------------------------------------------------------+
        |0.0  |[0.523730754859981,0.006564444943344147,0.46970480019667477]  |
        |1.0  |[0.7825074858166653,0.011001204994496623,0.206491309188838]   |
        |2.0  |[0.2085069748527087,0.005698459472719417,0.785794565674572]   |
        ...

    */
  }
}
```
虽然推断出K个主题，进行聚类是LDA的首要任务，但是从代码第4部分输出的结果（每篇文章的topicDistribution,即每篇文章在主题上的分布）我们还是可以看出，LDA还可以有更多的用途: 
（1）特征生成：LDA可以生成特征（即topicDistribution向量）供其他机器学习算法使用。如前所述，LDA为每一篇文章推断一个主题分布；K个主题即是K个数值特征。这些特征可以被用在像逻辑回归或者决策树这样的算法中用于预测任务。 
（2）降维：每篇文章在主题上的分布提供了一个文章的简洁总结。在这个降维了的特征空间中进行文章比较，比在原始的词汇的特征空间中更有意义。 
所以呢，我们需要记得LDA的多用途，（1）聚类，（2）降纬，（3）特征生成，一举多得，典型的多面手。
 
 > 原文 ：https://blog.csdn.net/qq_34531825/article/details/52608003
   
