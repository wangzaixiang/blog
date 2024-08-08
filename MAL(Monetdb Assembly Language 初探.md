MonetDB 是一个基于列存的 OLAP 数据库引擎，最近看到的很多 OLAP 引擎，包括 [DuckDB](https://duckdb.org)、[Polars](https://pola.rs) 
都引用了MonetDB的研究论文，这说明了 MonetDB 在 OLAP 领域的权威地位，我对 MonetDB 的实现原理也很感兴趣，本文尝试对 MAL 做一个初步的探究。

MonetDB 引入了一个 MAL 的组件（Monet Assembly Language），即将SQL语言首先编译成为一种中间语言（IL)，然后再执行这个IL，这与 sqlite3 
是非常相似的。在 SQLite3中，这个IL 又成为 bytecode，相关资料可以参考：[为什么SQLite使用 Bytecode](https://sqlite.org/whybytecode.html)。
而在 MonetDB 中，则称之为 MAL。

本文并不对 MAL 的内部实现进行深度分析，而是尝试通过一些简单的案例，来理解最终的 mal 是什么样子，是如何对应到 SQL 语句的？以终为始，如果能读懂mal
代码，则后续可以进一步理解 MonetDB 的其他组件。

# 准备工作
首先参考 [Tutorial](https://www.monetdb.org/documentation-Dec2023/user-guide/tutorials/voc-tutorial/) 在本地搭建好环境和测试数据库。
```bash
$ brew install monetdb

$ monetdbd create /path/to/mydbfarm  -- 初始化一个数据库实例目录
$ monetdbd start /path/to/mydbfarm   -- 启动该实例
$ monetdb create voc                 -- 创建一个数据库，处在维护模式
$ monetdb release voc                -- 将数据库切换到可用模式

$ wget https://dev.monetdb.org/Assets/VOC/voc_dump.zip -- 获取到示例数据库的数据脚本
$ unzip -p voc_dump.zip voc_dump.sql | mclient -u voc -d voc  -- 执行脚本，初始化示例数据库
```

# 简单案例
我们先来阅读一条最简单的 SQL 语句的 MAL。
```bash
$ mclient -u voc -d voc
sql> explain select * from invoices where chamber = 'A';

+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| mal                                                                                                                                                                     |
+=========================================================================================================================================================================+
| function user.main():void;                                                                                                                                              |
|     X_1:void := querylog.define("explain select * from invoices where chamber = 'A';":str, "default_pipe":str, 36:int);                                                 |
| barrier X_121:bit := language.dataflow();                                                                                                                               |
|     X_4:int := sql.mvc();                                                                                                                                               |
|     C_5:bat[:oid] := sql.tid(X_4:int, "sys":str, "invoices":str);                                                                                                       |
|     X_8:bat[:int] := sql.bind(X_4:int, "sys":str, "invoices":str, "number":str, 0:int);                                                                                 |
|     X_15:bat[:str] := sql.bind(X_4:int, "sys":str, "invoices":str, "number_sup":str, 0:int);                                                                            |
|     X_20:bat[:int] := sql.bind(X_4:int, "sys":str, "invoices":str, "trip":str, 0:int);                                                                                  |
|     X_27:bat[:str] := sql.bind(X_4:int, "sys":str, "invoices":str, "trip_sup":str, 0:int);                                                                              |
|     X_34:bat[:int] := sql.bind(X_4:int, "sys":str, "invoices":str, "invoice":str, 0:int);                                                                               |
|     X_39:bat[:str] := sql.bind(X_4:int, "sys":str, "invoices":str, "chamber":str, 0:int);                                                                               |
|     C_48:bat[:oid] := algebra.thetaselect(X_39:bat[:str], C_5:bat[:oid], "A":str, "==":str);                                                                            |
|     X_50:bat[:int] := algebra.projection(C_48:bat[:oid], X_8:bat[:int]);                                                                                                |
|     X_51:bat[:str] := algebra.projection(C_48:bat[:oid], X_15:bat[:str]);                                                                                               |
|     X_52:bat[:int] := algebra.projection(C_48:bat[:oid], X_20:bat[:int]);                                                                                               |
|     X_53:bat[:str] := algebra.projection(C_48:bat[:oid], X_27:bat[:str]);                                                                                               |
|     X_54:bat[:int] := algebra.projection(C_48:bat[:oid], X_34:bat[:int]);                                                                                               |
|     X_55:bat[:str] := algebra.projection(C_48:bat[:oid], X_39:bat[:str]);                                                                                               |
|     X_123:void := language.pass(C_48:bat[:oid]);                                                                                                                        |
|     X_124:void := language.pass(X_39:bat[:str]);                                                                                                                        |
|     X_57:bat[:str] := bat.pack("sys.invoices":str, "sys.invoices":str, "sys.invoices":str, "sys.invoices":str, "sys.invoices":str, "sys.invoices":str);                 |
|     X_58:bat[:str] := bat.pack("number":str, "number_sup":str, "trip":str, "trip_sup":str, "invoice":str, "chamber":str);                                               |
|     X_59:bat[:str] := bat.pack("int":str, "char":str, "int":str, "char":str, "int":str, "char":str);                                                                    |
|     X_60:bat[:int] := bat.pack(32:int, 1:int, 32:int, 1:int, 32:int, 1:int);                                                                                            |
|     X_61:bat[:int] := bat.pack(0:int, 0:int, 0:int, 0:int, 0:int, 0:int);                                                                                               |
| exit X_121:bit;                                                                                                                                                         |
|     X_56:int := sql.resultSet(X_57:bat[:str], X_58:bat[:str], X_59:bat[:str], X_60:bat[:int], X_61:bat[:int], X_50:bat[:int], X_51:bat[:str], X_52:bat[:int], X_53:bat[ |
: :str], X_54:bat[:int], X_55:bat[:str]);                                                                                                                                 :
| end user.main;                                                                                                                                                          |
| # inline                               0 actions 3 usecs                                                                                                                |
| # remap                                0 actions 2 usecs                                                                                                                |
| # costModel                            1 actions 2 usecs                                                                                                                |
| # coercions                            0 actions 5 usecs                                                                                                                |
| # aliases                              1 actions 9 usecs                                                                                                                |
| # evaluate                             0 actions 8 usecs                                                                                                                |
| # emptybind                            6 actions 9 usecs                                                                                                                |
| # deadcode                             6 actions 9 usecs                                                                                                                |
| # pushselect                           0 actions 18 usecs                                                                                                               |
| # aliases                              6 actions 4 usecs                                                                                                                |
| # for                                  0 actions 1 usecs                                                                                                                |
| # dict                                 0 actions 8 usecs                                                                                                                |
| # mitosis                                                                                                                                                               |
| # mergetable                           0 actions 9 usecs                                                                                                                |
| # aliases                              0 actions 1 usecs                                                                                                                |
| # constants                            0 actions 7 usecs                                                                                                                |
| # commonTerms                          0 actions 10 usecs                                                                                                               |
| # projectionpath                       0 actions 4 usecs                                                                                                                |
| # deadcode                             0 actions 3 usecs                                                                                                                |
| # matpack                              0 actions 1 usecs                                                                                                                |
| # reorder                              1 actions 11 usecs                                                                                                               |
| # dataflow                             1 actions 17 usecs                                                                                                               |
| # querylog                             0 actions 1 usecs                                                                                                                |
| # multiplex                            0 actions 1 usecs                                                                                                                |
| # generator                            0 actions 3 usecs                                                                                                                |
| # candidates                           1 actions 2 usecs                                                                                                                |
| # deadcode                             0 actions 4 usecs                                                                                                                |
| # postfix                              0 actions 5 usecs                                                                                                                |
| # profiler                             0 actions 1 usecs                                                                                                                |
| # garbageCollector                     1 actions 9 usecs                                                                                                                |
| # 29 optimizers 224 usecs                                                                                                                                               |
+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
59 tuples

```

下面我们会参考 [MAL Modules](https://www.monetdb.org/documentation-Dec2023/dev-guide/monetdb-internals/mal-modules/)中
的API文档为 MAL 代码添加上注释：

```
function user.main():void;
     X_1:void := querylog.define("explain select * from invoices where chamber = 'A';":str, "default_pipe":str, 36:int);

 barrier X_121:bit := language.dataflow();  // The current guarded block is executed using dataflow control.

     // Get the multiversion catalog context.
     // Needed for correct statement dependencies
     // (ie sql.update, should be after sql.bind in concurrent execution
     X_4:int := sql.mvc();

     // Return a column with the valid tuple identifiers associated with the table sname.tname.
     C_5:bat[:oid] := sql.tid(X_4:int, "sys":str, "invoices":str);

     // Bind the 'schema.table.column' BAT with access kind:
     // 0 - base table
     // 1 - inserts
     // 2 - updates
     //// 对应于 select * 中的六列
     X_8:bat[:int] := sql.bind(X_4:int, "sys":str, "invoices":str, "number":str, 0:int);
     X_15:bat[:str] := sql.bind(X_4:int, "sys":str, "invoices":str, "number_sup":str, 0:int);
     X_20:bat[:int] := sql.bind(X_4:int, "sys":str, "invoices":str, "trip":str, 0:int);
     X_27:bat[:str] := sql.bind(X_4:int, "sys":str, "invoices":str, "trip_sup":str, 0:int);
     X_34:bat[:int] := sql.bind(X_4:int, "sys":str, "invoices":str, "invoice":str, 0:int);
     X_39:bat[:str] := sql.bind(X_4:int, "sys":str, "invoices":str, "chamber":str, 0:int);

     // Select all head values of the first input BAT for which the tail value
     // obeys the relation value OP VAL and for which the head value occurs in
     // the tail of the second input BAT.
     // Input is a dense-headed BAT, output is a dense-headed BAT with in
     // the tail the head value of the input BAT for which the
     // relationship holds.  The output BAT is sorted on the tail value.
     //// θ select 进行过滤
     C_48:bat[:oid] := algebra.thetaselect(X_39:bat[:str], C_5:bat[:oid], "A":str, "==":str);

     // Project left input onto right input
     //// X_50 到 X_55 就是从过滤后的 oid 映射到对应的列的值
     X_50:bat[:int] := algebra.projection(C_48:bat[:oid], X_8:bat[:int]);
     X_51:bat[:str] := algebra.projection(C_48:bat[:oid], X_15:bat[:str]);
     X_52:bat[:int] := algebra.projection(C_48:bat[:oid], X_20:bat[:int]);
     X_53:bat[:str] := algebra.projection(C_48:bat[:oid], X_27:bat[:str]);
     X_54:bat[:int] := algebra.projection(C_48:bat[:oid], X_34:bat[:int]);
     X_55:bat[:str] := algebra.projection(C_48:bat[:oid], X_39:bat[:str]);

     // Cheap instruction to disgard storage while retaining the dataflow dependency
     X_123:void := language.pass(C_48:bat[:oid]);
     X_124:void := language.pass(X_39:bat[:str]);

     // Materialize the values into a BAT. Avoiding a clash with mat.pack() in mergetable
     //// resultset column schema 信息
     X_57:bat[:str] := bat.pack("sys.invoices":str, "sys.invoices":str, "sys.invoices":str,
                                "sys.invoices":str, "sys.invoices":str, "sys.invoices":str);
     //// resultset column name 信息
     X_58:bat[:str] := bat.pack("number":str, "number_sup":str, "trip":str,
                                "trip_sup":str, "invoice":str, "chamber":str);
     //// resultset column datatype 信息
     X_59:bat[:str] := bat.pack("int":str, "char":str, "int":str, "char":str, "int":str, "char":str);

     //// 应该是column的元信息，具体是什么，暂时还不清楚
     X_60:bat[:int] := bat.pack(32:int, 1:int, 32:int, 1:int, 32:int, 1:int);

     //// 应该是column的元信息，具体是什么，暂时还不清楚
     X_61:bat[:int] := bat.pack(0:int, 0:int, 0:int, 0:int, 0:int, 0:int);

 exit X_121:bit;

     // Prepare a table result set for the client front-end
     X_56:int := sql.resultSet(X_57:bat[:str], X_58:bat[:str], X_59:bat[:str], X_60:bat[:int], X_61:bat[:int],
            X_50:bat[:int], X_51:bat[:str], X_52:bat[:int], X_53:bat[:str], X_54:bat[:int], X_55:bat[:str]);                                                                                                                                 :

end user.main;

```
整理后的处理如图：<img width="1362" alt="image" src="https://github.com/user-attachments/assets/fcc66d79-a46b-4955-ae9f-1d750c3bd46a">

# group by 案例
```
sql> explain explain select chamber, sum(trip) from invoices group by chamber;

function user.main():void;
    X_1:void := querylog.define("explain select chamber, sum(trip) from invoices group by chamber;":str,
    			"default_pipe":str, 21:int);
barrier X_91:bit := language.dataflow();
    X_4:int := sql.mvc();

    C_5:bat[:oid] := sql.tid(X_4:int, "sys":str, "invoices":str);

    trip_0:bat[:int] := sql.bind(X_4:int, "sys":str, "invoices":str, "trip":str, 0:int);
    chamber_0:bat[:str] := sql.bind(X_4:int, "sys":str, "invoices":str, "chamber":str, 0:int);

    trip_1:bat[:int] := algebra.projection(C_5:bat[:oid], trip_0:bat[:int]);
    chamber_1:bat[:str] := algebra.projection(C_5:bat[:oid], chamber_0:bat[:str]);

    X_93:void := language.pass(C_5:bat[:oid]);

	// 这个函数文档没有找到，返回的两个 BAT 暂时语义不确定，猜测： X_22: [1..N, gid], X_23: [1,n, oid]
	// 其中 N 是行数， n 是分组数。
    (X_22:bat[:oid], C_23:bat[:oid]) := group.groupdone(chamber_1:bat[:str]);

    //// grouped [chamber]
    chamber_2:bat[:str] := algebra.projection(C_23:bat[:oid], chamber_1:bat[:str]);

    X_94:void := language.pass(chamber_1:bat[:str]);

    //// [ sum(trip) for each group ]  subsum(bid, gid, eid, skip_nils) 这个函数在官网搜不到
    trip_sum:bat[:hge] := aggr.subsum(trip_1:bat[:int], X_22:bat[:oid], C_23:bat[:oid], true:bit);

    X_95:void := language.pass(C_23:bat[:oid]);

    X_29:bat[:str] := bat.pack("sys.invoices":str, "sys.%1":str);
    X_30:bat[:str] := bat.pack("chamber":str, "%1":str);
    X_31:bat[:str] := bat.pack("char":str, "hugeint":str);
    X_32:bat[:int] := bat.pack(1:int, 128:int);
    X_33:bat[:int] := bat.pack(0:int, 0:int);
exit X_91:bit;
    X_28:int := sql.;(X_29:bat[:str], X_30:bat[:str], X_31:bat[:str], X_32:bat[:int], X_33:bat[:int],
    	chamber_2:bat[:str], trip_sum:bat[:hge]);
end user.main;
```
# 窗口函数案例
TODO

# join 案例
TODO

# 使用一个 IL 来执行 SQL 有什么优点？
Sqlite3 使用字节码，MonetDB 使用 MAL，二者是相似的，而大部份其他的数据库是基于AST的，通过将 SQL 解析成为 AST，再转换成为 Logic Plan 和 Physical Plan (也是 AST)。
在 [为什么SQLite使用 Bytecode](https://sqlite.org/whybytecode.html) 文中提到了
使用字节码的优点和缺点：
Pros:
  - 字节码易于理解。而 AST 相对难以展示为可读性好的表现形式。（实际上，现在很多数据库都可以以 Tree 方式来查看 Plan，这方面的差距其实不大）
  - 字节码易于调试。可以像普通的代码调试一样的进行字节码的调试。
  - 字节码可以增量式的运行。
  - 字节码更加短小。
  - 字节码更加快速。字节码相对于AST来说具有更小的解释成本
Cons:
  - AST 更便于在执行期后期的计划变更。尤其是根据实际数据的特征，而改变执行计划，例如调整 Join 策略等。
  - 基于数据流的处理更易于并行化。

我觉得还有一个点，是字节码模式更擅长的，可以更好的进行自下而上的性能优化，例如我们可以针对某个特定的优化场景，设计更适合的算法，基于字节码的场景，我们可以略过前面的 AST 构建过程，
直接手写出目标的字节码，然后独立的进行调试，进行性能压测。在达到预期目标后，调整前端的 AST 构建过程即可。

1. 在此之外，我们可以引入一个函数式的 IL，采用类似于 S-Expression 的方式，这样 AST 与 S-IL之间的距离也相对较小，可以更方便引入新的底层函数。
2. 基于 S-IL 的模式，还可以引入 Hot-Swap 的技术，即在运行期可以动态的修改部分的函数调用，这就可以解决上面的后期优化的问题。（有些类似于 JVM 的 JIT 替换的模式）
3. 从学习的角度来看，将 IL 的执行与 AST 的生成进行隔离，也比较便于对 Plan 和 Executor 两个关键组件的解耦，可读性、可扩展性都会更好一些。
