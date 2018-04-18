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
```
 <dependency>
   <groupId>org.ansj</groupId>
   <artifactId>ansj_seg-5.0.1-all-in-one</artifactId>
   <version>5.0.1</version>
 </dependency>
```

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

#### UserDefineLibrary API操作
```
/**添加单个词典***/
    UserDefineLibrary.insertWord("艾泽拉斯","n",10)
    //基础分词
    val parse5 = BaseAnalysis.parse("我在艾泽拉斯") // 基础分词不支持用户自定义词典，所以不发生改变
    //精准分词
    val parse6 = ToAnalysis.parse("我在艾泽拉斯")
    //NLP分词
    val parse7 = NlpAnalysis.parse("我在艾泽拉斯")
    /***单个移除词典**/
    UserDefineLibrary.removeWord("艾泽拉斯")
    val parse8 = ToAnalysis.parse("我在艾泽拉斯")

    /*****加载自定义词库*/
    /**
      * 词库格式（"自动义词"[tab]键"词性"[tab]键"词频"）
      * 第一个参数直接默认为 :UserDefineLibrary.FOREST
      * 第二个参数词库路径 address2.dic 格式
      */
    UserDefineLibrary.loadLibrary(UserDefineLibrary.FOREST,"")
    ToAnalysis.parse("我在艾泽拉斯至高岭雷霆图腾")
    
```

### Ansj中文分词Java开发自定义和过滤词库
Ansj中文分词应用时，需要自定义词库，比如城中村，分词成城、中、村，需自定义词库，有时，也需要过滤单词。具体代码如下，可以结合执行结果看代码效果。

**1、过滤词库**

```
package csc.ansj;  
  
import org.ansj.domain.Result;  
import org.ansj.recognition.impl.FilterRecognition;  
import org.ansj.splitWord.analysis.ToAnalysis;  
  
public class AnsjWordFilter {  
    public static void main(String[] args) {  
        String str = "不三不四，您好！欢迎使用ansj_seg,深圳有没有城中村这里有宽带吗?(ansj中文分词)在这里如果你遇到什么问题都可以联系我.我一定尽我所能.帮助大家.ansj_seg更快,更准,更自由!" ;  
        //过滤词性和词汇  
        FilterRecognition fitler = new FilterRecognition();  
        //http://nlpchina.github.io/ansj_seg/content.html?name=词性说明  
        fitler.insertStopNatures("w"); //过滤标点符号词性  
        fitler.insertStopNatures("null");//过滤null词性  
        fitler.insertStopNatures("m");//过滤m词性  
        fitler.insertStopWord("不三不四"); //过滤单词  
        fitler.insertStopRegex("城.*?"); //支持正则表达式  
        Result modifResult = ToAnalysis.parse(str).recognition(fitler); //过滤分词结果  
        System.out.println(modifResult.toString());  
    }  
}  
/* 
 * 您好/l,欢迎/v,使用/v,ansj/en,seg/en,深圳/ns,有/v,没/d,有/v,村/n,这里/r,有/v,
 宽带/nz,吗/y,ansj/en,中文/nz,分词/n,在/p,这里/r,如果/c,你/r,遇到/v,什么/r,问题/n,
 都/d,可以/v,联系/v,我/r,我/r,一定/d,尽/v,我/r,所/u,能/v,帮助/v,大家/r,ansj/en,seg/en,
 更/d,快/a,更/d,准/a,更/d,自由/a 
 */  
```

**2、自定义词库，可以设置歧义词等**

