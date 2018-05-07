**LikeSimplification**

LIKE是SQL中常见的操作，用于在WHERE子句中搜索列中的指定模式。使用的例子如下：

```sql
-- 从Persons表中选取姓名以“A”为开头的记录
SELECT * FROM Persons WHERE Name LIKE 'A%'

-- 从Persons表中选取城市名称以“g”为结尾的记录
SELECT * FROM Persons WHERE City LIKE '%g'
```

LIKE在Spark SQL逻辑计划中表示为：

```scala
case class Like(left: Expression, right: Expression) extends StringRegexExpression
```

其中left表示被匹配的列明，right表示匹配的目标正则表达式。

大家知道正则表达式匹配是比较耗时的操作，如果表中的每条记录都使用正则表达式取匹配的话，那计算速度就会变得很慢。本规则基于LIKE表达式中特定的匹配条件，将LIKE表达式进行适当转换，简化正则表达式匹配流程，提高计算效率。假设LIKE语句之后的模式为pattern，转换后的表达式为expr，它们的对应关系如下表所示：

| pattern               | expr                                               | 说明                                                         |
| --------------------- | -------------------------------------------------- | ------------------------------------------------------------ |
| `"([^_%]+)%"`         | StartsWith                                         | %放在模式最后，表示在特定列过滤以特定字符串开头的记录，使用StartsWith替代 |
| `"%([^_%]+)"`         | EndsWith                                           | %放在模式最前，表示在特定列过滤以特定字符串结尾的记录，使用EndsWith替代 |
| `"([^_%]+)%([^_%]+)"` | And(GreaterThanOrEqual, And(StartsWith, EndsWith)) | %放在模式中间，表示在特定列过滤以特定字符串s1开头且以特定字符串s2结尾的记录。使用三个表达式的And连接来表示，记录的长度必须大于等于s1和s2的长度之和，使用GreaterThanOrEqual表示；必须以s1开头，使用StartsWith表示；必须以s2结尾，使用EndsWith表示 |
| `"%([^_%]+)%"`        | Contains                                           | %放在模式开头和结尾，表示在特定列过滤包含特定字符串的记录，使用Contains替代 |
| `"([^_%]*)"`          | EqualTo                                            | 不包含%，表示在特定列中过滤与特定字符串相等的记录，使用EqualTo替代 |

一个例子如下：

**BooleanSimplification**

上一个规则是针对LIKE的表达式进行优化，通过将特定的LIKE表达式进行转换，避免了使用正则匹配带来的高额开销。本规则的思路与上一个规则相似，针对逻辑计划中与布尔值相关的与、或、非表达式进行优化，提前将其组合或者求值，提高计算效率。具体的场景和进行的优化如下表所示：

| 场景                                  | 优化                                                         |
| ------------------------------------- | ------------------------------------------------------------ |
| True常量与表达式做And操作             | 去掉True常量与And操作，使用表达式替代                        |
| False常量与表达式做Or操作             | 去掉False常量与Or操作，使用表达式替代                        |
| False常量与表达式做And操作            | 使用False常量替代                                            |
| True常量与表达式做Or操作              | 使用True常量替代                                             |
| 两个表达式做And操作且语义相反         | 使用False常量替代                                            |
| 两个表达式做Or操作且语义相反          | 使用True常量替代                                             |
| 两个表达式做And操作且语义相同         | 使用其中一个表达式替代                                       |
| 两个表达式做Or操作且语义相同          | 使用其中一个表达式替代                                       |
| a And (b Or c) 且a与b语义相反         | And(a, c)                                                    |
| a And (b Or c) 且a与c语义相反         | And(a, b)                                                    |
| (a Or b) And c 且a与c语义相反         | And(b, c)                                                    |
| (a Or b) And c 且b与c语义相反         | And(a, c)                                                    |
| a Or (b And c) 且a与b语义相反         | Or(a, c)                                                     |
| a Or (b And c) 且a与c语义相反         | Or(a, b)                                                     |
| (a And b) Or c 且a与c语义相反         | Or(b, c)                                                     |
| (a And b) Or c 且b与c语义相反         | Or(a, c)                                                     |
| And操作符的左右孩子表达式由Or连接而成 | 将左右孩子表达式按照Or展开，展开后如果一侧是另一侧的子集，则将子集元素使用Or连接后，替代原有And操作符；否则取公共子集，记其中元素为c1, ... , cn，两侧展开后抽去掉公共子集后的元素仍然使用Or连接，分别记为ldiff与rdiff，则转换后的表达式为c1 Or ... Or cn Or (ldiff And rdiff) |
| Or操作符的左右孩子表达式由And连接而成 | 将左右孩子表达式按照And展开，展开后如果一侧是另一侧的子集，则将子集元素使用And连接后，替代原有Or操作符；否则取公共子集，记其中元素为c1, ... , cn，两侧展开后抽去掉公共子集后的元素仍然使用And连接，分别记为ldiff与rdiff，则转换后的表达式为c1 And ... And cn And (ldiff Or rdiff) |
| Not(True)                             | 使用False常量替代                                            |
| Not(False)                            | 使用True常量替代                                             |
| Not(a GreaterThan b)                  | LessThanOrEqual(a, b)                                        |
| Not(a GreaterThanOrEqual b)           | LessThan(a, b)                                               |
| Not(a LessThan b)                     | GreaterThanOrEqual(a, b)                                     |
| Not(a LessThanOrEqual b)              | GreaterThan(a, b)                                            |
| Not(a Or b)                           | And(Not(a), Not(b))                                          |
| Not(a And b)                          | Or(Not(a), Not(b))                                           |
| Not(Not(e))                           | e                                                            |

