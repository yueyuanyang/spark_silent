## scala 基本类语法一 ：Future 和 Promise

### 概念

future对象用于表示异步方法获得的结果，例如需要网络连接的耗时service call的结果。有了future对象，我们可以让程序在没有获得数据的情况下继续执行。

如果没有future对象的调用叫 blocking call，因为进程会等待结果获取后才会继续进行。

```
val f: Future[List[Friend]] = Future {  
session.getFriends()  
}

session.getFriends()可能耗时长久，所以返回一个future的List[Friend]，这样其余的代码可以继续运行。
```

future生成后，程序会继续运行。若future的值可用，会自动callback onComplete方法， 或是更具体的调用onSuccess或onFailure方法。

但是调用callback的线程不能确定，如果有多个onSucess方法，他们可能会被同时调用，甚至可能会交错调用(如果callback中的操作不是atomic的话，

可能callback A运行了一步，callback B运行) , 内嵌的 onComplete callback function使用Try[T] => U参数。 

Try[T]类似Option[T]，有所优化的是若success，内含T值，否则包含一个Throwable exception。所以基本相当于Either[Throwable, T]

**可以用 onComplete {
    case Success() =>  
    case Failure() =>                  
}**

```
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



