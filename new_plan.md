##New Plan 介绍
先明确一下概念，这里提到的 new plan 实际上并不是特指 plan 而是包括 parser, optimizer, executor 的一个总称。
先大概交待一下 old plan 的执行流程，parser 直接 Compile 生成了一个 Statement，调用 Statement 的 Exec 方法得到一个 Recordset，调用 Recordset 的 Next 方法来读取结果。

new plan 的 Compile 分成了两个步骤，第一步是 Parse，得到一个 ast.StmtNode,
这个时候，只是简单的解析，没有做任何的调整和优化。第二步是执行 executor 包里的 Compiler 的 Compile方法, 把一个 ast.StmtNode 编译成一个 stmt.Statement。

因为重构整个执行流程，改动非常的大，不可能一下子把所有功能都支持，所以一段时间内，新版和旧版是共存的状态，新版会从最简单的功能开始支持，慢慢地支持更多的查询类型，在 Compile 语句里，会判断是否是新版已经支持的语句，如果支持，会用新版的来执行这条语句，如果不支持，就用转换器把 ast.StmtNode 转换成旧版的 stmt/stmts 的类型，用旧的逻辑来执行。

如果用新版的执行流程，Compile 做的工作是调用 optimizer.Optimize 方法把一个 ast.StmtNode 转换成一个 plan，然后用 statementAdapter 包装一下，作为一个 stmt.Statement 返回。

因为 stmt.Statement 只是一个 interface，依赖很少，所以为了方便平滑的升级，新版保留了这个 interface，用一个 adapter 来实现它的接口。

statementAdapter 实现了 Exec 方法，返回一个 rset.Recordset，这里我们返回的是一个 recordsetAdapter，它把 executor.Executor 封装了一下。

旧版的 plan/plans 里的文件，虽然名为 plan，但是做的实际上做的是 executor 的事情，新版把这部分逻辑放在了 executor 包里，所以 executor/executor.go 里的代码大部分是从 plan/plans 包里复制过来的。新版本的 plan 放在 optimizer/plan 里，是一个和旧版 plan/plans 完全不同的概念。

##New Plan 流程
New Plan 的逻辑代码实现主要放在 optimizer 包和 optimizer/plan 包里。

下面讨论一下 New Plan 执行 SQL 的流程。

* optimizer.Validate，验证语句的合法性，因为 parser 只能保证符合基本的语法规则，更高级的规则需要在整个语句解析完成后才能判断，    比如 BinaryOperation 的两个 Operand 的 Column 个数必须是相等的。

* optimizer.Preprocess
    * optimizer.ResolveName，创建 ResultField, 并找到 ColumnNameExpr 指向的 ResultField，方法名叫 ResolveName。
ResultField 是一个非常关键的元素，一个ResultField在创建之后，会一直用到最后，直到语句执行完成，不会被clone，只会被引用。它在语法解析的时候并不存在，在这一步被创建出来。创建 ResultField 的地方有两处，一个是 From，另一个是 Select list。	From 里的 ResultField 就是 Table 所有的 Column， Select List 的 ResultField，可能是一个简单的Value，也可能是 ColumnNameExpr, 或包含 ColumnNameExpr 的 expression。
我们在将来执行的时候，每一次调用 Next 方法，从 table 里读到一行数据，都会把这行数据更新到用这个 table 创建出来的 ResultField 里，保证 ResultFeild 的 Value 是当前这一行的 Value, 所以只要 ColumnNameExpr 找到了它指向的 ResultField，就可以得到当前这一行的 Value 了。
具体的实现比较复杂，因为一个 ColumnName 在语句不同的 位置，指向的规则是不同的。


* optimizer.Optimize
    + optimizer.InferType，计算 ast.ExprNode 的类型。一个 ast.ExprNode 的类型在解析的时候，很多情况下是不确定的，如果包含一个 ColumnNameExpr, 因为不知道 ColumnNameExpr 里指向的 Column 的类型，就无法确定，在上一步执行完成一行，类型就可以确定下来了。

    + optimizer.logicOptimize

        + preEvaluate 会把 static expression 用 ValueExpr 来替换，会把可以求值的表达式求值。

    + plan.BuildPlan，执行 BuildPlan 方法构建默认的 plan，这个 plan 是直接的转换，From Table 会转成 TableScan, Sort 会把 所有的数据读出来，在内存里排序，如果执行的话，开销会很大。在执行 Refine 方法后，如果这个 Table 是 PKIsHandle，会把PK的条件用来过滤 TableScan 的范围，如果 Sort 和 PK 的顺序一样，会忽略 Sort。

    + plan.Alternatives，执行 Alternatives 方法，用默认的 plan 生成多个候选的 plan，每一个 index 会生成一个使用这个 index 的 plan。plan 创建出来以后，调用 Refine 方法，根据 condition 生成 index 的 range，尝试忽略 Sort，设置 Limit。执行完refine以后，这个 plan 就不会再改变了，开销就是固定的了。

    + plan.Refine，Refine 执行之前, IndexScan 是 full range，也就是从头扫到尾，开销是非常大的，执行 Refine 以后，如果 condition 可以限定扫描范围，开销就会大幅的减小。Where condition 有可能是由多个 AND condition 组成的，我们把 AND 拆分成多个 condition, 这样每一个 condition 可以单独用来创建 IndexRange。判断一个condition 是否可以用来创建 IndexRange 的条件是，一端是这个 index 的 ColumnNameExpr, 另一端是 static expression。如果遇到 OR 语句， 需要取两边 range 的 并集，如果是 AND 语句，需要取两边 range 的交集。对于多个 column 组成的 index，先用第一个 column 来创建 range， 第二个 column 在第一个 column 的条件为等于某个 value 之后，进一步限定索引的 range。如果 Sort 的顺序和 Index 是一致的 Sort 会被设置为 Bypass，不会增加任何的开销。但是因为底层还不支持反向遍历，所以如果是降序的，Sort 不会 Bypass。如果有 Limit 会尝试着把上层的 Src 设置 limit 来降低开销。

    + 对所有的 plan 计算 cost， 选出 cost 最小的一个，返回。