**SimplifyConditionals**

此规则针对逻辑计划中的If和CaseWhen表达式进行优化，自底向上遍历节点的表达式，对于If的优化如下表所示：

| 场景                                | 优化               |
| ----------------------------------- | ------------------ |
| If(TrueLiteral, trueValue, _)       | 使用trueValue代替  |
| If(FalseLiteral, _, falseValue)     | 使用falseValue代替 |
| If(Literal(null, _), _, falseValue) | 使用falseValue代替 |

对于CaseWhen的优化如下表所示：

| 场景                                        | 优化                                                         |
| ------------------------------------------- | ------------------------------------------------------------ |
| CaseWhen的branches中包含false或者null的条件 | 如果branches中的全部条件均为false或者null，则使用elseValue替代原CaseWhen表达式；否则将条件为false或者null的branch过滤掉，只保留余下的branches |
| CaseWhen的第一个branch的条件为true          | 取第一个条件的值，替代原CaseWhen表达式                       |
| CaseWhen的banches中包含条件为true的分支     | 只保留条件为true的分支以及在此分支之前的分支，删除在此分支之后的分支以及elseValue |

一个例子如下：

**RemoveDispensableExpressions**

遍历逻辑计划中的所有表达式，将务必要的UnaryPositive和PromotePrecision节点去掉，只保留它们的孩子节点。

（此处应说明一下这两个节点的含义，以及为什么是不必要的）

**SimplifyBinaryComparison**

对于二元比较操作表达式，比如EqualTo、GreaterThan等，也可以通过结合其孩子表达式的关系来将其进行简化。具体的场景和优化规则如下：

| 场景                                        | 优化              |
| ------------------------------------------- | ----------------- |
| a EqualNullSafe b 且a和b语义相等            | 使用True常量代替  |
| a EqualTo b ，a、b非空且语义相等            | 使用True常量代替  |
| a GreaterThanOrEqual b ，a、b非空且语义相等 | 使用True常量代替  |
| a LessThanOrEqual b ，a、b非空且语义相等    | 使用True常量代替  |
| a GreaterThan b ，a、b非空且语义相等        | 使用False常量代替 |
| a LessThan b ，a、b非空且语义相等           | 使用False常量代替 |

一个例子如下：

**PruneFilters**

对于符合特定条件的Filter算子，可以通过对其进行转换，进行计算优化。处理的场景如下：

1. 如果filter condition为True常量，将Filter算子裁减掉，只保留孩子节点。
2. 如果filter condition为Null常量，则使用一个数据集合为空的LocalRelation替换Filter节点。
3. 如果filter condition为False常量，也使用一个数据集合为空的LocalRelation替换Filter节点。
4. 如果filter conditions中包含孩子节点的限制条件（constraints），则表示它们对每一个孩子节点的输入都会得到true的结果，所以将这些条件从filter conditions中删除。如果所有的过滤条件都是孩子节点的限制条件，则直接将Filter节点裁减掉。

本规则的一个示例如下：

**EliminateSorts**

Sort算子最主要的成员变量是order，它指定Sort操作排序的列以及升降序。如果order中包含可折叠的（foldable）表达式，则将其裁减掉；如果order集合为空，或者裁减后的order集合为空，则将Sort算子裁减掉。一个例子如下：

**SimplifyCasts**

（注：是否有必要在此处写明Cast的定义）

对于Cast表达式，如果被转换的表达式的类型和目标类型相同，则可以将其裁减掉。分为以下几种情况：

