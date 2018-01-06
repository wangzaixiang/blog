# 神奇的Scala Macro之旅（4）：BeanBuilder

在Java开发中，经常会有一个需求，将一个 Bean 复制到另外一个 Bean，尤其是在后台分层的场景下，在不同的层之间传递信息，经常需要进行
这样的一个对象复制工作，类似于：

```scala
val source: PersonSource = ...

val dest = new PersonDest()

dest.setName( source.getName )
dest.setEmail( source.getEmail )
dest.setAddress( source.getAddress )
...

``` 

因为这样的代码过于冗长，大量的这样的代码，大大的提升了代码的复杂性，不仅工作无趣，而且很容易遗漏，代码可阅读性差。万能的程序员
自然就会发明一个又一个的轮子来解决这个问题：

- apache BeanUtils
- dozer：http://dozer.sourceforge.net
- orika：http://orika-mapper.github.io/

所有这样的轮子，都包括这样的一些特性：
- 字段映射。 一般通过同名字段映射，或者使用 annotation 定义Bean间的字段映射。
- 类型映射。 诸如完成 String -> Int 这样的类型转换。
- 嵌套映射。

BeanUtils是一个最简单的，通过reflection来实现对象间的映射，这样存在的问题是有很大的反射开销，对性能要求很高的场景是不适合的。
orika等后续的框架则是通过字节码生成等技术，来实现性能的提升。

当我们走向函数式编程时，我们也会面临着对象的映射和复制的问题，BeanUtils/Orika等框架一般是基于Java Beans模型，而在函数式编程
中，我们更倾向于使用 immutable 的对象，beanutils/orika等模型存在不匹配的情况，于是，我们构造了一款适合于函数式编程的对象复制
方式，即 BeanBuilder.

这是一个基本的使用方法：
```scala
val source = PersonSource( name = "wangzx", email = "wangzx@qq.com", address="here" )

val dest = BeanBuilder.build[PersonDest](source)()

```
上面的代码中，我们会返回一个PersonDest对象（也是一个不可变的Case Class），并完成对象属性的复制工作，一个完整的使用方式如下：
```scala
  def build[T](sources: Any*)(additions: (String, Any)*) : T 
```

- `T` 目标类型，必须是一个Immutable Case Class，BeanBuilder 设计上是一个从 Case Class -> Case Class的转换工具
- `sources` 数据来源。 可以是0到多个对象。 这些对象的字段 会成为我们目标对象的字段 的来源。与BeanUtils、Orika等不的是，
我们的目标对象可以从多个源对象中复制属性。
- `additions` 以 `"name" -> value` 的形式定义的附加属性设置，意思是将 value 复制到 目标对象的 name 属性。

BeanBuilder 在设计上也不同于 BeanUtils，Orika，BeanBuilder首先被设计成为精确的、类型安全的对象复制，体现在：
- 精确。BeanBuilder 目前限定是同名复制，而且必须没有歧义。即值或者通过 additions 精确赋值，或者唯一存在于 sources 
对象之中，即只有一个 source 对象有这个同名属性。
- 所有目标对象中的非缺省值，都必须有数据来源，不能缺失（不会自动使用null作为目标值）。
- 类型安全。 BeanBuilder不会自动的进行 String -> Int 的转换，除非你主动的容许这个类型转换。当然，在BeanBuilder中，我们
可以更简单的定义源和目标的类型转换能力，并且在一种类型安全的方式进行。

BeanBuilder的类型转换采用如何策略，这里，`F`是源类型(值为`f`)，`T`是目标类型(值为`t`)：
- 如果 `F` 是 `T`的相同，或者子类型， 则总是成功的。`t = f`
- 如果存在一个隐式转换，则总是成功的。`t = implicit_convert(f)`
- 如果 存在 `t = f.copyTo[T]` 是类型安全的，那么，会使用这个转换。 `t = f.copyTo[T]`
- 如果 `F`和`T`是容器类型 `X[A]` 和 `X[B]`，并且 `A` 和 `B` 可以满足上述的转换关系，则 ` f = t.map ( A -> B )`
- 如果 `F` 和 `T`都是Case Class，并且可以递归使用  `t = BeanBuilder.build[T](f)()`

如果单纯根据如上的规则，可能很多的同学会觉得，这么复杂的一套规则，在Java中要么是完全做不到的（大部分），要么是这会产生巨大的
运行时性能损耗，会存在严重的性能问题。那么，最最神奇的事情发生了：

**BeanBuilde会是同类框架中性能最优的，它能够达到数据复制的理论最快速度，在没有任何性能损耗的情况下，实现类型安全的对象复制**

为什么作者可以这么自信的说呢？这就是Macro的神奇，与BeanUtils/Orika等不同的是， BeanBuilder会在编译期间，完成所有的类型
检查工作，完成源对象与目标对象之间的映射关系检查和建立，为你自动的编写出Copy代码，就像手工编写的转换代码代码一样。你或者可以
理解为 BeanBuilder 是一个扩展了的 scalac 编译特性，当然这一切建立在 scala 的标准语言特性之上。

经过前面系列的铺垫，相信读者已经具备了看懂这个代码的基础，BeanBuilder的全部源代码也很简单，约200行，具体内容就不在本文中
详细陈述了，读者如有兴趣，可以查看：https://github.com/wangzaixiang/scala-sql/blob/master/src/main/scala/wangzx/scala_commons/sql/BeanBuilder.scala

通过 BeanBuilder 这个案列，相信读者能够感知到一个完全不同的思维模式，打开新的想象空间。而这，就是 元编程 的魅力之所在吧。

## 剧透
- dapeng 2.0 即将全新发布，新版本将是对1.0版本的完全重构，对简化代码结构的情况下，提供了全新的异步支持、Scala服务开发支持。
我们的主力工程师 @EmuuGrass(美女)、@jackliang 后续会准备不少的技术分享。
- dapeng 2.0 将内置一个**极速json解析器**，dapeng-soa将在支持thrift binary/compact binary的基础之上，提供**极速**的JSON服务。
等待@ever大神的新博文分享。





