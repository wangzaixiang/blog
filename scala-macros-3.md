神奇的Scala Macro之旅（3）
===

再上一篇中，我们示范了使用macro来重写 Log 的 debug/info 方法，并大致的介绍了 macro 的基本语法、基本使用方法、以及macro背后的一些概念，
如AST等。那么，本篇中，我们将结合作者在 scala-sql 项目中的一些实际应用，向你展示 macro 可以用来做什么？怎么用？

## scala-sql 简介
scala-sql 是一个轻量级的 JDBC 库，提供了在scala中访问关系型数据库的一个简单的API，其定位是对面 scala开发者，提供一个可以替换 
spring-jdbc, iBatis 的数据访问库。相比 spring-jdbc, iBatis 等库，scala-sql有一些自己独特的特点：

1. 面向scala语言。因此，如果选择 java 或者其他编程语言，scala-sql基本上没有意义。而spring-jdbc, iBatis等显然不受这个限制。
2. 概念简单。 scala-sql 为 java.sql.Connection, javax.sql.DataSource 等对象扩展了：`executeUpdate`、`rows`、`foreach`等方法，
但其语义完全与 jdbc 中的概念是一致的。 熟悉jdbc的程序员，并且熟悉scala语法的程序员，scala-sql基本上没有新的概念需要学习。
3. 强类型。 scala-sql目前是2.0版本，使用了sql"sql-statement" 的插值语法来替代了Jdbc中复杂的 setParameter 操作，并且是强类型和
可扩展的。
    - 强类型：如果你试图传入 URL、FILE 等对象时，scala编译器会检测错误，提示无效的参数类型。
    - 可扩展：不同于JDBC，只能使用固定的一些类型，scala-sql通过Bound Context:`JdbcValueAccessor`来扩展，也就是说，如果你定义了
    对应的扩展，URL、FILE也是可以作为传给JDBC的参数的。（当然，需要在`JdbcValueAccessor`中正确的进行处理）。
4. 函数式支持。scala-sql支持从ResultSet到Object的映射，在1.0版本中，是映射到JavaBean，而在2.0中，则是映射到Case Class。选择
Case Class的原因也是为了更好的支持函数式编程。（支持Case Class与函数式编程有什么关系？函数式编程的精髓就是无副作用的值变换，
immutable才是函数式编程的真爱）
5. 编译期间的SQL语法检查。 这个特性可以让开发变得更加便捷一些，如果有SQL语法错误，或者存在错误拼写的字段名、表名等情况，在编译时期就能
发现错误，可以更快的暴露错误，缩短测试周期。

上面的几个特性中，从ResultSet到Case Class的映射、以及编译时期的SQL语法检查，都是依赖scala macro来完成的。

## 使用Macro 来支持 ResultSet 到 Case Class的映射
在scala-sql 1.0版本中，可以使用如下的代码：
```scala

class User {
  var name: String = _
  var age: Int = _
  var address: String = _
}

val name = "Q%"
val users: List[User] = dataSource.rows[User](sql"select * from users where name like $name")

```
在scala-sql 1.0版本中，是通过反射的方式来完成这个映射的。除了是使用mutable的数据结构这个缺点（对FP不友好），通过反射来完成映射，有
以下的缺点：
    - 性能损失。（通过使用cache来存储反射信息，可以降低性能损失到一个不需要关注的水平）
    - 类型检查。如果user中存在不适合映射的字段类型，那么只有在运行期间才能了解到。
    