1. 如果孩子节点的类型和目标类型相同，则将Cast节点裁减掉，只保留孩子节点。
2. 如果孩子节点的类型为Array且containsNull属性为false，目标类型为Array但containsNull属性为true，并且两个Array类型中的元素类型相同，则将Cast节点裁减掉，只保留孩子节点。
3. 如果孩子节点的类型为Map且containsNull属性为false，目标类型为Map但containsNull属性为true，并且两个Map类型的key type和value type分别相同，则将Cast节点裁减掉，只保留孩子节点。

一个例子如下：

**SimplifyCaseConversionExpressions**

对于相邻出现的大小写转换算子Upper和Lower，因为孩子节点的输出会被父节点再次转换，故可以裁减掉孩子节点，只保留父节点。具体的场景和优化如下：

| 场景                | 优化         |
| ------------------- | ------------ |
| Upper(Upper(child)) | Upper(child) |
| Upper(Lower(child)) | Upper(child) |
| Lower(Upper(child)) | Lower(child) |
| Lower(Lower(child)) | Lower(child) |

**RewriteCorrelatedScalarSubquery**

（注：待补充）

**EliminateSerialization**

在执行计算时，有很多操作都涉及到将内存对象序列化和反序列化的过程，而序列化和反序列化也是十分耗费空间和时间的操作。本规则检测逻辑计划中出现的序列化和反序列化节点，并在必要时进行优化。优化的场景包含以下几种情况：

1. 孩子节点是SerializeFromObject的DeserializeToObject的节点。记孩子节点为s，父节点为d。如果d输出的数据类型和s输入的数据类型相同，则使用一个Project算子替代相邻的两个节点，project list为s输入数据类型的别名，起占位符作用。

2. 孩子节点是SerializeFromObject的AppendColumns节点。记孩子节点为s，父节点为a。如果a的反序列化器的数据类型和s的输出数据类型相同，则使用AppendColumnsWithObject，即AppendColumns的一个优化版本来代替原有的两个节点。AppendColumnsWithObject的定义如下：

   ```scala
   case class AppendColumnsWithObject(
       func: Any => Any,
       childSerializer: Seq[NamedExpression],
       newColumnsSerializer: Seq[NamedExpression],
       child: LogicalPlan) extends ObjectConsumer
   ```
   func为作用于child的记录之上的函数，childSerializer、newColumnsSerializer分别为孩子节点记录的序列化器和新生成的列的序列化器，child为孩子节点。

   新生成的AppendColumnsWithObject的func即为a的func，childSerializer、newColumnsSerializer分别为s的序列化器以及a的序列化器，child为s的孩子节点。

3. 孩子节点为SerializeFromObject的TypedFilter节点。记孩子节点为s，父节点为f。如果f的反序列化器的类型与s的输入数据类型相同，则将f下压到s和其孩子节点之间，提前过滤不符合条件的数据，减少后续操作数据量。

4. 孩子节点为TypedFilter的DeserializeToObject节点。记孩子节点为f，父节点为d。如果父节点的输出数据类型和f的反序列化器的数据类型相同，则将d下压到f和其孩子节点之间，先进行反序列化，再在反序列化生成的数据集上使用Filter算子过滤数据。这取代了优化前将数据使用TypedFilter先反序列化再过滤，然后再用DeserializeToObject反序列化的过程，相当于减少了一次反序列化，提高计算效率。

一个例子如下：

**RemoveRedundantAliases**

本规则自顶向下遍历逻辑计划，裁减掉不必要的别名（Alias）。如果一个别名没有改变孩子节点任何列的名称、元数据，并且没有对孩子节点的数据进行去重，那么认为它是不必要的别名。

一个例子如下：

**RemoveRedundantProject**

遍历逻辑计划，如果Project节点的输出与其孩子节点的输出相同，则认为此Project节点是冗余的，将其裁减掉。

一个例子如下：

**SimplifyCreateStructOps**

(注：待补充)

**SimplifyCreateArrayOps**

(注：待补充)

**SimplifyCreateMapOps**

(注：待补充)

**extendedOperatorOptimizationRules**

扩展的优化规则，放在此Batch的最后。通过继承Optimizer类并且覆写此变量，可以在此Batch规则执行之后，对逻辑计划应用自定义的优化规则。

#### Join Reorder Batch

此Batch只包含一个规则，执行策略为Once，用以调整Join的顺序。

**CostBasedJoinReorder**

#### Decimal Optimizations Batch

此Batch只包含一个规则，执行策略为fixedPoint。

**DecimalAggregates**

