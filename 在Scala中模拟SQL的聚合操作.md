最近在尝试基于内存计算来替代数据库操作，这样做自然有利有弊，利的一块是如果寻觅到合适的解决方案，那肯定可以获得数量级的性能提升，
毕竟，我们可以采用更为灵活的数据结构，而不必拘泥于关系模型和SQL的表达能力，这自然是极富有想象空间的。而且，节省下来的IO开销，
可不是小数字。不过，与简单的关系模型 + SQL相比，复杂性上升也是数量级的。

如果是直接在Java这样的语言之中，基于JDK或者标准库提供的丰富数据结构，是可以表达足够强大的数据能力了，但是一个简单地SQL操作，
可能就需要大量的代码才能实现，这样的开发成本（包括变更、维护成本）会成为一个很大的障碍（毕竟，这年头，还能玩玩基础数据结构
的工程师已经很少了）。可惜Java中没有Linq这样的框架，scala中的filter/map/flatMap是可以解决很多的问题，但要进行一些诸如聚合、
分组等操作，现有的库还是不够。

```scala
  case class GroupKey(category: String, designation: String, manufacturerName: String)

  case class Price(category: String, designation: String, manufacturerName: String,
                   city: String, price: BigDecimal, weight: BigDecimal,
                   publish_date: Date, expire_date: Date, updated_at: Date,
                   status: Int
                    )
    val grouped: Map[GroupKey, List[Price]] = prices
      .filter(p => p.status == 1 && p.publish_date <= now && p.expire_date >= now)
      .groupBy(p => GroupKey(p.category, p.designation, p.manufacturerName))
```
这个可以模拟一个```sql select * from prices p where p.prices = 1 and p.publish_date <= now() and p.expire_date <= now() ```

```scala
    val query2: List[(GroupKey, Int, BigDecimal, BigDecimal, Date, List[Price])] = profile("query2") {
      val x  = grouped.map { case (gk: GroupKey, prices) =>
        val head = prices.head
        val (count: Int, weight: BigDecimal, price: BigDecimal, updated_at) =
          prices.foldLeft(0, 0: BigDecimal, head.price, head.updated_at) { case ((count, weight, price, updated_at), record) =>
            (count + 1,
              weight + record.weight,
              if (price < record.price) price else record.price,
              if (updated_at <= record.updated_at) record.updated_at else updated_at
              )
          }
        (gk, count, weight, price, updated_at, prices)
      } ( List.ReusableCBF.asInstanceOf[CanBuildFrom[Map[GroupKey,List[Price]], (GroupKey,Int,BigDecimal,BigDecimal, Date, List[Price]), List[(GroupKey, Int, BigDecimal, BigDecimal, Date, List[Price])]]] )

      x.sortBy(0 - _._5.getTime).drop(i*10).take(10)
    }
```
这段代码视图模拟这样的```sql select category, designation, manufacturerNane, count(1), sum(weight), min(price), max(updated_at) from prices group by category, designation, manufacrurerName order by max(updated_at) desc limit 100,10```

不过，这段代码的抽象度不高，相比SQL而言，实在是复杂多了，虽然还在一个可以勉强接受的范围之内，毕竟，如果在Java中实现，
估计更要吐血了。

接下来，我们看看抽象度更高的一个版本：
```scala
grouped.map { case (gk: GroupKey, prices: List[Price]) =>
  val (cnt, weight, price, updated_at) = prices.reduce( COUNT(_), SUM(_.weight), MIN(_.price), MAX(_.updated_at) )
  (gk, cnt, weight, price, updated_at)
}.toList.sortBy(0 - _5.getTime).drop(100).take(10)
```
这个版本的可读性就高了不少，也更接近于SQL语法了。

在这里，COUNT, SUM等都是一些聚合原语，其一个实现参考如下：
```scala
  abstract class ReduceOp[In, Acc] {
    def init: Acc
    def apply(acc: Acc, in: In): Acc
  }
  case class MIN[In, B](f: In => B)(implicit num: Numeric[B]) extends ReduceOp[In, B] {
    def init = num.zero
    def apply(acc: B, in: In) = {
      val it: B = f(in)
      if(num.compare(it, acc) < 0) it else acc
    }
  }
```
其它的SUM、MAX实现也是很简单的，这里就不贴代码了。

这里只是做一个简单的SQL的scala模拟，后续我回再整理一下，或许是一个不错的库实现。

不过，谈到做一个有价值的内存计算服务，还有一些问题是需要解决的：
1、直接基于Java的对象模型并不适合于进行大内存的、对实时性要求很高的场合，主要是受GC的限制，缓存内的应用和内存式数据库
和GC是矛盾的。解决的办法就是使用heap-off heap，不过，这样的问题就是Java的对象模型都应用不了。我们可以在heap-off的基础上，
开发出一套有用的数据结构出来，这样，就可以在此基础上，进行灵活的运算了。
2、在heap-off的基础上，我们有机会创建类似于redis、或者sql-like的自己的计算模型，结合函数式编程语言，我们完全可以以一个较低
的成本来实现有很高性能需要的应用来。

这是一个很直得一玩的题目，或许后续我们会有机会应用得上。
