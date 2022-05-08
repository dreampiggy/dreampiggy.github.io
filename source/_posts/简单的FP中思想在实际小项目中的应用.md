---
title: 简单的FP中思想在实际小项目中的应用
categories: Code
tags:
  - JavaScript
  - iOS
  - Functional
updated: '2016-03-30 01:50:11'
date: 2016-03-30 01:46:58
---

[函数式编程][1]，无论是谁，第一次听到都会感到好奇，疑惑以及畏惧。因为一提到函数式编程就会让人想到很多数学或者计算机科学理论研究的深奥原理，无论是[Lamda演算][2]还是[高阶函数][3]，似乎都和平常自己所接触到的编程语言毫无关系。

随着现代语言的发展，大多数传统面向对象语言已经支持了很多函数式编程中的语法，比如说：

[闭包][4]：就是指一个函数块，把连同这个函数所需要的所有参数（全局的or局部的）放入一个闭包中，然后这个函数可以单独用来执行，不会因为外部变量被修改而产生额外影响。相当于这个函数的上下文全部被保留了下来。

Lamda表达式：通俗点说，就是一个简单的闭包函数，通过表达式的方式来进行执行，而不需要再写复杂的逻辑指令式代码，或者无数的花括号来表明相关的上下文逻辑。这点对于简化繁杂的逻辑代码非常有帮助。

回调与改善的异步：其实，回调函数说实话也就是一个函数指针，也许很多人也用过，但是其实它的作用非行强大，如果你一直只接触过Java，C99的话是很难真正理解它对于IO或者网络请求的意义。尤其当你要处理多个异步事件流程，异常处理时，你就会发现它的真正意义。

其实，如果不是一定要从理论高度彻底理解函数式编程，没有必要从头看SICP或者所谓《21天精通Haskell》……（别打我），像Swift，C#，Java8，Python，C++11这种面向现代的语言中，都会有对于Lamda表达式的支持，像JavaScript这种更是纯粹可以当作函数式语言来写。所以，其实你已经在不知不觉中使用了函数式的一些思想，今天我就大概举几个例子来说明一下。

## 1\. 闭包

何谓“闭包”？简单的说就是一个把所有自由变量（全局变量，局部变量）包在一起的函数，什么叫做包在一起？就是指这个闭包中进行的任何操作，都不会对外部造成影响，外部变量的修改甚至是销毁，也不会影响闭包内部引用的全局变量。 是不是挺起来感觉有点奇怪，其实很多语言实现闭包的时候，都是把全局变量拷贝了一份到闭包内部，有些语言支持闭包内部修改外部变量（需要特殊的声明，比如说Swift的inout关键字） 比如下面一个简单的数组排序，用函数和闭包两种方法实现

```swift
var name = ["one","two"]
func compareName(s1:String,s2:String) -> Bool{
    return s1 < s2
}
var sortArrayByFunc = name.sorted(compareName)
var sortArrayByClosure = name.sorted({
    (s1:String,s2:String) -> Bool in
    name.removeAll(keepCapacity: true)
    return s1 < s2
})
println(name)
println(sortArrayByClosure)
```

