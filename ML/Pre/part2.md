## SPARK+ANSJ 中文分词基本操作

### ANSJ 5.0.1


这是一个基于n-Gram+CRF+HMM的中文分词的java实现.

分词速度达到每秒钟大约200万字左右（mac air下测试），准确率能达到96%以上

目前实现了.中文分词. 中文姓名识别 . 用户自定义词典,关键字提取，自动摘要，关键字标记等功能

可以应用到自然语言处理等方面,适用于对分词效果要求高的各种项目.

**下载jar：**

访问 http://maven.nlpcn.org/org/ansj/ 最好下载最新版 ansj_seg/

如果你用的是1.x版本需要下载 tree_split.jar。

如果你用的是2.x版本需要下载 nlp-lang.jar。

如果你用的是3.x以上版本只需要下载 ansj_seg-[version]-all-in-one.jar 一个jar包就能浪了。

本文使用的是ansj5.0.1版本,在云盘 https://pan.baidu.com/disk/home?#list/vmode=list&path=%2Fjar 下载

### Ansj Api 详解

**分词方式：**

- 基本分词：最基本的分词.词语颗粒度最非常小的(api: BaseAnalysis.parse() )
- 精准分词：在易用性,稳定性.准确性.以及分词效率上.都取得了一个不错的平衡. (api : ToAnalysis.parse)
- nlp分词：语法实体名抽取.未登录词整理.只要是对文本进行发现分析等工作(api : NlpAnalysis.parse() )
- 面向索引分词：故名思议就是适合在lucene等文本检索中用到的分词。( api :  IndexAnalysis.parse() )

**API 使用详解**

### 单个添加词典
```



```