scala-sql 2.0版本决定支持Case Class（并废弃掉对JavaBean的支持），Case Class数immutable的，也无法通过反射的方式在运行时动态的进行
对象的构建，要解决这个问题，我们只能为每一个Case Class在编译时期静态的生成一个从ResultSet到Case Class的映射。
```scala
  case class User(name:String, age:Int, address:String)
  
  trait ResultSetMapper[T] {
    def from(rs: ResultSet): T
  }
```
对某个Case Class比如`User`来说，我们需要有一个`ResultSetMappper[User]`的实现，如果手动的写代码，这个代码会形如：
```scala
    /**
      * the base class used in automate generated ResultSetMapper.
      */
    abstract class CaseClassResultSetMapper[T] extends ResultSetMapper[T] {
  
      case class Field[T: JdbcValueAccessor](name: String, default: Option[T] = None) {
        def apply(rs: ResultSetEx): T = {
          if ( rs hasColumn name ){
            rs.get[T](name)
          }
          else {
            default match {
              case Some(m) => m
              case None => throw new RuntimeException(s"The ResultSet have no field $name but it is required")
            }
          }
        }
      }
    }
  object UserMapper extends CaseClassResultSetMapper[User] {

    override def from(rs: ResultSet): User = {
      val NAME = Field[String]("name")
      val AGE = Field[Int]("age")
      val CLASSROOM = Field[Int]("classRoom")

      User(NAME(rs), AGE(rs), CLASSROOM(rs))
    }
    
  }
```
但是，如果对每一个Case Class都需要定义对应的Mapper的话，这个事情就变得不那么美好了：程序员都更喜欢做一些有挑战性的任务，而对这写近乎
Copy-Paste的任务或者不擅长，或者不乐意。当然，大量的Copy-Paste代码实际上也会构成工程的灾难，当某个点需要修改的时候，这简直就是灾难。
所以，在编程实践领域，有"重复的代码是最大的质量问题"这一说法，也是非常有道理的。

Macro实际上就是一种有效消除这类Copy-Paste代码的利器，如果有一个macro，能够根据定义的Case Class，自动的生成我们需要的上面的代码，同时，
还可以在生成代码的同时，做一些类型检查，在编译时候就能发现错误，例如，如果某个字段是无法从ResultSet映射的，则可以在编译时期报错。

实际上，这也正式macro的应用场景。

```scala
  
  object ResultSetMapper {
    implicit def material[T]: ResultSetMapper[T] = macro Macros.generateCaseClassResultSetMapper[T]
  }

  object Macros {
  
    def generateCaseClassResultSetMapper[T: c.WeakTypeTag](c: scala.reflect.macros.whitebox.Context): c.Tree = {
      import c.universe._
  
      val t: c.WeakTypeTag[T] = implicitly[c.WeakTypeTag[T]] // 获取到类型参数 T 对应的TypeTag
      assert( t.tpe.typeSymbol.asClass.isCaseClass, s"only support CaseClass, but ${t.tpe.typeSymbol.fullName} is not" )
  
      val companion = t.tpe.typeSymbol.asClass.companion  // 获取到 T 对应的伴生对象的类型，在这里，主要是获取Case Class某个字段的缺省值
  
      // 通过获取 Case Class的主构造方法，来获取Case Class的字段定义
      val constructor: c.universe.MethodSymbol = t.tpe.typeSymbol.asClass.primaryConstructor.asMethod
  
      var index = 0
      val args: List[(c.Tree, c.Tree)] = constructor.paramLists(0).map { (p: c.universe.Symbol) =>
        val term: c.universe.TermSymbol = p.asTerm
        index += 1
  
        // search "apply$default$X"
        val name = term.name.toString
        val newTerm = TermName(name.toString)
  
        val tree = if(term.isParamWithDefault) { // 这个字段有缺省值
          val defMethod: c.universe.Symbol = companion.asModule.typeSignature.member(TermName("$lessinit$greater$default$" + index))
          q"""val $newTerm = Field[${term.typeSignature}]($name, Some($companion.$defMethod) )"""
        }
        else
          q"""val $newTerm = Field[${term.typeSignature}]($name) """
  
        (q"""${newTerm}(rs)""", tree)
      }
  
      q"""
         import wangzx.scala_commons.sql._
         import java.sql.ResultSet
  
         new CaseClassResultSetMapper[$t] {
           ..${args.map(_._2)}  // 定义各个字段的提取器
  
           override def from(arg: ResultSet): $t = {
             val rs = new ResultSetEx(arg)
             new $t( ..${args.map(_._1) } )
           }
         }
      """ 
    }

  }
```

