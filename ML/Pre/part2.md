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

#### 基本分词
```
val parse = BaseAnalysis.parse("孙杨在里约奥运会男子200米自由泳决赛中，以1分44秒65夺得冠军");
System.out.println(parse);
result：[孙/nr,杨/nr,在/p,里/f,约/d,奥运会/j,男子/n,200/m,
米/q,自由泳/n,决赛/vn,中/f,，/w,以/p,1/m,分/q,44/m,秒/q,65/m,夺得/v,冠军/n]

```
#### 精准分词
```
val parse = ToAnalysis.parse("孙杨在里约奥运会男子200米自由泳决赛中，以1分44秒65夺得冠军");
System.out.println(parse);
result:[孙杨/nr,在/p,里/f,约/d,奥运会/j,男子/n,200米/m,自由泳/n,决赛/vn,中/f,
，/w,以/p,1分/m,44秒/m,65/m,夺得/v,冠军/n]

```
#### nlp分词
```
val parse = NlpAnalysis.parse("孙杨在里约奥运会男子200米自由泳决赛中，以1分44秒65夺得冠军");
System.out.println(parse);
result:[孙杨/nr,在/p,里约,奥运会/j,男子/n,200米/m,自由泳/n,决赛/vn,中/f,，/w,以/p,1分/m,44秒/m,65/m,夺得/v,冠军/n]
```
#### 面向索引分词
```
var parse =  IndexAnalysis.parse("主副食品")
result：[主副食品/n]
```
#### 加载自定义词典
```
val forest0 = Library.makeForest("E:/base.dic")
System.out.println(DicAnalysis.parse("孙杨在里约奥运会男子200米自由泳决赛中，以1分44秒65夺得冠军", forest0));
result:[孙杨/nr,在/p,里约/ns,奥运会男子200米自由泳/comb,决赛/vn,中/f,，/w,以/p,1分/m,44秒/m,65/m,夺得/v,冠军/n]
“奥运会男子200米自由泳”是加到词典中的
```

#### 去停用词
```
 var stopWord: Seq[String] = Seq("决赛")
    var filter = new FilterRecognition()
    filter.insertStopNatures("ns")
    filter.insertStopWords(stopWord)
    var word = "孙杨在里约奥运会男子200米自由泳决赛中，以1分44秒65夺得冠军"
    var result = DicAnalysis.parse(word).recognition(filter)
result：[孙杨/nr,在/p,里/f,约/d,奥运会/j,200米/m,中/f,，/w,以/p,1分/m,44秒/m,65/m,夺得/v]
去除了n词性“自由泳”，和停用词“决赛”，停用词可以是一个String，也可以是一个java List对象
```

#### 动态添加词典
```
UserDefineLibrary.insertWord("ansj中文分词", "userDefine", 1000);
var terms = ToAnalysis.parse("我觉得Ansj中文分词是一个不错的系统!我是王婆!");
System.out.println("增加新词例子:" + terms);
// 删除词语,只能删除.用户自定义的词典.
UserDefineLibrary.removeWord("ansj中文分词");
terms = ToAnalysis.parse("我觉得ansj中文分词是一个不错的系统!我是王婆!");
System.out.println("删除用户自定义词典例子:" + terms);
result:
增加新词例子:我/r,觉/v,得/ud,ansj中文分词/userDefine,是/v,一/m,个/q,不/d,错/n,的/uj,
系/v,统/v,!,我/r,是/v,王婆/nr,!删除用户自定义词典例子:我/r,觉/v,得/ud,ansj/en,中文/nz
,分/q,词/n,是/v,一/m,个/q,不/d,错/n,的/uj,系/v,统/v,!,我/r,是/v,王婆/nr,!
```