![](http://dreampiggy-image.test.upcdn.net/image/f/fa/c2499a9bce100ff9e18f392609d5d.png)

结果正如我们所料，就算你在闭包内部把name数组清空了，排序后的新数组返回的内容还是不变，这个name数组在闭包被拷贝了一份，你所有更改都不会影响到它（甚至外部的name被release也是）。

## 2\.匿名函数

匿名函数，顾名思义，就是没有名字的函数……这其实不是很稀奇，因为Swift中就有外部参数名和内部参数名，因为当你认为一个函数可以当作参数的时候，它的名字（外部参数名）就可以省略，也让整个代码看着简单一些。比如JavaScript可以这样写～

```javascript
function functionOne (functionTwo) {
    functionTwo();
}
functionOne(function(){
    console.log("I have no name~");
})
```

Swift就像这样写：

```swift
func functionOne(functionTwo:()->()) -> void{
    functionTwo()
}

functionOne({
    println("Seems like JavaScript!")
})
```

也很好理解吧？就是把一个函数的外部名称去掉而已，简化了代码的繁冗（不然你会看到一段代码中各种无意义的小函数的名称，而且还容易导致名称冲突……） 匿名函数通常都是一个闭包，意思你的匿名函数访问外部变量时候是通过拷贝的，当然，不同语言的语法可能不太一样，建议用的时候要多加注意。

## 3\.尾递归

递归函数大家都会用，最简单的求阶乘的应用就可以用两行简单的递归搞定。

```swift
func factorial(n:Int64) -> Int64{
    if (n == 1){
        return 1
    }
    else{
        return factorial(n - 1) * n
    }
}
```

看起来很完美对吧？（当然，为了简化，没有对参数进行任何验证，而且实际也不应该用Int64来存放数字），但是，有没有想过如果我传过来的参数非常大，比如100这样（结果非常大，实际上这时候的结果已经超过Int的最大值了，有人可能这时候就用String之类来存放结果，但是这里讨论的重点不是这个）

![普通递归执行的示意图](http://dreampiggy-image.test.upcdn.net/image/b/f8/1d2434005481a33dde2ed357b09de.png)

当执行factorial(100)的时候，会发生什么呢？你会在栈中存放100个factorial()，包括函数的地址，函数里面定义的参数，变量……如果再大一点，你马上就见识到[StackOverFlow][7]的美景。

怎么办？这时候尾递归就派上用场了。尾递归，顾名思义，就是把前一次递归的函数直接返回一个结果，释放掉相应的空间（出栈），然后执行新的递归，无论有多深的递归，真实存在于栈中的只有一个，这样就不会因为栈空间不足而导致程序崩溃了。

OK……其实一般来说改造都是很简单的，只需要对原来的参数新加一个，用来存放前一个递归的结果就可以。就像这样～

```swift
func factorialTail(n:Int64,result:Int64) -> Int64{
    if (n == 1){
        return result
    }
    else{
        var product = result * n
        return factorialTail(n - 1,product)
    }
}

func factorialByTail(n:Int64) -> Int64{
    return factorialTail(n, 1)
}
```

(为了所谓的用户体验，所以用一个factorialByTail()来调用真正的递归……一般也是这样的，在这里进行一些数值判断或者更好的优化）

![尾递归改良版的示意图](http://dreampiggy-image.test.upcdn.net/image/0/70/4985a8f68dd733af308d0da0c4905.png)

和上一个版本的递归比，确实就是多了一个参数来存放上一次递归的结果，但是能非常有效的解决栈溢出的问题，在实际问题中也是非常实用的（甚至说必不可少的），大概到这里就可以了

## 4\.lambda表达式

对于没有接触过lambda这个字母（希腊字母）的人，这个东西听起来就和我第一次听到什么半幺群的感觉一样…… 但其实，lamda表达式就是一个匿名函数的简化写法（别打……），不过正如很多编程大师说的，能够写出精简、易懂、易修改的代码才是真正的好代码。如果处处使用匿名函数，你的代码讲会变得异常之长，多层嵌套，各种花括号……这对于阅读和修改都是一个灾难，所以，这时候lamda表达式来拯救我们了。

支持lamda表达式的现代语言有很多，比如C++11、C#、Java8、Scala、Python（可惜Swift只有闭包而没有支持lambda，一般是通过预先定义几个func来传入参数或者使用闭包的简写方法来简化代码，在这几个编程语言中基本我都是只停留在会语法的层面……所以这里我选择使用C++11来写）

既然你已经知道lambda表达式实质就是一个匿名函数，那么我们就直接开始干活吧，看看C++11的写法

> [详细的C++11 lambda表达式][9]

C++11的lambda写法有点奇葩……大概是这样[]()->void{...}这样写(mutable是值是否在这个匿名函数内部修改外部的自由变量）

```cpp
vector<int> myVector(10,0);
int counter = 0;
for_each(myVector.begin(), myVector.end(),[&](int i) mutable throw(string) ->void{
    cout<<++counter<<endl;
});
cout<<counter<<endl;
```

怎么样……相比于for循环来遍历，其实这个有时候看的更清楚（真的吗？），当然，Swift也可以写出类似的优雅的代码～like this:

```swift
init(){
    knownOps["+"] = Op.BinaryOperation("+"){ $0 + $1 }
    knownOps["-"] = Op.BinaryOperation("-"){ $1 - $0 }
    knownOps["*"] = Op.BinaryOperation("*"){ $0 * $1 }
    knownOps["/"] = Op.BinaryOperation("/"){ ($0 != 0) ? $1 / $0 : nil }
    knownOps["^"] = Op.UnaryOperation("^"){ $0 * $0 }
    knownOps["√"] = Op.UnaryOperation("√"){ $0 >= 0 ? sqrt($0) :nil }
}
```

这是一个简单的计算器的计算函数，Op是一个枚举类型，Op.BinaryOperation对应双操作数的运算，Op.UnaryOperation对应单操作数的运算，不需要每个运算（加减乘除）定义一个函数，只需要一个简单的闭包（Swift的闭包简写，如果返回值只有一行语句可以把花括号放到整个参数列表后面，用`$0`代表第一个参数，所以写起来可以非常简单）

lambda表达式的关键就在于能够配合很多内置的方法来使用，比如map，reduce，避免了繁荣的for循环，而且在多层嵌套里面再也不用数花括号的个数来写，看起来非常简单明晰，便于维护。所以我很推荐使用。

**就到这里吧……正如题目所说，这里只写了简单的FP思想，以及简单的应用，不会深究函数式编程的实质和lambda演算的内容……有兴趣的话买一本[SICP][10]看看你就懂了～～～**

 [1]: http://zh.wikipedia.org/w/index.php?title=函數程式語言
 [2]: http://zh.wikipedia.org/wiki/Λ演算
 [3]: http://zh.wikipedia.org/wiki/高阶函数
 [4]: http://zh.wikipedia.org/zh-cn/闭包_(计算机科学)
 [7]: http://en.wikipedia.org/wiki/Stack_overflow
 [9]: http://en.cppreference.com/w/cpp/language/lambda
 [10]: http://book.douban.com/subject/1148282