```
import org.ansj.app.summary.SummaryComputer;  
import org.ansj.domain.Result;  
import org.ansj.domain.Term;  
import org.ansj.library.UserDefineLibrary;  
import org.ansj.splitWord.analysis.ToAnalysis;  
import org.nlpcn.commons.lang.tire.domain.Forest;  
import org.nlpcn.commons.lang.tire.domain.Value;  
import org.nlpcn.commons.lang.tire.library.Library;  
  
public class AnsjWordDefine {  
      
    public static void main(String[] args) throws Exception {  
        String str = "不三不四，您好！欢迎使用ansj_seg,深圳有没有城中村这里有宽带吗?
        (ansj中文分词)在这里如果你遇到什么问题都可以联系我.我一定尽我所能.
        帮助大家.ansj_seg更快,更准,更自由!" ;  
          
        // 增加新词,中间按照'\t'隔开  
        UserDefineLibrary.insertWord("城中村", "userDefine", 1000);//自定义词汇、自定义词性  
        UserDefineLibrary.insertWord("ansj中文分词", "userDefine", 1001);  
        Result terms = ToAnalysis.parse(str);  
        System.out.println("增加自定义词库:" + terms.toString());  
          
        // 删除词语,只能删除.用户自定义的词典.  
        UserDefineLibrary.removeWord("ansj中文分词");  
        UserDefineLibrary.removeWord("城中村");  
        terms = ToAnalysis.parse(str);  
        System.out.println("删除自定义词库:" + terms.toString());  
          
        // 歧义词  
        Value value = new Value("济南下车", "济南", "n", "下车", "v");  
        System.out.println(ToAnalysis.parse("我经济南下车到广州.中国经济南下势头迅猛!"));  
        Library.insertWord(UserDefineLibrary.ambiguityForest, value);  
        System.out.println(ToAnalysis.parse("我经济南下车到广州.中国经济南下势头迅猛!"));  
          
        // 多用户词典  
        String str1 = "神探夏洛克这部电影作者.是一个dota迷";  
        System.out.println(ToAnalysis.parse(str1));  
        // 两个词汇 神探夏洛克 douta迷  
        Forest dic1 = new Forest();  
        Library.insertWord(dic1, new Value("神探夏洛克", "define", "1000"));  
        Forest dic2 = new Forest();  
        Library.insertWord(dic2, new Value("dota迷", "define", "1000"));  
        System.out.println(ToAnalysis.parse(str1, dic1, dic2));  
    }  
}  
/* 
 增加自定义词库:不三不四/i,，/w,您好/l,！/w,欢迎/v,使用/v,ansj/en,_,seg/en,,,
 深圳/ns,有/v,没/d,有/v,城中村/userDefine,这里/r,有/v,宽带/nz,吗/y,?,
 (,ansj中文分词/userDefine,),在/p,这里/r,如果/c,你/r,遇到/v,什么/r,问题/n,
 都/d,可以/v,联系/v,我/r,./m,我/r,一定/d,尽/v,我/r,所/u,能/v,./m,
 帮助/v,大家/r,./m,ansj/en,_,seg/en,更/d,快/a,,,更/d,准/a,,,更/d,自由/a,! 
删除自定义词库:不三不四/i,，/w,您好/l,！/w,欢迎/v,使用/v,ansj/en,_,seg/en,,,
深圳/ns,有/v,没/d,有/v,城中/ns,村/n,这里/r,有/v,宽带/nz,吗/y,?,(,ansj/en,中文/nz,分词/n,),
在/p,这里/r,如果/c,你/r,遇到/v,什么/r,问题/n,都/d,可以/v,联系/v,我/r,./m,我/r,
一定/d,尽/v,我/r,所/u,能/v,./m,帮助/v,大家/r,./m,ansj/en,_,seg/en,
更/d,快/a,,,更/d,准/a,,,更/d,自由/a,! 
我/r,经济/n,南/f,下/f,车/n,到/v,广州/ns,./m,中国/ns,经济/n,南/f,下/f,
势头/n,迅猛/a,! 
我/r,经/p,济南/n,下车/v,到/v,广州/ns,./m,中国/ns,经济/n,南/f,下/f,
势头/n,迅猛/a,! 
神/n,探/v,夏洛克/nr,这部/r,电影/n,作者/n,./m,是/v,一个/m,dota/en,
迷/v 神探夏洛克/define,这部/r,电影/n,作者/n,./m,是/v,一个/m,dota迷/define 
 */  
```

**附：词性说明**

汉语文本词性标注标记集

**1. 名词  (1个一类，7个二类，5个三类)**

名词分为以下子类：

| 符号        | 说明    |
| --------   | -----:   |
|n        |名词       |
|nr        |人名      |
|nr1        |汉语姓氏     |
|nr2        |汉语名字      |
|nrj        |日语人名      |
|nrf        |音译人名      |
|ns        |地名      |
|nsf        |音译地名      |
|nt        |机构团体名      |
|nz        |其它专名      |
|nl        |名词性惯用语      |
|ng        |名词性语素      |
|nw        |新词      |

**2. 时间词(1个一类，1个二类)**

| 符号        | 说明    |
| --------   | -----:   |
|t        |时间词       |
|tg        |时间词性语素      |

 **3. 处所词(1个一类)**
 
| 符号        | 说明    |
| --------   | -----:   |
|s        |处所词       |

**4. 方位词(1个一类)**

| 符号        | 说明    |
| --------   | -----:   |
|f         |方位词      |

**5. 动词(1个一类，9个二类)**

