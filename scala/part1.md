## scala 基本类语法一 ：Future 和 Promise

### 概念

future对象用于表示异步方法获得的结果，例如需要网络连接的耗时service call的结果。有了future对象，我们可以让程序在没有获得数据的情况下继续执行。

如果没有future对象的调用叫 blocking call，因为进程会等待结果获取后才会继续进行。

```
val f: Future[List[Friend]] = Future {  
session.getFriends()  
}

session.getFriends()

可能耗时长久，所以返回一个future的List[Friend]，这样其余的代码可以继续运行。
```

future生成后，程序会继续运行。若future的值可用，会自动callback onComplete方法， 或是更具体的调用onSuccess或onFailure方法。

但是调用callback的线程不能确定，如果有多个onSucess方法，他们可能会被同时调用，甚至可能会交错调用(如果callback中的操作不是atomic的话，

可能callback A运行了一步，callback B运行) , 内嵌的 onComplete callback function使用Try[T] => U参数。 

Try[T]类似Option[T]，有所优化的是若success，内含T值，否则包含一个Throwable exception。所以基本相当于Either[Throwable, T]

import ExecutionContext.Implicits.global 如：导入默认的全局执行上下文

**常用的是提供回调函数方法**

**可以用 f onComplete {
    case Success() => println  
    case Failure() => println                 
}**

```
Future 编程范式(一)：第一种写法

val f: Future[List[String]] = Future {
    session.getRecentPosts  
}  
f onComplete {  
    case Success(posts) => for (post <- posts) println(post)  
    case Failure(t) => println("An error has occured: " + t.getMessage)  
}
```

**也可以用onSuccess或是onFailure, 或者只提供一个:**

```
Future 编程范式(二)：第二种写法

val f: Future[List[String]] = Future {  
   session.getRecentPosts  
}  
f onFailure {  
   case t => println("An error has occured: " + t.getMessage)  
}  
f onSuccess {  
   case posts => for (post <- posts) println(post)  
}  
```

**future类型的连锁:**

future可以直接当做变量使用在map, flatMap或是for中，生成的也是future， 并且在所有需要的future complete后，调用callback。

```
val usdQuote = Future { connection.getCurrentValue(USD) }  
val chfQuote = Future { connection.getCurrentValue(CHF) }  
val purchase = for {  
    usd <- usdQuote  
    chf <- chfQuote  
    if isProfitable(usd, chf)  
} yield connection.buy(amount, chf)  
  
purchase onSuccess {  
    case _ => println("Purchased " + amount + " CHF")  
}  
```

例中，purchase就是一个衍生的future， 当usdQuote和chfQuote准备好后，onSuccess才被调用。

也可以使用专为future定义的一些组合，例如recover(获取exception)， fallbackTo(某一个future异常后使用另一个future),

andThen等。具体情况具体使用。


## Promise 

Promise 允许你在 Future 里放入一个值，不过只能做一次，Future 一旦完成，就不能更改了。

一个 Future 实例总是和一个（也只能是一个）Promise 实例关联在一起。如果你在 REPL 里调用 future 方法，你会发现返回的也是一个 Promise：

### 给个实例
### 给出承诺

当我们谈论起承诺能否被兑现时，一个很熟知的例子是那些政客的竞选诺言。

假设被推选的政客给他的投票者一个减税的承诺。这可以用 Promise[TaxCut] 表示：

```
import scala.concurrent.Promise
case class TaxCut(reduction: Int)
val taxcut = Promise[TaxCut]()
val taxcut2: Promise[TaxCut] = Promise()
```

一旦创建了这个 Promise，就可以在它上面调用 future 方法来获取承诺的未来：

```
val taxCutF: Future[TaxCut] = taxcut.future
```
返回的 Future 可能并不和 Promise 一样，但在同一个 Promise 上调用 future 方法总是返回同一个对象,

以确保 Promise 和 Future 之间一对一的关系。

### 兑现承诺

为了成功结束一个 Promise，你可以调用它的 success 方法，并传递一个大家期许的结果：

```
taxcut.success(TaxCut(20))
```
这样做之后，Promise 就无法再写入其他值了，如果偏要再写，会产生异常。

### 完整例子

```
object Government {
  def redeemCampaignPledge(): Future[TaxCut] = {
    val p = Promise[TaxCut]()
    Future {
      println("Starting the new legislative period.")
      Thread.sleep(2000)
      p.success(TaxCut(20))  -- 承诺诺言
      // p.failure(LameExcuse("global economy crisis")) -- 违背诺言
      println("We reduced the taxes! You must reelect us!!!!1111")
    }
    p.future
  }
}

import scala.util.{Success, Failure}
val taxCutF: Future[TaxCut] = Government.redeemCampaignPledge()
println("Now that they're elected, let's see if they remember their promises...")
taxCutF.onComplete {
  case Success(TaxCut(reduction)) =>
    println(s"A miracle! They really cut our taxes by $reduction percentage points!")
  case Failure(ex) =>
    println(s"They broke their promises! Again! Because of a ${ex.getMessage}")
}

```

### 实例2

考虑下面的生产者 - 消费者的例子，其中一个计算产生一个值，并把它转移到另一个使用该值的计算。这个传递中的值通过一个promise来完成。

```
import scala.concurrent.{ future, promise }
import scala.concurrent.ExecutionContext.Implicits.global

val p = promise[T]
val f = p.future

val producer = future {
  val r = produceSomething()
  p success r
  continueDoingSomethingUnrelated()
}

val consumer = future {
  startDoingSomething()
  f onSuccess {
    case r => doSomethingWithResult()
  }
}


```

在这里，我们创建了一个promise并利用它的future方法获得由它实现的Future。然后，我们开始了两种异步计算。

第一种做了某些计算，结果值存放在r中，通过执行promise p，这个值被用来完成future对象f。第二种做了某些计算，

然后读取实现了future f的计算结果值r。需要注意的是，在生产者完成执行continueDoingSomethingUnrelated() 方法

这个任务之前，消费者可以获得这个结果值。