由于使用了 Quasiquote(http://docs.scala-lang.org/overviews/quasiquotes/intro.html)，可以更为方便的处理AST（包括构造AST，和从AST中提取信息），
这段代码还是比较简洁的。一些相关的API，读者可以通过：[Scala Macros](http://docs.scala-lang.org/overviews/macros/usecases.html)、
[Scala Reflection](http://docs.scala-lang.org/overviews/reflection/overview.html)、
[Scala Quasiquotes](http://docs.scala-lang.org/overviews/quasiquotes/setup.html)等文档做进一步的了解。

那么，scala-sql又是如何实现对类型的检查的呢？实际上，上面的这段代码自身并没有对Case Class的字段的类型进行检查，而是通过生成的代码：
`val NAME = Field[String]("name")` 来完成类型检查的，如果字段的类型，譬如为 URL， 则`val NAME = Field[URL]("name")`将无法通过编译
检查。因为在`Field[T: JdbcValueAccessor]`的构造中，所有的数据类型必须同时具备`bound context: JdbcValueAccessor`，即存在一个
`implicit val anyName: JdbcValueAccessor[T]`的隐式值，由这个值来负责处理从 T 到 SQL（包括parameter, ResultSet）的访问操作。

当然，我们也可以在macro中，显示的对T进行检查，如果不符合，则显示一个更为友好的编译错误：例如，`字段 name 的类型 URL，不是合法的数据库字段类型`,
这也是可以做到的，我们将会在后续的案列中，对其进行示范。

在我们的这个案例中，我们使用Macro来生成Case Class的ResultSetMapper，如果我们要为Case Class来提供JSON、XML映射，也可以使用同样的方式来
实现，相比通过反射方式（如GSON、fastjson）等，macro可以生成更高效的代码，同时，给到我们更好的类型安全支持。

## 使用 Macro 在编译期检查SQL语法 

```scala
 implicit class SQLStringContext(sc: StringContext) {
    def sql(args: JdbcValue[_]*) = SQLWithArgs(sc.parts.mkString("?"), args)

    /**
     * SQL"" will validate the sql statement at compiler time.
     */
    def SQL(args: JdbcValue[_]*): SQLWithArgs = macro  Macros.parseSQL
  }
  
  object Macros {
    def parseSQL(c: reflect.macros.blackbox.Context)(args: c.Tree*): c.Tree = {
      import c.universe._
   
      // 提取 SQL"..."中的sql字符串常量，这是我们需要校验的sql语句
      val q"""$a(scala.StringContext.apply(..$literals))""" = c.prefix.tree
  
      val it: c.Tree = c.enclosingClass
      // 从当前代码所在类的 @db(name="dbname") 中提取当前代码的db名称，主要是如果在项目中
      // 可能访问多个数据库时，确定当前sql的校验是通过哪个数据库进行。
      val db: String = it.symbol.annotations.flatMap { x =>
        x.tree match {
          case q"""new $t($name)""" if t.symbol.asClass.fullName == classOf[db].getName =>
            val Literal(Constant(str: String)) = name
            Some(str)
          case _ => None
        }
      } match {
        case Seq(name) => name
        case _ => "default"
      }
  
  
      val x: List[Tree] = literals
      
      // 提取字符串常量
      val stmt = x.map { case Literal(Constant(value: String)) => value }.mkString("?")
  
      try {
        // 连接到数据库进行语法检查。
        SqlChecker.checkSqlGrammar(db, stmt, args.length)
      }
      catch {
        case ex: Throwable =>
          // 有错误时，报告编译错误。
          c.error(c.enclosingPosition, s"SQL grammar erorr ${ex.getMessage}")
      }
  
      q"""wangzx.scala_commons.sql.SQLWithArgs($stmt, Seq(..$args))"""
    }
  }
```

这个例子相比上一个示例，增加了如下的特性：

- 提取当前代码中的更多信息，例如，获取当前定义类的 @db 标注信息。
- 提取当前代码中的 字符串常量，对这些字符串进行进一步的检查。在本例中，我们是通过在编译时期，连接到一个本地的数据库，在这个数据库上，
模拟的执行当前的SQL语句，既可以检查语法，还可以检查其中表名、字段名等标识符的拼写是否正确。

类似的，我们可以编写macro，对很多的字符串进行进一步的检查。 例如：
- 对正则表达式进行语法检查。
- 对日期格式进行语法检查
而这些在传统的代码中，都是在运行期进行的，提前到编译器进行，不仅可以让代码变得更安全，而且也更符合敏捷的思路，Let it fail fast.