| 符号        | 说明    |
| --------   | -----:   |
|v         |动词    |
|vd         |副动词    |
|vn         |名动词    |
|vshi         |动词“是”    |
|vyou         |动词“有”    |
|vf         |趋向动词    |
|vx         |形式动词    |
|vi         |不及物动词（内动词）    |
|vl         |动词性惯用语    |
|vg         |动词性语素    |


**6. 形容词(1个一类，4个二类)**

| 符号        | 说明    |
| --------   | -----:   |
|a         |形容词   |
|ad         |副形词   |
|an         |名形词   |
|ag         |形容词性语素   |
|al         |形容词性惯用语   |

**7. 区别词(1个一类，2个二类)**

| 符号        | 说明    |
| --------   | -----:   |
|b         |区别词   |
|bl         |区别词性惯用语  |

**8. 状态词(1个一类)**

| 符号        | 说明    |
| --------   | -----:   |
|z         | 状态词   |

**9. 代词(1个一类，4个二类，6个三类)**

| 符号        | 说明    |
| --------   | -----:   |
|r         |代词     |
|rr         |人称代词     |
|rz         |指示代词     |
|rzt         |时间指示代词     |
|rzs         |处所指示代词     |
|rzv         |谓词性指示代词     |
|ry         |疑问代词     |
|ryt         |时间疑问代词     |
|rys         |处所疑问代词     |
|ryv         |谓词性疑问代词     |
|rg         |代词性语素     |

**10. 数词(1个一类，1个二类)**

| 符号        | 说明    |
| --------   | -----:   |
|m          |数词     |
|mq         |数量词     |

**11. 量词(1个一类，2个二类)**

| 符号        | 说明    |
| --------   | -----:   |
|q          |量词     |
|qv         |动量词     |
|qt         |时量词    |

**12. 副词(1个一类)**

| 符号        | 说明    |
| --------   | -----:   |
|s          |副词     |

**13. 介词(1个一类，2个二类)**

| 符号        | 说明    |
| --------   | -----:   |
|p          |介词    |
|pba          |介词“把    |
|pbei        |介词“被”    |

**14. 连词(1个一类，1个二类)**

| 符号        | 说明    |
| --------   | -----:   |
|c          | 连词    |
|cc          |并列连词    |

**15. 助词(1个一类，15个二类)**

| 符号        | 说明    |
| --------   | -----:   |
|u       |助词      |
|uzhe       |着      |
|ule       |了 喽      |
|uguo       |过      |
|ude1       |的 底      |
|ude2       |地      |
|ude3       |得      |
|usuo       |所      |
|udeng       |等 等等 云云      |
|uyy       |一样 一般 似的 般      |
|udh       |的话      |
|uls       |来讲 来说 而言 说来      |
|uzhi       |之      |
|ulian       |连 （“连小学生都会”）      |

**16. 叹词(1个一类)**

| 符号        | 说明    |
| --------   | -----:   |
|e       |叹词      |

**17. 语气词(1个一类)**

| 符号        | 说明    |
| --------   | -----:   |
|y       |语气词(delete yg)      |

**18. 拟声词(1个一类)**

| 符号        | 说明    |
| --------   | -----:   |
|o       |拟声词     |

**19. 前缀(1个一类)**

| 符号        | 说明    |
| --------   | -----:   |
|h       |前缀     |

**20. 后缀(1个一类)**

| 符号        | 说明    |
| --------   | -----:   |
|k       |后缀     |

**21. 字符串(1个一类，2个二类)**

| 符号        | 说明    |
| --------   | -----:   |
|x       |字符串     |
|xx       |非语素字     |
|xu       |网址URL   |

**22. 标点符号(1个一类，16个二类)**

| 符号        | 说明    |
| --------   | -----:   |
|w       |标点符号      |
|wkz       |左括号，全角：（ 〔  ［  ｛  《 【  〖〈   半角：( [ { <      |
|wky       |右括号，全角：） 〕  ］ ｝ 》  】 〗 〉 半角： ) ] { >      |
|wyz       |左引号，全角：“ ‘ 『       |
|wyy       |右引号，全角：” ’ 』      |
|wj       |句号，全角：。      |
|ww       |问号，全角：？ 半角：?      |
|wt       |号，全角：！ 半角：!      |
|wd       |号，全角：， 半角：,      |
|wf       |号，全角：； 半角： ;      |
|wn       |号，全角：、      |
|wm       |号，全角：： 半角： :      |
|ws       |略号，全角：……  …      |
|wp       |折号，全角：——   －－   ——－   半角：---  ----      |
|wb       |分号千分号，全角：％ ‰   半角：%      |
|wh       |位符号，全角：￥ ＄ ￡  °  ℃  半角：$      |







