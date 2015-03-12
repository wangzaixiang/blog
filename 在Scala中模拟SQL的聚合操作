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
