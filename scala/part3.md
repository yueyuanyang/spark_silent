### 柯里化函数

#### 柯里化提供了二种写法

**写法一**
```
def func1(index : Int) = (array : Array[Int]) => if(array(index) == 1) true else false
```


**写法二**
```
def func(index : Int)(array : Array[Int]) : Boolean = if(array(index)==1) true else false
```

柯里化可以配合隐式转化进行默认值使用

**详细代码**

```
object CurryDemo {

  def main(args: Array[String]) {

    val arr = Array(Array(2,1,3,4,5),Array(2,1,3,4,5),Array(2,1,3,4,5))
    println(arr.filter(func(1)))

    println(m(2))
    println(m1(3,8))

  }
  // 偏函数

  // 柯里化写法(1)
  def func(index : Int)(array : Array[Int]) : Boolean = if(array(index)==1) true else false
  // 柯里化写法(2)
  def func1(index : Int) = (array : Array[Int]) => if(array(index) == 1) true else false

  //柯里化加隐式转化
  def m(x : Int)(implicit y : Int = 5) = x * y
  def m1(x : Int, y : Int = 5) = x * y
}
```
