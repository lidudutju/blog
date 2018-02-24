---
title: 初探Kotlin
author: lidu
date: 2018-02-24 17：35
tags: Android、Kotlin
---

#### 前言

说起Kotlin这门语言，我在2015年初刚学Android编程的时候听闻过，当时官方还没有出1.0稳定版，且相关的资料和Android开源项目非常少，所以也就不怎么放在心上了。

第二次接触Kotlin就是在Google I/O 2017中了，Google官方宣布将Kotlin作为Android编程的官方支持语言和推荐语言。同年，Android开源社区的大神Jake Wharton从Square跳槽到Google，并从事Kotlin在Android编程中相关的研究和推广工作，开源了[几个Kotlin项目](https://github.com/JakeWharton?utf8=%E2%9C%93&tab=repositories&q=&type=&language=kotlin)和[Kotlin Style Guide](https://android.github.io/kotlin-guides/index.html)，之后Kotlin在Android编程中的应用热度大增。同样是在2017年，Kotlin的爹爹Jetbrains在11月份搞了个[第一届Kotlin大会](https://www.kotlinconf.com/2017/)，这个会议的所有分享和演讲都是关于Kotlin这门语言及其跨平台应用的内容。会议非常成功，一时之间Kotlin热度达到巅峰。

综上所述，2017年注定是Kotlin语言不平凡的一年。从Google官方的支持、开源社区Jake、chrisbanes等大神的大力推广、Jetbrains的高度重视、Kotlin Conf的成功举办，无一不是Kotlin逐渐获得关注并在生产环境中得以应用的例证。

<!--more-->

#### 语言优势

官方给出的简述是：
> Statically typed programming language for modern multiplatform applications

注意两点：  

1. 这是一门静态类型语言，编译器在编译期间能够帮助我们解决类型赋值和类型转换的安全问题，确保运行时极少出现类型转换异常。Java也是静态类型语言，但是Kotlin在这方面做的更好，毕竟爹爹Jetbrains就是搞编译器和IDE出身的。Kotlin中有val和var，在编译期就确定了属性是只读的还是可变的。Kotlin中有可空类型和非空类型，能够极大减少运行时空指针异常的出现。
2. 这门语言是有一定野心的。Kotlin希望成为一个跨平台开发的语言，能够运行在Android和iOS系统的手机上、浏览器中和服务器端。2017 Kotlin-Conf中的[Demo](https://github.com/JetBrains/kotlinconf-app)就是一个跨平台开发的示例。在跨平台开发这一点上，我暂时是持观望态度的，这个思想是非常好的，但是要打破各个方向上的既有开发环境和三方库，阻力还是蛮大的。

对于Kotlin语言的特性，官方给出解释是：
> Concise、Safe、Interoperable、Tool-friendly

1. Concise也就是精简、简练的含义。英语相比于现代汉语、日语、韩语等等语言就精简很多，当我们想表达一个意思时，可以用更少的英文单词表现出更加准确的含义，这也是英语迄今仍然令我着迷的原因之一。对比Kotlin和Java，想要实现相同的功能，Kotlin只需更少的代码量，而且可读性也非常强。
2. 安全性得益于Kotlin语言的静态类型系统。刚开始接触Kotlin语法的时候，乍一看很多语法特性和Scala很相似。两者都有强大的静态类型系统，更加智能的编译器很多时候可以自动推测属性的类型或者函数的返回类型等等，而在编译期就确定的类型系统可以很好地减少运行时类型出错的可能性。一个很好的例证就是Kotlin有可空类型和非空类型，将null赋值给一个非空类型变量将会导致编译期出错，同时我们在使用非空类型变量时，无需再写很多的非空判断，也能很好地减少空指针异常。
3. 兼容性体现在Kotlin是一门运行在JVM之上的语言。Koltin代码编译后生成的JVM字节码和Java代码生成的JVM字节码基本相同，因此在一个工程中同时使用Java和Kotlin开发完全可行。互通性体现在Java代码和Kotlin代码之间可以互相调用，这样Java的很多成熟库都是可以保留的，无需造轮子重写。
4. 良好的工具支持这个就不用多说了吧，IDE都是自家爹爹做的，IDE能够提供给Java的大部分功能，对于Kotlin都是适用的。举个例子，Android Lint可以检查Java代码、xml文件等等规范性，现在也支持检查Kotlin代码。

下面挑选几个语法特性，简单讲一讲：

##### （1）高阶函数和Lambdas

高阶函数就是一类可以将函数作为入参或者返回值的函数。一个典型例子是：

```kotlin
fun <T> lock(lock: Lock, body: () -> T): T {
    lock.lock()
    try {
        return body()
    }
    finally {
        lock.unlock()
    }
}
```
函数lock就是一个高阶函数，它接收一个Lock实例和一个函数 **( ) -> T** 作为参数。

lambda表达式遵循如下特征：

1. lambda表达式总是被大括号包含 **{ }**
2. lambda表达式的参数写在 **->** 之前，如果没有参数，可省略
3. lambda表达式的内容写在 **->** 之后

一个最简单的lambda表达式可以写成这样：

```kotlin
{ x: Int, y: Int -> x + y }
```

如下，我们调用lock函数，传入lock对象，以及一个lambda表达式 **{ sharedResource.operation() }**（ 由于该lambda没有参数，可省略 **->** ）

在Kotlin中有一项传统，如果函数的最后一个参数是函数类型，那么我们在调用该函数时，可以将该函数的最后一个参数写在圆括号之外，这样我们可以创造自己的DSL，如Anko中DSL写代码布局文件就大量使用这项便捷语法糖。

```kotlin
val result = lock(lock, { sharedResource.operation() })

lock (lock) {
    sharedResource.operation()
}
```

##### （2）拓展函数（Extension Functions）

拓展函数可以让我们给既有的JDK、SDK或者其他库中的类增加拓展，而不会破坏这些类的实现。比如说我们想要写一个工具类获取一个字符串的最后一个字符：

```kotlin
fun String.last() : Char {
	return this[length - 1]
}

val x = "Hey!!"

println(x.last()) // prints '!'

```

##### （3）命名和默认参数函数（Named and default function arguments）

```kotlin
fun format(title: String, desc: String = "Desc") = title + "_" + desc

val x = format("one", "1")

val y = format(title = "two")

val z = format(title = "three", desc = "3")

```

我们定义了一个format函数，其中第二个参数包含默认值，且我们调用该函数时可以指定参数的名称，这样的函数就是命名参数和默认参数函数。

##### （4）安全的非空类型（Null Safety）

空类型安全要求我们在编译期就要指定某种类型是否可以为空。

```kotlin

var nonNullStr: String = "Hey!!"
var nullableStr: String? = "I can be null!!"

nonNullStr = null // 编译期错误，null无法赋值给非空类型
nullableStr = null // OK!!

val a = nonNullStr.length // OK!! 
val b = nullableStr.length // 编译期提示错误，nullableStr可能为null
val c = nullableStr?.length // OK!如果nullableStr为null，则返回null，否则返回length

```

##### （5）数据类（Data Class）

对于一些纯数据Model类，Kotlin有更加便捷的写法：

```kotlin
data class User(val name: String, val age: Int)

val person = User("lidu", 23)

```

编译器可以自动帮我们实现equals() / hashCode()方法，且默认实现toString()方法，返回**"User(name="lidu", age=23)"**

#### 编译速度与运行时性能

关于编译速度，Medium上有[一篇文章](https://medium.com/keepsafe-engineering/kotlin-vs-java-compilation-speed-e6c174b39b5d)做了详细的测试，结论大致是：

1. 在Gradle不开守护进程，Clean之后再Build的情况下，Java编译速度比Kotlin快17%
2. 在Gradle开启守护进程，Clean之后再Build的情况下，Java编译速度比Kotlin快13%
3. 在最常见的情况下，启用增量编译部分构建时，Kotlin编译速度和Java差不多，或略快于Java

关于运行时性能，也有[一遍文章](https://sites.google.com/a/athaydes.com/renato-athaydes/posts/kotlinshiddencosts-benchmarks)做了详细测试，这里暂时不展开讨论了

#### 社区活跃度和氛围

<image src="https://ws1.sinaimg.cn/large/dc50da5fgy1forkc1nzetj21f00ue4i4.jpg" width="50%" height="50%">

这是2017年有关Kotlin分享演讲的活跃地图。可以看到欧洲是最密集的区域，这个很好理解，因为Kotlin语言就是捷克的Jetbrains公司创造的，这帮欧洲人对于Kotlin的兴趣和掌握的熟练程度都是最高的。第二个密集区域是美国的东海岸地区，我猜测可能是东海岸创业公司比较多，对于新的语言使用起来没有什么顾虑，不用像西海岸一些成熟公司切换语言、修改代码库那么费事儿。日本人对于Kotlin语言好像很有兴趣，活跃度也很高。中国的话，活跃度目前很低，只有北京、深圳、台湾地区有零星的活跃人群。

![](https://ws1.sinaimg.cn/large/dc50da5fgy1forkdev6p3j20mu08l3zy.jpg)

Github上Kotlin的代码量在2017年飞速上涨，截至11月份，Kotlin代码行数超过2500万行。超过6300个Kotlin的相关问题在StackOverflow上被提出。去年安装Kotlin插件的开发者超过56万。

##### [Kotlin官方论坛](https://discuss.kotlinlang.org/)
##### [Kotlin相关链接](https://kotlin.link/)

#### 相关工具支持

1. Kotlin插件相关：Android Studio3.0及以上已内置Kotlin支持，无需再安装Kotlin插件，Android3.0以下需安装[Kotlin插件](https://plugins.jetbrains.com/plugin/6954-kotlin)
2. 使用Anko库用DSL替代xml文件的方式来写布局文件，需使用插件预览样式—[Anko预览插件](https://plugins.jetbrains.com/plugin/7734-anko-support)

#### 参考内容

1. [Kotlin Reference](https://kotlinlang.org/docs/reference/)
2. [Using Project Kotlin for Android
](https://docs.google.com/document/d/1ReS3ep-hjxWA8kZi0YqDbEhCqTt29hG8P44aA9W0DM8/edit#heading=h.v6i4iyf7kkxh)
3. [Kotlin编译过程分析](http://shinelw.com/2017/03/19/kotlin-compiler-process-analysis/)
4. [Kotlin vs Java: Compilation speed](https://medium.com/keepsafe-engineering/kotlin-vs-java-compilation-speed-e6c174b39b5d)
5. [Kotlin 1.2 Released: Sharing Code between Platforms](https://blog.jetbrains.com/kotlin/2017/11/kotlin-1-2-released/)
